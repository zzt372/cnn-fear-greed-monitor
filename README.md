# CNN Fear & Greed Monitor

CNN公式の Fear & Greed Index JSON をGitHub Actionsで高頻度に取得・検証し、ChatGPTなどの外部監視から参照しやすい `latest.json` として公開するためのリポジトリです。

第三者サイトやproxy値は使わず、CNN公式endpoint由来の値だけを採用します。

## Architecture

```text
CNN official JSON
      ↓
fetch.py
  ├─ root JSON × retry
  └─ dated JSON × retry fallback
      ↓ validation
  latest.json
      ↓
GitHub Raw / REST API
      ↓
ChatGPT / 外部監視
```

取得・JSON解析・値検証に成功した場合だけ `latest.json` を生成します。取得失敗時は既存のlast-known-good `latest.json` を壊しません。

## Data source

- Primary: `https://production.dataviz.cnn.io/index/fearandgreed/graphdata`
- Fallback: `https://production.dataviz.cnn.io/index/fearandgreed/graphdata/YYYY-MM-DD`
- Official page: `https://www.cnn.com/markets/fear-and-greed`

## 5分ポーリング

Primary workflowはGitHub Actionsの最短schedule間隔である**5分ごと**に実行します。

```yaml
schedule:
  - cron: '*/5 * * * *'
```

1時間あたり最大12回の取得機会を持たせ、scheduled workflowの遅延・dropや一時的な通信失敗が数回起きても、次の実行で自動回復しやすい構成にしています。

`workflow_dispatch` による手動実行にも対応しています。

## Commit抑制

API確認自体は5分ごとに行いますが、`fetched_at` の変化だけで毎回commitすると最大288 commit/日になるため、Git履歴の無駄な増加を抑えています。

Primary workflowでは次のルールで `latest.json` を公開します。

- `category` が変化した場合: 即commit
- `ok` / `source` / `endpoint` が変化した場合: 即commit
- 上記に変化がない場合: 原則として1時間ごとのheartbeat commit

つまり、**取得は5分ごと、重要な変化は即時公開、平常時のcommitは抑制**という設計です。

## Independent Watchdog

Primary workflowとは別に `.github/workflows/watchdog.yml` を用意しています。

Watchdogは毎時 `07 / 22 / 37 / 52` 分に独立して起動し、公開済み `latest.json` の `fetched_at` が45分以上更新されていない場合だけ `fetch.py` を再実行して復旧を試みます。

```text
Primary 5-minute poll
      ↓
正常なら継続

published latest.json stale >= 45 min
      ↓
Watchdog recovery
      ↓
fetch.py → validate → commit
```

PrimaryとWatchdogの両方に同じconcurrency groupを設定し、`cancel-in-progress: true` で古い競合実行を引きずりにくくしています。

## Retry / Fallback

`fetch.py` はCNN公式の2経路を使います。

```text
root JSON
  ├─ attempt 1
  ├─ attempt 2
  └─ attempt 3
       ↓ failure
dated JSON
  ├─ attempt 1
  ├─ attempt 2
  └─ attempt 3
       ↓ failure
workflow failure
       ↓
既存 latest.json を保持
```

各HTTP取得にはtimeoutを設定し、一時的なHTTP・DNS・ネットワーク・JSON解析エラーを再試行します。

## Validation

`latest.json` に採用する前に少なくとも次を検証します。

- scoreが数値
- scoreが0〜100
- timestampが解釈可能
- timestampが大幅に未来ではない
- timestampが明らかに古すぎない
- CNN公式JSON内の必要ブロックが存在する

カテゴリは次の境界で正規化します。

| Score | Category |
|---:|---|
| `0 <= x < 25` | Extreme Fear |
| `25 <= x < 45` | Fear |
| `45 <= x <= 55` | Neutral |
| `55 < x <= 75` | Greed |
| `75 < x <= 100` | Extreme Greed |

## Output

`latest.json` には以下を保存します。

| Field | 内容 |
|---|---|
| `score` | CNN公式スコア |
| `rating` | CNN公式ratingを正規化した文字列 |
| `category` | 監視比較用カテゴリ |
| `timestamp` | CNN側timestamp (UTC) |
| `fetched_at` | GitHub Actionsで取得した時刻 (UTC) |
| `source` | `CNN official` |
| `source_url` | 実際に使用したCNN公式URL |
| `endpoint` | `root JSON` または `dated JSON` |
| `ok` | 正常取得・検証済みなら `true` |

## Raw JSON

```text
https://raw.githubusercontent.com/zzt372/cnn-fear-greed-monitor/main/latest.json
```

GitHub REST Contents APIから同じファイルを取得することもできます。

## Failure policy

このモニターは「1回も失敗しない」ことではなく、**一時障害をlast-known-goodデータへ波及させず、複数の次回実行機会で自動回復すること**を重視しています。

```text
schedule / network failure
        ↓
次の5分poll
        ↓
さらに失敗しても次のpoll
        ↓
公開heartbeatが45分以上停止
        ↓
Independent Watchdog
```

外部consumer側でも、単発の取得失敗だけで正常値を破棄せず、`ok`・`source`・`timestamp`・`fetched_at` を確認してlast-known-goodを扱う設計を推奨します。

## Files

```text
.
├── .github/
│   └── workflows/
│       ├── update.yml       # 5分ごとのPrimary updater
│       └── watchdog.yml     # stale heartbeat復旧用Watchdog
├── fetch.py                 # CNN公式取得・retry・validation
├── latest.json              # 最新の検証済み正常値
└── README.md
```

## GitHub Actions permissions

`latest.json` を同一リポジトリへcommitするため、workflowでは次の権限だけを明示しています。

```yaml
permissions:
  contents: write
```

## Notes

- このリポジトリはCNN公式プロジェクトではありません。
- このリポジトリは投資助言を提供するものではありません。
- GitHub Actionsの`schedule`は厳密な実行時刻を保証しません。
- 外部サービスを利用する以上100%の稼働保証はできませんが、5分poll、retry、公式2経路fallback、last-known-good保持、独立Watchdogで障害耐性を高めています。

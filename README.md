# CNN Fear & Greed Monitor

CNN公式の Fear & Greed Index JSON をGitHub Actionsで毎時取得し、ChatGPTなどから読みやすい最小JSON (`latest.json`) として公開するためのリポジトリです。

## Data source

- Primary: `https://production.dataviz.cnn.io/index/fearandgreed/graphdata`
- Fallback: `https://production.dataviz.cnn.io/index/fearandgreed/graphdata/YYYY-MM-DD`
- Official page: `https://www.cnn.com/markets/fear-and-greed`

第三者サイトの指数値は使用しません。

## Output

`latest.json` には以下を保存します。

- `score`: CNN公式スコア（元の数値）
- `rating`: CNN公式ratingを正規化した文字列
- `category`: 通知比較用の標準カテゴリ
- `timestamp`: CNN側timestamp (UTC)
- `fetched_at`: GitHub Actionsで取得した時刻 (UTC)
- `endpoint`: 実際に成功したCNN公式endpoint
- `ok`: 正常取得できた場合 `true`

取得に失敗した場合は `latest.json` を上書きせず、workflowを失敗させます。

## Schedule

GitHub Actionsは毎時17分 (UTC基準の毎時 `:17`) に実行します。GitHubの混雑しやすい毎時00分を避けています。

手動実行 (`workflow_dispatch`) と、取得コード・workflow変更時の自動テスト (`push`) にも対応しています。

## Raw JSON

`https://raw.githubusercontent.com/zzt372/cnn-fear-greed-monitor/main/latest.json`

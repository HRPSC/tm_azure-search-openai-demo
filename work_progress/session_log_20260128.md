# セッションログ 2026-01-28

## 目的
Azure App Service `hrpsc-ai` の起動問題の解決

## 実施内容

### 1. 前日からの状況確認
- アプリは起動タイムアウト（230秒）で失敗し続けていた
- `WEBSITES_CONTAINER_START_TIME_LIMIT=600` は設定済み

### 2. 認証エラーの対応
- アプリUIは表示されたが、チャット機能で `openai.AuthenticationError` が発生
- 原因: `AZURE_OPENAI_API_KEY_OVERRIDE` が空だった
- 対応: APIキーを設定

### 3. 環境変数の追加設定
- `WEBSITES_PORT=8000` を追加（ポート設定の明示化）
- `AZURE_OPENAI_API_KEY_OVERRIDE` にAPIキーを設定

### 4. 起動タイムアウト問題の調査
ログ分析の結果、起動プロセスに約6-7分かかっていることが判明：
- 証明書更新: 約1-2分
- `output.tar.gz` 展開: 約3分
- Gunicorn起動: 数秒

Gunicornは起動するが、その後アプリの初期化処理でハングしている模様。

### 5. 原因の絞り込み
確認済み:
- `AZURE_SEARCH_AGENT`: 空
- `USE_AGENTIC_RETRIEVAL`: false
- AI Search (`cognitive-search-for-oai-je`): パブリックアクセス「すべてのネットワーク」で問題なし

未確認:
- **Azure OpenAI (`hrpsc-oai-eastus2`) のネットワーク設定** ← 次回要確認

### 6. azd deploy の試行
- エラー: `resource not found: unable to find a resource tagged with 'azd-service-name: backend'`
- 原因: `hrpsc-ai` は azd でプロビジョニングされていないため、タグがない

## 現在の状態
- Gunicornは起動するが、アプリの初期化処理でハング
- 504 Gateway Timeout が発生
- Azure OpenAI へのネットワーク接続が疑わしい

## 次回のタスク

### 優先度高
1. **Azure OpenAI (`hrpsc-oai-eastus2`) のネットワーク設定を確認**
   - VNet からのアクセスが許可されているか
   - `hrpsc-ai` の VNet/サブネット (`vnet-ag-hrpx/vnet-ag-subnet-ai`) がルールに含まれているか

2. ネットワーク設定に問題があれば修正

3. 修正後、App Service を再起動して動作確認

### 代替案
- VNet 統合を一時的に無効にしてテスト（パブリックアクセスで動作確認）
- Azure CLI で直接デプロイ: `az webapp deploy`

## 環境変数の現在の設定

| 変数名 | 値 |
|--------|-----|
| AZURE_OPENAI_SERVICE | hrpsc-oai-eastus2 |
| AZURE_OPENAI_CHATGPT_DEPLOYMENT | gpt-5.2-eastus2 |
| AZURE_OPENAI_CHATGPT_MODEL | gpt-5.2 |
| AZURE_OPENAI_EMB_DEPLOYMENT | text-embedding-3-large-eastus2 |
| AZURE_OPENAI_EMB_MODEL_NAME | text-embedding-3-large |
| AZURE_OPENAI_API_VERSION | 2025-04-01-preview |
| AZURE_OPENAI_API_KEY_OVERRIDE | (設定済み) |
| AZURE_SEARCH_SERVICE | cognitive-search-for-oai-je |
| AZURE_SEARCH_INDEX | branch-vector-large3-2nd-718346267580 |
| WEBSITES_PORT | 8000 |
| WEBSITES_CONTAINER_START_TIME_LIMIT | 600 |

## 参考情報
- 起動ログでは Gunicorn が `http://0.0.0.0:8000` でリッスン開始
- Worker 3つが起動 (pid: 2123, 2124, 2125)
- その後、新しいトレースが出ない状態が続く

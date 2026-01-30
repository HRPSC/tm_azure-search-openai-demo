# セッションログ 2026-01-29

## 目的
Azure App Service `hrpsc-ai` の起動タイムアウト問題の解決

## 実施内容

### 1. 前日からの状況確認
- Gunicornは起動するが、アプリの初期化処理でハング
- 504 Gateway Timeout が発生
- VNet統合とAzure OpenAIのネットワーク設定は正しく構成されていることを確認済み

### 2. Azure リソース情報のドキュメント化
`work_progress/azure_resources_inventory.md` を作成し、以下の情報を記録：
- Azure OpenAI (hrpsc-oai-eastus2) のネットワーク設定
- App Service (hrpsc-ai) のVNet統合設定
- AI Search, Application Gateway, VNetの構成
- 環境変数の設定

**確認結果**:
- App Service `hrpsc-ai` のVNet統合: 正常（vnet-ag-hrpx/vnet-ag-subnet-ai）
- Azure OpenAI側の許可リスト: VNetが含まれている ✓
- ネットワーク設定に問題なし

### 3. 最新ログの分析
`hrpsc-ai-logs-latest.zip` をダウンロードして分析

**起動プロセスのタイムライン**（最新の試行 03:36-03:47）:
1. 03:36:57 - コンテナ起動開始
2. 03:38:25 - 証明書更新完了（約1分30秒）
3. 03:38:30 - output.tar.gz展開開始
4. 03:41:06 - 展開完了（約2分36秒）
5. 03:41:10 - Gunicorn起動、3つのworkerが起動
6. **03:41:10以降 - アプリケーションログが一切出力されない**
7. 03:46:59 - 起動タイムアウト（600秒経過）

### 4. 問題の特定
`app/backend/app.py` の `setup_clients()` 関数を分析：

**ブロックしている可能性が高い処理**:
1. **ManagedIdentityCredential の初期化**（行490-500）
2. **SearchClient の初期化**（行512-516）
3. **AZURE_USE_AUTHENTICATION=true の場合**:
   - `search_index_client.get_index()` でAzure Searchへ接続（行536）
   - これがVNet経由でタイムアウトしている可能性

4. **USE_USER_UPLOAD=true の場合**:
   - `setup_search_info()` でAzure Searchへ接続（行608-610）

### 5. 対応策の実施
診断のため、以下の環境変数を設定：
```
AZURE_USE_AUTHENTICATION=false
APP_LOG_LEVEL=DEBUG
GUNICORN_CMD_ARGS="--log-level debug --access-logfile - --error-logfile -"
```

**理由**:
- `AZURE_USE_AUTHENTICATION=false` により、Search Index取得処理をスキップ
- デバッグログを有効化して、どこでブロックしているか特定

App Serviceを再起動して動作確認中。

## 現在の仮説

**問題**: `setup_clients()` 関数内で、Azure Searchへの接続がVNet経由でタイムアウトしている

**根拠**:
1. Gunicorn workerは正常に起動している
2. その後、アプリケーションログが一切出力されない
3. `@bp.before_app_serving` デコレータで `setup_clients()` が実行される
4. この関数内でAzure Searchへの同期的な接続が試みられる

**次のステップ**:
1. `AZURE_USE_AUTHENTICATION=false` で起動するか確認
2. 起動した場合、Azure Searchへの接続が問題と確定
3. 起動しない場合、他の初期化処理（Managed Identity、OpenAI Client等）を調査

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
| AZURE_USE_AUTHENTICATION | false ← 変更 |
| APP_LOG_LEVEL | DEBUG ← 追加 |
| GUNICORN_CMD_ARGS | --log-level debug --access-logfile - --error-logfile - ← 追加 |
| WEBSITES_PORT | 8000 |
| WEBSITES_CONTAINER_START_TIME_LIMIT | 600 |

## 参考情報

### ネットワーク構成
- App Service: rg-hrpsc-ai-production/hrpsc-ai
- VNet統合: rg-oai-je/vnet-ag-hrpx/vnet-ag-subnet-ai (10.1.4.0/24)
- サービスエンドポイント: Microsoft.CognitiveServices, Microsoft.AzureCosmosDB
- Azure OpenAI: rg-hrpsc-oai-eastus2/hrpsc-oai-eastus2
  - VNet許可リスト: vnet-ag-hrpx/vnet-ag-subnet-ai ✓
- AI Search: rg-oai-je/cognitive-search-for-oai-je
  - publicNetworkAccess: Enabled

### .gitignore更新
ログファイルをリポジトリから除外するため、以下を追加：
```
# Azure logs and diagnostics
hrpsc-ai-logs/
hrpsc-ai-logs.zip
hrpsc-ai-logs-latest/
hrpsc-ai-logs-latest.zip
*-logs/
*-logs.zip
```

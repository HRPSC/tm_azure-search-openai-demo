# セッションログ 2026-01-27

## 目的
Azure App Service `hrpsc-ai` の復旧と設定変更

## 実施内容

### 1. 問題の特定
- `hrpsc-ai.azurewebsites.net` で「Application Error」が発生
- 原因: Azure OpenAI サービス `cog-5n4lediudihnq` が削除されていた

### 2. Azure OpenAI の設定変更
既存の `hrpsc-oai-eastus2` を使用するように変更:
- エンドポイント: `https://hrpsc-oai-eastus2.openai.azure.com/`
- Chat モデル: `gpt-5.1-eastus2` (gpt-5.1)
- Embedding モデル: `text-embedding-3-large-eastus2` (新規追加)

App Service 環境変数を更新:
```
AZURE_OPENAI_SERVICE=hrpsc-oai-eastus2
AZURE_OPENAI_CHATGPT_MODEL=gpt-5.1
AZURE_OPENAI_CHATGPT_DEPLOYMENT=gpt-5.1-eastus2
AZURE_OPENAI_EMB_MODEL_NAME=text-embedding-3-large
AZURE_OPENAI_EMB_DEPLOYMENT=text-embedding-3-large-eastus2
AZURE_OPENAI_API_VERSION=2024-12-01-preview
```

### 3. VNet 統合
- `hrpsc-ai` を `vnet-ag-hrpx/vnet-ag-subnet-ai` に VNet 統合
- `hrpsc-oai-eastus2` のネットワークルールに `vnet-ag-subnet-ai` を追加

### 4. AI Search の設定変更
`hrpsc-dx` と同じ AI Search を使用するように変更:
```
AZURE_SEARCH_SERVICE=cognitive-search-for-oai-je
AZURE_SEARCH_INDEX=branch-vector-large3-2nd-718346267580
AZURE_SEARCH_FIELD_NAME_EMBEDDING=text_vector
```

### 5. 起動タイムアウト対応
- `WEBSITES_CONTAINER_START_TIME_LIMIT=600` を設定（デフォルト230秒→600秒）

## 現在の状態
- アプリは起動タイムアウトで失敗中
- `output.tar.gz` の展開に時間がかかりすぎている
- 再デプロイが必要な可能性あり

## 次回のタスク

### 優先度高
1. アプリの起動確認（タイムアウト延長後）
2. 起動しない場合は `azd deploy` で再デプロイ
3. 動作確認（チャット機能のテスト）

### 検討事項
- CosmosDB の共有について
  - `hrpsc-dx` は `db-hrpsc-dx` を使用
  - スキーマが異なるため直接共有は難しい
  - 同じアカウント内に別コンテナを作成することでコスト削減可能
- リポジトリの更新（Azure-Samples/azure-search-openai-demo の最新版への追従）

## 変更したリソース

| リソース | 変更内容 |
|---------|---------|
| hrpsc-ai (App Service) | 環境変数更新、VNet統合 |
| hrpsc-oai-eastus2 (OpenAI) | VNetルール追加 |

## 参考情報

### hrpsc-dx の設定（参考）
- OpenAI: `oai-jaeast`
- AI Search: `cognitive-search-for-oai-je`
- CosmosDB: `db-hrpsc-dx` / `db_conversation_history` / `conversations`

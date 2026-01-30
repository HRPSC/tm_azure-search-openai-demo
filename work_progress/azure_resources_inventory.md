# Azure リソース一覧

最終更新: 2026-01-29

## 1. Azure OpenAI

| リソース名 | リソースグループ | リージョン | ネットワーク設定 |
|-----------|-----------------|-----------|-----------------|
| hrpsc-oai-eastus2 | rg-hrpsc-oai-eastus2 | East US 2 | 下記参照 |
| hrpsc-oai-uswest | rg-uswest | West US | - |
| hrpsc-oai-swedencentral | rg-oai-swedencentral | Sweden Central | - |

### hrpsc-oai-eastus2 ネットワーク設定

- **publicNetworkAccess**: Enabled
- **defaultAction**: Deny

#### 許可されているVNet
| VNet/Subnet | リソースグループ |
|-------------|-----------------|
| vnet01/subnet-1 | rg-hrpsc-oai-eastus2 |
| vnet-ag-hrpx/vnet-ag-subnet-ai | rg-oai-je |

#### 許可されているIP
- 133.254.5.0/24
- 133.254.130.0/24
- 133.254.131.0/24
- 210.160.8.64/27
- 147.161.192.0/24
- 147.161.194.0/24
- 48.218.79.47

---

## 2. App Service

| リソース名 | リソースグループ | リージョン | 用途 |
|-----------|-----------------|-----------|------|
| hrpsc-ai | rg-hrpsc-ai-production | Japan East | RAGアプリ本番 |
| hrpsc-dx | rg-oai-je | Japan East | - |
| hrpsc-gb | rg-uswest | Japan East | - |
| hrpsc-bz | rg-hrpsc-bz | Japan East | - |
| hrpsc-ai-assistants | rg-hrpsc-ai-assistants | Japan East | - |
| hrpscHelloWorldSample | maeda.takio_rg_8129 | East US | - |

### hrpsc-ai VNet統合

- **VNet**: vnet-ag-hrpx
- **Subnet**: vnet-ag-subnet-ai
- **VNet リソースグループ**: rg-oai-je

---

## 3. AI Search (Cognitive Search)

| リソース名 | リソースグループ | リージョン | publicNetworkAccess |
|-----------|-----------------|-----------|---------------------|
| cognitive-search-for-oai-je | rg-oai-je | Japan East | Enabled |
| gptkb-2r63n7e4nttnq | rg-hrpsc-ai-final | Japan East | - |
| gptkb-5n4lediudihnq | rg-hrpsc-ai-production | Japan East | - |

---

## 4. Application Gateway

| リソース名 | リソースグループ | リージョン |
|-----------|-----------------|-----------|
| ag-hrpx | rg-oai-je | Japan East |

### バックエンドプール

| プール名 | バックエンドアドレス |
|---------|---------------------|
| ag-be-hrpx-ai | hrpsc-ai.azurewebsites.net |
| ag-be-hrpx | hrpsc-dx.azurewebsites.net |
| ag-be-gb | hrpsc-gb.azurewebsites.net |
| ag-be-hello | hrpschelloworldsample.azurewebsites.net |
| ag-be-n8n | 10.0.1.4 |
| pool-verify-temp | verify-temp-hrpx.azurewebsites.net |
| verification-backend | sthrpxverify.blob.core.windows.net |

### フロントエンドIP
- パブリックIP: ip-ag-hrpx (rg-oai-je)

---

## 5. VNet (主要なもの)

| VNet名 | リソースグループ | リージョン | アドレス空間 |
|--------|-----------------|-----------|-------------|
| vnet-ag-hrpx | rg-oai-je | Japan East | 10.1.0.0/16 |
| vnet-oai-je | rg-oai-je | Japan East | 10.0.0.0/16 |
| vnet01 | rg-hrpsc-oai-eastus2 | East US 2 | 172.16.0.0/26 |
| vnet-hrpsc-oai-eastus2 | rg-hrpsc-ai | East US 2 | 10.16.0.0/16 |
| vnet-swedencentral | rg-oai-swedencentral | Sweden Central | 10.17.0.0/16 |

### vnet-ag-hrpx サブネット

| サブネット名 | アドレス範囲 | サービスエンドポイント |
|-------------|-------------|----------------------|
| default | 10.1.0.0/24 | Microsoft.Web |
| vnet-ag-subnet1 | 10.1.1.0/24 | CosmosDB, CognitiveServices, Web |
| vnet-ag-subnet2 | 10.1.2.0/24 | CosmosDB, CognitiveServices |
| vnet-ag-subnet3 | 10.1.3.0/24 | CognitiveServices |
| vnet-ag-subnet-ai | 10.1.4.0/24 | CognitiveServices, CosmosDB |
| subnet-db-dx | 10.1.5.0/24 | なし |
| subnet-db-ai | 10.1.6.0/24 | なし |
| vnet-ag-subnet-4 | 10.1.7.0/24 | CognitiveServices |

---

## 6. 接続関係図

```
[ユーザー]
    |
    v
[Application Gateway: ag-hrpx] (rg-oai-je)
    |
    v
[App Service: hrpsc-ai] (rg-hrpsc-ai-production)
    | VNet統合: vnet-ag-hrpx/vnet-ag-subnet-ai
    |
    +---> [Azure OpenAI: hrpsc-oai-eastus2] (rg-hrpsc-oai-eastus2)
    |     ※ VNet許可済み
    |
    +---> [AI Search: cognitive-search-for-oai-je] (rg-oai-je)
          ※ パブリックアクセス有効
```

---

## 7. hrpsc-ai 環境変数 (主要)

| 変数名 | 値 |
|--------|-----|
| AZURE_OPENAI_SERVICE | hrpsc-oai-eastus2 |
| AZURE_OPENAI_CHATGPT_DEPLOYMENT | gpt-5.2-eastus2 |
| AZURE_OPENAI_CHATGPT_MODEL | gpt-5.2 |
| AZURE_OPENAI_EMB_DEPLOYMENT | text-embedding-3-large-eastus2 |
| AZURE_OPENAI_EMB_MODEL_NAME | text-embedding-3-large |
| AZURE_OPENAI_API_VERSION | 2025-04-01-preview |
| AZURE_SEARCH_SERVICE | cognitive-search-for-oai-je |
| AZURE_SEARCH_INDEX | branch-vector-large3-2nd-718346267580 |
| WEBSITES_PORT | 8000 |
| WEBSITES_CONTAINER_START_TIME_LIMIT | 600 |

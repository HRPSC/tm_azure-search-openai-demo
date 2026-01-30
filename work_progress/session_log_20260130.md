# セッションログ 2026-01-30

## 目的

Azure App Service `hrpsc-ai` の起動タイムアウト問題の解決（継続）

## 実施内容

### 1. 前日（1/29）の状況確認

**前回の対応**:
- `AZURE_USE_AUTHENTICATION=false` に設定
- `APP_LOG_LEVEL=DEBUG` を追加
- Gunicornのデバッグログを有効化
- App Serviceを再起動して動作確認中

**問題**:
- App Serviceが依然として起動タイムアウト（600秒）
- https://hrpsc-ai.azurewebsites.net/ にアクセスしても表示されない

### 2. 最新ログの分析

最新のログ（`hrpsc-ai-logs-latest3.zip`）をダウンロードして分析：

**起動試行のタイムライン**（2026-01-30 06:24-06:36）:
1. 06:24:30 - コンテナ起動開始
2. 06:25:01 - コンテナ起動完了
3. 06:34:32-06:36:20 - **Pythonモジュールのインポート処理中**
4. 06:36:20 - **ログ出力が完全に停止**
5. 06:36:47 - コンテナがタイムアウトで強制終了

**重要な発見**:
- ログに大量の `# code object from` や `# bytecode is stale` などのメッセージ
- これは **Python verbose import モード** の出力
- インポート処理の詳細が延々と出力され、途中で停止

### 3. 根本原因の特定

環境変数を確認した結果、**`PYTHONVERBOSE=1`** が設定されていることを発見：

```bash
az webapp config appsettings list --name hrpsc-ai --resource-group rg-hrpsc-ai-production --query "[?contains(name, 'PYTHON')]"
```

**結果**:
- `PYTHON_ENABLE_GUNICORN_MULTIWORKERS=true`
- **`PYTHONVERBOSE=1`** ← これが原因！

**問題の詳細**:
- `PYTHONVERBOSE=1` により、すべてのPythonモジュールのインポート処理が詳細にログ出力される
- 大量のログ出力によるI/Oオーバーヘッドで起動が極端に遅くなる
- 600秒のタイムアウト内に起動が完了しない

### 4. 修正作業

#### 4.1 PYTHONVERBOSE環境変数の削除

```bash
az webapp config appsettings delete --name hrpsc-ai --resource-group rg-hrpsc-ai-production --setting-names PYTHONVERBOSE
```

#### 4.2 GUNICORN_CMD_ARGSの最適化

デバッグログレベルから通常のinfoレベルに変更：

```bash
az webapp config appsettings set --name hrpsc-ai --resource-group rg-hrpsc-ai-production --settings GUNICORN_CMD_ARGS="--timeout 600 --access-logfile - --error-logfile - --log-level info"
```

#### 4.3 App Serviceの再起動

```bash
az webapp restart --name hrpsc-ai --resource-group rg-hrpsc-ai-production
```

## 現在の状態

### 環境変数の設定（修正後）

| 変数名 | 値 | 変更 |
|--------|-----|------|
| `AZURE_USE_AUTHENTICATION` | `false` | 前回設定 |
| `USE_AGENTIC_RETRIEVAL` | `false` | 前回設定 |
| `AZURE_SEARCH_AGENT` | （空） | - |
| `PYTHONVERBOSE` | **削除** | ✓ 本日削除 |
| `GUNICORN_CMD_ARGS` | `--timeout 600 --access-logfile - --error-logfile - --log-level info` | ✓ 本日変更 |
| `APP_LOG_LEVEL` | `DEBUG` | 前回設定 |
| `WEBSITES_CONTAINER_START_TIME_LIMIT` | `600` | - |
| `WEBSITES_PORT` | `8000` | - |

### App Serviceの状態

- State: `Running`
- URL: https://hrpsc-ai.azurewebsites.net/
- 再起動後、約90秒待機
- **動作確認は次回セッションで実施**

## 問題の根本原因まとめ

1. **`PYTHONVERBOSE=1`** が設定されていた
2. すべてのPythonモジュールインポートの詳細がログ出力される
3. 大量のログI/Oにより起動が極端に遅延
4. 600秒のタイムアウト内に起動完了できず

## 次回の作業予定

### 1. 動作確認

- https://hrpsc-ai.azurewebsites.net/ にアクセス
- アプリケーションが正常に表示されるか確認
- レスポンス時間を確認

### 2. ログの確認

- 最新のログをダウンロード
- 起動時間を確認（目標：60秒以内）
- エラーがないか確認

### 3. 正常起動した場合

- `AZURE_USE_AUTHENTICATION=true` に戻す
- 認証機能を有効化して動作確認
- `USE_AGENTIC_RETRIEVAL` の設定を検討

### 4. まだ問題がある場合

- 最新ログから新たなブロック箇所を特定
- 必要に応じて追加の環境変数調整

## 学んだこと

1. **`PYTHONVERBOSE` は本番環境では絶対に使用しない**
   - デバッグ時のみローカル環境で使用
   - 大量のログ出力により起動が極端に遅くなる

2. **App Serviceのログ分析の重要性**
   - ログの内容から問題の本質を見抜く
   - verbose importログは異常な状態のサイン

3. **環境変数の確認は必須**
   - 予期しない環境変数が設定されている可能性
   - `PYTHON*` で始まる環境変数は特に注意

## 参考情報

### Python環境変数

- `PYTHONVERBOSE`: Pythonのverboseモード（0または1）
- `PYTHON_ENABLE_GUNICORN_MULTIWORKERS`: Gunicornのマルチワーカー有効化

### Gunicorn設定

- `--timeout 600`: ワーカータイムアウト（秒）
- `--log-level info`: ログレベル（debug, info, warning, error, critical）
- `--access-logfile -`: アクセスログを標準出力
- `--error-logfile -`: エラーログを標準出力

### App Service設定

- `WEBSITES_CONTAINER_START_TIME_LIMIT`: コンテナ起動タイムアウト（秒）
- `WEBSITES_PORT`: アプリケーションのポート番号

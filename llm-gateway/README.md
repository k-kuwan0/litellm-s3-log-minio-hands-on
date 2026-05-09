# LLM Gateway (LiteLLM + S3 ログ + MinIO)

LiteLLM Proxy を使って Amazon Bedrock を呼び出し、リクエストログを S3 互換ストレージ（ローカルでは MinIO）に出力するハンズオン環境です。

## 構成

| サービス | 役割 | ポート |
|---|---|---|
| litellm | LLM Proxy ゲートウェイ | 4000 |
| postgres | LiteLLM の設定/ログ DB | 5432 |
| minio | S3 互換オブジェクトストレージ | 9000 (API) / 9001 (Console) |
| minio-init | 起動時にバケットを作成する使い捨てコンテナ | - |

LiteLLM の `s3_v2` success callback が、リクエスト 1 回ごとに `StandardLoggingPayload` を JSON として MinIO バケット `litellm-log-archive` に書き出します。

## 前提

- Docker Desktop
- [aws-vault](https://github.com/99designs/aws-vault) （Bedrock 呼び出し用の AWS 認証情報を渡すため）
- Bedrock のモデルアクセスが許可された AWS アカウント・プロファイル
  - 本ハンズオンでは `amazon.nova-lite-v1:0` を `ap-northeast-1` で利用

## 起動

```bash
aws-vault exec <profile> -- docker compose up -d
```

`<profile>` には Bedrock を呼び出せる AWS プロファイル名を指定。

起動確認:

```bash
curl -s http://localhost:4000/health/readiness
```

## 動作確認

### Bedrock 呼び出し

```bash
curl -sS -X POST http://localhost:4000/v1/chat/completions \
  -H "Authorization: Bearer sk-local-dev-key" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "nova-lite",
    "messages": [{"role":"user","content":"hello"}],
    "max_tokens": 30
  }'
```

### MinIO に書き出されたログを見る

ブラウザで MinIO Console を開く:

- URL: http://localhost:9001
- ユーザー: `minioadmin`
- パスワード: `minioadmin`

`litellm-log-archive` バケット → `request_logs/<日付>/` に JSON ファイルが並びます。中身は `messages`（プロンプト）, `response`, トークン数, コスト, タイムスタンプなどを含む `StandardLoggingPayload` 形式です。

CLI で見る場合:

```bash
docker exec litellm-minio mc alias set local http://localhost:9000 minioadmin minioadmin
docker exec litellm-minio mc ls --recursive local/litellm-log-archive
```

### LiteLLM Admin UI

- URL: http://localhost:4000/ui
- ユーザー: `admin`
- パスワード: `sk-local-dev-key`

ここで Virtual Key やチームを発行すると、それぞれの alias が S3 のキー階層に反映されます（`request_logs/<team_alias>/<key_alias>/...`）。

## 設定ファイルの構成

```
llm-gateway/
├── docker-compose.yaml       # サービス定義（ローカル環境）
├── lite-llm/
│   └── config.local.yaml     # ローカル開発用 LiteLLM 設定（MinIO 接続情報込み）
├── database/postgresql/data/ # PostgreSQL の永続データ
└── storages/minio/data/      # MinIO の永続データ
```

`docker-compose.yaml` は `config.local.yaml` を `/app/config.yaml` にマウントしてコンテナへ渡しています。本番（ECS 等）で利用する想定の設定ファイルは本リポジトリには含めていません。

## 設計判断（重要）

### Bedrock と MinIO の認証情報を分離している

ローカルでは

- **Bedrock 呼び出し**: aws-vault が発行する AWS の STS 一時クレデンシャル（`AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` / `AWS_SESSION_TOKEN`）
- **MinIO へのログ書き込み**: ローカル固定の `minioadmin` / `minioadmin`

の 2 系統のクレデンシャルが必要です。`docker-compose.yaml` では:

- `BEDROCK_AWS_*` という別名で aws-vault の値を受け取り、`config.local.yaml` の `model_list.litellm_params` で明示的に指定 → Bedrock 認証
- `S3_AWS_*` で MinIO 用の値を渡し、`config.local.yaml` の `s3_callback_params` で明示的に指定 → MinIO 認証
- `AWS_SESSION_TOKEN: ""` でコンテナ内のデフォルト環境変数を空に上書き

としています。

#### なぜ `AWS_SESSION_TOKEN: ""` が必要か

LiteLLM の s3_v2 callback は内部で boto3 を使いますが、`s3_aws_access_key_id` / `s3_aws_secret_access_key` を明示しても、`aws_session_token` を指定しないと **環境変数 `AWS_SESSION_TOKEN` を自動でフォールバックして拾います**。

aws-vault が出した STS のセッショントークンが MinIO への PUT に `X-Amz-Security-Token` ヘッダとして付与されると、MinIO は「自身が発行していない不明なトークン」として `InvalidTokenId` で 403 を返します。

これを避けるため、**コンテナ内の `AWS_SESSION_TOKEN` を明示的に空に上書き**し、Bedrock には `BEDROCK_AWS_SESSION_TOKEN` 経由で別途渡しています。

### `config.local.yaml` の位置付け

`config.local.yaml` はローカル開発専用の LiteLLM 設定です。MinIO 接続情報や Bedrock 用の明示的な認証情報など、本番（ECS 等）には不要なローカル都合の設定を含みます。

本番用の設定は本リポジトリでは管理せず、ECS タスク定義や ConfigMap など別手段で配布する想定です。本番では `aws_session_token` の明示や MinIO 関連の `s3_callback_params` は不要で、Bedrock も S3 も Task Role に任せる構造になります。

### s3_v2 ログのキー階層

`config.local.yaml` で:

```yaml
s3_path: request_logs/
s3_use_team_prefix: true
s3_use_key_prefix: true
```

としているため、最終的なオブジェクトキーは:

```
s3://litellm-log-archive/request_logs/<team_alias>/<key_alias>/<YYYY-MM-DD>/<request_id>.json
```

の階層で書き込まれます。team / key alias が無いリクエスト（マスターキー直叩き）の場合、その階層は省略されます。

## 停止・クリーンアップ

```bash
# 停止のみ（データは保持）
docker compose stop

# 完全にクリーンアップ（コンテナ削除、ネットワーク削除）
docker compose down

# ボリュームも含めて全削除（DB・MinIO データも消える）
docker compose down -v
rm -rf database/postgresql/data storages/minio/data
```

## 参考

- [LiteLLM 公式ドキュメント - Logging](https://docs.litellm.ai/docs/proxy/logging)
- [LiteLLM 公式ドキュメント - StandardLoggingPayload](https://docs.litellm.ai/docs/proxy/logging_spec)
- [LiteLLM 公式ドキュメント - Bedrock](https://docs.litellm.ai/docs/providers/bedrock)

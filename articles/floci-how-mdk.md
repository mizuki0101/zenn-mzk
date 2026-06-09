---
title: "Terraform + lambroll で Floci 上に Lambda をデプロイしてみた"
emoji: "📝"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["terraform","floci" ]
published: true
---

# Terraform + lambroll で Floci 上に Lambda をデプロイしてみた

## はじめに

### Flociとは

[LocalStack](https://www.localstack.cloud/)のライセンス・ポリシー変更をきっかけに、代替となるローカルAWSエミュレーターが注目されています。  
その中でも[Floci](https://floci.io/)が気になったので試してみました。  

Flociは「軽くて速い、無料のAWSローカルエミュレーター」を謳うOSSです。特徴は次のとおりです。

- ポート4566ひとつで複数のAWSサービスをエミュレート（LocalStackと同じ作法）
- 認証トークン・サインアップ不要。Dockerイメージをpullすればすぐ動く
- Lambdaを「本物のDockerコンテナ」で実行する（モックではなく公式ランタイムイメージを使う）

この記事では、Flociに対してTerraformでインフラを作り、
Lambdaの関数本体は[lambroll](https://github.com/fujiwara/lambroll)でデプロイするという、
構成を試した記録をまとめます。

### この記事の対象読者

- LocalStackの代替を探していてFlociが気になっている人
- TerraformでAWSリソースを書いたことがある人
- Lambdaをローカルで動かして試したい人

### この記事でわかること

- TerraformからFlociに向けるためのprovider設定（カスタムエンドポイント・ダミー認証）の意味
- FlociでLambdaを動かすために必要なDocker設定とその理由
- Terraform（基本編）とlambroll（応用編）でLambdaをデプロイする方法
- 実際にハマったポイントと回避方法

### この記事で扱わないこと

- 本物のAWS環境への適用手順（同じコードで切り替え可能な設計にはしていますが、検証はローカルのみ）
- Flociの全サービスの網羅的な検証（S3 / DynamoDB / SQS / Lambdaに絞っています）

---

## Flociの基本的な使い方

Docker Composeから立ち上げます。

```yaml
services:
  floci:
    image: floci/floci:latest
    ports:
      - "4566:4566"
    environment:
      FLOCI_STORAGE_MODE: persistent
    volumes:
      - ./data:/app/data
```

```bash
docker compose up -d
```

起動したら、AWS CLIのエンドポイントをFlociに向けるだけで操作できます。

```bash
aws --endpoint-url http://localhost:4566 s3 ls
aws --endpoint-url http://localhost:4566 sqs list-queues
aws --endpoint-url http://localhost:4566 dynamodb list-tables
```

---

## Terraformでの使い方

Terraformからは「同じコードをFloci（dev）と本物のAWS（prod）で共有する」ことを目指し、`use_floci`変数ひとつで切り替えられるようにしました。

```hcl
provider "aws" {
  region = var.region

  # Floci 利用時のみダミー認証を渡す。false のときは標準の認証チェーンに委ねる。
  access_key = var.use_floci ? "test" : null
  secret_key = var.use_floci ? "test" : null

  s3_use_path_style           = var.use_floci
  skip_credentials_validation = var.use_floci
  skip_metadata_api_check     = var.use_floci
  skip_requesting_account_id  = var.use_floci

  # Floci 利用時のみカスタムエンドポイントを差し込む。
  dynamic "endpoints" {
    for_each = var.use_floci ? [1] : []
    content {
      s3       = var.floci_endpoint
      dynamodb = var.floci_endpoint
      sqs      = var.floci_endpoint
      lambda   = var.floci_endpoint
      iam      = var.floci_endpoint
      logs     = var.floci_endpoint
    }
  }
}
```

### カスタムエンドポイントとは

AWS SDK / Terraform AWS Providerは、APIリクエストを送る「宛先URL（エンドポイント）」を持っています。  
通常は`s3.ap-northeast-1.amazonaws.com`のようなAWS公式のホスト名に自動で向きます。
  
`endpoints`ブロックは、この宛先を自分で別のURLに上書きする設定です。  
ここでは各サービスの宛先をすべて`var.floci_endpoint`（既定`http://localhost:4566`）に差し替え、  
「AWSと話しているつもりで実際にはFlociにリソースを作る」状態にしています。

`use_floci = false`（本物のAWS）のときは`dynamic`ブロックが空になり、`endpoints`自体が生成されません。  
AWS既定のエンドポイントに委ねるため、本番では何も上書きしません。

### なぜFlociのときだけ各種フラグを立てるのか

|設定| Flociのときだけ有効にする理由|
|---|---|
| `s3_use_path_style` | `localhost`ではバケット名をサブドメインにする方式（`my-bucket.localhost`）が名前解決できないため、`localhost:4566/my-bucket/...`のパス方式にする|
| `access_key` / `secret_key`にダミー値| Flociは認証を検証しないが、providerは「認証情報がある」状態でないと動かないためダミーを渡す|
| `skip_credentials_validation` |起動時のSTS（`GetCallerIdentity`）検証を止める。Flociに本物のSTSは無い|
| `skip_metadata_api_check` | EC2インスタンスメタデータ（`169.254.169.254`）を見に行かせない。ローカルには無くタイムアウトするため|
| `skip_requesting_account_id` | IAM/STSからのアカウントID取得を止める|

大本の理由は「FlociはAWS互換APIを喋るだけで、AWSの周辺インフラ（DNS・STS・メタデータ・IAM）は持っていない」という一点に集約されます。

---

## TerraformでLambdaをSQSと繋ぐ（基本編）

まずは関数コードまで含めて、すべてTerraformで管理する構成です。

```hcl
data "archive_file" "lambda" {
  type        = "zip"
  source_dir  = "${path.module}/lambda"
  output_path = "${path.module}/build/lambda.zip"
}

resource "aws_lambda_function" "demo" {
  function_name    = "${var.name_prefix}-fn"
  runtime          = "python3.14"
  handler          = "handler.handler"
  role             = aws_iam_role.lambda.arn
  filename         = data.archive_file.lambda.output_path
  source_code_hash = data.archive_file.lambda.output_base64sha256
  timeout          = 30
}

# SQS にメッセージが入ると Lambda が起動する
resource "aws_lambda_event_source_mapping" "sqs_to_lambda" {
  event_source_arn = aws_sqs_queue.demo.arn
  function_name    = aws_lambda_function.demo.arn
  batch_size       = 10
}
```

### FlociでLambdaを動かすためのDocker設定

FlociのLambdaは「関数を内部で偽物っぽく動かす」のではなく、関数1つにつき本物のDockerコンテナを1つ立てて実行します。  
（`public.ecr.aws/lambda/python:3.14`などの公式ランタイムイメージを使う）  
そのため、Floci起動時に追加の設定が必要です。

```yaml
services:
  floci:
    image: floci/floci:latest
    user: root          # docker.sock を触れる root で動かす
    hostname: floci
    ports:
      - "4566:4566"
    environment:
      FLOCI_STORAGE_MODE: persistent
      FLOCI_HOSTNAME: floci                  # Lambda コンテナのコールバック先
      FLOCI_SERVICES_DOCKER_NETWORK: floci-net
    volumes:
      - ./data:/app/data
      - /var/run/docker.sock:/var/run/docker.sock   # ★Lambda 実行に必須
    networks:
      - floci

networks:
  floci:
    name: floci-net
```

ポイントは3つです。

- `docker.sock`のマウント: Floci自身もコンテナの中で動くため、Lambda実行用コンテナを「外側のホストDocker」に立ててもらう窓口が必要。
  - `.sock`はDockerデーモンとの通信窓口（＝Dockerを操作する権限そのもの）です。
- `user: root`: `docker.sock`はroot / dockerグループのみが触れるため。rootで動かしてパーミッション制限を越えます。
- `FLOCI_HOSTNAME`＋Dockerネットワーク:起動したLambdaコンテナは実行結果をFlociにコールバックします。
  - 既定の`localhost`だとLambdaコンテナ自身を指してしまうため、`http://floci:4566`のように同じネットワーク上で解決できる名前を教える必要があります。

> 「`.sock`のマウント」＝Dockerを操作する窓口を渡す、「`root`」＝その窓口を触る権限を与える、という補い合いの関係です。  
なお、これはホストのDockerを完全に支配できる強い設定なので、ローカル開発でのみ設定がよいかと思います。

### 動作確認

```bash
terraform apply -var-file=floci.tfvars

QUEUE_URL=$(terraform output -raw sqs_queue_url)
aws --endpoint-url http://localhost:4566 sqs send-message \
  --queue-url "$QUEUE_URL" --message-body "hello from sqs"

# ログ確認（後述の理由で logs tail ではなく filter-log-events を使う）
aws --endpoint-url http://localhost:4566 logs filter-log-events \
  --log-group-name /aws/lambda/floci-demo-fn \
  --query 'events[].message' --output text
```

ログに`received message: hello from sqs`が出れば、SQS → Lambdaの連携が動いています。

---

## lambrollを使って関数部分のデプロイ（応用編）

次にインフラはTerraform、関数本体はlambrollで試します。 
lambrollはAWS SDKベースなので、Terraformと同じくダミー認証＋エンドポイント差し替えでFlociに向きます。

```bash
cd terraform/lambroll

export AWS_ACCESS_KEY_ID=test
export AWS_SECRET_ACCESS_KEY=test
export AWS_REGION=us-east-1
export AWS_LAMBDA_ENDPOINT=http://localhost:4566

lambroll deploy
```

`function.json`の一例。  
`Role`にはTerraformが作成したロールARNを指定しています。

```json
{
  "FunctionName": "floci-demo-fn-lambroll",
  "Runtime": "python3.14",
  "Handler": "handler.handler",
  "Role": "arn:aws:iam::000000000000:role/floci-demo-lambda-role",
  "Timeout": 30,
  "MemorySize": 128
}
```

### lambrollで作成されたかを確認する

「関数はあるが、本当にlambrollが配ったコードなのか」を確かめる方法です。

```bash
# ① メタ情報（関数名は function.json 由来）
aws --endpoint-url http://localhost:4566 lambda get-function \
  --function-name floci-demo-fn-lambroll \
  --query 'Configuration.{Runtime:Runtime,Handler:Handler,LastModified:LastModified}'

# ② lambroll の視点: ローカル定義とデプロイ済みの差分（出力なし＝一致）
lambroll diff

# ③ 実行コードの確認
aws configure set cli_binary_format raw-in-base64-out
echo '{"Records":[{"body":"hi from lambroll"}]}' > /tmp/payload.json
aws --endpoint-url http://localhost:4566 lambda invoke \
  --function-name floci-demo-fn-lambroll --payload file:///tmp/payload.json /tmp/out.json
cat /tmp/out.json   # => {"processed": 1}

aws --endpoint-url http://localhost:4566 logs filter-log-events \
  --log-group-name /aws/lambda/floci-demo-fn-lambroll \
  --query 'events[].message' --output text
```

応用編のハンドラは接頭辞`[lambroll]`付きにしておいたので、  
ログに`[lambroll] received message: hi from lambroll`が出れば、lambrollでデプロイしたコードが動いていると確定できます。

---

## ハマったポイント

実際に試して踏んだ落とし穴と回避方法です。Floci固有のものが多く、ここが一番の知見でした。

### 1. `python3.14`がTerraformで弾かれる

AWS Providerが古いと`expected runtime to be one of [...]`というエラーで`python3.14`を受け付けません。  
`python3.14`はterraform-provider-aws 6.20.0以降で対応しています。providerのバージョン制約を`~> 6.20`に上げて解決しました。

### 2. `Failed to start Lambda container: java.net.BindException: Permission denied`

docker.sockのマウントとrootだけではLambdaが起動せず、このエラーが出ました。  
原因はLambdaコンテナのコールバック先で、既定の`localhost`だとLambdaコンテナ自身を指してしまいます。  
`FLOCI_HOSTNAME=floci`と専用Dockerネットワークを設定し、`http://floci:4566`でFlociに戻れるようにして解決しました。

### 3. `aws logs tail`が`'logStreamName'`だけ出して失敗する

`aws logs tail`は内部で`logStreamName`フィールドを参照しますが、FlociのCloudWatch Logsレスポンスにそれが含まれず`KeyError`になります。  
ログ自体は出ているので、`aws logs filter-log-events`（または`get-log-events`）で読めば問題ありません。

### 4. `lambda invoke`の出力先`/dev/stdout`でPermission denied

環境によっては`/dev/stdout`を出力先に指定すると`Permission denied`になります。`/tmp/out.json`のような実ファイルを指定すれば確実です。  
また、長いコマンドは貼り付け時に途中改行が入りやすいので、`AWS_ENDPOINT_URL`を`export`して`--endpoint-url`を省くと安定します。

---

## おわりに

他のエミュレーターをじっくり使用したことはないですが、配布されているDcokerを立ち上げるだけで直ぐに試せる環境が出来上がるので楽でした。  
Lambdaを使用する場合は追加設定が必要になるので、この記事を参考にしていただけたらと思います。

### 参考リンク

- [Floci公式サイト](https://floci.io/)
- [floci-io/floci (GitHub)](https://github.com/floci-io/floci)
- [fujiwara/lambroll (GitHub)](https://github.com/fujiwara/lambroll)
- [Terraform AWS Providerカスタムエンドポイント公式ガイド](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/guides/custom-service-endpoints)

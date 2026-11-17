# Serverless API（CloudFormation版）

`GET /users?userid=1` → API Gateway REST API → Lambda → DynamoDBという構成です。

## デプロイ

AWS CLIと対象アカウントの認証情報、およびCloudFormationと各リソースを作成する権限が必要です。このディレクトリで実行してください。

```bash
aws cloudformation validate-template \
  --template-body file://template.yaml \
  --region ap-northeast-1

aws cloudformation deploy \
  --template-file template.yaml \
  --stack-name user-api-serverless \
  --capabilities CAPABILITY_NAMED_IAM \
  --region ap-northeast-1
```

固定名のIAMロールを作成するため `CAPABILITY_NAMED_IAM` を指定します。CloudFormationコンソールからテンプレートをアップロードする方法でも作成できます。

ロググループ `/aws/lambda/get-user` はCloudFormationでLambdaより先に作成します。同名のロググループが既に存在する場合は作成が失敗するため、既存ログを残す場合はCloudFormationへのインポート、不要な場合は既存ロググループの削除を行ってからデプロイしてください。

## 動作確認

検証用レコードを登録してAPIを呼び出します。`id=1` が既にある場合は上書きせずエラーにします。

```bash
aws dynamodb put-item \
  --table-name users \
  --item '{"id":{"N":"1"},"username":{"S":"test-user"},"registered_date":{"S":"2026-11-09"}}' \
  --condition-expression 'attribute_not_exists(id)' \
  --region ap-northeast-1

API_ENDPOINT=$(aws cloudformation describe-stacks \
  --stack-name user-api-serverless \
  --query 'Stacks[0].Outputs[?OutputKey==`ApiEndpoint`].OutputValue | [0]' \
  --output text \
  --region ap-northeast-1)

curl -i "${API_ENDPOINT}?userid=1"
```

期待するレスポンスはHTTP 200と `{"username": "test-user", "registered_date": "2026-11-09"}` です。存在しないIDは404になります。Lambdaの入力処理・例外処理は元コードのままです（数値に変換できないIDは500、`queryStringParameters` がnullのイベントは未処理例外になります）。

## CloudWatchアラームの追加（AWS CLI）

障害を発生させる前に、API名 `user-api` の `prod` ステージで、1分間に5xxエラーが1件以上発生したら `ALARM` になるアラームを作成します。実行には `cloudwatch:PutMetricAlarm`、確認には `cloudwatch:DescribeAlarms` と `cloudwatch:DescribeAlarmHistory`、削除には `cloudwatch:DeleteAlarms` の権限が必要です。

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name user-api-prod-5xx \
  --alarm-description 'Detect one or more API Gateway 5xx errors in one minute' \
  --namespace AWS/ApiGateway \
  --metric-name 5XXError \
  --dimensions Name=ApiName,Value=user-api Name=Stage,Value=prod \
  --statistic Sum \
  --period 60 \
  --evaluation-periods 1 \
  --datapoints-to-alarm 1 \
  --threshold 1 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --treat-missing-data notBreaching \
  --region ap-northeast-1
```

REST APIの標準メトリクス `5XXError` を利用するため、ログのメトリクスフィルターや詳細メトリクスの有効化は不要です。対象はステージ全体で、4xxエラーは検知対象に含みません。同じリージョンに同名API・同名ステージがある場合はメトリクスが集約されるため、API名を一意にしてください。[AWS公式のメトリクス仕様](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-metrics-and-dimensions.html)

データがない期間は `notBreaching`（しきい値超過なし）として扱います。作成直後は `INSUFFICIENT_DATA` になる場合があります。この手順ではアラームの状態変化による検知のみを設定し、メール通知は設定しません。通知も必要な場合は、購読設定済みのSNSトピックARNを `--alarm-actions` に指定します。[AWS CLIの仕様](https://docs.aws.amazon.com/cli/latest/reference/cloudwatch/put-metric-alarm.html)

## 障害の発生と調査

### 1. 事前確認とポリシーの保存

上の「動作確認」でHTTP 200になることを確認してから進めます。

```bash
aws sts get-caller-identity

aws iam get-role-policy \
  --role-name get-user-lambda-role \
  --policy-name dynamodb-read \
  --query PolicyDocument \
  --output json > dynamodb-read-backup.json

cat dynamodb-read-backup.json
```

対象アカウントが正しく、保存したJSONに `dynamodb:GetItem` と `users` テーブルのARNが含まれていることを確認します。

### 2. AWS CLIでDynamoDB読み取りポリシーを削除する

`template.yaml` では、`get-user-lambda-role` にインラインポリシー `dynamodb-read` を設定しています。インラインポリシーなので、管理ポリシーのデタッチではなく `delete-role-policy` で削除します。

```bash
aws iam delete-role-policy \
  --role-name get-user-lambda-role \
  --policy-name dynamodb-read
```

上記のAWS CLIコマンドで `dynamodb-read` のみを削除します。ログ出力用の `AWSLambdaBasicExecutionRole` は残します。

### 3. APIのエラーを確認して調査する

```bash
API_ENDPOINT=$(aws cloudformation describe-stacks \
  --stack-name user-api-serverless \
  --query 'Stacks[0].Outputs[?OutputKey==`ApiEndpoint`].OutputValue | [0]' \
  --output text \
  --region ap-northeast-1)

curl -i "${API_ENDPOINT}?userid=1"
```

IAMの変更が反映されるまで時間がかかる場合があります。HTTP 200のままであれば、少し待って再度呼び出します。期待する結果はHTTP 500と `{"error": "Internal server error"}` です。

HTTP 500を確認したら、CloudWatchアラームの状態を確認します。メトリクスの送信とアラーム評価には時間がかかるため、数分待って確認してください。

```bash
aws cloudwatch describe-alarms \
  --alarm-names user-api-prod-5xx \
  --query 'MetricAlarms[].{Name:AlarmName,State:StateValue,Reason:StateReason}' \
  --output table \
  --region ap-northeast-1

aws cloudwatch describe-alarm-history \
  --alarm-name user-api-prod-5xx \
  --history-item-type StateUpdate \
  --max-records 10 \
  --region ap-northeast-1
```

期待する状態は `ALARM` です。エラーリクエストが止まると、その後の評価で `OK` に戻る場合があるため、現在の状態に加えて履歴の `ALARM` への遷移も確認します。`OK` は権限が復旧したことを意味しません。今回はLambdaが例外を捕捉してHTTP 500を返すため、APIの5xxを直接監視します。

[エージェントスペースの作成手順](../README.md)で用意したWebアプリから調査を開始します。調査対象のリージョン `ap-northeast-1`、API名 `user-api`、Lambda名 `get-user`、HTTP 500が発生した時刻（タイムゾーン付き）を伝えます。

## 更新と削除

AWS CLIで作成したアラームはCloudFormationの管理対象外です。スタック削除だけでは残るため、不要になったら別途削除します。

```bash
aws cloudwatch delete-alarms \
  --alarm-names user-api-prod-5xx \
  --region ap-northeast-1
```

Lambdaのコード更新は同じ `deploy` コマンドで反映できます。APIのメソッド・パス・統合などを変更する場合は、`ApiDeploymentV1` の論理IDを `ApiDeploymentV2` などに変更し、`ProdStage.DeploymentId` の参照も更新してからデプロイしてください。API Gatewayのデプロイはスナップショットであり、メソッドの更新だけでは既存ステージに反映されません。

```bash
aws cloudformation delete-stack \
  --stack-name user-api-serverless \
  --region ap-northeast-1

aws cloudformation wait stack-delete-complete \
  --stack-name user-api-serverless \
  --region ap-northeast-1
```

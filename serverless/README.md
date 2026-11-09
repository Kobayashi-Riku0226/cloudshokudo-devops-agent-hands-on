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

## 更新と削除

`LambdaLogGroup` に `DeletionPolicy: Delete` と `UpdateReplacePolicy: Delete` を指定しています。スタック削除時はLambdaの削除後にロググループと保存済みログも削除されます。スタック稼働中のログ保持期間は無期限です。Lambda名を変更する場合はロググループ名も `/aws/lambda/<新しい関数名>` に合わせて変更してください。

Lambdaのコード更新は同じ `deploy` コマンドで反映できます。APIのメソッド・パス・統合などを変更する場合は、`ApiDeploymentV1` の論理IDを `ApiDeploymentV2` などに変更し、`ProdStage.DeploymentId` の参照も更新してからデプロイしてください。API Gatewayのデプロイはスナップショットであり、メソッドの更新だけでは既存ステージに反映されません。

```bash
aws cloudformation delete-stack \
  --stack-name user-api-serverless \
  --region ap-northeast-1

aws cloudformation wait stack-delete-complete \
  --stack-name user-api-serverless \
  --region ap-northeast-1
```

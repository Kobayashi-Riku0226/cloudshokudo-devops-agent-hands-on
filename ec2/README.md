# Webサーバー on EC2（CloudFormation版）

ブラウザ → EC2（Amazon Linux 2023 + nginx）という構成です。OS内の設定ミスでWebサーバーを止め、DevOps Agentに原因を調べてもらうシナリオで使います。

- 1分ごとに `curl http://localhost/` でヘルスチェックし、結果をカスタムメトリクス `Shokudo/WebApp HttpCheckSuccess` に送ります。失敗するとアラーム `<スタック名>-web-http-check-failed` がALARMになります
- CloudWatch Agentで、nginxのログ・`/var/log/messages`・ヘルスチェックのログをロググループ `/cloud-shokudo/<スタック名>/web` に送ります
- SSHは開けていません。インスタンスの操作はSSM Run Commandで行います
- 起動から180分でインスタンスが自動でシャットダウンし、終了します（パラメータ `AutoShutdownMinutes`、0で無効）

## デプロイ

AWS CloudShell（東京リージョン）での実行を想定しています。リポジトリを取得して、このディレクトリで実行してください。

```bash
git clone https://github.com/Kobayashi-Riku0226/cloudshokudo-devops-agent-hands-on.git
cd cloudshokudo-devops-agent-hands-on/ec2
```

```bash
aws cloudformation validate-template \
  --template-body file://template.yaml \
  --region ap-northeast-1

aws cloudformation deploy \
  --template-file template.yaml \
  --stack-name web-server-ec2 \
  --capabilities CAPABILITY_IAM \
  --region ap-northeast-1
```

スタックの作成は3分ほどで終わります。その後、UserDataでnginxとCloudWatch Agentのセットアップが走るため、アラームがOKになるまでさらに2分ほど待ってください。

Windowsのローカル環境のAWS CLIで実行すると、テンプレートの読み込みで `'cp932' codec can't decode` になることがあります。その場合は環境変数 `AWS_CLI_FILE_ENCODING=UTF-8` を設定してください（CloudShellでは不要です）。

## 動作確認

以降のコマンドで使う値をスタックの出力から取得します。

```bash
STACK_NAME=web-server-ec2

INSTANCE_ID=$(aws cloudformation describe-stacks \
  --stack-name ${STACK_NAME} \
  --query 'Stacks[0].Outputs[?OutputKey==`InstanceId`].OutputValue | [0]' \
  --output text \
  --region ap-northeast-1)

WEB_URL=$(aws cloudformation describe-stacks \
  --stack-name ${STACK_NAME} \
  --query 'Stacks[0].Outputs[?OutputKey==`WebUrl`].OutputValue | [0]' \
  --output text \
  --region ap-northeast-1)

echo ${INSTANCE_ID} ${WEB_URL}
```

Webページ・アラーム・SSMの状態を確認します。

```bash
curl -s -o /dev/null -w '%{http_code}\n' ${WEB_URL}

aws cloudwatch describe-alarms \
  --alarm-names ${STACK_NAME}-web-http-check-failed \
  --query 'MetricAlarms[0].StateValue' \
  --output text \
  --region ap-northeast-1

aws ssm describe-instance-information \
  --filters Key=InstanceIds,Values=${INSTANCE_ID} \
  --query 'InstanceInformationList[0].PingStatus' \
  --output text \
  --region ap-northeast-1
```

期待する結果は `200`、`OK`、`Online` です。アラームは起動直後は `INSUFFICIENT_DATA` なので、数分待ってください。`WEB_URL` をブラウザで開くと「Welcome to Cloud Shokudo #10」が表示されます（httpsではなくhttpで開いてください）。

## 障害を起こす

アップロード上限を上げる設定を追加しようとして、セミコロンを付け忘れたという想定です。nginxの再起動に失敗してWebサーバーが止まります。

```bash
aws ssm send-command \
  --instance-ids ${INSTANCE_ID} \
  --document-name AWS-RunShellScript \
  --comment "Add upload size limit" \
  --parameters '{"commands":["echo client_max_body_size 50m > /etc/nginx/conf.d/upload.conf","systemctl restart nginx"]}' \
  --region ap-northeast-1
```

nginxの再起動が失敗するため、コマンドのステータスは `Failed`（`Job for nginx.service failed ...`）になります。すぐに `curl` が `000` になり、2〜3分でアラームがALARMになります。

```bash
curl -s -o /dev/null -w '%{http_code}\n' --max-time 5 ${WEB_URL}

aws cloudwatch describe-alarms \
  --alarm-names ${STACK_NAME}-web-http-check-failed \
  --query 'MetricAlarms[0].StateValue' \
  --output text \
  --region ap-northeast-1
```

## DevOps Agentで調査する

1. AWS DevOps AgentコンソールのAgent Space画面で「IAM 経由で起動する」を押し、Web Appを開きます（Web Appは最初チャット画面が開きます。チャットにも課金されるので、ここでは入力しません）
2. 左メニューの「インシデントレスポンス」→「調査を開始」→「最新のアラーム」を選び、「調査を開始」を押します

通しで試したときは、約2分で根本原因（`/etc/nginx/conf.d/upload.conf:2` の構文エラー）を特定し、約3分半で緩和計画まで出ました。SSMのコマンド履歴から、障害を起こした `SendCommand` のCommandIdと実行したIAMプリンシパルまで特定していました。

答え合わせ用に、エージェントが参照するログを直接見ることもできます。

```bash
aws logs tail /cloud-shokudo/${STACK_NAME}/web \
  --log-stream-name-prefix ${INSTANCE_ID}/nginx-error \
  --since 10m \
  --region ap-northeast-1
```

`[emerg] unexpected end of file, expecting ";" or "}" in /etc/nginx/conf.d/upload.conf:2` が出ていれば想定どおりです。

## 復旧

DevOps Agentの緩和計画に出ている `aws ssm send-command` を、上から順にCloudShellで実行します（設定ファイルを退避してから無効化し、`nginx -t` を通してからnginxを起動する手順です）。

緩和計画を使わずに直す場合は、セミコロンを付けて再起動します。

```bash
aws ssm send-command \
  --instance-ids ${INSTANCE_ID} \
  --document-name AWS-RunShellScript \
  --comment "Fix upload size limit" \
  --parameters '{"commands":["echo \"client_max_body_size 50m;\" > /etc/nginx/conf.d/upload.conf","nginx -t","systemctl restart nginx"]}' \
  --region ap-northeast-1
```

`curl` が `200` に戻り、2分ほどでアラームがOKに戻ります。

## 削除

VPC・EC2・IAMロール・ロググループ・アラームはすべてスタックと一緒に削除されます。

```bash
aws cloudformation delete-stack \
  --stack-name web-server-ec2 \
  --region ap-northeast-1

aws cloudformation wait stack-delete-complete \
  --stack-name web-server-ec2 \
  --region ap-northeast-1
```

カスタムメトリクス（`Shokudo/WebApp`、`CWAgent`）は手動では削除できませんが、データが届かなくなれば課金されず、15か月後に自動で消えます。

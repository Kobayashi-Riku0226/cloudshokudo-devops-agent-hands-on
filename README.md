# cloudshokudo-devops-agent-hands-on

## リポジトリの取得と作業ディレクトリへの移動

AWS CloudShellまたはGitが利用できるターミナルで、リポジトリをクローンし、ルートディレクトリへ移動します。

```bash
git clone https://github.com/Kobayashi-Riku0226/cloudshokudo-devops-agent-hands-on.git
cd cloudshokudo-devops-agent-hands-on
```

取得済みの場合はクローンを省略し、既存のリポジトリのルートディレクトリを開いてください。続いて、実施する構成のディレクトリへ移動します。以下の `cd` コマンドは、それぞれリポジトリのルートから実行してください。

### サーバーレス構成

```bash
cd serverless
```

移動後は、[serverless/README.md](serverless/README.md)に従って、デプロイ、動作確認、IAMポリシーの削除による障害再現、復旧を進めてください。各AWS CLIコマンドは `serverless` ディレクトリで実行します。

### EC2構成

```bash
cd ec2
```

移動後は、[ec2/README.md](ec2/README.md)に従って、デプロイ、動作確認、nginxの設定ミスによる障害再現、DevOps Agentでの調査、復旧を進めてください。すでにリポジトリを取得して移動済みの場合は、EC2側のREADMEにある `git clone` とディレクトリ移動のコマンドを省略し、デプロイのコマンドから実行します。各AWS CLIコマンドは `ec2` ディレクトリで実行します。

`serverless` から続けてEC2構成へ移動する場合は、`cd ../ec2` を実行してください。

AWS DevOps Agentで調査するためのエージェントスペースは、以下の手順で作成します。

## AWS DevOps Agentのエージェントスペース作成

エージェントスペース（Agent Space）は、AWS DevOps Agentが調査できるAWSリソースや外部ツール、利用者のアクセス範囲を設定する単位です。ここでは、AWSマネジメントコンソールでハンズオン用のスペースを作成し、Webアプリを開くまでを説明します。

### 前提条件

- ハンズオンに使用するAWSアカウントにサインインできること。
- AWS DevOps Agentの設定と、IAMロールの作成・ポリシーの設定に必要な権限があること。以下ではIAMロールの自動作成を使用します。
- リージョンは東京（`ap-northeast-1`）を使用します。利用可能なリージョンは[公式ドキュメント](https://docs.aws.amazon.com/devopsagent/latest/userguide/about-aws-devops-agent-supported-regions.html)で確認できます。

この手順では、調査対象のリソースと同じAWSアカウントにスペースを作成します。このアカウントがプライマリアカウントになります。サンプルAPIの構築手順は[serverless/README.md](serverless/README.md)を参照してください。

### 1. 作成画面を開く

1. AWSマネジメントコンソールで **AWS DevOps Agent** を検索して開きます。
2. リージョンを東京（`ap-northeast-1`）に切り替えます。
3. **Create Agent Space** を選択します。

### 2. 基本情報を入力する

**Agent Space details** で次の値を設定します。名前と説明は任意に変更できます。

| 項目 | 設定例 |
| --- | --- |
| Name | `cloudshokudo-devops-agent-hands-on` |
| Description（任意） | `クラウド食堂ハンズオン用の調査環境` |
| Agent response language（任意） | `Japanese (Japan)` |

### 3. AWSリソースへのアクセスを設定する

1. **Give this Agent Space AWS resource access** で、IAMロールを自動作成する選択肢を選びます。
2. 作成されるロール名と付与される権限を確認します。ロール名は必要に応じて変更できます。

このロールは、エージェントがプライマリアカウント内のリソースを検出・調査するために使用します。アクセス範囲を絞る場合は、ポリシーテンプレートを使ってロールを作成する方法もあります。既存ロールを使用する場合は、AWS DevOps Agent用の権限と信頼ポリシーが設定されたものを選んでください。

### 4. Webアプリへのアクセスを設定する

1. Agent Space Web Appの設定で、Webアプリを有効にします。
2. Webアプリへのアクセス用IAMロールも、自動作成する選択肢を選びます。
3. 付与される権限を確認します。

こちらは利用者がWebアプリにアクセスするためのロールです。手順3のAWSリソース調査用ロールとは用途が異なります。

### 5. 作成して動作を確認する

1. 必要に応じてタグを設定し、入力内容を確認して **Create** を選択します。
2. 作成したスペースの詳細画面で **Operator access** を選択します。
3. 新しいタブでWebアプリが開き、認証されることを確認します。Webアプリでは調査の実行や推奨事項の確認ができます。

作成時に権限エラーが出る場合は、操作中のIAMユーザーまたはロールがIAMロールを作成・設定できるか確認してください。Webアプリへの接続に失敗する場合は、手順4の設定とアクセス用ロールの権限を確認します。

### 参考資料

- [エージェントスペースの作成（AWS公式）](https://docs.aws.amazon.com/devopsagent/latest/userguide/getting-started-with-aws-devops-agent-creating-an-agent-space.html)
- [エージェントスペースの概要（AWS公式）](https://docs.aws.amazon.com/devopsagent/latest/userguide/about-aws-devops-agent-what-are-devops-agent-spaces.html)

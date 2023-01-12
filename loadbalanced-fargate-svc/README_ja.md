# AWS Proton Sample Load-Balanced Web Service Using Amazon ECS and AWS Fargate

このディレクトリには、AWS Fargate 上で動作する Amazon ECS ロードバランスサービス用の AWS Proton 「環境テンプレート」、および 「サービステンプレート」 のサンプルと、テンプレートを使用して Proton 環境およびサービスを作成するためのスペックファイルのサンプルが含まれています。

「環境テンプレート」には、ECS クラスタと、2 つのパブリックサブネットを持つ VPC が含まれています。

「サービステンプレート」には、その環境のロードバランサーの背後に ECS Fargate サービスを作成するために必要なすべてのリソースと、テンプレートを使用して Proton 環境とサービスを作成するためのサンプルスペックが含まれています。

サービスをプロビジョニングする開発者は、そのサービス仕様を通じて以下のプロパティを設定できます。

- Fargate の CPU サイズ
- Fargate のメモリサイズ
- 実行するのコンテナ数

もしもサービス内で実行するアプリケーションコードが必要な場合、以下にサンプルアプリケーションがあります:
https://github.com/yuhei-otsuka/aws-proton-sample-services

※ 訳注: このアプリケーションサンプルの場所は、以前は https://github.com/aws-samples/aws-proton-sample-fargate-service でした。

```
# ※ 訳注: ディレクトリ内容のまとめ
loadbalanced-fargate-env/ ... 環境テンプレート
loadbalanced-fargate-svc/ ... サービステンプレート
policies/                 ... IAM ロール作成時に参照するポリシーファイル
specs/                    ... 環境とサービスを作成する際のスペック(変数ファイル)サンプル
```

# Registering and deploying these templates

これらのテンプレートは、AWS Proton コンソールを使用して、登録およびデプロイすることができます。

そのためには、以下の手順でテンプレートを圧縮し、S3 バケットにアップロードし、Proton コンソールを使って登録とテストを行う必要があります。

Command Line Interface を使用したい場合は、以下の手順に従ってください。

## Prerequisites

まず、AWS CLI がインストールされ、設定されていることを確認します。

以下のコマンドを実行し、AWS リージョンと環境変数`account_id` を設定します。

```bash
export AWS_DEFAULT_REGION=us-west-2
account_id=`aws sts get-caller-identity --query Account --output text`

# 確認
echo $account_id
```

※ 訳注: 原文通り us-west-2 (オレゴン) を用います。AWS Proton をサポートするリージョンの情報は [AWS Proton のよくある質問](https://aws.amazon.com/jp/proton/faqs/) にあります。

## Configure IAM Role, S3 Bucket, and CodeStar Connections Connection

テンプレートを登録し、環境とサービスをデプロイする前に、

- AWS Proton が AWS アカウントのリソースを管理できるように Amazon IAM ロール
- テンプレートを保存する Amazon S3 バケット
- アプリケーションコードをプルしてデプロイする CodeStar Connections 接続
  を作成する必要があります。

テンプレートを格納するための S3 バケットを作成します。

```bash
aws s3api create-bucket \
  --bucket "proton-cli-templates-${account_id}" \
  --create-bucket-configuration LocationConstraint=us-west-2

# 確認
aws s3 ls "proton-cli-templates-${account_id}"
```

Proton がリソースのプロビジョニングや AWS CloudFormation スタックの管理を行うために想定する IAM ロールを AWS アカウントに作成します。

```bash
aws iam create-role \
  --role-name ProtonServiceRole \
  --assume-role-policy-document file://./policies/proton-service-assume-policy.json

aws iam attach-role-policy \
  --role-name ProtonServiceRole \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess
```

そして、Proton がそのロールを使って、サービスの継続的デリバリー・パイプラインのリソースをプロビジョニングできるようにします。

```bash
aws proton update-account-settings \
  --pipeline-service-role-arn "arn:aws:iam::${account_id}:role/ProtonServiceRole"
```

GitHub や Bitbucket のソースコードリポジトリに保存されているアプリケーションコードへの AWS CodeStar Connections の接続を作成します。

この接続により、コードをビルドして Proton サービスにデプロイする前に、CodePipeline がアプリケーションのソースコードをプルすることができます。

サンプルアプリケーションのコードを使用するには、まずここでサンプルアプリケーションリポジトリのフォークを作成します。https://github.com/aws-samples/aws-proton-sample-services

※ 訳注: 前述のサンプルアプリケーションリポジトリであり、名前が変わっています。

ソースコード接続の作成は、CodeStar Connections コンソールで完了する必要があります。https://us-west-2.console.aws.amazon.com/codesuite/settings/connections?region=us-west-2

# Register An Environment Template

ECS クラスタと 2 つのパブリックサブネットを持つ VPC を含む、サンプル環境テンプレートを登録します。

まず、「環境テンプレート」を作成し、「環境テンプレート」のすべてのバージョンを格納します。

```bash
aws proton create-environment-template \
  --name "public-vpc" \
  --display-name "PublicVPC" \
  --description "VPC with Public Access and ECS Cluster"
```

ここで、サンプルの「環境テンプレート」の内容を含むバージョンを作成します。

サンプルのテンプレートファイルを圧縮して、バージョンを登録します。

```bash
tar -zcvf env-template.tar.gz loadbalanced-fargate-env/

aws s3 cp env-template.tar.gz s3://proton-cli-templates-${account_id}/env-template.tar.gz

rm env-template.tar.gz

aws proton create-environment-template-version \
  --template-name "public-vpc" \
  --description "Version 1" \
  --source s3="{bucket=proton-cli-templates-${account_id},key=env-template.tar.gz}"
```

※ 訳注: 原文では `environment/` を圧縮するコマンドになっていましたが、実体に合わせてコマンドを変更しています。

「環境テンプレート」のバージョンが正常に登録されるのを待ちます。

```bash
aws proton wait environment-template-version-registered \
  --template-name "public-vpc" \
  --major-version "1" \
  --minor-version "0"

aws proton get-environment-template-version \
  --template-name "public-vpc" \
  --major-version "1" \
  --minor-version "0"
```

これで「環境テンプレート」バージョンを公開し、AWS アカウントのユーザーが Proton 環境を作成できるようになります。

```bash
aws proton update-environment-template-version \
  --template-name "public-vpc" \
  --major-version "1" \
  --minor-version "0" \
  --status "PUBLISHED"
```

※ 訳注: このコマンドで status を `DRAFT` から `PUBLISHED` に変更しています。

## Register A Service Template

ロードバランサーの背後にある ECS Fargate サービスのプロビジョニングに必要なすべてのリソースと、AWS CodePipeline を使用した継続的デリバリーパイプラインを含む、サンプルサービステンプレートを登録します。

まず、「サービステンプレート」を作成します。

```bash
aws proton create-service-template \
  --name "lb-fargate-service" \
  --display-name "LoadbalancedFargateService" \
  --description "Fargate Service with an Application Load Balancer"
```

ここで、サンプルの「サービステンプレート」の内容を含むバージョンを作成します。

サンプルのテンプレートファイルを圧縮して、バージョンを登録します。

```bash
tar -zcvf svc-template.tar.gz loadbalanced-fargate-svc/

aws s3 cp svc-template.tar.gz s3://proton-cli-templates-${account_id}/svc-template.tar.gz --region us-west-2

rm svc-template.tar.gz

aws proton create-service-template-version \
  --template-name "lb-fargate-service" \
  --description "Version 1" \
  --source s3="{bucket=proton-cli-templates-${account_id},key=svc-template.tar.gz}" \
  --compatible-environment-templates '[{"templateName":"public-vpc","majorVersion":"1"}]'
```

※ 訳注: 原文では `service/` を圧縮するコマンドになっていましたが、実体に合わせてコマンドを変更しています。

「サービステンプレート」が正常に登録されるのを待ちます。

```bash
aws proton wait service-template-version-registered \
  --template-name "lb-fargate-service" \
  --major-version "1" \
  --minor-version "0"

aws proton get-service-template-version \
  --template-name "lb-fargate-service" \
  --major-version "1" \
  --minor-version "0"
```

これで、「サービステンプレート」バージョンを公開し、AWS アカウントのユーザーが Proton サービスを作成できるようになります。

```bash
aws proton update-service-template-version \
  --template-name "lb-fargate-service" \
  --major-version "1" \
  --minor-version "0" \
  --status "PUBLISHED"
```

※ 訳注: このコマンドで status を `DRAFT` から `PUBLISHED` に変更しています。

## Deploy An Environment

登録・公開された環境テンプレートを使って、テンプレートから Proton 環境をインスタンス化できるようになりました。

環境作成時には、2 種類の環境プロビジョニング方法を利用することができます。

- (A) 単一のアカウントで、環境を作成、管理、プロビジョニングする。

- (B) 単一の管理アカウントで、環境アカウント接続で別のアカウントでプロビジョニングされた環境を作成し、管理する。詳しくは、[Create an environment in one account and provision in another account](https://docs.aws.amazon.com/proton/latest/adminguide/ag-create-env.html#ag-create-env-deploy-other) と [Environment account connections](https://docs.aws.amazon.com/proton/latest/adminguide/ag-env-account-connections.html) を参照してください。

※ 訳注: (A)(B) をアルファベットを振りました。

### (A) Create and Provision Environment in a single account

まず、Proton 環境をデプロイします。

このコマンドは、`specs/env-spec.yaml` にある環境スペックを読み込み、上で作成した「環境テンプレート」とマージし、Proton サービスロールを使用して AWS アカウントの CloudFormation スタックにリソースをデプロイします。

```bash
aws proton create-environment \
  --name "Beta" \
  --template-name public-vpc \
  --template-major-version 1 \
  --proton-service-role-arn arn:aws:iam::${account_id}:role/ProtonServiceRole \
  --spec file://specs/env-spec.yaml
```

環境が正常にデプロイされるのを待ちます。

```bash
aws proton wait environment-deployed --name Beta

aws proton get-environment --name Beta
```

### (B) Create Environment in one account and Provision in another account

まず、環境リソースをプロビジョニングしたい環境アカウントにログインし、AWS アカウントでリソースのプロビジョニングと AWS CloudFormation スタックの管理を行うために Proton が想定する IAM ロールを作成します。

これはコンソールから行うこともできます。https://docs.aws.amazon.com/proton/latest/adminguide/security_iam_service-role-policy-examples.html#proton-svc-role

```bash
environment_account_id=`your_environment_account_id`

aws iam create-role \
  --role-name ProtonServiceRole \
  --assume-role-policy-document file://./policies/proton-service-assume-policy.json

aws iam attach-role-policy \
  --role-name ProtonServiceRole \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess
```

その後、環境アカウント接続リクエストを作成し、管理アカウントに送信します。

リクエストが受理されると、AWS Proton は関連する環境アカウントで環境リソースのプロビジョニングを許可する関連 IAM ロールを使用することができます。

その際、環境に使用する環境名を指定する必要があります。

```bash
aws proton create-environment-account-connection \
  --management-account-id ${account_id} \
  --environment-name "Beta" \
  --role-arn arn:aws:iam::${environment_account_id}:role/ProtonServiceRole

environment_account_connection_id=`replace_with_the_environment_account_connection_id_returned_above`
```

管理アカウントにログインし、環境アカウントからの接続要求を承諾します。これはコンソールからも可能です。

```bash
aws proton accept-environment-account-connection --id ${environment_account_connection_id}
```

続いて、Proton 環境を作成します。

このコマンドは `specs/env-spec.yaml` にある環境スペックを読み込み、上で作成した「環境テンプレート」とマージし、環境アカウントの接続にアタッチされた Proton サービスロールを使用して、環境の AWS アカウントに CloudFormation スタックでリソースをデプロイします。

```bash
aws proton create-environment \
  --name "Beta" \
  --template-name public-vpc \
  --template-major-version 1 \
  --environment-account-connection-id ${environment_account_connection_id} \
  --spec file://specs/env-spec.yaml
```

環境が正常にデプロイされるのを待ちます。デプロイの状況を確認するには `get` 呼び出しを使用します。

```bash
aws proton wait environment-deployed --name Beta

aws proton get-environment --name Beta
```

## Deploy A Service

登録・公開した「サービステンプレート」とデプロイした環境があれば、あとは Proton サービスを作成し、Proton 環境にデプロイするだけです。

このコマンドは `specs/svc-spec.yaml` にあるサービススペックを読み込んで、上で作成した「サービステンプレート」とマージし、環境の AWS アカウントに CloudFormation スタックでリソースをデプロイします。

このサービスでは、Lambda ベースの CRUD API エンドポイントと、アプリケーションコードをデプロイするための CodePipeline パイプラインがプロビジョニングされます。

このコマンドに CodeStar Connections の接続 ID とソースコードリポジトリの詳細を記入します。

```bash
aws proton create-service \
  --name "front-end" \
  --repository-connection-arn arn:aws:codestar-connections:us-west-2:${account_id}:connection/<your-codestar-connection-id> \
  --repository-id "<your-source-repo-account>/<your-repository-name>" \
  --branch "main" \
  --template-major-version 1 \
  --template-name lb-fargate-service \
  --spec file://specs/svc-spec.yaml
```

サービスが正常にデプロイされるのを待ちます。

```bash
aws proton wait service-created --name front-end

aws proton get-service --name front-end
```

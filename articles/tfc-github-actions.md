---
title: "TerraformCloudとgithub actionsを連携する"
emoji: "😸"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["terraform", "AWS"]
published: false
---

# はじめに

社内で初めてterraformを導入することなりました。terraformの開発経験のあるエンジニアが
誰も居らず、試行錯誤しながらインフラのコード化を進めている状態です。
初めは、S3をbackendに指定して開発を進めていたのですが、TerraformCloudの以下の点に
魅力を感じ、TerraformCloudの導入を進めることになりました。

### 1. 監査ログが残る

![](https://storage.googleapis.com/zenn-user-upload/2cfc28fd7918a37eddf8111e.png)
実行環境をTerraformCloudに移管すると、誰が・いつ・何のコマンドを実行したのかが
ログとして残ります。インフラの開発はセキュリティ面も考慮する必要があると思うので、
この様に監査ログが残るのは大変ありがたいです。

### 2. applyを承認制に出来る

![](https://storage.googleapis.com/zenn-user-upload/27f5a77c61a508eed8451c5c.png)
TerraformCloudの設定でApplyMethodを`Manual apply`にして`terraform apply`を実行すると、
こんな感じで承認フローを設定出来ます。例えば本番環境へのデプロイは承認が必須というような
ワークフローを簡単に構築出来ます。

### 3. AWSのアクセスキーの管理が楽

TerraformCloudを利用する前は、ローカルとGithubと２つの環境でAWSのアクセスキーを
持っていたのですが、実行環境がTerraformCloudに統一されたので、TerraformCloudだけで
アクセスキーを管理するだけで良くなりました。

# 環境

terraform 1.0.1

# 1. Terraform Cloudの設定

### workspaceを作成

まずworkspaceを作成する所から始めます。
![](https://storage.googleapis.com/zenn-user-upload/b55135d3bb45cbfc79dcea27.png)
やりたいことはローカルとgithub actionsからterraformの各種コマンドを実行することなので、
`API-driven workflow` を選択します。

```
.
├── README.md
├── environments
│   |── staging
│   |   ├── main.tf
│   |   ├── outputs.tf
│   |   └── variables.tf
|   |── production
|   |   |
|   |── dev
|   |   |
├── modules
│   |── network
│   |   ├── main.tf
│   |   ├── outputs.tf
│   |   └── variables.tf
|   |
```
現在ディレクトリ構造はこんな感じで各環境ごとにmain.tf置いて、開発を進めています。
各環境ごとにworkspaceが必要になるのですが、ここではステージング環境のworkspaceを作る
という前提で解説を進めて行きます。

### terraformのversionと作業ディレクトリを指定する

次に「Settings」タブ->「General」で、
![](https://storage.googleapis.com/zenn-user-upload/617582582c5f312f0ca17f4c.png)
の様にterraformのバージョンと作業ディレクトリを指定します。
今はステージング用のworkspaceを作っているので、`environments/staging`と指定します。

### AWSアクセスキーを設定する

「Variables」タブから環境変数を設定出来ます。
![](https://storage.googleapis.com/zenn-user-upload/7d19592bdb2c45dd0c202393.png)
TerraformCloud上でterraformを実行するために、`AWS_ACCESS_KEY_ID`と`AWS_SECRET_ACCESS_KEY` を設定します。

### API TOKENを発行する

ローカルとgithub actionsからterraformを実行するために、トークンを取得します。

```
$ terraform login
Terraform will request an API token for app.terraform.io using your browser.

If login is successful, Terraform will store the token in plain text in
the following file for use by subsequent commands:
    /Users/machamp/.terraform.d/credentials.tfrc.json

Do you want to proceed?
  Only 'yes' will be accepted to confirm.
```
yesと入力すると、トークンが発行されます。
これをgithubで「Settings」->「Secrets」に移動して登録します。
![](https://storage.googleapis.com/zenn-user-upload/9279d0e2b025815871a1713d.png)

これでTerraformCloudでやることは全て完了です！

# 2. TerraformCloudをbackendとして指定する

```HCL
terraform {
  backend "remote" {
    organization = "organization名"

    workspaces {
      name = "workspace名"
    }
  }
}
```
このコードをmain.tfに記述します。

# 3. Github Actionsを設定する

PRを作成した時に実行されるworkflowを作成します。実行内容は以下の通りです。

1. terraform fmt -check -recursive
2. terraform init
3. tflint
4. terraform validate
5. terraform plan

```yaml
name: Terraform Plan

on:
  pull_request:
    branches:
      - main
    paths:
      - environments/staging/**

jobs:
  plan:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        workdir: [./environments/staging]
    env:
      tf_version: '1.0.1'
      tflint_version: '0.30.0'
    steps:
      - name: Checkout
        uses: actions/checkout@v2

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v1
        with:
          terraform_version: ${{ env.tf_version }}
          cli_config_credentials_token: ${{ secrets.TF_API_TOKEN }}

      - uses: terraform-linters/setup-tflint@v1
        name: Setup TFLint
        with:
          tflint_version: ${{ env.tflint_versions }}

      - name: Terraform Format
        run: terraform fmt -check -recursive

      - name: Terraform Init
        run: cd ./${{ matrix.workdir }} && terraform init

      - name: TFLint
        run: cd ./${{ matrix.workdir }} && tflint -f compact

      - name: Terraform Validate
        run: cd ./${{ matrix.workdir }} && terraform validate -no-color
        continue-on-error: true

      - name: Terraform Plan
        id: plan
        run: cd ./${{ matrix.workdir }} && terraform plan -no-color
        continue-on-error: true
```
ポイントはTerraform Setupの時に、`cli_config_credentials_token`を設定することです。

ここまで来たら、main.tfで何かresourceを定義してPRを作成して、「Actions」タブから
実行結果を確認出来れば、TerraformCloudとGithub Actionsの連携は完了となります。
![](https://storage.googleapis.com/zenn-user-upload/363aca0829239bb2dd6df63a.png)
planコマンドの出力中にTerraformCloudのリンクが含まれているので、
そこからTerraformCloud上で確認することも出来ます。

## さいごに

以上、TerraformCloudとGithub Actionsの連携方法のご紹介でした！

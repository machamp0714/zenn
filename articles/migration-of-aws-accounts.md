---
title: "AWSアカウントを移行する際にやったことまとめ"
emoji: "🐳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["AWS", "terraform"]
published: false
---

## 1. はじめに

最近アサインされたプロジェクトで、AWS アカウントの廃止が決定し、新しいアカウントへの移行を進めることになりました。旧アカウントの現状を詳しく確認していくと、リソースの管理において以下のような問題を抱えていることがわかりました。

- リソースの作成者や作成時期、作成意図が不明確な状態となっていた
- サーバーがパブリックサブネット上に配置されており、セキュリティリスクが存在していた
- DB などの重要なリソースが暗号化されていない状態でした
- IAM ユーザーが複数のアカウントに分散していた

本記事では、これらの課題を解決しながら、新環境への移行作業で実施した内容について詳しく説明していきます。既存環境の問題点を改善しつつ、より安全で管理しやすい AWS 環境を構築するための取り組みをまとめています。

## 2. AWS Organizations の導入

スタンドアロンなアカウントが複数存在していたので、移行にあたり、まず AWS Organizations を導入するところから着手しました。AWS Organizations は、複数の AWS アカウントを一元管理するためのサービスです。

組織内のアカウントは「管理アカウント」と「メンバーアカウント」の2種類に分別され、「管理アカウント」とは、 Organization を作成するためのアカウントのことで、それ以外を「メンバーアカウント」と呼びます。管理アカウントは、メンバーアカウントで発生した料金を支払う責任を持ちます。

:::message alert
管理アカウントを変更できないので注意が必要です。

さらに、管理アカウント内でのリソース作成は避けるべきです。これは、サービス制御ポリシー（SCP）が管理アカウント内のリソースには適用されないためです。
:::

### Organization の設計

Organization の設計は悩ましいのですが、公式が設計方針を示してくれています。

https://docs.aws.amazon.com/prescriptive-guidance/latest/security-reference-architecture/architecture.html

### IAM リソースの棚卸し

既存環境では、各アカウントに `UserA` や `UserB` といった個別の IAM ユーザーが存在していました。この状況を改善するため、[IAM Identity Center](https://docs.aws.amazon.com/ja_jp/singlesignon/latest/userguide/what-is.html) を導入しました。IAM Identity Center を利用すると Okta や Google Workspace の認証情報を利用して複数の AWS アカウントにアクセスできるようになり、各アカウントで IAM ユーザーを作成する必要がなくなります。

また、GitHub Actions などの CI 上で使用されていたデプロイ用の IAM ユーザーについても見直しを行ないました。具体的にはアクセスキーを使用した認証から IAM OpenID Connect を利用した認証方式に切り替えました。

1. GitHub Actions ワークフローは GitHub OIDC プロバイダから一時的な JWT を取得
2. `sts:AssumeRoleWithWebIdentity` と JWT を利用して一時的な認証情報を取得する
3. (2)で取得した認証情報で AWS リソースにアクセスできるようになる

という流れになります。

これでアクセスキーを長期間、GitHub のシークレットに保存する必要がなくなりました。 
Terraform のコードも載せておきます。

:::details OIDC プロバイダ

```hcl
data "tls_certificate" "this" {
  url = format("https://token.actions.githubusercontent.com")
}

resource "aws_iam_openid_connect_provider" "this" {
  url             = data.tls_certificate.this.url
  client_id_list  = ["sts.amazonaws.com"]
  thumbprint_list = [data.tls_certificate.this.certificates[0].sha1_fingerprint]
}

resource "aws_iam_role" "this" {
  name = "GithubActionsRole"
  assume_role_policy = data.aws_iam_policy_document.this.json
}

resource "aws_iam_role_policy_attachment" "this" {
  role       = aws_iam_role.this.name
  policy_arn = each.value.policy_arn
}

data "aws_iam_policy_document" "this" {
  statement {
    actions = ["sts:AssumeRoleWithWebIdentity"]

    principals {
      type        = "Federated"
      identifiers = [aws_iam_openid_connect_provider.this.arn]
    }

    condition {
      test     = "StringEquals"
      variable = "token.actions.githubusercontent.com:aud"
      values   = ["sts.amazonaws.com"]
    }

    condition {
      test     = "StringLike"
        variable = "token.actions.githubusercontent.com:sub"
        values   = ["repo:<organization>/<repository>:*"]
    }
  }
}
```
:::

## 3. IaC の導入

今までの運用は「誰がいつどのような変更を加えたのかわからない」、という課題があったので、
新アカウントでは Chatbot のような一部のリソースを除いてコード化に取り組んでいます。

### ステートファイルの分離

```tsx
.
└── environments
    ├── dev
    │   ├── provider.tf
    │   ├── main.tf
    │   ├── outputs.tf
    │   └── variables.tf
    ├── production
    │   ├── provider.tf
    │   ├── main.tf
    │   ├── outputs.tf
    │   └── variables.tf
    └── staging
        ├── provider.tf
        ├── main.tf
        ├── outputs.tf
        └── variables.tf
```

以前、上記のように環境ごとにステートファイルを分離していたのですが、この構成は学習コストが低い一方で、 `tfstate` が巨大なりデプロイが遅くなる、コードの見通しが悪くなるなどの課題もありました。

今回の構成は、環境フォルダ配下にコンポーネントレベルで分離しています。コンポーネントは一緒にデプロイされる可能性のあるリソースの集まりです。

```tsx
.
└── environments
    └── production
        ├── application
        │   ├── ecs
        │   └── secrets
        ├── monitoring
        ├── network
        ├── vpc
        │   └── alb
        ├── security
        │   ├── iam-role
        │   ├── oidc
        │   └── waf
        ├── storage
        │   ├── rds-mysql
        │   └── redis
        └── terragrunt.hcl
```

コンポーネント単位でステートファイルを細かく分けると下記のメリットがあります。

- tfstate のサイズが小さくなるのでデプロイの時間が短縮される
- コンポーネントの責務が明確になるので、コードの見通しが良くなる
- 1つのステートファイルの破損が他に影響しないので安全性が向上する

一方で、デメリットも存在します。

#### 設定ファイルを DRY にする工夫が必要になる

 [Terragrunt](https://terragrunt.gruntwork.io/) という Terraform のラッパーツールを活用するとバックエンドや `provider`  の設定ファイルを DRY にできます。
 
 terragrunt の`generate` ブロックを利用すると `terragrunt` の working directory にファイルを生成できるので、これを利用して provider と backend の設定ファイルを DRY にできます。

:::details generate ブロック
```hcl
# terragrunt.hcl

generate "provider" {
  path      = "provider.tf"
  if_exists = "overwrite_terragrunt"

  contents = <<-EOF
    terraform {
      required_providers {
        aws = {
          source  = "hashicorp/aws"
          version = "= 5.62.0"
        }
      }

      required_version = "= 1.9.7"

      cloud {
        organization = "<organization>"

        workspaces {
          name = "<workspace>"
        }
      }
    }

    provider "aws" {
      region = "ap-northeast-1"

      assume_role {
        role_arn = "arn:aws:iam::123456789123:role/terraform-role"
      }
    }
  EOF
}
```
:::


#### `terraform apply` のワンコマンドでデプロイできない

インフラをコンポーネント単位で分けている都合上、`apply` を一度に実行できません。コンポーネント A->コンポーネント B->コンポーネント C...といったように順番に `apply` していく必要がありますが、`terragrunt` には [run-all](https://terragrunt.gruntwork.io/docs/reference/cli-options/#run-all) という再起的に `terraform` コマンドを実行してくれるオプションがあります。 また、`terragrunt` の `dependencies` ブロックを利用して `apply` の実行順を制御できます。

```hcl
include "root" {
  path = find_in_parent_folders()
}

dependencies {
  paths = [
    find_in_parent_folders("network/vpc"),
  ]
}

dependency "vpc" {
  config_path = find_in_parent_folders("network/vpc")
  mock_outputs = {
    vpc_id = "vpc-xxxxxxxxxxxxxxxxx"
    subnet_ids = ["subnet-xxxxxxxxx", "subnet-xxxxxxxxx"]
  }
}

inputs = {
  vpc_id = dependency.vpc.outputs.vpc_id
  subnet_ids = dependency.vpc.outputs.database_subnets
}
```

このように `terragrunt.hcl` を記述して `production` ディレクトリで
 `terragrunt run-all apply` を実行すると、順番にコンポーネントがデプロイされます。

```hcl
Group 1
- Module /app/environments/production/network/vpc

Group 2
- Module /app/environments/production/storage/rds-mysql
```

さらに `dependency` ブロックを利用して別コンポーネントの `outputs` を参照できます。`outputs` がまだデプロイされていない場合は、`mock_outputs` を設定することでコンポーネントをテストできます。

### Terraform Workspace を使わなかった理由

インフラのコード化において、ステージング環境や本番環境といった複数の環境を管理する方法として、[Terraform Workspace](https://developer.hashicorp.com/terraform/language/state/workspaces) という機能があります。この機能を使用すると、同じ Terraform コードから異なる環境のインフラを作成できます。

```hcl
resource "aws_instance" "this" {
  instance_type = terraform.workspace == "production" ? "t3.medium" : "t3.small"

  tags = {
    Environment = terraform.workspace
  }
}
```

以前、小規模で環境間の差がそこまでないと考えていたプロジェクトで Workspace を採用したことがあるのですが、それでも `terraform.workspace == "production"` のようなコードが散在することになり、今回は見送りました。というより使う勇気がありませんでした😭

### AWS パブリックモジュールの活用

実装面では AWS 公式が提供する [パブリックモジュール](https://github.com/terraform-aws-modules) を採用しました。

理由としては、私のの技術的な習熟度では公式のモジュールより使いやすいモジュールを作れる気がしなかったのと、公式のモジュールを使うと簡単に [AWS のセキュリティのベストプラクティス](https://docs.aws.amazon.com/ja_jp/securityhub/latest/userguide/fsbp-standard.html) に準拠した設定になるからです。

例えば、[S3.5](https://docs.aws.amazon.com/ja_jp/securityhub/latest/userguide/s3-controls.html#s3-5) では HTTPS のリクエストのみ許可するバケットポリシーを設定する必要があるのですが、これは下記のように `attach_deny_insecure_transport_policy` を `true` にするだけで

```hcl
module "s3_bucket_for_logs" {
  source = "terraform-aws-modules/s3-bucket/aws"

  // Other arguments...

  attach_deny_insecure_transport_policy = true
}
```

下記のようなアクセスポリシーを設定してくれます。このようなコードはなるべく自分では書きたくなかったのでありがたいです。

:::details アクセスポリシー
```hcl
data "aws_iam_policy_document" "deny_insecure_transport" {
  count = local.create_bucket && var.attach_deny_insecure_transport_policy ? 1 : 0

  statement {
    sid    = "denyInsecureTransport"
    effect = "Deny"

    actions = [
      "s3:*",
    ]

    resources = [
      aws_s3_bucket.this[0].arn,
      "${aws_s3_bucket.this[0].arn}/*",
    ]

    principals {
      type        = "*"
      identifiers = ["*"]
    }

    condition {
      test     = "Bool"
      variable = "aws:SecureTransport"
      values = [
        "false"
      ]
    }
  }
}
```
:::

ただし、パブリックモジュールにもデメリットがあり、基本的に`count` を使用してリソースを作成するので、上記のコード中にもありますが、リソースへのアクセスがインデックスアクセスになってしまいます。

## 4. リソースの移行

### ネットワーク

ネットワーク関連のリソースに関しては完全に新規で作成しつつ、AWS サービスとの通信に関しては AWS PrivateLink 経由でアクセスするように構成を変更しました。

![](/images/internet_gateway.png)

以前は上記の図のようにインターネットゲートウェイ経由で AWS サービスと通信していました。

![](/images/vpc_endpoint.png)

AWS PrivateLink を利用した構成だと例えば CloudWatch にリクエストを送る場合…

1. DNS 解決が行なわれる。DNS クエリの結果としてエンドポイントネットワークインターフェース(ENI)のプライベート IP が返される
2. 解決されたプライベート IP にリクエストが送られる
3. ENI から AWS サービスにトラフィックが転送される

という流れになり、トラフィックは完全にプライベートネットワーク内で完結します。

### データベース

移行元のアカウントの RDS は暗号化されていなかったので、暗号化しつつ移行することになりました。アカウント間の移行でなければ

1. スナップショットを取得する
2. スナップショットをコピーするときに暗号化を有効にする
3. スナップショットを復元

という流れで行けるのですが、移行元アカウントで AWS マネージドキーの KMS で暗号化したスナップショットを移行先アカウントに移行しても、復元できませんでした。というのも AWS マネージドキーは AWS アカウントの同リージョン内でのみ使用可能なようです。

カスタマーKMS はキーポリシーを修正することで他アカウントに共有できるので、今回は移行先アカウントに KMS を作成し、移行元アカウントの IAM ユーザーに共有し、暗号化する際にこの KMS を指定することで対応しました。他のアカウントに共有するためのキーポリシーはこちらです。

:::details キーポリシー
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::123456789123:root"
            },
            "Action": "kms:*",
            "Resource": "*"
        },
        {
            "Sid": "Allow use of the key",
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::<移行元のアカウントID>:user/machamp"
            },
            "Action": [
                "kms:CreateGrant",
                "kms:Encrypt",
                "kms:Decrypt",
                "kms:ReEncrypt*",
                "kms:GenerateDataKey*",
                "kms:DescribeKey"
            ],
            "Resource": "*"
        },
        {
            "Sid": "Allow attachment of persistent resources",
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::<移行元のアカウントID>:user/machamp"
            },
            "Action": [
                "kms:CreateGrant",
                "kms:ListGrants",
                "kms:RevokeGrant"
            ],
            "Resource": "*",
            "Condition": {
                "Bool": {
                    "kms:GrantIsForAWSResource": "true"
                }
            }
        }
    ]
}
```
:::

これで移行元のアカウントでスナップショットをコピーする際に上記の KMS の arn を指定して暗号化して、移行先アカウントにスナップショットを共有して復元すれば完了となります。

### S3

S3 バケットのオブジェクトの移行には AWS コンソール内で作業が完結できる [AWS DataSync](https://docs.aws.amazon.com/ja_jp/datasync/latest/userguide/tutorial_s3-s3-cross-account-transfer.html) を利用しました。

#### 1. 移行元アカウントで DataSync ソースロケーションを作成する

AWS DataSync コンソールから新しいロケーションを作成します。
「S3 URI」には移行したい S3 バケットの URI を選択します。この時、IAM ロールが自動で生成されます。

#### 2. 移行元アカウントに移行先バケットに書き込む権限を持ったロールを作成する

IAM ロールを作成する際のユースケースは `DataSync` を選択します。さらに追加で下記のポリシーを IAM ロールに追加します。

:::details 移行用ポリシー
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Action": [
        "s3:GetBucketLocation",
        "s3:ListBucket",
        "s3:ListBucketMultipartUploads"
      ],
      "Effect": "Allow",
      "Resource": "arn:aws:s3:::example-bucket"
    },
    {
      "Action": [
        "s3:AbortMultipartUpload",
        "s3:DeleteObject",
        "s3:GetObject",
        "s3:ListMultipartUploadParts",
        "s3:PutObject",
        "s3:GetObjectTagging",
        "s3:PutObjectTagging"
      ],
      "Effect": "Allow",
      "Resource": "arn:aws:s3:::example-bucket/*"
    }
  ]
}
```
:::

#### 3. 移行先のアカウントで S3 バケットの ACL を無効にする

バケットを選択して「アクセス許可」タブを選択し、「オブジェクト所有者」から「ACL 無効(推奨)」を選択します。

#### 4. 移行先アカウントで S3 バケットポリシーを更新する

:::details バケットポリシー
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "denyOutdatedTLS",
            "Effect": "Deny",
            "Principal": "*",
            "Action": "s3:*",
            "Resource": [
                "arn:aws:s3:::example-bucket/*",
                "arn:aws:s3:::example-bucket"
            ],
            "Condition": {
                "NumericLessThan": {
                    "s3:TlsVersion": "1.2"
                }
            }
        },
        {
            "Sid": "denyInsecureTransport",
            "Effect": "Deny",
            "Principal": "*",
            "Action": "s3:*",
            "Resource": [
                "arn:aws:s3:::example-bucket/*",
                "arn:aws:s3:::example-bucket"
            ],
            "Condition": {
                "Bool": {
                    "aws:SecureTransport": "false"
                }
            }
        },
        {
            "Sid": "DataSyncCreateS3LocationAndTaskAccess",
            "Effect": "Allow",
            "Principal": {
                "AWS": "<手順2で作成した移行用ロールのARN>"
            },
            "Action": [
                "s3:PutObjectTagging",
                "s3:PutObject",
                "s3:ListMultipartUploadParts",
                "s3:ListBucketMultipartUploads",
                "s3:ListBucket",
                "s3:GetObjectTagging",
                "s3:GetObject",
                "s3:GetBucketLocation",
                "s3:DeleteObject",
                "s3:AbortMultipartUpload"
            ],
            "Resource": [
                "arn:aws:s3:::example-bucket/*",
                "arn:aws:s3:::example-bucket"
            ]
        },
        {
            "Sid": "DataSyncCreateS3Location",
            "Effect": "Allow",
            "Principal": {
                "AWS": "<手順2で作成した移行用ロールのARN>"
            },
            "Action": "s3:ListBucket",
            "Resource": "arn:aws:s3:::example-bucket"
        }
    ]
}
```
:::

#### 5. 移行元アカウントで、DataSync ロケーションを作成する

移行先アカウントが別アカウントの場合は、コンソールからロケーションを作成できないので、 AWS Cloud Shell から CLI を利用して作成することになります。

```sh
aws datasync create-location-s3 \
  --s3-bucket-arn arn:aws:s3:::<移行先のバケット名> \
  --s3-config '{
    "BucketAccessRoleArn":"arn:aws:iam::123456789123:role/<2で作ったロール名>"
  }'
```

#### 6. 移行元アカウントで、DataSync 転送タスクを作成して開始する

まず「送信元のロケーション」を既存のロケーションから選択し、次に「送信先のロケーション」を選択します。最後にオプションを選択し、タスクを開始すればオブジェクトの移行が完了します。

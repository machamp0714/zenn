---
title: "AWSアカウントを移行する際にやったことまとめ"
emoji: "🐳"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["AWS", "terraform"]
published: false
---

## 1. はじめに

最近アサインされたプロジェクトで、理由は記載できませんが、 AWS アカウントの廃止が決定し、新しいアカウントへの移行を進めることになりました。旧アカウントの現状を詳しく確認していくと、リソースの管理において以下のような問題を抱えていることが分かりました。

- リソースの作成者や作成時期、作成意図が不明確な状態となっていました
- リソースの命名規則に一貫性がなく、管理が困難な状況でした
- サーバーがパブリックサブネット上に配置されており、セキュリティリスクが存在していました
- データベースなどの重要なリソースが暗号化されていない状態でした
- IAM が複数のアカウントに分散して存在しており、権限管理の透明性が損なわれていました

本記事では、これらの課題を解決しながら、新環境への移行作業で実施した内容について詳しく説明していきます。既存環境の問題点を改善しつつ、より安全で管理しやすい AWS 環境を構築するための取り組みをまとめています。

## 2. AWS Oraganizations の導入

スタンドアロンなアカウントが複数存在していたので、移行にあたり、まず AWS Organizations を導入するところから着手しました。AWS Organizations は、複数の AWS アカウントを一元管理するためのサービスです。

組織内のアカウントは「管理アカウント」と「メンバーアカウント」の2種類に分別され、「管理アカウント」とは、 Organization を作成するためのアカウントのことで、それ以外を「メンバーアカウント」と呼びます。管理アカウントは、メンバーアカウントで発生した料金を支払う責任を持ちます。

:::message alert
管理アカウントを変更することはできないので注意が必要です。

さらに、管理アカウント内でのリソース作成は避けるべきです。これは、サービス制御ポリシー（SCP）が管理アカウント内のリソースには適用されないためです。
:::

### Organization の設計

Organization の設計は悩ましいのですが、公式が設計方針を示してくれています。

https://docs.aws.amazon.com/prescriptive-guidance/latest/security-reference-architecture/architecture.html

### IAM リソースの棚卸し

既存環境では、各アカウントに `userA` や `userB` といった個別の IAM ユーザーが存在していました。この状況を改善するため、[IAM Identity Center](https://docs.aws.amazon.com/ja_jp/singlesignon/latest/userguide/what-is.html) を導入しました。IAM Identity Center を利用すると Okta や Google Workspace の認証情報を利用して複数の AWS アカウントにアクセス出来るようになり、各アカウントで IAM を発行する必要がなくなります。

また、GitHub Actions で使用されていた デプロイ用の IAM ユーザーについても見直しを行いました。具体的にはアクセスキーを使用した認証から IAM OpenID Connect を利用した認証方式に切り替えました。

1. GitHub Actions ワークフローは GitHub OIDC プロバイダから一時的な JWT を取得
2. `sts:AssumeRoleWithWebIdentity` と JWT を利用して一時的な認証情報を取得する
3. (2)で取得した認証情報で AWS リソースにアクセス出来るようになる

という流れになります。

これでアクセスキーを長期間、GitHub のシークレットに保存する必要がなくなりました。 

Terraform のコードも載せておきます。

:::details OIDC プロバイダ

```
data "tls_certificate" "this" {
  url = format("https://%s", each.value.hostname)
}

resource "aws_iam_openid_connect_provider" "this" {
  url             = data.tls_certificate.this[each.key].url
  client_id_list  = [each.value.audience]
  thumbprint_list = [data.tls_certificate.this[each.key].certificates[0].sha1_fingerprint]
}

resource "aws_iam_role" "this" {
  name = each.value.role_name
  assume_role_policy = data.aws_iam_policy_document.this[each.key].json
}

resource "aws_iam_role_policy_attachment" "this" {
  for_each = var.oidc_providers

  role       = aws_iam_role.this[each.key].name
  policy_arn = each.value.policy_arn
}

data "aws_iam_policy_document" "this" {
  for_each = var.oidc_providers

  statement {
    actions = ["sts:AssumeRoleWithWebIdentity"]

    principals {
      type        = "Federated"
      identifiers = [aws_iam_openid_connect_provider.this[each.key].arn]
    }

    dynamic "condition" {
      for_each = each.key == "terraform_cloud" ? [1] : []

      content {
        test     = "StringEquals"
        variable = "${each.value.hostname}:aud"
        values   = [each.value.audience]
      }
    }

    dynamic "condition" {
      for_each = each.key == "terraform_cloud" ? [1] : []

      content {
        test     = "StringLike"
        variable = "${each.value.hostname}:sub"
        values   = ["organization:${each.value.organization_id}:project:*:workspace:*:run_phase:*"]
      }
    }

    dynamic "condition" {
      for_each = each.key == "github_actions" ? [1] : []

      content {
        test     = "StringEquals"
        variable = "${each.value.hostname}:aud"
        values   = ["sts.amazonaws.com"]
      }
    }

    dynamic "condition" {
      for_each = each.key == "github_actions" ? [1] : []

      content {
        test     = "StringLike"
        variable = "${each.value.hostname}:sub"
        values   = ["repo:timecrowdinc/timecrowd-enterprise:*"]
      }
    }
  }
}
```

:::

# IaC の実践

今までの運用は誰がいつどのような変更を加えたのか分からない、という問題があったので、

新アカウントでは Chatbot のようなリソースを除いてコード化に取り組んでいます。

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

別件ではこののように環境ごとにステートファイルを分離していたのですが、この構成は学習コストが低い一方で問題もありました。

- ステートファイルが巨大でデプロイが遅い
- コードの見通しが悪い

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
    │   ├── vpc
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
- 一つのステートファイルの破損が他に影響しないので安全性が向上する

一方で、以下のようなデメリットも存在します。

**設定ファイルを各階層に作らないといけないので、これを DRY にする工夫が必要になる**

 [Terragrunt](https://terragrunt.gruntwork.io/) という Terraform のラッパーツールを活用するとバックエンドや `provider`  の設定ファイルを DRY にすることができます。

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

terragrunt の`generate` ブロックを利用すると `terragrunt` の working directory にファイルを生成することができるので、これを利用して provider と backend の設定ファイルを DRY にすることができます。

**コンポーネント間に依存関係が出来るので `terraform apply` のワンコマンドでデプロイ出来なくなる**

---

コンポーネント単位で分けている都合上、デプロイを順番通りに実行しなければいけません。

例えば `vpc` をデプロイしてから `rds` の順番に apply する必要があります。

`terragrunt` の `dependencies` ブロックを利用して `apply` の順番を制御できます。さらに、別コンポーネントの `outputs` を参照したい場合は、 `dependecy` ブロックを利用して `variables` として渡すことも可能です。

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
}

inputs = {
  vpc_id = dependency.vpc.outputs.vpc_id
  subnet_ids = dependency.vpc.outputs.database_subnets
}
```

このように `terragrunt.hcl` を記述して `terragrunt run-all plan` を実行すると、

```hcl
Group 1
- Module /app/environments/production/network/vpc

Group 2
- Module /app/environments/production/storage/rds-mysql
```

このように先に `vpc` で `plan` が実行され、次に `rds-mysql` で実行されていることが分かります。

### workspace を使わなかった理由

インフラのコード化において、開発環境や本番環境といった複数の環境を管理する方法として、Terraform Workspace という機能があります。この機能を使用すると、同じ Terraform コードから異なる環境のインフラを作成できます。

```hcl
resource "aws_instance" "this" {
  instance_type = terraform.workspace == "production" ? "t3.medium" : "t3.small"

  tags = {
    Environment = terraform.workspace
  }
}

```

以前、小規模で環境間の差がそこまでないと考えていたプロジェクトで Workspace を採用したことがあるのですが、それでも `terraform.workspace == "production"` のようなコードが散在することになり、今回は見送りました。というより使う勇気がありませんでした。。。

### AWS パブリックモジュールの活用

コード化に関しては、AWS が提供する [パブリックモジュール](https://github.com/terraform-aws-modules) を採用しました。

理由としては、自分の技術的な習熟度では公式のモジュールより使いやすいモジュールを作れる気がしないのと、公式のモジュールを使うと簡単に [AWS のセキュリティのベストプラクティス](https://docs.aws.amazon.com/ja_jp/securityhub/latest/userguide/fsbp-standard.html) に準拠した設定になるように構成されている所です。

例えば、[S3.5](https://docs.aws.amazon.com/ja_jp/securityhub/latest/userguide/s3-controls.html#s3-5) では HTTPS のリクエストのみ許可するバケットポリシーを設定する必要があるのですが、これは下記のように `attach_deny_insecure_transport_policy` を `true` にするだけで、

```hcl
module "s3_bucket_for_logs" {
  source = "terraform-aws-modules/s3-bucket/aws"

  bucket = "example-bucket"

  attach_deny_insecure_transport_policy = true
}
```

このようにアクセスポリシーを設定してくれます。このようなコードは自分では書きたくないのでありがたいです。

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

ただし、パブリックモジュールにもデメリットがあり、基本的に`count` を使用してリソースを作成するので、上記のコード中にもありますが、リソースへのアクセスがインデックスアクセスになってしまうのがイマイチに感じます。ただ個人的に Terraform はあくまで設定ファイルなので、これくらいであれば許容の範囲内と考えています。

# リソースの移行

### ネットワーク編

ネットワーク関連のリソースに関しては完全に新規で作成しつつ、AWS サービスとの通信に関しては AWS PrivatLink 経由でアクセスするように構成を変更しました。

以前は下記の図のようにインターネットゲートウェイ経由で AWS サービスと通信していました。

構成図A

構成図B

この構成だと例えば CloudWatch にリクエストを送る場合…

1. まず DNS 解決が行われる。DNS クエリの結果としてエンドポイントネットワークインターフェース(ENI)のプライベート IP が返される
2. 解決されたプライベート IP にリクエストが送られる
3. ENI から AWS サービスにトラフィックが転送される

という流れになるので、トラフィックは完全にプライベートネットワーク内で完結します。

### データベース編

移行元のアカウントの RDS は暗号化されていなかったので、暗号化しつつ移行することになりました。アカウントを移行しなければ

1. スナップショットを取得する
2. スナップショットをコピーするときに暗号化を有効にする
3. スナップショットを復元

という流れで行けるのですが、移行元アカウントで AWS マネージドキーの KMS で暗号化したスナップショットを移行先アカウントに移行しても、復元することができませんでした。というのも AWS マネージドキーは AWS アカウントの同リージョン内でのみ使用可能なようです。

カスタマーKMS はキーポリシーを修正することで他アカウントに共有することが出来るので、今回は移行先アカウントに KMS を作成し、移行元アカウントの IAM ユーザーに共有し、暗号化する際にこの KMS を指定することで対応しました。

キーポリシーはこちらです。

```json
{
    "Version": "2012-10-17",
    "Id": "key-consolepolicy-3",
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

スクショ

### S3編

S3 バケットのオブジェクトの移行には AWS コンソール内で作業が完結できる [AWS DataSync](https://docs.aws.amazon.com/ja_jp/datasync/latest/userguide/tutorial_s3-s3-cross-account-transfer.html) を利用しました。

**手順1** **移行元アカウントで DataSync ソースロケーションを作成する**

ここでは移行したい S3 バケットの URI を選択します。IAM ロールは自動で生成されます。

**手順2 移行元アカウントに移行先バケットに書き込む権限を持ったロールを作成する**

IAM ロールを作成する際のユースケースは `DataSync` を選択します。さらに追加で下記のポリシーを IAM ロールに追加します。

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
			"Resource": "arn:aws:s3:::timecrowd-sync-stg-report"
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
			"Resource": "arn:aws:s3:::timecrowd-sync-stg-report/*"
		}
	]
}
```

**手順3 移行先のアカウントで S3 バケットの ACL を無効にする**

バケットを選択して「アクセス許可」タブを選択し、「オブジェクト所有者」から「ACL無効(推奨)」を選択します。

**手順4** **移行先アカウントで S3 バケットポリシーを更新します。**

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
                "arn:aws:s3:::timecrowd-sync-stg-report/*",
                "arn:aws:s3:::timecrowd-sync-stg-report"
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
                "arn:aws:s3:::timecrowd-sync-stg-report/*",
                "arn:aws:s3:::timecrowd-sync-stg-report"
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
                "AWS": "arn:aws:iam::446736013352:role/EnterpriseDataSyncRole"
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
                "arn:aws:s3:::timecrowd-sync-stg-report/*",
                "arn:aws:s3:::timecrowd-sync-stg-report"
            ]
        },
        {
            "Sid": "DataSyncCreateS3Location",
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::446736013352:role/EnterpriseDataSyncRole"
            },
            "Action": "s3:ListBucket",
            "Resource": "arn:aws:s3:::timecrowd-sync-stg-report"
        },
        {
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::471112578051:role/enterprise_staging-20241108070716256100000006"
            },
            "Action": "s3:GetObject",
            "Resource": [
                "arn:aws:s3:::timecrowd-sync-stg-report/*",
                "arn:aws:s3:::timecrowd-sync-stg-report"
            ]
        }
    ]
}
```

**手順5 移行元アカウントで、DataSync ロケーションを作成する**

移行先アカウントが別アカウントの場合は、コンソールからロケーションを作成することが出来ないので、 AWS Cloud Shell から CLI を利用して作成することになります。

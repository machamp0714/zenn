---
title: "Terraformで作成済みリソース名を変更する方法"
emoji: "🐈"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["terraform", "AWS"]
published: true
---

terraform を書いていて、後になってresource名やmodule名を変えたくなる時があると思います。
ただresourceの名前をそのまま変更しても差分が発生してしまい、変更するリソースによっては
変更対象のリソース以外にも影響を与えてしまい、よろしくありません。

## 目次

1. リソース名を変更すると...
2. tfstate を pull する
3. tfstate を編集する
4. tfstate を push する

## 1. リソース名を変更すると...

この様なVPCがあったとします。
```hcl
resource "aws_vpc", "vpc_for_hoge" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true
}
```
このVPCをresource名を変更します。
```hcl
resource "aws_vpc", "vpc_for_piyo" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true
}
```
この状態でplanを実行してみます。
```hcl
  # aws_vpc.vpc_for_hoge will be destroyed
  - resource "aws_vpc" "this" {
      - arn                              = "arn:aws:ec2:ap-northeast-1:123456789:vpc/vpc-0198e05f543275242" -> null
      - assign_generated_ipv6_cidr_block = false -> null
      - cidr_block                       = "10.0.0.0/16" -> null
      - default_network_acl_id           = "acl-0c5a08d26cf7d020b" -> null
      - default_route_table_id           = "rtb-058f952e4293e7f6b" -> null
      - default_security_group_id        = "sg-0600ca88084f60c0d" -> null
      - dhcp_options_id                  = "dopt-53e6f734" -> null
      - enable_classiclink               = false -> null
      - enable_classiclink_dns_support   = false -> null
      - enable_dns_hostnames             = true -> null
      - enable_dns_support               = true -> null
      - id                               = "vpc-0198e05f543275242" -> null
      - instance_tenancy                 = "default" -> null
      - main_route_table_id              = "rtb-058f952e4293e7f6b" -> null
      - owner_id                         = "123456789" -> null
    }

# aws_vpc.vpc_for_piyo will be created
  + resource "aws_vpc" "this" {
      + arn                              = (known after apply)
      + assign_generated_ipv6_cidr_block = false
      + cidr_block                       = "10.0.0.0/16"
      + default_network_acl_id           = (known after apply)
      + default_route_table_id           = (known after apply)
      + default_security_group_id        = (known after apply)
      + dhcp_options_id                  = (known after apply)
      + enable_classiclink               = (known after apply)
      + enable_classiclink_dns_support   = (known after apply)
      + enable_dns_hostnames             = true
      + enable_dns_support               = true
      + id                               = (known after apply)
      + instance_tenancy                 = "default"
      + ipv6_association_id              = (known after apply)
      + ipv6_cidr_block                  = (known after apply)
      + main_route_table_id              = (known after apply)
      + owner_id                         = (known after apply)
    }
```

このように `create-> destroy` が発生してしまいます。
これを解消するには `tfstate` を修正する必要があります。

## 2. tfstateをpullする

terraformで開発する時、backendにS3やTerraformCloudを指定して開発していると思います。
まずは、remote backendからtfstateをローカルにpullします。

```shell
$ terraform state pull > local.tfstate
```

これで作業ディレクトリに `local.tfstate` という名前でtfstateをpull出来ました。

## 3. tfstateを編集する

```shell
$ terraform state mv -state=local.tfstate aws_vpc.vpc_for_hoge aws_vpc.vpc_for_piyo
```
terraformのmvコマンドでtfstateを編集できます。その時 `-state` オプションで
編集するtfstate ファイルを指定してください。実行するとバックアップ用の
tfstateも同時に生成されます。編集に失敗しても安心ですね。

## 4. planで差分がないか確認する

念の為、ローカルでplanを実行して差分がないことを確認しましょう。
その前にbackendをローカルに変更します。

```hcl
# backend "remote" {
#   organization = "personal"

# workspaces {
#   name = "machamp-workspace"
#  }
# }

backend "local" {}
```

```shell
$ terraform init --reconfigure
```
backendをローカルに変更したらplanをtfstateを指定して実行します。
```
$ terraform plan -state=local.tfstate
```
resourceに変更がないことが確認出来たら、backendをremoteに戻します。

```hcl
backend "remote" {
  organization = "personal"

  workspaces {
    name = "machamp-workspace"
  }
}
```

## 5. tfstateをpushする

最後に編集したtfstateをremote backendにpushします。

```shell
$ terraform state push local.tfstate
```

これでresourceの名前を変更することが出来ました。
作業が完了したらlocal.tfstateとバックアップ用のtfstateを削除しましょう。

resourceの名前を変えるだけでこんな手順を踏まないと行けないのかーと思ってしまいます(泣)
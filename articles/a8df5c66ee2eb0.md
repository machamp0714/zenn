---
title: "(仮)Terraformで複数のリソースを作りたい時 ~for_each VS count~ 徹底比較"
emoji: "⛳"
type: "tech"
topics: ["AWS", "terraform"]
published: false
---
## はじめに

terraformでリソースを複数削除したい時、 `count` や `for_each` が選択肢に上がると思います。
業務でterraformを使ってる時基本的に `count` を使って複数リソースを作成していたのですが、
適切に使い分けたいと思い、それぞれの良い所・微妙に感じた所をまとめてみました。

コードは[こちら](https://github.com/machamp0714/for_each_vs_count)

## 目次

1. countのメリット・デメリット
2. for_eachのメリット・デメリット

## 1. countでリソースを作成した場合

### countで複数リソースを作る

countの挙動を確かめるために下記の様にサブネットを作成します。
　
```hcl
resource "aws_vpc" "this" {
  cidr_block           = var.cidr_block
  enable_dns_hostnames = var.enable_dns_hostnames
  enable_dns_support   = var.enable_dns_support
}

resource "aws_subnet" "public" {
  count = length(var.public_blocks)

  vpc_id            = aws_vpc.this.id
  cidr_block        = element(var.public_blocks, count.index)
  availability_zone = element(var.availability_zones, count.index)
}
```
```hcl
locals {
  region = "ap-northeast-1"
}

module "count" {
  source = "./count"

  cidr_block         = "10.0.0.0/16"
  public_blocks      = ["10.0.1.0/24", "10.0.2.0/24"]
  availability_zones = ["${local.region}a", "${local.region}c"]
}
```

countは作成したいリソースの数を指定するだけなので、for_eachを使ったときに比べて
シンプルに記述出来ます。またサブネットはサブネットグループを作成する時にサブネットのIDを
listで渡すことになりますが、そんな時も以下の様にoutputを書けば簡単にサブネットのidを
listで参照出来ます。

```hcl
output "subnet_ids" {
  value = aws_subnet.public.*.id
}
```
これを使いたい時は、
```
module.count.subnet_ids
```
とすればOKです。
複数リソースを参照したい時は良いのですが、個別で参照したい時、今回の例ですと
AZが `ap-northeast-1a` のサブネットを参照したい時はoutputを↓の様に書くか、
```
output "subnet_1a_id" {
  value = aws_subnet.public[0].id
}
```
または、
```
module.count.subnet_ids[0]
```
と書くなど番号を指定してリソースを参照するので何を参照しているのか分かりにくいです。

リソースを削除したい時
途中でAZが `1a` のサブネットを削除したくなったとします。
```
module "count" {
  source = "./count"

  cidr_block         = "10.0.0.0/16"
  public_blocks      = ["10.0.2.0/24"]
  availability_zones = ["${local.region}c"]
}
```
この様に書き換えてplanを実行してみると・・・
```
Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following symbols:
  - destroy
-/+ destroy and then create replacement

Terraform will perform the following actions:

  # module.count.aws_subnet.public[0] must be replaced
-/+ resource "aws_subnet" "public" {
      ~ arn                             = "arn:aws:ec2:ap-northeast-1:762742784206:subnet/subnet-06f995d8d74733a4b" -> (known after apply)
      ~ availability_zone               = "ap-northeast-1a" -> "ap-northeast-1c" # forces replacement
      ~ availability_zone_id            = "apne1-az4" -> (known after apply)
      ~ cidr_block                      = "10.0.1.0/24" -> "10.0.2.0/24" # forces replacement
      ~ id                              = "subnet-06f995d8d74733a4b" -> (known after apply)
      + ipv6_cidr_block_association_id  = (known after apply)
      - map_customer_owned_ip_on_launch = false -> null
      ~ owner_id                        = "762742784206" -> (known after apply)
      - tags                            = {} -> null
      ~ tags_all                        = {} -> (known after apply)
        # (3 unchanged attributes hidden)
    }

  # module.count.aws_subnet.public[1] will be destroyed
  - resource "aws_subnet" "public" {
      - arn                             = "arn:aws:ec2:ap-northeast-1:762742784206:subnet/subnet-0bd46cfab1a7d126e" -> null
      - assign_ipv6_address_on_creation = false -> null
      - availability_zone               = "ap-northeast-1c" -> null
      - availability_zone_id            = "apne1-az1" -> null
      - cidr_block                      = "10.0.2.0/24" -> null
      - id                              = "subnet-0bd46cfab1a7d126e" -> null
      - map_customer_owned_ip_on_launch = false -> null
      - map_public_ip_on_launch         = false -> null
      - owner_id                        = "762742784206" -> null
      - tags                            = {} -> null
      - tags_all                        = {} -> null
      - vpc_id                          = "vpc-00e075e18377e14b8" -> null
    }

Plan: 1 to add, 0 to change, 2 to destroy.
```
この様に `module.count.aws_subnet.public[0]` をreplaceする挙動を取ります。
AZが `ap-northeast-1c` のサブネットを残したまま変更を適用したい場合は
以下の手順を実行すればOKです。

まず、stateを確認しましょう。
```
❯ terraform state list  
module.count.aws_subnet.public[0]
module.count.aws_subnet.public[1]
module.count.aws_vpc.this
```
次に `public[0]` に1以外の適当な番号を割り当てます。
```
terraform state mv module.count.aws_subnet.public[0] module.count.aws_subnet.public[10]
```
次に `public[1]` を `public[0]` に変更します。
```
terraform state mv module.count.aws_subnet.public[1] module.count.aws_subnet.public[0]
```
この状態でplanを実行すると
```
Terraform will perform the following actions:

  # module.count.aws_subnet.public[10] will be destroyed
  - resource "aws_subnet" "public" {
      - arn                             = "arn:aws:ec2:ap-northeast-1:762742784206:subnet/subnet-0850badcdfdf2d6b9" -> null
      - assign_ipv6_address_on_creation = false -> null
      - availability_zone               = "ap-northeast-1a" -> null
      - availability_zone_id            = "apne1-az4" -> null
      - cidr_block                      = "10.0.1.0/24" -> null
      - id                              = "subnet-0850badcdfdf2d6b9" -> null
      - map_customer_owned_ip_on_launch = false -> null
      - map_public_ip_on_launch         = false -> null
      - owner_id                        = "762742784206" -> null
      - tags                            = {} -> null
      - tags_all                        = {} -> null
      - vpc_id                          = "vpc-079d79a4be16828a8" -> null
    }

Plan: 0 to add, 0 to change, 1 to destroy.
```
これでAZが `ap-northeast-1a` のサブネットのみ削除することが出来るのですが、
ちょっと面倒ですね。。。countを使うときはリソースを削除した時の挙動をしっかり
把握した上で使うのが良いと思っています。
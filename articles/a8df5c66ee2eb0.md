---
title: "terraformで複数リソースを作りたい時、countとfor_eachどちらを使うか"
emoji: "⛳"
type: "tech"
topics: ["AWS", "terraform"]
published: false
---

terraformで複数リソースを作成する時、countかfor_eachを使うと思いますが、
両者を適切に使い分けるためにそれぞれの挙動をまとめてみました。

ソースコードは[こちら](https://github.com/machamp0714/for_each_vs_count)

## 目次

1. countの場合
2. for_eachの場合
3. まとめ

## 1. countの場合

### countを使って複数リソースを作る

countの挙動を確かめるために下記の様にサブネットを作成します。
　
```hcl
resource "aws_vpc" "this" {
  cidr_block  = var.cidr_block
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

countは作成したいリソースの数を指定することで複数リソースを作成出来ます。
作成したいリソースの数を指定するだけなので `for_each` よりもシンプルに記述出来ます。

### 作成した複数リソースをoutputしたい時

```hcl
output "subnet_ids" {
  value = aws_subnet.public.*.id
}
```
countで作ったリソースを参照したい時、 `*` を使えば `list` で取得出来ます。

```
subnet_ids = [
  "subnet-01eb09ab26904e369",
  "subnet-06c6128ce474cd3eb",
]
```
ただ、 `ap-northeast-1a` のサブネットを参照したい場合、次の様に
```
module.count.subnet_ids[0]
```
と配列の番号を指定することになってしまい、これでは何のリソースを参照しているのか
分かりにくいですね。 後述しますが `for_each` だとこの問題を解消することが出来ます。

### リソースを削除したい時
途中でAZが `ap-northeast-1a` のサブネットを削除したくなったとします。
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
      ~ arn                             = "arn:aws:ec2:ap-northeast-1:76345784206:subnet/subnet-06f995d8d74733a4b" -> (known after apply)
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
      - arn                             = "arn:aws:ec2:ap-northeast-1:76274484206:subnet/subnet-0bd46cfab1a7d126e" -> null
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
この様に `module.count.aws_subnet.public[0]` をreplaceする挙動となります。
こんな時、出来れば `ap-northeast-1c` のサブネットを残したまま、 `ap-northeast-1a` の
サブネットを削除したいと思いますが、残念ながら出来ません。

ただstateを直接修正することで、既存のサブネットを残したまま修正することも可能です。

まず、stateを確認しましょう。
```
❯ terraform state list  
module.count.aws_subnet.public[0]
module.count.aws_subnet.public[1]
module.count.aws_vpc.this
```
次に `module.count.aws_subnet.public[0]` に `1` 以外の適当な番号を割り当てます。
```
terraform state mv module.count.aws_subnet.public[0] module.count.aws_subnet.public[10]
```
次に `module.count.aws_subnet.public[1]` を `module.count.aws_subnet.public[0]` に
変更します。
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
`1 to destroy` となってますね！
これで `ap-northeast-1a` のサブネットのみ削除することが出来ました。

## 2. for_eachを使う場合

### for_eachで複数リソースを作成する

countと同じくfor_eachの挙動を確認したいので、リソースを作ってみます。
```hcl
resource "aws_vpc" "this" {
  cidr_block           = var.cidr_block
}

resource "aws_subnet" "public" {
  for_each = var.public_subnets

  vpc_id            = aws_vpc.this.id
  cidr_block        = each.value.cidr_block
  availability_zone = each.key
}
```
```hcl
locals {
  public_subnets = {
    ap-northeast-1a = {
      cidr_block　= "10.0.64.0/24"
    }
    ap-northeast-1c = {
      cidr_block　= "10.0.65.0/24"
    }
  }
}

module "for_each" {
  source = "./for_each"

  cidr_block     = "10.0.0.0/16"
  public_subnets = local.public_subnets
}
```
for_eachには `map` と `set` を渡します。 `each.value` `each.key` で
それぞれのkeyとvalueを参照することが出来ます。

### 作成したリソースをoutputしたい時

```hcl
output "public_1a_id" {
  value = aws_subnet.public["ap-northeast-1a"].id
}

output "subnet_ids" {
  value = [ for value in aws_subnet.public : value.id ]
}
```
`count` で作成した時と違い `list` で取得したい時は[for文](https://www.terraform.io/docs/language/expressions/for.html)を使う所に注意してください。
また、個々のリソースを参照したい場合、上記の様に `aws_subnet.public["ap-northeast-1a"]` 
とkeyで参照するので `count` で作成した時と比べて個々のリソースを参照しやすいですね。

### リソースを削除したい時

```
locals {
  public_subnets = {
    ap-northeast-1c = {
      cidr_block        = "10.0.65.0/24"
    }
  }
}
```
と `ap-northeast-1a` のサブネットを削除して `plan` を実行してみます。
```
Terraform will perform the following actions:

  # module.for_each.aws_subnet.public["ap-northeast-1a"] will be destroyed
  - resource "aws_subnet" "public" {
      - arn                             = "arn:aws:ec2:ap-northeast-1:76276724206:subnet/subnet-0f966e9cd41aaf87c" -> null
      - assign_ipv6_address_on_creation = false -> null
      - availability_zone               = "ap-northeast-1a" -> null
      - availability_zone_id            = "apne1-az4" -> null
      - cidr_block                      = "10.0.64.0/24" -> null
      - id                              = "subnet-0f966e9cd41aaf87c" -> null
      - map_customer_owned_ip_on_launch = false -> null
      - map_public_ip_on_launch         = false -> null
      - owner_id                        = "762742784206" -> null
      - tags                            = {} -> null
      - tags_all                        = {} -> null
      - vpc_id                          = "vpc-06767e6aec20a67ca" -> null
    }

Plan: 0 to add, 0 to change, 1 to destroy.
```
countと違い、stateを修正することなく特定のリソースを削除することが出来ます。

## 3. まとめ

個人的な考えになりますが、対象のリソースを他のリソースを作成する時に参照する必要がなく、
ただ複数個作成作成出来れば良いという場合は `count` を使い、サブネットの様に個々の
リソースを作成する必要がある場合は、 `for_each` を使うのが良いかなーと思いました。

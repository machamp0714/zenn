---
title: "CapacityProvider"
emoji: "🐡"
type: "tech"
topics: ["AWS", "Terraform"]
published: false
---
## はじめに

AutoScalingGroupを導入しようと思ったきっかけ

ハマった経験

## CapacityProvider とは？

[https://docs.aws.amazon.com/ja_jp/AmazonECS/latest/developerguide/asg-capacity-providers.html](https://docs.aws.amazon.com/ja_jp/AmazonECS/latest/developerguide/asg-capacity-providers.html)

ECS Capacity Providerを利用すると、 Auto Scaling Groupを利用してクラスターに登録されたEC2インスタンスを管理できる。

Capacity Providerを利用するにあたってまず、 Auto Scaling Groupが必要になるので、

先に必要なリソースを作成します。

## Auto Scaling グループ を作成する

Auto Scalingグループを作成するときは、 EC2起動テンプレートを先に作る必要がある。

起動テンプレートに基づいてインスタンスが起動する

```ruby
# Launch Template
```

```ruby
# Auto Scalingグループ
```

## クラスターにCapacity Providerを関連付ける

```ruby
# cluster
```

## ECS Service に Capacity Provider Strategyを定義する

```ruby
# service
```

## 動作検証

グループの `capacity` を変更してインスタンスを起動させる。タスクが起動することを確認。

キャパシティプロバイダの管理下にインスタンスがあることをAPIで確認する

`desired_count` を増やしてインスタンスが新たに起動することを確認

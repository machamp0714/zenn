---
title: "CapacityProvider"
emoji: "ð¡"
type: "tech" # tech: æè¡è¨äº / idea: ã¢ã¤ãã¢
topics: ["AWS", "Terraform"]
published: false
---
## ã¯ããã«

AutoScalingGroup ãå°å¥ãããã¨æã£ããã£ãã

ããã£ãçµé¨

## CapacityProvider ã¨ã¯ï¼

[https://docs.aws.amazon.com/ja_jp/AmazonECS/latest/developerguide/asg-capacity-providers.html](https://docs.aws.amazon.com/ja_jp/AmazonECS/latest/developerguide/asg-capacity-providers.html)

ECS Capacity Provider ãå©ç¨ããã¨ã Auto Scaling Group ãå©ç¨ãã¦ã¯ã©ã¹ã¿ã¼ã«ç»é²ããã EC2 ã¤ã³ã¹ã¿ã³ã¹ãç®¡çãããã¨ãåºæ¥ãã

Capacity Providerãå©ç¨ããã«ããã£ã¦ã¾ãã Auto Scaling Groupãå¿è¦ã«ãªãã®ã§ã

åã«å¿è¦ãªãªã½ã¼ã¹ãä½æãã¾ãã

## Auto Scaling ã°ã«ã¼ã ãä½æãã

Auto Scaling ã°ã«ã¼ããä½æããæã¯ã EC2 èµ·åãã³ãã¬ã¼ããåã«ä½ãå¿è¦ãããã

èµ·åãã³ãã¬ã¼ãã«åºã¥ãã¦ã¤ã³ã¹ã¿ã³ã¹ãèµ·åãã

```ruby
# Launch Template
```

```ruby
# Auto Scalingã°ã«ã¼ã
```

## ã¯ã©ã¹ã¿ã¼ã«Capacity Providerãé¢é£ä»ãã

```ruby
# cluster
```

## ECS Service ã« Capacity Provider Strategyãå®ç¾©ãã

```ruby
# service
```

## åä½æ¤è¨¼

ã°ã«ã¼ãã® `capacity` ãå¤æ´ãã¦ã¤ã³ã¹ã¿ã³ã¹ãèµ·åããããã¿ã¹ã¯ãèµ·åãããã¨ãç¢ºèªã

ã­ã£ãã·ãã£ãã­ãã¤ãã®ç®¡çä¸ã«ã¤ã³ã¹ã¿ã³ã¹ããããã¨ãAPIã§ç¢ºèªãã

`desired_count` ãå¢ããã¦ã¤ã³ã¹ã¿ã³ã¹ãæ°ãã«èµ·åãããã¨ãç¢ºèª

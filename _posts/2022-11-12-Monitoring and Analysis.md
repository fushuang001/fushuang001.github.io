---
layout:         post
title:          AWS 监控、分析，以及数据分析
subtitle:		    备考 SAA 的笔记
date:           2022-11-12
author:         Luke
cover:          '/assets/img/bg-monitor-and-analysis.png'
tags:           AWS, SAA, CloudWatch, Cloudtrail, EventBridge, Trust Advisor, Athena, Kinesis
---
- [CloudWatch](#cloudwatch)
- [Cloudtrail](#cloudtrail)
  - [AWS Global Services](#aws-global-services)
- [EventBridge](#eventbridge)
- [Trusted Advisor](#trusted-advisor)
- [Cost Explore](#cost-explore)
- [Athena](#athena)
- [Kinesis](#kinesis)
  - [Kinesis 三种分类](#kinesis-三种分类)

# CloudWatch
- 将 AWS 各个 services 相关的 metrics/指标，logs/日志 集中起来，方便查看，以及做一些 insights 分析  
- 通过创建 CloudWatch Alert，在特定情况下（比如 EC2 CPU 利用率 5 分钟内持续 80%) 获得 SNS 告警，然后采取行动  
- CloudWatch 默认采集 EC2 的 CPU 利用率，NetworkIn/Out 情况，不采集 EC2 memory usage, disk swap 信息  
- 可以通过安装 CloudWatch Agent 来采集对应信息  

# Cloudtrail
- AWS 的资源操作，实际上都是 API call，或者通过 SDK, mgmt console, AWS CLI 执行的操作   
- Events are typically updated in CloudTrail **within 15 minutes** after an API call  
- 可以通过多种方式来过滤 Cloudtrail  
- AWS global services 的 Cloudtrail 会保存到 us-east-1 region  
- Cloudtrail -- Trails -- Insights events, 可以针对 Trails 打开 insights，实现异常情况下自动提醒。  
- 用户可以自己配置 Cloudtrail trails  
  - Cloudtrail 是保存在 AWS internal S3 桶的，Cloudtrail Eventhistory 会保持最近 90 天记录；若客户需要保持更久的记录，需要自行配置 Trails  
  - Cloudtrail 可以记录对于大部分 AWS 资源操作的记录，不过并不包括类似于 S3 upload object（可以配置 Cloudtrail trails 来记录） 
  - [AWS CLI 配置 Trails](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-create-and-update-a-trail-by-using-the-aws-cli.html)，需要注意`--is-multi-region-trail`(default: false) 参数，以及`--include-global-service-events`(default: true)    

## AWS Global Services
[Global services](https://jayendrapatil.com/aws-global-vs-regional-vs-az-resources/) 的特点：  
- service 在所有 AWS region 可用  
- global service 相关的 API call 的 Cloudtrail 会记录到 US EAST(N.Virginia, us-east-1)  
- regional service 的 API call，Cloudtrail 都在本 region  

IAM  
Route 53  
WAF  
Cloudfront  
Global Accellerator  
PHD  

__Regional__  
VPC  
VPC Endpoint  
TGW  
EC2  
ELB  
S3(data is regional)  
DynamoDb  

# EventBridge
- 根据特定 event，触发后续的联动操作  
- 举个栗子，比如 S3 上传图片后应该使用 ECS 做处理，根据 PUT object 应该增加 ECS task 数量，就可以借助 EventBridge 捕捉 S3 created object 事件，然后 target 选择 ECS，可以设置 task defination, task 数量；等到执行任务结束，object 被删除，应该减少 ECS task 数量，可以用 EventBridge 捕捉 S3 remove object，联动 ECS  

![Event pattern](/assets/img/post-EventBridge-Event-pattern.png)  
![target](/assets/img/post-EventBridge-target.png)  

# Trusted Advisor
Optimize Performance and Security, monitor Fault tolerance, cost, Service limits, **based on AWS best practice**  
优化性能、安全、成本、容错、服务限额  
[需要购买 support plan 才能解锁](https://aws.amazon.com/premiumsupport/plans/?nc1=h_ls),developer plan 解锁一部分，business 解锁全部  
![Trusted_Advisor_Dashboard](assets/Monitoring%20and%20Analysis/IMG_20220414-154646933.png)  

# Cost Explore
- 费用相关  

# Athena

# Kinesis
__streaming data__, 持续产生的数据，来源可能是数千个不同的 sources，比如购物网站、游戏、股票价格、社交网络、IOT 传感器数据  
用户可以把 streaming data 发送到 Kinesis  

## Kinesis 三种分类 

Kinesis Streams 数据流  
- 大量数据，IOT 场景  
- client/data producers -- `Kinesis Streams`(store in Shard, 24 hours - 7 days retention) ---> Consumers --> DDB, S3, EMR, Redshift  

---

Kinesis Firehose 数据流接收、处理  
- client/data producers -- `Kinesis Firehose`(Lambda to process data…) --> S3/Elasticsearch --> Redshift  
- data-ingestion  

![Kinesis Firehose](/assets/img/post-Kinesis Firehose.png)   

A manufacturing company wants to implement predictive maintenance on its machinery equipment. The company will install thousands of IoT sensors that will send data to AWS in real time. A solutions architect is tasked with implementing a solution that will receive events in an ordered manner for each machinery asset and ensure that data is saved for further processing at a later time.
<details>
  <summary>Which solution would be MOST efficient?</summary>

Use Amazon Kinesis Data Streams for real-time events with a partition for each equipment asset. Use Amazon Kinesis Data Firehose to save data to Amazon S3.
</details>

---

Kinesis Analytics 数据分析  
- client/data producers -- `Kinesis Ayalytics`(process data on Consumer or in-flight) --> DDB, S3, EMR, Redshift  

[Kinesis streams vs. Firehose](https://www.sumologic.com/blog/kinesis-streams-vs-firehose/)  

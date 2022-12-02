---
layout:         post
title:          AWS 监控、管理、分析，以及数据分析
subtitle:		    备考 SAA, SAP 的笔记
date:           2022-11-12
author:         Luke
cover:          '/assets/img/bg-monitor-and-analysis.png'
tags:           AWS, SAA, CloudWatch, Cloudtrail, EventBridge, Trust Advisor, Athena, Kinesis
---
- [PHD(Personal Health Dashboard)](#phdpersonal-health-dashboard)
- [CloudWatch](#cloudwatch)
  - [CW Alarms](#cw-alarms)
  - [CW Metrics](#cw-metrics)
  - [CW Logs, Logs Insight](#cw-logs-logs-insight)
  - [CW Insights for Lambda, Container, Contributor](#cw-insights-for-lambda-container-contributor)
  - [X-Ray trace](#x-ray-trace)
- [Cloudtrail](#cloudtrail)
  - [AWS Global Services](#aws-global-services)
- [AWS Config](#aws-config)
- [AWS Service Catalog](#aws-service-catalog)
- [CloudFormation CFN](#cloudformation-cfn)
  - [CFN StackSets 堆栈集](#cfn-stacksets-堆栈集)
  - [CFN Stack Drift Detection // 堆栈集资源的配置发生了变化](#cfn-stack-drift-detection--堆栈集资源的配置发生了变化)
- [EventBridge](#eventbridge)
- [Trusted Advisor](#trusted-advisor)
- [Cost Explore](#cost-explore)
- [Athena](#athena)
- [Kinesis](#kinesis)
  - [Kinesis 三种分类](#kinesis-三种分类)

# PHD(Personal Health Dashboard)
- 查看 EC2, Direct Connect，RDS 等维护通知  
- 其他一些由 AWS 发起的维护事件  

# CloudWatch
- 将 AWS 各个 services 相关的 metrics/指标，logs/日志 集中起来，方便查看，以及做一些 insights 分析  
- 通过创建 CloudWatch Alert，在特定情况下（比如 EC2 CPU 利用率 5 分钟内持续 80%) 获得 SNS 告警，然后采取行动  
- CloudWatch 默认采集 EC2 的 CPU 利用率，NetworkIn/Out 情况，不采集 EC2 memory usage, disk swap 信息  
- 可以通过安装 CloudWatch Agent 来采集对应信息  

## CW Alarms
- based on Metrics to set Alarms, if triggered then you see Alerts  

## CW Metrics
![post-CW-metrics-looks-like](/assets/img/post-CW-metrics-looks-like.ong)
![post-CW-how-to-read-CW-example](/assets/img/post-CW-how-to-read-CW-example.png)
> [题目参考文档](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/cloudwatch_concepts.html#Statistic)

## CW Logs, Logs Insight

## CW Insights for Lambda, Container, Contributor

## X-Ray trace

# Cloudtrail
- AWS 的资源操作，实际上都是 API call，或者通过 SDK, mgmt console, AWS CLI 执行的操作   
- Events are typically updated in CloudTrail **within 15 minutes** after an API call  
- 可以通过多种方式来过滤 Cloudtrail  
- AWS global services 的 Cloudtrail 会保存到 us-east-1 region  
- Cloudtrail -- Trails -- Insights events, 可以针对 Trails 打开 insights，实现异常情况下自动提醒。  
- 用户可以自己配置 Cloudtrail trails  
  - Cloudtrail 是保存在 AWS internal S3 桶的，默认开启 SSE-S3 加密；客户配置 Trails 时候，默认开启了 SSE-KMS，客户指定 KMS
  - 通过 log file **integrity validation **来校验 Trails log 没有被修改过，Trails 配置选项 "Log file validation"，默认打开
  - Cloudtrail Eventhistory 会保持最近 90 天记录；若客户需要保持更久的记录，需要自行配置 Trails  
  - Cloudtrail 可以记录对于大部分 AWS 资源操作的记录，不过并不包括类似于 S3 upload object（可以配置 Cloudtrail trails 来记录） 
  - [AWS CLI 配置 Trails](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-create-and-update-a-trail-by-using-the-aws-cli.html)，需要注意`--is-multi-region-trail`(default: false) 参数，以及`--include-global-service-events`(default: true)    
  - 
- An AWS CloudTrail log file provide the following details.
  * Identity of the API caller
  * Time of the API call
  * Source IP address of the API caller
  * Request parameters
  * Response elements

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
DXGW  
PHD  

__Regional__  
VPC  
VPC Endpoint  
Route 53 Resolver Inbound/Outbound Endpoint  
TGW  
EC2  
ELB  
S3(data is regional)  
DynamoDb  

# AWS Config
[AWS Config Tutorial](https://www.youtube.com/watch?v=qHdFoYSrUvk)  
- [AWS Config](https://aws.amazon.com/cn/config/faq/) 是 AWS 托管服务，提供 AWS 资源库存、配置历史记录（保存到 S3 桶）和配置更改通知 (SNS)，以确保安全性和方便管理，near real-time, post/after-provisioning
- 借助 AWS Config，您可以找到现有的 AWS 资源，导出 AWS 资源的完整库存清单与所有配置详细信息，并确定在任何时间点上配置资源的方式  
- automatic remediation/修正 with Config rules，比如 security　group 不合规，可以自动修正（借助 SSM automation documents)
- conformance packs(organization-wide 推送，member account 无法 disable、删除 rules)
- custom cfg items（AWS 之外的，比如 AD 域的配置等）
- 这些功能提供了合规性审计、安全分析、资源更改跟踪和故障排除，顺便还统计了 account 下有多少资源
- 很好用，不过 [费用](https://www.amazonaws.cn/en/config/pricing/) 可能比较高
![post-AWS-Config-how-to-and-why](/assets/img/post-AWS-Config-how-to-and-why.png)  
- [可以通过 SNS, EventBridge 等发送通知](https://docs.aws.amazon.com/config/latest/developerguide/monitoring.html)
  - 可以通过 SNS --> Endpoint SQS 的方式，筛选感兴趣的 changes，比如 security group 的变化，忽略某些 changes 比如 EC2 tags
  ![post-Config-filter-specify-changes-example](/assets/img/post-Config-filter-specify-changes-example.png)
  - EventBridge 可以和 Config 联动，比如基于 event/ruls 的通知，take corrective action；EventBridge 可以和 lambda 联动
- 有很多 aws 已经定义的规则，比如 CFN-drift，也可以通过 lambda 定义规则，比如检查是否所有 EBS 都是 gp3
![AWS Config SAA example](/assets/img/post-AWS-Config-SAA.png)  
![post-AWS-Config-automate-audit-cfg-changes](/assets/img/post-AWS-Config-automate-audit-cfg-changes.png)  

# AWS Service Catalog
- [create, share, organize, and govern your IaC templates](https://aws.amazon.com/servicecatalog/)
- 在符合相关规定的情况下，加速 IaC 的部署
- 和 **ServiceNow**, **Jira Service Management** 整合，实现 developers 的 self-service，不需要每次找到 admin 去申请
  - 实现方式，是预先定义 CFN template 给 Service Catalog 使用，预定义合规的资源类型、要求（比如只允许 t2.micro type)；admin 将定义好的 **Catalog Product** 添加到 **Catalog portfoils**，用来限制 user 自助服务一定是合规的
  - user 通过 Service Catalog 发起**自助请求**（比如 launch EC2，不直接通过 EC2 console，可以不给相关权限）
  - Service Catalog 和 SSM run commands 整合，允许 user 做一些操作，比如 reboot EC2

|                          | AWS Config                         | Service Catalog                   |
| ------------------------ | ---------------------------------- | --------------------------------- |
| how to ensure compliance | Detective controls, post-provision | Preventive controls, provisioning |
| integration with         | AWS Systems Manager/SSM            | AWS CloufFormation/CFN            |

# CloudFormation CFN

## CFN StackSets 堆栈集
- AWS [CloudFormation StackSets](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/what-is-cfnstacksets.html) extends the capability of stacks by enabling you to create, update, or delete stacks across multiple accounts and AWS Regions with a single operation.  
- 在单次操作中跨多个账户、region 场景、更新、删除 stack；通过 admin account，定义 CFN template，用模板来定义具体操作（比如创建 IAM role），应用到指定 target accounts  
![post-CFN-StackSets-how-it-works](/assets/img/post-CFN-StackSets-how-it-works.png)  
![post-CFN-StackSets-SAP-example](/assets/img/post-CFN-StackSets-SAP-example.png)  

## CFN Stack Drift Detection // 堆栈集资源的配置发生了变化
- 可以通过 check stack drift 来观察 CFN 创建的资源，相关配置没通过 CFN 做了修改
- AWS Config 有一条 rule `cloudformation-stack-drift-detection-check` 来做相关检查
![post-Config-CFN-drift-detection-check](/assets/img/post-Config-CFN-drift-detection-check.png)
![post-Config-CFN-drift-detection-check-example](/assets/img/post-Config-CFN-drift-detection-check-example.png)

# EventBridge
- 根据特定 event，触发后续的联动操作  
- 举个栗子，比如 S3 上传图片后应该使用 ECS 做处理，根据 PUT object 应该增加 ECS task 数量，就可以借助 EventBridge 捕捉 S3 created object 事件，然后 target 选择 ECS，可以设置 task defination, task 数量；等到执行任务结束，object 被删除，应该减少 ECS task 数量，可以用 EventBridge 捕捉 S3 remove object，联动 ECS  

![Event pattern](/assets/img/post-EventBridge-Event-pattern.png)  
![target](/assets/img/post-EventBridge-target.png)  

# Trusted Advisor
Optimize Performance and Security, monitor Fault tolerance, cost, Service limits, **based on AWS best practice**  
优化性能、安全、成本、容错、服务限额  
[需要购买 support plan 才能解锁](https://aws.amazon.com/premiumsupport/plans/?nc1=h_ls),developer plan 解锁一部分，business 解锁全部  
![Trusted_Advisor_Dashboard](/assets/img/IMG_20220414-154646933.png)  

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

![Kinesis Streams SAA example](/assets/img/post-Kinesis-Streams-SAA.png)  

---

Kinesis Firehose 数据流接收、处理  
- client/data producers -- `Kinesis Firehose`(Lambda to process data…) --> S3/Elasticsearch --> Redshift  
- data-ingestion  

![Kinesis Firehose](/assets/img/post-Kinesis Firehose.png)   

![Kinesis Firehose SAA example](/assets/img/post-Kinesis-SAA.png)  
![Kinesis Firehose SAA example](/assets/img/post-Kinesis-Firehose-SAA.png)  
![post-Kinesis-Firehose-example1](/assets/img/post-Kinesis-Firehose-example1.png)  

---

Kinesis Analytics 数据分析  
- client/data producers -- `Kinesis Ayalytics`(process data on Consumer or in-flight) --> DDB, S3, EMR, Redshift  

[Kinesis streams vs. Firehose](https://www.sumologic.com/blog/kinesis-streams-vs-firehose/)  

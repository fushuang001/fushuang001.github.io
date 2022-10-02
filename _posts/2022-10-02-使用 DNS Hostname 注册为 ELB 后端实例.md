---
layout:         post
title:          使用 DNS Hostname 注册为 ELB 后端实例
subtitle:		可以做，但是不推荐
date:           2022-10-02
author:         Forest
cover:          '/assets/img/post-NLB-DNS-target.png'
tags:           AWS, ELB, target, DNS, RDS
---

# 相关的需求，架构
RDS 部署在 private subnets，正常情况下只能从 AWS VPC 内部访问；客户希望 RDS 可以被 public Internet 访问，  
客户提出来的一个实现方式，使用 RDS instance private IP address 注册为 NLB 后端实例，通过 NLB 来提供公网访问。  
public client -- NLB -- (internal, private to VPC)RDS  

# 实现方式
首先通过 RDS hostname 的 DNS 解析，获取 RDS instance private IP address；  
之后创建 NLB 目标组，将 RDS 通过 private IP 注册为后端实例；  
将目标组与 NLB 关联，NLB 监听 TCP 3306 端口，将流量转发到后端 RDS 实例；  
测试通过 NLB 访问 RDS  

# 以上方式存在的问题
主要问题有两个，和 RDS 服务有关。
首先 RDS 的 private IP 并不是固定的，可能会在维护事件、底层硬件故障中发生变化；  
若发生变化，如何及时获取通知？使用新的 IP 来注册为 NLB 后端实例，注册过程耗时较长；  

其次 NLB --> RDS 的健康检查/health check，有可能达到 RDS 的`max_connect_errors`上限，从而导致 health check 失败，并且 RDS 产生报错：  
> Host is blocked because of many connection errors; unblock with 'mysqladmin flush-hosts'
[具体行为、解释可以参考](https://www.cnblogs.com/mask-xiexie/p/16174875.html)  

# 引申的需求
客户认为既然 RDS 的 IP 会发生变化，那么是否能直接使用 RDS DNS hostname 注册为 NLB 后端实例？  
由 NLB 解析最新的 RDS IP，若 IP 发生变化，自动指向新的 RDS 实例。  

# 实现方式
首先从 AWS 提供的方式来说，NLB 可以注册 EC2 instance id, ip address, ALB 类型的后端实例，不支持直接注册 DNS/Hostname 类型；  
类似的需求在 AWS 一篇 global blog 当中提到了可以使用 Eventbridge(crontab)，Lambda(python script) 来实现，参考[Hostname-as-Target for Network Load Balancers](https://aws.amazon.com/blogs/networking-and-content-delivery/hostname-as-target-for-network-load-balancers/)；  
Eventbridge 的 crontab 持续检查 RDS DNS Hostname 解析到的 IP 是否发生变化；  
若发生变化，Eventbridge 作为 Lambda trigger，调用 Lambda(python script) 将新的 IP 注册为 NLB 后端实例；  
可以借助 S3 桶来保存 IP address 的历史记录。  

# 存在的问题
首先依然是新注册 RDS IP 到 NLB 的延迟；  
其次是 Eventbridge 的考量，crontab 间隔越短那么发现 RDS IP 变化越及时，但是相对费用会高一些；   
再次是方案整体复杂度，虽说可以借助 CloudFormation 快速部署，不过学习成本比较高；  
如果准备采用类似方案，更简易自己在 Linux EC2 DIY 一套 python 脚本，借助 AWS IAM User 的 AKSK 来实现 NLB 相关的操作，借助 crontab 来实现快速监控；  
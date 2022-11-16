---
layout:         post
title:          AWS Networking and Content Delivery
subtitle:		    备考 ANS(Advanced Networking - Specialty) 的笔记
date:           2022-11-15
author:         Luke
cover:          '/assets/img/bg-AWS-Networking-CDN.png'
tags:           AWS, Networking, Content Delivery, VPC, Cloudfront, Route 53, ELB
---
- [VPC](#vpc)
  - [EC2 bandwidth](#ec2-bandwidth)
    - [Enhanced networking - ENA](#enhanced-networking---ena)
  - [prefix-list](#prefix-list)
  - [VPC Endpoint, Endpoint Services, PrivateLink](#vpc-endpoint-endpoint-services-privatelink)
  - [VPN](#vpn)
- [Transit Gateway](#transit-gateway)
- [ELB 对比](#elb-对比)
- [ALB](#alb)
- [NLB](#nlb)
- [GWLB](#gwlb)
- [Direct Connect DX 专线](#direct-connect-dx-专线)
  - [VIF 分类和使用场景](#vif-分类和使用场景)
  - [one DX access to multiple US regions](#one-dx-access-to-multiple-us-regions)
  - [路由控制](#路由控制)
- [Route53 DNS](#route53-dns)
  - [R53 DNS 解析的优先级](#r53-dns-解析的优先级)
  - [DNSSEC DNS 安全扩展](#dnssec-dns-安全扩展)
  - [R53 Resolver DNS Firewall](#r53-resolver-dns-firewall)
  - [Split-view DNS，对内对外提供不同服务](#split-view-dns对内对外提供不同服务)
  - [R53 Resolver](#r53-resolver)
    - [Inbound Resolver Endpoint](#inbound-resolver-endpoint)
    - [Outbound Resolver Endpoint](#outbound-resolver-endpoint)
    - [Query logging](#query-logging)
- [Global Accelerator](#global-accelerator)
  - [Global Accelerator vs Cloudfront](#global-accelerator-vs-cloudfront)
- [Cloudfront](#cloudfront)
  - [Versioning](#versioning)
  - [Customing with edge functions 边缘函数](#customing-with-edge-functions-边缘函数)
    - [Cloudfront Functions](#cloudfront-functions)
    - [Lambda@Edge](#lambdaedge)
- [API Gateway](#api-gateway)
- [Troubleshooting Tools](#troubleshooting-tools)
  - [DNS](#dns)
  - [packets capture & analysis](#packets-capture--analysis)
  - [SSL/TLS](#ssltls)
  - [trace forwarding path](#trace-forwarding-path)
  - [Connections](#connections)
  - [performance, concurrent requests](#performance-concurrent-requests)

[AWS 考试预约、培训等资料](https://aws.amazon.com/certification/certified-advanced-networking-specialty/)  
[考试大纲，查漏补缺](https://d1.awsstatic.com/training-and-certification/docs-advnetworking-spec/AWS-Certified-Advanced-Networking-Specialty_Exam-Guide.pdf)  

# VPC
![VPC Sharing](/assets/img/IMG_20220504-212047378.png)  
![post-VPC-pricing-SAA](/assets/img/post-VPC-pricing-SAA.png)  

## EC2 bandwidth
- [network bandwidth available to an EC2 instance depends on several factors](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-network-bandwidth.html)  
- 同 region 内，Traffic can utilize the full network bandwidth available to the instance  
- 跨 region，或者去 IGW, DX 等，当前一代实例 with a minimum of 32 vCPUs 可以用 50% of the network bandwidth；如果不足 32 vCPUs，limited to 5 Gpbs.  
- 相同五元组的流量，若 EC2 不在相同 cluster placement group，limited to 5 Gpbs; 若在相同 cluster placement group，up to 10 Gbps.  

### Enhanced networking - ENA
- 借助 `single root I/O virtualization (SR-IOV)` 来提供高性能网络，没有额外费用  
  - 当前一代 Nitro 实例支持 enhanced networking  
    - 需要 Kernel module (ena) 支持，查看`modinfo ena`  
    - 需要 Instance attribute (enaSupport)  
    - 需要 Image attribute (enaSupport)  
    - 需要特定 Network interface driver，查看`ethtool -i eth0`  
      - 若输出 driver: ena，表示`ena` module is loaded  
      - 若输出 driver: vif，表示`ena` module is not loaded  
  - 可以通过`Elastic Network Adapter (ENA)`或者`Intel 82599 Virtual Function (VF) interface`两种方式启用 enhanced networking  
- high-performance networking, higer bandwidth, higer packet per second(PPS), consistently lower inter-instance lateny  

![post-EC2-bandwidth-ENA](/assets/img/post-EC2-bandwidth-ENA.png)  

## prefix-list
- 通过 prefix-list 来包含多个 IP 地址/段，可以被 security-group，route-table 引用  
- prefix-list 的 Max entries 大小，占用 security-group 的空间；比如 Max entries = 10 但是只写了两个 entries，也会占用 10 个 SG 容量；Max entries 可以修改变大，不能变小  
- 可以通过 [AWS-managed prefix-list](https://docs.aws.amazon.com/vpc/latest/userguide/working-with-aws-managed-prefix-lists.html)，比如`com.amazonaws.region.s3` 动态获取 AWS service IP ranges 最新地址范围，目前支持 CF, DDB, S3, Ground Station  
![post-AWS-managed-prefix-list](/assets/img/post-AWS-managed-prefix-list.png)  

## VPC Endpoint, Endpoint Services, PrivateLink
S3 intf 走的是 private subnet/ip；gw 是 public ip

## VPN

# Transit Gateway

# ELB 对比
把之前做的表格添加过来  

# ALB

# NLB
![post-NLB-tcp-443-ssl-cert-target-ANS-example](/assets/img/post-NLB-tcp-443-ssl-cert-target-ANS-example.png)  
![post-NLB-register-IP-targets-PPv2-client-IP](/assets/img/post-NLB-register-IP-targets-PPv2-client-IP.png)  

# GWLB

# Direct Connect DX 专线

## VIF 分类和使用场景

## one DX access to multiple US regions
- on-prem 和某个 US regions VPC 建立 **dedicated** DX，[同一条 DX 可以打通 on-prem 与 US 其他 regions](https://aws.amazon.com/cn/blogs/aws/aws-direct-connect-access-to-multiple-us-regions/), DX inter-region capability     
- on-prem -- Direct Connect -- US region 1 -- AWS network -- US region 2，跨 region 的流量由 AWS 负责，路由表由 AWS 负责 (BGP 路由通告）    
![post-Direct-Connect-inter-region-capability](/assets/img/post-Direct-Connect-inter-region-capability.png)  

## 路由控制

# Route53 DNS
![Route53 failover-health-check SAA example](/assets/img/post-R53-HC.png)  
![post-R53-geo-SAA](/assets/img/post-R53-geo-SAA.png)  

## R53 DNS 解析的优先级

## DNSSEC DNS 安全扩展
- [Domain Name System Security Extensions](https://aws.amazon.com/about-aws/whats-new/2020/12/announcing-amazon-route-53-support-dnssec/?nc1=h_ls)，DNS 安全扩展，为 DNS 提供数据来源认证和数据完整性验证，满足 FedRAMP 等法规要求  
- public hosted zone 功能  
- Route 53 DNSSEC provides __data origin authentication__ and __data integrity verification__ for DNS and can help customers meet compliance mandates, such as FedRAMP.  
- 若在 hosted zone 启用 DNSSEC，R53 会以加密方式对 hosted zone 当中每条 record 进行签名；R53 管理区域签名密钥，用户在 KMS 的 CMK(Customer Managed Key) 管理密钥签名密钥  
  - CMK 是 asymmetric，ECC_NIST_P256  
  - 必须在 us-east-1 region  
- When you enable DNSSEC signing on a hosted zone, Route 53 cryptographically signs each record in that hosted zone. Route 53 manages the zone-signing key, and you can manage the __key-signing key(KSK)__ in AWS Key Management Service (AWS KMS).   
![post-R53-DNSSEC-Key-signing-key-cfg](/assets/img/post-R53-DNSSEC-Key-signing-key-cfg.png)  
![post-R53-DNSSEC-KSK-CMK-ANS-example](/assets/img/post-R53-DNSSEC-KSK-CMK-ANS-example.png)  

## R53 Resolver DNS Firewall 
- monitor and control the domains that applications within your VPCs can access   
- supports allow lists or deny lists to filter the set of domains that you can use
- can effectively prevent the use of DNS queries to exfiltrate data 防范 [域名解析泄露](https://www.infoblox.com/dns-security-resource-center/dns-security-issues-threats/dns-security-threats-data-exfiltration/)    
  - DNS tunneling 的一种应用，重点在于将数据送出去，不一定得到 DNS response   
  - ![post-R53-DNS-exfiltrate-how-it-works](/assets/img/post-R53-DNS-exfiltrate-how-it-works.png)   

## Split-view DNS，对内对外提供不同服务
- 假设 example.com 对外、对内提供的服务不同，可以通过 R53 配置两个 hosted zone 都叫做 example.com，一个 public 对外，一个 private 关联 VPC 对内  
- VPC 内设备会优先参考 PHZ(Private Hosted Zone) 做解析  
![post-R53-split-view-dns-ANS-example](/assets/img/post-R53-split-view-dns-ANS-example.png)  

## R53 Resolver

### Inbound Resolver Endpoint

### Outbound Resolver Endpoint

### Query logging 
- public and private DNS query logging  
- [中国区暂时不支持 public DNS query logs](https://docs.amazonaws.cn/en_us/aws/latest/userguide/route53.html)  

# Global Accelerator 
- [Global Accelerator](https://aws.amazon.com/global-accelerator/?nc1=h_ls) 使用 AWS 全球骨干网，加速用户的访问  
- 降低延迟、抖动、丢包  
- 提供两个全球静态共有 IP(static public IP)  
- 后端可以是 ALB, NLB, EC2, 或者 EIP

![how it works](/assets/img/post-GA-how-it-works.png)  
![GA SAA example](/assets/img/post-GA-SAA.png)  

## Global Accelerator vs Cloudfront
                                                                                                                                                                                                   
|     | Global Accelerator    |Cloudfront|
| --- | --- |---|
|  共性   | 用户分布广泛，服务端希望提供低延迟服务    |用户分布广泛，服务端希望提供低延迟服务|
| 场景    |networking service to improve application's performance and availability(Regional failover, 多个 Endpoint 分布） for global users 低延迟、高可用，eg. game     |cloud distributed networking service for web applications that provides low latency and speed 用户分布广泛，访问内容有重复所以可以被 Pop 节点缓存，CF 可以缓存、压缩文件，降低 origin 压力；Edge Functions 提供一定的边缘计算能力|
| 实现方式    |client - GA --寻找最近的 endpoint；GA -- Endpoint 走 AWS backbone network；提供两个 static IP     |client -- CF Pop 缓存，miss cache then refer Origin；根据 clients 地理位置不同，Pop public IP 不同|
| 支持的协议    | TCP, UDP, HTTP, HTTPS, gRPC；通常用于 non-HTTP 场景比如 gaming, IoT, VoIP    |static & dynamic content, HTTP, HTTPS, WebSocket|
| security    |AWS Shield to prevent DDoS; if Endpoint is ALB then could integrate with WAF     |AWS Shield to prevent DDoS, WAF for additional protection against malicious traffic|
| 价格    |fixed hourly fee, Data Transfer-Premium     |Data Transfer Out, HTTP requests|

# Cloudfront

![CF SAA example](/assets/img/post-CF-SAA.png)  
![post-CF-SAA-example1](/assets/img/post-CF-SAA-example1.png)  
![post-CF-S3-OAI-SAA](/assets/img/post-CF-S3-OAI-SAA.png)  
![post-CF-S3-OAI-WAF-SAA](/assets/img/post-CF-S3-OAI-WAF-SAA.png)  
![post-CF-geo-restriction-example](/assets/img/post-CF-geo-restriction-example.png)  

## Versioning
- 提供多个版本的文件，可以控制哪一个版本的文件可以被用户访问  
- 如果通过 invalidate 操作，由于 client 端、ISP 等缓存，用户看到的可能还是过期版本  
- versioning 比 invalidate 便宜，invalidate 操作收费，versioning 只是收取将 file 传递到 Edge 的费用  
- CF access log 包含文件版本号，方便查看、排错  

## Customing with edge functions 边缘函数
- 用户自定义代码，在 CF edge 运行，针对 HTTP request、respond 自定义   
- 用户只需要关心部署到 CF 的代码/函数 (coding)，不需要关心 coding 所运行底层环境，按用量付费  
- CF 提供两种方式 to [write and manage edge functions](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/edge-functions.html)  

### Cloudfront Functions
- lightweight functions in JavaScript；轻量级，CF 原生支持  
- for high-scale, latency-sensitive CDN customizations；大规模、延迟敏感  
- offers submillisecond start times, scales immediately to handle millions of requests per second；1 毫秒内启动，快速扩容适应百万级请求量  
- 适用场景  
  - cache key normalization；通过操作 HTTP 属性比如 header，cookie 等，提高缓存命中率  
  - header manipulation； 在 request, response 中插入、修改、删除 HTTP header   
  - URL redirects or rewrites  
  - request authorization  

![post-CF-edge-CF-Functions](/assets/img/post-CF-edge-CF-Functions.png)  

### Lambda@Edge
- [Lambda 需要部署在 us-east-1 region](https://www.stormit.cloud/blog/lambda-at-edge/)，Node.js, Python  
- 在接近 viewer 的 edge location 执行，[实际上是 REC(Regional Edge Cache)](https://aws.amazon.com/cn/blogs/aws/introducing-cloudfront-functions-run-your-code-at-the-edge-with-low-latency-at-any-scale/)，并不是 Pop 节点  
- 适用场景
  -  inspect cookie and rewrite URLs so users see different versions of a site for A/B testing；检查 cookie 并且重写 URL，对用户提供不同测试版本  
  -  检查 User-Agent header 来基于 viewer 使用设备 (mobile phone, computer) 来提供适合尺寸、大小的 object  

![post-CF-edge-LambdaEdge](/assets/img/post-CF-edge-LambdaEdge.png)  
![post-CF-edge-Lambda-at-Edge-ANS-example](/assets/img/post-CF-edge-Lambda-at-Edge-ANS-example.png)  

# API Gateway

# Troubleshooting Tools

## DNS
dig  
nslookup  
whois  

## packets capture & analysis
tcpdump  
tshark(Linux)  
wireshark  

## SSL/TLS
openssl  
sslscan  

## trace forwarding path
mtr  
tracert  
tracerout  

## Connections
netstat  
ss  
telnet  
ssh  
iptables  
firwared  
ufw  
curl  
wget  
netcat  
wscat  
nping  
hping3  
tcpping  

## performance, concurrent requests
ab  
apache jMeter  
iPerf  
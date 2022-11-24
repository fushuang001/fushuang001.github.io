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
  - [AmazonProvidedDNS](#amazonprovideddns)
  - [Subnet](#subnet)
    - [CIDR Reservation](#cidr-reservation)
    - [IPv6 - DNS64 settings](#ipv6---dns64-settings)
  - [Route Table](#route-table)
    - [Route Table 优先级](#route-table-优先级)
    - [Route Table propagation](#route-table-propagation)
  - [IPAM - IP Address Manager](#ipam---ip-address-manager)
  - [Network Analysis](#network-analysis)
  - [EIGW - Egress-Only Internet Gateway](#eigw---egress-only-internet-gateway)
  - [EC2 bandwidth](#ec2-bandwidth)
    - [Enhanced networking - ENA](#enhanced-networking---ena)
    - [Elastic Fabric Adapter - EFA](#elastic-fabric-adapter---efa)
  - [prefix-list](#prefix-list)
  - [VPC Endpoint, Endpoint Services, PrivateLink](#vpc-endpoint-endpoint-services-privatelink)
  - [VPN](#vpn)
  - [Transit VPC](#transit-vpc)
  - [VPC flowlog](#vpc-flowlog)
  - [VPC Traffic Mirroring](#vpc-traffic-mirroring)
- [Transit Gateway](#transit-gateway)
  - [TGW flowlog](#tgw-flowlog)
  - [TGW attachment types](#tgw-attachment-types)
  - [TGW Network Manager](#tgw-network-manager)
    - [Route Analyzer](#route-analyzer)
- [ELB 对比](#elb-对比)
- [ALB](#alb)
  - [Listener Rule condition types](#listener-rule-condition-types)
  - [Stick session](#stick-session)
- [NLB](#nlb)
- [GWLB](#gwlb)
- [Direct Connect DX 专线](#direct-connect-dx-专线)
  - [DXGW](#dxgw)
  - [VIF 分类和使用场景](#vif-分类和使用场景)
  - [DX 故障排查](#dx-故障排查)
  - [data encrypt](#data-encrypt)
    - [MACsec](#macsec)
  - [one DX access to multiple US regions](#one-dx-access-to-multiple-us-regions)
  - [access a remote AWS region](#access-a-remote-aws-region)
  - [路由控制、优先级](#路由控制优先级)
    - [Public VIF](#public-vif)
    - [Private VIF](#private-vif)
    - [on-prem 视角](#on-prem-视角)
    - [Active/Passive 路由](#activepassive-路由)
    - [symmetry of flow](#symmetry-of-flow)
- [VPN](#vpn-1)
  - [Site-to-Site VPN](#site-to-site-vpn)
  - [Client VPN](#client-vpn)
- [Route53 DNS](#route53-dns)
  - [R53 DNS 解析的优先级](#r53-dns-解析的优先级)
  - [R53 支持的 DNS 类型](#r53-支持的-dns-类型)
    - [alias vs. CNAME](#alias-vs-cname)
  - [R53 DNS routing policy](#r53-dns-routing-policy)
  - [DNSSEC DNS 安全扩展](#dnssec-dns-安全扩展)
  - [R53 Resolver DNS Firewall](#r53-resolver-dns-firewall)
  - [Split-view DNS，对内对外提供不同服务](#split-view-dns对内对外提供不同服务)
  - [DNS resolution bw on-prem and AWS using AWS Directory Service](#dns-resolution-bw-on-prem-and-aws-using-aws-directory-service)
  - [R53 Resolver](#r53-resolver)
    - [Inbound Resolver Endpoint](#inbound-resolver-endpoint)
    - [Outbound Resolver Endpoint](#outbound-resolver-endpoint)
    - [Public DNS Query logging for Public Hosted Zone](#public-dns-query-logging-for-public-hosted-zone)
    - [Resolver query logging for Private Hosted Zone](#resolver-query-logging-for-private-hosted-zone)
  - [R53 Health Check](#r53-health-check)
- [Global Accelerator](#global-accelerator)
  - [Global Accelerator vs Cloudfront](#global-accelerator-vs-cloudfront)
- [Cloudfront](#cloudfront)
  - [secure access and restrict access to content](#secure-access-and-restrict-access-to-content)
    - [Signed URL, Signed cookes](#signed-url-signed-cookes)
    - [restrict access to S3 origin, OAC and OAI](#restrict-access-to-s3-origin-oac-and-oai)
    - [field-level encryption](#field-level-encryption)
    - [Geographically restrict](#geographically-restrict)
  - [Versioning](#versioning)
  - [Optimizing caching and availability 提高缓存命中率](#optimizing-caching-and-availability-提高缓存命中率)
  - [Customing with edge functions 边缘函数](#customing-with-edge-functions-边缘函数)
    - [Cloudfront Functions](#cloudfront-functions)
    - [Lambda@Edge](#lambdaedge)
  - [CF and edge function logging](#cf-and-edge-function-logging)
- [API Gateway](#api-gateway)
  - [Deployment types/部署类型](#deployment-types部署类型)
  - [Security](#security)
  - [API Caching, throttle](#api-caching-throttle)
  - [Same Origin Policy](#same-origin-policy)
    - [CORS 简单介绍](#cors-简单介绍)
  - [Monitor, Metrics, Logs](#monitor-metrics-logs)
- [Troubleshooting Tools](#troubleshooting-tools)
  - [DNS](#dns)
  - [packets capture \& analysis](#packets-capture--analysis)
  - [SSL/TLS](#ssltls)
  - [trace forwarding path](#trace-forwarding-path)
  - [Connections](#connections)
  - [performance, concurrent requests](#performance-concurrent-requests)

[AWS 考试预约、培训视频、白皮书](https://aws.amazon.com/certification/certified-advanced-networking-specialty/)  
[Network Learning Plan，完整的学习路径](https://explore.skillbuilder.aws/learn/public/learning_plan/view/89/networking-learning-plan?la=sec&sec=lp)  
[AWS 官方讲解考试大纲，把书给读薄了](https://explore.skillbuilder.aws/learn/course/14434/exam-prep-advanced-networking-specialty-ans-c01)  
注意 module 3 里面有一些问题，已经提交 feedback  
Domain 3: Network Management and Operations  
True or False?  
The Route Analyzer feature analyzes routes in transit gateway route tables only.  
  
[The Route Analyzer analyzes routes in transit gateway route tables only. It does not analyze routes in VPC route tables or in your customer gateway devices.](https://docs.aws.amazon.com/vpc/latest/tgwnm/route-analyzer.html)   

What is Amazon Route 53 Resolver Outbound Endpoint?  
A feature you can enable in Amazon Route 53 that cryptographically signs each record in the hosted zone. This ensures that the DNS responses have not been tampered with during transit  

[the question should be DNSSEC](https://aws.amazon.com/about-aws/whats-new/2020/12/announcing-amazon-route-53-support-dnssec/?nc1=h_ls)  

[TD, tutorialsdojo，总结、exam dumps](https://tutorialsdojo.com/aws-certified-advanced-networking-specialty-exam-study-path-guide-ans-c01/)  
[exam dumps，答案并不全对，需要自己判断](https://www.awslagi.com/aws-certified-advanced-networking-specialty-practice-exam/)  
[exampracticetests，很多题的答案有问题，推荐用 Bing 搜索题干或者某个选项内容，查看 ExamTopics 的讨论](https://exampracticetests.com/aws/Advanced_Networking-Specialty_ANS-C00/)  

# VPC
![VPC Sharing](/assets/img/IMG_20220504-212047378.png)
![post-VPC-pricing-SAA](/assets/img/post-VPC-pricing-SAA.png)  

## AmazonProvidedDNS
- VPC CIDR plus 2, eg. 10.0.0.2  
- enableDnsHostNames, enableDnsSupport  

## Subnet
- [one subnet could only in one specific AZ](https://docs.aws.amazon.com/vpc/latest/userguide/configure-subnets.html)  
- could ipv6 only  
- *Auto-assign IP settings*: public IP enalbe or not  
- *Resource-based Name (RBN) settings*: the hostname type for EC2 instances in the subnet, how DNS A/AAAA record queries are handled  
- minimum /28, maximum /16  
- network ACL 在 subnet 级别生效  

### CIDR Reservation
- prevent AWS from automatically assigning IPv4 or IPv6 addresses within a CIDR range you specify   

### IPv6 - DNS64 settings
- 对于 IPv6 only 的 subnet，如果需要和 IPv4 的资源通信，需要开启 subnet setting 的 *DNS64*  
- 当 IPv6 only 请求 IPv4 DNS 信息，对应的 IPv4 DNS 会被 DNS64 添加 64:ff9b::/96 前缀，假装是 IPv6  
- DNS 解决之后，IPv6 --> IPv4 发送报文，为了把 IPv6 转换为 IPv4，需要 NAT-GW 的 *NAT64*（自动开启）功能，手动添加路由    
- IPv6 的 DNS 解析，依赖 RBN  

![post-VPC-subnet-RBN-cfg](/assets/img/post-VPC-subnet-RBN-cfg.png)
![post-VPC-subnet-DNS64-howto](/assets/img/post-VPC-subnet-DNS64-howto.png)
![post-VPC-NAT64-DNS64](/assets/img/post-VPC-NAT64-DNS64.png)  

## Route Table
- 每个 subnet 只能有一个 route table  
- [Edge association](https://docs.amazonaws.cn/en_us/vpc/latest/userguide/VPC_Route_Tables.html) 一般用在 GWLB/security appliance 环境，关联 IGW  

### Route Table 优先级
- local 优先级最高，比如 local CIDR 10.0.0.0/16，那么即使用户有 LPM 10.0.0.0/24 static，也还是 local 优先  
- 有一个例外，GWLB endpoint 环境，Internet --> VPC 的流量，用 LPM 先送去 GWLBe 做安全检查
![post-RT-priority-VPC](/assets/img/post-RT-priority-VPC.png)
![post-RT-priority-VPC--LPM-GWLBe](/assets/img/post-RT-priority-VPC--LPM-GWLBe.png)  

### Route Table propagation
- 创建 VGW  
- 将 VGW attach to VPC（需要等待大约 5 分钟，状态更新为 Attached)  
- 修改 route table 条目，指定 VGW 作为特定路由条目的 target/destination  
- Eidt route table，可以打开 propagation, allows a virtual private gateway to automatically propagate routes to the route tables. This means that you don't need to manually enter VPN routes to your route tables   
![post-VPC-RT-propagation](/assets/img/post-VPC-RT-propagation.png)
![post-VPC-VGW-propagation-example](/assets/img/post-VPC-VGW-propagation-example.png)  

## IPAM - IP Address Manager
- manage all the CIDR blocks that are used in an account or across an organization in AWS Organization  
- to avoid CIDR overlap  
- CloudFormation can request a CIDR block from IPAM  
- for IPv4 & IPv6  
- with AWS *[CloudFormation custom resource](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-cfn-customresource.html)* and Lambda invocation, you could allocate IP when create VPC, and reclaim the VPC's IP allocation when VPC is deleted.
  - CloudFormation Custom resources provide a way for you to write custom provisioning logic in CloudFormation template and have CloudFormation run it during a stack operation, such as when you create, update or delete a stack
![post-VPC-IPAM-CFN-custom-resource-example](/assets/img/post-VPC-IPAM-CFN-custom-resource-example.png)
![post-VPC-IPAM-example](/assets/img/post-VPC-IPAM-example.png)  

## Network Analysis
- **Reachability Analyzer**  
  - 主要作用是检测 source、destination 流量不通时候，[security group、ACL、route table、ELB 等配置问题](https://docs.aws.amazon.com/vpc/latest/reachability/what-is-reachability-analyzer.html)   
  - a configuration analysis tool that enables you to *perform connectivity testing* between a source and destination resources *inside your VPCs*
  - verify that your network cfg matches your intended connectivity  
  - automate the verification of your connectivity intent as your network cfg changes  
 
- **Network Access Analyzer**  
  - [主要作用是提高 AWS 云上资源的安全性，检测 network access 是否合规](https://docs.aws.amazon.com/vpc/latest/network-access-analyzer/what-is-vaa.html)  
  - identifie unintended network access to your resources on AWS 检测不应该被访问、扫描的服务  
![post-VPC-Network-Access-Analyzer-howto](/assets/img/post-VPC-Network-Access-Analyzer-howto.png)  

## EIGW - Egress-Only Internet Gateway
- for ipv6, 效果类似于 NAT-GW，VPC --> Internet 流量主动出，拒绝外部始发的流量  
- 和 NAT-GW 不同点在于，ipv6 并不区分 private, public 网段，所以说，Egress-only IGW 和 clients 都是放在 public subnet 的  
![post-VPC-Egress-Only-IGW-example](/assets/img/post-VPC-Egress-Only-IGW-example.png)  

## EC2 bandwidth
- [network bandwidth available to an EC2 instance depends on several factors](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-network-bandwidth.html)  
- 同 region 内，Traffic can utilize the full network bandwidth available to the instance  
- 跨 region，或者去 IGW, DX 等，当前一代实例 with a minimum of 32 vCPUs 可以用 50% of the network bandwidth；如果不足 32 vCPUs，limited to 5 Gpbs.  
- 相同五元组的流量，若 EC2 不在相同 cluster placement group，limited to 5 Gpbs; 若在相同 cluster placement group，up to 10 Gbps.  

### Enhanced networking - ENA
- 借助 `single root I/O virtualization (SR-IOV)` 来提供高性能网络，没有额外费用  
  - 当前一代 Nitro 实例支持 enhanced networking  
    - 需要 Kernel module (ena) 支持，查看 `modinfo ena`  
    - 需要 Instance attribute (enaSupport)  
    - 需要 Image attribute (enaSupport)  
    - 需要特定 Network interface driver，查看 `ethtool -i eth0`  
      - 若输出 driver: ena，表示 `ena` module is loaded  
      - 若输出 driver: vif，表示 `ena` module is not loaded  
  - 可以通过 `Elastic Network Adapter (ENA)` 或者 `Intel 82599 Virtual Function (VF) interface` 两种方式启用 enhanced networking  
- high-performance networking, higer bandwidth, higer packet per second(PPS), consistently lower inter-instance lateny  

![post-EC2-bandwidth-ENA](/assets/img/post-EC2-bandwidth-ENA.png)  

### Elastic Fabric Adapter - EFA
- ENA provide traditional IP networking features that are required to support VPC networking   
- EFAs provide all of the same traditional IP networking features as ENAs, and they also [support OS-bypass capabilities](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/efa.html)  
- OS-bypass enables *HPC and machine learning applications* to bypass the operating system kernel and to communicate directly with the EFA device  
- low latency, high throught, and *instane could communicate directly with the network interface hardware*  
- 通常用于 HPC 高性能集群、机器学习场景，instance 可以 bypass OS kernel，直接和网卡硬件/EFA 通信    

![post-EC2-EFA](/assets/img/post-EC2-EFA.png)  

## prefix-list
- 通过 prefix-list 来包含多个 IP 地址/段，可以被 security-group，route-table 引用  
- prefix-list 的 Max entries 大小，占用 security-group 的空间；比如 Max entries = 10 但是只写了两个 entries，也会占用 10 个 SG 容量；Max entries 可以修改变大，不能变小  
- 可以通过 [AWS-managed prefix-list](https://docs.aws.amazon.com/vpc/latest/userguide/working-with-aws-managed-prefix-lists.html)，比如`com.amazonaws.region.s3` 动态获取 AWS service IP ranges 最新地址范围，目前支持 CF, DDB, S3, Ground Station  
![post-AWS-managed-prefix-list](/assets/img/post-AWS-managed-prefix-list.png)  

## VPC Endpoint, Endpoint Services, PrivateLink
S3 intf 走的是 private subnet/ip；gw 是 public ip

![post-VPC-Endpoint-STS-proxy-exampel](/assets/img/post-VPC-Endpoint-STS-proxy-exampel.png)  
[题目参考文档](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-proxy.html)  
>If you configure a proxy on an Amazon EC2 instance launched with an attached IAM role, ensure that you exempt the address used to access the instance metadata.   
>To do this, set the NO_PROXY environment variable to the IP address of the instance metadata service, 169.254.169.254. This address does not vary.

## VPN

## Transit VPC
- on-prem 与 [Transit VPC](https://docs.aws.amazon.com/whitepapers/latest/aws-vpc-connectivity-options/transit-vpc-option.html) 打通 VPN 连接，之后通过 Transit VPC，打通 on-prem 与其他 VPC 之间的连接  
- hub-spoke 模型提供 inter-VPC connectivity, hub/central VPC 通常使用`BGP over IPsec VPN` 连接 spoke VPC  
- Transit/hub/central VPC 使用 EC2 virtual Router 借助第三方 software appliances that route incoming traffic to their destinations using the VPN overlay    
- 优点是 hub/central 的 EC2 virtual router 可以提供 IPS/WAF 等功能；hub-spoke 设计相对简单，transitive routing enabled using the overlay VPN network  
- 缺点是 third-party vendor virtual appliances on EC2 费用高（实际上 TGW 费用也不低啊），VPN connection 吞吐量有限 (up to 1.25 Gbps per VPN tunnel)，手动配置、管理等   
- 所以 AWS 是比较推荐使用 TGW，[表格里面提供了 VPC peering(year 2014)、transit VPC(year 2016)，TGW(year 2018) 的对比](https://docs.aws.amazon.com/whitepapers/latest/building-scalable-secure-multi-vpc-network-infrastructure/transit-vpc-solution.html)  

![post-transit-vpc-how-it-design](/assets/img/post-transit-vpc-how-it-design.png)
![post-transmit-VPC-ANS-example](/assets/img/post-transmit-VPC-ANS-example.png)  

## VPC flowlog
- 三个层面可以配置，VPC, Subnet, ENI
- 默认格式，自定义格式（比如 EKS 环境，希望查看真实 src/dest packets IP)
- 保存到 S3 桶或者 CW Logs group，或者直接最为 producer 发送 stream data 到 Kinesis Firehose
- 聚合时间 1 分钟，或者 10 分钟
- [有一些 limitation](https://docs.aws.amazon.com/vpc/latest/userguide/flow-logs.html) 比如 DHCP, traffic mirror, 169.254 metadata, DNS, to NLB Endpoint Service 不会被记录
![post-VPC-flowlog-cfg](/assets/img/post-VPC-flowlog-cfg.png)
![post-VPC-flowlog-GuardDuty-example](/assets/img/post-VPC-flowlog-GuardDuty-example.png)

## VPC Traffic Mirroring
- target/destination:  
  - ENI  
  - NLB, UDP listener 4789  
  - GWLBe, layer3 GW  

# Transit Gateway
- centrally connect multiple same region(could cross account) VPCs  
- supports an MTU of 8500 bytes for traffic between VPCs, DX, TGW Connect, TGW peering  
- support an MTU of 1500 bytes for traffic over VPN  

## TGW flowlog

## TGW attachment types
- VPC  
- peering  
- Connect 
  - use GRE(Generic Routing Encapsulation) for higher bandwidth performance compared to a VPN connection, SD-WAN     
- VPN  

## TGW Network Manager
- [YouTube 视频](https://www.youtube.com/watch?v=xjH7GI95Pgg)  
- 主要作用是管理复杂的 global network，提高可见性，辅助快速定位故障，集中式监控、event 通知    
- [a service that provides a global view of your private network](https://aws.amazon.com/transit-gateway/network-manager/), allowing you to manage your AWS and on-premises resources and seamlessly integrate with your SD-WAN solutions  
- 集中式网络监控，Centralized Network monitoring for events, metrics to monitor the quality of your [global network](https://docs.aws.amazon.com/vpc/latest/cloudwan/cloudwan-concepts.html), both AWS and on-premises  
- 全球网络可见性，Global Network Visiblity  
- SD-WAN integration  

### Route Analyzer
- [perform an analysis of the routes](https://docs.aws.amazon.com/vpc/latest/tgwnm/route-analyzer.html) in your *TGW route tables only*, for specifid source & destination    
  - TGW have to register to Network Manager global network  
  - does not analyze routes in VPC route tables or in your customer gateway devices  
  - does not analyze security group rules or network ACL rules  
  - Free to use as part of AWS Transit Gateway Network Manager  
  - [example of HowTo](https://aws.amazon.com/blogs/networking-and-content-delivery/diagnosing-traffic-disruption-using-aws-transit-gateway-network-manager-route-analyzer/)  
- validate your existing route cfg  
- diagnose route-related issues that are causing traffic disruption in your global network  

# ELB 对比
把之前做的表格添加过来  

# ALB

## Listener Rule condition types
[参考文档](https://docs.amazonaws.cn/en_us/elasticloadbalancing/latest/application/load-balancer-listeners.html)  
- host-header
  - *.example.com
- http-header
  - [User-Agent](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/User-Agent)
    - The User-Agent request header is a characteristic string that lets servers and network peers identify the application, operating system, vendor, and/or version of the requesting user agent.
- http-request-method
  - GET
- path-pattern
  - /img/*
- query-string
  - Env:dev
- source-ip
  - 1.1.1.1

![post-ALB-Listener-rules](/assets/img/post-ALB-Listener-rules.png)
![post-ALB-Listener-rules-example](/assets/img/post-ALB-Listener-rules-example.png)

## Stick session
- [stickiness session 支持两种类型](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/sticky-sessions.html)，duration based 和 application based  
  - duration based // 基于持续时间   
    - cookie name `AWSALB`，无法修改；所以说，多层 ALB 场景，只有一层可以启用 duration based cookie  
  - application based // 基于应用程序  
    - cookie name per target group 不同  
- 如果 forward action 里面比如基于 weight，转发到多个 target group，而 target group 启用了 sticky，那么 ALB 层面必须启用 target group sticky，cookie name `AWSALBTG`  
- 以上针对 HTTP/HTTPS listener；WebSocket 天生具有 sticky 属性；client 请求协议升级，返回 HTTP 101 来接受协议升级的 target，在后续 WebSocket 中会被使用；WebSocket 升级后，不再参考基于 cookie 的 sticky  

![post-ALB-sticky-session-ALWALB-ANS-example](/assets/img/post-ALB-sticky-session-ALWALB-ANS-example.png)  

# NLB
![post-NLB-tcp-443-ssl-cert-target-ANS-example](/assets/img/post-NLB-tcp-443-ssl-cert-target-ANS-example.png)
![post-NLB-register-IP-targets-PPv2-client-IP](/assets/img/post-NLB-register-IP-targets-PPv2-client-IP.png)  

# GWLB

# Direct Connect DX 专线
[15 道 AWS 官方题库，DX 占比例很大](https://explore.skillbuilder.aws/learn/course/12676/aws-certified-advanced-networking-specialty-official-practice-question-set-ans-c01-english)  

![post-Direct-Connect-resilience-example](/assets/img/post-Direct-Connect-resilience-example.png)  

**申请步骤：**
- 提交申请 connection request
- 下载证书 LoA, Letter of Authorization
- 将 LoA 提供给 APN，拉线，物理层 UP（打环测试）
- 配置 VIF(virtual intf) 建立连接，只能使用单模光纤/SMF
- 在 on-premise，VPC 添加路由
- [如果 public VIF 状态卡在 verifying](https://aws.amazon.com/cn/premiumsupport/knowledge-center/public-vif-stuck-verifying/)  

**单模光纤/SMF(Physical, layer1)：**
- 用来传输信号的载体应该是激光，适合长距离、单模式 light 光信号
- 线缆的颜色一般是黄色、橙色
- 型号，举个例子
  - 1000BASE-LX transceiver/SFP for 1Gb
  - 10GBASE-LR transfer/SFP for 10Gb

## DXGW
- DXGW 只是用来打通 on-premise & VPC 之间通信链路，并不负责 VPC 之间互相通信    
  - 新功能 [SiteLink](https://aws.amazon.com/blogs/networking-and-content-delivery/introducing-aws-direct-connect-sitelink/)，创建 private/transit VIF 时候如果选择打开 *SiteLink*，就可以实现穿越 DXGW 通信  
  - DXGW 是 global 的，不区分 region；VGW 是 regional 的  
  - DXGW 类似于 redundant *BGP RR*，或者说 distributed VRF  
- on-premise --- DX --- ZHY VPC, VGW -- DXGW --- BJS VPC，可以通过 DXGW 打通 BJS, on-prem 的连接  
- DXGW -- TGW -- transit VIF -- VPC, if you connect to multiple transit gateways that are in different resions, use unique BGP ASNs for each transit gateway.  
![post-Direct-Connect-DXGW-example](/assets/img/post-Direct-Connect-DXGW-example.png)
![post-DXGW-TGW-example](/assets/img/post-DXGW-TGW-example.png)
![post-DXGW-example](/assets/img/post-DXGW-example.png)
![post-Direct-Connect-private-transit-VIF-DXGW-SiteLink](/assets/img/post-Direct-Connect-private-transit-VIF-DXGW-SiteLink.png)  

## VIF 分类和使用场景
- VIF 实际上是 dot1q vlan encapsulation 的 sub-interface  
- public VIF 连接 AWS Edge locations，访问 AWS public services  
- private VIF 连接 VGW, DXGW  
  - on-prem -- DX -- private VIF -- VPC -- PrivateLink/Endpoint -- AWS public services，[只针对 interface Endpoint，并不针对 gateway Endpoint](https://docs.aws.amazon.com/vpc/latest/privatelink/vpc-endpoints-ddb.html)，这个架构当前也是 supported  
![post-Direct-Connect-VPC-Endpoint-example](/assets/img/post-Direct-Connect-VPC-Endpoint-example.png)  
- 同一个 physical link、LAG 最多承载 50 个 public & private VIF，1 个 transit VIF    
- VIF default MTU 1500，如果 private VIF, transmit VIF 需要开启 Jumbo Frames，需要手动配置
  - 若开启 Jumbo Frames，需要整个转发路径上的设备都支持 Jumbo Frames，比如 EC2, Router  
  - this can cause an update to the underlying physical connection if it wasn't updated to support jumbo frames  
  - updating the connection disrupts network connectivity for all VIF associated with the connection for up to 30 seconds    
- transmit VIF 最小 bandwidth 要求 1G，如果客户只需要 300M 带宽那么可以向 APN/ISP 订购 1G dedicate 专线，借助 QoS 在 ISP 限速  
- private VIF，VGW 只能关联 1 个 VPC，DXGW 可以关联不同 region 最多 10 个 VGW(VGW -- VPC)    
- 从购买、配置的角度来看：  
  - standard VIF  
    - 本 AWS account 下的 DX, VIF  
  - hosted VIF  
    - 跨 AWS account，共享 DX connection；或者从 APN 购买的 VIF  
    - ![post-Direct-Connect-hosted](/assets/img/post-Direct-Connect-hosted.png)
  - hosted connection
    - sub-1G connection，一般是从 APN/ISP 购买  

|                        | public VIF                                           | private VIF                                                          | transmit VIF                                                         |
| ---------------------- | ---------------------------------------------------- | -------------------------------------------------------------------- | -------------------------------------------------------------------- |
| Connection Speed       | any                                                  | any                                                                  | speed >= 1G                                                          |
| Gateway type           | n/a                                                  | VGW/DXGW                                                             | DXGW                                                                 |
| Prefix limit(inbound)  | 1000                                                 | 100                                                                  | 100                                                                  |
| Prefix limit(outbound) | n/a                                                  | n/a                                                                  | 20 per TGW-DXGW association                                          |
| Peer IPs minimum CIDR  | /31                                                  | /30                                                                  | /30                                                                  |
| Support Jumbo Frames   | no                                                   | yes(9001)                                                            | yes(8500)                                                            |
| Use case               | to AWS public services, eg. S3                       | to VPC(1/10)                                                         | to multiple VPCs(1000) via TGW                                       |
| uRPF check             | enabled                                              | disabled                                                             | disabled                                                             |
| Technical requirement  | vlan id, IP prefix                                   | vlan id, VGW/DXGW                                                    | vlan id, DXGW                                                        |
| BGP ASN                | public/private                                       | public/private                                                       | public/private                                                       |
| BGP community          | scope(no_export)                                     | local pref                                                           | local pref                                                           |
| Route Control          | LPM, Local Pref(on-prem outbound), AS_PATH(only supported when using a Public ASN) | LPM, Local Pref(on-prem outbound), Local Pref BGP community tags(VPC outbound), AS_PATH, MED(lower is preferred) | LPM, Local Pref(on-prem outbound), Local Pref BGP community tags(VPC outbound), AS_PATH, MED(lower is preferred) |

> Local preference 相当于 metadata，可以用来控制路由  
> 只在 IBGP 邻居之间传递，不会传递到 EBGP  

![post-Diirect-Connect-private-VIF-ANS-example](/assets/img/post-Diirect-Connect-private-VIF-ANS-example.png)
[题目的参考文档](https://docs.amazonaws.cn/en_us/directconnect/latest/UserGuide/WorkingWithVirtualInterfaces.html)  
![post-Direct-connect-public-VIF-example](/assets/img/post-Direct-connect-public-VIF-example.png)
![post-Direct-Connect-design-example](/assets/img/post-Direct-Connect-design-example.png)  

## DX 故障排查
- [参考文档](https://docs.aws.amazon.com/directconnect/latest/UserGuide/Troubleshooting.html)  
- physical, Tx/Rx, if any CRC/error    
- layer2 VLAN encap, ARP  
- layer3, correct peered IP address on sub-intf, not physical intf  
- layer4, TCP, BGP    

![post-Direct-Connect-mpls-Q-in-Q-example](/assets/img/post-Direct-Connect-mpls-Q-in-Q-example.png)  

## data encrypt
- DX 本身只提供 private connection，并不提供加密  
- 可以通过 MACsec 或者 IPsec VPN 来提供加密  

### MACsec
- IEEE 802.1 layer2 standard, provides data confidentiality, data integrity, data origin authenticity for DX  
- only available on dedicated connection  
- near line rate，可能会有考题去选择 S2S VPN over DX，或者 MACsec；VPN tunnel 1.25 Gbps  
- MACsec security key is a *pre-shared key* that establishes the MACsec connectivity between CGW & AWS, the key is generated by the devices at the ends of the connection using the CKN/CAK pair that you provide to AWS and have also provisioned on your device  
- the key pairs used to generate the MACsec secret key:  
  - Connectivity Association Key (CAK)   
  - Connection Key Name (CKN)  

![post-Direct-Connect-MACsec-howtocfg](/assets/img/post-Direct-Connect-MACsec-howtocfg.png)
![post-Direct-Connect-MACsec-associate-MACsec-keys](/assets/img/post-Direct-Connect-MACsec-associate-MACsec-keys.png)  

## one DX access to multiple US regions
- on-prem 和某个 US regions VPC 建立 **dedicated** DX，[同一条 DX 可以打通 on-prem 与 US 其他 regions](https://aws.amazon.com/cn/blogs/aws/aws-direct-connect-access-to-multiple-us-regions/), DX inter-region capability     
- on-prem -- Direct Connect -- US region 1 -- AWS network -- US region 2，跨 region 的流量由 AWS 负责，路由表由 AWS 负责 (BGP 路由通告）    
![post-Direct-Connect-inter-region-capability](/assets/img/post-Direct-Connect-inter-region-capability.png)  

## access a remote AWS region
- AWS DX locations in *public Regions or AWSGovCloud(US)* can **access public services** in any other public region(excluding China)
  - public VIF, and BGP
  - 只能通过 public VIF 来访问其他 region public service
  - 注意下面 “路由控制” 提到的 public VIF 适用的 community 7224:9300, global, by default
- could **access a VPC** in your account as well
  - *DXGW in any region*, DXGW -- DX connection --- private VIF --- remote region VPCs, or TGW
  - or *public VIF for DX connection, then establis a VPN connection to your VPC in remote region* 
  - ![post-Direct-Connect-assecc-remote-region-example](/assets/img/post-Direct-Connect-assecc-remote-region-example.png)
- [single AWS DX connection for multi-Region servies](https://docs.aws.amazon.com/directconnect/latest/UserGuide/remote_regions.html)
- All networking traffic remains on the AWS global network backbone
![post-Direct-Connect-one-DX-connection-remote-Regions](/assets/img/post-Direct-Connect-one-DX-connection-remote-Regions.png)  

## 路由控制、优先级
- 对于 VPC --> on-prem 方向的流量来说，首先参考 VPC 路由表，然后 DX 路由   
- DX 路由表的 LPM(Longest Prefix Match) 条目优先级最高，然后 Local preference, AS-PATH   
![post-RT-priority-Direct-Connect](/assets/img/post-RT-priority-Direct-Connect.png)  

### Public VIF
- [public VIF 的 Active/Passive 路由控制](https://aws.amazon.com/premiumsupport/knowledge-center/dx-create-dx-connection-from-public-vif/?nc1=h_ls)  
- 默认情况下，public VIF 会将 AWS global public IP prefix 通过 BGP 通告给 on-prem；用户可以联系 AWS 来通告 customer-owned IP prefix  
- public VIF, private VIF 都可以使用 public(customer must own it) or private(64512-65535) ASN  
  - 如果使用public BGP ASN，customer must own it; only public BGP ASG support AS_PATH Prepending
  - [post-Direct-Connect-public-VIF-route-control-example](/assets/img/post-Direct-Connect-public-VIF-route-control-example.png)
- [public VIF 收到的客户侧路由明细，不会传播到 Internet 以及 AWS partner](https://docs.amazonaws.cn/en_us/directconnect/latest/UserGuide/routing-and-bgp.html)，使用 `no_export` 属性控制  
  - 但是 [客户侧通告到 AWS 的路由明细，是保留在 AWS DX 同 region，还是到 AWS all region(global)](https://docs.amazonaws.cn/en_us/directconnect/latest/UserGuide/routing-and-bgp.html)，可以通过 `scope BGP community tags` 来控制  
  - [本地配置](https://aws.amazon.com/premiumsupport/knowledge-center/control-routes-direct-connect/?nc1=h_ls)，on the public prefixs that you advertise to AWS, indicate how far to propagate your prefixs in AWS network，**support by public VIF only**  
    - 7224:9100—Local Amazon Region  
    - 7224:9200—All Amazon Regions for a continent  
      - North America–wide  
      - Asia Pacific  
      - Europe, the Middle East and Africa  
    - 7224:9300—Global (all public Amazon Regions), by default  
  - AWS DX 为其通告的路由使用以下 BGP community, **support by all VIF**:    
    - 7224:8100 – 源自关联了 Amazon 接入点的 Amazon Direct Connect 区域的路由  
    - 7224:8200 – 源自关联了 Amazon Direct Connect 接入点的大陆的路由。  
    - No tag    – 全球 （所有公有 Amazon 区域）。  

### Private VIF
- 站在 AWS DX/VIF 的视角去看，影响 `AWS --> on-prem` 的路由选路，不过所有手段都是开放给 on-prem 来执行  
- 首先是 LPM 最优先  
![post-Direct-Connect-Route-LPM-first](/assets/img/post-Direct-Connect-Route-LPM-first.png)  
- 若无法通过 LPM 控制，并且 [DX 与 VPC 在同 region](https://aws.amazon.com/premiumsupport/knowledge-center/active-passive-direct-connect/?nc1=h_ls)，可以通过 on-prem 的 AS_PATH prepending 来控制   
![post-Direct-Connect-Route-same-region-AS_PATH_shorter](/assets/img/post-Direct-Connect-Route-same-region-AS_PATH_shorter.png)  
- 若 DX 与 VPC 不在相同 region，可以通过 on-prem 设置 Local Preference BGP community tags 来控制  
- private，transmit VIF 支持 `Local Preference BGP community tags` 来 [控制 BGP 路由选路优先级](https://youtu.be/DXFooR95BYc?t=2007)，public VIF 不支持    
  - 7224:7100 — 低    
  - 7224:7200 — 中     
  - 7224:7300 — 高    
- 实现方式，AWS 在收到 on-prem 通告 BGP 路由时，检测到 **on-prem 添加的 Local Preference BGP community tags**，将对应 tags 当作 metadata 来处理，触发了一个行为，类似于使用 prefix-list 抓取携带了特定 tags 的路由条目，然后 set Local Pref  
- 对于 AWS --> on-prem 方向的流量，将会参考 Local Pref    
![post-Direct-Connect-Route-Local_Pref_BGP-community-tags](/assets/img/post-Direct-Connect-Route-Local_Pref_BGP-community-tags.png)  

### on-prem 视角
- 站在 on-prem 视角，影响 `on-prem --> AWS` 的路由选路  
- Local Pref，on-prem 设置  
- Advertise more specific prefixes over one DX connection，从 AWS 侧控制  
![post-Direct-Connect_Route-how-to-example](/assets/img/post-Direct-Connect_Route-how-to-example.png)  

### Active/Passive 路由
[Active/Passive](https://aws.amazon.com/premiumsupport/knowledge-center/active-passive-direct-connect/?nc1=h_ls)    
[Creating active/passive BGP connections over AWS Direct Connect](https://aws.amazon.com/es/blogs/networking-and-content-delivery/creating-active-passive-bgp-connections-over-aws-direct-connect/) 
![post-BGP-routing-overview](/assets/img/post-BGP-routing-overview.png)
>Local Pref, AS_Path 都可以用来做 inbound & outbound tuning  
> MED 用来做 inbound tuning, lower metric values are preferred

![post-Direct-Connect-VIF-Act-Stby](/assets/img/post-Direct-Connect-VIF-Act-Stby.png)
![post-Direct-Connect-Act-Pas-example](/assets/img/post-Direct-Connect-Act-Pas-example.png)
![post-Direct-Connect-Act-Passive-example](/assets/img/post-Direct-Connect-Act-Passive-example.png)  

### symmetry of flow
- 云上、云下两个方向的流量，经过同一条线路  
![post-Direct-Connect-symmetry-of-flow](/assets/img/post-Direct-Connect-symmetry-of-flow.png)  

# VPN
- AWS managed VPN, Site-to-Site, on-prem DC & VPC  
- Client VPN, remote client & VPC  
- -[YouTube 视频，AWS re:Invent 2018](https://www.youtube.com/watch?v=qmKkbuS9gRs)  

## Site-to-Site VPN
- fully managed and highly available VPN termination endpoints at AWS end  
- *two VPN tunnels* per one VPN connection  
- IPsec Site-to-Site tunnel with AES-256, SHA-2, and latest DH groups  
- support for NAT-T  
- [firewall rules bw the internet and your CGW](https://docs.aws.amazon.com/vpn/latest/s2svpn/your-cgw.html#FirewallRules)
![post-VPN-firewall-rules](/assets/img/post-VPN-firewall-rules.png)  
- charged per hour per VPN connection  
- VPN setup options:
  - Static  
    - policy-based or route-based  
    - static routing  
    - pre-shared key  for tunnel authenication
  - Dynamic  
    - route-based only  
    - dynamic routing(both side support BGP)  
    - pre-shared key  
- tunnel establishment - static & dynamic:
  - one connection, two tunnel(in differ AZ for HA)  
  - each tunnel limited up to 1.25 Gbps(due to IPsec requires CPU resources)    
  - on-prem --> AWS, could ECMP on both tunnels  
  - AWS --> on-prem, if with VGW, could use only one of the tunnel  
  - AWS --> on-prem, if with TGW, could use all of the tunnels for ECMP    

![post-VPN-S2S-VGW-traffic-flow](/assets/img/post-VPN-S2S-traffic-flow.png)
![post-VPN-S2S-TGW-traffic-flow-ECMP](/assets/img/post-VPN-S2S-TGW-traffic-flow-ECMP.png)  

## Client VPN
- AWS managed client-based VPN service  
- secure access to any resource in AWS and on-premises from anywhere using *OpenVPN* clients  
- client VPN Endpoint, is the terminate endpoint for your client VPN, in specific subnet    
  - client VPN Endpoint could in *same VPC multiple subnets*, each AZ one subnet(similar with TGW)  
  - AWS 在指定 subnet 放置一张 ENI，用来做 routing，实现方式和 TGW 类似  
- *client authentication*，[认证之后，可以连接 client VPN，但是流量不通](https://docs.aws.amazon.com/vpn/latest/clientvpn-admin/client-authentication.html)  
  - Active Directory，无论 AD 部署在什么位置，如果需要，可以借助 AD Connector 转发对应流量   
  - Federated, SAML  
  - manually authentication  
- *client authorization*，授权，通过授权之后，才可以访问具体服务  
  - network-based, authorization rule   
  - security groups  
- *Connectivity*，配置路由表  
  - UDP performance better than TCP  
- *manageability*，是否收集日志，配置过程中必须手动 enable 或者 disable  
  - active connections 可以从 console 查看，终止  
 
![post-VPN-client-vpn-authentication-authorization](/assets/img/post-VPN-client-vpn-authentication-authorization.png)
![post-VPN-client-vpn-authorization-rule](/assets/img/post-VPN-client-vpn-authorization-rule.png)
![post-VPN-client-vpn-connectivity](/assets/img/post-VPN-client-vpn-connectivity.png)  

![post-VPN-example1](/assets/img/post-VPN-example1.png)
![post-VPN-TGW-ECMP-example](/assets/img/post-VPN-TGW-ECMP-example.png)  

# Route53 DNS
![post-R53-at-a-glance](/assets/img/post-R53-at-a-glance.png)  
![Route53 failover-health-check SAA example](/assets/img/post-R53-HC.png)  
![post-R53-geo-SAA](/assets/img/post-R53-geo-SAA.png)  

## R53 DNS 解析的优先级

## R53 支持的 DNS 类型

### alias vs. CNAME

## R53 DNS routing policy
- **latency-based**  
  - for an application that is hosted in multiple AWS regions, latency-based routing can *improve performance* for the application users by serving requests from the AWS region that provides the *lowest latency*, thus ensuring *highest performance*.  
- geolocation
- 
## DNSSEC DNS 安全扩展
- [Domain Name System Security Extensions](https://aws.amazon.com/about-aws/whats-new/2020/12/announcing-amazon-route-53-support-dnssec/?nc1=h_ls)，DNS 安全扩展，为 DNS 提供数据来源认证和数据完整性验证，满足 FedRAMP 等法规要求  
- public hosted zone 功能  
- Route 53 DNSSEC provides __data origin authentication__ and __data integrity verification__ for DNS and can help customers meet compliance mandates, such as FedRAMP.  
- 若在 hosted zone 启用 DNSSEC，R53 会以加密方式对 hosted zone 当中每条 record 进行签名；R53 管理区域签名密钥，用户在 KMS 的 CMK(Customer Managed Key) 管理密钥签名密钥  
  - 后续不要对 CMK 进行任何权限上的修改，可能导致 KSK 状态变为 ACTION_NEEDED  
  - CMK 是 asymmetric，ECC_NIST_P256  
  - 必须在 us-east-1 region  
- When you enable DNSSEC signing on a hosted zone, Route 53 cryptographically signs each record in that hosted zone. Route 53 manages the zone-signing key, and you can manage the __key-signing key(KSK)__ in AWS Key Management Service (AWS KMS).   
![post-R53-DNSSEC-Key-signing-key-cfg](/assets/img/post-R53-DNSSEC-Key-signing-key-cfg.png)
![post-R53-DNSSEC-KSK-CMK-ANS-example](/assets/img/post-R53-DNSSEC-KSK-CMK-ANS-example.png)
![post-R53-DNSSEC-KSK-CMK-ANS-example1](/assets/img/post-R53-DNSSEC-KSK-CMK-ANS-example1.png)  

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

## DNS resolution bw on-prem and AWS using AWS Directory Service
- 类似于 R53 inbound、outbound resolve endpoint，打通 on-prem 与 VPC 之间的 DNS 解析  
- [需要将 R53(VPC CIDR x.x.x.2) 与 AWS Directory Service Simple AD 结合使用](https://aws.amazon.com/cn/blogs/security/how-to-set-up-dns-resolution-between-on-premises-networks-and-aws-using-aws-directory-service-and-amazon-route-53/)  
- Simple AD provides redundant and managed DNS services across AZs  
- 将 on-prem DNS 解析放到 R53  
![post-On-Prem-AWS-Simple-AD-forward-DNS-to-R53](/assets/img/post-On-Prem-AWS-Simple-AD-forward-DNS-to-R53.png)  
- 或者 VPC 的 DNS 解析放到 on-prem，若 on-prem 没有记录，再使用 `condition forwarder` 转发回 Simple AD --> R53  
![post-VPC-DNS-to-On-prem](/assets/img/post-VPC-DNS-to-On-prem.png)  
- 举个栗子，ANS-C00 考试题
![post-intergate-DNS-on-prem-VPC-R53](/assets/img/post-intergate-DNS-on-prem-VPC-R53.png)  

## R53 Resolver

### Inbound Resolver Endpoint

### Outbound Resolver Endpoint

### Public DNS Query logging for Public Hosted Zone 
- 所谓 public，两层含义，第一是 clients 来自公网，第二是 hosted zone 类型为 public  
- [public hosted zone 可以配置，入口在 hosted zone，CW Logs group 接收日志](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/query-logs.html)  
- [中国区暂时不支持 public DNS query logs](https://docs.amazonaws.cn/en_us/aws/latest/userguide/route53.html)  
- Query logs contain only the queries that DNS resolvers forward to Route 53. 如果 DNS resolver 已经由缓存了，会在 R53 声明的 TTL 到期前，回复缓存内容  
- 如果不需要明细内容，可以查看 [CW Metrics -- R53 -- Hosted Zone Metrics -- DNSQueries](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/hosted-zone-public-viewing-query-metrics.html)

![post-R53-public-hosted-zone-query-logging](/assets/img/post-R53-public-hosted-zone-query-logging.png)
![post-R53-public-hosted-zone-query-logging-cfg](/assets/img/post-R53-public-hosted-zone-query-logging-cfg.png)  

### Resolver query logging for Private Hosted Zone
- [private hosted zone，配置入口在 R53 Resolver，需要关联 VPC](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resolver-query-logs.html)  

![post-R53-Resolver-query-logging-public-and-PHZ](/assets/img/post-R53-Resolver-query-logging-public-and-PHZ.png)  

## R53 Health Check

# Global Accelerator 
- [Global Accelerator](https://aws.amazon.com/global-accelerator/?nc1=h_ls) 使用 AWS 全球骨干网，加速用户的访问  
- 降低延迟、抖动、丢包  
- to reduce latency by routing traffic to the nearest application endpoint that is closest in proximity with the users  
- provide 2 static public IP, 借助 *anycast*，AWS 可以在所有 GA edge location/PoP 提供相同的 static public IP，用户的 requests 总是被路由到最近的位置    
  - [AWS re:Invent 2020 How AWS Global Accelerator improves performance](https://www.youtube.com/watch?v=daJ2bmw_css) 提到了 GA 如何使用 anycast，来代替中间的 multiple hop ISPs 来提高性能  
- listener 支持 TCP, UDP；支持 client affinity  
- 后端可以是 ALB, NLB, EC2, 或者 EIP  
- 通过修改 R53 或者其他 vendor DNS 设置，把 domain name 指向 GA；DNS TTL 可以设置大一些，比如 1 day  
- 建议 reduce your request and cookie size
  - client --> AWS 方向，接收到完整 requests 之后，application 才能处理；更大的 request & cookie size，如果网络状况差，可能传输慢  
  - AWS --> client 方向，DTO 是收费的  
  

|                | public internet, multi hop ISPs                        | AWS GA, Backbone                     |
| -------------- | ------------------------------------------------------ | ------------------------------------ |
| throughput     | variable, high or low                                  | consistent high                      |
| RTT            | close to client, so low RTT                            | most of the path, so most of the RTT |
| possible issue | higher probability of congestion, packet loss & jitter |                                      | low |
| TCP MSS        | restricted MSS, usually 1460                           | Jumbo frames, large MSS              |

![how it works](/assets/img/post-GA-how-it-works.png)
![post-GA-how-it-works-low-latency-higher-MSS](/assets/img/post-GA-how-it-works-low-latency-higher-MSS.png)
![GA SAA example](/assets/img/post-GA-SAA.png)  

## Global Accelerator vs Cloudfront
                                                                                                                                                                                                   
|            | Global Accelerator                                                                                                                                         | Cloudfront                                                                                                                                                                                                                        |
| ---------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 共性       | 用户分布广泛，服务端希望提供低延迟服务                                                                                                                     | 用户分布广泛，服务端希望提供低延迟服务                                                                                                                                                                                            |
| 场景       | networking service to improve application's performance and availability(Regional failover, 多个 Endpoint 分布） for global users 低延迟、高可用，eg. game | cloud distributed networking service for web applications that provides low latency and speed 用户分布广泛，访问内容有重复所以可以被 Pop 节点缓存，CF 可以缓存、压缩文件，降低 origin 压力；Edge Functions 提供一定的边缘计算能力 |
| 实现方式   | client - GA --寻找最近的 endpoint；GA -- Endpoint 走 AWS backbone network；提供两个 static IP                                                              | client -- CF Pop 缓存，miss cache then refer Origin；根据 clients 地理位置不同，Pop public IP 不同                                                                                                                                |
| 支持的协议 | TCP, UDP, HTTP, HTTPS, gRPC；通常用于 non-HTTP 场景比如 gaming, IoT(MQTT), VoIP                                                                            | static & dynamic content, HTTP, HTTPS, WebSocket                                                                                                                                                                                  |
| security   | AWS Shield to prevent DDoS; if Endpoint is ALB then could integrate with WAF                                                                               | AWS Shield to prevent DDoS, WAF for additional protection against malicious traffic                                                                                                                                               |
| 价格       | fixed hourly fee, Data Transfer-Premium                                                                                                                    | Data Transfer Out, HTTP requests                                                                                                                                                                                                  |

# Cloudfront
![post-CF-overview](/assets/img/post-CF-overview.png)
![post-CF-request-flow](/assets/img/post-CF-request-flow.png)
![post-CF-Measuring-what-matters](/assets/img/post-CF-Measuring-what-matters.png)  
> TTFB, Time To First Bytes  

![CF SAA example](/assets/img/post-CF-SAA.png)
![post-CF-SAA-example1](/assets/img/post-CF-SAA-example1.png)

## secure access and restrict access to content
[保护数据安全](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/SecurityAndPrivateContent.html)  

### Signed URL, Signed cookes

### restrict access to S3 origin, OAC and OAI
![post-CF-S3-OAI-SAA](/assets/img/post-CF-S3-OAI-SAA.png)
![post-CF-S3-OAI-WAF-SAA](/assets/img/post-CF-S3-OAI-WAF-SAA.png)  

### field-level encryption
- enforce secure *end-to-end* connections *to origin* servers by using HTTPS  
- client/viewer -- HTTPS --- Cloudfront --- HTTP Listener --- origin ALB --- EC2, only EC2 could decrypt and see the content 
- [you need to specify the set of fields in POST requests that you want to be encrypted](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/field-level-encryption.html), and the *public key*(asymmetric encryption) to use to encrypt them   

![post-CF-field-level-encryption-howto](/assets/img/post-CF-field-level-encryption-howto.png)
![post-CF-field-level-encryption-example](/assets/img/post-CF-field-level-encryption-example.png)  

### Geographically restrict
![post-CF-geo-restriction-example](/assets/img/post-CF-geo-restriction-example.png)  

## Versioning
- 提供多个版本的文件，可以控制哪一个版本的文件可以被用户访问  
- 如果通过 invalidate 操作，由于 client 端、ISP 等缓存，用户看到的可能还是过期版本  
- versioning 比 invalidate 便宜，invalidate 操作收费，versioning 只是收取将 file 传递到 Edge 的费用  
- CF access log 包含文件版本号，方便查看、排错  

## Optimizing caching and availability 提高缓存命中率
- **Origin Shield**  
  - 如果 PoP 没有 cache，PoP 去 REC；如果 REC 也没有 cache，就去 origin  
  - 为了降低 origin 压力，更好的性能，可以指定 PoP 先去指定的 [REC(Origin Shield)](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/origin-shield.html) 查看有没有 cache  
  - 适用于实时流式处理、动态图像、clients 分布在不同地理区域  
![post-CF-origin-shield-howto](/assets/img/post-CF-origin-shield-howto.png)  

- **What you can do to Improving performance**  
![post-CF-content-breakdown-per-web-page](/assets/img/post-CF-content-breakdown-per-web-page.png)  
  - [参考文档](https://aws.amazon.com/blogs/networking-and-content-delivery/improve-your-website-performance-with-amazon-cloudfront/)   
  - [AWS re:Invent 2020: Improving website performance using Amazon CloudFront](https://www.youtube.com/watch?v=JUtM0hRoo8Q)  
  - define your caching strategy  
  - improve cache hit ratio  
  - use HTTP2 if possible  
  - use TLS1.3 if possible  
  - utilize CF capabilities at Edge  
    - HTTP redirect to HTTPs
    - compression(cache policy, Gzip, Brotli) 省钱 (DTO)  
      - Brotli compression 是 Google 提出的一种压缩方式，比 gzip 优化 25%  
    - Origin Shield

![post-CF-WordPress-CF-architecture-TTL-recommendations](/assets/img/post-CF-WordPress-CF-architecture-TTL-recommendations.png)
![post-CF-performance-improvements-TLS1-3](/assets/img/post-CF-performance-improvements-TLS1-3.png)  

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

## CF and edge function logging
- [doc about the log feature](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/logging.html)  
- **standsard logs(access logs)**  
  - contain detailed info(33 fields) about every user requests that CF receives 并不是实时日志  
  - delivery to S3 bucket    
  - *Timing of log file delivery*: up to several times an hour, usually within an hour  
- **real-time logs**  
  - get info about requests made to a distribution in real time(*logs are delivered within seconds of receiving the requests*)，可以是多个 distribution，近乎实时的日志  
  - delivery to data stream in Amazon Kinesis Data Streams  
  - sameling rate: the percentage of requests for which you want to receive real-time log records  
  - specific fields(total 43 fields available)  
    - origin-fbl(first-byte latency)  
    - origin-lbl  
    - cs-header-  
  - specific cache behaviors (path patterns) that you want to receive real-time logs for  
- **edge function logs**  
  - for Lambda@Edge and CF Functions  
  - delivery to CW logs group in us-east-1 region  

![post-CF-realtime-logging-Kinesis-Data-Streams](/assets/img/post-CF-realtime-logging-Kinesis-Data-Streams.png)  

# API Gateway
- [API GW](https://www.amazonaws.cn/api-gateway/?nc1=h_ls)  
- fully managed service/[完全托管服务](https://www.youtube.com/watch?v=1XcpQHfTOvs)，使用场景是创建、发布、维护、监控、保护任意规模的 API
- [AWS re:Invent 2019: [REPEAT 2] I didn’t know Amazon API Gateway did that (SVS212-R2)](https://www.youtube.com/watch?v=yfJZc3sJZ8E)，幽默风趣，内容丰富
- APIs act as "front door" for applications to access data/function from your backend services. API 作为 app 访问后端数据/服务的“前门”，后端可以是 EC2, Lambda, 或者任何 web application  
- 支持 HTTP, REST, WebSocket API
![REST API](/assets/img/IMG_20221012-163914187.png) 
- **常见的 integration types/使用场景：**
  - Lambda function
    - connect via *proxy* or direct *integration*
    - proxy:
      - client --> API-GW(Wrapper with metadata and pass to the backend) --> Lambda
      - client <-- API-GW(response pass, untouched, to client) <-- ([must be JSON format](https://docs.aws.amazon.com/apigateway/latest/developerguide/set-up-lambda-proxy-integrations.html))Lambda
    - integration:
      - client --> API-GW(by using VTL, requests can be modified by API-GW) --> Lambda
      - ![post-APIGW-Lambda-integration-flow](/assets/img/post-APIGW-Lambda-integration-flow.png)
  - HTTP/S
  - AWS Services endpoints, such as EC2, DDB, Kinesis
  - VPC PrivateLink(resources behind NLB), private integrations
  - Mock
    - respond to requests without a backend service(OPTIONS)
    - 对比 CORS

## Deployment types/部署类型
不同类型对应不同的场景，[参考文档](https://www.sentiatechblog.com/amazon-api-gateway-types-use-cases-and-performance)  
- **Edge-optimized endpoint**  
  - client -- CF -API GW
    - CF to reduce TLS connection overhead(reduce roundtrip time)
  - reduced latency for requests from around the world, designed for a globally distributed set of clients
- **Regional endpoint**  
  - reduced latency for requests that originate in the same region
  - can cfg your own CDN/CF
  - can protect with WAF
  - recommended API type for general use cases
- **Private endpoint**
  - securely expose your REST APIs only to other services within your VPC/DX

![post-APIGW-architecture-types](/assets/img/post-APIGW-architecture-types.png)   

**Stages**
- 不同环境，可能使用了不同的后端服务
- 可以借助 Lambda Env Variable 或者 Aliases 来操作
![post-APIGW-Stages](/assets/img/post-APIGW-Stages.png)  

## Security
- Authorization 认证
  - Open, no authentication/authorization
  - [IAM permissions](https://docs.aws.amazon.com/apigateway/latest/developerguide/security-iam.html)
    - IAM policy, AWS credentials to grant access
    - 简单场景，比如只暴露 API 给特定 IP range
  - Amazon Cognito authorizer
    - Amazon Cognito is a managed user directory for authentication
    - connect to the Amazon *Cognito User Pool* and, with *OAuth*, scope to enable authorization
  - Lambda authorizers
    - 自定义 Lambda 来验证 bearer token(OAuth or SAML)
- WAF
  - 比如防范 SQLi，XSS
  - block IP(API GW 借助 IAM resource policy 也可以做到，可能更经济实用）
  - block requests originating from a specify country or region
  - match specify string or regular expression pattern in the HTTP header, method, query string, URI, request body
  - block atachks from specific user-agent, bad bots（对比 Bing, Google 等对 SEO 有好处的 good bots), and content scrapers/爬虫

## API Caching, throttle
- API GW 从 Endpoint 拿到信息之后，第二次 client 请求就不会回源了，类似于 CF PoP 
- leverage **cache keys** in order to optimize response
  - path, headers, query strings
- can be set up by stage or method
- API GW is low cost and scales automatically
- could  **throttle** API GW to prevent attacks，throttle 是 per region/per account 级别
  - steady-state requests rate 10,000 per second(RPS)  
  - maximum concurrent requests is 5,000 per second across all APIs within an AWS account  
  - configure method-level throtting in a **Usage Plan**
    - Usage Plans and API Keys，区分不同的用户，对 VIP 用户提供更多/更好的服务 
  - if exceed the limit, got `429 Too Many Requests` error response; client could resubmit the failed requests    
![caching](/assets/img/IMG_20221012-171018008.png) 
![API Keys](/assets/img/IMG_20221012-171905991.png) 

## Same Origin Policy
- [同源策略](https://developer.mozilla.org/zh-CN/docs/Web/Security/Same-origin_policy)，防止 Cross-Site-Scripting/XSS 攻击  
- **从 client browser 层面来实现**，不过 Postman/curl 可能会忽略 XSS  
- web browser permits scripts contained in a first web page to access data in a second web page, but only if both web pages have the same origin/domain name. 同源/同域名，同 scheme(http, https)，浏览器才认为 XSS 有效  

### CORS 简单介绍
- Cross-origin Resource Sharing，[跨域资源共享](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/CORS)   
- **server 层面来控制**，相当于在特定条件下 bypass Same Origin Policy  
- let servers describe which origins are permitted to read that information from a web browser，所以实际上 CORS 是 enforced by client/broswer  
- allows restricted resources(eg. fonts, images) on a web page to be requested from another domain outside the domain from which the first resource was served.  
- when use Javascript/AJAX that uses multiple domains with API GW, ensure you have enabled CORS on API GW

![CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS/cors_principle.png)  
![post-APIGW-CORS-SAP-example1](/assets/img/post-APIGW-CORS-SAP-example1.png)  

## Monitor, Metrics, Logs
[docs](https://docs.aws.amazon.com/apigateway/latest/developerguide/security-monitoring.html)  
**Monitoring**  
- CloudWatch
  - 提供三个维度的 metrics，API-, stage-, method-level
  - 4xx, 5xx
  - Count, CacheHitCount, CacheMisCount
  - IntegrationLatency
- AWS X-Ray
  - trace and analyze requests end to end
- Cloudtrail
- AWS Config
  - assess/评估，audit, evaluate/评价 configuration updates

**Logs**  
- Logs to CloudWatch logs
  - Execution logs
  - error/info level
  - log full requests/responses
- Access logs
  - customize log format
  - CLF(Common Log Format), JSON, XML, CSV
- Logs to Amazon Kinesis Data Firehose

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
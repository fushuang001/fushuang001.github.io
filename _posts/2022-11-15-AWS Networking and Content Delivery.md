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
  - [Route Table](#route-table)
    - [Route Table 优先级](#route-table-优先级)
    - [Route Table propagation](#route-table-propagation)
  - [IPAM - IP Address Manager](#ipam---ip-address-manager)
  - [Egress-Only Internet Gateway](#egress-only-internet-gateway)
  - [EC2 bandwidth](#ec2-bandwidth)
    - [Enhanced networking - ENA](#enhanced-networking---ena)
  - [prefix-list](#prefix-list)
  - [VPC Endpoint, Endpoint Services, PrivateLink](#vpc-endpoint-endpoint-services-privatelink)
  - [VPN](#vpn)
  - [Transit VPC](#transit-vpc)
- [Transit Gateway](#transit-gateway)
- [ELB 对比](#elb-对比)
- [ALB](#alb)
  - [Stick session](#stick-session)
- [NLB](#nlb)
- [GWLB](#gwlb)
- [Direct Connect DX 专线](#direct-connect-dx-专线)
  - [DXGW](#dxgw)
  - [VIF 分类和使用场景](#vif-分类和使用场景)
  - [one DX access to multiple US regions](#one-dx-access-to-multiple-us-regions)
  - [路由控制、优先级](#路由控制优先级)
    - [Public VIF](#public-vif)
    - [Private VIF](#private-vif)
    - [on-prem 视角](#on-prem-视角)
    - [symmetry of flow](#symmetry-of-flow)
- [VPN](#vpn-1)
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

[AWS 考试预约、培训视频、白皮书](https://aws.amazon.com/certification/certified-advanced-networking-specialty/)  
[考试大纲，查漏补缺](https://d1.awsstatic.com/training-and-certification/docs-advnetworking-spec/AWS-Certified-Advanced-Networking-Specialty_Exam-Guide.pdf)  

# VPC
![VPC Sharing](/assets/img/IMG_20220504-212047378.png)  
![post-VPC-pricing-SAA](/assets/img/post-VPC-pricing-SAA.png)  

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

## IPAM - IP Address Manager
- manage all the CIDR blocks that are used in an account or across an organization in AWS Organization  
- to avoid CIDR overlap  
- CloudFormation can request a CIDR block from IPAM  
![post-VPC-IPAM-example](/assets/img/post-VPC-IPAM-example.png)  

## Egress-Only Internet Gateway
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

## Transit VPC
- on-prem 与 [Transit VPC](https://docs.aws.amazon.com/whitepapers/latest/aws-vpc-connectivity-options/transit-vpc-option.html) 打通 VPN 连接，之后通过 Transit VPC，打通 on-prem 与其他 VPC 之间的连接  
- hub-spoke 模型提供 inter-VPC connectivity, hub/central VPC 通常使用`BGP over IPsec VPN` 连接 spoke VPC  
- Transit/hub/central VPC 使用 EC2 virtual Router 借助第三方 software appliances that route incoming traffic to their destinations using the VPN overlay    
- 优点是 hub/central 的 EC2 virtual router 可以提供 IPS/WAF 等功能；hub-spoke 设计相对简单，transitive routing enabled using the overlay VPN network  
- 缺点是 third-party vendor virtual appliances on EC2 费用高（实际上 TGW 费用也不低啊），VPN connection 吞吐量有限 (up to 1.25 Gbps per VPN tunnel)，手动配置、管理等   
- 所以 AWS 是比较推荐使用 TGW，[表格里面提供了 VPC peering(year 2014)、transit VPC(year 2016)，TGW(year 2018) 的对比](https://docs.aws.amazon.com/whitepapers/latest/building-scalable-secure-multi-vpc-network-infrastructure/transit-vpc-solution.html)  

![post-transit-vpc-how-it-design](/assets/img/post-transit-vpc-how-it-design.png)  
![post-transmit-VPC-ANS-example](/assets/img/post-transmit-VPC-ANS-example.png)  

# Transit Gateway

# ELB 对比
把之前做的表格添加过来  

# ALB

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
  - DXGW 是 global 的，不区分 region；VGW 是 regional 的  
  - DXGW 类似于 redundant BGP RR，或者说 distributed VRF  
- on-premise --- DX --- ZHY VPC, VGW -- DXGW --- BJS VPC，可以通过 DXGW 打通 BJS, on-prem 的连接  
- DXGW -- TGW -- transit VIF -- VPC, if you connect to multiple transit gateways that are in different resions, use unique BGP ASNs for each transit gateway.  
![post-Direct-Connect-DXGW-example](/assets/img/post-Direct-Connect-DXGW-example.png)  
![post-DXGW-TGW-example](/assets/img/post-DXGW-TGW-example.png)  

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

|                        | public VIF                     | private VIF                                 | transmit VIF                                |
| ---------------------- | ------------------------------ | ------------------------------------------- | ------------------------------------------- |
| Connection Speed       | any                            | any                                         | speed >= 1G                                 |
| Gateway type           | n/a                            | VGW/DXGW                                    | DXGW                                        |
| Prefix limit(inbound)  | 1000                           | 100                                         | 100                                         |
| Prefix limit(outbound) | n/a                            | n/a                                         | 20 per TGW-DXGW association                 |
| Peer IPs minimum CIDR  | /31                            | /30                                         | /30                                         |
| Support Jumbo Frames   | no                             | yes(9001)                                   | yes(8500)                                   |
| Use case               | to AWS public services, eg. S3 | to VPC(1/10)                                | to multiple VPCs(1000) via TGW              |
| uRPF check             | enabled                        | disabled                                    | disabled                                    |
| Technical requirement  | vlan id, IP prefix             | vlan id, VGW/DXGW                           | vlan id, DXGW                               |
| BGP ASN                | public/private                 | public/private                              | public/private                              |
| BGP community          | scope(no_export)               | local pref                                  | local pref                                  |
| Route Control          | LPM, AS_PATH                   | LPM, Local Pref BGP community tags, AS_PATH | LPM, Local Pref BGP community tags, AS_PATH |

> Local preference 相当于 metadata，可以用来控制路由  
> 只在 IBGP 邻居之间传递，不会传递到 EBGP  

![post-Diirect-Connect-private-VIF-ANS-example](/assets/img/post-Diirect-Connect-private-VIF-ANS-example.png)  
[题目的参考文档](https://docs.amazonaws.cn/en_us/directconnect/latest/UserGuide/WorkingWithVirtualInterfaces.html)  
![post-Direct-connect-public-VIF-example](/assets/img/post-Direct-connect-public-VIF-example.png)  
![post-Direct-Connect-design-example](/assets/img/post-Direct-Connect-design-example.png)  

## one DX access to multiple US regions
- on-prem 和某个 US regions VPC 建立 **dedicated** DX，[同一条 DX 可以打通 on-prem 与 US 其他 regions](https://aws.amazon.com/cn/blogs/aws/aws-direct-connect-access-to-multiple-us-regions/), DX inter-region capability     
- on-prem -- Direct Connect -- US region 1 -- AWS network -- US region 2，跨 region 的流量由 AWS 负责，路由表由 AWS 负责 (BGP 路由通告）    
![post-Direct-Connect-inter-region-capability](/assets/img/post-Direct-Connect-inter-region-capability.png)  

## 路由控制、优先级
- 对于 VPC --> on-prem 方向的流量来说，首先参考 VPC 路由表，然后 DX 路由  
- DX 路由表的 LPM(Longest Prefix Match) 条目优先级最高，然后 Local preference, AS-PATH 
![post-RT-priority-Direct-Connect](/assets/img/post-RT-priority-Direct-Connect.png)  

### Public VIF
- 默认情况下，public VIF 会将 AWS global public IP prefix 通过 BGP 通告给 on-prem；用户可以联系 AWS 来通告 customer-owned IP prefix  
- public VIF, private VIF 都可以使用 public(customer must own it) or private(64512-65535) ASN  
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

[Active/Passive](https://aws.amazon.com/premiumsupport/knowledge-center/active-passive-direct-connect/?nc1=h_ls)    
A/A  
BGP 参数  

### symmetry of flow
- 云上、云下两个方向的流量，经过同一条线路  
![post-Direct-Connect-symmetry-of-flow](/assets/img/post-Direct-Connect-symmetry-of-flow.png)  

# VPN
![post-VPN-example1](/assets/img/post-VPN-example1.png)  

# Route53 DNS
![Route53 failover-health-check SAA example](/assets/img/post-R53-HC.png)  
![post-R53-geo-SAA](/assets/img/post-R53-geo-SAA.png)  

## R53 DNS 解析的优先级

## R53 支持的 DNS 类型

### alias vs. CNAME

## R53 DNS routing policy

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
                                                                                                                                                                                                   
|            | Global Accelerator                                                                                                                                         | Cloudfront                                                                                                                                                                                                                        |
| ---------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 共性       | 用户分布广泛，服务端希望提供低延迟服务                                                                                                                     | 用户分布广泛，服务端希望提供低延迟服务                                                                                                                                                                                            |
| 场景       | networking service to improve application's performance and availability(Regional failover, 多个 Endpoint 分布） for global users 低延迟、高可用，eg. game | cloud distributed networking service for web applications that provides low latency and speed 用户分布广泛，访问内容有重复所以可以被 Pop 节点缓存，CF 可以缓存、压缩文件，降低 origin 压力；Edge Functions 提供一定的边缘计算能力 |
| 实现方式   | client - GA --寻找最近的 endpoint；GA -- Endpoint 走 AWS backbone network；提供两个 static IP                                                              | client -- CF Pop 缓存，miss cache then refer Origin；根据 clients 地理位置不同，Pop public IP 不同                                                                                                                                |
| 支持的协议 | TCP, UDP, HTTP, HTTPS, gRPC；通常用于 non-HTTP 场景比如 gaming, IoT(MQTT), VoIP                                                                            | static & dynamic content, HTTP, HTTPS, WebSocket                                                                                                                                                                                  |
| security   | AWS Shield to prevent DDoS; if Endpoint is ALB then could integrate with WAF                                                                               | AWS Shield to prevent DDoS, WAF for additional protection against malicious traffic                                                                                                                                               |
| 价格       | fixed hourly fee, Data Transfer-Premium                                                                                                                    | Data Transfer Out, HTTP requests                                                                                                                                                                                                  |

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
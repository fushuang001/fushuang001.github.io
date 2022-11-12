---
layout:         post
title:          如何禁止 clients 通过 ALB IP 来访问服务
subtitle:		要求 clients 必须通过 DNS 域名来访问服务  
date:           2022-07-17
author:         Luke
cover:          '/assets/img/http-vs-https.jpg'
tags:           AWS, ALB, DNS, WAF,regular-expression
---
- [需求](#需求)
- [实现方式 1](#实现方式-1)
- [实现方式 2](#实现方式-2)
  - [注意事项](#注意事项)
  - [测试效果](#测试效果)
  - [WAF 相关规则](#waf-相关规则)

# 需求   
禁止 clients 直接通过 ALB IP 来访问服务  
要求 clients 必须通过 DNS 域名来访问服务  

# 实现方式 1
[ALB listener 去匹配 Host Header](https://docs.amazonaws.cn/en_us/elasticloadbalancing/latest/application/load-balancer-listeners.html)，比如 `*.example.com` 通配符；  
其他匹配不到的，就用默认的 `fixed-response`，要么客户访问了不在 ALB 托管的域名，要么客户直接通过 ALB IP 来访问的  
__缺点：__  
    如果客户有很多的 host header/url path，设置起来会比较麻烦，可以考虑 AWS ELBv2 API  
    容易遇到 limit Condition values limit (5 per rule) has been met  
![图 3](/assets/img/IMG_20220715-120628901.png)  

# 实现方式 2
ALB + WAF，[WAF 直接 regular expression 正则表达式](https://docs.amazonaws.cn/en_us/waf/latest/developerguide/waf-rule-statement-type-regex-match.html) 来匹配 HTTP request 的 host/Host 字段，就很厉害  

## 注意事项 
1. 通过`RegexString`: `(\\b25[0-5]|\\b2[0-4][0-9]|\\b[01]?[0-9][0-9]?)(\\.(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)){3}` 来匹配 IP 地址，其他正则表达式可选，可以自行 Google  
2. [若 ALB 同时是 NLB 的 target](https://aws.amazon.com/premiumsupport/knowledge-center/alb-static-ip/)，以上正则表达式同时会把 NLB ---> ALB 的 health check 给 block，导致 NLB --> ALB --> targets 的 health check failed；  
需要在 WAF Rules 使用第二条 `NotStatement "RegexString": "172.31.*` 将 NLB private IP CIDR 排除在外  
3. 针对 HTTP, HTTPS requests 都可以生效  

## 测试效果 
可以直接通过域名访问 ALB  
mynxalb.com 是一个 Route 53 alias 记录，指向我自己的 ALB；通过配置 R53 private hosted zone，关联 VPC，解决域名映射问题  
![connect hostname](/assets/img/post-connect-hostname.png)  

---

无法直接通过 ALB IP 访问  
![block ip](/assets/img/post-block-ip.png)  

---

如果自定义 HTTP header 的 host 字段为 172.31.x.x 是可以绕过去的，算是一个隐藏的缺点   
![not block Host Header for 172.31](/assets/img/post-not-block-Host-172.png)  

## WAF 相关规则  
`RegexString": "(\\b25[0-5]|\\b2[0-4][0-9]|\\b[01]?[0-9][0-9]?)(\\.(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)){3}"`
![WAF rules](/assets/img/post-WAF-refuse-connect-ALB-via-IP.png)  
---
layout:         post
title:          如何禁止clients通过ALB IP来访问服务
subtitle:		要求 clients 必须通过 DNS 域名来访问服务  
date:           2022-07-17
author:         Forest
cover:          '/assets/img/http-vs-https.jpg'
tags:           AWS, ALB, DNS, WAF,regular-expression
---

**需求描述：**    
禁止 clients 直接通过 ALB IP 来访问服务  
要求 clients 必须通过 DNS 域名来访问服务  

**实现方式 1：**  
[ALB listener 去匹配 Host Header](https://docs.amazonaws.cn/en_us/elasticloadbalancing/latest/application/load-balancer-listeners.html)，比如 `*.example.com` 通配符；  
其他匹配不到的，就用默认的 `fixed-response`，要么客户访问了不在 ALB 托管的域名，要么客户直接通过 ALB IP 来访问的  
__缺点：__  
    如果客户有很多的 host header/url path，设置起来会比较麻烦，可以考虑 AWS ELBv2 API  
    容易遇到 limit Condition values limit (5 per rule) has been met  
![图 3](/assets/img/IMG_20220715-120628901.png)  

**实现方式 2：**  
ALB + WAF，[WAF 直接 regular expression 正则表达式](https://docs.amazonaws.cn/en_us/waf/latest/developerguide/waf-rule-statement-type-regex-match.html) 来匹配 HTTP request 的 host/Host 字段，就很厉害  

__注意事项：__  
1. 通过`RegexString`: `(\\b25[0-5]|\\b2[0-4][0-9]|\\b[01]?[0-9][0-9]?)(\\.(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)){3}` 来匹配 IP 地址，其他正则表达式可选，可以自行 Google  
2. [若 ALB 同时是 NLB 的 target](https://aws.amazon.com/premiumsupport/knowledge-center/alb-static-ip/)，以上正则表达式同时会把 NLB ---> ALB 的 health check 给 block，导致 NLB --> ALB --> targets 的 health check failed；  
需要在 WAF Rules 使用第二条 `NotStatement "RegexString": "172.31.*` 将 NLB private IP CIDR 排除在外  
3. 针对 HTTP, HTTPS requests 都可以生效  

__测试效果：__  
```
可以直接通过域名访问
mynxalb.com 是一个 Route 53 alias 记录，指向我自己的 ALB；通过配置 R53 private hosted zone，关联 VPC，解决域名映射问题
[ec2-user@Bastion-ConsumerVPC-ZHY ~]$ curl https://mynxalb.com:8443 -ik
HTTP/2 200
date: Sun, 17 Jul 2022 03:05:59 GMT
content-type: application/json; charset=utf-8
content-length: 594
x-powered-by: Express
etag: W/"252-QIguCahwaAR4VB6SFo9Uy3ZxaT8"

{
  "path": "/",
  "headers": {
    "x-forwarded-for": "52.83.252.130:55224",
    "x-forwarded-proto": "https",
    "x-forwarded-port": "8443",
    "host": "mynxalb.com:8443",
    "x-amzn-trace-id": "Root=1-62d37c97-364b34ff3dd4d2e66043efa1",
    "user-agent": "curl/7.79.1",
    "accept": "*/*"
  },
  "method": "GET",
  "body": "",
  "fresh": false,
  "hostname": "mynxalb.com",
  "ip": "52.83.252.130:55224",
  "ips": [
    "52.83.252.130:55224"
  ],
  "protocol": "https",
  "query": {},
  "subdomains": [],
  "xhr": false,
  "os": {
    "hostname": "f58229823e66"
  },
  "connection": {}
```

```
无法直接通过 ALB IP 访问
[ec2-user@Bastion-ConsumerVPC-ZHY ~]$ curl https://69.235.154.93:8443 -ik
HTTP/2 403
server: awselb/2.0
date: Sun, 17 Jul 2022 03:05:00 GMT
content-length: 119
content-type: application/json

{
"Block Reason": "You are trying to connect ALB with it's IP address directly",
"How To": "Try with ALB domain name"
}
```

```
如果自定义 HTTP header 的 host 字段，172.31.x.x 是可以绕过去的
[ec2-user@Bastion-ConsumerVPC-ZHY ~]$ curl mynxalb.com:8211 -H "Host: 1.1.1.1"
{
"Block Reason": "You are trying to connect ALB with it's IP address directly",
"How To": "Try with ALB domain name"
}
[ec2-user@Bastion-ConsumerVPC-ZHY ~]$ curl mynxalb.com:8211 -H "Host: 172.31.1.1"
apache always Listen 80; You make ELB Listener to listen different ports.
```
··
__WAF 相关规则__  
```
{
  "Name": "regular-expression-block-ip-direct-access",
  "Priority": 2,
  "Statement": {
    "AndStatement": {
      "Statements": [
        {
          "RegexMatchStatement": {
            "RegexString": "(\\b25[0-5]|\\b2[0-4][0-9]|\\b[01]?[0-9][0-9]?)(\\.(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)){3}",
            "FieldToMatch": {
              "SingleHeader": {
                "Name": "host" // 自动匹配 Host, host
              }
            },
            "TextTransformations": [
              {
                "Priority": 0,
                "Type": "NONE"
              }
            ]
          }
        },
        {
          "NotStatement": { // 将 NLB internal IP CIDR 排除在外，避免影响 NLB --> ALB -->targets 的 health check
            "Statement": {
              "RegexMatchStatement": {
                "RegexString": "172.31.*",
                "FieldToMatch": {
                  "SingleHeader": {
                    "Name": "host" // 自动匹配 Host, host
                  }
                },
                "TextTransformations": [
                  {
                    "Priority": 0,
                    "Type": "NONE"
                  }
                ]
              }
            }
          }
        }
      ]
    }
  },
  "Action": {
    "Block": {
      "CustomResponse": {
        "ResponseCode": 403,
        "CustomResponseBodyKey": "Block-access-ALB-via-IP-directly"
      }
    }
  },
  "VisibilityConfig": {
    "SampledRequestsEnabled": true,
    "CloudWatchMetricsEnabled": true,
    "MetricName": "regular-expression-block-ip-direct-access"
  }
}
```
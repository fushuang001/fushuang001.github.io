
---
layout:         post
title:          AWS ALB health check failed 故障排查
subtitle:		Windows IIS server 的 HTTP response HSTS header 不符合标准
date:           2022-07-17
author:         Forest
cover:          '/assets/img/alb_component_architecture.png'
tags:           AWS, ALB, health check, HTTP header, HSTS
---

# 相关的问题描述
ALB 与后端实例健康检查失败，提示 state: unhealthy, reason: Target.FailedHealthChecks   

# 分析过程  
1. 首先根据 [ALB health check 相关文档](https://docs.amazonaws.cn/en_us/elasticloadbalancing/latest/application/target-group-health-checks.html)，简单预判 unhealthy reason
> Related reason codes: 
> Target.ResponseCodeMismatch
> Target.Timeout 
> Target.FailedHealthChecks, 一般是 target 原因导致
> Elb.InternalError
2. 检查 ALB 出向、EC2 入向 `Security Group` 放行了 health check 流量，`ACL` 保持默认；ALB health check success code 200  
3. 由于客户的 EC2 Security Group 针对 0.0.0.0/0 开放 health check 端口的访问，于是从自己 EC2 测试，返回 HTTP code 200 OK  
```
curl <customer.ec2.ip.addr>/Ahealth_check_path -is | head -1  
    HTTP/1.1 200 OK  
```
3. 由于客户 EC2/target 返回 HTTP 200 OK，暂时未发现异常；而 unhealthy reason 提示是 target 问题，于是去查看 ALB 日志，提示`TARGET_ERROR: Could not parse data entirely`，ALB 无法解析 EC2 的 response  
4. 在自己 EC2 抓包，wireshark 分析如下  
```
Strict-Transport-Security : 31536000\r\n
    [Expert Info (Warning/Protocol): Illegal characters found in header name] <<< 在 HTTP header Strict-Transport-Security 发现无效字符
        [Illegal characters found in header name]
        [Severity level: Warning]
        [Group: Protocol]
```
5. 做个对比  
```
curl <customer.ec2.ip.addr>/Ahealth_check_path -is | egrep "Strict|Server"
    Server: Microsoft-IIS/10.0
    Strict-Transport-Security : 31536000     <<<< "Strict-Transport-Security" header 与冒号之间多出来一个空格
```
# 解决方法  
通过查阅 [HTTP Strict-Transport-Security](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Strict-Transport-Security)(HSTS) 相关内容，建议服务器/EC2 针对 HTTP 服务，不要启用 HSTS 功能；服务器/EC2 应该只针对 HTTPS 服务启用 HSTS 功能，并且开启之后，不要关闭此功能。
> Note: 
    The Strict-Transport-Security header is ignored by the browser when your site is accessed using HTTP; this is because an attacker may intercept HTTP connections and inject the header or remove it. When your site is accessed over HTTPS with no certificate errors, the browser knows your site is HTTPS capable and will honor the Strict-Transport-Security header. 

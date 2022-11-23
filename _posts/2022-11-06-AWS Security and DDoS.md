---
layout:         post
title:          AWS Security and DDoS
subtitle:		    备考 SAA 的笔记，Cloud Security
date:           2022-11-06
author:         Luke
cover:          '/assets/img/cloud-security.jpg'
tags:           AWS, SAA, IAM, STS, KMS, SecurityHub, GuardDuty, Organization, WAF, Shiled
---
- [Shared responsibility model](#shared-responsibility-model)
- [AWS Identity and Access Management (IAM)](#aws-identity-and-access-management-iam)
  - [Security Best Practice：](#security-best-practice)
  - [IAM user 和 role 有什么不同](#iam-user-和-role-有什么不同)
    - [granting cross-account access bw two accounts](#granting-cross-account-access-bw-two-accounts)
  - [IAM Policy](#iam-policy)
  - [IAM Role 和 Resource based policy 有什么不同](#iam-role-和-resource-based-policy-有什么不同)
  - [STS to assume a role](#sts-to-assume-a-role)
  - [Permission boundary](#permission-boundary)
  - [Identity Federation 联合身份认证](#identity-federation-联合身份认证)
- [AWS Organizations](#aws-organizations)
  - [Service Control Policy - SCP](#service-control-policy---scp)
- [Directory Service, AD](#directory-service-ad)
  - [AWS Directory Service AD Connector](#aws-directory-service-ad-connector)
- [Control Tower](#control-tower)
- [Compliance](#compliance)
  - [AWS Macie](#aws-macie)
  - [AWS Artifact](#aws-artifact)
  - [AWS Textract](#aws-textract)
  - [AWS Comprehend Medical](#aws-comprehend-medical)
- [Denial-of-service attacks - DDoS](#denial-of-service-attacks---ddos)
  - [Shield](#shield)
  - [WAF](#waf)
    - [WAF 的行为/Action](#waf-的行为action)
    - [WAF rules 分类](#waf-rules-分类)
    - [使用 WAF 的最佳实践](#使用-waf-的最佳实践)
- [Data security](#data-security)
  - [KMS](#kms)
  - [CloudHSM](#cloudhsm)
  - [GuardDuty](#guardduty)
  - [AWS Inspector](#aws-inspector)
  - [SecurityHub](#securityhub)
- [ACM - Certification](#acm---certification)
- [KMS - Key mgmt service](#kms---key-mgmt-service)
- [参考文档](#参考文档)

# Shared responsibility model
责任共担模型  
AWS 负责 DC 的安全 (security 'of' the Cloud), AWS operates, manages, and controls the components at all layers of infrastructure.    
客户负责 云上资源的安全， security 'in' the cloud, Customers are responsible for the security of everything that they create and put in the AWS Cloud.    
![shared_responsibility_model](/assets/img/IMG_20220412-194714970.png)  

# AWS Identity and Access Management (IAM)
## Security Best Practice：
- 仅赋予必要的权限，Granting only the permissions that are needed to perform specific job tasks.  
- 启用 MFA(Multi-factor authentication)  
- 可以使用 role 的时候，就不要使用 user；因为 role 是临时的，没有 username/password  

## IAM user 和 role 有什么不同
- user 是 long-term usage，有 username/password，如果给 AWS CLI 使用，有 AKSK；  
- IAM roles are ideal for situations in which access to services or resources needs to be granted **temporarily**, instead of long-term.  并没有 AKSK，可以用 STS 获取临时 credentials；     
- 可以想象一个场景，只要带上某一顶帽子，就拥有了特定权限；那么 role policy 就定义了权限，**assume role** 就是戴帽子。  
  - EC2 Instance Profile/Role  
  - Service Roles: API GW, CodeDeploy, etc...  
  - Cross Account roles(this is great, as you never need to share user credentials/AKSK)    
    - 不过 cross account assume role 的场景，并不总是适用的；
    - 因为 assume role 之后，user/service/application 原来的 IAM policy/permissions 将会失效，仅有用 role 的 permissions  
    - 如果同时需要 origin policy & assume role 的 policy 才能执行完整操作，这个场景会遇到问题  
    - 参考 IAM role 和 Resource based policy 有什么不同来解决  
![assume_role](/assets/img/IMG_20220412-205830230.png)  

### granting cross-account access bw two accounts
[docs link](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_common-scenarios_aws-accounts.html)  

## IAM Policy
- A JSON format document that grants or denies permissions to AWS services and resources.  
- explicit DENY has precedence over ALLOW  
- AWS managed policy  
- user managed policy  

<span style='background:lime;color:black'>user managed policy 分为 Identity policy 和 Resource policy</span>
|resource-based policy|user/identity-based policy|
|----|----|
|applied to a service or resource|when u don't need to manage the credentials for individual users on a granular level|
|附加到 AWS 服务或者资源|直接附加到 user, group, role|
|比如 S3 bucket policy, SNS topic, SQS queue|实际生效的 principal 是 IAM policy 所关联的 user, role|
![example](/assets/img/IMG_20221019-142434898.png)  

<span style='background:lime;color:black'>AWS 托管 policy PowerUserAccess，举个栗子：</span>  
```diff
{
    "Version": "2012-10-17",
    "Statement": [
        {
-           "Effect": "Allow",       <<<< 使用了 Allow & NotAction，不能使用 DENY，会干掉下面的 allow
            "NotAction": [
                "iam:*",
                "organizations:*",
                "account:*"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "iam:CreateServiceLinkedRole",
                "iam:DeleteServiceLinkedRole",
                "iam:ListRoles",
                "organizations:DescribeOrganization",
                "account:ListRegions"
            ],
            "Resource": "*"
        }
    ]
}
```

<span style='background:lime;color:black'>Policy Variables and Tags</span>
一些比较实用的 IAM policy 扩展，比如
`${aws:username}` 表示 policy 所附加的对应 IAM user，常见的使用场景，比如每个 IAM user 有各自的 S3/path，写一个 IAM Policy 仅允许每个 user 访问自己目录下的内容：  
`"Resource":["arn:aws-cn:s3:::bucket-name/${aws:username}/*"]`

## IAM Role 和 Resource based policy 有什么不同
在一个具体场景里面讨论，更容易理解：  
Account A 需要访问 Account B 的 S3 桶，可以通过 assumeRole B，或者 S3 bucket policy 允许 A 访问；  
- assume role(user, application or service)，会丢掉原有 permissions，然后继承 role 的 permissions  
- resource based policy, principal 不会丢掉之前的 permissions  
  
考虑一个新的场景，user in account A needs to scan a DDB table in Account A, and dump it in an S3 bucket in Account B  
以上场景，resource based policy 更合适；如果使用 assumerole，那么将无法 scan DDB table 
其他相似服务还有 SNS topics，SQS queues   

## STS to assume a role
**STS(Security Token Service)** 用来 assume role，比如 A 用户临时提升权限到 B 用户，但是不想让 A 长久拥有 B 的权限
<span style='background:lime;color:black'>assume role 的一般过程：</span>  
- 在本账户或者其他账户创建 IAM role，附加对应的 policy（比如 PowerUserAccess)，指明 role 所拥有的 policy/permissions     
- 在 **Trusted relationships** 指定哪些 principals（可以是本账号、跨账号 IAM User 或者 AWS Service, eg. S3, EC2) 可以 assume role   
- 通过 STS(Security Token Service) 服务来获取临时 credentials(AssumeRole API)，注意 assume role 之后，user 原有的 policy/permissions 会暂时失效      
- 临时 credentials 有效期可以修改，1-12 小时
- 可以对之前存在的 assume role session 进行 revoke，revoke 之后，还是可以建立新的 assume role session  

![STS](/assets/img/IMG_20221019-195956337.png)  

<span style='background:lime;color:black'>举个栗子，Trusted relationships</span>
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws-cn:iam::012345678901:user/dev",
                "Service": "ec2.amazonaws.com.cn"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}
```
<span style='background:lime;color:black'>举个栗子，AWSRevokeOlderSessions</span>    
```diff
- 实现方式是给当前的 IAM Role 添加一条 inline policy，叫做 AWSRevokeOlderSessions；将之前的 session 干掉
- 删除 AWSRevokeOlderSessions，就可以恢复之前的临时 credentials 可用性  
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Deny",
            "Action": [
                "*"
            ],
            "Resource": [
                "*"
            ],
            "Condition": {
                "DateLessThan": {
                    "aws:TokenIssueTime": "2022-10-19T12:03:45.901Z"
                }
            }
        }
    ]
}
```
<span style='background:lime;color:black'>External-ID，避免 混淆代理人问题</span>   
[confused deputy 混淆代理人](https://docs.aws.amazon.com/IAM/latest/UserGuide/confused-deputy.html) 场景：
- 你的账户授权第三方 AWS account 去 assume role，比如做审计  
- 第三方账户和其他 AWS account 有合作关系，比如和 attacker  
- 假设 attacker 提供了你账户的 role ARN，假装告知第三方账号，是 assume role 到 attacker，实际是会 assume role 到你账户的  
- external id 是第三方账户 assume role 时候必须提供的一串 string；string 内容并不是保密/加密的，有权限的人都可以看得到    
- **Access analyzer** 可以用来分析 non-trust-zone（比如第三方账号，不属于你的 organization)，然后给出告警  

![external id](/assets/img/IMG_20221019-210329795.png)  

**Trusted relationships 的部分有 condition**
```diff
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws-cn:iam::012345678901:root"
            },
            "Action": "sts:AssumeRole",
-            "Condition": {
                "StringEquals": {
                    "sts:ExternalId": "dev"
                }
            }
        }
    ]
}
```
**[做一个测试](https://www.youtube.com/watch?v=DLVlW3dOJww)，AWS console 无法在 ExternalID 的情况下 switch role，只能用 AWS CLI, SDK**  
<span style='background:lime;color:black'>能不能写一个 shell 或者 python 来自动化这个测试？</span> 
```diff
- assume role 之前
aws sts get-caller-identity
  {
      "Account": "012345678901",
      "UserId": "ARO:i-0123",
      "Arn": "arn:aws-cn:sts::012345678901:assumed-role/admin/i-0123"
  }
- assume role
aws sts assume-role --role-session-name externalID --role-arn arn:aws-cn:iam::567887657890:role/012345678901-ExternalID --external-id dev
  {
      "AssumedRoleUser": {
          "AssumedRoleId": "ARO:externalID",
          "Arn": "arn:aws-cn:sts::567887657890:assumed-role/012345678901-ExternalID/externalID"
      },
      "Credentials": {
          "SecretAccessKey": "3BU",
          "SessionToken": "IQo",
          "Expiration": "2022-10-19T13:52:17Z",
          "AccessKeyId": "ASIA"
      }
  }
- 使用 assume role 之后临时 credentials
export AWS_ACCESS_KEY_ID=ASIA
export AWS_SECRET_ACCESS_KEY=3BU
export AWS_SESSION_TOKEN=IQo

aws sts get-caller-identity
  {
      "Account": "567887657890",
      "UserId": "ARO:externalID",
      "Arn": "arn:aws-cn:sts::567887657890:assumed-role/012345678901-ExternalID/externalID"
  }
```

**测试一下 python 完成 STS**
参考资源如下：
- [Boto3 Docs, STS](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/sts.html#id8)  
- [AWS IAM 官方示例](https://docs.aws.amazon.com/zh_cn/IAM/latest/UserGuide/id_roles_use_switch-role-api.html)  
- [classmethod 一篇博客](https://dev.classmethod.jp/articles/aws-sdk-for-python-boto3-assumerole/)  
- [StackOverflow 一篇帖子](https://stackoverflow.com/questions/44171849/aws-boto3-assumerole-example-which-includes-role-usage)  
- [AWS Code Sample，获取 print list bucket 的方式](https://docs.aws.amazon.com/code-samples/latest/catalog/python-s3-s3_basics-demo_bucket_basics.py.html)  
- [cppsecrets.com 一篇博客，list bucket 的方式](https://cppsecrets.com/users/167310010511812197110115104117107555064103109971051084699111109/Python-AWS-S3-List-Buckets.php)

```python
#!/usr/bin/python3
import boto3
from boto3.session import Session

sts_client = boto3.client('sts')
response = sts_client.get_caller_identity()["Arn"]
print('origin ARN is {}'.format(response))

IAM_ROLE_ARN = 'arn:aws-cn:iam::567887657890:role/012345678901-ExternalID'
IAM_ROLE_SESSION_NAME = 'foo'
External_ID = 'dev'
REGION = 'cn-northwest-1'

response = sts_client.assume_role(
        RoleArn = IAM_ROLE_ARN,
        RoleSessionName = IAM_ROLE_SESSION_NAME,
        ExternalId = External_ID
        )

credentials = response['Credentials']

session = Session(
            aws_access_key_id=credentials['AccessKeyId'],
            aws_secret_access_key=credentials['SecretAccessKey'],
             aws_session_token=credentials['SessionToken'],
             region_name = REGION
             )

new_client = session.client('sts')
response = new_client.get_caller_identity()["Arn"]
print('assumed role ARN is {}'.format(response))\

s3 = session.client('s3')
buckets = s3.list_buckets()
#for b in buckets['Buckets']:
 #   print(b['Name'])
print('Buckets:\n\t', *[b['Name'] for b in buckets['Buckets']], sep="\n\t")

#ec2 = session.client('ec2')
#ec2_instances = ec2.describe_instances()
#print(ec2_instances)
```
执行效果如下：
```
python3 assumerole.py
origin ARN is arn:aws-cn:sts::012345678901:assumed-role/admin/i-0123
assumed role ARN is arn:aws-cn:sts::567887657890:assumed-role/012345678901-ExternalID/foo
Buckets:

        alb-accesslog-bucket

```
对应的 Cloudtrail 记录如下：
```diff
{
    "eventVersion": "1.08",
    "userIdentity": {
        "type": "AWSAccount",
        "principalId": "ARO:i-0123",
        "accountId": "012345678901"
    },
    "eventTime": "2022-10-20T08:36:45Z",
    "eventSource": "sts.amazonaws.com",
-    "eventName": "AssumeRole",
    "awsRegion": "cn-northwest-1",
    "sourceIPAddress": "x.x.x.x",
    "userAgent": "Boto3/1.22.4 Python/3.7.10 Linux/5.10.109-104.500.amzn2.aarch64 Botocore/1.25.4",
    "requestParameters": {
        "roleArn": "arn:aws-cn:sts::567887657890:assumed-role/012345678901-ExternalID",
        "roleSessionName": "foo",
        "externalId": "dev"
```

## Permission boundary
- 相当于一个笼子，即使是 admin，也可以被关进笼子里，比如不允许 admin terminate EC2  
- AWS support permission boundaries for IAM entities(users or roles)    
- use a managed policy to set the maximum permissions that an identity-based policy can grant to an IAM entity  
- An entity's permissions boundary allows it to perform only the actions that are allowed by both its identity-based policies and its permissions boundaries  
![post-IAM-permission-boundary-example](/assets/img/post-IAM-permission-boundary-example.png)  

## Identity Federation 联合身份认证
AWS 外部 (eg. facebook, google) 用户可以通过联合认证，结合 STS，获取临时 credentials  
外部用户有特定的 username，可以基于这个特性来设置 IAM policy  
AssumeRoleWithSAML
SSO
Cogntio

# AWS Organizations
可能拥有多个 AWS accounts，用于 dev, production, billing 等  
合并账单付费  
[目前 organizations 在中国区的支持并不完善](https://docs.amazonaws.cn/en_us/aws/latest/userguide/organizations.html)  
[如果需要更换 AWS account 所属 organization，网络层面需要注意什么](https://aws.amazon.com/blogs/networking-and-content-delivery/migrating-accounts-between-aws-organizations-from-a-network-perspective/)  

![post-Organization-SAA-example2](post-Organization-SAA-example2.png)  

## Service Control Policy - SCP
在 Organization 当中，通过类似 IAM policy 的形式来定义 OU(organization unit) 当中的 AWS account 的 permission boundary  
如果 organization SCP 定义了 deny action，那么即使 OU 当中 IAM user/role 定义 allow，也会被干掉  
[参考资料](https://www.stormit.cloud/blog/aws-scp-service-control-policy/)  
![Organization and SCP](/assets/img/AWS_organizations_SCP.png)
![post-Organization-SCP-SAA-example](/assets/img/post-Organization-SCP-SAA-example.png)  

# Directory Service, AD
- AWS Directory Service lets you run Microsoft Active Directory (AD) as a managed service.   
- AWS Directory Service for Microsoft Active Directory, also referred to as AWS Managed Microsoft AD, is powered by Windows Server 2012 R2.   
- When you select and launch this directory type, it is created as a highly available pair of domain controllers connected to your virtual private cloud (VPC).   
- The domain controllers run in different Availability Zones in a region of your choice.   
- Host monitoring and recovery, data replication, snapshots, and software updates are automatically configured and managed for you.  

![post-directory-service-AD-SAA](/assets/img/post-directory-service-AD-SAA.png)  

## AWS Directory Service AD Connector
- 本地使用了 Active Directory  
- AD Connector 可以和 AD 集成   
- 集成之后，可以将 IAM Role 指定给 AD user/groups  
- AD Connector can redirect directory requests to your on-premises Microsoft AD  

# Control Tower
对于 initial AWS account setup，[Control Tower](https://aws.amazon.com/controltower/faqs/) 可以提供 best practice  
![Control Tower](/assets/img/Product-Page-Diagram_AWS-Control-Tower.9281926228bb2900c76b4b6d85b2819efc078978.png)

# Compliance 
关于合规性，以及针对 PII(Personal Identifiable Information), PHI(Protected Health Information) 等的保护  

## AWS Macie
- ML-powered security service  
- help you prevent data loss by automatically discovering, classifying, and protecting sevsitive data stored in S3  
- Macie recognize sensitive data such as PII, PHI, assigns a business value and provide visiblity into where this data is stored and how it is being used in your organization.  
- Macie continuously monitors data access activity for anomalies/异常监测，and delivers alers when detects risk of unauthorized access    
- 对比 AWS Rekognition 服务，用来识别人物、text、风景等，不恰当内容检测 in image, videos  

## AWS Artifact
关于 AWS 合规性文档，以及是否接受特定 agreements  

## AWS Textract
[基于机器学习，可以从各种 PDF, 表单来识别、提取文本](https://aws.amazon.com/cn/textract/)  

## AWS Comprehend Medical
[可以从非结构化医学文本当中提取、处理信息](https://aws.amazon.com/cn/comprehend/medical/)  

A hospital recently deployed a RESTful API with Amazon API Gateway and AWS Lambda The hospital uses API Gateway and Lambda to upload reports that are in PDF format and JPEG format The hospital needs to modify the Lambda code to identify protected health information (PHI) in the reports  

<details>
	<summary> Which solution will meet these requirements with the LEAST operational overhead?  
 </summary>
	Use Amazon Textract to extract the text from the reports Use Amazon Comprehend Medical to identify the PHI from the extracted text 
</details>

# Denial-of-service attacks - DDoS
视频里面只是简单说了一些场景，以及 AWS 如何应对  

<span style='background:lime;color:black'>attacker 请求 UDP，比如天气信息，然后 destination 伪装为客户 </span>  
- 设置合理的 Security Group, Network ACL 来避免大多数的 DDOS 攻击  
- 入向流量会首先进入 subnet 也就是首先匹配 Network ACL，假设 permit；之后需要进入网卡也就是匹配 Security Group  
- Security Group 虽然生效在 EC2 ENI 级别，但是 DDOS 攻击实际上是 AWS infrastructure 来承受，并非用户的 EC2  
- 有一个扩展的问题，比如用户的 EC2 ENI 配置了 VPC Flowlog，若被 DDOS，那么 VPC Flowlog 应该会记录很多的 REJECT 条目，这会占用 CloudWatch Logs 或者 S3 的存储费用  

<span style='background:lime;color:black'>HTTP 攻击，attacker 假设自己的速率非常慢，占用 TCP/HTTP 资源，其他正常 requests 无法被处理  </span>   
- ALB，ALB 会和 attacker 建立 TCP 连接，直到完整接收 HTTP request，才转给 backend targets  
- ALB(regional service) 会随着 requests 数量自动扩缩容，而用户不需要为扩容付费  

## Shield
- Shield Standard, layer 4 TCP/UDP 防护   
- Shield Advanced，DDoS 防护，实时查看攻击，和 WAF 集成  

## WAF
[Web Application Firewall](https://www.amazonaws.cn/en/waf/)，OSI 模型中工作在 layer7 的应用层防火墙，可以识别 client IP, HTTP/S headers, body, URI strings 来防范攻击，保护对应的资源。  
目前 WAF 在中国区可以和 API Gateway, ALB 联动，暂不支持 Cloudfront    
WAF 有一个功能是 **rate-based rule**，在最短 5 分钟内最多允许多少 requests，可以用来防范 DDoS 攻击；可以借助 ab(Apache Benchmark) 等工具做测试，实际效果是 30 秒左右就开始间歇性 block 对应 DDoS 流量；假设是 WAF-ALB 方式，可以观察 ALB 相关的 CloudWatch 指标 HTTPCode_ELB_4XX_Count，与对应 WAF rate-based rule 指标的趋势。  
因为默认被 WAF block 的流量，ALB 会返回 HTTP error code 403；图表中 ALB `RequestCount`为 0 的原因，是对应流量被 block 因此不会被 ALB route 到任何一个后端 target，这种情况下`RequestCount`为 0  
![WAF-ALB-CW-Metrics](/assets/img/IMG_20221002-102549928.png)  

### WAF 的行为/Action
<span style='background:lime;color:black'>主要有四类：allow, block, count，captcha</span>    
其中 allow、block 都是 terminated rule，也就是匹配到对应 rule 以后，不再继续向下匹配其他规则；  
count 是 non-terminate rule，只计数，方便后续统计有多少特定类型流量，匹配 count 之后，会继续向下匹配其他 rule，直到 allow/block，或者使用默认的 allow/block 来结束规则匹配。  
captcha 实际是将用户请求重定向到 waf/token 做图形验证码，如果通过 (next time 携带有效 token)，会 allow，否则 deny  

### WAF rules 分类
- Amazon managed 托管规则  
  - 具体配置不对外公开，避免黑客针对性设计攻击手段  
  - 默认 action: block  
  - 可以将默认 action: block 覆盖/override 为 action:count（比如刚开始使用 WAF，用于测试）  
  - 或者使用`scope-down` 来做进一步的规则重新定义  
- 用户自定义规则
  - 可以针对 Layer3/IP, Layer7/HTTP 做进一步定制  

### 使用 WAF 的最佳实践
1. 首先明确需求，比如需要设置白名单、黑名单（默认 action 为 block 或者 allow)；业务流量的特性 (source IP、URI、访问频率 等）  
2. 其次针对性测试，比如将 WAF rule action 定义为 count，从 `Sampled requests` 观察最近 3 小时的采样，或者从 WAF logs 分析；然后根据流量特性做设置，最终将 rule action 定义为 allow 或者 block  

# Data security
数据安全可以区分两个方面，at rest/被存储、静态，in transmit/传输过程中  

## KMS
AWS Key Management Service (AWS KMS)  
**encryption data at rest**，比如和 EBS, S3, RDS 联动  
you can specify which IAM users and roles are able to manage keys  
CMK(Customer Managed Keys) 是 per region 的   

## CloudHSM
- [Hardware Security Module](https://docs.aws.amazon.com/cloudhsm/latest/userguide/introduction.html) in AWS Cloud.  
- generate, store, import, export and manage cryptographic keys, and KMS key materials  
- audit the key usage independently of AWS Ctrail  
- if requere to store the keys that has been validated to FIPS 140-S level 3  
- Meet performance requirements of your applications through [elasticity, adding or removing HSM instances while achieving latency and reliability goals](https://aws.amazon.com/cloudhsm/features/).
  - When you create an AWS CloudHSM cluster with more than one HSM, [you automatically get load balancing](https://docs.aws.amazon.com/cloudhsm/latest/userguide/clusters.html#cluster-high-availability-load-balancing)
![post-CloudHSM-performance-scaling-load-balance](/assets/img/post-CloudHSM-performance-scaling-load-balance.png)
![post--CloudHSM-performance-scaling-load-balance-example](/assets/img/post--CloudHSM-performance-scaling-load-balance-example.png)  

## GuardDuty
- provides intelligent **threat detection** for your AWS infrastructure and resources(eg. instances, container workloads, users, storage). 
- [Amazon GuardDuty is a continuous security monitoring service that analyzes and processes data sources, such as AWS CloudTrail data events for Amazon S3 logs, CloudTrail management event logs, DNS logs, Amazon EBS volume data, Kubernetes audit logs, and Amazon *VPC flow logs*.](https://docs.aws.amazon.com/guardduty/latest/ug/what-is-guardduty.html)
- 可以看到 GuardDuty 并没有传统 IDS 对于流量的监控，应该是通过分析 VPC flowlog 来实现；[通过一些第三方测试，GuardDuty 可以实现传统 IDS 的功能](https://aws.amazon.com/blogs/security/new-third-party-test-compares-amazon-guardduty-to-network-intrusion-detection-systems/) 
- 可以通过 master account 来管理一批账号的设置  
- [基于 Amazon GuardDuty 威胁级别的自动化通知](https://aws.amazon.com/cn/blogs/china/automated-notification-based-on-amazon-guardduty-threat-level/)
![guardduty](/assets/img/IMG_20220412-220302613.png)  
![post-VPC-flowlog-GuardDuty-example](/assets/img/post-VPC-flowlog-GuardDuty-example.png)

## AWS Inspector
- [automated and continual vulnerability management at scale](https://aws.amazon.com/inspector/?nc1=h_ls)  
- [大规模、自动化、持续的漏洞管理](https://docs.aws.amazon.com/inspector/latest/user/what-is-inspector.html)  
- 自动扫描 EC2, ECR 当中 container image 的软件漏洞，意外的网络暴露  

![post-aws-inspector-how-it-works](/assets/img/post-aws-inspector-how-it-works.png)  
![post-AWS-Inspector-example](/assets/img/post-AWS-Inspector-example.png)  

## SecurityHub
provides a consolidated view of your security status in AWS. Automate security checks, manage security findings, and identify the highest priority security issues across your Amazon Web Services environment.  

# ACM - Certification

# KMS - Key mgmt service

# 参考文档
[AWS Security WhitePaper](https://docs.aws.amazon.com/whitepapers/latest/introduction-aws-security/welcome.html)  
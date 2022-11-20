---
layout:         post
title:          AWS Compute and Serverless
subtitle:		    备考 SAA 的笔记 EC2, ECS/EKS, Lambda
date:           2022-06-30
author:         Luke
cover:          '/assets/img/IMG_20220630-200347542.png'
tags:           AWS, SAA, EC2, Lambda, Serverless, ECS, EKS, Container
---
- [Compute Service 分类](#compute-service-分类)
  - [Difference between Containers and VMs](#difference-between-containers-and-vms)
  - [Difference between Servessless and VMs](#difference-between-servessless-and-vms)
- [VM - EC2](#vm---ec2)
  - [EC2 types](#ec2-types)
  - [EC2 Pricing](#ec2-pricing)
  - [Amazon Compute Optimizer 提出对 EC2 的建议](#amazon-compute-optimizer-提出对-ec2-的建议)
  - [EC2 Lifecycle](#ec2-lifecycle)
  - [Snapshot and AMI](#snapshot-and-ami)
  - [EC2 SG, ACL](#ec2-sg-acl)
  - [ENI, ENA, EFA](#eni-ena-efa)
  - [EC2 Placement Group 置放群组](#ec2-placement-group-置放群组)
  - [ASG - AutoScaling Group](#asg---autoscaling-group)
    - [ASG 扩缩容策略](#asg-扩缩容策略)
    - [ASG 缩容，终止 EC2 的策略](#asg-缩容终止-ec2-的策略)
    - [ASG 的一些重要参数](#asg-的一些重要参数)
- [Containers](#containers)
  - [ECS - Elastic Container Service](#ecs---elastic-container-service)
  - [EKS - Elastic Kubernetes Service](#eks---elastic-kubernetes-service)
    - [EKS add-on for ELB](#eks-add-on-for-elb)
- [Serveless](#serveless)
  - [Fargate](#fargate)
  - [Lambda](#lambda)
    - [How AWS Lambda works](#how-aws-lambda-works)
    - [AWS Lambda function handler](#aws-lambda-function-handler)
    - [Environment variables](#environment-variables)
  - [Elastic Beanstalk](#elastic-beanstalk)
  - [API Gateway](#api-gateway)
    - [一些组件](#一些组件)
- [SAM - Serverless Application Model](#sam---serverless-application-model)
- [不同架构举例](#不同架构举例)
- [Lambda, ECS，EKS 取舍](#lambda-ecseks-取舍)

# Compute Service 分类  
[SAA 培训内容](https://explore.skillbuilder.aws/learn/course/1851/play/45289/aws-technical-essentials-104)  

At a fundamental level, three types of compute options are available  
**virtual machines (VMs)**   / EC2  
**container services**       / EKS, ECS  
**serverless**               / Lambda, Fargate  

What are the four main factors you should take into consideration when choosing a Region?  
**Compliance, Latency, price, service availability**  

## Difference between Containers and VMs
![Containers_VM](/assets/img/IMG_20220411-205413963.png)  

Containers **share** the same operating system and kernel as the host they exist on, whereas virtual machines contain their own operating system.   
Each virtual machine must maintain a copy of an operating system, which results in a degree of wasted resources.  
A container is more **lightweight**. They spin up quicker, almost instantly. This difference in startup time becomes instrumental when designing applications that need to scale quickly during input/output (I/O) bursts.  
While containers can provide speed, virtual machines offer the full strength of an operating system and more resources, like package installation, dedicated kernel, and more  

## Difference between Servessless and VMs  
![Servessless_VM](/assets/img/IMG_20220411-205606892.png)  

# VM - EC2  
## EC2 types  
![some of the EC2 types](/assets/img/IMG_20220411-213020100.png)  
Spot instance 在被 AWS terminate 后，对应的 hour 不收费  
on demand instance，用户自行 terminate，对应 hour 是收费的 

## EC2 Pricing
- 如果 application 可以适应随时中断，__spot instance__ 竞价实例类型可能比较便宜，[最高可享受按需价格 90％ 的折扣](https://www.amazonaws.cn/ec2/pricing/?nc1=h_ls)  
![EC2 pricing SAA example](/assets/img/post-EC2-pricing-spot.png)  

- 如果未来一段时间的用量可以预期，并且要求 EC2 稳定运行，可以考虑 [Savings Plans](https://docs.aws.amazon.com/zh_cn/whitepapers/latest/cost-optimization-reservation-models/savings-plans.html)，可以节约大概 66% - 72% 的费用  
- __Saving Plans__ 有两种形式，主要区别是 **Compute Saving Plans** 适用于 AWS account 下 EC2, Fargate, Lambda 等计算资源，可能更加灵活；**EC2 Instance Saving Plans** 只针对 EC2  
![EC2 pricing SAA example](/assets/img/post-EC2-pricing-SAA.png)  

- 稳定使用的 EC2，还可以考虑 __RI__ 预留实例，相比按需实例，预留实例为您提供大幅折扣（最高 72%）  
- convertible RI/可转换 RI，可以在未来转换为其他 instance family  
- __Scheduled Reserved Instances (Scheduled Instances)__ enable you to purchase capacity reservations that recur on a daily, weekly, or monthly basis, with a specified start time and duration, for a one-year term.   
- You reserve the capacity in advance, so that you know it is available when you need it. You pay for the time that the instances are scheduled, even if you do not use them.    
- You can’t stop or reboot Scheduled Instances, but you can terminate them manually as needed. If you terminate a Scheduled Instance before its current scheduled time period ends, you can launch it again after a few minutes. Otherwise, you must wait until the next scheduled time period.  
![EC2 pricing SAA example](/assets/img/post-EC2-pricing-RI.png)  

## Amazon Compute Optimizer 提出对 EC2 的建议  
客户自行选择 EC2 types，在使用一段时间之后，可以根据 console 工具 - Compute Optimizer 来查看信息，基于 EC2 使用率、成本优化等的建议  
![Compute Optimizer](/assets/img/IMG_20220420-173120029.png)  
![EC2-EBS](/assets/img/IMG_20220420-173324986.png)  

## EC2 Lifecycle
When you stop your instance, the data stored in memory (RAM) is lost.  
__Hibernate 休眠__，[记录当前状态，类似于当前快照，避免 memory/RAM 当中数据丢失](https://docs.amazonaws.cn/en_us/AWSEC2/latest/UserGuide/Hibernate.html)；  
- 从 hibernate 恢复，可以避免 1. OS 层面启动的延迟；2. application 启动的延迟。 
- 最长 hibernate 60 天，适用于 on-demand, RI instances  
- RAM 最多不能超过 150 GB   
- 保证 root EBS 有足够空间来存储 RAM 的数据  
- 创建 EC2 时候，需要勾选“休眠”  
- root volume 必须启用加密      

When you stop-hibernate an instance, AWS signals the operating system to perform hibernation (suspend-to-disk), which **saves the contents from the instance memory (RAM) to the Amazon EBS root volume**.  

![Hiberante_cycle](/assets/img/IMG_20220526-150521856.png)

![Enable_Hibernate_when_launch_EC2](/assets/img/IMG_20220526-151216558.png)  

__Terminate 终止__，销毁 EC2 实例，默认情况下，EC2 根卷 EBS 的 DeleteOnTermination 是 enable 状态，即 root EBS 随着 EC2 消失，无法找回。数据卷 EBS 的  DeleteOnTermination 默认 disabled。  
tips 避免数据丢失：  
  - launch EC2 时候，关闭 root EBS DeleteOnTermination   
  - 启用  EC2 terminate protection(DisableApiTermination)  
  - 使用 lifecycle 功能，定期做 AMI，移除过期 AMI  
  - 做 snapshots  

![EC2 lifecycle](/assets/img/IMG_20220411-213159099.png)  

## Snapshot and AMI
保存 EC2/VM 当前配置的一种手段  

**Snapshot**
- Snapshot 是将当前 EC2 状态保存快照，上传到 AWS managed S3  
- Snapshot 打快照是立刻完成的，但是需要 asynchronously 上传到 S3，
  - 初次打快照，上传的过程可能持续 minutes to hours  
  - 后续打快照，__增量更新__，完成速度取决于有多少内容发生了变化   
- Snapshot 操作并不影响 EC2 使用 (ongoing volumes reads and writes)  
- 从 Snapshot 恢复 volume，实际上是从 AWS managed S3 拉取数据的过程
  - replicated volume loads data lazily in background so that you can begin using it immediately.  
  - 如果所访问的数据还没有从 S3 拉取，将会触发 loading  

**AMI**
- Amazon machine Image，可以简单理解为 EC2 的 OS/操作系统  
- 新启动的 EC2，可以根据业务需要做定制化，比如安装一些软件，修改特定配置；之后可以做一个 AMI，从 AMI 启动的后续新的 EC2，将拥有相同的之前修改的配置  
- 注意导出 AMI 时候，EC2 可以选择 __No Reboot__，默认 EC2 会重启  

## EC2 SG, ACL
Security Group is instance(ENI) level firewall. Stateful(Connection track), only allow  
Network ACL act as Subnet level firewall. Stateless, allow and deny  

## ENI, ENA, EFA
![ENI, ENA, EFA exam tips](/assets/img/IMG_20220526-110513168.png)  
[ENA - 增强联网](https://docs.amazonaws.cn/zh_cn/AWSEC2/latest/UserGuide/enhanced-networking.html)，single root I/O Virtualization(SR-IOV) 提供更高网络性能，没有额外费用。可选 ENA/up to 100G, Intel 82599 Virtual Function(VF) intf/up to 10G  
non-Nitro 平台的 EC2（比如 t2.micro) 升级 Nitro 平台（比如 t3.micro)，需要启用 ENA。  
[EFA通常用于HPC, 机器学习场景，instance可以bypass OS 直接和网卡硬件交互](https://docs.aws.amazon.com/zh_cn/AWSEC2/latest/UserGuide/efa.html)  

## EC2 Placement Group 置放群组
为了满足特定 workload 需求，可以将 EC2 逻辑上放到同一个 [placement group](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/placement-groups.html)，优势可能包括低延迟、高吞吐量  
>There is no charge for creating a placement group.

**Cluster-集群** 
- instances inside same AZ  
- low-latency network performance
- HPC  
![post-EC2-placement-example](/assets/img/post-EC2-placement-example.png)  

---

**Partition-分区**
- 将实例分布在不同的逻辑分区上，以便一个分区中的实例组不会与不同分区中的实例组使用相同的基础硬件  
- 该策略通常为大型分布式和重复的工作负载所使用  
- 例如，Hadoop、Cassandra 和 Kafka  

---

**Spread-分布** 
- 将一小组实例严格放置在不同的基础硬件上以减少相关的故障  
- 可以跨可用区  

## ASG - AutoScaling Group
- 自动扩展组，可以根据配置，对 EC2 的数量进行扩缩容  
- 可以和 ELB 配合使用  

### ASG 扩缩容策略
**simple policy**  
- 通过设置 minimum/desired/maximum，来指定所需要的 EC2 的数量  
- 在一些简单的场景适用，比如业务需要 10 台 EC2，可以设置 desired=10  

**scheduled policy**  
- 周期性的业务变化，比如工作日早晨的流量高峰，月底的流量高峰  
- 通过设置 minimum/desired/maximum，来指定所需要的 EC2 的数量  

![post-ASG-scheduled-policy-SAA](/assets/img/post-ASG-scheduled-policy-SAA.png)  

**step policy**  

**target tracking policy**  
- 根据监控设置，比如 CPU 40%，将 EC2 资源利用率维持在一个相对稳定的水平  

![post-ASG-target-tracking-SAA](/assets/img/post-ASG-target-tracking-SAA.png)   

### ASG 缩容，终止 EC2 的策略
- [default termination policy](https://docs.aws.amazon.com/autoscaling/ec2/userguide/ec2-auto-scaling-termination-policies.html)  
- 如果 EC2 在多个 AZ，会尽量保证 AZ 的 EC2 数量平衡  
- 如果 AZ 的 EC2 数量相同，那么 terminate 没有被 scale in protection 的，使用最老 launch configuration/template 的 EC2  
- 如果存在多个满足以上条件的 EC2，那么寻找最接近下一个 billing hour 的 EC2，节省费用  
- 如果多个 EC2 接近 next billing hour，那么随机干掉一个  

### ASG 的一些重要参数
**scale-in protection**  
- 如果 EC2 因为故障需要替换，ASG 不会主动 terminate failure EC2，可以登录查看对应日志  
- 手动加入 ASG 的 EC2，这个选项默认打开  

**Lifecycle hooks**  
- Use lifecycle hooks to put instances into a wait state and perform custom actions on the instances.   
- Custom actions are performed as the instances launch or before they terminate.   
- Instances remain in a wait state until you either complete the lifecycle action, or the timeout period ends(default 1 hour).  
- To create lifecycle hooks for scaling out (instances launching) and scaling in (instances terminating), you must create two separate hooks.  
- 如果希望在 ASG 启动、终止 EC2 时候，执行一些脚本/custom script，比如发送通知，或者收集日志  

**Activity Notification**
- EC2 launch, terminate, fail to launch, fail to terminate 时候，发送 SNS 通知  
- 相当于开箱即用的 Lifecycle hooks，不需要自己写 custom script 来发送通知  

![Activity notification](/assets/img/post-ASG-Activity-Notification.png)  

**Warm Pool**  
- A warm pool helps you balance performance and cost for applications with long first boot times.  
- A warm pool is a pool of pre-initialized EC2 instances.   
- When there is an increase in traffic to your application, an Auto Scaling group can draw on the warm pool to meet increases to its desired capacity.   
- The goal of a warm pool is to ensure that instances are ready to quickly start serving application traffic, accelerating the response to a scale-out event.  

# Containers
对比 EC2，containers 启动更快。  
**一次编译，在任意位置运行**。Development -  QA - Production , 应该不会因为 code environment 出现问题。    
可以在 EC2 运行多个 containers，将 EC2 分布在不同 AZ，container 或者 EC2 出问题，需要 Orchestrate。  
ECS, EKS 都是 containers 的 协调，管理工具/ 容器编排。  

如果不想要操心底层，可以使用 fargate(Serverless)  

A container is a standardized unit that packages your code and its dependencies. This package is designed to run reliably on any platform, because the container creates its own independent environment. With containers, workloads can be carried from one place to another, such as from development to production or from on premises to the cloud.  

## ECS - Elastic Container Service
Amazon ECS is an end-to-end container orchestration service that helps you spin up new containers and manage them across a cluster of EC2 instances.    
To run and manage your containers, you need to install the Amazon **ECS container agent** on your EC2 instances. This agent is open source and responsible for communicating to the Amazon ECS service about cluster management details. You can run the agent on both Linux and Windows AMIs. An instance with the container agent installed is often called a **container instance**.  

Once the Amazon ECS container instances are up and running, you can perform actions that include, but are not limited to, launching and stopping containers, getting cluster state, scaling in and out, scheduling the placement of containers across your cluster, assigning permissions, and meeting availability requirements.  

SAA 考试题举个栗子，比如 ECS 需要调用 database 密码，应该怎么设置  
Use the AWS System Manager Parameter Store to keep the database credentials and then encrypt thme using AWS KMS. Create an IAM Role for your ECS task execution role(taskRoleArn) and reference it with your task defination, which allows access to both KMS and the Parameter Store.
Within your container defination, specify secrets with the name of the environment variable to set in the container and the full ARN of the SSM Parameter Store containing the sevsitive data to present to the container.
也可以将敏感数据保存到 AWS Secrets Manager  
注意 ECS 不支持 resource based policy(example, S3 bucket policy)，ECS task 通过 assume role 的方式来获取权限  

![ECS Cluster](/assets/img/IMG_20220411-212704829.png)  

To prepare your application to run on Amazon ECS, you create a **task definition**. The task definition is a text file, in JSON format, that describes one or more containers. A task definition is similar to a blueprint that describes the resources you need to run a container, such as CPU, memory, ports, images, storage, and networking information.  

## EKS - Elastic Kubernetes Service
Kubernetes is a portable, extensible, open source platform for managing containerized workloads and services.  
If you already use Kubernetes, you can use Amazon EKS to orchestrate the workloads in the AWS Cloud. If you have containers running on Kubernetes and want an advanced orchestration solution that can provide simplicity, high availability, and fine-grained control over your infrastructure, Amazon EKS could be the tool for you.  
Amazon EKS is conceptually similar to Amazon ECS, but with the following differences:  
  - An EC2 instance with the ECS agent installed and configured is called a container instance. In Amazon EKS, it is called a **worker node**.  
  - An ECS container is called a task. In Amazon EKS, it is called a **pod**.  
  - While Amazon ECS runs on AWS native technology, Amazon EKS runs on top of Kubernetes.  

![EKS SAA example](/assets/img/post-EKS-SAA.png)  

### EKS add-on for ELB
- `AWS Load Balancer Controller` optional add-on in EKS cluster that manages AWS ELB for EKS  
  - Kubernetes Ingress  
    - Load Balancer Controller will provision ALB  
  - Load Balancer  
    - Load Balancer Controller will provision NLB

# Serveless
合适的才是最好的，关注业务本身即可  
Every definition of serverless mentions the following four aspects:  
  - No servers to provision or manage  
  - Scales with usage  
  - You never pay for idle resources  
  - Availability and fault tolerance are built-in  

## Fargate
无服务器运算，EC2 的选型，运维由 AWS 负责，客户需要配置，维护 Container。  
AWS Fargate is a **purpose-built** serverless compute engine for containers.   
Fargate scales and manages the infrastructure, allowing developers to work on what they do best – application development. It achieves this by allocating the right amount of compute, eliminating the need to choose and handle EC2 instances and cluster capacity, and scaling. Fargate supports both Amazon ECS and Amazon EKS architecture, and provides workload isolation and improved security by design.  
目前 EKS Fargate 在 AWS 中国区还不可用。  

## Lambda
适合场景：  
  - 既不想维护 EC2，也不想维护 Container   
  - 特定事件触发代码，短时间运行  
    - 比如不需要部署 EC2 来 7x24 小时监控是否有 file upload to S3  
    - Lambda 更加 lightweight，扩容比 EC2 ASG, ECS, EB 都要快一些  
  
有丰富的 trigger，比如 HTTP request，upload files to S3 bucket  
代码运行时收费，最长运行 15 分钟。  
AWS rounds up duration to the nearest millisecond with no minimum execution time. With this pricing, it can be cost effective to run functions whose execution time is very low, such as functions with durations under 100 ms or low latency APIs  

![Lamdba](/assets/img/IMG_20220411-214122122.png)  
![build Lambda Functions](/assets/img/IMG_20220513-173152878.png)  
Access Permissions: Define which events can invoke the functin and what services Lambda is permitted to interact with.  

![post-Lambda-SAA](/assets/img/post-Lambda-SAA.png)  

### How AWS Lambda works
[参考链接](https://explore.skillbuilder.aws/learn/course/99/play/497/aws-lambda-foundations)  
- A Lambda function has three primary components – **trigger, code, and configuration**    
- **code** 决定了 Lambda 的功能，以及使用什么 runtime 比如 Python  
- 如果 Lambda 需要调用 VPC 内资源，需要将 Lambda 放到对应 VPC，并且按需配置 security group  

The **configuration** of a Lambda function consists of information that describes how the function should run. In the configuration, you specify network placement, environment variables, memory, invocation type, permission sets, and other configurations  

**Triggers** describe when a Lambda function should run. A trigger integrates your Lambda function with other AWS services, enabling you to run your Lambda function in response to certain API calls that occur in your AWS account. This increases your ability to respond to events in your console without having to perform manual actions. All you need is the what, how, and when of a Lambda function to have functional compute capacity that runs only when you need it to.
trigger 实际上应该和 Event Sources 相似  

### AWS Lambda function handler
The AWS Lambda function handler is **the method in your function code that processes events**. When your function is invoked, Lambda runs the handler method. When the handler exits or returns a response, it becomes available to handle another event.   
You can use the following general syntax when creating a function handler in Python.  
![lambda handler example](/assets/img/post-Lambda-handler.png)  

### Environment variables
- 环境变量，可能存在有一些敏感信息，比如密钥    
- 可以通过 KMS 加密，确保 vars 在 AWS mgmt console 不是明文的  
  - 如果创建 lambda 时候带有 Env vars，lambda & KMS 默认会创建一个 default service key 用于加密；但是如果 user 同时有 default KMS key 权限和 Lambda 权限，就可以看到明文信息  
  - 推荐自定义

![Env Vars KMS](/assets/img/post-Lambda-Env-Vars-KMS.png)

## Elastic Beanstalk
对于 developer 来说，如果不希望关注 infrastructure 比如 EC2, ELB, scaling, health check 等，可以交给 EB 负责；
developer 一键式部署应用即可

## API Gateway
从分类来讲，[API GW](https://www.amazonaws.cn/api-gateway/?nc1=h_ls) 可以放到 Networking & Content Delivery，不过和 Lambda 集成多一些，所以放这里了   
fully managed service/[完全托管服务](https://www.youtube.com/watch?v=1XcpQHfTOvs)，使用场景是创建、发布、维护、监控、保护任意规模的 API  
APIs act as "front door" for applications to access data/function from your backend services. API 作为 app 访问后端数据/服务的“前门”，后端可以是 EC2, Lambda, 或者任何 web application  
client -- API GW -- Lambda/EC2/DDB  

### 一些组件
<span style='background:lime;color:black'>Deployment types/部署类型</span>
不同类型对应不同的场景，[参考文档](https://www.sentiatechblog.com/amazon-api-gateway-types-use-cases-and-performance)  
- Edge-optimized endpoint
  - client -- CF -API GW
  - reduced latency for requests from around the world
- Regional endpoint
  - reduced latency for requests that originate in the same region
  - can cfg your own CDN/CF
  - can protect with WAF
- Private endpoint
  - securely expose your REST APIs only to other services within your VPC/DX

<span style='background:lime;color:black'>REST API 调用</span>
![REST API](/assets/img/IMG_20221012-163914187.png)  

<span style='background:lime;color:black'>API Caching, throttle</span>
- API GW 从 Endpoint 拿到信息之后，第二次 client 请求就不会回源了，类似于 CF PoP 
- API GW is low cost and scales automatically
- could  **throttle** API GW to prevent attacks   
  - steady-state requests rate 10,000 per second  
  - maximum concurrent requests is 5,000 across all APIs within an AWS account  
  - if exceed the limit, got `429 Too Many Requests` error response; client could resubmit the failed requests    
![caching](/assets/img/IMG_20221012-171018008.png)  

<span style='background:lime;color:black'>Usage Plans and API Keys</span>
- 区分不同的用户，对 VIP 用户提供更多/更好的服务  
![API Keys](/assets/img/IMG_20221012-171905991.png)  

<span style='background:lime;color:black'>Same Origin Policy</span>
- [同源策略](https://developer.mozilla.org/zh-CN/docs/Web/Security/Same-origin_policy)，防止 Cross-Site-Scripting/XSS 攻击  
- **从 client browser 层面来实现**，不过 Postman/curl 可能会忽略 XSS  
- web browser permits scripts contained in a first web page to access data in a second web page, but only if both web pages have the same origin/domain name. 同源/同域名，同 scheme(http, https)，浏览器才认为 XSS 有效  

<span style='background:lime;color:black'>CORS</span>
- Cross-origin Resource Sharing，[跨域资源共享](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/CORS)   
- **server 层面来控制**，相当于在特定条件下 bypass Same Origin Policy  
- let servers describe which origins are permitted to read that information from a web browser，所以实际上 CORS 是 enforced by client/broswer  
- allows restricted resources(eg. fonts, images) on a web page to be requested from another domain outside the domain from which the first resource was served.  
- when use Javascript/AJAX that uses multiple domains with API GW, ensure you have enabled CORS on API GW

![CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS/cors_principle.png)  
![post-APIGW-CORS-SAP-example1](/assets/img/post-APIGW-CORS-SAP-example1.png)  

# SAM - Serverless Application Model
类似于 EB，借助 CloudFormation 来部署 serverless 应用的一种方式  
从实现的角度来说，通过 [SAM](https://aws.amazon.com/serverless/sam/) 部署，可以不那么关注 Lambda, API GW 的单独部署  
SAM 还可以将测试环境部署到 local(docker)，节省 AWS 资源/cost    
![SAM Template](/assets/img/IMG_20221015-133001967.png)  

# 不同架构举例
视频里面通过 ELB_VM/EC2_DB_S3 的方式，讲解了一种通用的 3-tires 架构  
![ELB_VM_DB](/assets/img/IMG_20220411-205732780.png)  

API_Gateway 以及 Lambda  
![API_Gateway_Lambda](/assets/img/IMG_20220411-205931420.png)  

有些客户选择 ECS fargate，两层 ALB，应该是为了区分 presentation and application 层，避免服务器因为不同 workload 而过载。  
![ELB_ECS_internal_ELB_ECS](/assets/img/IMG_20220411-210844117.png)  

# Lambda, ECS，EKS 取舍
![Containers_or_Lambda](/assets/img/IMG_20220504-104738196.png)  

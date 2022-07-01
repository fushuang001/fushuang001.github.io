---
layout:         post
title:          AWS Compute and Serverless
subtitle:		    备考 SAA 的笔记 EC2, ECS/EKS, Lambda
date:           2022-06-30
author:         Forest
cover:          '/assets/img/IMG_20220630-200347542.png'
tags:           AWS, SAA, EC2, Lambda, Serverless, ECS, EKS, Container
---

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

## Amazon Compute Optimizer 提出对 EC2 的建议  
客户自行选择 EC2 types，在使用一段时间之后，可以根据 console 工具 - Compute Optimizer 来查看信息，基于 EC2 使用率、成本优化等的建议  
![Compute Optimizer](/assets/img/IMG_20220420-173120029.png)  
![EC2-EBS](/assets/img/IMG_20220420-173324986.png)  

## EC2 lifecycle
When you stop your instance, the data stored in memory (RAM) is lost.  
__Hibernate 休眠__，[记录当前状态，类似于当前快照，避免 memory/RAM 当中数据丢失](https://docs.amazonaws.cn/en_us/AWSEC2/latest/UserGuide/Hibernate.html)；  
- 从 hibernate 恢复，可以避免 1. OS 层面启动的延迟；2. application 启动的延迟。 
- 最长 hibernate 60 天，适用于 on-demand, RI instances  
- RAM 最多不能超过 150GB   
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

## EC2 SG, ACL
Security Group is instance(ENI) level firewall. Stateful(Connection track), only allow  
Network ACL act as Subnet level firewall. Stateless, allow and deny  

## ENI, ENA, EFA
![ENI, ENA, EFA exam tips](/assets/img/IMG_20220526-110513168.png)  
[ENA - 增强联网](https://docs.amazonaws.cn/zh_cn/AWSEC2/latest/UserGuide/enhanced-networking.html)，single root I/O Virtualization(SR-IOV) 提供更高网络性能，没有额外费用。可选 ENA/up to 100G, Intel 82599 Virtual Function(VF) intf/up to 10G  
non-Nitro 平台的 EC2（比如 t2.micro) 升级 Nitro 平台（比如 t3.micro)，需要启用 ENA。  

# Containers
对比 EC2，containers 启动更快。  
**一次编译，在任意位置运行**。Development -  QA - Production , 应该不会因为 code environment 出现问题。    
可以在 EC2 运行多个 containers，将 EC2 分布在不同 AZ，container 或者 EC2 出问题，需要 Orchestrate。  
ECS, EKS 都是 containers 的 协调，管理工具/ 容器编排。  

如果不想要操心底层，可以使用 fargate(Serverless)  

A container is a standardized unit that packages your code and its dependencies. This package is designed to run reliably on any platform, because the container creates its own independent environment. With containers, workloads can be carried from one place to another, such as from development to production or from on premises to the cloud.  

## Manage containers with Amazon Elastic Container Service
Amazon ECS is an end-to-end container orchestration service that helps you spin up new containers and manage them across a cluster of EC2 instances.    
To run and manage your containers, you need to install the Amazon **ECS container agent** on your EC2 instances. This agent is open source and responsible for communicating to the Amazon ECS service about cluster management details. You can run the agent on both Linux and Windows AMIs. An instance with the container agent installed is often called a **container instance**.  

Once the Amazon ECS container instances are up and running, you can perform actions that include, but are not limited to, launching and stopping containers, getting cluster state, scaling in and out, scheduling the placement of containers across your cluster, assigning permissions, and meeting availability requirements.  

![ECS Cluster](/assets/img/IMG_20220411-212704829.png)  

To prepare your application to run on Amazon ECS, you create a **task definition**. The task definition is a text file, in JSON format, that describes one or more containers. A task definition is similar to a blueprint that describes the resources you need to run a container, such as CPU, memory, ports, images, storage, and networking information.  

## Use Kubernetes with Amazon Elastic Kubernetes Service
Kubernetes is a portable, extensible, open source platform for managing containerized workloads and services.  
If you already use Kubernetes, you can use Amazon EKS to orchestrate the workloads in the AWS Cloud. If you have containers running on Kubernetes and want an advanced orchestration solution that can provide simplicity, high availability, and fine-grained control over your infrastructure, Amazon EKS could be the tool for you.  
Amazon EKS is conceptually similar to Amazon ECS, but with the following differences:  
  - An EC2 instance with the ECS agent installed and configured is called a container instance. In Amazon EKS, it is called a **worker node**.  
  - An ECS container is called a task. In Amazon EKS, it is called a **pod**.  
  - While Amazon ECS runs on AWS native technology, Amazon EKS runs on top of Kubernetes.  

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
  - 特定事件触发代码，短时间运行。  
  - 比如不需要部署 EC2 来 7x24 小时监控是否有 file upload to S3  
  
有丰富的 trigger，比如 HTTP request，upload files to S3 bucket  
代码运行时收费，最长运行 15 分钟。  
AWS rounds up duration to the nearest millisecond with no minimum execution time. With this pricing, it can be cost effective to run functions whose execution time is very low, such as functions with durations under 100 ms or low latency APIs  

![Lamdba](/assets/img/IMG_20220411-214122122.png)  
![build Lambda Functions](/assets/img/IMG_20220513-173152878.png)  
Access Permissions: Define which events can invoke the functin and what services Lambda is permitted to interact with.  

### How AWS Lambda works
[参考链接](https://explore.skillbuilder.aws/learn/course/99/play/497/aws-lambda-foundations)  
A Lambda function has three primary components – **trigger, code, and configuration**。  
**code** 决定了 Lambda 的功能，以及使用什么 runtime 比如 Python  

The **configuration** of a Lambda function consists of information that describes how the function should run. In the configuration, you specify network placement, environment variables, memory, invocation type, permission sets, and other configurations  

**Triggers** describe when a Lambda function should run. A trigger integrates your Lambda function with other AWS services, enabling you to run your Lambda function in response to certain API calls that occur in your AWS account. This increases your ability to respond to events in your console without having to perform manual actions. All you need is the what, how, and when of a Lambda function to have functional compute capacity that runs only when you need it to.
trigger 实际上应该和 Event Sources 相似  

### AWS Lambda function handler
The AWS Lambda function handler is **the method in your function code that processes events**. When your function is invoked, Lambda runs the handler method. When the handler exits or returns a response, it becomes available to handle another event.   
You can use the following general syntax when creating a function handler in Python.  
```
import boto3
region = 'cn-north-1' 
instances = ['i-12345cb6d1111', 'i-08c2222222'] 
def lambda_handler(event, context):
ec2 = boto3.client('ec2', region_name=region)
ec2.start_instances(InstanceIds=instances)
print 'started your instances: ' + str(instances)
```

# 不同架构举例
视频里面通过 ELB_VM/EC2_DB_S3 的方式，讲解了一种通用的 3-tires 架构  
![ELB_VM_DB](/assets/img/IMG_20220411-205732780.png)  

API_Gateway 以及 Lambda  
![API_Gateway_Lambda](/assets/img/IMG_20220411-205931420.png)  

有些客户选择 ECS fargate，两层 ALB，应该是为了区分 presentation and application 层，避免服务器因为不同 workload 而过载。  
![ELB_ECS_internal_ELB_ECS](/assets/img/IMG_20220411-210844117.png)  

# Lambda, ECS，EKS 取舍
![Containers_or_Lambda](/assets/img/IMG_20220504-104738196.png)  
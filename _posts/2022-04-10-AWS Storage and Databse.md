---
layout:         post
title:          AWS 亚马逊云 Storage 和 Database 信息
subtitle:		    备考 SAA 的笔记
date:           2022-04-10
author:         Forest
cover:          '/assets/img/post-aws-storage-db-bg.png'
tags:           AWS, SAA, Storage, EBS, S3, Database, RDS, DynamoDB
---

# AWS Storage as a Service
[AWS technical essenticals](https://explore.skillbuilder.aws/learn/course/1851/play/45289/aws-technical-essentials-104)

## File storage
类比 MacOS Finder，NAS(network attached storage)
文件有目录结构/tree-like hierarchy，
按照目录-子目录-文件来寻址
metadata like filename, date, size, type...
更新文件内容，本质上是替换
适合多个 hosts 文件共享，sotage mounted onto multiple hosts

### instance store
费用是包含在 EC2 里面的，重启即丢失。
ephemeral block storage

### EFS for linux, NFS file system
一般情况下，EBS 只能够单次 attach 到单独 EC2
如果需要 attach to multiple EC2 the same time，可以考虑 EFS
[EFS FAQs](https://aws.amazon.com/efs/faq/)

EFS, FSx 都是按照用量付费，自动按需扩展容量；不需要像 EBS 一样 provision storage in advance.

### FSx for Windows, SMB protocol
Fully managed file server built on Windows Server that supports the SMB protocol
[FSx for Windows file server FAQs](https://aws.amazon.com/fsx/windows/faqs/?nc=sn&loc=8)

## Block storage
类比 SAN(storage area network)，
将文件分为固定大小的块/splits files into fixed-size chunks,
每一块都有自己的地址；按照地址来查询数据
metadata 只有 chunk address，所以更快
更新文件内容，本质上是替换 specified chunk，和其他 chunks 无关
适合低延迟、高性能场景

### EBS use case
Amazon EBS is useful when you must **retrieve data quickly** and **have data persist long-term**. Volumes are commonly used in the following scenarios.

EBS 的容量是有上限的，AWS 自动给 EBS 做多 AZ 冗余。
费用上来说，EBS 实际上会需要先指定容量，比如 100G，然后按照 100G 付费；可以修改。

- **Operating systems:** Boot/root volumes to store an operating system. The root device for an instance launched from an Amazon Machine Image (AMI) is typically an Amazon EBS volume. These are commonly referred to as EBS-backed AMIs.
- **Databases:** A storage layer for databases running on Amazon EC2 that rely on transactional reads and writes.
- **Enterprise applications:** Amazon EBS provides reliable block storage to run business-critical applications.
- **Throughput-intensive applications:** Applications that perform long, continuous reads and writes.

### EBS volume types
**SSD**, solid state drives，固态硬盘
    strong performance for random input/output (I/O) 随机读取
    Ideal for transactional workloads, such as databases and boot volumes. 启动卷、数据库
**HDD**，hard disk drives
    strong performance for sequential， throughput，I/O 顺序读取
    Ideal for throughput-intensive workloads, such as big data, data warehouses, log processing, and sequential data I/O. 大数据、日志处理

需要关注 IOPS, throughput 之间的取舍。

### EBS Snapshots
incremental backups，增量备份到 S3。

## Object storage
类似 File storage，当作一整个文件存储，但是是 flat structure，并不是 hierachy
**object = file & metadata**，每个 object 都有单独 identifier, using unique identifiers to look up objects when requested.
更新文件内容，实际是完整替换
适合大量数据、unstructured files

### S3
自动在三个 AZ 备份数据，提供灾备恢复能力
    One Zone-Infrequent Access(One-Zone-IA)，低成本存储层级
单个文件最大 5TB
Everything in Amazon S3 is private by default // 默认情况下，所有 objects 是被保护起来的
  - **IAM policy** attached to users, groups, roles
        IAM policies for private buckets 场景：
            有多个 S3 buckets 需要设置不同权限，可以用 IAM，避免对每个桶去设置
            想要把所有 policy 集中起来
  - **Bucket policy** attached to bucket. cannot be used for folders or objects.
        You need a simple way to do cross-account access to S3, without using IAM roles.
        Your IAM policies bump up against the defined size limit. S3 bucket policies have a larger size limit.

object identifier 由几部分构成，bucket_name 参与了，所以 bucket_name 在 global 范围必须唯一
    https://fusenawss3bucket.s3.cn-north-1.amazonaws.com.cn/Downloads/123.pcap
虽然看上去 S3 可以配置/folders，不过实际上 files & folders 是在同一个 level，并不是 actual file hierarchy

### S3 use cases
The following list summarizes some of the most common ways you can use Amazon S3:

- **Backup and storage:** Amazon S3 is a natural place to back up files because it is highly redundant. As mentioned in the last unit, AWS stores your EBS snapshots in S3 to take advantage of its high availability.
- **Media hosting:** Because you can store unlimited objects, and each individual object can be up to 5 TBs, Amazon S3 is an ideal location to host video, photo, and music uploads.
- **Software delivery:** You can use Amazon S3 to host your software applications that customers can download.
- **Data lakes:** Amazon S3 is an optimal foundation for a data lake because of its virtually unlimited scalability. You can increase storage from gigabytes to petabytes of content, paying only for what you use.
- **Static websites:** You can configure your S3 bucket to host a static website of HTML, CSS, and client-side scripts.
- **Static content:** Because of the limitless scaling, the support for large files, and the fact that you access any object over the web at any time, Amazon S3 is the perfect place to store static content.
  
### S3 Versioning
如果 upload 了同名 object，在不开启 versioning 情况下，原始文件就会被覆盖，无法找回
为了避免无意的覆盖，或者想要保持不同 object 版本，推荐使用 versioning
删除的话，其实是添加了 **delete marker**，而并不是真是的 remove。
This preserves old versions of an object without using different names, which helps with file recovery from accidental deletions, accidental overwrites, or  application failures.
开启了 versioning，可能导致存储 size 很大，所以建议是配合 **Lifecycle configuration** 以及**不同存储层级转换**。

When you define a lifecycle policy configuration for an object or group of objects, you can choose to automate two actions – transition and expiration actions.
- **Transition actions** define when objects should transition to another storage class.
- **Expiration actions** define when objects expire and should be permanently deleted.

# AWS Databases
[架构师 blog-选择正确的 DB](https://aws.amazon.com/cn/blogs/architecture/selecting-the-right-database-and-database-migration-plan-for-your-workloads/)
[数据库自由](https://aws.amazon.com/cn/products/databases/freedom/?nc=sn&loc=5)

![Database types](/assets/img/post-aws-db-types.png)  

## 关系型数据库 RDS
db.instance 按小时收费，不管数据库是不是被使用。

RDS 可以开启 multi-AZ deployment 模式，在不同 AZ launch db.ec2.instance 来满足 HA。
  - AWS 来负责 HA 之间的数据备份、failover；通过 DNS 提供服务，DNS TTL = 20 seconds
  - client/application 层面通过 RDS DNS 来访问，需要负责在访问异常时候清理 DNS 缓存，retry
  - primary copy in AZ 提供服务，synchronously replicated to the standby copy/同步复制
  - primary fail，自动 failover；为了保证 multi-AZ，若原有 primary 依然可用，就降级为 standby；若不可用了，就 launch new standby copy

Amazon RDS engines are:
    Commercial: Oracle, SQL Server
    Open Source: MySQL, PostgreSQL, MariaDB
    Cloud Native: Amazon Aurora(MySQL- and PostgreSQL-compatible DB, more durable/available/faster)

RDS 底层实际是 EC2，客户可以选择 EC2 机型，EC2 的 EBS 等。
RDS subnet group 子网组，如果是 public facing，注意 subnet 使用的路由表，0.0.0.0 -- IGW，不能使用 NAT-Gate
  - 要用 SG, ACL 限制 RDS 访问权限
  - 可以用 IAM 限制对 RDS 的操作权限

### RDS 备份
  - 自动备份 entire DB & transaction logs，默认打开，创建 RDS 时候选择 backup window(latency, downtime)
    - backup 保留期 0 - 35 天
    - 0 days setting disables automatic backups, will also delete all existing automated backups
    - Point-in-time recovery 会创建一个新 db.instance，可以 restore full backup 或者根据 transactions 恢复到特定时间
  - 手动 snapshots
    - 比如需要保持 35 days 以上的 backup；类似于 EBS snapshots
  - 推荐是两种方式都使用，automated backups 可以有 point in time recovery，manual snapshots 可以保存超过 35 days

## 非关系型，key-value DynamoDB
按使用情况和 DDB 存储容量收费，并不是 per hour/second
DDB 是 **serveless DB**，用户不需要关系 db.instance 和 HA；fully **managed NoSQL** database service.
高性能、毫秒级延迟，all of your data is stored on solid-state disks (SSDs) and is automatically replicated across multiple AZs, providing built-in high availability and data durability.

### Components 组件
  - table -- items -- attributes
    - table 是数据的集合
    - 一个 item 代表一条数据 key & value，比如张三、电话号、住址、兴趣爱好；table 中的 items 理论上可以无限添加
    - attribute 是 数据/value 的 key，比如电话号、住址
  - primary keys to uniquely identify each item in a table
  - secondary indexes to provide more querying flexibility

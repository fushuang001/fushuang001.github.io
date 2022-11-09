---
layout:         post
title:          AWS 亚马逊云 Storage 和 Database 信息
subtitle:		    备考 SAA 的笔记
date:           2022-04-10
author:         Forest
cover:          '/assets/img/post-aws-storage-db-bg.png'
tags:           AWS, SAA, Storage, EBS, S3, Database, RDS, DynamoDB, RedShift, ElastiCache
---

# AWS Storage 存储
[AWS technical essenticals](https://explore.skillbuilder.aws/learn/course/1851/play/45289/aws-technical-essentials-104)
![示意图](/assets/img/IMG_20220410-113827773.png)  

可以区分为两大类，**File storage** 和 **object storage**  
**File storage**  
- 类比 MacOS Finder，NAS(network attached storage)  
- 文件有目录结构/tree-like hierarchy，  
- 按照目录-子目录-文件来寻址  
- metadata like filename, date, size, type...  
- 更新文件内容，本质上是替换发生变化的一小部分即可    
- 适合多个 hosts 文件共享，sotage mounted onto multiple hosts  

**Object storage**  
- 类似 File storage，当作一整个文件存储，但是是 flat structure，并不是 hierachy  
- **object = file & metadata**，每个 object 都有单独 identifier, using unique identifiers to look up objects when requested.  
- 更新文件内容，实际是完整替换  
- 适合大量数据、unstructured files  

![object_storage](/assets/img/IMG_20220412-114307486.png)  

## instance store - Block storage
- ephemeral block storage  
- This storage is located on disks that are physically attached to the host computer.   
  - **费用是包含在 EC2 里面的，不会额外收取其他费用**  
  - lifetime 会随着 EC2 的 status 发生变化：
    - 重启 EC2，理论上来说 EC2 依然会在原来的 physical host 启动，所以 instance store 数据不会丢失  
    - stop, start EC2，就会在不同 physical host 启动，那么 instance store 数据会丢失  
    - hibernates, terminate EC2，或者 underlying disk drive fails,instance store 数据会丢失  
- 只能在 launch EC2 时候选择特定的 type，比如 c5d.large，其中 d 表示带有 instance store。  
  
![instance_store_volumes](/assets/img/post-instance_store_volumes.png)  

## EBS - Block storage  
- 类比 SAN(storage area network)  
- 将文件分为固定大小的块/splits files into fixed-size chunks  
- 每一块都有自己的地址；按照地址来查询数据  
- metadata 只有 chunk address，所以更快  
- 更新文件内容，本质上是替换 specified chunk，和其他 chunks 无关  
- 适合低延迟、高性能场景  

### EBS use case
[EBS architecture](https://explore.skillbuilder.aws/learn/course/9344/play/31324/planning-and-designing-your-amazon-elastic-block-store-amazon-ebs-architecture)  
Amazon EBS is useful when you must **retrieve data quickly(EBS provides sub-millisecond latency)** and **have data persist long-term**. Volumes are commonly used in the following scenarios.  

- EBS 的容量是有上限的，AWS 自动给 EBS 做多 AZ 冗余。  
- 费用上来说，EBS 实际上会需要先指定容量，比如 100G，然后按照 100G 付费；可以修改。  
- EBS 支持在线扩容，[EBS 扩容以后，必须从 OS 层面扩展 EC2 的文件系统，来适应新的容量](https://aws.amazon.com/premiumsupport/knowledge-center/ebs-volume-increase-os/)  
  - __6 小时内，只能修改一次__  
    - You've reached the maximum modification rate per volume limit. Wait at least 6 hours between modifications per EBS volume.  
  - 其实看起来重启 EC2，也可以让 OS 层面重新加载新的 EBS volume size  
- **Operating systems:** Boot/root volumes to store an operating system. The root device for an instance launched from an Amazon Machine Image (AMI) is typically an Amazon EBS volume. These are commonly referred to as EBS-backed AMIs.  
- **Databases:** A storage layer for databases running on Amazon EC2 that rely on transactional reads and writes.  
- **Enterprise applications:** Amazon EBS provides reliable block storage to run business-critical applications.  
- **Throughput-intensive applications:** Applications that perform long, continuous reads and writes.  

### EBS volume types
**SSD**, solid state drives，固态硬盘  
    strong performance for random input/output (I/O) 随机读取  
    Ideal for __transactional workloads__, such as databases and boot volumes. 启动卷、数据库 // __事务密集型__  
**HDD**，hard disk drives  
    strong performance for sequential， throughput，I/O 顺序读取  
    Ideal for __throughput-intensive workloads__, such as big data, data warehouses, log processing, and sequential data I/O. 大数据、日志处理  

[需要关注 IOPS, throughput 之间的取舍。](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ebs-volume-types.html)  

![SSD](/assets/img/IMG_20220528-182449903.png)  

![HDD](/assets/img/IMG_20220528-182517779.png)  

### EBS Snapshots
incremental backups，增量备份到 S3。  

## EFS for linux, NFS file system
[EFS FAQs](https://aws.amazon.com/efs/faq/)
- serveless 无服务器弹性文件系统  
- EFS uses the Network File System version 4 (NFS v4) protocol.  
- EFS 只支持 Linux EC2，不支持 Windows EC2  
- __strong consistency and file locking__    
- 适用场景包括大数据、分析、内容管理、Web 服务   

[Amazon EFS: How it works](https://docs.aws.amazon.com/efs/latest/ug/how-it-works.html)  
![how-it-works](/assets/img/post-efs-ec2-how-it-works-Regional.png)  

**对比 EBS：**    
- EBS 只能够单次 attach 到单独 EC2  
- EBS 与 EC2 需要在相同 AZ  
- EBS 支持扩容，[扩容以后需要在 OS 层面做操作来识别新的容量](https://aws.amazon.com/premiumsupport/knowledge-center/ebs-volume-increase-os/)  
- EBS 需要提前预置容量 (provision storage in advance.)，按总容量收费  
- EFS 是 Region service，支持 attach to multiple EC2 the same time    
- EFS 容量可以自动扩容，不影响 application  
- EFS, FSx 不需要预置容量，可以按需自动扩展，按用量收费  
- EFS 支持 on-premises server 通过 DX 连接  

![EBS & EFS](/assets/img/IMG_20220417-211334691.png)  

## FSx for Windows, SMB protocol
- Fully managed file server built on Windows Server that supports the SMB/NTFS protocol, Microsoft Active Directory(AD) integration  
- 比如用于 Sharepoint, Microsoft SQL Server, Workspaces, IIS Web Server 或者任何其他 native Microsoft application   
- Migrating Existing Files to Amazon FSx for Windows File Server Using AWS DataSync  

![FSx for Windows](/assets/img/post-FSx-for-Windowns.png)  

[FSx for Windows file server FAQs](https://aws.amazon.com/fsx/windows/faqs/?nc=sn&loc=8)

## FSx for Lustre
- AWS 托管服务，为计算负载提供可靠高效的 **shared storage**    
- 高性能文件系统，可扩展  
- 通过创建 Lustre filesystem 然后关联 S3 bucket，创建 filesystem 时候，Lustre 查看 objects 并且保存一些信息到 metadata  
- 适用的场景，比如 machine learning, high performance computing (HPC), video rendering, EDA  
- [Mounting FSx for Lustre on an Amazon Fargate launch type isn't supported](https://docs.amazonaws.cn/en_us/fsx/latest/LustreGuide/mounting-ecs.html). 有一道类似 SAA-C03 考试题   
![FSx for Lustre](/assets/img/post-FSx-forLustre.png)  

## S3 - Object storage  
- 自动在三个 AZ 备份数据，提供灾备恢复能力；相对于 EBS 来说，S3 属于 Serverless  
- 单个文件最大 5 TB  
- Everything in Amazon S3 is private by default // 默认情况下，所有 objects 是被保护起来的  
  - **IAM policy** attached to users, groups, roles  
        IAM policies for private buckets 场景：  
            有多个 S3 buckets 需要设置不同权限，可以用 IAM，避免对每个桶去设置  
            想要把所有 policy 集中起来  
  - **Bucket policy** attached to bucket. cannot be used for folders or objects.  
        You need a simple way to do cross-account access to S3, without using IAM roles.  
        Your IAM policies bump up against the defined size limit. S3 bucket policies have a larger size limit.  

- object identifier 由几部分构成，bucket_name 参与了，所以 bucket_name 在 global 范围必须唯一，避免混淆  
  - https://myexamplebucket.s3.cn-north-1.amazonaws.com.cn/Downloads/123.pcap  
- 虽然看上去 S3 可以配置/folders，不过实际上 files & folders 是在同一个 level，并不是 actual file hierarchy  

### S3 不同层级的存储类型
不同生命周期不同场景，[为 objects 找到合适的存储层级](https://aws.amazon.com/cn/s3/storage-classes/?nc1=h_ls)，来节省费用  
![S3 Storage Class](/assets/img/IMG_20220504-193742380.png)  

||Standard 标准|Intelligent-Tiering 智能分层|Standard-IA 标准-IA|One Zone-IA 单区-IA|Glacier Instant Retrieval 即时检索|Glacier Flexible Retrieval 灵活检索|Deep Archive 深层归档|
|----|----|----|----|----|----|----|----|
|场景|频繁访问的数据，比如云应用程序、动态网站、内容分配、移动和游戏应用程序以及大数据分析|未知或变化的访问，根据访问频率自动将数据移至最经济实惠的访问层|不频繁访问，毫秒级检索；适合长期存储、备份|同左，单区|很少访问/per 季度，毫秒级检索；长期存储，比 Standard-IA 更经济，如医学图像、新闻媒体资产或用户生成的内容归档|很少访问/per half year，不需要立即访问但需要灵活地免费检索大量数据的归档数据|成本最低，监管严格的行业，如金融服务、医疗保健和公共部门 – 为了满足监管合规要求，将数据集保留 7—10 年或更长时间|
|检索时间，首字节延迟|ms|ms|ms|ms|ms|minutes or hours|within 12 hours|
|每个 object 最低容量费用|NA|NA|128KB|128KB|128KB|40KB|40KB|
|最低存储持续时间费用|NA|NA，但是收取监控和自动化费用|30 天|30 天|90 天|90 天|180 天|
|检索费用|NA|NA|每检索 1GB|每检索 1GB|每检索 1GB|每检索 1GB|每检索 1GB|
|生命周期转换|Y|Y|Y|Y|Y|Y|Y|

>S3 Intelligent-Tiering 可监控访问模式，并将连续 30 天未访问的对象移动到不频繁访问层，并在 90 天未访问之后，移动到归档即时访问层。对于不需要即时检索的数据，您可以设置 S3 Intelligent-Tiering，以监控对象并在 180 天以上未访问后将其则移至深度归档访问层，从而实现高达 95% 的存储成本节省。

### S3 use cases
The following list summarizes some of the most common ways you can use Amazon S3:  
- **Backup and storage:** Amazon S3 is a natural place to back up files because it is highly redundant. As mentioned in the last unit, AWS stores your EBS snapshots in S3 to take advantage of its high availability.  
- **Media hosting:** Because you can store unlimited objects, and each individual object can be up to 5 TBs, Amazon S3 is an ideal location to host video, photo, and music uploads.  
- **Software delivery:** You can use Amazon S3 to host your software applications that customers can download.  
- **Data lakes:** Amazon S3 is an optimal foundation for a data lake because of its virtually unlimited scalability. You can increase storage from gigabytes to petabytes of content, paying only for what you use.  
- **Static websites:** You can configure your S3 bucket to host a static website of HTML, CSS, and client-side scripts.  
- **Static content:** Because of the limitless scaling, the support for large files, and the fact that you access any object over the web at any time, Amazon S3 is the perfect place to store static content.  
  
### S3 Versioning 版本
- 如果 upload 了同名 object，在不开启 versioning 情况下，原始文件就会被覆盖，无法找回  
- 为了避免无意的覆盖，或者想要保持不同 object 版本，推荐使用 versioning   
- 假设 version 1 object 可以公开访问，那么上传一份新的文件，假设是 version 2 是否 public accessable 呢？并不能，version 2 默认是 private 的  
- 若不指定 specify version 版本，直接去删除的话，其实是添加了 **delete marker**，而并不是真是的删除了。  
  - 不过如果是删除 specify version 版本的 object，就是真正的 remove，不能找回。  
- This preserves old versions of an object without using different names, which helps with file recovery from accidental deletions, accidental overwrites, or  application failures.  
- 开启了 versioning，可能导致存储 size 很大，所以建议是配合 **Lifecycle configuration** 以及**不同存储层级转换**。  

When you define a lifecycle policy configuration for an object or group of objects, you can choose to automate two actions – transition and expiration actions.
- **Transition actions** define when objects should transition to another storage class.
- **Expiration actions** define when objects expire and should be permanently deleted.

### S3 Encryption 加密
支持两种方式：传输到 S3 之前 (before-transmit，client-side) 加密，在 S3 存储的 objects 加密 (at-rest，server-side)  

<span style='background:lime;color:black'>SSE(Server Side Encryption)，S3 服务器端加密</span>
- [在 S3 将数据保存到 disk 之前加密数据](https://docs.aws.amazon.com/AmazonS3/latest/userguide/serv-side-encryption.html)，在你下载数据时候解密  
- 数据加密和密钥保管，都可以托管给 AWS    
- 用于加密的 key，取决于用户是否需要控制密钥，区分为三种  
  - SSE-S3  
    - encrypt key 是 S3 托管，不需要客户负责，客户无法干预  
  - SSE-KMS  
    - encrypt key 通过 KMS 管理，客户指定 KMS  
  - SSE-C  
    - encrypt key 是客户管理，encrypt/decrypt 是 AWS 管理  
    - 客户通过 `PutObject` 来上传 object，以及 encrypt key，必须使用 `HTTPS`；  
    - AWS 通过 `AES-256` 加密 object，保存到 disk，然后 AWS 删除 customer encrypt key   
    - 客户下载数据时候，必须提供相同的 encrypt key，由 AWS 完成解密  
    - AWS 并不会存储具体的 encrypt key，而是存储一个 HMAC hash 用于对比   

![SSE](/assets/img/post-S3-SSE.png)  

<span style='background:lime;color:black'>Client Side Encryption，客户端加密</span>
- [在将数据传递到 S3 之前，client 端加密数据](https://docs.aws.amazon.com/AmazonS3/latest/userguide/serv-side-encryption.html)  
- 用于加密的 key，取决于是否托管给 AWS，区分为两种  
  -  KMS  
     -  server-side master key storage，将密钥保存在 KMS  
     -  仅支持同步加密 (symmetric encryption)，不支持异步加密  
  -  Client provided/managed key  
     -  客户自己管理密钥，若密钥丢失，就无法解密了  

### MFA delete 
- 只有 root 可以删除 objects，提高安全性   
- 只能够通过 CLI, SDK 来 enable  
- AWS 中国区没有 root account 所以不支持对应 feature  
```
	 aws s3api put-bucket-versioning --bucket testbucket --versioning-configuration Status=Enabled,MFADelete=Enabled --mfa "SERIAL 959521"
		An error occurred (NotDeviceOwnerError) when calling the PutBucketVersioning operation: The device with serial number SERIAL that generated token 959521 is not owned by the authenticated user
```

### Object Lock 对象锁定
- 创建 S3 桶时候需要打开，创建后，客户无法编辑此选项，可以联系 AWS support  
- 若启用，默认打开 versioning  
- 启用之后，无法关闭  
- store objects using a write-once-read-many (WORM) model to help you prevent objects from being deleted or overwritten for a fixed amount of time or indefinitely  

### Glacier Vault Lock 文件库锁定
- 为了满足合规性要求，比如 objects 在五年内不应该删除  
- write-once-read-many (WORM) model，一次写入，多次读取 
- [文件库锁定](https://docs.aws.amazon.com/amazonglacier/latest/dev/vault-lock.html) 策略在锁定后不能再更改或删除策略   
- 建议先创建文件库，完成文件库锁定策略，然后将档案上传到文件库，以便将该策略应用于它们  

<span style='background:lime;color:black'>Glacier Vault Lock Policies 文件库锁定策略</span>
- 配合 Glacier Vault Lock，实现合规性要求  
- 比如下面的 [Vault Lock Policy](https://docs.aws.amazon.com/amazonglacier/latest/dev/vault-lock-policy.html)，禁止删除 < 365 天的 archives  
```
{
     "Version":"2012-10-17",
     "Statement":[
      {
         "Sid": "deny-based-on-archive-age",
         "Principal": "*",
         "Effect": "Deny",
         "Action": "glacier:DeleteArchive",
         "Resource": [
            "arn:aws:glacier:us-west-2:123456789012:vaults/examplevault"
         ],
         "Condition": {
             "NumericLessThan" : {
                  "glacier:ArchiveAgeInDays" : "365"
             }
         }
      }
   ]
}
```
<span style='background:lime;color:black'>Glacier Vault access Policies 文件库访问策略</span>
- [与合规性要求无关](https://docs.aws.amazon.com/amazonglacier/latest/dev/vault-access-policy.html)，指定谁可以访问对应 vault 的 archives/objects  
- resource-based policy，任何时候都可以修改（对比 Vault Lock Policy，买定离手，无法修改）  
```
{
    "Version":"2012-10-17",
    "Statement":[
       {
          "Sid":"cross-account-upload",
          "Principal": {
             "AWS": [
                "arn:aws:iam::123456789012:root",
                "arn:aws:iam::444455556666:root"
             ]
          },
          "Effect":"Allow",
          "Action": [
             "glacier:UploadArchive",
             "glacier:InitiateMultipartUpload",
             "glacier:AbortMultipartUpload",
             "glacier:CompleteMultipartUpload"
          ],
          "Resource": [
             "arn:aws:glacier:us-west-2:999999999999:vaults/examplevault"                                           
          ]
       }
    ]
}
```

### Event Notification
- 针对 S3 objects, object ACL 的一些操作比如 PUT, GET, DELETE，发送通知  
- 支持 SNS, SQS, Lambda，但是 target 只能是一个   
- 如果多个 teams 需要收到通知 (parallel asynchronous processing)，可以发送通知给 SNS，然后不同 teams 去订阅 SNS topic  

### Replication rules 复制规则

### S3 SELECT
- 支持 SQL 来过滤 S3 objects 的 contents/内容，做筛选  
- 注意费用  

### AWS S3 CLI
- It is a best practice to use aws s3 commands (such as **aws s3 cp**) for **multipart uploads and downloads**.   
- These aws s3 commands automatically perform multipart uploading and downloading based on the file size.   
- To learn more about using the AWS CLI to perform multipart uploads, see: [How do I use the AWS CLI to perform a multipart upload of a file to Amazon S3? ](https://aws.amazon.com/premiumsupport/knowledge-center/s3-multipart-upload-cli)  
![Multipart upload](/assets/img/IMG_20220519-151902183.png)  

### S3 bucket policy

### Pre-signed URL
- all objects private by default, the object owner can optionally share objects with others by pre-signed URL, using their own security credentials, to grant time-limited permission to download the object  

### pricing, S3 vs EFS
S3 不同存储层级的价格不同，[不过 S3 不只是有存储费用](https://dzone.com/articles/confused-by-aws-storage-options-s3-ebs-amp-efs-explained)，还有其他费用，比如 API 调用、数据传出 S3(Data Transfer Out)  
EFS 整体定价更便宜一些  

# 数据传输服务，混合云存储
AWS 有多个服务可以将本地数据迁移到云端，何时应该选择哪一种服务？  

||S3 Storage Gateway|DataSync|aws s3 sync|Snowball|S3 Batch Operation 批操作|S3 Replication
|----|----|----|----|----|----|----|
|功能|配合 DataSync，SGW file GW 提供对已迁移数据的低延迟访问；File GW 提供 SMB, NFS-based access to data in S3 with local caching|端到端安全的数据转移、发现；初始数据传输|自动 multiple parts upload|TB, PB 级数据传输|copy objects, set object tags or ACLs, initiate object restores from Glacier, invoke Lambda function|new objects 复制|
|场景|DataSync 迁移数据，然后使用 SGW 的 file GW 配置来保留对已迁移数据的访问权限，从本地基于文件的应用程序进行持续更新|在线传输数据，本地和云端共存|日常使用|离线传输，比如带宽受限|针对大量 objects 操作，可以通过 S3 inventory report 来针对性列出需要操作的 objects|将 src S3 持续复制到 dst S3|

## S3 Storage Gateway
- **混合云**存储服务，打通客户本地 (__低延迟__) 和 AWS 云上 (__容量几乎无上限__) 存储
- [基本介绍](https://www.amazonaws.cn/storagegateway/)  

### SGW 分类
File Gateway  
- 直接存储到 S3  
- 支持 SMB, NFS-based 文件系统  

FSx File Gateway  
- store and retrieve files in FSx for Windows File Server  
- SMB protocol  
- 通过 FSx File Gateway 写入的数据，可以直接被 FSx for Windows File Server 读取  

Volume Gateway  
- Stored Volumes  
  - 所有数据保存在本地，异步备份到 S3  
  - Entire dataset is stored on site and is asynchronously backed up to S3  
- Cached Volumes  
  - 所有数据保存在 S3，本地有经常访问数据的缓存  
  - Entire dataset is stored on S3 and the most frequently accessed data is cached on site.  
  - 节省本地存储空间  
  
Gateway Virtual Tape Library 磁带网关  

## DataSync
- end-to-end/端到端将本地 NFS, SMB, HDFS 数据迁移到 AWS S3, FSx, EFS；支持指定 sub-folder 增量移动数据  
- 适用于初次将本地所有数据迁移到 cloud，后续可以使用 SGW 保持本地、cloud 混合存储、同步   
- 也可以在 AWS services 之间传输数据      

# AWS Databases 数据库
[架构师 blog-选择正确的 DB](https://aws.amazon.com/cn/blogs/architecture/selecting-the-right-database-and-database-migration-plan-for-your-workloads/)
[数据库自由](https://aws.amazon.com/cn/products/databases/freedom/?nc=sn&loc=5)

![Database types](/assets/img/IMG_20220408-151714840.png)  

## 关系型数据库 RDS
- db.instance 按小时收费，不管数据库是不是被使用。Reserved instance 付费方式，同样可以用于 db.instance，更加省钱  
- db.instance 实际是 AWS managed EC2，也会需要升级、打补丁等，但是完全交给 AWS 来维护了  
- RDS 适合比较复杂的场景，数据之间存在关联关系。   
- 可以简单类比 excel 表格，每个数据都有相同的列 (name, age, address)    
- RDS 可以开启 multi-AZ deployment 模式，在不同 AZ launch db.ec2.instance 来满足 HA。  
  - AWS 来负责 HA 之间的数据备份、failover；通过 DNS 提供服务，DNS TTL = 20 seconds  
  - client/application 层面通过 RDS DNS 来访问，需要负责在访问异常时候清理 DNS 缓存，retry  
  - primary copy in AZ 提供服务，synchronously replicated to the standby copy/同步复制  
  - primary fail，自动 failover；为了保证 multi-AZ，若原有 primary 依然可用，就降级为 standby；若不可用了，就 launch new standby copy  
- RDS 底层实际是 EC2(RDS is NOT Serverless, RDS runs on Virtual Machines)，客户可以选择 EC2 机型，EC2 的 EBS 等；
- RDS 的 db.instance 是 AWS 托管，客户没有权限登录，或者说安装更新包；      
- Amazon Aurora is Serveless  
- RDS subnet group 子网组，如果是 public facing，注意 subnet 使用的路由表，0.0.0.0 -- IGW，不能使用 NAT-Gate  
  - 要用 SG, ACL 限制 RDS 访问权限  
  - 可以用 IAM 限制对 RDS 的操作权限  
  
**Amazon RDS engines are:**
  - Commercial: Oracle, SQL Server  
  - Open Source: MySQL, PostgreSQL, MariaDB  
  - Cloud Native: Amazon Aurora(MySQL- and PostgreSQL-compatible DB, more durable/available/faster)  

**简单的 demo：**  
client -- EC2/WordPress 前端 --- db.instance/RDS 后端数据库，[可以参考](https://www.bilibili.com/video/BV1fV41127vz?p=58&t=7.9)  

### RDS 备份
> 分为两类，automated backups & datebase snapshots  
> 推荐是两种方式都使用，automated backups 可以有 point in time recovery，manual snapshots 可以保存超过 35 days  

  - __自动备份__ 
    - entire DB & transaction logs，默认打开，创建 RDS 时候选择 backup window（备份期间会有 latency, downtime)    
      - backup data is stored in S3 and you get free storage space equal to the size of your db.  
    - backup retention period/保留期 0 - 35 天  
      - 0 days setting disables automatic backups, will also delete all existing automated backups  
    - Point-in-time recovery 会创建一个新 db.instance，可以 restore full backup 或者根据 transactions 恢复到特定时间（within retention period） 
  
  - __手动 snapshots__  
    - 比如需要保持 35 days 以上的 backup；类似于 EBS snapshots  
    - snapshots are stored even after you deleted the original RDS instance, unlike automated backups  

### RDS 恢复 Restoring Backups
从自动备份或者手动 snapshots 恢复的，是一个新的 RDS db.instance，有一个新的 DNS endpoint/DNS name    

### Multi-AZ, Standby Replica
- have an exact copy of your production database in another AZ, AWS 托管的 <span style='background:lime;color:black'>synchronized replication</span>   
- 作用主要是 Disaster Recovery/HA/failover，并不是提升性能  
  - automatic failover 只会在 primary database 出问题时候才会发生，比如
    - Loss of availability in primary AZ  
    - storage failure on primary  
- 如果发生切换，AWS 会把原来 primary DB instance 的 DNS endpoint 解析 (A, IP 记录）替换为 <span style='background:lime;color:black'>standby replica</span>，不需要客户手动干预；对 application 来说，仍然是访问之前的 DNS endpoint  
- 用户可以自己在 AWS console 控制台，手动 failover from one AZ to another by rebooting the RDS instance  
- [Multi-AZ 部署有两种方式](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Concepts.MultiAZ.html)，区别在于 one standby or two standby DB instances  
  - one standby DB instance(standby repliac), 叫做 __Multi-AZ DB instance deployment__  
    - 支持 failover 但是 standby repliac 并不支持 DB read  
  - two standby DB instance(standby repliac)，叫做 __Multi-AZ DB cluster deployment__  
    - 支持 failover & DB read traffic  
- Aurora 是 AWS 托管，不需要客户配置 Multi-AZ；客户可以为其他 RDS DB 配置 Multi-AZ  

Multi-AZ DB instance deployment  
![Multi-AZ DB instance deployment](/assets/img/con-multi-AZ.png)  

Multi-AZ DB cluster deployment  
![Multi-AZ DB cluster deployment](/assets/img/multi-az-db-cluster.png)  

### Read Replica 只读副本
- 如果有一些程序需要经常读取数据，可以把读取的任务放到 Read Replica 去执行  
- Read Replica 有自己单独的 DNS endpoint  
- Read Replica 可以提升/promot 为主库/master，会打破 read replica 关系  
- Read Replica 用于提高性能：  
  - 假如说 db.instance 的负载过高，而且有大量 read 操作，与其扩容 db.instance，不如使用 Read Replica 更经济实惠。  
  - 另一个可以提升性能的方式，是考虑 ElastiCache  
  - 指定 source DB，然后 RDS 会创建 snapshot of the source DB，创建 read-only instance from the snapshot  
  - Asynchronous replication，是用于提升性能，并不是 Disaster Recovery  
  - To further maximize read performance, Amazon RDS for MySQL allows you to add table indexes directly to Read Replicas, without those indexes being present on the master.  
- 需要打开 automatic backups 才能使用 read replica  
![Read_Replica](/assets/img/IMG_20220420-131050551.png)  

### when to use EC2 自建数据库
如果客户需要底层硬件的控制权，或者使用一些 RDS 不支持的 feature，可以考虑在 EC2 自建数据库  
[EC2 for Oracle, When to choose Amazon EC2](https://docs.aws.amazon.com/prescriptive-guidance/latest/migration-oracle-database/ec2-oracle.html)

### Storage AutoScaling
用户设置 maximum storage size，然后 RDS 根据实际需求自动扩容  

### RDS Enhanced Monitoring
- RDS 底层硬件是 AWS managed EC2，AWS 默认提供对应 EC2 的 CloudWatch 监控，数据来源是 the hypervisor for a DB instancce  
- Enhanced Monitoring 从 EC2 instance 里面一个预先安装的 agent 收集指标信息，以 JSON 格式发送到 CloudWatch Logs，数据更准确    
- 如果希望 [了解不同进程或线程对 CPU 的使用差异](https://docs.aws.amazon.com/zh_cn/AmazonRDS/latest/UserGuide/USER_Monitoring.OS.html)，增强监测指标非常有用  

### RDS event notification
- RDS events provide operational events such as SB instance events, DB parameter group events, DB security group events, and snapshot events  
- not for RDS data event(inject, remove). Could invoke Lambda function from Amazon Aurora MySQL-Compatible Edition DB cluster with a __native function or stored procedure__. 举个栗子，如果你希望在 DB 数据发生变化的时候，联动其他 AWS service，比如用 Lambda 发送消息到 SQS  

### RDS Proxy
- [fully managed, HA database proxy for RDS](https://aws.amazon.com/rds/proxy/)  
- 当很多的 clients 尝试和 DB 建立连接，connection 频繁 open, close 会消耗 DB 的 memory、compute resource，可能会看到 DB 回复"too many connection, retry later"类似问题。可以通过 RDS Proxy 来解决这类问题  
- applications -- RDS Proxy -- database，RDS Proxy allows applications to pool and share connections established with the db，提高 DB 效率，降低 DB failover 概率  
![how it works](/assets/img/product-page-diagram_RDS Proxy_How-it-works@2x.a18916586f49718a16fd11579d168ab08c83d333.png)

### IAM database authenication
- 不 [使用 DB password 连接，而是借助 authentication token](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/UsingWithRDS.IAMDBAuth.html)(generated using AWS Signature V4)，由 IAM 存储 token，每个 token 有效期 15 分钟  
- 支持 MariaDB, MySQL, and PostgreSQL  
- 优点是不需要单独管理 DB password，可以通过 IAM 集中管理  
- For applications running on Amazon EC2, you can use profile credentials specific to your EC2 instance to access your database instead of a password, for greater security.  

## Aurora
- MySQL, PostgreSQL-compatible RDS, open source  
- capacity type 分为两类  
  - __Aurora Serverless__   
    - simple, cost-effective option for infrequent, intermittent, or unpredictable workloads  
    - 通过指定 minimum、maximum capacity，而不是 instance type，由 Aurora 根据 workload 来扩缩容  
    - 适合 intermittent, unpredictable workload 场景  
    - db endpoint --- proxy fleet --- fleet of resources，resources/db 扩缩容；warm resoures，快速扩展    
    - 存储和 processing 费用分开的，没有 processing 的话，只收取存储的费用  
  - __provisioned__  
    - customer  manage the server instance types  
    - non-serverless DB cluster，用户指定 DB instance type  
    - 通过调整 replica 数量来提高 read throughput  
    - 适合 predictable/可预测的 DB workload 场景   
- Automated backups 是默认且必须 enabled，Backups do not impact database performance  

![Writer, Reader 有各自 DNS Endpoint](/assets/img/IMG_20220609-153921417.png)  

![Aurora Replica](/assets/img/IMG_20220609-145209607.png)  

### Aurora endpoints
目前 [四种类型 endpoints 可用](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.Overview.Endpoints.html)，了解对应的功能和场景先  
Cluster endpoint
- connects to the current primary DB instance for that DB cluster  
  - primary instance for DDL(Data Definition Language), DML(Data Manipulation Language)  
- the only one that can perform write operations  
- provides failover support for read/write connections to the DB cluster  

Reader endpoint
- provides __load-balancing__ support for __read-only__ connections to the Aurora DB cluster  
- reduces the overhead on the primary instance  
- each Aurora DB cluster has one reader endpoint  
- if Aurora DB cluster contains one or more Aurora Replicas, the reader endpoint load-balances each connection request among the Aurora Replicas.  

Custom endpoint
- you choose a set of DB instances put into custom endpoint  
- load-balancing and chooses one of the instances in the group  
- you define which instances, what purpose the endpoint serves
- LB 自定义程度比较高，不只是 read-only or read/write capability，比如 mission-critical 的业务使用 high-capability db, 一般的查询业务使用 low-capability db；或者将流量引入特定 parameter group 的 db instance        

Instance endpoint
- connect to a specify DB instance within an Aurora cluster  
- each DB instance(primary, Replicas) has its own unique instance endpoint  
- direct control over connections the the DB cluster, for scenarios where using cluster/reader endpoint might not be appropriate.  

### Aurora Global Database
- designed for globally distibuted applications, single Aurora db to span multiple AWS regions.  
- replicates data with no impact on db performance, enables fast local reads with low latency in each region  
- support storage-based replication that has a latency of less than 1 second(RPO, Recovery Point Objective 1s)  
  - RPO 1s 表示能恢复到上一秒的状态    
- Cross-Region Disater Recovery, if there is an unplanned outage, one of the secondary region you assigned can be promoted to read and write capabilities in less than 1 minute(RTO, Recovery Time Objective 1 min)  
  - RTO 1min 表示能在 1 分钟内完成跨 region 故障切换  

## 非关系型，key-value DynamoDB
- 按使用情况和 DDB 存储容量收费 (read/write 不收费），并不是 per hour/second  
- 可以类比 JSON，其中还可以有 key:value pairs，每个记录都可以有只属于它的 key    
- DDB 是 **serveless DB**，用户不需要关系 db.instance 和 HA；fully **managed NoSQL** database service.  
- 高性能、毫秒级延迟，all of your data is __stored on solid-state disks (SSDs)__ and is automatically replicated across multiple AZs, providing built-in high availability and data durability.  
- ElastiCache 和 DDB 都比较适合用来存储 HA 的 web application 的 __session data__，高速读取   
- 默认是 [Eventually consistent](https://aws.amazon.com/dynamodb/faqs/)（写入，1 秒后所有数据一致）；另外还支持 Strongly consistent  
![JSON/NoSQL](/assets/img/IMG_20220528-202315635.png)  

### Components 组件
  - table -- items -- attributes  
    - table 是数据的集合  
    - 一个 item 代表一条数据 key & value，比如张三、电话号、住址、兴趣爱好；table 中的 items 理论上可以无限添加  
    - attribute 是 数据/value 的 key，比如电话号、住址  
  - primary keys to uniquely identify each item in a table  
  - partition key portion of a table's primary key determines the logical partition in which a table's data is stored.  will impact how balance distribut I/O reqeust  
  - secondary indexes to provide more querying flexibility  

### Backup and Restore
> Point-in-Time Recovery(PITR)
- protects against accidental writes or deletes  
- restore to any point in the last 35 days  
- incremental backups/增量备份  
- not enabled by default  
- latest restorable: __5 minutes__ in the past

### DAX(Cluster DynamoDB Accelerator) 加速器
- 类似于 RDS Read Replica，数据高可用，降低延迟，可以达到微秒级/microseconds 响应  
- AWS 托管，in-memory cache，支持 read, write  
- DAX Cluster 会提供一个 Endpoint 给 client 使用，隐藏后面 scaling 的细节  
![DAX_Cluster](/assets/img/IMG_20220420-131351110.png)  

### DDB Stream
- DDB Stream is an ordered flow of info about changes to items in an DDB table  
- 如果打开 DDB Stream，就可以获取到 table 中的修改 (create, update, delete)  
- DDB Stream 会记录发生变化 table item 的 primary key attributes  
- DDB 与 Lambda 集成，将 DDB Stream 也就是 table 变化信息推送给 Lambda 做处理，比如你关注的人发表了新的动态，DDB Stream --- Lambda --- SNS topic, email update  

# AWS Data Warehouse 数据仓库
本质上还是数据库，只不过从 architecture，infrastructure 与和 RDS, DynamoDB 都不一样  

## OLTP vs OLAP
- RDS belongs to OLTP(Transaction)  
- OLAP is much more complex, AWS Redshift is OLAP(Analytics)  
![OLTP](/assets/img/IMG_20220528-202456159.png)  
![OLAP](/assets/img/IMG_20220528-202520719.png)  

## Redshift -- cloud data warehouse
- used for business intelligence(OLAP), use SWL to analyze   
- data stored in a [columnar fashion](https://docs.aws.amazon.com/redshift/latest/dg/c_columnar_storage_disk_mem_mgmnt.html)，列式存储，用来提升 query performance，原理是降低 overall disk I/O, reduce the amount of data you need to load from disk  
- Amazon Redshift 是一种完全托管的企业 PB 级数据仓库服务  
- fast and powerful, fully managed, petabyte-scale data warehouse service  
- [Massively Parallel Processing(MPP)](https://docs.aws.amazon.com/zh_cn/redshift/latest/dg/c_challenges_achieving_high_performance_queries.html)/大规模并行处理，Redshift automatically distributes data and query load across all nodes  
- available in only 1 AZ  

### Redshift cfg
- Single Node(160Gb)  
- Multi-Node  
  - Leader Node(manages client connections and receives queries) 不收费  
  - Compute Node(store data and perform queries and computations). 最多支持 128 Compute Nodes 收费  

### Backups
- 默认打开，保留期 1 天，最长保留期 35 天  
- 至少三份 copies of your data(original, replica on compute node, backup in S3)  
- 支持 asynchronously 复制 snapshots 到另一个 region S3，做 DR  

### Redshift Spectrum
- enables you to query and analyze all of your data in S3 using the open data formats you already use, with no data loading or transformations needed  
- [Redshift Spectrum](https://docs.aws.amazon.com/redshift/latest/dg/c-using-spectrum.html) queries employ massive parallelism to run very fast against large datasets.  
- Much of the processing occurs in the Redshift Spectrum layer, and most of the data remains in Amazon S3  

# AWS ElastiCache 云缓存
- ElastiCache is a web service that makes it easy to deploy, operate and scale an in-memory cache in the cloud.  
- it improves the performance of web applications by allowing you to retrive info from fast, managed, in-memory caches, instead of relaying entirely on slower disk-based database  
- ElastiCache 和 DDB 都比较适合用来存储 HA 的 web application 的 __session data__，高速读取    
- ElastiCache to speed up performance of existing databases(frequent identical queries)   
  - 比如网站的 top 50 浏览量，基本上变化不大，那么就没必要把这些数据存储到 disk-based database，可以直接从 __in-memory cache__ 读取  
  - 用于提高现有 DB 的性能（经常访问的相同的内容）    
  - 和 RDS Read Replica 类似，比如说，都可以用来 increase db and web application performance  
  - 但是 ElastiCache 是 nonrelelational database    
- 支持两款 open-source in-memory caching engines:  
  - **Memcached**  
    - if you need a simple solution, to scale horizontally  
  - **Redis**  
    - Redis is Multi-AZ
    - you can do backups and restores of Redis  
    - Redis Cluster supports up to 15 shards and single cluster supports to run workloads up to 6.1 TB of in-memory capacity  

# Lake Formation 数据湖
在数天内建立安全的 [数据湖](https://aws.amazon.com/lake-formation/?nc1=h_ls)，从 S3, DDB, RDS 当中获得源数据，放到目标 S3 桶用于分析  
![how it works](/assets/img/Lake-formation-HIW.9ea3fab3b2ac697a42ae7a805b986278ffd4f41e.png)
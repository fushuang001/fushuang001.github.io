---
layout:         post
title:          AWS 消息队列，解耦
subtitle:		    备考 SAA 的笔记
date:           2022-11-13
author:         Luke
cover:          '/assets/img/bg-messaging-queuing.png'
tags:           AWS, SAA, SQS, SNS, Amazon MQ, queuing, decouple
---
- [queuing 的原因](#queuing-的原因)
- [SQS -  Queue](#sqs----queue)
  - [retention period](#retention-period)
  - [visibility timeout](#visibility-timeout)
  - [long polling](#long-polling)
  - [SQS offers two types of message queues.](#sqs-offers-two-types-of-message-queues)
  - [payload](#payload)
- [SNS - Notifications](#sns---notifications)
- [Amazon MQ](#amazon-mq)

# queuing 的原因
架构来源是将不同组件之间实现松耦合 loosely coupled，比如点餐/收银 & 后厨，要确保把消息送达。若后厨无响应，就会拖慢收银。  
通过中间的 messaging/queueing 来提供额外 __buffer__，避免后厨繁忙时候收银必须等待确认，无法为下一个客户服务；也避免后厨无应答时单点故障。  
queuing 的一个重要概念是解耦 **decouple**  
![SQS-SAA-example](/assets/img/post-SQS-SAA-example.png)  

# SQS -  Queue
fully managed message queuing service that enables you to __decouple and scale__ microservices, __distributed__ systems, and serverless applications  
send msg, store msg, receive ms, between software components, without losing messages or requiring other services to be available, at any scale  
__pull-based__, not pushed-based.  

## retention period
- messages can be kept in the queue from 1 minute to 14 days; default __retention period__ is 4 days. 
- retention period 到期，message 将会删除；或者 consumer 处理完成并且主动告知 SQS 删除 message     
- 换句话说，visibility timeout 到期之前，message 只是不可见，并没有被删除  
- 如果 consumer application crash/down，重启之后还是可以从 SQS 获取之前的 msg，__durably stored__  
- messages are __inflight__ after they have been received from the queue by a consuming component, but have not yet been deleted from the queue. 正在处理并且还未被删除的 message，处于 inflight 状态  
- SQS 本身不限制 messages 数量，不过 standand queue 120,000 inflight, FIFO 20,000 inflight  

![SQS-durably-store-msg](/assets/img/post-SQS-durably-store-msg.png)  

## visibility timeout
- __visibility timeout__ is the amount of time that the msg is __invisible__ in the SQS queue after a reader picks up that message. 
- if the job is not processed within visibility timeout interval, the message will become visible again and another reader will process it. this could result in the same msg being delivered twice. (application process time is longer than the invisible time), could increase the visibility timeout interval(maximum 12 hours, default 30s)  
- 如果发现有的 msg 被处理两次，可以考虑 Use the ChangeMessageVisibility API call to increase the visibility timeout.  

## long polling
- __long polling__ is a way to save cost  
- regular short polling(WaitTimeSeconds by default 0s) returns immediately even if the msg queue being polled is empty  
- long polling(WaitTimeSeconds, eg. 20s) doesn't return a response until a msg arrives in the msg queue, or the long poll times out.  

## SQS offers two types of message queues.   
**Standard queues**
- maximum throughput, nearly-unlimited number of transactions per second(TPS)  
- best-effort ordering, and __at-least-once__ delivery  
- due to highly-distributed architecture that allows high throughput, more than one copy of a message might be dlivered out of order.   

---

**SQS FIFO queues** 
- limited to 300 transactions per second(TPS)  
- designed to guarantee that messages are processed __exactly once__, in the __exact order__ that they are sent.  

A company has an ecommerce checkout workflow that writes an order to a database and calls a service to process the payment. Users are experiencing timeouts during the checkout process. When users resubmit the checkout form, multiple unique orders are created for the same desired transaction.

<details>
    <summary>How should a solutions architect refactor this workflow to prevent the creation of multiple orders?</summary>

Store the order in the database. Send a message that includes the order number to an Amazon SQS FIFO queue. Set the payment service to retrieve the message and process the order. Delete the message from the queue.
</details>

## payload
data contains in a msg, 比如点餐餐桌号，菜品，下单时间  
protected until delivery  

![SQS](assets/Messaging%20and%20Queuing/IMG_20220416-175550611.png)  

# SNS - Notifications
Amazon SNS is a managed messaging service that lets you **decouple** publishers from subscribers. This is useful for **application-to-application** messaging for microservices, distributed systems, and serverless applications

可以给 AWS Service 以及用户发送通知，__pushed-based__  
publish/subscribe 或者说 producers/consumers 模式，简称 **pub/sub**，用户订阅 topic，收到特定 event 通知。  
__SNS 并不是 queue，所以并不保证送达__；即使送达，比如接收者不查看，接收的 EC2/application crash/down，也并不会有什么用。  
对比 SQS，__SQS 可以保证消息 durably stored__，即使 application crash，app 重启后，还是可以从 SQS 读取之前的消息；要么 retention period 到期删除，要么 application 处理完成主动删除，要么正在处理，处于 visiblity timeout 不可见   

# Amazon MQ
- [managed msg broker service](https://aws.amazon.com/amazon-mq/), support industary-standard APIs and protocols, including JMS, NMS, AMQP, STOMP, MQTT, Websocket  
- 适用于将 on-prem 已经有的 messaging application 向 cloud 迁移，迁移过程中不希望改动代码  
- Amazon MQ 是适用于 Apache ActiveMQ 和 RabbitMQ 的托管消息代理服务  
- 如果是在 cloud 新建 messaging 服务，可以考虑 SNS, SQS, to decouple and scale microservices, distributed systems and serverless applications    
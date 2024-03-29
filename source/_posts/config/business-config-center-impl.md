---
title: 业务配置中心的实现
date: 2021-05-13 15:44:23
tags:
  - config
---

# 前言

之前在InfoQ的《华为云物联网四年配置中心实践》文章中分享了业务配置中心。

本文讲述业务配置中心（下文简述为配置中心）的关键技术和实现方式。华为云物联网平台按照本文的实现方式实现了一个业务配置中心，该配置中心2020年1月上线，平稳运行至今。

# 概念

## 运维配置

和用户无关，通常为集群界级别的配置，程序只会进行读取，如数据库配置、邮箱服务器配置、网卡配置、子网地址配置等。

## 业务配置

作为SaaS 服务，每个用户在上面都有一些业务配置。如用户的证书配置、用户服务器的流控配置等，这些业务配置相对**运维配置**来说更加复杂，且可能会有唯一性限制，如按用户 id 唯一。这部分配置数据一般由用户操作触发，代码动态写入，并且通知到各个微服务实例。通常，我们希望这些配置能在界面展示，且支持人为修改。上述逻辑如果由各微服务自己实现，会存在大量重复代码，并且质量无法保证。我们希望由一个公共组件来统一实现这个能力。开源或体量较小的项目就不会选择依赖一个配置中心，而是直接通过连接数据库或etcd来解决问题

## env

代表一个部署环境。

## cluster

代表环境下的集群。常见于单环境下蓝绿发布，蓝集群、绿集群、金丝雀集群等。

## 配置

配置名称，如用户证书配置、用户流控配置等。

## Key

配置的唯一键，如用户id。

## Value

配置唯一键对应的值。

# 配置中心设计梗概

## 业务配置特点

- 虽然业务配置写入可能存在并发，但并发量不大，频率较低。
- 业务配置常常以用户为id，单集群用户量有限，一般不超过5万。

## 配置中心要解决的问题

![business-config-center-impl1](business-config-center-impl1.png)

## 设计要点

- 单配置要求有配置id，每个id上通过version的乐观并发控制来解决多版本冲突问题
- 通知不追求可靠，应用程序和配置中心断链无法接收通知的场景下，通过定期同步数据来保证数据的可靠
- 支持Schema的变更，因Schema变更不频繁，也采用version的乐观并发控制来解决多版本冲突问题

## 通知是否包含消息内容
我认为应该只通知Key，具体的数值让应用程序再去配置中心查询。仅通知Key实现简洁易懂。同时通知Key&Value需要多考虑定期同步和通知两条通道并发，可能引起的竞态冲突。

# 配置中心业务流程

本小节描述业务配置中心的所有业务流程，并试图从交互中抽象出与具体实现无关的接口

## 配置的增删改查

![business-config-center-impl2](business-config-center-impl2.png)

## 配置值的增删改查

![business-config-center-impl3](business-config-center-impl3.png)

## 定期同步

分布式场景下，通知有可能无法送达，如程序陷入网络中断（或长gc），通知消息送达超时，待程序恢复后，数据不再准确。因此需要对数据做定期同步，提高可靠性。

![business-config-center-impl4](business-config-center-impl4.png)

同步过程中，仅仅请求交互id和version，避免传输大量数据。应用程序接收到需要同步的数据后：

- 删除操作，触发删除通知，从本地缓存中移除数据。
- 添加、修改操作，向配置中心查询最新数据，触发通知并写入本地缓存。

## 服务启动

服务启动也可看做是一个同步的流程，只是需要同步大量的数据添加。为了避免向配置中心频繁大量的请求，引入批量操作来减轻压力

![business-config-center-impl5](business-config-center-impl5.png)

## 限制

该配置中心设计思路依赖客户端可把数据全量放入到内存中，如用户量太大，则不适合采用这种模式。

注：一个节省内存的思路是，内存中只放置全量的id和version，数据只有当用到的时候再去查询。这个思路要求配置中心持久化一些老旧数据以供以下场景的查询使用
-  业务流程中，需要使用该配置值的。

- 回调业务程序修改的时候，需要提供旧值的。


除此之外没有任何区别。

# 业务配置抽象实现

从上述描述的业务场景，我们抽象出业务配置中心的交互接口和抽象实现。接口的Swagger Yaml已上传到Github：https://gist.github.com/Shoothzj/68c9c2ecae72cc2a125184e95b0a741e

## 配置相关接口

- 提供env、cluster、配置名称、配置Schema、配置版本号添加配置
- 提供env、cluster、配置名称删除配置
- 提供env、cluster、配置名称、新Schema、新Version来修改配置
- 提供env、cluster、配置名称来查询配置

## 配置值相关接口
- 提供env、cluster、配置名称、Key、Value来添加配置值
- 提供env、cluster、Key、ValueVersion（可选）来删除配置值
- 提供env、cluster、Key、Value、ValueVersion（可选）修改配置值
- 提供env、cluster、Key查询配置值
- 根据env、cluster、应用程序当前的配置数据来做定期同步
- 根据Key列表批量查询配置值

## 通知相关接口

- 通知某env某cluster下，配置项中的一个Key发生变化，新增、修改或是删除。可选方式有HTTP长链接（Inspired by Apollo）、Mqtt、WebSocket等。

## 配置中心存储层抽象实现

配置中心存储层需要存储**配置**和**配置值**数据，支持UpdateByVersion，且需要捕捉数据的变化，用来通知到应用程序

## 服务发现抽象实现

为了使应用程序连接到配置中心，需要一个发现机制可以让应用程序感知到配置中心的地址。高可用的方式很多，如K8s发现、ZooKeeper、Etcd、ServiceComb、业务环境变量注入ELB地址（ELB后端挂载配置中心的地址）等。

## 抽象总结

![business-config-center-impl6](business-config-center-impl6.png)

根据这个抽象，我们可以进行关键技术点选型，来实现业务配置中心。

# 配置中心实现

## 华为云物联网配置中心实现

![business-config-center-impl7](business-config-center-impl7.png)

- env+cluster+config组成数据表的名称
- 一个key、value对应一行数据

## 另一种实现方式

只要实现上述接口和抽象能力，都可以实现业务配置中心，也可以这么实现

![business-config-center-impl8](business-config-center-impl8.png)

- env+cluster+config+key 组合成etcd的key
- 一个key、value对应一个键值对

## 又一种实现方式
当然也可以

![business-config-center-impl9](business-config-center-impl9.png)

- env+cluster+config+key 组合成RocksDB的key
- 一个key、value对应一个键值对

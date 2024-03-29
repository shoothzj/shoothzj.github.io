---
title: 异步网络请求编码
date: 2023-12-10 08:32:54
tags:
  - Code
---

本文介绍常见的异步网络请求编码手法。尽管像golang这些的语言，支持协程，可以使得Programmer以同步的方式编写代码，大大降低编码者的心智负担。但网络编程中，批量又非常常见，这就导致即使在Golang中，也不得不进行协程的切换来满足批量的诉求，在Golang中往往对外以callback的方式暴露接口。

无论是callback、还是返回future、还是返回Mono/Flux，亦或是从channel中读取，这是不同的异步编程范式，编码的时候，可以从项目整体、团队编码风格、个人喜好来依次考虑。本文将以callback为主，但移植到其他异步编码范式，并不困难。

使用callback模式后，对外的方法签名类似:

go
```go
func (c *Client) Get(ctx context.Context, req *Request, callback func(resp *Response, err error)) error
```

java
```java
public interface Client {
    void get(Request req, Callback callback);
}
```

## 网络编程中的批量

对于网络请求来说，批量可以提高性能。 批量处理是指将多个请求或任务组合在一起，作为单一的工作单元进行处理。批量尽量对用户透明，用户只需要简单地对批量进行配置，而不需要关心批量的实现细节。

常见的批量相关配置
- batch interval: 批量的时间间隔，比如每隔1s，批量一次
- batch size: 批量的最大大小，比如每次最多批量100个请求

批量可以通过定时任务实现，也可以做一些优化，比如队列中无请求时，暂停定时任务，有请求时，启动定时任务。

## 编码细节

整体流程大概如下图所示：

![async-network-code](async-network-code.png)

### 一定要先把请求放到队列/map中

避免网络请求响应过快，导致callback还没注册上，就已经收到响应了。

### 队列中的消息一定要有超时机制

避免由于丢包等原因，导致请求一直没有响应，而导致队列中的请求越来越多，最终内存溢出。

### wait队列生命周期与底层网络client生命周期一致

wait队列中请求一定是依附于client的，一旦client重建，队列也需要重建，并触发callback、future的失败回调。

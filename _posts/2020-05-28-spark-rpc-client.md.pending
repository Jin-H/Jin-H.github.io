---
layout: post
title:      spark rpc client
subtitle:   初探 spark rpc
date:       2020-05-28
author:     redscarf
header-img:
catalog:    true
tags:
    - spark rpc
---

#### 背景
> 如无特别声明，本篇及后续博客涉及的 spark 版本为 2.11 版本。

    最近对 spark 充满兴趣，于是开始了各个模块的研究，首当其冲的就是 rpc 模块了。Spark 在 2.0.0 版本中[移除了 Akka](https://issues.apache.org/jira/browse/SPARK-5293)，
并使用 Netty 来实现相同的功能。以下链接为 RPC 模块标准接口设计的草稿文档。
* [Pluggable RPC - draft1.pdf](https://issues.apache.org/jira/secure/attachment/12691096/Pluggable%20RPC%20-%20draft%201.pdf)
* [Pluggable RPC - draft2.pdf](https://issues.apache.org/jira/secure/attachment/12698710/Pluggable%20RPC%20-%20draft%202.pdf)

#### 基础组件

##### RpcEnv (目前只有一个实现类 NettyRpcEnv)
> rpc 环境

我们首先看一下 NettyRpcEnv 中的成员变量，这里主要是有个印象，之后介绍具体情况。

![NettyRpcEnv](../img/spark/spark-rpc.png)




##### RpcEndpoint

##### RpcEndpointRef

##### RpcAddress

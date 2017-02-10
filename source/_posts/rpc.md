---
title: RPC框架 & 服务治理
date: 2017-02-04 20:00:00
author: Joshua
tags: arch
categories: arch
---


## RPC框架

### 概念

`RPC`，即远程过程调用，说得通俗一点就是：调用远程计算机上的服务，就像调用本地服务一样。RPC框架是SOA架构的重要基石。

### IO模型

衡量一个RPC框架性能的好坏与否，RPC的网络IO模型的选择，至关重要。不同的网络IO模型，在高并发的状态下，处理性能上会有很大的差别。如Thrift提供支持以下IO模型的server实现:

- 单线程阻塞式IO服务模型 - `TSimpleServer`
- 多线程阻塞式IO服务模型 - `TThreadPoolServer`
- 非阻塞式IO服务模型 - `TNonblockingServer`
- 半同步半异步的服务模型 - `THsHaServer`
- 多线程半同步半异步的服务模型 - `TThreadedSelectorServer`

<!-- more -->

### Serialization & Deserialization

另外一个衡量RPC框架性能的标准，就是传输协议。通讯协议约定了RPC请求中`client`和`server`的通讯方式，序列化和反序列化属于通讯协议的一部分，是设计RPC框架时一个重要的考虑因素。在选择序列化/反序列化方式时通常需要考虑以下几个方面:

- 通用性
	-  技术层面: 序列化协议是否支持跨平台、跨语言。
	-  流行程度: 这通常意味着学习成本的高低和是否有足够的社区支持 
- 强健性
	- 协议的强健性依赖于大量而全面的测试，对于致力于提供高质量服务的系统，采用处于测试阶段的序列化协议会带来很高的风险。
- 可读性
	- 序列化和反序列化的数据正确性和业务正确性的调试往往需要很长的时间，良好的调试机制会大大提高开发效率。这方面`文本流`比`二进制流`有明显的优点。
- 性能      
	- 主要在于`空间开销`(传输的数据量)和`时间开销`(序列化和反序列化时的CPU消耗)
- 可扩展性
	- 好的序列化协议应该对于业务的扩展支持友好

主要的序列化/反序列化技术选型有以下几种:   

#### HTTP

##### XML

XML是一种常用的序列化和反序列化协议，具有跨机器，跨语言等优点。webservice就完全基于XML通讯。可读性强，但是性能较差。异构系统，open api类型的应用中常用。

##### JSON

将对象转换成JSON结构化字符串。在http协议下，这是常用的方案，具备Javascript的先天性支持。JSON可读性强，利于调试和排错，但是性能稍差。

#### TCP

##### thrift

Thrift并不仅仅是序列化协议，而是一个RPC框架。Thrift在空间开销和解析性能上有了比较大的提升。Thrift框架本身并没有透出序列化和反序列化接口，这导致其很难和其他传输层协议共同使用

##### Avro

Avro在大数据存储（RPC数据交换，本地存储）时比较常用

##### protobuf

Protobuf是一个纯粹的展示层协议，可以和各种传输层协议一起使用。它的主要问题在于其所支持的语言相对较少，另外由于没有绑定的标准底层传输层协议，在公司间进行传输层协议的调试工作相对麻烦。

##### 各语言内置序列化/反序列化机制

使用python pickle, [java序列化库](https://github.com/jobbole/awesome-java-cn#serialization)等

> 更多关于序列化和反序列化技术的对比，请参考: [序列化和反序列化](http://www.infoq.com/cn/articles/serialization-and-deserialization)

### 技术选型

笔者遇到的RPC框架的技术组合有以下几种:

- `Java: protobuf + netty`
- `Java: json + netty`
- `Python: thrift + thriftpy + thrift_connector + gunicorn`

> thrift的相关库可以参考: [thriftpy](https://github.com/eleme/thriftpy), [swift](https://github.com/facebook/swift)

## 服务治理

服务治理涵盖的内容很多，以下是服务治理的一些关键技术点

### Service Registry

服务注册中心，是服务治理最重要的组件之一。本质上是为了解耦服务提供者和服务消费者。可以基于`ZooKeeper`或者`Etcd`实现一套服务注册机制。

### LB

有了服务注册中心后，就可以在client拿到服务全部的列表，则对应的负载均衡策略在RPC框架的client里面做就可以了。这避免了haproxy配置复杂和不能热加载的问题。

### Config Center

同样可以基于zookeeper实现服务不同环境(开发，测试，预发布，生产)的配置功能

### CI/CD

可以基于`jenkins`和`docker`实现[持续集成](https://www.docker.com/use-cases/cicd)。服务的发布(灰度发布)同时需要一个完善的CMDB系统的存在

### Log

ELK是比较成熟的日志分析技术栈

### Monitoring & Alerting

关于监控和报警，请关注笔者的另外一篇文章: [服务监控和报警](https://joshua-hw.github.io/2017/01/24/monitoring_and_alerting/)

### Trace

能够得到每一个调用的`调用链路`对于服务的分析有很大的辅助作用。trace系统的实现可以参考[鹰眼下的淘宝_EagleEye with Taobao](http://www.slideshare.net/terryice/eagleeye-with-taobaojavaone)

### 限流，降级和熔断

这三者都是服务自我保护的利器，均可以依赖于服务注册及配置中心实现。
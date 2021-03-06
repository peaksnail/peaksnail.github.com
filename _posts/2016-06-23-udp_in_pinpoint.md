---
layout: post
title: "udp in pinpoint"
description: "udp and serialization"
category: apm
#tags: [pinpoint, apm]
---
{% include JB/setup %}

### UDP Protocol

今天说下agent监控元数据传输所使用的的UDP协议和其序列化

##### 数据结构文件的生成

pinpoint官方提供了thrift的数据结构源文件,使用对应的语言和平台编译
下即可得到相应的数据结构文件.

##### 数据结构及属性

首先介绍下 transaction span spanEvent annotationKey等重要数据结构的意义

* transaction 一次请求,从用户发起到请求到请求结束,这整个请求称为一次transaction
由transactionId 标识整个请求.如 A(node:front) -> B(node:middle) -> c(node:server)

* span transaction中请求由各个节点组成，一个节点相当于一个span，前后span由spanParentId和spanId关联

* spanEvent 在一个span(B: methodA(spanEvent) -> methodB(spanEvent) -> methodC(spanEvent))中，
调用链涉及到的方法都可以称为一个spanEvent来存储在span中

* annotationKey span或者spanEvent的属性，用来显示用户定义一个或者多个属性

综上，可以理解到一次完整请求为一个transaction，开发者需要做的是 组合使用spanParentId关联起span，
然后在span中监控各个函数，即可完成一个完整的transaction

##### 序列化

有了上述的基础数据结构的理解，现在说下 数据的序列化

* TCP 协议 [上一篇](http://peaksnail.github.io/apm/2016/05/24/tcp_in_pinpoint)
    * TAgent TApiInfo 以及agent汇报的信息采用tcp协议发送
    * 发送的数据格式 | type | requestId | length | buffer |
    其中buffer是要发送的数据

* UDP 协议 
    * TSpan 等跟踪数据使用udp协议发送
    * 发送的数据格式 | signature | version | type | buffer |
    其中buffer是要发送的数据

    具体可以从[源码](https://github.com/naver/pinpoint/blob/master/collector/src/main/java/com/navercorp/pinpoint/collector/receiver/udp/BaseUDPHandlerFactory.java) 开始阅读，详细的说明了从获取数据到序列化的过程,主要是通过thrift压缩协议生成具体数据后，修改头部信息来标识其具体的数据结构来实现


[UDP协议数据处理流程](https://raw.githubusercontent.com/peaksnail/peaksnail.github.com/master/_pictures/udp_serialize.png)

* 压缩协议
    上述的数据发送格式中的buffer，是将采集到的数据 经过thrift的压缩协议(TCompact protocol)
    后生成的数据

##### nodejs agent实现

基于上述理解,目前实现了pinpoint nodejs agent,欢迎使用和提出相关建议,谢谢

* [pinpoint nodejs agent](https://github.com/peaksnail/pinpoint-node-agent)

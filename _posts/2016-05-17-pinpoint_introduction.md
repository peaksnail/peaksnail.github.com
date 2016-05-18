---
layout: post
title: "Pinpoint Introduction"
description: "introduce pinpoint apm"
category: apm
#tags: [pinpoint, apm]
---

### APM 

介绍Pinpoint 之前,首先了解APM(application performance monitor/manage)

应用性能监控/管理:
主要关注在分布式应用性能方向,如qps,每次RPC请求所消耗的时间以及细化到每次访问涉及到的模块，
及在各个模块上的性能数据等。随着目前系统越来越趋向于多层次，多模块的发展，
APM近年来发展也比较迅速，目前国内外的APM产品都是基于Google 
论文 [dapper](http://bigbully.github.io/Dapper-translation/) 实现的。
Pinpoint也不例外，具体可移步 [github](https://github.com/naver/pinpoint)

### Pinpoint 具体架构
#### 大概架构

目前apm产品的架构都遵循着dapper类似的架构，主要有以下四部分

* agent 
* collector
* web
* database

#####database

数据库 可考虑下系统的规模，以及结合现有的基础架构和后续计划，如果后续要基于hadoop进行
大数据分析，就可以直接使用hdfs或者hbase，也可以使用mongodb等

#####web

展示这块 主要是从database读取数据展示
    
#####collector
接受来自agent Trace的采集信息，并整合存储到database中，这块主要关注agent和collector之间的
传输协议和数据的序列化传输

#####agent
agent采集Trace 数据，然后发送至collector。agent主要关注 采集的方式。即如何采集 
        
* 能够做到对业务的侵入性低
* 性能损耗低
        
对于第一点，目前产品有两种实现方式，一种是如BAT等大公司，内部已经有一套自己的基础架构，所以
将底层的基础架构改造即可，对上层业务做到透明

另外一种是 类似java bci技术(字节码增强),在加载模块的时候修改代码逻辑来实现对业务的透明
但是这种方式相对其他来说，风险要高一点。因为它相当于是直接修改业务代码，插入采集代码
           
Pinpoint 目前只支持Java程序，采用bci技术实现

####多语言模块系统

前一段时间调研了下各个apm产品，发现在没有统一的基础架构的基础上，对多语言模块系统的支持都不是很好，一般是只支持特定语言
的分布式系统。所以采用Pinpoint的基础上，开发其他语言的agent。

        
### 国内外的一些产品

 * [淘宝 鹰眼](http://wenku.it168.com/d_001241168.shtml) 
 * [大众点评CAT](http://mp.weixin.qq.com/s?__biz=MzA5Nzc4OTA1Mw==&mid=410422335&idx=1&sn=6f155b1b5211a26cf0c1e92f954c9874#rd)
 * [zipkin](https://github.com/openzipkin/zipkin)
 * [商业产品 oneApm](http://www.oneapm.com/)
 * [new relic](https://newrelic.com/)
 * [apm 汇总](https://github.com/sdcuike/DistributedTracingSystem)

### 计划

 目前Pinpoint团队并没有要支持其他语言的计划，毕竟团队人力不足，而且系统还在优化之中，同时缺少重要的文档来支持开发者
 开发不同语言的agent(只有有开发java plugin的文档，但是还远远不够),而且目前Pinpoint 的数据传输格式等等，
 将来都不确定说会不会变化，所以暂时文档是不会出现的。

 这段时间 我自己通过阅读部分代码，并开发了nodejs agent demo，结合我最近的实践，想写下Pinpoint的部分实现，
 这样可以基于这部分实现自己需要的agent demo.

 下一篇，我先说下Pinpoint中agent和collector的传输协议和序列化实现


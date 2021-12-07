---
layout: post
title: "datax 状态管理器的实现"
description: "datax 基于redis的状态管理器"
category: datax redis
#tags: [datax, redis, data]
---
{% include JB/setup %}

## 为何需要datax的状态管理器

目前业务刚起步，数据量不多，通过azkaban+datax来实现mongodb数据同步到clickhouse，通过azkaban设置的定时任务来运行，但是每次同步任务运行时，怎么知道上次同步到哪里呢

实际上，随着业务的发展，对应的技术架构也会随着调整，按照目前流行的架构,可以通过mongodb的changeStream+消息队列+计算引擎即可实现，但实际上是数据量还没到一定要这样做的地步，目前看azkaban+datax足够解决问题，所以考虑在此架构下，实现以下两点：

    1 mongodb中每条数据都有一个updateDate字段表示最后更新时间，并作为索引，每次同步时，对一定时间范围内的数据按updateDate进行升序排序

    2 最后一条数据的updateDate表示最近一次同步的最新时间，下一次任务运行时就可以知道从哪里开始同步

刚开始考虑直接使用mongodb或者mysql存储每次同步任务中数据的最新更新时间，但是数据是在datax内部在处理，考虑到各种异常和脏数据，同时每次datax reader获取数据时，同步条件需要根据最新的更新时间来调整，以及各个自定义的参数处理，所以考虑在datax中实现一个状态管理器，作为一个插件，后续其他数据源也可使用

## 理解datax插件

datax实际上有两种类型的插件

    1 各种数据源的reader/writer 即读写同步的插件

    2 reader/writer的内部插件，对任务运行时的数据同步，信息采集和统计等（collertor插件）

所以需要实现一个内部的状态管理插件，由reader/writer调用，记录下每次同步的updateDate

既然要实现这么一个插件，首先需要了解下datax怎么启动，怎么同步,怎么统计执行的信息

    1 datax 通过调用datax.py(core/src/main/bin/datax.py)启动任务，由datax.py中定义的启动命令可以知道datax如何启动 

    ENGINE_COMMAND = "java -server ${jvm} %s -classpath %s  ${params} com.alibaba.datax.core.Engine -mode ${mode} -jobid ${jobid} -job ${job}" % (
    DEFAULT_PROPERTY_CONF, CLASS_PATH)

    即从Engine(core/src/main/java/com/alibaba/datax/core)出发，看下datax如何运行

    2 通过参数解析和加载配置文件(core/src/main/conf/core.json)，会启动TaskGroupContainer（默认配置下）,然后由TaskGroupContainer启动TaskExecutor，最后由TaskExecutor加载对应的reader/writer和相关的统计插件进行数据同步

    可以具体看下TaskGroupContainer实现(core/src/main/java/com/alibaba/datax/core/taskgroup/TaskGroupContainer.java)

    3 TaskGroupContainer相当于是任务管理器，负责具体执行器，也就是TaskExecutor的生成和监控， TaskExecutor会去执行真正的数据同步, 执行时会通过内部统计插件统计相关信息。具体如图：

    [datax数据处理逻辑](https://raw.githubusercontent.com/peaksnail/peaksnail.github.com/master/_pictures/datax_task.jpg)

    4 接下来具体以mongoreader看下如何使用内部的plugin
    每个Reader都必须实现两个内部类 Job内部类和Task内部类
    Job类 负责整个任务的运行
    Task类 负责具体的数据同步
    同时各自都有对应的内部plugin，Job类的plugin用来处理整个任务的运行状态和信息，Task类的plugin用来处理同步时的问题，比如脏数据处理，错误输出等

    而想要实现的状态管理器是跟着数据同步走的，所以需要参考Task级别的plugin来实现


### 实现

理解完 datax 内部插件的实现，基于此，开始实现datax的数据状态管理

先看下collector插件如何加载

1 主要的流程在TaskGroupContainer中实现，从start()函数开始，会启动TaskExecutor(完整的执行器，包括了reader和writer),TaskExecutor中通过函数generateRunner会生成两个线程，一个是reader线程，一个是writer线程

2 generateRunner根据任务配置获取对应的reader/writer插件进行初始化，同时会根据系统配置文件加载collector插件

3 在对应的reader/writer插件实现中直接调用collertor对应的函数即可收集相关信息

通过以上分析，参考collertor插件的实现，即可实现状态管理的插件

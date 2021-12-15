---
layout: post
title: "azkaban接口"
description: "azkaban 接口"
category: azkaban
#tags: [azkaban]
---
{% include JB/setup %}

#### 背景 

由于业务上的一些需求，最近看了下azkaban的一些接口，[官网](https://azkaban.github.io/azkaban/docs/latest/#ajax-api)上介绍的接口已经很丰富了，但是看了下azkaban的源码，发现还是有挺多接口没在文档上说明的，接下来就简单介绍几个接口。
azkaban提供了基于servlet实现的ajax api。从官方文档了解到，azkaban主要包含三个组件:
    
    azkaban web server
    azkaban executor server
    database

先从启动脚本来看，web server和executor server的入口在哪里

    start-internal-web.sh
    java $AZKABAN_OPTS $JAVA_LIB_PATH -cp $CLASSPATH azkaban.webapp.AzkabanWebServer -conf $conf $@ &

    start-internal-executor.sh
    java $AZKABAN_OPTS $JAVA_LIB_PATH -cp $CLASSPATH azkaban.execapp.AzkabanExecutorServer -conf $conf $@ &

所以我们需要从 azkaban.webapp.AzkabanWebServer和azkaban.execapp.AzkabanExecutorServer进入看下各自都实现了哪些接口
    

#### web server接口

从AzkabanWebServer类的实现可以看到web server默认端口8081,接口配置由 函数configureRoutes加载路由,不同模块有各自的路由前缀

    routesMap.put("/index", new ProjectServlet());
        routesMap.put("/manager", new ProjectManagerServlet());
        routesMap.put("/executor", new ExecutorServlet());
        routesMap.put("/schedule", new ScheduleServlet());
        routesMap.put("/triggers", new TriggerManagerServlet());
        routesMap.put("/flowtrigger", new FlowTriggerServlet());
        routesMap.put("/flowtriggerinstance", new FlowTriggerInstanceServlet());
        routesMap.put("/history", new HistoryServlet());
        routesMap.put("/jmx", new JMXHttpServlet());
        routesMap.put("/stats", new StatsServlet());
        routesMap.put("/notes", new NoteServlet());
        routesMap.put("/", new IndexRedirectServlet(defaultServletPath));
        routesMap.put("/status", new StatusServlet(this.statusService));

而这些接口主要都在azkaban-web-server/src/main/java/azkaban/servlet下实现了

这些接口在官网上基本都说明了用法,这里就不再说了，主要看下executor

#### executor接口

从AzkabanExecutorServer类的实现可以看到executor server默认端口为12321,同时在ExecJettyServerModule类中实现路由配置，如下

    private Context createRootContext(@Named(EXEC_JETTY_SERVER) final Server server) {
        final Context root = new Context(server, "/", Context.SESSIONS);
        root.setMaxFormContentSize(MAX_FORM_CONTENT_SIZE);

        root.addServlet(new ServletHolder(new ExecutorServlet()), "/executor");
        root.addServlet(new ServletHolder(new JMXHttpServlet()), "/jmx");
        root.addServlet(new ServletHolder(new StatsServlet()), "/stats");
        root.addServlet(new ServletHolder(new ServerStatisticsServlet()), "/serverStatistics");
        return root;
      }

选executor接口来看，在ExecutorServlet类中handleRequest方法实现具体接口的相关逻辑

* 通过对比官方文档，可以看到官方文档介绍的都是web-server提供的接口，并没有介绍executor接口，而我们可以通过executor提供的接口来进行一些额外的操作，比如监控等

1 登陆

    请求登陆
    curl -k -X POST --data "action=login&username=azkaban&password=azkaban" http://localhost:8081
    返回
    {
    "session.id" : "4372a68b-fd50-4552-9f6e-72d3d44360e4",
    "status" : "success"
    }

2 获取节点状态

    请求节点状态
    curl -k -X POST --data "session.id=4372a68b-fd50-4552-9f6e-72d3d44360e4" http://localhost:12321/executor\?action\=getStatus
    返回
    {"isActive":"true","executor_id":"2","status":"success"}

3 节点停止运行和激活
    
    请求节点停止运行
    curl -k -X POST --data "session.id=4372a68b-fd50-4552-9f6e-72d3d44360e4" http://localhost:12321/executor\?action\=deactivate
    返回
    {"status":"success"}

    请求节点状态
    curl -k -X POST --data "session.id=4372a68b-fd50-4552-9f6e-72d3d44360e4" http://localhost:12321/executor\?action\=getStatus
    返回
    {"isActive":"false","executor_id":"2","status":"success"}

    请求激活节点
    azkaban-solo-server curl -k -X POST --data "session.id=4372a68b-fd50-4552-9f6e-72d3d44360e4" http://localhost:12321/executor\?action\=activate
    返回
    {"status":"success"}   

    请求节点状态
    curl -k -X POST --data "session.id=4372a68b-fd50-4552-9f6e-72d3d44360e4" http://localhost:12321/executor\?action\=getStatus
    返回
    {"isActive":"true","executor_id":"2","status":"success"}

以上简单举几个操作说明下，其他详细的接口可以具体看下代码
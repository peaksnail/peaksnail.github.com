---
layout: post
title: "How to transport data on Pinpoint"
description: "transport protocol and serialize"
category: apm
#tags: [pinpoint, apm]
---
{% include JB/setup %}

### Transport Protocol

直接进入正题，今天介绍下 pinpoint agent 和collector之间的通信协议和序列化反序列化的实现

* 在阅读前，需要netty，thrift的基础知识

#### agent

先描述下Agent 启动的大概流程，Agent 启动流程：

* 读取pinpoint-*.properties和hbase.properties相关配置
* 初始化 tcp udp socket状态
* 生成agentInfo信息通过TCP协议向collector汇报
* 对所有类中申明的方法(api)生成methodDescriptor(描述方法的对象) 
  methodDescriptor中包含了唯一标识的apiId(作用是每次传输span的时候，对api只传输apiId即可，
  减少网络带宽)通过TCP协议传输至collector，成功则会收到collector发来的成功码
* 完全启动后，定时汇报agent信息，包括 jvm信息，主机cpu等相关可以采集到的数据
* 随后对于涉及到的模块,如果被请求到，则会生成相关Trace的数据，发送到collector进行解析和存储

#### collector

这里我们先看下collector是如何接收和处理agent的数据包的.即 TCP协议的一些处理.
UDP协议数据包格式和处理方式会有点不同，下文会分析到

Pinpoint collector模块使用netty异步框架来实现数据的接收解析和处理,所以这里需要去理解netty做了些什么来解析接收到的
数据和进一步的数据处理

更多关于netty信息，请移步[netty](http://netty.io/)

-- 对netty有相关的基础知识后，我们就会知道，在netty中，可以实现encode和decode来对数据进行编解码，pinpoint 实现了这
两个类[github](https://github.com/naver/pinpoint/tree/master/rpc/src/main/java/com/navercorp/pinpoint/rpc/codec)

从 [github](https://github.com/naver/pinpoint/blob/master/rpc/src/main/java/com/navercorp/pinpoint/rpc/server/ServerPipelineFactory.java)
看出，数据先经过编解码再进行处理

##### decode 

* 先看下collecotr对数据的解码
    ``` java
    protected Object decode(ChannelHandlerContext ctx, Channel channel, ChannelBuffer buffer) throws Exception {
            if (buffer.readableBytes() < 2) {
                return null;
            }
            buffer.markReaderIndex();
            final short packetType = buffer.readShort();
            switch (packetType) {
                case PacketType.APPLICATION_SEND:
                    return readSend(packetType, buffer);
                case PacketType.APPLICATION_REQUEST:
                    return readRequest(packetType, buffer);
                case PacketType.APPLICATION_RESPONSE:
                    return readResponse(packetType, buffer);
                case PacketType.APPLICATION_STREAM_CREATE:
                    return readStreamCreate(packetType, buffer);
                case PacketType.APPLICATION_STREAM_CLOSE:
                    return readStreamClose(packetType, buffer);
                case PacketType.APPLICATION_STREAM_CREATE_SUCCESS:
                    return readStreamCreateSuccess(packetType, buffer);
                case PacketType.APPLICATION_STREAM_CREATE_FAIL:
                    return readStreamCreateFail(packetType, buffer);
                case PacketType.APPLICATION_STREAM_RESPONSE:
                    return readStreamData(packetType, buffer);
                case PacketType.APPLICATION_STREAM_PING:
                    return readStreamPing(packetType, buffer);
                case PacketType.APPLICATION_STREAM_PONG:
                    return readStreamPong(packetType, buffer);
                case PacketType.CONTROL_CLIENT_CLOSE:
                    return readControlClientClose(packetType, buffer);
                case PacketType.CONTROL_SERVER_CLOSE:
                    return readControlServerClose(packetType, buffer);
                case PacketType.CONTROL_PING:
                    PingPacket pingPacket = (PingPacket) readPing(packetType, buffer);
                    if (pingPacket == PingPacket.PING_PACKET) {
                        sendPong(channel);
                        // just drop ping
                        return null;
                    }
                    return pingPacket;
                case PacketType.CONTROL_PONG:
                    logger.debug("receive pong. {}", channel);
                    readPong(packetType, buffer);
                    // just also drop pong.
                    return null;
                case PacketType.CONTROL_HANDSHAKE:
                    return readEnableWorker(packetType, buffer);
                case PacketType.CONTROL_HANDSHAKE_RESPONSE:
                    return readEnableWorkerConfirm(packetType, buffer);
            }
            logger.error("invalid packetType received. packetType:{}, channel:{}", packetType, channel);
            channel.close();
            return null;
        }
    ```

* 可以看到collector端收到消息后，读取buffer的前两个bytes(16bits,short类型) 来判断是什么类型的数据，根据相应的
类型,读取数据，返回对应的数据对象。这里以 PacketType.APPLICATION_SEND 来说明下后续的处理

* readSend(packetType, buffer); 中buffer是SendPacket类型，进入SendPacket中readBuffer函数
继续读取4bytes(32bits int)数据长度，通过这个长度读取实际数据长度，返回SendPacket对象,然后转给相应的handler处理。
可以发现其他数据对象通过同样的方法进行处理

* [packet type](https://github.com/naver/pinpoint/tree/master/rpc/src/main/java/com/navercorp/pinpoint/rpc/packet)
这里是所有数据包类型


* 然后转到DefaultPinpointServer MessageReceived方法,参数 message即decode获取到的实际Packet数据,然后进行实际的数据处理的响应
    ``` java
    public void messageReceived(Object message) {
            if (!isEnableCommunication()) {
                // FIXME need change rules.
                // as-is : do nothing when state is not run.
                // candidate : close channel when state is not run.
                logger.warn("{} messageReceived() failed. Error: Illegal state this message({}) will be ignore.", objectUniqName, message);
                return;
            }
            
            final short packetType = getPacketType(message);
            switch (packetType) {
                case PacketType.APPLICATION_SEND: {
                    handleSend((SendPacket) message);
                    return;
                }
                 case PacketType.APPLICATION_REQUEST: {
                    handleRequest((RequestPacket) message);
                    return;
                }
                case PacketType.APPLICATION_RESPONSE: {
                    handleResponse((ResponsePacket) message);
                    return;
                }
                case PacketType.APPLICATION_STREAM_CREATE:
                case PacketType.APPLICATION_STREAM_CLOSE:
                case PacketType.APPLICATION_STREAM_CREATE_SUCCESS:
                case PacketType.APPLICATION_STREAM_CREATE_FAIL:
                case PacketType.APPLICATION_STREAM_RESPONSE:
                case PacketType.APPLICATION_STREAM_PING:
                case PacketType.APPLICATION_STREAM_PONG:
                    handleStreamEvent((StreamPacket) message);
                    return;
                case PacketType.CONTROL_HANDSHAKE:
                    handleHandshake((ControlHandshakePacket) message);
                    return;
                case PacketType.CONTROL_CLIENT_CLOSE: {
                    handleClosePacket(channel);
                    return;
                }
                case PacketType.CONTROL_PING: {
                    handlePingPacket(channel, (PingPacket) message);
                    return;
                }            
                default: {
                    logger.warn("invalid messageReceived msg:{}, connection:{}", message, channel);
                }
            }
        }

    ```

* 再到profiler模块中阅读源码,看到 agentInfo 是SendPacket(头部信息为type(2bytes) 数据长度(4bytes) agentInfo数据)，
apiInfo是RequestPacket(头部信息为type(2bytes) 请求id(4bytes) 数据长度(4bytes) apiInfo数据)
所以encode/decode就是按照上述方法实现.
其他类型可以具体再深入，目前有这个两个类型就足够了.那agent和api信息如何存储呢，看下文

* 获取到实际数据后 反序列化处理 序列化处理下文会提到

#### encode

encode 实际上就和数据decode流程相反了


####  TAgentInfo 和 TApiMetaData 序列化和反序列化

上面说了agent启动后，两个最重要数据类型TAgentInfo和TApiMetaData的发送协议。这里具体说下这两个数据类型的序列化
和反序列化.需要了解下[thrift](http://thrift.apache.org/)的基础知识

* 首先看下官方[wiki](https://github.com/naver/pinpoint/wiki/Technical-Overview-Of-Pinpoint#using-binary-format-thrift)
pinpoint使用thrift的压缩协议进行序列化和反序列化数据

* [TcpDataSender](https://github.com/naver/pinpoint/blob/master/profiler/src/main/java/com/navercorp/pinpoint/profiler/sender/TcpDataSender.java)

* sendPacket 函数，可见在发送之前进行了使用[serialize](https://github.com/naver/pinpoint/blob/master/thrift/src/main/java/com/navercorp/pinpoint/thrift/io/HeaderTBaseSerializer.java)函数序列化

* 从数据序列化这里可以看到，pinpoint是对数据使用thrift压缩协议序列化后，再修改头部信息
Header 4bytes，分别为signature(固定值 1byes) version(固定值 1bytes) type(变量 2bytes)
通过设置头部信息中的type来区分不同的对象



--综上 可以理解 agent启动后 向collector基础数据的汇报
整个过程 实际就是 选择发送的对象，通过thrift的压缩协议生成binary数据，
然后根据不同的对象设置相应的头部信息，最后设置数据包的类型，向collector汇报


[TCP协议数据处理流程](https://raw.githubusercontent.com/peaksnail/peaksnail.github.com/master/_pictures/tcp_serialize.png)


--下一篇将会说下 Trace信息的序列化和反序列化(UDP协议) 及其发送机制

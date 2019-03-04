---
title: WebSocket协议初探和在Android里的应用
date: 2018-09-12 20:11:15
tags: Websocket
categories: [网络,WebSocket]
---

## 一、内容概览

本文将从以下几方面讲解：  

1. 什么是WebSocket?使用场景？  
2. 如何建立连接的？
3. 数据帧格式
4. 如何维持连接的？
5. websocket在Android里的应用

## 二、什么是WebSocket

WebSocket一种浏览器与服务器进行全双工通讯的网络技术，属于应用层协议。它基于TCP传输协议，并复用HTTP的握手通道。WS(WebSocket)相对于Http有如下优点：

1. 支持双向通信，实时性更强。
2. 更好的二进制支持。
3. 较少的控制开销。连接创建后，ws客户端、服务端进行数据交换时，协议控制的数据包头部较小。在不包含头部的情况下，服务端到客户端的包头只有2~10字节（取决于数据包长度），客户端到服务端的的话，需要加上额外的4字节的掩码。而HTTP协议每次通信都需要携带完整的头部。
4. 支持扩展。ws协议定义了扩展，用户可以扩展协议，或者实现自定义的子协议。（比如支持自定义压缩算法等）

WS主要应用于需要消息实时发送的系统中，类似于QQ或者股票交易类等实时性要求强的项目。

<!--more-->

## 三、如何建立连接

### 1、客户端发送升级协议请求

```xml
GET / HTTP/1.1
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: 0Qpv22jNrTQ2jy3V1niBCQ==
Sec-WebSocket-Version: 13
Host: 127.0.0.1:8181
Accept-Encoding: gzip
User-Agent: okhttp/3.11.0
```
可以看到采用的Http报文格式，且只支持Get方法。讲解主要头部字段：  

* `Connection: Upgrade` ：表示要升级协议。
* `Upgrade: websocket`：表示要升级到websocket协议。
* `Sec-WebSocket-Version: 13`：表示websocket的版本。如果服务端不支持该版本，需要返回一个`Sec-WebSocket-Version`header，里面包含服务端支持的版本。
* `Sec-WebSocket-Key`：与后面服务端响应首部的`Sec-WebSocket-Accept`是配套的，提供基本的防护，比如恶意的连接，或者无意的连接。

### 2、服务端响应协议升级

```xml
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: 5vt/AbHaiRcJ/S+njyfEodK/2AM=
```
服务端返回内容如下，状态代码101表示协议切换。到此完成协议升级。  
握手成功后，通讯不再使用Http协议，而是采用WS独立的数据帧，下一节我们讲解数据帧。

## 四、数据帧格式

数据帧格式如下图，从左到右，单位是比特，内容包含了标识、操作代码、掩码、数据、数据长度等。

<img src="/img/201809/websocket_data.jpg" alt="websocket_data" style="width: 600px;">

图中个名词含义如下：

```xml
FIN：1bit，是否为信息的最后一帧
RSV 1-3：1bit，备用，默认为0
opcode：4bit，帧类型
                         0x00 连续消息分片
                         0x01 文本消息分片
                         0x02 二进制消息分片
                         0x03 ~ 0x07 为将来的非控制消息片段保留测操作吗
                         0x08 连接关闭
                         0x09 心跳检查ping
                         0x0a 心跳检查pong
                         0x0b ~ 0x0f 为将来的控制消息片段保留的操作码
MASK：定义传输的数据是否有加掩码，如果设置为1,掩码键必须放在masking-key区域，客户端发送给服务端的所有消息，此位的值都是1
payload length：7bit，传输数据长度，以字节为单位。当这个长度为7bit数字为126时，紧随其后的2个字节也是表示数据长度。当这个长度为7bit数字为127时，紧随其后的8个字节也是表示数据长度。
Masking-key：0或者4bit，只有当MASK设置为1时才有效。
Playload data：负载数据，为扩展数据和应用数据之和，Extension data + Application data。
Extension data：扩展数据，如果客户端和服务端没有特殊的约定，那么扩展数据长度始终为0
Application data：应用数据，

```

## 如何维持连接

使用心跳包来维持连接，ping、pong操作对应的是WS的两个控制帧，opcode分别是0x09和0x0a。

## 应用 

说起okhttp，想必很多Android的老司机都不会陌生，其实okhttp还支持WS，真的是太牛逼了，而且okhttp已经把心跳机制封装好了，真的是非常优秀！因为最近项目中用到了，所以想着二次封装下，便于以后的使用，项目地址：[WsManager](https://github.com/Jesse505/WsManager)

## 写在后面

其实WS还有很多需要学习的地方，由于时间和能力有限，先总结到这个地方，以后会继续更新






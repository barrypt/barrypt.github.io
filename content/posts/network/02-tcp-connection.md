---
title: "计算机网络(二)---TCP三次握手四次挥手"
description: "TCP三次握手四次挥手具体步骤及其原因分析"
date: 2019-01-08 22:00:00
draft: false
categories: ["Network"]
tags: ["Network"]
---

本文主要介绍了`TCP`的三次握手和四次挥手具体步骤及其原因分析。

<!--more-->

### 1. 三次握手

![](https://github.com/barrypt/blog/raw/master/images/network/tcp-connection-three.jpg)

`step1:第一次握手`
建立连接时，客户端发送SYN包到服务器，其中包含客户端的初始序号seq=x，并进入**SYN_SENT**状态，等待服务器确认。（其中，SYN=1，ACK=0，表示这是一个TCP连接请求数据报文；序号seq=x，表明传输数据时的第一个数据字节的序号是x）。

`step2:第二次握手`
服务器收到请求后，必须确认客户的数据包。同时自己也发送一个SYN包，即SYN+ACK包，此时服务器进入**SYN_RCVD**状态。（其中确认报文段中，标识位SYN=1，ACK=1，表示这是一个TCP连接响应数据报文，并含服务端的初始序号seq(服务器)=y，以及服务器对客户端初始序号的确认号ack(服务器)=seq(客户端)+1=x+1）。

`step3:第三次握手`

客户端收到服务器的SYN+ACK包，向服务器发送一个序列号(seq=x+1)，确认号为ack(客户端)=y+1，此包发送完毕，客户端和服务器进入**ESTABLISHED **(TCP连接成功)状态，完成三次握手。

```
建立连接前要确认客户端和服务端的接收和发送功能是否正常。
第一次客户端发送SYN时 什么也确认不了
第二次服务端发送SYN+ACK 可以确认服务端发送功能正常
第三次 客户端收到服务端发送的YSN+ACK 可以确认客户端发送接收功能正常
最后客户端发送ACK 服务端接收到后 可以确认服务端发送功能正常
到此就确认完毕了。
```

### 2. 四次挥手

![](https://github.com/barrypt/blog/raw/master/images/network/tcp-close-connection-four.jpg)

`step1：第一次挥手`
首先，客户端发送一个FIN，用来关闭客户端到服务器的数据传送，然后等待服务器的确认。其中终止标志位FIN=1，序列号seq=u。 **客户端**进入**FIN_WAIT1**状态

```
我（Client端）没有数据要发给你（Server端）了"，但是如果你（Server端）还有数据没有发送完成，则不必急着关闭Socket，可以继续发送数据。所以你先发送ACK
```

`step2：第二次挥手`
**服务器**收到这个FIN进入**CLOSE_WAIT**状态，然后它给客户端发送一个ACK，确认ack为收到的序号加一。

**客户端**收到ACK应答后进入**FIN_WAIT2**状态

```
告诉Client端，你的请求我收到了，但是我（Server端）还没准备好，请继续你等我的消息"
```

`step3：第三次挥手`
服务端关闭服务器到客户端的连接，发送一个FIN给客户端。**服务端**进入**LAST_ACK**状态

```
告诉Client端，好了，我（Server端）这边数据发完了，准备好关闭连接了
```

`step4：第四次挥手`

**客户端**收到FIN后，进入**TIME_WAIT**状态	并发回一个ACK报文确认，并将确认序号seq设置为收到序号加一。

服务端收到客户端回复的ACK后立即关闭，服务端进入**CLOASED**状态

而客户端要等待2MSL后关闭 进入**CLOASED**状态

```
Client端收到FIN报文后，"就知道可以关闭连接了，所以发送ACK。但是他还是不相信网络，怕Server端不知道要关闭，所以发送ACK后没有立即，而是进入TIME_WAIT状态，如果Server端没有收到ACK那么自己还可以重传。Server端收到ACK后，"就知道可以断开连接了"。Client端等待了2MSL后依然没有收到回复，则证明Server端已正常关闭，那好，我Client端也可以关闭连接了。Ok，TCP连接就这样关闭了！
```

### 3. TIME-WAIT状态详解

为什么Client端要先进入TIME-WAIT状态，等待2MSL时间后才进入CLOSED状态？

**保证TCP协议的全双工连接能够可靠关闭，保证这次连接的重复数据段从网络中消失**

假设由于IP协议的不可靠性或者是其它网络原因，导致Server没有收到Client最后回复的ACK。那么Server就会在超时之后继续发送FIN，Client端在等待2MSL时间后都没收到信息，说明Server端已经收到自己发送的ACK并且成功关闭了。
**假设CLient端直接关闭了：**

```java
1.由于IP协议的不可靠性或者是其它网络原因，导致Server没有收到Client最后回复的ACK。那么Server就会在超时之后继续发送FIN，此时由于Client已经CLOSED了，就找不到与重发的FIN对应的连接，最后Server就会收到RST而不是ACK，Server就会以为是连接错误把问题报告给高层。这样的情况虽然不会造成数据丢失，但是却导致TCP协议不符合可靠连接的要求。所以，Client不是直接进入CLOSED，而是要保持TIME_WAIT，当再次收到FIN的时候，能够保证对方收到ACK，最后正确的关闭连接。

2.如果Client直接CLOSED，然后又再向Server发起一个新连接，我们不能保证这个新连接与刚关闭的连接的端口号是不同的。也就是说有可能新连接和老连接的端口号是相同的。一般来说不会发生什么问题，但是还是有特殊情况出现：假设新连接和已经关闭的老连接端口号是一样的，如果前一次连接的某些数据仍然滞留在网络中，这些延迟数据在建立新连接之后才到达Server，由于新连接和老连接的端口号是一样的，又因为TCP协议判断不同连接的依据是socket pair，于是，TCP协议就认为那个延迟的数据是属于新连接的，这样就和真正的新连接的数据包发生混淆了。所以TCP连接还要在TIME_WAIT状态等待2倍MSL，这样可以保证本次连接的所有数据都从网络中消失。

```

**2MSL:Maximum Segment Lifetime 即数据在网络中保存的最大时间。**

*简单易懂的说法:*

```
假设Client端发起中断连接请求，也就是发送FIN报文。Server端接到FIN报文后，意思是说"我Client端没有数据要发给你了"，但是如果你还有数据没有发送完成，则不必急着关闭Socket，可以继续发送数据。所以你先发送ACK，"告诉Client端，你的请求我收到了，但是我还没准备好，请继续你等我的消息"。这个时候Client端就进入FIN_WAIT状态，继续等待Server端的FIN报文。当Server端确定数据已发送完成，则向Client端发送FIN报文，"告诉Client端，好了，我这边数据发完了，准备好关闭连接了"。Client端收到FIN报文后，"就知道可以关闭连接了，但是他还是不相信网络，怕Server端不知道要关闭，所以发送ACK后进入TIME_WAIT状态，如果Server端没有收到ACK则可以重传。“，Server端收到ACK后，"就知道可以断开连接了"。Client端等待了2MSL后依然没有收到回复，则证明Server端已正常关闭，那好，我Client端也可以关闭连接了。Ok，TCP连接就这样关闭了！
```

### 4. TCP 的有限状态机

红色为客户端 蓝色为服务端 细箭头为异常变化

![TCP](https://github.com/barrypt/blog/raw/master/images/network/tcp-status-map.png)

### 5. 参考

`https://www.baidu.com/link?url=_mlor11BLttd1jmMU4k9OP0gqcjNKhZQ9fJuvbMOhkuH9-lVeB-y3VIVK1neZURi_tmR3rg1lj2lfgvvGhTV-q&wd=&eqid=d0144c250007b69c000000035bfdfafc`
---
title: TCP相关思考
date: 2018-02-25 10:07:12
tags:
- 网络
categories:
- 思考总结
---
{% asset_img default.jpg TCP %}

## TCP三次握手
TCP之所以需要三次握手，问题的本质是一个通信问题。即在信道不可靠的情况下，如何使通信双方就某一问题达成一致。这就需要通信的双方通过某种方式来保证信道在一定程度上可靠。
因此，要让通信的双方都能确认自己的信息能传达到对方，同时能够收到对方的信息。如此，信息一去一来，对任意一方来说都需要两次握手才能让他自己确认，自己的信息能被对方接收同时自己能正确接收对方的信息。结论，三次通信是理论上的最小值。
**通信双方经过三次握手达成建立连接的共识后，自然而然解决掉了诸多通信的异常问题。因此用解决“为了防止已失效的连接请求报文段突然又传送到了服务端，因而产生错误”之类的说法说明三次握手的作用不够准确。三次握手的实质就是通信双方基于不那么可靠的信道建立可靠通信连接的一种方式。**

## TCP四次挥手
TCP通信的双方断开连接需要双方确认没有数据发送并请求断开，在已经建立连接的情况下理论上需要双方各两次通信。一次请求，一次ACK，可以确认一方没有数据发送并断开连接。但最后一次ACK怎么确保一定送达，这里通过time_wait状态加上重试机制保证。
当一方发出最后一个ACK后等待2MSL时间，期间若对方没有收到ACK必会重试，2MSL内必会收到重试FIN报文。因此，保证了2MSL时间后对方收到ACK，可以正常断开TCP连接。
**个人觉得2MSL不够准确，应该取MSL+超时重传累积时间的总和。猜测取2MSL是因为，最差可能是1MSL后重试，并且重试一次足矣，网络也没那么差**

## TCP心跳的意义
由于TCP连接是虚拟的连接，当一方断网或是断电等异常断开后，另一方是无法知道TCP连接已经断开的。为了减少这种不必要的连接**浪费资源**，需要心跳及时发现连接断开这种情况。心跳有TCP自带的keep_alive和应用层心跳。**重点：运营商也会断掉长时间没有数据的连接**
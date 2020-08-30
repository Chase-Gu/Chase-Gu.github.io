---
layout:     post
title:      面向连接传输协议—TCP
subtitle:   Connection oriented transport protocol-TCP
date:       2019-12-17
author:     Chase Gu
header-img: img/post-bg-os-metro.jpg
catalog: true
tags:
    - 学习笔记
    - 计算机网络
---

# 3.6 面向连接传输协议—TCP

**目录：[计算机网络-课程笔记目录](https://gushichen.gitee.io/2019/10/31/network-catalog/)**



### 前言

- 点对点
- 可靠的、按序的字节流
- 流水线机制
  - TCP拥塞控制和流量控制机制设置窗口尺寸，**窗口尺寸是动态的**
- 发送方/接收方缓存：采用介于GBN和SR之间的，故都有缓存
- 全双工
  - 同一连接中双向传输
- 面向连接
  - 通信双方在发送数据之前必须建立连接（**面向连接**）
  - 连接状态只在链接的两端中维护，在沿途节点中并不维护状态
  - TCP连接包括：两台主机上的缓存、链接状态变量、socket等
- 流量控制机制





### TCP段结构

- <img src="/img-post/2019-12-22-network-tcp/TCP数据报格式.png" alt="TCP数据报格式" style="zoom: 33%;" />

- sequence number 和 ack number：跟字节数有关，不是段的（？）

- 序列号（sequence number）

  - segment中第一个**字节**的编号，而不是segment的编号
    - 如发送两个，不是1、2，可能是501、502，是字节的编号
  - 建立TCP连接时，双方随机选择序列号

- ACKs：

  - **希望**接收到的下一个**字节**的序列号（GBN和SR的ACK是表示这个收到了）
  - 累计确认：**该序列号**之前的所有字节均已被正确接收到**（像GBN）**序列号不一定连续

- 选项

  - U：URG，一般不使用
  - A：ACK，表明ACK字段是否有效
  - P：PSH，立刻推到上层，没用
  - RSF：RST、SYN、FIN，与连接有关

- Receive window：：接收窗口的大小，所愿意接收的字节的数目，可以流量控制

- Checksum：Internet校验和

- Q：接收方如何处理乱序到达的segment？

  A：TCP规范中没有规定，有TCP的实现者做出决策





### TCP可靠数据传输

- 流水线+累计确认+单一重传定时器
- 触发重传事件
  - 超时
  - 收到重复ACK

#### RTT和超时（考点）

- 如何设置定时器的超时时间
  - 大于RTT：但是RTT（往返延时）是变化的（根据网络情况）（如何测量RTT也是个问题）
  - 过短：不必要的重传
  - 过长：对段丢失时间反应慢
- **如何估计RTT（考点）**
  - SampleRTT：测量从段发出去到收到ACK的时间
    - 需要忽略重传
  - SampleRTT变化
    - 测量多个SampleRTT，求平均值形成估计值 EstimatedRTT
    - ERTT = (1 - α) * ERTT + α * SRTT（指数加权移动平均）（考虑了以前也考虑了现在）
    - α典型值：0.125
- **定时器超时时间的设置（考点）**
  - EstimatedRTT + 安全边界
  - ERTT变化大 -> 较大边界
  - 测量RTT变化值：SRTT和ERTT的差值
    - `DevRTT = (1 - β) * DevRTT + β * |SRTT - ERTT|`
    - β典型值：0.25
  - **TimeOutInterval = EstimatedRTT + 4 * DevRTT**

#### 发送方事件

- 从应用层收到数据
  - 创建Segment
  - 序列号是Segment第一个字节的编号
  - 开启计时器
  - 设置超时时间TimeOutInterval
- 超时
  - 重传引起超时的Segment（和GBN又不同了）
  - 重启定时器
- 收到ACK
  - 如果确认此前未确认的Segment
    - 更新SendBase
    - **如果窗口中还有未被确认的分组，重新启动计时器**

#### 接收方事件

- 接收到之后会等待一会，如果没有下一个段到来发送ACK

#### 快速重传

- 通过重复ACK检测分组丢失
  - sender会背靠背地发送多个分组
  - 如果某个分组丢失，可能会引发多个重复的ACK
    - 某一丢了，乱序到达，但是会不停发丢了的ACK（我要啊）
- 如果sender收到对同一数据的**三**个ACK，假定该数据之后的段已经丢失
  - 在定时器超时之前即进行重传
  - Q：为什么是3次？





### TCP流量控制

- 接收方为TCP连接分配buffer
- 上层应用可能处理buffer中数据的速度较慢
- 流量控制：发送方不会传输得太多、太快以至于淹没接收方（**buffer溢出**）
- 速度匹配机制

#### 流量控制操作

- 假定丢弃乱序得segment（虽然实际上不是这么做）

- Buffer可用空间：spare room

  - = RcvWindow

    = RcvBuffer - [LastByteRcvd - LastByteRead]

- **Reciever通过Segment得RcvWindow告诉Sender**

- Sender限制**自己已发送但是还未接收到的ACK的数据**不超过接收方空闲RcvWindow尺寸

- 接收方告诉发送方RcvWindow=0：

  - **此时还可以发一个很小的信息，使得接收方可以通过返回的ACK得指RcvWindow从而避免死锁**





### TCP连接管理

- 传输之前建立连接
- 初始化TCP变量
  - Seq.#
  - Buffer和流量控制信息

#### 建立：三次握手

- 客户端：发送SYN报文段
  - SYN段置1：表示建立一个连接
  - 传递初始序列号
  - 不携带数据
- 服务器：答复SYNACK报文段
  - 服务器分配必要的资源
  - 选择自己的初始的序列号并告知客户端
  - ACK，确认连接请求收到
- 客户端：答复ACK报文段
  - SYN不再是1了
  - 我收到了你同意建立连接的报文段
  - **可以携带数据，这里应该就可以请求数据了如果时HTTP的话**
    - 所以非持续HTTP一个对象需要两个RTT
- ![TCP连接](/img-post/2019-12-22-network-tcp/TCP连接.jpg)
  - 第二次握手：ack是我想要的，所以是client_isn+1，我想要你的第二个序列号
  - 第三次握手：seq就是client_isn+1，给你你要的，同时我也要setver_isn+1，且此时SYN为0

#### 关闭：四次

- 客户端和服务器都可以发起，一般是客户端发起
- 当客户端clientSocket.close()时会执行以下步骤
  - 客户端：发送FIN控制报文段（segment）
  - 服务器：收到FIN，回复ACK报文段
  - 服务器：关闭连接，发送FIN报文段
  - 客户端：收到FIN，恢复ACK
    - 随后进入“等待”，如果收到FIN（ACK丢包），会重新发送ACK，确保服务器关闭
  - 服务器：收到ACK，真正关闭连接





### 备注（自己）

- 三个状态：慢启动、拥塞避免、快速恢复
- **收到三个重复ACK：执行快速重传之后进入快速恢复状态**
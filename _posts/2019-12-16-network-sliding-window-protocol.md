---
layout:     post
title:      滑动窗口协议
subtitle:   Sliding Window Protocol
date:       2019-12-16
author:     Chase Gu
header-img: img/post-bg-os-metro.jpg
hide: false
catalog: true
tags:
    - 学习笔记
    - 计算机网络
---

# 3.5 滑动窗口协议

**目录：[计算机网络-课程笔记目录](https://chase-gu.github.io/2019/10/31/network-catalog/)**



### 前言

- 之前所介绍的都是基于停等操作（rdt）




### 流水线机制

- Rdt3.0：利用L/R时间发送出去，随后等待RTT时间，故性能很差
  - U sender = (L/R) / (RTT + L/R)
- 一次性发送多个，然后再等
  - U sender = (n * L/R) / (RTT + L/R)
  - 提高n倍
- 允许发送方在接收ACK之前连续发送多个分组
  - 更大的**序列号范围**
  - 发送方和/或接收方需要更大的存储空间以缓存分组





### 滑动窗口协议：实现流水线

- 英文：Sliding-window protocol
- 窗口
  - 允许使用的序列号范围
  - 窗口尺寸为N：最多有N个等待确认的消息
- 滑动窗口
  - 随着协议的运行，窗口在序列号空间向前滑动
  - <a href="/img-post/2019-12-22-network-sliding-window-protocol/滑动窗口1.png">![滑动窗口1](/img-post/2019-12-22-network-sliding-window-protocol/滑动窗口1.png)</a>
  - 绿色：已确认的**序列号**
  - 黄色：已发送、未确认的**序列号**
  - 蓝色：可用**序列号**
  - 白色：不可使用的**序列号**
- 滑动窗口协议：GBN，SR

#### GBN、SR是两种滑动窗口协议





### Go-Back-N协议

#### 发送方

- 分组头部包含k-bit序列号

- 窗口尺寸为N，再多允许N个分组未确认

  - 最小的已发送未确认：send_base
  - 最小的可用**序列号：**nextseqnum

- **采用累积确认机制** ACK(n)：表示确认到序列号n（包含n）的分组均已被正确接收

  - （还是返回最后一个嘛？或者最大一个）
  - 可能收到重复ACK

- 为空中的分组设置**计时器（timer）**：分组可能丢失

- **第n个分组发生超时事件：重传序列号大于等于n，还未收到ACK的所有分组**

  - 重发send_base到最后一个（可能没有N个，没用完）
  - 潜在的资源浪费

- **状态机（花里胡哨的，看状态机就好了）**

  <a href="/img-post/2019-12-22-network-sliding-window-protocol/GBN状态机.png">![GBN状态机](/img-post/2019-12-22-network-sliding-window-protocol/GBN状态机.png)</a>

- 收到ACK之后

  - base = 接收到的 + 1
  - 如果base == next（全部都成功接收），停止计时器
  - 否则重启计时器

- **备注**

  - **也就是说，在发送之后会不停地接收到ACK，每接收到一个就滑动一下，然后重启计时器，直到所有的分组都正确接收，关闭计时器。开始下一次发送，开启的时候又会开启计时器。**
  - **发送方可能接收到乱序的丢失的ACK，但是收到的最大的表示之前都成功接收了，如0、2、3表示0、1、2、3都被成功接收了。如果说1没有接收到，返回的ACK应该是0、0、0、0。所以ACK丢包也没关系**



#### 接收方

- ACK机制：发送拥有最高序列号的、已经被正确接收的分组ACK
  - 可能产生重复ACK
  - 只需要记住唯一的expectedSeqNum：期望序列号
- **乱序到达的分组：**
  - **直接丢弃：接收方没有缓存**
  - **然后重新确认序列号最大的、按序到达的分组**
  - 停等协议不会乱序到达



#### GBN的缺陷

- 重传时重传很多分组





### SR协议

- 英文：Selective Repeat

- 接收方对每个分组**单独进行确认**

  - 设置**缓存机制**，缓存乱序到达的分组

- 发送方只重传没收到ACK的分组

  - 为每个分组设置定时器

- 发送方的窗口

  - N个连续的序列号（同GBN）
  - 限制已发送且未确认的分组（同GBN）
  - **绿色和黄色的顺序不像GBN了**

- 接收方窗口（GBN没有）

  - 灰色：希望收到但是没有收到
  - 红色：乱序到达
  - 蓝色：可接收的序列号范围

  <a href="/img-post/2019-12-22-network-sliding-window-protocol/SR滑动窗口.png">![SR滑动窗口](/img-post/2019-12-22-network-sliding-window-protocol/SR滑动窗口.png)</a>

- **接收方和发送方窗口不是同步的**



#### 发送方

- 超时：重传**该分组**并重置**该分组**的计时器
- 收到ACK(n)在窗口内 [sendBase, sendBase + N]
  - 标记接收
  - **如果是最小的没被接收的，现在接收到了可以滑动窗口了**



#### 接收方

- 接收到在 [rcvBase, rcvBase + N -1]
  - 发送ACK
  - 乱序：缓存
  - **按序：把后面可以连上的一起交付给上层，然后滑动这么多窗口**
- 接收到在 [rcvBase - N, rcvBase - 1]，不在窗口范围（在之前）
  - 发送ACK(n)，**这意味着之前发送的ACK丢失了，发送方不知道**
- 否则，忽略掉：在右边，超过缓存



### 困境

- <a href="/img-post/2019-12-22-network-sliding-window-protocol/SR困境.png">![SR困境](/img-post/2019-12-22-network-sliding-window-protocol/SR困境.png)</a>
- **这两个序列号0的分组不是同一个分组！**
- **发送方尺寸+接收方尺寸 <= 序列号数量**，k为序列号位数，即共2^k个序列号





## 疑问

- 为什么SR中接收方是rcvBase + N - 1而发送方是sendBase + N？
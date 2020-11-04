---
layout:     post
title:      拥塞控制原理
subtitle:   Congestion Control
date:       2019-12-18 11:44
author:     Chase Gu
header-img: img/post-bg-os-metro.jpg
hide: false
catalog: true
tags:
    - 学习笔记
    - 计算机网络
---

# 3.7 拥塞控制原理

**目录：[计算机网络-课程笔记目录](https://gushichen.gitee.io/2019/10/31/network-catalog/)**



### 前言

* 表现
  * 分组丢失：路由器缓存溢出
  * 分组延迟过大：在路由器缓存中排队
* 拥塞控制 和 流量控制 的区别



### 拥塞的成因和代价

#### 场景1：无限缓存

* 两个senders和两个receivers

* 一个路由器，**无限缓存**

* 没有重传

* λin：主机AB发送的速率

  λout：主机CD接收到的速率

* 拥塞时分组延迟太大，达到最大throughput（吞吐率）

* <a href="/img-post/2019-12-18-network-congestion-control/拥塞场景1.png">![拥塞场景1](/img-post/2019-12-18-network-congestion-control/拥塞场景1.png)</a>

  * C为带宽，当发送速率超过C/2时，接收速率只能是一个恒定值，**此时达到最大吞吐率**
  * 当λ靠近C/2时，**时延无限大**

#### 场景2：丢包重传

* 一个路由器，有限buffers

* Sender重传分组

* λin：发送的速率

  λ’in：原始+重传

  λout：接收

* 情况a：Sender能够通过某种机制获知路由器buffer，有空闲才发：λin = λout（goodput）

  * 但是λin和λout**不可能大于R/2**，因为超过就没有空闲

* 情况b：丢失后才重发：λ'in > λout

  * 有效吞吐率变低，重传浪费资源

* 情况c：分组丢失和定时器超时后都重发，λ’in变得更大

* <a href="/img-post/2019-12-18-network-congestion-control/拥塞场景2.png">![拥塞场景2](/img-post/2019-12-18-network-congestion-control/拥塞场景2.png)</a>

* 拥塞代价

  * 对给定的”goodput“，需要做更多工作

#### 场景3：多跳

* 四个发送方

* 多跳
* 超时、丢失时重传
* <a href="/img-post/2019-12-18-network-congestion-control/拥塞场景3.png">![拥塞场景3](/img-post/2019-12-18-network-congestion-control/拥塞场景3.png)</a>
* 另一个代价
  * 红线和绿线竞争，如果红线的分组到到达了R2，意味着R1已经成功地处理了转发路由。但是如果该分组在R2丢弃了，**意味着在R1的操作白费了**
  * 当分组被丢弃时，任何用于该分组的“**上游**”传输能力全部被**浪费**掉
* <a href="/img-post/2019-12-18-network-congestion-control/拥塞场景3_2.png"><img src="/img-post/2019-12-18-network-congestion-control/拥塞场景3_2.png"></a>
  * 可以看到当λ'in很大的时候λout几乎为0，意味着**网络瘫痪**





### 拥塞控制的方法

* 方法一：端到端的拥塞控制
  * 网络层（路由器）不需要显式的提供支持
  * **端系统**通过观察loss，delay等网络行为判断是否发生拥塞
  * **TCP采取这种方法**
* 方法二：网络辅助的拥塞控制
  * 路由器向发送方显式地反馈网络拥塞信息
  * 简单的拥塞提示（1bit）：SNA、DECbit、TCP/IP ECN、ATM
  * 指示发送方应该采取何种速率
* Q：为什么这件事情要放在**传输层**和不放在**应用层**？

#### ATM ABR拥塞控制（非考点，没看）

* 网络辅助拥塞控制
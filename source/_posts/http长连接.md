---
title: http长连接
date: 2019-04-20 19:00:47
categories: 技术 
tags: 网络
toc: true
---

# http长连接 

#### http特点
1. 简单快速
2. 灵活
3. 无连接  *
	- 限制每次连接只处理一个请求
	- 用完即断 （后期加入）
4. 无状态  *
	- 协议对于事务处理没有记忆能力
	- 每个请求都是独立的
	- 借助 Cookie Session 完成上下文联系
5. 支持B/S(Browser/Server) 及C/S(Client/Serve)模式 

**Q 无连接的特性如何实现 长连接**

#### 网络协议分层简介
![Alt text](./1545898647502.png)

#### http 之 Keep-alive 
> 上图我们可以知道http 基于 tcp 协议。虽然http本身是无连接的，不过tcp是协议是完全面向连接具有强可靠性。故可以通过复用tcp连接的方式实现所谓的http长连接
> 即HTTP的长连接和短连接本质上是TCP长连接   和  短连接（握手说句话再见）

#### 实验模型
基于 nginx 的 keepAlive 实验
![Alt text](./1545903070532.png)

> 该场景下有两个地方可以完成http长连接配置
> 针对10.10.7.179  < - > 10.9.71.78 进行分析

#### 抓包分析
看包 说话

> 短连接 不复用
demo_no_keepalive_simple.cap
![Alt text](./1545912376653.png)

> 长连接 复用tcp连接
demo_keepalive_simple.cap
![Alt text](./1545912386780.png)


#### 小节
![Alt text](./1545912037612.png)


#### nginx http长连接 配置
> netstat -nat | grep -i "10.9.71.78:6601" 
> 观察不同配置下tcp连接的不同变化

upstream module 中
![Alt text](./1545903165154.png)

location module 中
![Alt text](./1545903182152.png)

http module 中
![Alt text](./1545903225225.png)
按需配置




####性能对比
> 统一 nginx 单worker, server为简单的hello world服务 

1. 禁用长连接
![Alt text](./1545878416799.png)
2. 启用http 长连接
![Alt text](./1545878429062.png)

性能差异 达到 70%+ （这个压测结果应该算是极限差距吧。按server耗时与传输占比来算）


#### 适用场景
长连接多用于操作频繁，点对点的通讯，而且连接数不能太多情况
WEB网站的http服务一般都用短链接
选则法则：面向"用户多少"选择

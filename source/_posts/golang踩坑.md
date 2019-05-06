---
title: golang踩坑
date: 2019-04-21 01:54:02
categories: 技术 
tags: golang
toc: true
---

# golang 线上问题汇总
##  定时器
### 现象
> 线上cpu不定时抖动
### 问题发现
> 大量go协程启用了NewTicker 而未主动关闭，而ticker对象会默认存储在一个最小堆上，todo
### 问题总结
- time.After vs time.NewTicker 区分使用

## concurrent map iteration and map write
### 现象
> 线上服务直接崩溃
### 问题发现
> 并发读写map引起
> 不是一个panic 无法被 recover
> This means that the Go runtime may detect if a map is read or modified in a goroutine, and it is also modified by another goroutine, concurrently, without synchronization.

### 问题解决
> 锁处理
> sync.Map
### 问题衍生
- 当一个 map被json encode时也会导致此问题（等同于读操作）



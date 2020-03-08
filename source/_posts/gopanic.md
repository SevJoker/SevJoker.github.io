---
title: 消失的panic
date: 2019-10-14 00:08:08
categories: 技术 
tags: golang
toc: true
---

# 消失的panic

## 现场
![Alt text](/gopanic/1.png)

![Alt text](/gopanic/2.png)
![Alt text](/gopanic/3.png)
![Alt text](/gopanic/4.png)

- 线上运行142行 报panic 
- 线下模拟不复现
- 线上模拟不复现


## 复现之旅
- 并发压测尝试后发现会出现这种case

## 问题发现之旅
### 尝试1
>第一认知 以为panic的原因是MultiDayTypeConfig 或者 MultiDayTypeConfig.d 为nil，故在程序中打印这两个值，看是否有nil，并发压测复现问题未曾发现log中有任何 nil输出

失败
### 尝试2
> 怀疑是否会有并发读写该值的情况导致nil，故将程序中所有关于该值得写入口注释（除了程序启动时），并发测试，panic依旧

失败
### 尝试3
> 因为涉及并发，考虑可能会牵扯数据竞争问题, 所以编译时加入 `-race` 参数看改值是否存在竞争情况，并发模拟，data race log比较多，过滤筛选并未发现 MultiDayTypeConfig相关的日志，而且加race参数后 协程数量不可超过8192，此时未复现问题，（log中有 不少关于 context.TripCountry 的日志）

失败
### 尝试4
> 抱着试一试的心态，当时以为会不会是 interface 在层层函数传递时候发生了意料之外的情况，将函数改写如下
![Alt text](/gopanic/5.png)
> 并发模拟 未复现问题 （amazing）

瞎猫碰到死耗子

### 尝试5
> 认为这个毫无道理可言，而且这么改写也不科学，继续挖。
> 看着data race的log，突发尝试，将context的TripCountry得修改放在并发外
> 并发模拟尝试，未复现问题

bug 修复

>But 可是并发写入string 理论上并不会crash，而且最终的nil造成原因我们还是一无所知

## 黎明前

### 意外之喜
> 偶然尝试 在`GetDayTypeByMemory` 中添加recover 尝试捕获第一现场，在recover中打印 MultiDayTypeConfig.d MultiDayTypeConfig 及 countryCode，将外层的recover 去除，目的是为了并发下打印污染，希望能直接看到panic现场内容
> 意外发现 recover中的 fmt.Println 二次panic 
> 分开打印，居然发现是打印  countryCode 的时候 panic
> What the fxxk！！！！
![Alt text](/gopanic/6.png)

### 漫漫猜测路
大概了解了一下golang string 运行时底层本质上是这么一个结构
```
type StringHeader struct {
        Data uintptr
        Len  int
}
```
> Data代表string数据的首地址，len 代表字节长度
#### 猜测1
怀疑在并发写入这个string的时候，协程1传递到函数的值 被协程2修改导致协程1 的string被gc（理论上这个时候协程1的值有句柄在函数栈上，不过数据存在堆上，到底会不会被gc，值得试一下），线下写了一个并发写入string 主动调用gc的 demo 
![Alt text](/gopanic/7.png)

并不能复现 panic
失败

#### 继续尝试
考虑是否可以打印 countryCode 的首地址，看是否有变化及差异。
代码如下所示
![Alt text](/gopanic/8.png)
复现问题后输出如下所示
`-------------------------------------data:00000000-------len:2`
 
 OMG！！！
 数据地址为空？ 长度为不为0 ？？？？
> 这个地方就解释了为啥 fmt 也会nil panic 
> 这个string 只要涉及取值操作就回 nil panic


#### 线下模拟

![Alt text](/gopanic/9.png)

并发写入struct的string再输出
模拟失败（中间尝试切换go版本保证线下线上环境一致均失败）
`ok panic`
> 中间还有个小误会，开始用1.8 跑的时候 跑了好久没panic 以为版本差异会导致panic
> 后来才发现 1.8 只不过稳定的时候长很多，panic该来还是会来的

#### 后续
##### 汇编分析
![Alt text](/gopanic/10.png)
![Alt text](/gopanic/11.png)

>Len Data 分开赋值 多条指令。并发场景并不安全
> 简化版本处理

 ![Alt text](/gopanic/12.png)
 


### 结论
并发场景赋值的不安全性


## 学习巩固知识点
- golang 的 string
- golang gc
- 堆与栈
- golang  data race
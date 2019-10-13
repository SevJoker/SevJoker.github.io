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

### 问题复原及深究
##### 单层map
```
package main
import (
    "time"
    "fmt"
    "sync"
)
var m = make(map[string]string) //wrong
var sm = &sync.Map{} // right

func main() {
    sm.Store("x","aaa")
    m["x"] = "aaa"
    sm.Store("x", m)
    go func() {
        if err1 := recover(); err1 != nil {
            return
        }
        for {
            m["x"] = "xxxx"
            // sm.Store("x","bbbb")
        }
    }()
    go func() {
        if err1 := recover(); err1 != nil {
            return
        }
        for {
            _ = m["x"]
            // v,ok := sm.Load("x")
            // sm.Store("x","bbbb")
            // fmt.Println(v,ok)
        }
    }()
    fmt.Println("----")
    time.Sleep(1 * time.Second)
}
```
> 当写map时  无论并发去读还是去写都会fatal 且无法被捕获
> 可用sync.Map 绝对安全
#### 多层map
```
package main

import (
    "time"
    "sync"
    // "fmt"
)

type AA struct {
    mu *sync.Mutex
    aa map[string]map[string]int
}

func (a *AA)set(b int) {
    a.mu.Lock()
    defer a.mu.Unlock()
    a.aa["aaa"]["aaa"] = b
}

func (a *AA)get() map[string]int {
    a.mu.Lock()
    defer a.mu.Unlock()
    return a.aa["aaa"]    
}

var sm = &sync.Map{} 

func main() {
    sm.Store("aaa",map[string]string{"aaa":"aaa"})
    // a := &AA{aa:map[string]map[string]int{"aaa":map[string]int{"aaa":1}},mu:new(sync.Mutex)} 
    go func() {
        if err1 := recover(); err1 != nil {
            return
        }
        for {
            // a.set(123)
            v,_ := sm.Load("aaa")
            v1 := v.(map[string]string)
            // fmt.Println(v,v1,ok)
            v1["aaa"] = "bbb"
        }
    }()
    go func() {
        if err1 := recover(); err1 != nil {
            return
        }
        for {
            // v := a.get()
            // _ = v["aaa"]

            // v,ok := sm.Load("x")
            // sm.Store("x","bbbb")
            // fmt.Println(v,ok)

            v,_ := sm.Load("aaa")
            v1 := v.(map[string]string)
            // _ = v1[""]
            _ = v1["aaa"]
            // fmt.Sprintln("111",v1)
            // fmt.Println(v,v1,ok)
        }
    }()

    time.Sleep(10 * time.Second)
}
```
> 多层map 无论是锁还是sync.Map都无可避免的会出现将底层的map句柄暴露给上层，继而引发同时读写错误

##### 结论
> 当你操作的map可能存在同时读写的情况下就必须加锁
> 读写操作必须都加锁
> 即 存在数据竞争的map引用不可暴露给上层
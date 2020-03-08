---
title: redis集群踩坑
date: 2019-11-17 23:20:37
categories: 技术 
tags: 架构
toc: true
---


# redis集群踩坑 

## 现场
公司MVP项目线下测试通过，线上部署时，联调会有以下报错
`crossslot keys in request don't hash to the same slot`

#### 现场相关环境依赖
##### 线上
1. redis集群版本4.0（redis-cluster架构）
2. plutus极简版本（lua操作）
###### 线下
1. redis单机版
2. plutus(lua操作redis)


## 问题追踪
> google发现大概了解了错误原因是redis-cluster集群会进行拟槽分区（slot），所有的键根据哈希函数映射到0~16383整数槽内，计算公式：slot=CRC16（key）&16383
> 每一个节点负责维护一部分槽以及槽所映射的键值数据
> redis集群执行命令是会先计算key值的所属槽区，再将请求打到指定节点由该节点操作
> 多key的情况下，redis集群节点会检测多key的hash值是否命中同一个slot，如果未能命中同一key则会爆上述错误

##### redis-cluster架构图如下
![Alt text](/redis集群踩坑/0.png)

## 代码分析
> 项目中运用了以下lua代码嵌入redis执行
```
local version = nil
version = redis.call("HGET", KEYS[1], ARGV[1]);
if version and version ~= ARGV[2] then
    return 1
else
    ARGV[2] = tonumber(ARGV[2]) + 1
end;
redis.call("HMSET", KEYS[1], unpack(ARGV));
redis.call("EXPIRE", KEYS[1], KEYS[2]);
return 0
```
##### ***注***
- 其中KEYS 代表输入的redis key数组 ；ARGV 代表输入redis 参数value数组
- 关于redis与lua的应用原理`https://juejin.im/post/5bce7e9fe51d457a772bcfc6`
    - 本质就是运用了redis的 eval 命令

这个地方输入是多参数的（key多个），redis-cluster收到key list后会进行hash取slot操作，线上联调的运行实例中未能命中同一个slot，故报错
比较尴尬的是当时我线上测试的case 刚好能命中同一个slot，未能发现问题，囧

## 修复
### 官方手段
针对多key的执行问题，官方提供的方式是可以针对多key 进行`{}`包裹处理，集群就可以统一取该标识符里的字符串取slot，这样能保证命中的slot的一致的。
`注意，redis的key里还是不要使用{}字符比较好，不然可能会导致slot分配不均匀`
相关实例操作如下
![Alt text](/redis集群踩坑/1.png)

### 我的选择
该场景将过期时延用来做第二个key，如果采取包裹处理，则还需要在lua里做字符操作，如果都取一个常量则会导致 slot 分配不均匀
这边可以将起异步化，EXPIRE 操作可以等lua脚本执行完，再单独执行。
觉得拆两步不好，不优雅 又改了一版
```
local version = nil
local expire = nil
version = redis.call("HGET", KEYS[1], ARGV[2]);
if version and version ~= ARGV[3] then
    return 1
else
    ARGV[3] = tonumber(ARGV[3]) + 1
end;
expire = ARGV[1]
table.remove(ARGV, 1)
redis.call("HMSET", KEYS[1], unpack(ARGV));
redis.call("EXPIRE", KEYS[1], expire);
return 0
```


## 延伸
极简版本是对plutus的缩减版本，redis存储这个地方工期等因素影响，并未修改，而我们线上也未出现类似错误，于是需要去研究一下codis集群设置
### codis
codis是开源的一个redis集群方案，公司做了很多优化，架构层面应该是类似的。
![Alt text](/redis集群踩坑/2.png)
`值得一提的是，codis对于lua多key操作会以第一个key为准，只会讲lua发送到第一个key所属的redis实例运行`
这里就解释了我们线上是运行正常的
需要注意的是同时操作多key时，需要对这个概率有所理解，不然会导致key操作失败

### kedis
kedis是公司自研的redis集群方案，基于redis-cluster架构封装的，与redis-cluster结构类似，外面引入了一层 proxy
`注` 后续如需升级为kedis，则需要注意多key写入的slot检测问题，kedis童鞋说是可以配置选择的

---
title: hexo踩坑
date: 2019-10-13 21:51:09
categories: 技术 
tags: blog
toc: true
---
# hexo 

> 年关将至，目标还差的很远啊~~~
> 简单整理一下这些年自己的经历画成图谱（后续完善细节）
> 嗯，发现可以做，要做的事有好多好多，多想能活500年哇
> 慢慢来吧，先把blog捡起来。。。。。
> 把 hexo 的坑统一汇总下，之前搞懂了又忘了。。。。。


## hexo 简介
**写blog的神器，看中了他兼容markdown语法且github托管**
hexo基于nodejs，将markdown可以直接编译成html，同时支持直接托管到github上，所以hexo的包管理均由npm管理。
hexo托管git需要绑定name.github.io的仓库，托管后改仓库的master即为编译后的html文件。
#### 环境保存方案
name.github.io 代码库 新建一个分支，存储所有文件，提交。
所以仓库结构就是 
-  master 托管 编译完的html文件，由**hexo命令**管理
- 新建分支托管其他所有文件，有**git命令**管理

## 命令相关
### hexo
1. 清理本地空间  `hexo clean`
2. 编译生成html `hexo g`
3. 本地服务   `hexo s`
4. 发布托管到git  `hexo d`
5. 一键部署命令(编译并发布)  `hexo g -d`

### npm包管理
1. 检测本地js包情况   `npm ls --depth 0`
2. 下载安装  `npm install packagename --save`
3. 检测项目依赖中的漏洞并自动安装 `npm audit fix`

## 报错处理
### git 提交失败
**https 改为 git模式push**

# 要写点什么


写博客也挺久了，是时候重构一下博客的内容了。

<!--more-->

## 1. 新的博客
有段时间没更新自己的博客了，最开始写博客还是刚开始学编程的时候，跟着马哥学 Linux，把每天学习的内容都整理到博客上。后来博客成了自己的"笔记本"，几乎所有学到的东西都会整理在博客上。时间久了，内容就越来越多，越来越杂。2023 年准备从北京回老家了，想规划一下自己的未来。心里做了计划，准备继续在博客上写写笔记，却突然发现不知道应该把内容放在博客哪了。就跟代码一样，时间一长博客也变得杂乱无章。所以这段时间准备把博客好好规整一番，算是对知识再一次的总结和回顾。

## 2. 博客写那些内容
根据自己的知识结构和以后想学的东西，我把博客的内容做了下面的分类:
1. Program: 编程语言，这个分类会记录所学的编程语言，以及与语言密切相关的周边知识包括:
  - Python
  - Golang
  - C
  - JavaScript
  - Rust
  - 数据结构与算法
  - 设计模式
2. Linux: 操作系统，这个分类包含了所有与操作系统相关的底层知识，包括:
  - 汇编语言，包括 x86 汇编以及 gcc asm 内联汇编
  - 程序的编译、连接和加载
  - 操作系统的实现
  - Linux 的基础使用 
  - Linux 的源码解析 
  - Linux 性能优化
  - profiling/epbf 
  - 网络的基本原理
  - 容器和云的网络架构
3. Distribution: 分布式系统，这个部分包含了分布式系统相关的理论研究
4. Architecture: 架构，这部分包含了当前流行的开源组件和数据库的原理和使用，包括:
  - k8s
  - etcd
  - mysql
  - redis
  - elasticsearch
  - kafka
  - flink
  - ....
5. Browser: 前端，这个部分包含的是前端的知识体系，包括:
  - 浏览器的实现原理
  - html/css
  - Vue
  - React
6. Hacker: 黑客，这部分会包含渗透相关的知识
6. Investment: 投资，这部分会记录学习量化投资相关的知识
7. Thinking: 记录自己的思考和感悟
8. tool: 记录平时用到的好用的工具

框架很大，内容很多，也不一定都会去学去写，先占个位置，什么时候想学了就往里面填一笔。学习应该成为一种乐趣。

## 3. 如何去学习一门语言
如何学习一门编译语言这是一个很大，也经常有大佬会谈的一个话题。我不是大佬，没法从类型系统，编译原理等各个层面去高度抽象的总结我们应该怎么学习一门语言。但是我想总结的是: 如果我们去学一门语言，我们到底要学些什么。

你可能经常也听身边的大佬说，我花了多少个小时学习了什么什么语言，这个语言怎么怎么样。如果说只是语法，花个几个小时的确就能很容易学会，但是要成为一个语言的开发者，仅仅了解一个语言的语法是远远不够的。我觉得从浅入深至少要学习下面这些内容:
1. 语法: 学会了语法代表你已经踏入了这个语言的领域
2. 惯例: 惯例包括了这个语言特有的最佳实践，最适合的设计模式等等，了解了这个语言惯例你才能写出这个语言纯正的代码，而不是把 go 写的像 java
3. 实现: 了解一个语言底层实现，才能让你成为这个语言的专家
4. 并发: 实现一个高性能的程序，离不开对并发的深入理解
5. 库: 包括标准库和第三方库，掌握常用库的用法，编码才能更加高效
6. 框架: 框架代表这个语言的技术生态和最佳实践
7. 周边工具: 包括单元测试，这个语言提供的性能分析工具等等，对这些工具的使用能从侧面反映你对这个语言的熟练程度

所以，对于博客里面出现的编程语言，我都会按照上面的分块，由浅入深分步去学习。

## 4. 内容组织
上面经过一些简单的划分，其实是将知识划分了多个相对独立的模块，在这个博客里每个知识模块都对应着一个系列。每个系列我会选择一本书或多本书作为我们学习的核心内容，在核心内容之后，我会把我看到的好的与这个知识模块密切相关的文章整理更新在核心内容之后，以此来不断完善对这个知识的理解。


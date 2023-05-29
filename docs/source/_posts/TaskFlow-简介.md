---
title: TaskFlow-简介
date: 2023-05-29 10:49:22
categories: 
- [TaskFlow]
tags: 
- TaskFlow
excerpt: 本文简单介绍了TaskFlow
---

# basic
Taskflow 是一款 C++ 下任务流程框架，可以管理 Pipeline 并执行并行工作

官方网站: [https://taskflow.github.io/](https://taskflow.github.io/)

github 地址: [https://github.com/taskflow/taskflow](https://github.com/taskflow/taskflow)

官方文档: [https://taskflow.github.io/taskflow/index.html](https://taskflow.github.io/taskflow/index.html)

使用参考: [https://www.zywvvd.com/notes/coding/cpp/taskflow/taskflow/](https://www.zywvvd.com/notes/coding/cpp/taskflow/taskflow/)

# TaskFlow里各个类介绍

## Node

Node类是TaskFlow中最重要的类，该类保存了当前节点的执行函数、前驱节点指针、后继节点指针、前继节点完成计数器。

执行函数使用std::variant 保存，std::variant 是c++17新加入的容器，提供了更安全的union, 在TaskFlow中，目前可以接受的函数类型为Static、Dynamic、Condition、MultiCondition、Module、Async、SilentAsync、cudaFlow、syclFlow、Runtime 10类。

_precede 用于向该节点添加后继节点，是构建任务拓扑关系的重要函数。

## Task

Task类是对Node类的包装，该类弱引用一个Node对象指针，可以防止使用方直接操作内部存储。

Task提供了succeed和precede来为Task间添加依赖关系，A.succeed(B)代表A依赖B，与B.precede(A)等价，实际上A的succeed方法就是调用了B的Node的_precede方法。

## Graph

Graph类负责管理Node对象的内存，为了避免赋值和拷贝引起的问题，Graph显示删除了拷贝构造函数和赋值操作函数。 

Graph中的_nodes使用vector容器维护Graph 中Node对象的指针。node_pool是一个管理Node的内存池，采用内存池可以避免频繁调用malloc/free, 减少内存碎片化，提高内存使用率，提高内存分配速度，降低分配开销。该类使用emplace_back将执行函数封装成Node节点，emplace_back方法从node_pool中管理的内存池里获取一个node，并初始化node对象，然后指向该node对象的指针会被加入到Graph的_nodes中。

## TaskFlow
TaskFlow 类继承于FlowBuilder，保存了Graph对象。提供了emplace方法在Graph中创建新节点，并使用callable参数作为现新节点的执行函数。

## Topology
Topology 用于管理Graph的拓扑结构，_bind将绑定一个Graph对象，保存了所有入口节点指针，出口个数，出口节点完成计数器。

## Executor
Executor 类是负责执行TaskFlow的类。在该类中会创建工作线程，如果没有指定工作线程数量，调用hardware_concurrency获取能并发在一个程序中的线程数量，hardware_concurrency在linux系统中一般返回CPU核心数量。
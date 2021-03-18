---
title: Java 15 http 源码解析 之 Dispatcher 
date: 2021-03-18 18:39:32
categories:
- [Java, Http, HttpServer]
tags:
- Java
- Http
- HttpServer
excerpt: 本文浅析了Java的 Dispatcher
---

Dispatcher是ServerImpl的内部类，实现了Runnable方法，作为是事件的分配器。

以下是流程图(存疑)

<style> 
.flow-chart{
    background-color: wheat;
}
</style>
```flow
st=>start: 开始
e=>end: 结束
is_finished=>condition: 服务停止
get_lolock=>operation: 获取lolock的监视器锁
check_events_is_empty=>condition: events不为空
dup_events=>operation: 将events赋值给list
clear_events=>operation: events指向新的事件链表
free_lolock=>operation: 释放lolock的监视器锁
handle_events=>operation: 处理每一个事件
调用handleEvent()，对于WriteFinishedEvent事件进行检查
检查通过则调用handle()
handle()中创建Exchange，利用executor启动线程处理Exchange
reRegister=>operation: 将已经处理完的连接放入idleConnections
selector=>operation: 轮询处理selector中的事件

st->is_finished
is_finished(no)->e
is_finished(yes)->get_lolock->check_events_is_empty
check_events_is_empty(yes)->dup_events->clear_events->free_lolock
check_events_is_empty(no)->free_lolock
free_lolock->handle_events->reRegister->selector->is_finished

```
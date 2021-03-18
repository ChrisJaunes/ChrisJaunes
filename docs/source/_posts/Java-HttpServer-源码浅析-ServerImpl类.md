---
title: Java 15 http 源码解析 之 ServerImpl 
date: 2021-03-18 20:39:32
categories:
- [Java, Http, HttpServer]
tags:
- Java
- Http
- HttpServer
excerpt: 本文浅析了Java的 ServerImpl
---

ServerImpl是http、https服务的实现。

下表为ServerImpl的成员
<style> 
.main-con .article-description table td {
    padding: 5px 10px!important;
    text-align:left!important;
}
</style>

| 属性名 | 属性类型 | 笔者的理解|
| ----- | ----- | ----- |
| protocol | String | 协议|
| https | boolean | ?? |
| executor | Executor | ?? |
| httpsConfig | HttpsConfigurator| |
| sslContext | SSLContext | |
| contexts | ContextList | | 
| address | InetSocketAddress | |
| schan | ServerSocketChannel| 用于管理socket，将在构造函数中绑定Ip地址 | 
| selector | Selector | |
| listenerKey | SelectionKey | |
| idleConnections | Set&lt;HttpConnection&gt; | 在构造函数中会获得一个线程安全的hashset， idleConnections = Collections.synchronizedSet (new HashSet&lt;HttpConnection&gt;())| 
| allConnections | Set&lt;HttpConnection&gt; | | 
| reqConnections | Set&lt;HttpConnection&gt; | | 
| rspConnections | Set&lt;HttpConnection&gt; | | 
| events | List&lt;Event&gt;| | 
| lolock | Object = new Object() | | 
| finished | volatile boolean = false| 当terminating且exchanges == 0被设置为true| 
| terminating | volatile boolean  = false | 在stop方法中设置为true|
| bound | boolean = false| | 
| started | private boolean = false | 启动标志位 | 
| time | volatile long| | 
| subticks | volatile long = 0 | | 
| ticks | volatile long | | 
| wrapper | HttpServer | | 
| CLOCK_TICK | final static int = ServerConfig.getClockTick() | | 
| IDLE_INTERVAL | final static long = ServerConfig.getIdleInterval() | | 
| MAX_IDLE_CONNECTIONS | final static int  = ServerConfig.getMaxIdleConnections()| | 
| TIMER_MILLIS | final static long = ServerConfig.getTimerMillis () | |
| MAX_REQ_TIME | final static long =getTimeMillis(ServerConfig.getMaxReqTime()) | | 
| MAX_RSP_TIME | final static long =getTimeMillis(ServerConfig.getMaxRspTime()) | | 
| timer1Enabled | final static boolean  = MAX_REQ_TIME != -1 || MAX_RSP_TIME != -1 | |
| timer | Timer | | 
| timer1 | Timer1 | | 
| logger | final Logger  | 日志管理器 |
| dispatcherThread | Thread | | 
| dispatcher | Dispatcher | | 

部分成员函数

bind (InetSocketAddress addr, int backlog) : 绑定ip地址和backlog

```java
public void bind (InetSocketAddress addr, int backlog) throws IOException {
    if (bound) {
        throw new BindException ("HttpServer already bound");
    }
    if (addr == null) {
        throw new NullPointerException ("null address");
    }
    ServerSocket socket = schan.socket();
    socket.bind (addr, backlog);
    bound = true;
}
```

setExecutor (Executor executor)：设置Executor， 必须在server没有启动的时候设置。

DefaultExecutor会直接执行run(), 这种情况下是没有启动新的线程的。


```java
public void setExecutor (Executor executor) {
    if (started) {
        throw new IllegalStateException ("server already started");
    }
    this.executor = executor;
}
private static class DefaultExecutor implements Executor {
    public void execute (Runnable task) {
        task.run();
    }
}
public Executor getExecutor () {
    return executor;
}
```

start() : 启动一个dispatcher线程，将started设置为true

```java
public void start () {
    if (!bound || started || finished) {
        throw new IllegalStateException ("server in wrong state");
    }
    if (executor == null) {
        executor = new DefaultExecutor();
    }
    dispatcherThread = new Thread(null, dispatcher, "HTTP-Dispatcher", 0, false);
    started = true;
    dispatcherThread.start();
}
```

stop(): 唤醒selector处理事件; 关闭所有的HttpConnection(对于allConnections上锁); 取消定时器; 等待dispatcherThread结束。

```java
public void stop (int delay) {
    if (delay < 0) {
        throw new IllegalArgumentException ("negative delay parameter");
    }
    terminating = true;
    try { schan.close(); } catch (IOException e) {}
    selector.wakeup();
    long latest = System.currentTimeMillis() + delay * 1000;
    while (System.currentTimeMillis() < latest) {
        delay();
        if (finished) {
            break;
        }
    }
    finished = true;
    selector.wakeup();
    synchronized (allConnections) {
        for (HttpConnection c : allConnections) {
            c.close();
        }
    }
    allConnections.clear();
    idleConnections.clear();
    timer.cancel();
    if (timer1Enabled) {
        timer1.cancel();
    }
    if (dispatcherThread != null && dispatcherThread != Thread.currentThread()) {
        try {
            dispatcherThread.join();
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            logger.log (Level.TRACE, "ServerImpl.stop: ", e);
        }
    }
}
```

createContext (String path, HttpHandler handler)： 会根据路径和处理者去获得一个HttpContextImpl对象， 并且加入到ContextList类型的对象contexts中。该方法被synchronized修饰。

removeContext (String path), removeContext (HttpContext context)： 从contexts中移除一个HttpContext类型的对象。该方法被synchronized修饰。

addEvent (Event r) ：添加事件并且唤醒selector，利用lolock上锁。
```java
void addEvent (Event r) {
    synchronized (lolock) {
        events.add (r);
        selector.wakeup();
    }
}
```

closeConnection(HttpConnection conn): 关闭连接，并且从连接池中移除
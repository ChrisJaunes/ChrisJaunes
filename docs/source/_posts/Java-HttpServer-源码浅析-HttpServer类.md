---
title: Java 15 http 源码解析
date: 2021-03-18 18:39:32
categories:
- [Java, Http, HttpServer]
tags:
- Java
- Http
- HttpServer
excerpt: 本文浅析了Java的 HttpServer、 HttpServerImpl、HttpsServer、 HttpsServerImpl
---

JDK15 中和httpserver相关的包为com.sun.net.httpserver 和 sum.net.httpserver

HttpServer 是一个抽象类， HttpServerImpl是HttpServer的实现类；抽象类HttpsServer继承于 HttpServer，而HttpsServerImpl是HttpServer的实现类。

HttpServer 采用了工厂模式，将构造函数设置为protected，并且提供了create函数创建HttpServer对象。

HttpServer提供抽象方法bind()去绑定IP地址和backlog； 提供了getAddress() 获得绑定的地址。

HttpServer提供抽象方法Start()、Stop()去启动和关闭服务。

HttpServer提供抽象方法createContext (String path, HttpHandler handler) 根据 URL path 和 handler 去构造HttpContext对象；提供抽象方法removeContext()去移除HttpContext对象

接下来分析HttpServer的实现类HttpServerImpl

HttpServerImpl 是一个代理模式（存疑）， 内部缓存了一个复杂实体ServerImpl，并且将工作委派给ServerImpl。

```java
ServerImpl server;
HttpServerImpl () throws IOException { this (new InetSocketAddress(80), 0);}
HttpServerImpl (InetSocketAddress addr, int backlog) throws IOException {
    server = new ServerImpl (this, "http", addr, backlog);
}
public void bind (InetSocketAddress addr, int backlog) throws IOException { 
    server.bind (addr, backlog);
}
public void start () { server.start();}
public void setExecutor (Executor executor) { server.setExecutor(executor);}
public Executor getExecutor () { return server.getExecutor();}
public void stop (int delay) { server.stop (delay); }
public HttpContextImpl createContext (String path, HttpHandler handler) { 
    return server.createContext (path, handler); 
}
public HttpContextImpl createContext (String path) { return server.createContext (path); }
public void removeContext (String path) throws IllegalArgumentException { 
    server.removeContext (path); 
}
public void removeContext (HttpContext context) throws IllegalArgumentException { 
    server.removeContext (context); 
}
public InetSocketAddress getAddress() { return server.getAddress();}
```

比起 HttpServer， HttpsServer 增加了成员方法 setHttpsConfigurator (HttpsConfigurator config) 和 getHttpsConfigurator();

作为HttpsServer的实现类， HttpsServerImpl同样是在内部缓存了复杂实体ServerImpl，并且将工作委派给ServerImpl。
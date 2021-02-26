---
title: Servlet的生命流程
date: 2021-02-01 00:00:20
categories:
- [Java, Servlet]
tags:
- Servlet
- Java
- Net
excerpt: 本文描述了Servlet的生命周期

---
# Servlet 生命周期

Servlet 运行在 Servlet 容器中，没有main函数

Servlet 生命周期可被定义为从创建直到毁灭的整个过程

    Servlet 初始化后调用 init () 方法。
    Servlet 调用 service() 方法来处理客户端的请求。
    Servlet 销毁前调用 destroy() 方法。
    最后，Servlet 是由 JVM 的垃圾回收器进行垃圾回收的。

![avatar](https://www.runoob.com/wp-content/uploads/2014/07/Servlet-LifeCycle.jpg)

## 1. init方法：

init 方法被设计成只调用一次。它在第一次创建 Servlet 时被调用，在后续每次用户请求时不再调用。因此，它是用于一次性初始化。

&emsp;Servlet\[interface\] 中声明了方法init()。

&emsp;GenericServlet\[class\] 实现了接口Servlet实现了方法init(), 并且重载了函数init(ServletConfig)。

&emsp;HttpServlet\[class\] 继承于GenericServlet, 没有重写init()。

&emsp;我们编写类 ExpServlet 继承于 HttpServlet\[class\], 重写了init、init(ServletConfig)。

实验结果：

&emsp;在重写了init(ServletConfig)将调用init(ServletConfig), 否则调用init()。

## 2. service方法

&emsp;service()方法是执行实际任务的主要方法。Servlet 容器调用 service() 方法来处理来自客户端的请求，并把格式化的响应写回给客户端。

&emsp;每次服务器接收到一个 Servlet 请求时，服务器会产生一个新的线程并调用服务。service() 方法检查 HTTP 请求类型（GET、POST、PUT、DELETE 等），并在适当的时候调用 doGet、doPost、doPut，doDelete 等方法。

&emsp;Servlet\[interface\] 中声明了方法 service(ServletRequest req, ServletResponse res)。

&emsp;GenericServlet\[class\] 没有了实现接口Servlet的方法service(ServletRequest req, ServletResponse res)。

&emsp;HttpServlet\[class\] 继承于GenericServlet, 实现了service(ServletRequest req, ServletResponse res), 并且重载了service(HttpServletRequest, HttpServletResponse)。service(ServletRequest req, ServletResponse res)主要是将req由ServletRequest类型强转为HttpServletRequest， res由ServletResponse类型强转为HttpServletResponse。然后调用 service(HttpServletRequest, HttpServletResponse), 该函数检查Http类型，调用对应的方法

```java
public void service(ServletRequest req, ServletResponse res) throws ServletException, IOException {
    HttpServletRequest request;
    HttpServletResponse response;
    try {
        request = (HttpServletRequest)req;
        response = (HttpServletResponse)res;
    } catch (ClassCastException var6) {
        throw new ServletException("non-HTTP request or response");
    }

    this.service(request, response);
}
```

&emsp;我们编写类 ExpServlet 继承于 HttpServlet\[class\], 重写了service(ServletRequest req, ServletResponse res)、service(HttpServletRequest, HttpServletResponse)。

实验结果：

&emsp;每次请求打开一个新的线程，都会调用service(ServletRequest req, ServletResponse res)，然后 service(ServletRequest req, ServletResponse res)调用service(HttpServletRequest, HttpServletResponse)运行。

## 3. destroy方法

&emsp;destroy() 方法只会被调用一次，在 Servlet 生命周期结束时被调用。destroy() 方法可以让您的 Servlet 关闭数据库连接、停止后台线程、把 Cookie 列表或点击计数器写入到磁盘，并执行其他类似的清理活动。

实验结果:

&emsp;destory在结束的时候被主线程调用。

## 测试代码

```java
package com.chrisjaunes.base_exp;

import javax.servlet.ServletException;
import javax.servlet.ServletRequest;
import javax.servlet.ServletResponse;
import javax.servlet.annotation.WebServlet;
import javax.servlet.http.HttpServlet;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;
import java.util.logging.Logger;

/**
 * 本实验用于了解 servlet 的流程
 * @author ChrisJaunes
 */
@WebServlet(name = "Exp1Servlet", urlPatterns = "/Exp1")
public class Exp1Servlet extends HttpServlet {
    private final Logger logger = Logger.getLogger("Exp1Servlet");
    @Override
    public void init() throws ServletException {
        logger.info("init " + Thread.currentThread());

        super.init();
    }
//    @Override
//    public void init(ServletConfig config) throws ServletException{
//        logger.info("init_ServletConfig " + Thread.currentThread());
//        super.init();
//    }
    @Override
    public void service(ServletRequest req, ServletResponse res) throws ServletException, IOException {
        logger.info("service_ServletRequest_ServletResponse "  + Thread.currentThread());
        super.service(req, res);
    }
    @Override
    public void service(HttpServletRequest req, HttpServletResponse res) throws  ServletException, IOException{
        logger.info("service_HttpServletRequest_HttpServletResponse "  + Thread.currentThread());
        super.service(req, res);
    }

    @Override
    protected void doPost(HttpServletRequest request, HttpServletResponse response) {
        logger.info("doPost_HttpServletRequest_HttpServletResponse " + Thread.currentThread());

    }

    @Override
    protected void doGet(HttpServletRequest request, HttpServletResponse response) {
        logger.info("doGet_HttpServletRequest_HttpServletResponse "  + Thread.currentThread());
    }

    @Override
    public void destroy(){
        logger.info("destroy "  + Thread.currentThread());
        super.destroy();
    }
}

```

测试结果

```shell
27-Feb-2021 01:01:25.106 信息 [http-nio-8080-exec-7] com.chrisjaunes.base_exp.Exp1Servlet.init init Thread[http-nio-8080-exec-7,5,main]

27-Feb-2021 01:01:25.108 信息 [http-nio-8080-exec-7] com.chrisjaunes.base_exp.Exp1Servlet.service service_ServletRequest_ServletResponse Thread[http-nio-8080-exec-7,5,main]

27-Feb-2021 01:01:25.109 信息 [http-nio-8080-exec-7] com.chrisjaunes.base_exp.Exp1Servlet.service service_HttpServletRequest_HttpServletResponse Thread[http-nio-8080-exec-7,5,main]

27-Feb-2021 01:01:25.109 信息 [http-nio-8080-exec-7] com.chrisjaunes.base_exp.Exp1Servlet.doGet doGet_HttpServletRequest_HttpServletResponse Thread[http-nio-8080-exec-7,5,main]

27-Feb-2021 01:01:26.968 信息 [http-nio-8080-exec-8] com.chrisjaunes.base_exp.Exp1Servlet.service service_ServletRequest_ServletResponse Thread[http-nio-8080-exec-8,5,main]

27-Feb-2021 01:01:26.968 信息 [http-nio-8080-exec-8] com.chrisjaunes.base_exp.Exp1Servlet.service service_HttpServletRequest_HttpServletResponse Thread[http-nio-8080-exec-8,5,main]

27-Feb-2021 01:01:26.969 信息 [http-nio-8080-exec-8] com.chrisjaunes.base_exp.Exp1Servlet.doGet doGet_HttpServletRequest_HttpServletResponse Thread[http-nio-8080-exec-8,5,main]

27-Feb-2021 01:04:27.428 信息 [main] com.chrisjaunes.base_exp.Exp1Servlet.destroy destroy Thread[main,5,main]

```
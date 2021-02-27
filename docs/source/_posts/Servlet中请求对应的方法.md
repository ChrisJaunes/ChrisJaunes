---
title: Servlet中请求对应的方法
date: 2021-02-27 01:24:11
categories:
- [Java, Servlet]
tags:
- Servlet
- Java
- Net
excerpt: 本文描述了Servlet的生命周期

---

## GET 请求

GET 请求来自于一个 URL 的正常请求，或者来自于一个未指定 METHOD 的 HTML 表单，它由 doGet() 方法处理。

在 Service 中:

&emsp;检查之前是否访问过，如果没有访问过，调用doGet

&emsp;否则获取If-Modified-Since属性。

&emsp;注： If-Modified-Since是标准的HTTP请求头标签，在发送HTTP请求时，把浏览器端缓存页面的最后修改时间一起发到服务器去，服务器会把这个时间与服务器上实际文件的最后修改时间进行比较。如果时间一致，那么返回HTTP状态码304（不返回文件内容），客户端接到之后，就直接把本地缓存文件显示到浏览器中。如果时间不一致，就返回HTTP状态码200和新的文件内容，客户端接到之后，会丢弃旧文件，把新文件缓存起来，并显示到浏览器中。
    
&emsp;参考链接1：https://www.cnblogs.com/moxiaotao/p/9670109.html

&emsp;参考链接2：https://blog.csdn.net/weixin_34023863/article/details/92065577

&emsp;HttpServlet假装实现了这个功能，其实并没有。因为getLastModified方法始终返回-1。我们可以同重写该方法去覆盖HttpServlet的getLastModified。

```java
protected void service(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
    String method = req.getMethod();
    long lastModified;
    if (method.equals("GET")) {
        lastModified = this.getLastModified(req);
        if (lastModified == -1L) {
            this.doGet(req, resp);
        } else {
            long ifModifiedSince = req.getDateHeader("If-Modified-Since");
            if (ifModifiedSince < lastModified) {
                this.maybeSetLastModified(resp, lastModified);
                this.doGet(req, resp);
            } else {
                resp.setStatus(304);
            }
        }
    } 
    ...
}

protected long getLastModified(HttpServletRequest req) {
    return -1L;
}
```

实验：

&emsp;重写getLastModified方法, 然后利用http_request、postman、burp等工具去构造请求

```java
@Override
protected long getLastModified(HttpServletRequest req) {
    Calendar calendar = Calendar.getInstance();
    calendar.set(2021, Calendar.FEBRUARY, 1, 1, 1, 1);
    Date date = calendar.getTime();
    return date.getTime();
}
```

&emsp;构造请求

```http
GET /basic/Exp2 HTTP/1.1
Host: localhost:8080
```

```http
GET /basic/Exp2 HTTP/1.1
Host: localhost:8080
If-Modified-Since: Fri, 26 Feb 2022 18:00:01 GMT
```

实验结果：

&emsp;第一条请求返回200 OK,  第二条请求返回304 Not Modified


## POST 请求

POST 请求通常来自于一个特别指定了 METHOD 为 POST 的 HTML 表单、或者客户端需要传输实体，它由 doPost() 方法处理。

可以重写doPost方法去解决请求。

实验：

```java
@Override
protected void doPost(HttpServletRequest request, HttpServletResponse response) {
    logger.info("doPost_HttpServletRequest_HttpServletResponse single parameter:" + request.getParameter("test_single_parameter"));
    for(String mp : request.getParameterValues("test_multi_parameter")) {
        logger.info("doPost_HttpServletRequest_HttpServletResponse multi parameter:" + mp);
    }
    Enumeration<String> ap = request.getParameterNames();
    while (ap.hasMoreElements()) {
        logger.info("doPost_HttpServletRequest_HttpServletResponse :" + ap.nextElement());
    }
}
```

构造请求：

```http
POST /basic/Exp2 HTTP/1.1
Host: localhost:8080
Content-Type: application/x-www-form-urlencoded
Cache-Control: no-cache
Postman-Token: f9b026c5-5ea6-0af6-30d8-f4133f80b97a

test_single_parameter=1&test_multi_parameter=2&test_multi_parameter=3&test_other_parameter=4
```

实验结果：

```shell
doPost_HttpServletRequest_HttpServletResponse single parameter:1
doPost_HttpServletRequest_HttpServletResponse multi parameter:2
doPost_HttpServletRequest_HttpServletResponse multi parameter:3
doPost_HttpServletRequest_HttpServletResponse: test_single_parameter
doPost_HttpServletRequest_HttpServletResponse: test_multi_parameter
doPost_HttpServletRequest_HttpServletResponse: test_other_parameter
```

实验结果

## HEAD 请求

检查一个对象是否存在, HttpServlet实现了该方法，此请求通常无需复写。

```java
protected void doHead(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
    NoBodyResponse response = new NoBodyResponse(resp);
    this.doGet(req, response);
    response.setContentLength();
}
```

## PUT 请求

向Web服务器发送数据并存储在Web服务器内部

## DELETE 请求

从Web服务器上删除一个文件
 
## OPTIONS 请求

查询Web服务器的性能

## TRACE 请求

跟踪到服务器的路径

## HttpServlet 中 doGet、doPost、doPut、doDelete的默认行为

注：400:告诉客户端它发送了一条异常请求。400页面是当用户在打开网页时，返回给用户界面带有400提示符的页面。其含义是你访问的页面域名不存在或者请求错误。主要分为两种。1、语义有误，当前请求无法被服务器理解。除非进行修改，否则客户端不应该重复提交这个请求。2、请求参数有误。

注：405：请求行中指定的请求方法不能被用于请求相应的资源。该响应必须返回一个Allow 头信息用以表示出当前资源能够接受的请求方法的列表。

```java

protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
    String protocol = req.getProtocol();
    String msg = lStrings.getString("http.method_get_not_supported");
    if (protocol.endsWith("1.1")) {
        resp.sendError(405, msg);
    } else {
        resp.sendError(400, msg);
    }
}

```

出于以上环节，重写doGet、doPost、doPut、doDelete的时候不能调用super.doGet()、super.doPost()、super.doPut()、super.doPut()、super.doDelete()
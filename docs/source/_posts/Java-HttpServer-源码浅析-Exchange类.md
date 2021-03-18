---
title: Java-HttpServer-源码浅析 之 Exchange
date: 2021-03-19 00:58:23
categories:
- [Java, Http, HttpServer]
tags:
- Java
- Http
- HttpServer
excerpt: 本文浅析了Java的 Exchange
---

run() :

以下是流程图(存疑)

<style> 
.flow-chart{
    background-color: wheat;
}
</style>
```flow
st=>start: 开始
e=>end: 结束
init=>operation: context = connection.getHttpContext()
check_context=>condition: context 不为空
get_rawio_by_connection=>operation: 从connection 中获取 rawin 和 rawout
check_https=>condition: 检查 是否是https连接
get_rawio_by_https=>operation: 构造SSLStreams流 从SSLStreams流中获取 rawin 和 rawout
get_rawio_by_http=>operation: 构造 BufferedInputStream 和 Request.WriteStream流并获取 rawin 和 rawout
create_request=>operation: 根据rawin和rawout构造request对象
get_method_uriStr_version=>operation: 获取request的请求行并且解析method、url、version
find_httpContextImpl=>operation: 获取HttpContextImpl对象
setContext=>operation: 为connection设置HttpContext
create_ExchangeImpl=>operation: 构造ExchangeImpl对象
set_connection_ka=>operation: 根据http版本判断是否需要设置keep-alive
set_parameters=>operation: 根据是否是新连接为connection设置参数
create_filter_chain=>operation: 责任链模式，构造链并且处理request

st->init->check_context
check_context(yes)->get_rawio_by_connection->create_request
check_context(no)->check_https
check_https(yes)->get_rawio_by_https->create_request
check_https(no)->get_rawio_by_http->create_request
create_request->get_method_uriStr_version
get_method_uriStr_version->find_httpContextImpl
find_httpContextImpl->setContext
setContext->create_ExchangeImpl->set_connection_ka
set_connection_ka->set_parameters->create_filter_chain->e
```

LinkHandler: 用于拼接两条Filter链

reject：拒绝连接

sendReply：发送回复
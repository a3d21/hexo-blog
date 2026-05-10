---
title: Go反向代理避坑指南
date: 2026-05-10 11:39:18
tags: 编程
---

Go原生自带了`httputil.ReverseProxy`，可以直接用来做业务网关，很方便。
<!-- more -->
我有个业务需要按用户UserID做流量灰度，并支持实时切换。如果引入第三方网关组件实现会很复杂，所以决定使用`httputil.ReverseProxy`自己实现一个。
实测性能不错，只可以满足大多数业务场景。

> 测试报告
> - **单次请求耗时 (ns/op)**: `~84,586 ns`（约 84 微秒）。在单核单线程下，理论 QPS 可达 **11,800+**。
> - **内存分配 (B/op)**: `~45,711 Bytes`（约 45 KB）。主要来自 Gin 上下文及 HTTP 对象的封装和传递。
> - **对象分配次数 (allocs/op)**: `132 次`。


下面是我遇到的一些坑，记录一下。

## 1 Client断连`panic`处理

默认Client断连时会panic，如果处理不当会导致服务退出。

```go
// source code: net/http/httputil/reverseproxy.go
err = p.copyResponse(rw, res.Body, p.flushInterval(res))
if err != nil {
    defer res.Body.Close()
    // Since we're streaming the response, if we run into an error all we can do
    // is abort the request. Issue 23643: ReverseProxy should use ErrAbortHandler
    // on read error while copying body.
    if !shouldPanicOnCopyError(req) {
        p.logf("suppressing panic for copyResponse error in test; copy error: %v", err)
        return
    }
    panic(http.ErrAbortHandler)
}
```

解决方法，加一层recover拦截

```go
defer func() {
    if err := recover(); err != nil {
        if err == http.ErrAbortHandler {
            return
        }

        panic(err)
    }
}()
proxy.ServeHTTP(c.Writer, c.Request)
```

## 2 异常处理
`context.Canceled`和`io.EOF`是可忽略的，可以设置`ErrorHandler`跳过。

```go
rp := &httputil.ReverseProxy{
    Director: func(req *http.Request) {
        // todo
    },
    Transport: transport,
    ErrorHandler: func(writer http.ResponseWriter, request *http.Request, err error) {
        if errors.Is(err, context.Canceled) {
            return
        }

        if errors.Is(err, io.EOF) {
            return
        }

        log.Errorf(request.Context(), "proxy error: %v", err)
    },
}
```

## 3 连接复用 
`ReverseProxy`的`Transport`会维护连接池，做连接复用。应该尽量复用`ReverseProxy`和`Transport`实例，避免每次代理新建连接池。

```go
var (
	transport = &http.Transport{
		Proxy: http.ProxyFromEnvironment,

		DialContext: (&net.Dialer{
			Timeout:   3 * time.Second,  // 建连超时
			KeepAlive: 30 * time.Second, // TCP keepalive
		}).DialContext,

		ForceAttemptHTTP2: true, // 开启 HTTP/2

		MaxIdleConns:        5000, // 全局空闲连接池
		MaxIdleConnsPerHost: 500,  // 每个后端连接池（关键）
		MaxConnsPerHost:     1000, // 每个后端最大连接数（防止打爆）

		IdleConnTimeout:       120 * time.Second, // 空闲连接回收
		TLSHandshakeTimeout:   5 * time.Second,
		ExpectContinueTimeout: 1 * time.Second,

		ResponseHeaderTimeout: 60 * time.Second, // 后端响应头超时

		DisableKeepAlives: false, // 一定要开启连接复用
	}
)

rp := &httputil.ReverseProxy{
    Transport: transport,   // 复用transport
}
```

## 4 避免重复CORS错误

如果代理服务器做了CORS，转发时需要移除`Origin` Header，防止CORS异常。
说明：这场景是业务服务A转发请求到服务B，服务A做了CORS。一般网关不做CORS，不会遇到这问题。

```go
proxy := httputil.ReverseProxy{
    Director: func(req *http.Request) {
        req.Header.Del("Origin")
        req.Header.Add("X-Forwarded-Host", req.Host)
        // todo
    },
}
```
---
title: 一个轻量的Go Web框架
date: 2026-05-02 23:49:33
tags: 编程
---

## 0 背景
grpc + grpc-gateway是个很好的设计，优点有
1. 独立定义的proto，方便跨端沟通与维护
2. 生态成熟，有很多proto编译插件，可以增加校验、openapi spec生成等
3. grpc + http 双栈设计，内部RPC走grpc协议，外接请求走http，只需要实现一次
4. grpc 服务实现标准化，`func XXX(ctx, in)(out, err)`的格式很容易解析和扩展中间件
5. error 定义标准化

如果是从零开始一个新项目，我会优先选择grpc。我们线上运行着go gin的web项目，转成grpc学习成本偏高。所以我思考，怎么能在已有项目中适当改造，获得grpc栈的优点呢？


## 1 标准化服务方法签名
标准化服务方法签名是grpc简单与易扩展基础。所以改造的第一步是让go服务的接口方法标准化。
```go
type Fn[IN, OUT any] func(context.Context, *IN) (OUT, error)
```

然后，通过辅助方法将 `HandlerFn` 转换成 `gin.HandlerFunc`

```go
func F2G[IN, OUT any](fn Fn[IN, OUT]) gin.HandlerFunc {
    // 参数绑定
    // 统一err处理
    // 统一返回包装
}
```

这样，就拥有一个简化版grpc-gateway。

## 2 接口路由

grpc-gateway 的接口使用proto定义，用以上方法，需要手动注册路由，而注册路由的代码可以看做接口定义文件。

```go
//// 用户信息
v1.GET("/user/me", g.F2G(userSrv.GetUserInfo))  //获取我的信息
v1.PUT("/user/me", g.F2G(userSrv.UpdateMe))     //更新服务信息
v1.POST("/user/register", g.F2G(userSrv.Register))  //注册
```

采用这种方式，得到了意外的好处：http接口可以与接口实现分离，让接口实现可复用。


## 3 openapi spec生成

grpc最吸引我的一点是：可以proto生成openapi spec，方便调试和沟通。
特别是在AI编程背景下，当接口沟通成为前后端沟通最大痛点时，这个特点格外显眼。

go也有openapi spec生成方案：swag。
我不喜欢它，因为
1. 侵入性强。为了生成文档，需要手写一堆`@doc`注解
2. swag 无法直接识别以上`F2G`的接口

我认为代码是最好的文档，代码已经包含生成openapi spec需要的所有内容。
参考proto生成openapi spec的生成方式，我编写了一个go源码分析工具(`f2gspec`)，它能
1. 通过代码扫描`F2G()`调用，生成接口列表
2. 从`F2G()`的调用处往上递归，获取接口完整URL+Method
3. 分析`F2G()`参数，解析生成 `request`和`response` scheme
4. 将代码的注释解析成openapi spec注释
5. 扩展分组功能，`////` 开头的注释当成分组处理

## 4 如何发布openapi spec
完成上面的`f2gspec`工具后，效果惊艳到我。它能几秒钟分析出200多个接口的spec。
那么，下一个问题是：如何发布它?

好的openapi spec应该是实时的，它应该和服务一起发布。
所以：
1. 在服务build过程中，解析spec保存为 `openapi.json`
2. 服务增加 `GET /openapi.json`接口，查实时spec
   
这样实现spec和服务一同发布。
为了分辨spec更新与否，我在spec title上加了datetime。

## 5 基于`F2G`+`f2gspec`如何AI开发

开发流程
1. 优先定义接口。这是通常协同开发的第一步，有了`F2G`+`f2gspec`工具后，可以直接用go代码写接口签名，用CICD生成接口文档
2. 并行开发
   1. 后端通过接口签名和注释AI生成代码和测试
   2. 前端通过`openapi.json`生成调用方法
3. 调试。使用swagger-ui直接查看和调试
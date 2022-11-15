---
title: Go配置热更——动态绑定
date: 2022-11-15 19:39:45
tags: 编程
---

当我们使用配置中心时，我们希望配置热更。一般的实现方式是，读取配置初始化并增加变更监听。这种方式可以实现，但需要我们维护配置变更逻辑，不友好。

在大多数场景下，我们想要的其实是一个`func()`，每次调用返回最新的结构化配置。Go 1.18增加了泛型，可以很优雅地解决配置热更问题。

我以nacos为例，写了一个**动态绑定库**[bind_nacos_cfg](https://github.com/a3d21/bind_nacos_cfg)。

接口如下

```go
// Supplier 获取最新配置
type Supplier[T any] func() T
func (sup Supplier[T]) Get() T {
	return sup()
}

// MustBind 绑定某配置项
func MustBind[T any](cli config_client.IConfigClient, dataID, group string, typ T) Supplier[T] {}
```


使用Demo
```go
type ConfigStruct struct {
	Name string
	City string
}

func main() {
	var cli config_client.IConfigClient // TODO 初始化
	dataID := "abc"
	group := "def"
	var GetConf = bind_nacos_cfg.MustBind(cli, dataID, group, &ConfigStruct{})
	// 也支持原生结构类型
	// var GetConf = bind_nacos_cfg.MustBind(cli, dataID, group, StructConfig{})
	// var GetConf = bind_nacos_cfg.MustBind(cli, dataID, group, map[string]string{})
	// var GetConf = bind_nacos_cfg.MustBind(cli, dataID, group, []string{})

    // c.(type) == *ConfigStruct
	c := GetConf() // 获取最新配置
	c = GetConf()
	c = GetConf() // 配置变更自动监听更新

	fmt.Println(c)
}
```

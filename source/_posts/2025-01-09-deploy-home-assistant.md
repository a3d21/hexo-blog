---
title: "[智能家居]HomeAssistant折腾笔记"
date: 2025-01-09 17:15:23
tags: [HA, 智能家居]
---

最近开始折腾智能家居。试用过米家、涂鸦和homekit后, 觉得Home Assistant(HA)方案最符合我的需求。
这个系列整理HA的部署笔记,作为总结,方便以后查阅。
<!-- more -->


## Why HA？
HA的优点
- 跨平台支持。可以接入米家、homekit等不同平台
- 强大编程能力。自带自动化引擎很强大,支持并行、延时、条件控制；除此之外还可以接入node-red图形化编程
- 本地化。本地计算, 一般响应更快, 对网络依赖低
- 强大社区支持。插件更新频繁, 遇到问题很容易找到解决方案

缺点是, 有一定学习成本。
如果喜欢折腾, 就直接上HA吧。用其它平台或多或少都会遇的问题, 在HA常常可以很容易地解决。
比如, 米家极客版, 为了实现寄存器功能, 需要购买硬件“米极桥”, 或者用虚拟事件曲线实现。但在HA, 定义个虚拟开关即可解决。


## HA部署
HA官方提供有[四种部署方式](https://www.home-assistant.io/installation), 我推荐使用**Container**方式。
Container相对精简, 常用add-ons也能通过插件方式实现, 备份和迁移更加方便。

HA对硬件要求极低（建议1GB以上内存和16GB以上存储空间）, 适合跑在软路由、nas或armbian盒子。
我以armbian盒子为例演示。

```sh
# armbian安装docker
apt install docker.io
# 运行ha容器, 启动用使用:8123端口访问服务
docker run -itd --net=host --restart always --name="hass" -v /data/hass/config:/config homeassistant/home-assistant:latest
```

## 安装扩展
### 安装HACS
[HACS](https://github.com/hacs/integration)是个插件库, 很方便安装第三方插件。
下载最新release包, 解压到`/data/hass/config/custom_components/hacs`, 重启hass。
重启完成后, 依次 设置-增加集成-搜hacs, 按引导配置即可。


### 安装米家扩展
截至2025-01-09, 米家的集成方式有3种
- [ha_xiaomi_home](https://github.com/XiaoMi/ha_xiaomi_home),官方集成, 功能不完善, 支持部分wifi设备本地化控制, 不支持蓝牙设备本地化控制, 正在快速迭代更新中
- [hass-xiaomi-miot](https://github.com/al-one/hass-xiaomi-miot),开源方案, 同官方支持部分wifi本地化控制, 但状态更新有延迟
- [XiaomiGateway3](https://github.com/AlexxIT/XiaomiGateway3), 开源方案, 支持蓝牙设备本地化控制, 需要购买特定版本网关

目前最稳定的方案是hass-xiaomi-miot + XiaomiGateway3。XiaomiGateway3用于接入蓝牙设备, 其它设备使用hass-xiaomi-miot接入, 实现大部分设备本地化控制。状态更新延迟问题不太重要。

安装方法。从HACS页面搜相应插件安装, 重启HA。在“集成”页面增加插件集成, 按引导配置即可。
建议使用*Include*方式手动添加设备, 避免导入太多不需要接入的设备和实体。


### 安装node-red(可选)
node-red是个强大的add-on, 可以实现图形化通用编程扩展。类似于米家极客版, 但功能更强大。支持第三方js库, 可以实现推微信消息等功能。
Container版本HA不支持add-on功能, 但可以手动安装。原理是, docker运行node-red服务, 配置打通两个系统。

首先, HA通过HACS安装“Node-RED Companion”插件, 方法与安装米家插件相同。
然后运行node-red服务
```
# 1880端口访问服务
docker run --restart=always -e TZ="Asia/Shanghai" -d --net=host -v /data/nodered_data:/data --name nodered nodered/node-red
```

再访问node-red管理页面, 安装`node-red-contrib-home-assistant-websocket`节点, 按提示配置即可。

## 接入Apple Home

使用HomeBridge接入Apple Home, 可以实现用Siri操作HA上设备。
HA自带HomeBridge集成。在集成页面增加HomeBridge, 选择要接入的设备。用iphone扫二维码接入即可。


## 自动化案例
虽然node-red很强大, 但大部分场景HA自动化已够用, 而且少一层通信, 时延更低。所以我更建议HA自动化。

常用自动化分享
- 离家。关闭空调、灯, 打开监控
- 回家。由门锁或传感器自动触发, 关监控, 天黑自动开灯, 音箱播放欢迎信息
- 晚安。关灯、关客厅空调, 开监控
- 早安。时间+人体传感器自动触发, 关监控、音箱播放早安信息&天气&新闻
- 夜间开关灯。时间、晚安状态配合人体传感器自动触发
- 潮湿自动抽湿, 干燥自动加湿。



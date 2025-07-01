---
title: "[智能家居]跨平台智能家居——红外方案"
date: 2025-07-01 18:38:39
tags: [HA, 智能家居]
---

智能家居平台有很多：小米、华为、涂鸦、aqara...，无法互通，很不方便。尤其是当买了多个平台设备时，每次操作都得想想用哪个平台APP，呼叫哪个语音助手去执行。

但几乎所有平台都支持红外(IR)，而且红外是一种本地协议，可以用红外来本地化打通不同平台。
<!-- more -->

## 1 控制流程
```plantuml
rectangle "A平台设备\n(带红外音箱)" as a
rectangle "IR接收器" as ir
rectangle "Home Assistant\n(控制中心)" as ha
rectangle "B平台设备" as b

a -> ir: "发送自定义IR命令"
ir -> ha: "传递"
ha -> b: "控制"
```

第一步，就是DIY一个IR接收器，用来接收**自定义IR编码**（建议使用NEC码）。

## 2 使用esphome制作IR接收器

使用esphome可以很容易编译固件，制作DIY设备，几乎0代码。
硬件可以使用 esp32/esp8266 + 红外接收头。可以买模块自己焊接，也买成品模块或者兼容设备，如ct30w。可参考 [刷机教程](https://www.bilibili.com/opus/967982171394408465) 进行刷机。

我使用的是ct30w，以下是ct30w对应的配置文件。可以用`esphome compile`编译，也可以直接下载[预编译固件](/assets/ir2ha.bin)。刷机后，连接AP配置wifi信息使用。
```yaml
esphome:
  name: ir2ha

esp8266:
  board: esp01_1m

# Enable logging
logger:
  # level: DEBUG

web_server:
  port: 80

# Enable Home Assistant API
api:
  actions:
    - action: send_raw_ir
      variables:
        raw_data: 'int[]'
      then:
        - remote_transmitter.transmit_raw:
              code: !lambda 'return raw_data;' 
              carrier_frequency: 38kHz
    - action: send_nec_ir
      variables:
        address: int
        command: int
      then:
        - remote_transmitter.transmit_nec:
            address: !lambda 'return address;' #0x1234
            command: !lambda 'return command;'  #0x78AB

ota:
  - platform: esphome
    password: ""

wifi:
  ssid: "1111"
  password: "11111111"

  # Enable fallback hotspot (captive portal) in case wifi connection fails
  ap:
    ssid: "ir2ha Fallback Hotspot"  # 通电后登录配置WIFI
    password: "66668888"

captive_portal:

output:
  - platform: gpio
    id: LEDD5
    pin:
      number: GPIO15

remote_receiver:
  - pin: 
      number: GPIO5
      inverted: True
    id: IR_receiver
    tolerance: 55%
    dump: [nec]
    on_nec: # 使用nec码
      then:
        - homeassistant.event:
            event: esphome.ir_received
            data:
              address: !lambda 'return x.address;'
              command: !lambda 'return x.command;'
              protocol: "nec"
              device: "ir2ha"

remote_transmitter:
  - pin: GPIO14
    carrier_duty_percent: 50%
    id: IR_transmitter
    on_transmit:
      then:
        - output.turn_on: LEDD5
    on_complete:
      then:
        - output.turn_off: LEDD5


button:         
  - platform: template
    name: "media_fan_power" # 测试用
    on_press:
      then:
        - remote_transmitter.transmit_raw:
            code: [9018, -4395, 634, -1594, 635, -480, 633, -481, 634, -480, 634, -481, 634, -480, 634, -482, 633, -480, 634, -481, 635, -1593, 634, -1595, 634, -1595, 633, -1595, 634, -1594, 634, -1595, 633, -1599, 631, -1594, 633, -1595, 634, -481, 634,
                 -480, 634, -480, 634, -480, 634, -480, 634, -480, 634, -480, 634, -481, 634, -1594, 634, -1594, 634, -1596, 633, -1594, 634, -1595, 634, -1594, 634, -1595, 634, -1594, 634, -480, 635, -480, 634, -480, 634, -481, 634, -479, 634, -481, 634, -480, 634,
                 -481, 633, -1594, 635, -1594, 635, -1594, 634, -1594, 634, -1595, 634, -1594, 634]
            carrier_frequency: 38kHz

  - platform: template
    name: "media_fan_speed"
    on_press:
      then:
        - remote_transmitter.transmit_raw:
            code: [9017, -4392, 635, -1595, 634, -480, 634, -480, 634, -481, 633, -481, 634, -480, 634, -481, 634, -480, 634, -480, 634, -1595, 634, -1595, 634, -1595, 635, -1595, 634, -1595, 634, -1595, 635, -1595, 634, -1595, 634, -1595, 634, -1595, 634,
                  -480, 634, -481, 634, -481, 634, -480, 633, -481, 634, -481, 634, -480, 634, -481, 633, -1595, 634, -1595, 634, -1595, 634, -1595, 634, -1595, 634, -1594, 634, -1595, 634, -1595, 634, -480, 634, -480, 634, -480, 635, -480, 634, -480, 634, -480, 635,
                  -480, 634, -480, 634, -1595, 634, -1594, 633, -1596, 634, -1595, 634, -1595, 634]
            carrier_frequency: 38kHz
```

## 3 测试IR转发HA事件

断电重启IR接收器，Home Assistant设备页面会自动发现设备，将其添加进来。
到“开发者工具”->“动作”标签页，可以测试发自定义NEC码。NEC码由address和command两个uint16(0-65535)组成（省略repeats）。

![](/img/send-nec-ir-action.png)

从路由器查看IR接收器地址，浏览器访问，可查看红外接收日志。

Home Assistant至“开发者工具”->“事件”标签页，测试红外转发是否成功。

![](/img/ha-listen-event.png)

## 4 自定义IR码并配置自动化
按需自定义NEC码功能，最好用文档记录
```
addr: 6666
10001  开台灯
10002  关台灯
10003  开除湿机
10004  关除湿机
...
```
在红外音箱上，创建相应红外设备，用**发自定义NEC码**动作对设备学习。
![](/img/add-ir-device.jpg)

Home Assistant上定义**自动化**，响应红外事件操作。
![](/img/ha-listen-ir-event.png)

如果使用node-red，配置起来更直观。

![](/img/node-red-listen-ir-event.png)

## 总结
这种打通方式，原理是为Home Assistant制作一个红外遥控接收器，把音箱当遥控器使用。
实际上，IR接收器也能发IR，当房子有多个IR接收器时，可以进行IR中继。

```plantuml
actor User as u

box "书房"
participant "书房音箱\n(不带IR)" as s2
participant "书房IR接收器" as ir2
participant "书房风扇" as fan
end box

box "客厅"
participant "客厅音箱\n(带IR)" as s1
participant "客厅IR接收器" as ir1
end box

u -> s2: say:开风扇
s2 --> s1: 云调用
s1 -> ir1: 发IR
ir1->ir2: 中继
ir2 -> fan: 控制
```
更多有趣玩法期待你来解锁 :)

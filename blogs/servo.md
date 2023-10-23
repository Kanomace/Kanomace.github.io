---
layout: page
permalink: /blogs/servo/index.html
title: 矿山风门控制系统
---

# WiFi-servo-controller

## 概述

本项目使用ESP32通讯板及继电器对快速门电机控制器进行控制，并使用`MQTT`向本地服务器`EMQX`进行通讯。
如需修改，请先参阅[乐鑫科技](https://www.espressif.com.cn/en/products/sdks/esp-idf)中的开发引导了解ESP32模组开发相关概念和乐鑫物联网开发框架。

调试过程可对接EMQX本地服务器，可参考这篇博客[图文手把手教程--ESP32 MQTT对接EMQX本地服务器(VSCODE+ESP-IDF)](https://blog.csdn.net/felix_tao/article/details/125882339?spm=1001.2014.3001.5506)

项目仓库地址[WiFi-servo-controller](https://github.com/Kanomace/WiFi-servo-controller)

```bash
.根目录
├── Cloud Server
├── Hardware
│   ├── BOM_V1_2023-08-09              硬件接线图
│   ├── BOM_V1_2023-08-09              物料清单
│   ├── Gerber_V1_2023-08-09           PCB制版文件
│   ├── PCB_V1_2023-08-09              PCB文件
│   └── SCH_Schematic_V1_2023-08-09    原理图文件
├── Software
│   ├── Example
│   │   ├── MQTT_connect.ino           MQTT连接例程
│   │   └── MQTT_control.ino           MQTT控制例程
│   └── main
│       └── main20230813.ino           通讯板烧录代码
├── System architecture
│   └── 架构图                         系统整体架构
├── .git                               git仓库文件
├── .gitattributes                     git仓库文件
└── README.md               
```

## 硬件连接

继电器接口
- H6(上行端口) -> IO25  
- H7(下行端口) -> IO26  
- H8(停止端口) -> IO27  
 
## 通信协议

1. **远程控制开关门:订阅MQTT开关门控制主题**

- 主题名称: `DoorControl`
- 内容格式: `DoorControl/ID/Sender/Operation`
其中:
- *DoorControl* : 远程控制开关门：订阅MQTT开关门控制主题。
- *ID*          : 表示通讯板的ID，使用三位数字表示。( `001`: 001号通讯板，`031` : 031号通讯板)
- *Sender*      : 表示信息传输方。                (`TX`:服务器传输给通讯板的指令,`RX`:通讯板传输给服务器的信息)      
- *Operation*   : 表示此次信息执行的操作。         (`open`: 开门, `close`: 关门, `stop`: 停止)
例如 :
“DoorControl/001/TX/close” 表示001号*通讯板*接收到来自*服务器*的推送信息，执行相应的关门动作。
“DoorControl/031/RX/open ” 表示031号*服务器*接收到来自*通讯板*的推送信息，更新服务器上的动作信息。

2. **定时设置:订阅定时设置主题**

- 主题名称: `TimerSetting`
- 内容格式: `TimerSetting/ID/Sender/Operation/Hour/min`
其中:
- *TimerSetting* : 远程控制开关门：订阅MQTT开关门控制主题。
- *ID*           : 表示通讯板的ID，使用三位数字表示。( `001`: 1号通讯板，`031` : 31号通讯板)
- *Sender*       : 表示信息传输方。                (`TX`:服务器传输给通讯板的指令,`RX`:通讯板传输给服务器的信息)      
- *Operation*    : 表示此次信息执行的操作。         (`open`: 定时开启, `close`: 定时关闭, `stop`: 关闭定时功能)
- *Hour*         : 表示定时功能的时钟设置。         (`07`: 7时XX分 , `11`: 11时XX分)
- *min*          : 表示定时功能的分钟设置。         (`04`: XX时04分, `59`: XX时59分)
例如:
“DoorControl/001/TX/close” 表示001号*通讯板*接收到来自*服务器*的推送信息，执行相应的关门动作。
`DoorControl/001/TX/close`，即表示第一组定时时间为上午9:30开门。
`04/04/20/0`，即表示第二组定时时间为上午4:20关门  
`-1/xx/xx/xx`表示取消第一组的定时时间。
`00/xx/xx/xx`表示取消全部组别的定时时间

3. 查询定时列表：，收到该内容时向定时查询主题

- 主题名称：`001/Listing`
- 信息格式  “定时组1/定时时间/开关门控制;定时组2/定时时间/开关门控制..”
  例如：“1/8/30/0 ; 2/10/00/1;....”

## 开发日记

### 看门狗饱和导致ESP32重启，注意esp32-timer-hal.c中定时器和中断函数是不可被阻塞的

解决方法：加个定时器标志位，在`main`中执行中断函数
参考博客：
https://blog.csdn.net/Beihai_Van/article/details/125793806?ops_request_misc=&request_id=&biz_id=102&utm_term=sGuru%20Meditation%20Error:%20Core%20%20&utm_medium=distribute.pc_search_result.none-task-blog-2~all~sobaiduweb~default-1-125793806.142^v92^insert_down28v1&spm=1018.2226.3001.4187

### 隔次开机会有概率连接不上WiFi网络

导致原因未知 
解决方法：使用ESP内置函数`ESP.restore` 在连接不上WiFi时,尝试重启ESP32

## 参考资料
- [图文手把手教程--ESP32 MQTT对接EMQX本地服务器(VSCODE+ESP-IDF)](https://blog.csdn.net/felix_tao/article/details/125882339?spm=1001.2014.3001.5506)
- [MQTT服务器搭建和ESP32实现MQTT代码](https://blog.csdn.net/wcc243588569/article/details/123557400?spm=1001.2014.3001.5506)
- [ESP32 连接到免费的公共 MQTT 服务器](https://www.emqx.com/zh/blog/esp32-connects-to-the-free-public-mqtt-broker?utm_source=mqttx&utm_medium=referral&utm_campaign=mqttx-help-to-blog)
- [ESP8266 + MQTT ：如何实现 LED 灯的远程控制](https://www.emqx.com/zh/blog/esp8266_mqtt_led)
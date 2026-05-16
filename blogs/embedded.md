---
layout: page
permalink: /blogs/embedded/index.html
title: 嵌入式开发平台横向对比
---

## 嵌入式开发平台横向对比

> 最后更新: 2025年12月

经历了智能车竞赛、无人机开发、矿业物联网以及变电站科研项目，踩过了不少坑，也积累了一些心得。这篇文章整理了我在 **Arduino / STM32 / ESP32 / TC264（英飞凌）** 这四款主流嵌入式平台上的实战经验，供大家选型和入坑参考。

---

### Arduino

Arduino 是最适合嵌入式入门的平台，没有之一。标志性的 `setup()` / `loop()` 两函数结构让你在完全不理解寄存器的情况下就能驱动 LED、读传感器。

**硬件参数（Uno）**
- 主控: ATmega328P，16MHz
- 接口: 数字IO × 14，模拟IO × 6，UART / SPI / I2C

**核心 API**

```cpp
// 基础 IO
pinMode(pin, OUTPUT);
digitalWrite(pin, HIGH);
int val = analogRead(A0);  // 10-bit ADC，范围 0~1023

// PWM 输出（占空比 50%）
analogWrite(pin, 128);     // 0~255

// 非阻塞计时（替代 delay）
if (millis() - lastTime > 100) {
    lastTime = millis();
    doSomething();
}
```

**适用场景**: 第一块开发板、快速验证原型（如 PD 控制器仿真）、科普教学项目

**踩坑记录**: `delay()` 会阻塞整个程序，稍微复杂一点的逻辑就必须改用 `millis()` 计时。在普林斯顿的电机控制项目里，就是用 Arduino + 中断 + `millis()` 实现的自适应 PD 控制闭环。

---

### STM32

从 Arduino 过渡到 STM32 的核心门槛是**理解外设和时钟树**，但一旦上手，性能天花板远高于 Arduino。

**硬件参数（F4 系列）**
- 内核: ARM Cortex-M4，168MHz
- 外设: 12-bit ADC、多路 UART / SPI / I2C、定时器、DMA
- 常用型号: STM32F103（蓝色药丸，经典入门）/ STM32F407（性能主力）

**推荐工具链: STM32CubeMX + HAL 库**

CubeMX 图形化配置时钟和外设，自动生成初始化代码，新手友好。

```c
// UART 发送
HAL_UART_Transmit(&huart1, (uint8_t*)"Hello\r\n", 7, HAL_MAX_DELAY);

// 定时器 PWM 输出
HAL_TIM_PWM_Start(&htim1, TIM_CHANNEL_1);
__HAL_TIM_SET_COMPARE(&htim1, TIM_CHANNEL_1, 500);

// DMA 非阻塞串口接收
HAL_UART_Receive_DMA(&huart1, rxBuf, BUF_SIZE);
```

**项目经验**: 智能车项目中，STM32F1 负责底层传感器读取，用 TIM + DMA 驱动编码器测速，用 SPI 与 OLED 屏通信，整体性能完全够用。

**踩坑记录**:
- `HAL_Delay()` 同样阻塞，中断服务函数（ISR）内禁止调用，改用 `DWT` 或 `uwTick`
- 时钟树配置错误是最常见 bug，CubeMX 改完记得重新生成代码
- Flash 擦写寿命有限，频繁调试建议开 RAM 调试模式（`ST-Link` + `SWD`）

---

### ESP32

ESP32 是目前嵌入式 IoT 项目的最优解：双核处理器 + 内置 WiFi/BT + 丰富外设 + 低价格，加上 Arduino 框架支持，上手成本极低。

**硬件参数**
- 双核 Xtensa LX6，240MHz，内置 4MB Flash
- 协议: WiFi 802.11 b/g/n + Bluetooth 4.2 / BLE 5.0
- 外设: 34个GPIO、12-bit ADC × 18、SPI / I2C / UART / Touch Sensor

**网络与 MQTT**

```cpp
#include <WiFi.h>
#include <PubSubClient.h>

WiFi.begin("SSID", "password");
while (WiFi.status() != WL_CONNECTED) delay(500);

// MQTT 上云
client.publish("device/status", "{\"speed\": 1.2, \"battery\": 85}");
client.subscribe("device/cmd");
```

**FreeRTOS 双核分工**

ESP32 原生支持 FreeRTOS，可将传感器采集和网络上传分配到两个核心：

```cpp
// Core 0: 网络/MQTT 任务
xTaskCreatePinnedToCore(networkTask, "Net", 4096, NULL, 1, NULL, 0);
// Core 1: 传感器采集任务
xTaskCreatePinnedToCore(sensorTask,  "Sen", 4096, NULL, 1, NULL, 1);
```

**项目经验**:

无人机项目中，ESP32 实现 MAVLink 飞控通信 + UWB 定位数据融合 + MQTT 云端上报，双核的优势在这里非常明显，采集和通信完全不干扰。

矿业风门项目里，ESP32 作为远程控制板主控，通过 MQTT 与云平台对接，整套方案从设计到联调不到两周完成。

**踩坑记录**:
- ESP32 的 ADC 在 WiFi 启用时噪声增大，精密传感器建议走 I2C/SPI
- `Deep Sleep` 唤醒后 GPIO 状态重置，注意初始化顺序
- 项目复杂时建议迁移到 **ESP-IDF**（比 Arduino 框架更接近底层，性能更好）

---

### TC264

TC264（英飞凌 AURIX TC264DA）是全国大学生智能汽车竞赛的主流控制芯片，也是我在独轮机器人项目中的核心平台。双核 TriCore 架构，硬件实时性极强，但入门曲线比前三者陡峭不少。

**硬件参数**
- 双核 TriCore 1.6P，200MHz，8MB Flash / 1MB SRAM
- 特色: GTM 通用定时器模块（高精度 PWM / 编码器捕获），12-bit ADC
- 配套: 逐飞科技 TC264 核心板 + 开源 SDK

**逐飞 SDK 示例**

```c
// 编码器测速
encoder_init(ENCODER_1, ENCODER1_A, ENCODER1_B);
int32_t count = encoder_get_count(ENCODER_1);
encoder_clear_count(ENCODER_1);

// 电机 PWM（17kHz）
pwm_init(ATOM0_CH0, 17000, 0);
pwm_set_duty(ATOM0_CH0, 5000);  // 50% 占空比

// ADC 采集
adc_init(ADC_12BIT, ADC0, ADC_CH0, 0);
uint32_t val = adc_convert(ADC0, ADC_CH0);
```

**项目经验**: 独轮机器人同时跑：IMU（SPI）读取 + 编码器测速 + 串级 PID（姿态环 + 速度环）+ OpenMV 视觉通信（UART），5ms 主循环时间片，实时性完全满足需求。TC264 的 GTM 模块在编码器捕获精度上远超 STM32 的普通 TIM。

**踩坑记录**:
- GTM 模块配置较复杂，直接用逐飞封装，不要从手册从头写
- TriCore 无 OS 时，中断优先级管理要格外细心，高优先级 ISR 内避免耗时操作
- ADS（Aurix Development Studio）的 Watch Window 调试功能很强大，善用能大幅提升效率

---

### 横向对比

| 对比项 | Arduino Uno | STM32F4 | ESP32 | TC264 |
| :---: | :---: | :---: | :---: | :---: |
| **参考价格** | ¥30~50 | ¥20~60 | ¥15~40 | ¥200~400 |
| **主频** | 16 MHz | 168 MHz | 240 MHz | 200 MHz |
| **网络** | ✗ | ✗ | WiFi + BT ✓ | ✗ |
| **实时性** | 一般 | 好 | 好（RTOS） | 极好（GTM） |
| **入门难度** | ⭐ | ⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐ |
| **适用场景** | 原型/教学 | 工业/科研 | IoT/无人机 | 智能车/高实时控制 |

---

### 选型建议

- **刚入门嵌入式** → Arduino，两周内上手，资料极多，先建立感性认识
- **需要 WiFi / 蓝牙 / 云端** → ESP32，性价比无敌，直接替代 Arduino + 网络模块的组合
- **工业控制 / 科研项目** → STM32，外设齐全，HAL 库生态成熟，企业界认可度高
- **参加智能车竞赛 / 高实时性控制** → TC264，逐飞开源库 + 竞赛社区生态完善
---

*欢迎交流: jiacheng008@e.ntu.edu.sg*

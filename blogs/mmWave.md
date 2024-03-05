---
layout: page
permalink: /blogs/mmWave/index.html
title: 毫米波雷达文献综述
---

## 毫米波雷达文献综述

---

### 毫米波雷达传感器基础知识

<br>**毫米波 (mmWave)** 是一类使用短波长电磁波的特殊雷达技术。雷达系统发射的电磁波信号被其发射路径上的物体阻挡继而会发生反射。通过捕捉反射的信号，雷达系统可以确定物体的距离、速度和角度。毫米波雷达可发射波长为毫米量级的信号。在电磁频谱中，这种波长被视为短波长，也是该技术的优势之一。
<br>短波长的另一项优势是**高准确度**。工作频率为 76–81GHz（对应波长约为 4mm）的毫米波系统将能够检测小至零点几毫米的移动。
<br>完整的毫米波雷达系统包括**发送 (TX)** 和**接收 (RX)** **射频 (RF)** 组件，以及时钟等模拟组件，还有**模数转换器 (ADC)**、**微控制器 (MCU)** 和**数字信号处理器 (DSP)** 等数字组件。过去，这些系统都是通过分立式组件实现的，这增加了功耗和总体系统成本。其复杂性和高频率要求使得系统设计颇具挑战性。TI 器件可实现一种称为调频连续波 (FMCW) 的特殊毫米波技术。顾名思义，FMCW 雷达连续发射调频信号，以测量距离以及角度和速度。这与周期性发射短脉冲的传统脉冲雷达系统不同。

---

### 毫米波雷达的优势

- 高分辨率：毫米波雷达能够提供较高的空间分辨率，可以准确地检测和跟踪目标物体。它可以提供细节丰富的图像和数据，使得目标的形状、大小和运动轨迹等信息更加清晰。

- 适应不同环境：毫米波雷达对环境条件的适应性较强，无论是在恶劣的天气条件下（如雨、雪、雾等），还是在复杂的场景中（如城市街道、森林等），都能够保持较高的性能稳定性。

- 高精度测距：毫米波雷达能够实现高精度的距离测量，其测距误差通常在几厘米以内。这使得它在无人驾驶汽车、智能交通系统和工业自动化等领域中具有重要的应用潜力。

- 高抗干扰性：毫米波雷达对于干扰源（如其他雷达、无线电频段等）的抗干扰能力较强，能够有效地区分目标和干扰信号，减少误报和误警的可能性。

- 隐私保护：与某些传感器（如摄像头）相比，毫米波雷达不会直接获取目标的视觉信息，因此更具隐私保护性。这使得它在一些对隐私要求较高的应用场景中更加受欢迎。

相比于其他传感器技术（即摄像头，激光雷达和超声波传感器）相比，有高分辨率、适应不同环境、高精度测距、多目标检测和跟踪、高抗干扰性以及隐私保护等优点，使其在自动驾驶、智能交通、安防监控和工业应用等领域中得到广泛应用。

---

### 关于毫米波雷达硬件介绍

#### IWR1843BOOST 

> IWR1843 BoosterPack™ 插件模块是一款面向单芯片 IWR1843 器件的易用型 77GHz 毫米波传感器评估板，可直接连接至 TI MCU LaunchPad™ 开发套件生态系统。
> BoosterPack™ 包含开始为片上 C67x DSP 核心和低功耗 ARM® R4F 控制器开发软件所需的一切资源，包括用于编程和调试的板载仿真，以及用于快速集成简单用户界面的板载按钮和 LED。此套件配备有毫米波工具和软件。

<center>
<img src = "/blogs/mmWave.assets/iwr1843boost-angled.png" width="400" height="240">
</center>

在通过uniflash烧录官方例程时，可通过Ti官方上位机软件**mmWave_Demo_Visualizer**如图所示：

<center>
<img src = "/blogs/mmWave.assets/mmWave_Demo_Visualizer.png">
</center>

根据官方技术文档的解码（具体解码过程可参照[**Data Analysis**](https://github.com/Kanomace/Kanomace.github.io/blob/main/blogs/mmWave.assets/DataAnalysis.docx)），可得返回数据为LVDS格式，解码后可得一帧的数据格式为：

<center>
<img src = "/blogs/mmWave.assets/LVDS structure.png" width="300" height="500">
</center>

由于串口传输速率等问题，雷达板不能直接为PC端提供原始数据，所输出的数据为经过下图流程处理后的点云数据：

<center>
<img src = "/blogs/mmWave.assets/mmWave singal processing.png">
</center>

但是，通过标配 20 引脚 BoosterPack 接头，该评估板可与多种 MCU LaunchPad 开发套件兼容并简化原型设计工作。可以使用附加板来启用其他功能。**DCA1000EVM** 支持通过 **LVDS** 接口访问传感器的原始数据。

---

#### DCA1000EVM

> DCA1000 评估模块 (EVM) 为来自 TI AWR 和 IWR 雷达传感器 EVM 的两通道和四通道低电压差分信号 (LVDS) 流量提供实时数据捕获和流式传输。数据可以通过 1Gbps 以太网实时流式传输到运行 MMWAVE-STUDIO 工具的 PC 机上，以进行捕获、可视化，然后可以将其传递给所选的应用进行数据处理和算法开发。
>支持实验室和移动采集方案，从 AWR/IWR 雷达传感器捕获 LVDS 数据，并通过 1Gbps 以太网实时流式传输输出。 

<center>
<img src="/blogs/mmWave.assets/dca1000evm-angled.png" width="400" height="240">
</center>

通过 mmWave Studio (MMWAVE-STUDIO)，可以得到如图所示数据：

<center>
<img src="/blogs/mmWave.assets/mmWave Studio.png" >
</center>

---

#### 是否使用数据采集板的差异

| 型号 | 传输方式  | 信号类型 | 上位机软件 |
| :--- | :---: | ---: | ---: | 
|IWR1843|串口|经3次FFT处理后的点云数据|mmWave Studio|
|IWR1843+DAC1000|以太网|信号回波的原始数据|mmWave_Demo_Visualizer |

---

### 文献综述

[2020年~2024年毫米波雷达检测领域的文献调研](https://github.com/Kanomace/Kanomace.github.io/blob/main/blogs/mmWave.assets/mmWaveLiteratureReview.xlsx)

---

### 问题与总结

1. 毫米波雷达研究领域较新，多为高校实验室自主采集数据，数据集小且不公开，导致系统性能高度依赖训练集，模型适应性较弱
2. 对人体位姿的估计仅考虑单人状态，难以实现多人在场时的准确判断
3. 在室外道路复杂场景中，雷达回波噪声显著，人体RCS较小，动作幅度中提取的信号特征不明显
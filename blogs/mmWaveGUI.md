---
layout: page
permalink: /blogs/mmWaveGUI/index.html
title: 毫米波雷达文献综述
---

## 毫米波雷达串口采集并解析数据GUI

--- 

### 前言

>此教程适用于单TI毫米波雷达板卡串口返回数据并解析,未添加DCA1000数据采集板卡

<br>在使用TI的毫米波雷达板卡时,TI官方提供了如**mmWave Studio**,**mmWave_Demo_Visualizer**等上位机软件,以便新手能在短时间内运行一个DEMO程序可视化雷达板点云数据和进行雷达配置。
<br>虽然TI官方和网上的教程都很完整地给出了解析雷达版+DCA数据采集板的ADC数据教程,可是TI官方明确说明了并没有提供单雷达版点云数据采集的教程,笔者在网络论坛以及和同行们交流的过程中,大家也各自有自己开发采集代码的想法。
<br>因此,在笔者进行几天的串口数据解析的探索之后,终于成功地实现了简单地自定义采集帧率,并将原始数据和解析成功的数据以bin文件和xlsx文件储存的代码。
---

### 下载源码

1. 下载TI官方提供的radar_toolbox工具包，接下来的步骤会使用到其中的py文件，下载地址可点击这里

2. 根据下面的路径找到需要的上位机软件:*radar_toolbox_1_30_01_03\tools\visualizers\Industrial_Visualizer*找到*mmWave_Industrial_Visualizer.exe*
   
<br> 其实，在这个上位机软件中，除了可视化点云和运行TI的DEMO以外,已经可以实现简单地串口采集功能，大家不妨一试。

<br> 但是这个功能聊胜于无，因为采集的数据是不仅原始的**二进制数据**，还不能**自定义**采集文件的帧率（默认为100帧采集为一个bin文件，而TI雷达版的串口是1秒10帧，除非是静态点云的项目，否则这个数据采集之后还需要手动分割）

<br> 在尝试无果之后，笔者在阅读这个文档的时候发现了有趣的东西*mmWave_Industrial_Visualizer_User_Guide.html*

```bash
.radar_toolbox_1_30_01_03\tools\visualizers\Industrial_Visualizer
├── gui_main.py         the main program which controls placement of all items in the GUI and schedules UART read and graphing tasks.
├── gui_parser.py       defines an object used for parsing the UART stream. If you want to parse the UART data, you can use this file to do so.
├── gui_threads.py      defines the different threads that are run by the demo. These threads handle updating the plot and calling the UART parser.
├── graphUtilities.py   contains functions used to draw objects.
├── gui_common.py       aggregates several variables and configurables which are used across several files.
├── parseFrame.py       is responsible for parsing the frame format of incoming data.
└── parseTLVs.py        is responsible for parsing all TLV’s which are defined in the demos.
    
```

---

### 代码通读

#### parseTLVs.py

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
| :---: | :---: | :---: | :---: | 
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
---
layout: page
permalink: /blogs/mmWaveGUI/index.html
title: 毫米波雷达文献综述
---

## 毫米波雷达串口采集并解析数据GUI

--- 

### 🔥 前言

>此教程适用于单TI毫米波雷达板卡串口返回数据并解析,未添加DCA1000数据采集板卡

<br>在使用TI的毫米波雷达板卡时,TI官方提供了如**mmWave Studio**,**mmWave_Demo_Visualizer**等上位机软件,以便新手能在短时间内运行一个DEMO程序可视化雷达板点云数据和进行雷达配置。

<br>虽然TI官方和网上的教程都很完整地给出了解析雷达版+DCA数据采集板的ADC数据教程,可是TI官方明确说明了并没有提供单雷达版点云数据采集的教程,笔者在网络论坛以及和同行们交流的过程中,大家也各自有自己开发采集代码的想法。

<br>因此,在笔者进行几天的串口数据解析的探索之后,终于成功地实现了简单地自定义采集帧率,并将原始数据和解析成功的数据以bin文件和xlsx文件储存的代码。

---

### 🍴 下载源码

1. 下载TI官方提供的radar_toolbox工具包,接下来的步骤会使用到其中的py文件,下载地址可[点击这里](https://www.ti.com/tool/download/RADAR-TOOLBOX/1.30.01.03)

2. 根据的路径:*radar_toolbox_1_30_01_03\tools\visualizers\Industrial_Visualizer*
   找到*mmWave_Industrial_Visualizer.exe*这个上位机软件
   
<br> 其实,在这个上位机软件中,除了可视化点云和运行TI的DEMO以外,已经可以实现简单地串口采集功能,大家不妨一试。

<br> 但是这个功能聊胜于无😂,因为采集的数据是不仅原始的**二进制数据**,还不能**自定义**采集文件的帧率（默认为100帧采集为一个bin文件,而TI雷达版的串口是1秒10帧,除非是静态点云的项目,否则这个数据采集之后还需要手动分割）

<br> 在尝试无果之后,笔者在阅读这个文档的时候发现了有趣的东西[mmWave_Industrial_Visualizer_User_Guide.html](https://dev.ti.com/tirex/explore/node?node=A__AOFxUnA4gLo7lRDXedw8XQ__radar_toolbox__1AslXXD__LATEST)

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

<br> 💡 也就是说,上位机可以通过*parseFrame.py*和*parseTLVs.py*这两个脚本进行TLVS数据解析
<br> 通过*gui_parser.py*可以实现串口数据解析

---

### 🍭 代码通读

#### parseTLVs.py 

<br>这个文件主要是用于解析TLVs数据,用于其他脚本调用,这里不再赘述
<br>值得注意的是,下面的注释表明了解析后的点云数据格式为**x,y,z,Dopper,SNR,Noise**

````python
```
    # Parse each point
    numPoints = int((tlvLength-pUnitSize)/pointSize)
    for i in range(numPoints):
        try:
            x, y, z, doppler, snr, noise = struct.unpack(pointStruct, tlvData[:pointSize])
        except:
            numPoints = i
            print('Error: Point Cloud TLV Parser Failed')
            break
        
        tlvData = tlvData[pointSize:]
        # Decompress values
        pointCloud[i,0] = x * pUnit[0]          # x
        pointCloud[i,1] = y * pUnit[0]          # y
        pointCloud[i,2] = z * pUnit[0]          # z
        pointCloud[i,3] = doppler * pUnit[1]    # Dopper
        pointCloud[i,4] = snr * pUnit[2]        # SNR
        pointCloud[i,5] = noise * pUnit[3]      # Noise
    return numPoints, pointCloud
````

#### parseFrame.py 

<br>这个文件用于调用*def parseStandardFrame(frameData)*进行数据的解析
<br>读者可从**串口**copy一段毫米波雷达的**原始数据**放进**txt**文件,可使用笔者提供的例程尝试解析数据:

````python
```
# @file        real_data
# @type        python
# @author      Kano
# @date        2024-03-18
#@brief        用于解析毫米波雷达的数据
from gui_parser import *
import pandas as pd
import numpy as np

# 定义输入文件和输出文件路径
input_file = 'originData.txt'
output_file = 'ProcessedData.xlsx'  # 修改输出文件为xlsx格式
def parse_input_data(file_path):
    # 读取输入文件的数据
    with open(file_path, 'r') as file:
        data = file.read()
    # 移除换行符,并将数据拆分为帧
    frames = data.split('\n')
    # 解析每个帧的数据
    parsed_data = []
    for frame in frames:
        if len(frame) > 0:
            # 将帧数据转换为字节流
            frame_data = bytes.fromhex(frame)
            # 调用解析函数解析帧数据
            parsed_frame = parseStandardFrame(frame_data)
            # 将解析结果添加到列表中
            parsed_data.append(parsed_frame)
    return parsed_data

def extract_point_cloud(parsed_data):
    # 提取点云数据
    point_cloud = []
    for frame in parsed_data:
        point_cloud.extend(frame['pointCloud'])
    return point_cloud

def write_output_data(file_path, point_cloud):
    # 创建DataFrame对象
    df = pd.DataFrame(point_cloud, columns=['X', 'Y', 'Z', 'Doppler', 'SNR', 'Noise', 'Track index'])
    # 将DataFrame写入Excel文件
    df.to_excel(file_path, index=False)

# Press the green button in the gutter to run the script.
if __name__ == '__main__':
    # 解析输入数据
    parsed_data = parse_input_data(input_file)
    # 提取点云数据
    point_cloud = extract_point_cloud(parsed_data)
    # 将解析结果写入输出文件
    write_output_data(output_file, point_cloud)
````

<br>解析后的点云格式应为:

<center>
<img src = "/blogs/mmWaveGUI.assets/pointCloudDatatxt.png" width="400" height="240">
</center>

<br>至此,我们便完成了一段原始数据的解析。那么,我们如何实现串口返回数据的实时解析并保存呢

#### gui_parser.py 

<br>这个代码为官方上位机所使用的串口数据解析文件,如果读者有兴趣,也可以根据这个代码实现自己的实时上位机显示（开发时间会有点长）

<br>还记得上面说的:

> 在这个上位机软件中,除了可视化点云和运行TI的DEMO以外,已经可以实现简单地串口采集功能

<br>也就是说,只要我们能够理解此文件中的参数设置并尝试捕获解析后的数据,我们便可以实现在使用TI上位机的同时,修改它的功能,自定义**采集帧率,数据格式,每文件帧数,以及文件格式**

<br>直入正题,由于笔者所使用的的雷达版型号为**IWR1843**,所对应上位机串口处理函数为
<br>*def readAndParseUartDoubleCOMPort(self)*,各位根据自己的雷达版对应函数修改即可

<br>其中,保存**二进制**文件的代码段为:

````python
```
# If save binary is enabled
    if(self.saveBinary == 1):
        self.binData += frameData
        # Save data every framesPerFile frames
        self.uartCounter += 1
        if (self.uartCounter % self.framesPerFile == 0):
            # First file requires the path to be set up
            if(self.first_file is True): 
                if(os.path.exists('binData/') == False):
                    # Note that this will create the folder in the caller's path, not necessarily in the Industrial Viz Folder                        
                    os.mkdir('binData/')
                os.mkdir('binData/'+self.filepath)
                self.first_file = False
            toSave = bytes(self.binData)
            fileName = 'binData/' + self.filepath + '/pHistBytes_' + str(math.floor(self.uartCounter/self.framesPerFile)) + '.bin'
            bfile = open(fileName, 'wb')
            bfile.write(toSave)
            bfile.close()
            # Reset binData and missed frames
            self.binData = []
    # frameData now contains an entire frame, send it to parser
    if (self.parserType == "DoubleCOMPort"):
        outputDict = parseStandardFrame(frameData)
    else:
        print ('FAILURE: Bad parserType')
        
    return outputDict
````

用GPT解析一下该代码逻辑

````python
```
这段代码处理了接收到的数据帧(frameData)和保存二进制数据的逻辑.下面是对代码逻辑的解释:
1. 如果启用了保存二进制数据的选项(self.saveBinary == 1),则执行以下操作:
    将当前接收到的数据帧(frameData)添加到binData中
    每接收到framesPerFile帧数据时,保存一次数据。
    uartCounter递增1,用于计算接收到的帧数
    如果uartCounter是framesPerFile的倍数,说明需要保存数据到文件。
    第一次保存时需要设置路径:
        如果first_file为True,即第一个文件,检查是否存在binData/文件夹,如果不存在,则创建该文件夹
        在binData/文件夹下创建self.filepath文件夹
        将first_file设置为False,表示已经设置过路径
    将binData转换为字节流(toSave)
    根据当前保存的文件编号(math.floor(self.uartCounter/self.framesPerFile))构造文件名(fileName)
    打开文件(bfile)并将字节流写入文件
    关闭文件(bfile)
    重置binData,准备保存下一批数据
2.如果parserType是"DoubleCOMPort",则调用parseStandardFrame函数解析接收到的数据帧(frameData),并将解析结果存储在outputDict中
3.如果parserType不是"DoubleCOMPort",则打印错误信息"FAILURE: Bad parserType"
4.返回outputDict作为结果
总体上,这段代码实现了将接收到的数据帧保存为二进制文件,并调用相应的解析器对数据进行解析.解析的结果存储在outputDict中,并作为函数的返回值
````

<br>也就是,可以截取*outputDict*这个输出结果,仿照该代码段中bin文件保存的形式,进行xlsx,txt等等读者自己喜欢的方式保存文件！

<br>而*parseStandardFrame(frameData)*这个函数返回的值为一个字典,包含点云数,帧数,点云数据等内容,我们只需通过**上文所提供例程**的同样方法提取点云数据

<br>并且,可在函数开头修改*self.framesPerFile* 这个参数实现自定义采集帧率的代码啦

> 请注意,修改后的上位机应该经过重新编译上位机主函数*gui_main.py*,python环境请自行配置

### 📉 采集结果

<br>至此,我们便完成了自定义帧率的串口数据解析和采集啦！根据笔者提供的例程逻辑,解析后的数据如下所示:

<center>
<img src = "/blogs/mmWaveGUI.assets/pointCloudDataxlsx.png" width="400" height="240">
</center>

<br>此教程旨在讲解如何修改TI上位机,读者可根据自己的项目需求自行调整

<br>后续会将修改后的具体代码放到这个github仓库中(目前仅提供笔者所用的调试例程)

### 🏁 结语

最后的最后,感谢你阅读这份文字。如果笔者的博客成功帮助你实现了串口数据的解析和采集,还请你给本仓库的右上角点一个**star**❤️

与此同时,如果你有任何的建议/意见：

- 欢迎你在Github Issues中提出意见,并附上建议方案
- 或者在Github Discussions进行谈论,欢迎你提出任何看法
- 联系我的QQ:3065566278 
- 一起交流毫米波雷达项目吧😸~


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

<br> 也就是说，可以通过*parseFrame.py*和*parseTLVs.py*这两个脚本进行TLVS数据解析
<br> 通过gui_parser.py*可以实现串口数据解析

那么，这几个文件应该如何使用？
---

### 代码通读

#### parseTLVs.py

````python
```
import struct
import sys
import serial
import binascii
import time
import numpy as np
import math
import os
import datetime

# Local File Imports
from gui_common import *

# ================================================== Common Helper Functions ==================================================

# Convert 3D Spherical Points to Cartesian
# Assumes sphericalPointCloud is an numpy array with at LEAST 3 dimensions
# Order should be Range, Elevation, Azimuth
def sphericalToCartesianPointCloud(sphericalPointCloud):
    shape = sphericalPointCloud.shape
    cartestianPointCloud = sphericalPointCloud.copy()
    if (shape[1] < 3):
        print('Error: Failed to convert spherical point cloud to cartesian due to numpy array with too few dimensions')
        return sphericalPointCloud

    # Compute X
    # Range * sin (azimuth) * cos (elevation)
    cartestianPointCloud[:,0] = sphericalPointCloud[:,0] * np.sin(sphericalPointCloud[:,1]) * np.cos(sphericalPointCloud[:,2]) 
    # Compute Y
    # Range * cos (azimuth) * cos (elevation)
    cartestianPointCloud[:,1] = sphericalPointCloud[:,0] * np.cos(sphericalPointCloud[:,1]) * np.cos(sphericalPointCloud[:,2]) 
    # Compute Z
    # Range * sin (elevation)
    cartestianPointCloud[:,2] = sphericalPointCloud[:,0] * np.sin(sphericalPointCloud[:,2])
    return cartestianPointCloud


# ================================================== Parsing Function For Individual TLV's ==================================================

# Point Cloud TLV from SDK
def parsePointCloudTLV(tlvData, tlvLength, pointCloud):
    pointStruct = '4f'  # X, Y, Z, and Doppler
    pointStructSize = struct.calcsize(pointStruct)
    numPoints = int(tlvLength/pointStructSize)

    for i in range(numPoints):
        try:
            x, y, z, doppler = struct.unpack(pointStruct, tlvData[:pointStructSize])
        except:
            numPoints = i
            print('Error: Point Cloud TLV Parser Failed')
            break
        tlvData = tlvData[pointStructSize:]
        pointCloud[i,0] = x 
        pointCloud[i,1] = y
        pointCloud[i,2] = z
        pointCloud[i,3] = doppler
    return numPoints, pointCloud


def parseVelocityTLV(tlvData):
    velocity = []
    valid = False
    try:
        tempVel, tempConf = struct.unpack('1f1?', tlvData[:struct.calcsize('1f1?')])
        velocity.append(tuple((tempVel, tempConf)))
    except:
        velocity = []
        print('Error: Velocity TLV Parser Failed')
    return velocity

# Point Cloud Ext TLV from SDK for IWRL6432
def parsePointCloudExtTLV(tlvData, tlvLength, pointCloud):
    pUnitStruct = '4f2h' # Units for the 5 results to decompress them
    pointStruct = '4h2B' # x y z doppler snr noise
    pUnitSize = struct.calcsize(pUnitStruct)
    pointSize = struct.calcsize(pointStruct)

    # Parse the decompression factors
    try:
        pUnit = struct.unpack(pUnitStruct, tlvData[:pUnitSize])
    except:
            print('Error: Point Cloud TLV Parser Failed')
            return 0, pointCloud
    # Update data pointer
    tlvData = tlvData[pUnitSize:]

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

# Enhanced Presence Detection TLV from SDK
def parseEnhancedPresenceInfoTLV(tlvData, tlvLength):
    pointStruct = '1b'  # While there are technically 2 bits per zone, we need to use at least 1 byte to represent
    pointStructSize = struct.calcsize(pointStruct)
    numZones = (tlvData[0]) # First byte in the TLV is the number of zones, the rest of it is the occupancy data
    zonePresence = [0]
    tlvData = tlvData[1:]
    zoneCount = 0
    while(zoneCount < numZones):
        try:
            idx = math.floor((zoneCount)/4)
            zonePresence.append(tlvData[idx] >> (((zoneCount) * 2) % 8) & 3)
            zoneCount = zoneCount + 1
        except:
            print('Error: Enhanced Presence Detection TLV Parser Failed')
            break
    tlvData = tlvData[pointStructSize:]
    return zonePresence

# Side info TLV from SDK
def parseSideInfoTLV(tlvData, tlvLength, pointCloud):
    pointStruct = '2H'  # Two unsigned shorts: SNR and Noise
    pointStructSize = struct.calcsize(pointStruct)
    numPoints = int(tlvLength/pointStructSize)

    for i in range(numPoints):
        try:
            snr, noise = struct.unpack(pointStruct, tlvData[:pointStructSize])
        except:
            numPoints = i
            print('Error: Side Info TLV Parser Failed')
            break
        tlvData = tlvData[pointStructSize:]
        # SNR and Noise are sent as uint16_t which are measured in 0.1 dB Steps
        pointCloud[i,4] = snr * 0.1
        pointCloud[i,5] = noise * 0.1
    return pointCloud

# Range Profile Parser
# MMWDEMO_OUTPUT_MSG_RANGE_PROFILE
def parseRangeProfileTLV(tlvData):
    rangeProfile = []
    rangeDataStruct = 'I' # Every range bin gets a uint32_t
    rangeDataSize = struct.calcsize(rangeDataStruct)

    numRangeBins = int(len(tlvData)/rangeDataSize)
    for i in range(numRangeBins):
        # Read in single range bin data
        try:
            rangeBinData = struct.unpack(rangeDataStruct, tlvData[:rangeDataSize])
        except:
            print(f'Error: Range Profile TLV Parser Failed To Parse Range Bin Number ${i}')
            break
        rangeProfile.append(rangeBinData[0])

        # Move to next value
        tlvData = tlvData[rangeDataSize:]
    return rangeProfile

# Occupancy state machine TLV from small obstacle detection
def parseOccStateMachTLV(tlvData):
    occStateMachOutput = [False] * 32 # Initialize to 32 empty zones
    occStateMachStruct = 'I' # Single uint32_t which holds 32 booleans
    occStateMachLength = struct.calcsize(occStateMachStruct)
    try:
        occStateMachData = struct.unpack(occStateMachStruct, tlvData[:occStateMachLength])
        for i in range(32):
            # Since the occupied/not occupied flags are individual bits in a uint32, mask out each flag one at a time
            occStateMachOutput[i] = ((occStateMachData[0] & (1 << i)) != 0)
    except Exception as e:
        print('Error: Occupancy State Machine TLV Parser Failed')
        print(e)
        return None
    return occStateMachOutput

# Spherical Point Cloud TLV Parser
# MMWDEMO_OUTPUT_MSG_SPHERICAL_POINTS
def parseSphericalPointCloudTLV(tlvData, tlvLength, pointCloud):
    pointStruct = '4f'  # Range, Azimuth, Elevation, and Doppler
    pointStructSize = struct.calcsize(pointStruct)
    numPoints = int(tlvLength/pointStructSize)

    for i in range(numPoints):
        try:
            rng, azimuth, elevation, doppler = struct.unpack(pointStruct, tlvData[:pointStructSize])
        except:
            numPoints = i
            print('Error: Point Cloud TLV Parser Failed')
            break
        tlvData = tlvData[pointStructSize:]
        pointCloud[i,0] = rng
        pointCloud[i,1] = azimuth
        pointCloud[i,2] = elevation
        pointCloud[i,3] = doppler
    
    # Convert from spherical to cartesian
    pointCloud[:,0:3] = sphericalToCartesianPointCloud(pointCloud[:, 0:3])
    return numPoints, pointCloud

# Point Cloud TLV from Capon Chain
# MMWDEMO_OUTPUT_MSG_COMPRESSED_POINTS
def parseCompressedSphericalPointCloudTLV(tlvData, tlvLength, pointCloud):
    pUnitStruct = '5f' # Units for the 5 results to decompress them
    pointStruct = '2bh2H' # Elevation, Azimuth, Doppler, Range, SNR
    pUnitSize = struct.calcsize(pUnitStruct)
    pointSize = struct.calcsize(pointStruct)

    # Parse the decompression factors
    try:
        pUnit = struct.unpack(pUnitStruct, tlvData[:pUnitSize])
    except:
            print('Error: Point Cloud TLV Parser Failed')
            return 0, pointCloud
    # Update data pointer
    tlvData = tlvData[pUnitSize:]

    # Parse each point
    numPoints = int((tlvLength-pUnitSize)/pointSize)
    for i in range(numPoints):
        try:
            elevation, azimuth, doppler, rng, snr = struct.unpack(pointStruct, tlvData[:pointSize])
        except:
            numPoints = i
            print('Error: Point Cloud TLV Parser Failed')
            break
        
        tlvData = tlvData[pointSize:]
        if (azimuth >= 128):
            print ('Az greater than 127')
            azimuth -= 256
        if (elevation >= 128):
            print ('Elev greater than 127')
            elevation -= 256
        if (doppler >= 32768):
            print ('Doppler greater than 32768')
            doppler -= 65536
        # Decompress values
        pointCloud[i,0] = rng * pUnit[3]          # Range
        pointCloud[i,1] = azimuth * pUnit[1]      # Azimuth
        pointCloud[i,2] = elevation * pUnit[0]    # Elevation
        pointCloud[i,3] = doppler * pUnit[2]      # Doppler
        pointCloud[i,4] = snr * pUnit[4]          # SNR

    # Convert from spherical to cartesian
    pointCloud[:,0:3] = sphericalToCartesianPointCloud(pointCloud[:, 0:3])
    return numPoints, pointCloud


# Decode 3D People Counting Target List TLV
# MMWDEMO_OUTPUT_MSG_TRACKERPROC_3D_TARGET_LIST
#3D Struct format
#uint32_t     tid;     /*! @brief   tracking ID */
#float        posX;    /*! @brief   Detected target X coordinate, in m */
#float        posY;    /*! @brief   Detected target Y coordinate, in m */
#float        posZ;    /*! @brief   Detected target Z coordinate, in m */
#float        velX;    /*! @brief   Detected target X velocity, in m/s */
#float        velY;    /*! @brief   Detected target Y velocity, in m/s */
#float        velZ;    /*! @brief   Detected target Z velocity, in m/s */
#float        accX;    /*! @brief   Detected target X acceleration, in m/s2 */
#float        accY;    /*! @brief   Detected target Y acceleration, in m/s2 */
#float        accZ;    /*! @brief   Detected target Z acceleration, in m/s2 */
#float        ec[16];  /*! @brief   Target Error covarience matrix, [4x4 float], in row major order, range, azimuth, elev, doppler */
#float        g;
#float        confidenceLevel;    /*! @brief   Tracker confidence metric*/
def parseTrackTLV(tlvData, tlvLength):
    targetStruct = 'I27f'
    targetSize = struct.calcsize(targetStruct)
    numDetectedTargets = int(tlvLength/targetSize)
    targets = np.empty((numDetectedTargets,16))
    for i in range(numDetectedTargets):
        try:
            targetData = struct.unpack(targetStruct,tlvData[:targetSize])
        except:
            print('ERROR: Target TLV parsing failed')
            return 0, targets

        targets[i,0] = targetData[0] # Target ID
        targets[i,1] = targetData[1] # X Position
        targets[i,2] = targetData[2] # Y Position
        targets[i,3] = targetData[3] # Z Position
        targets[i,4] = targetData[4] # X Velocity
        targets[i,5] = targetData[5] # Y Velocity
        targets[i,6] = targetData[6] # Z Velocity
        targets[i,7] = targetData[7] # X Acceleration
        targets[i,8] = targetData[8] # Y Acceleration
        targets[i,9] = targetData[9] # Z Acceleration
        targets[i,10] = targetData[26] # G
        targets[i,11] = targetData[27] # Confidence Level
        
        # Throw away EC
        tlvData = tlvData[targetSize:]
    return numDetectedTargets, targets

def parseTrackHeightTLV(tlvData, tlvLength):
    targetStruct = 'I2f' #incoming data is an unsigned integer for TID, followed by 2 floats
    targetSize = struct.calcsize(targetStruct)
    numDetectedHeights = int(tlvLength/targetSize)
    heights = np.empty((numDetectedHeights,3))
    for i in range(numDetectedHeights):
        try:
            targetData = struct.unpack(targetStruct,tlvData[i * targetSize:(i + 1) * targetSize])
        except:
            print('ERROR: Target TLV parsing failed')
            return 0, heights

        heights[i,0] = targetData[0] # Target ID
        heights[i,1] = targetData[1] # maxZ
        heights[i,2] = targetData[2] # minZ

    return numDetectedHeights, heights

# Decode Target Index TLV
def parseTargetIndexTLV(tlvData, tlvLength):
    indexStruct = 'B' # One byte per index
    indexSize = struct.calcsize(indexStruct)
    numIndexes = int(tlvLength/indexSize)
    indexes = np.empty(numIndexes)
    for i in range(numIndexes):
        try:
            index = struct.unpack(indexStruct, tlvData[:indexSize])
        except:
            print('ERROR: Target Index TLV Parsing Failed')
            return indexes
        indexes[i] = int(index[0])
        tlvData = tlvData[indexSize:]
    return indexes

def parseVitalSignsTLV (tlvData, tlvLength):
    vitalsStruct = '2H33f'
    vitalsSize = struct.calcsize(vitalsStruct)
    
    # Initialize struct in case of error
    vitalsOutput = {}
    vitalsOutput ['id'] = 999
    vitalsOutput ['rangeBin'] = 0
    vitalsOutput ['breathDeviation'] = 0
    vitalsOutput ['heartRate'] = 0
    vitalsOutput ['breathRate'] = 0
    vitalsOutput ['heartWaveform'] = []
    vitalsOutput ['breathWaveform'] = []

    # Capture data for active patient
    try:
        vitalsData = struct.unpack(vitalsStruct, tlvData[:vitalsSize])
    except:
        print('ERROR: Vitals TLV Parsing Failed')
        return vitalsOutput
    
    # Parse this patient's data
    vitalsOutput ['id'] = vitalsData[0]
    vitalsOutput ['rangeBin'] = vitalsData[1]
    vitalsOutput ['breathDeviation'] = vitalsData[2]
    vitalsOutput ['heartRate'] = vitalsData[3]
    vitalsOutput ['breathRate'] = vitalsData [4]
    vitalsOutput ['heartWaveform'] = np.asarray(vitalsData[5:20])
    vitalsOutput ['breathWaveform'] = np.asarray(vitalsData[20:35])

    # Advance tlv data pointer to end of this TLV
    tlvData = tlvData[vitalsSize:]
    return vitalsOutput

def parseClassifierTLV(tlvData, tlvLength):
    classifierProbabilitiesStruct = str(NUM_CLASSES_IN_CLASSIFIER) + 'c'
    classifierProbabilitiesSize = struct.calcsize(classifierProbabilitiesStruct)
    numDetectedTargets = int(tlvLength/classifierProbabilitiesSize)
    outputProbabilities = np.empty((numDetectedTargets,NUM_CLASSES_IN_CLASSIFIER))
    for i in range(numDetectedTargets):
        try:
            classifierProbabilities = struct.unpack(classifierProbabilitiesStruct,tlvData[:classifierProbabilitiesSize])
        except:
            print('ERROR: Classifier TLV parsing failed')
            return 0, probabilities
        
        for j in range(NUM_CLASSES_IN_CLASSIFIER):
            outputProbabilities[i,j] = float(ord(classifierProbabilities[j])) / 128
        # Throw away EC
        tlvData = tlvData[classifierProbabilitiesSize:]
    return outputProbabilities

# Extracted features for 6843 Gesture Demo
def parseGestureFeaturesTLV(tlvData):
    featuresStruct = '10f'  
    featuresStructSize = struct.calcsize(featuresStruct)
    gesturefeatures = []

    try:
        wtDoppler, wtDopplerPos, wtDopplerNeg, wtRange, numDetections, wtAzimuthMean, wtElevMean, azDoppCorr, wtAzimuthStd, wtdElevStd = struct.unpack(featuresStruct, tlvData[:featuresStructSize])
        gesturefeatures = [wtDoppler, wtDopplerPos, wtDopplerNeg, wtRange, numDetections, wtAzimuthMean, wtElevMean, azDoppCorr, wtAzimuthStd, wtdElevStd]
    except:
        print('Error: Gesture Features TLV Parser Failed')
        return None

    return gesturefeatures

# Raw ANN Probabilities TLV for 6843 Gesture Demo
def parseGestureProbTLV6843(tlvData):
    probStruct = '10f'
    probStructSize = struct.calcsize(probStruct)

    try:
        annOutputProb = struct.unpack(probStruct, tlvData[:probStructSize])
    except:
        print('Error: ANN Probabilities TLV Parser Failed')
        return None

    return annOutputProb

# 6432 Gesture demo features
def parseGestureFeaturesTLV6432(tlvData):
    featuresStruct = '16f'
    featuresStructSize = struct.calcsize(featuresStruct)
    gestureFeatures = []

    try:
        gestureFeatures = struct.unpack(featuresStruct, tlvData[:featuresStructSize])
    except:
        print('Error: Gesture Features TLV Parser Failed')
        return None
    
    return gestureFeatures

# Detected gesture
def parseGestureClassifierTLV6432(tlvData):
    classifierStruct = '1b'
    classifierStructSize = struct.calcsize(classifierStruct)
    classifier_result = 0

    try:
        classifier_result = struct.unpack(classifierStruct, tlvData[:classifierStructSize])
    except:
        print('Error: Classifier Result TLV Parser Failed')
        return None

    return classifier_result[0]
# Surface Classification
def parseSurfaceClassificationTLV(tlvData):
    classifierStruct = '1f'
    classifierStructSize = struct.calcsize(classifierStruct)
    classifier_result = 0

    try:
        classifier_result = struct.unpack(classifierStruct, tlvData[:classifierStructSize])
    except:
        print('Error: Classifier Result TLV Parser Failed')
        return None

    return classifier_result[0]

# Mode in Gesture/KTO demo 
# def parseGesturePresenceTLV6432(tlvData):
#     presenceStruct = '1b'
#     presenceStructSize = struct.calcsize(presenceStruct)
#     presence_result = 0

#     try:
#         presence_result = struct.unpack(presenceStruct, tlvData[:presenceStructSize])
#         print(presence_result)
#     except:
#         print('Error: Gesture Presence Result TLV Parser Failed')
#         return None

#     return presence_result[0]

def parseExtStatsTLV(tlvData, tlvLength):
    extStatsStruct = '2I8H' # Units for the 5 results to decompress them
    extStatsStructSize = struct.calcsize(extStatsStruct)
    # Parse the decompression factors
    try:
        interFrameProcTime, transmitOutTime, power1v8, power3v3, \
        power1v2, power1v2RF, tempRx, tempTx, tempPM, tempDIG = \
        struct.unpack(extStatsStruct, tlvData[:extStatsStructSize])
    except:
            print('Error: Ext Stats Parser Failed')
            return 0

    tlvData = tlvData[extStatsStructSize:]

    procTimeData = {}
    powerData = {}
    tempData = {}
    # print("IFPT : " + str(interFrameProcTime))

    procTimeData['interFrameProcTime'] = interFrameProcTime
    procTimeData['transmitOutTime'] = transmitOutTime

    powerData['power1v8'] = power1v8
    powerData['power3v3'] = power3v3
    powerData['power1v2'] = power1v2
    powerData['power1v2RF'] = power1v2RF

    tempData['tempRx'] = tempRx
    tempData['tempTx'] = tempTx
    tempData['tempPM'] = tempPM
    tempData['tempDIG'] = tempDIG

    return procTimeData, powerData, tempData


def parseRXChanCompTLV(tlvData, tlvLength):
    compStruct = '13f' # One byte per index
    compSize = struct.calcsize(compStruct)
    coefficients = np.empty(compSize)
    try:
        coefficients = struct.unpack(compStruct, tlvData[:compSize])
    except:
        print('ERROR: RX Channel Comp TLV Parsing Failed')
    # Print results to the terminal output
    print("compRangeBiasAndRxChanPhase", end=" ")
    for i in range(13):
        print(f'{coefficients[i]:0.4f}', end=" ")
    print('\n')

````

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
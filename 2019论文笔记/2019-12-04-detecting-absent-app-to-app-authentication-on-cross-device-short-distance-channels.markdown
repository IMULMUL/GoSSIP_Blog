---
layout: post
title: "Detecting (Absent) App-to-app Authentication on Cross-device Short-distance Channels"
date: 2019-12-04 01:15:26 -0500
comments: true
categories: 
---
> 作者：Stefano Cristalli，Long Lu，Danilo Bruschi，Andrea Lanzi
>
> 单位：University of Milan，Northeastern University
>
> 会议：[ACSAC 2019](https://www.acsac.org/)
>
> 链接：[pdf](https://www.longlu.org/downloads/catch.pdf)

---

##　1 Introduction

移动应用越来越多地使用短距离以跨设备的方式交互或交换数据。在本文中，作者关注跨设备app-app的通信劫持 （ CATCH ，cross-device app-to-app communication hijacking）问题，App使用短距离信道工作（例如蓝牙和 Wi-Fi-Direct）。之前有工作提出了提供了设备配对和身份验证方法，但他们对设备上运行的应用程序不起作用，也会导致未经身份验证或恶意的app-to-app交互。

作者设计了一种基于数据流分析的算法，用于检测 Android 应用中 CATCH 的存在。该算法检查给定应用是否包含防止 CATCH 所必需的app-to-app的身份验证方案。在包含 662 个使用蓝牙技术的 Android 应用（于Androzoo 存储库中）的数据集上执行实验，显CATCH 问题存在于所有应用上。

<!--more-->

通过设计和开发一个身份验证方案检测器来分析 Android 应用程序以发现潜在的漏洞，从而为 CATCH 问题提供解决方案。

手动分析验证在 Android 应用程序上的系统结果，并测试其检测身份验证方案的准确性。结果表明，该方法没误报和漏报。

## 2 Background

目前，大多数跨设备、peer-to-peer的信道都使用out-of-band方案进行身份验证，其工作方式如下。

用户（requesting user）A 启动从其设备到附近设备的通信，然后提示其用户（accepting user）B进行确认。请求确认时，B 必须通过单独的信道（例如，口头）向A 提供秘密信息PIN，或者作为简单的"接受"按钮能够识别尝试启动的设备的信息。这些步骤已在 Android 中实现，不需要为通信通道重新实现身份验证。身份验证通过后，可以开始通信。蓝牙使用加密来保护信道。在此方案中，请务必注意与身份验证相关的两点：

1. 身份验证是通过共享out-of-band信息/机密进行的。

2. 在信道上执行的身份验证（图 1）不足以保证更高级别应用程序之间在信道上进行身份验证。

   ![img](/images/2019-12-04/fig1.png)

第 1 点非常重要。确认身份验证所需的交换信息实际上是两个用户之间的视觉和口头接触，并且out-of-band元素在所有此类身份验证中都是常量。

作者认为这种设备身份验证是危险的，如图二例。使用 Wi-Fi-Direct 的聊天应用打开服务器套接字，接受通过其通信，并将传入消息显示给最终用户。

![img](/images/2019-12-04/fig2.png)

应用的预期用途是安装在以对等方式相互通信的两台设备上。如果一台设备上存在恶意应用。由于是设备身份验证，而不是应用，因此恶意应用有权通过通道进行通信，就像设备上安装的任何其他应用一样。因此，恶意应用可以制作自定义消息以发送到其他设备，这些消息的显示方式与从原始应用发送一样。如果良性应用中没有执行身份验证的代码，则不可能检测此类操作。可能会导致以下攻击：

**网络钓鱼**：在上述示例中，恶意应用可能会将网络钓鱼材料发送到其他应用。用户可能会信任并打开内容，因为他无法将其与从与他通信的设备发送的良性内容区分开来。

**恶意软件传递**：同一系统可用于向用户传递恶意软件，其形式为恶意文件，在打开时触发漏洞（例如，针对易受攻击的 PDF 阅读器的恶意 PDF 文件）。

**Exploitation**：如果目标应用根据从信道收到的命令执行内部操作，则恶意应用可能会发送命令造成恶意代码执行。例如，可以给通过蓝牙设备给文件管理器应用传删除文件命令。

作者认为其他网络攻击（如 MITM）在此条件下中很难完成，因为攻击者应该物理地靠近设备才能劫持通信，并且还需要克服或绕过信道保护，如加密。所以作者认为CATCH攻击才是最现实的，要构建一个自动分析 Android 应用的系统，旨在检测特定通信渠道上是否存在（或缺乏）身份验证。作者实现的目的是提供一个工具，以帮助应用程序开发人员保护其软件，以及用作应用市场（例如 Google Play）上的安全扫描器，用于检测潜在的身份验证漏洞。

## 3 Overview

###  3.1 认证定义

身份验证模型考虑了两个设备，D1 和 D2，分别安装了应用 A1 和 A2。这两个设备建立一个经过身份验证的信道，其中 A1 和 A2 启动通信。这种形式的身份验证建议使用与标准 RF 信道不同的几种方法在移动设备之间进行经过身份验证的信息交换。这些被称为out-of-band、侧信道或位置受限信道 （LLC），包括音频、视觉、红外、超声波和其他形式的传输。这些技术允许接收器对传输源进行物理验证。使用此信息，设备经过相互身份验证，并可以建立安全共享密钥。更确切地说，在这样的身份验证方案中，我们识别以下步骤：

1. A2 获取将用于验证通信的机密。此机密在设备 D2 上生成，然后传达到应用 A1，或者由 D1 生成，然后与 A2 共享。这种通信使用out-of-band信道，也称为"人辅助信道"。攻击者无法操纵此类信道，因此根据定义，它被视为受信任的信道。
2. A1 和 A2 共享同一个秘密后，可以使用该秘密作为身份证明开始发送数据。根据秘密是什么和应用程序协议，数据可以使用从秘密派生的密钥（例如hash(secret)）进行加密，也可以将秘密与用于验证传输的数据一起作为纯文本发送。
3. 在这两种情况下（使用从秘密派生的密钥 进行加密，或以简单密码短语/PIN 方式随数据发送的机密），应用 A2 需要对接收的数据执行身份验证检查。在第一种情况下，A2 需要检查密钥执行的解密操作是否正确，在第二种情况下，A2 需要检查密码短语/PIN 是否正确。这些检查必须在数据的任何关键使用之前进行，否则通信未经身份验证。只有在检查正确的情况下，数据经过身份验证，通信才能继续。

作者给出了以下几个一般性定义：

模型中的**通信**定义为从 A1 到 A2 的一些数据交换，从 A2 从通信通道读取数据开始。

数据的**使用**定义为任何结果依赖数据本身的操作。

将数据的**已认证使用**定义为在访问数据之前需要验证的任何指令。

在模型中给出了以下**身份验证定义**：给定通过peer-to-peer信道的通信与交换数据 D ，则身份验证是位于通信开始和第一次D的身份验证使用之间的代码。D：（1） 允许在 D 成功验证的情况下继续执行，或者 （2） 在每次身份验证不成功时防止对 D 进行任何经过身份验证的使用。

身份验证检查的内部逻辑取决于上下文，因此无法将其包含在定义中。

### 3.2 身份认证的检测

第一步是识别应用程序中的边界代码点，即入口点和出口点，中间部分可能包括身份验证方案。入口点为通过分析信道（例如，数据接收）开始通信例如11行的socket.getInputStream()。出口点为第一次来自监视信道数据的**已认证使用**。例如15行的写出文件操作。但是出口点存在歧义，如果是个写日志的操作就不能算是出口点。

![img](/images/2019-12-04/list1.png)

由于数据使用的这种歧义，作者设计了一个不依赖于出口点的检测策略。作者设计了一种基于程序分析技术的算法（算法3.1），用于执行数据和控制流分析。该算法开始计算每个分析目标应用（第 7-8 行）的控制流图 （CFG） 和数据依赖关系图 （DDG）。对于CFG中的每个节点，系统通过使用函数isEP确定它是否是一个入口点。此函数使用基于与特定通信通道相关的函数签名的预定义表（第 4.2 节）。如果未找到入口点，则返回结果"NO AUTH NEEEDED"（第 9-12 行）。在所有其他情况下，将分析 DDG 中的每个节点。如果节点表示条件，则系统将检查 DDG 中是否存在将入口点连接到条件节点的路径（第 16-17 行）。

![img](/images/2019-12-04/al1.png)

如果存在这样的路径，则意味着可能找到了身份验证方案。但是，仍然有可能获得误报：对数据进行简单的健全性检查或其他控制都会被错误地标识为身份验证。为了减少候选身份验证条件中的误报数，该算法应用了恒定传播( constant propagation)技术。从技术上讲，这种技术是使用达到定义分析结果。特别是，如果将常量值分配给变量，并且此类变量未在代码中的点 P 之前修改，则变量在 P 处具有常量值，并可以替换为常量。

由于分析的身份验证模型必须使用通常存储在动态内存（例如堆、堆）中的某种动态生成秘密执行，因此，通过使用恒定传播，可以放弃在比较中使用常量值的所有条件，因为它们肯定不表示对数据的身份验证。恒定传播是一种非常强大的技术，它有助于将误报率降低到 0%。

## 4 SYSTEM IMPLENMENTATION

### 4.1 Overview

系统基于Argus-SAF框架设计，由三个主要组件组成：（1） 图形生成器，（2） 路径查找器，（3） app-to-app身份验证查找器。输入APK文件，输出找到的身份验证或者包含身份验证检查的特定指令列表。

1. 图形生成器在 APK 上启动 Argus-SAF 分析。该框架应用四个连续步骤：

   > （1）从 Dalvik 字节码生成 Java IR 。

   > （2） 生成 Android 系统的环境模型。这对捕获组件之间的控制流和交互（如活动之间的intent调度）至关重要。

   > （3） 此时，Argus-SAF 构建了整个应用程序的组件间控制流图 （ICFG）。同时，它执行数据流分析，并在ICFG之上构建组件间数据流图（IDFG）。

   > （4） 该框架在IDFG顶部构建数据依赖关系图（DDG）。

2. 第二个组件 Path Finder 的主要目标是在代码中查找可能存在身份验证方案的区域。

   > 遍历从图形生成器接收的 CFG，并根据预定义的方法签名列表标记分析信道的入口点。

   > 提取 Argus-SAF 中"IfStatement"类型的所有节点遍历所有条件。

   > 找到从信道到读取数据的变量，是否有在IfStatment中使用过，将这样的定义-使用对映射到DDG中的边，获得其路径。

3. app-to-app身份验证查找器，对路径进行检查，排除误报。它分析 Java IR 中的 if 语句，该语句可以分为两种类型：（1） 两个变量之间的比较，（2） 变量和常量之间的比较。（2）会被丢弃，而第一种类型下，系统会通过恒定传播继续检查其中一个变量是否是常量，如果是再次被丢弃。

### 4.2 入口点的选择

作者在这里只关注蓝牙的信道研究，但是他们任务修改入口点的识别可以扩展到对所有信道的研究。

对于基于BluetoothSocket的蓝牙通信，有两个可能的入口点（即蓝牙套接字流开始接收数据的位置）：BluetoothSocket.getInputStream 和 InputStream.read。

由于后者在别的地方也有使用，容易造成误报，并且这两种都是成对出现的，作者只将前者定义为入口点。

## 5 EXPERIMENTAL EVALUATION

### 1.数据集分析

作者从Androzoo数据集收集APK，Androzoo 数据集包含超过 300 万个 Android 应用程序，这些应用程序来自多个 Android 市场：Google Play、Anzhi和 AppChina。作者在数据集中随机选择了210,425个APK，经过三步筛选：

（1）申请了蓝牙权限且包含了某些与蓝牙相关的库和类（BluetoothSocket），剩下2,793个APK。

（2）去除使用ProGuard进行代码混淆：用启发式识别方法（a.class)，剩下942个APK。

（3） 在CFG中没有蓝牙通信的入口点，可能是某些库和类中有蓝牙功能，但是未使用，剩下238个APK。

在没有开恒定传播的检测下，26个（大约11%）被认为是有身份认证，其他都没有。

有恒定传播检测下，所有被判别为没有身份认证。作者手动检查了20个APK，没有发现误报。

作者又对被识别为有混淆的1797个APK进行了分析，其中使用蓝牙的有662个，分析结果也是全部没有进行身份验证。作者随机选取了15个进行手动分析确认。

对每个Apk的检测大概需要5分钟。

### 2.案例分析

#### BluetoothChat

使用BlutoothSocket通信的的聊天软件，开源

（1）恶意应用知道目标应用安装

（2）恶意应用知道目标应用运行

使用UUID与对方设备聊天

![img](/images/2019-12-04/fig3.png)

#### Wifi Direct+

# Discussion

Argus-SAF 无法处理特定的组件内和组件间转换，例如反射和并发。反射在这不常用，但是为了避免阻塞输入/输出，并发操作肯定会使用，所以作者手工检查了662个中的20个应用，也没有发现认证。



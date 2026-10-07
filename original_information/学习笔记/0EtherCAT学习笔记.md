# EtherCAT 学习笔记

## 1. 总体结构：先把主站、ESC、MCU 放进一张图

### 1.1 系统组成与四层分工

主站：以太网控制器

从站：远程IO、伺服、变频、步进、分支器

先只记四层：

| 层 | 你可以把它理解成 | 典型名字 |
|---|---|---|
| 网络外面 | 数据怎样从一台设备跑到另一台设备 | 网线、PHY、MII、Port、EBUS |
| ESC 本体 | EtherCAT 协议真正被硬件执行的地方 | ESC、EPU、寄存器、DPRAM |
| ESC 和 MCU 之间 | MCU 怎么读写 ESC | PDI、SPI、MCI、IRQ |
| 应用 | 真正控制电机、IO 的程序 | MCU、驱动程序、控制算法 |

```text
前导码，8 字节
14字节的EtherNet头 + 2字节的EtherCAT头 + 30字节的EtherCAT Datagram中的data总和 + 6 *（10字节的子头 + 2字节的WKC）+FCS 
帧间距，12 字节

						   【主站】
                              │
                    Ethernet / EtherCAT Frame
                              │
                         PHY / MII
                              │
                         ┌────▼────┐
                         │   ESC   │
                         └────┬────┘
                              │
        ┌─────────────────────┼────────────────────────┐
        │                     │                        │
   【帧/地址层】          【共享数据层】              【时间层】
        │                     │                        │
  Port / EPU                DPRAM                     DC
        │                     │                        │
  Auto/Configured            SM                  Delay/Offset
  /Logical Address           │                   Difference
        │              ┌──────┴──────┐                │
       FMMU          Buffer         Mailbox           SYNC
        │              │              │                │
        │             PDO        CoE/SDO/...          MCU同步
        │              │              │                │
        └──────────────┴──────┬───────┴────────────────┘
                              │
                             PDI
                              │
                         SPI / MCI
                              │
                             MCU
                              │
                         电机 / IO

另外两套“管理机制”横跨全局：

AL State Machine：决定从站开放到 INIT / PRE-OP / SAFE-OP / OP 哪一阶段
Event / IRQ / Watchdog：负责通知异常、数据变化和超时

EEPROM / SII：负责“上电时我是谁、我应该怎么配置”
```

### 1.2 ESC：替 MCU 处理 EtherCAT 的硬件

**ESC = EtherCAT Slave Controller，EtherCAT 从站控制器。**

普通以太网设备往往是：

```text
网线 → PHY → MAC → MCU
```

而典型 EtherCAT 从站是：

```text
网线 → PHY → ESC → MCU
```

MCU 不需要自己逐字节分析 EtherCAT 帧，也不需要自己完成高速转发。
ESC 在数据帧经过时就完成检查、寻址匹配、读写数据、WKC 更新和继续转发。

所以以后看到一句：

> “EtherCAT 通信性能不受 从站微处理器 性能限制。”

你应该翻译成：

> **网络上那条高速实时通道主要由 ESC 硬件跑，MCU 主要做应用。**

这也是为什么一个 MCU 不算很快，从站仍然可以参加高速 EtherCAT 网络。

### 1.3 ET1100：ESC 的一种具体芯片

**ESC 是类别，ET1100 是具体型号。**

```text
ESC 是类别
ET1100 是 Beckhoff 的一种 ESC 芯片
ET1200 是另一种 ESC 芯片
```

ET1100 有这些你需要记住的“大件”：

```text
4 个物理通信端口
8 个 FMMU
8 个 SyncManager
4 KB 控制寄存器区
8 KB 过程数据 RAM

64 位 Distributed Clocks
PDI 接口：数字 IO / SPI / 并行 MCU 接口
EEPROM 接口
PHY 管理
```

> **ET1100 里面已经把“通信端口 + 内存 + 映射 + 同步 + 时钟”这整套 EtherCAT 从站基础设施做成了硬件。**

### 1.4 EPU：ESC 内部处理 Datagram 的模块

**EPU = EtherCAT Processing Unit。**

简单说：

> **帧从 ESC 里面经过时，EPU 是检查 EtherCAT Datagram，并决定要不要对本 ESC 的地址空间读/写。**

不要把 EPU 想成一颗单独 CPU。这里更适合把它理解成 ESC 内部的一个硬件处理模块。

以后看到：

```text
Port Receive Time
EPU Receive Time
```

区别是：

- Port Receive Time：帧到某个物理端口时的时间戳；
- EPU Receive Time：帧真正到达 EtherCAT 处理单元时的时间戳。

<a id="chapter-02"></a>

## 2. 网络基础：Ethernet、IP、TCP/UDP 与 EtherCAT 的关系

### 2.1 五层模型：每一层看什么

```text
网络：
应用层（XDPro、微信，点击“下载程序到PLC”这是应用层动作，XNet是上层通信协议）
传输层（XDPro、微信都有数据，传输层靠 端口号 区分“这个数据给谁”，TCP 或 UDP 端口号，TCP/UDP是传输层协议）
网络层（IP地址）
链路层（MAC地址）
物理层（网线里面的电信号）
```

五层模型不是说：每个协议都必须把五层全部用满
EtherCAT直接使用 Ethernet 帧承载自己的数据

### 2.2 MAC、IP 与 TCP/UDP 端口各是什么

```text
MAC
网卡的链路层地址
IP
不是网卡“天生自带”的，操作系统把 IP 地址配置到一个网络接口上，192.168.1.100可被改成192.168.1.200，也可以设置多IP
TCP/UDP端口
由操作系统的TCP/IP协议栈管理，传输层编号
```

物理网口：电脑RJ45网口、交换机1口、交换机2口 

TCP/UDP端口号：80、50000（Windows网络协议栈的逻辑编号），端口号范围是0 ～ 65535，但TCP 5000和UDP 5000是两回事

一个网卡可以有多个 IP

### 2.3 普通应用通信：Ethernet → IPv4 → TCP → 应用数据

```text
如果 Xnet 实际通过 TCP/UDP 承载
┌────────────────────────────┐
│ ⑤ 应用层                   │
│                            │
│ XDPro                      │
│ Xnet协议                   │
├────────────────────────────┤
│ ④ 传输层                   │
│                            │
│ TCP/UDP                    │
│                            │
│ 使用：源端口、目标端口       │
├────────────────────────────┤
│ ③ 网络层                   │
│                            │
│ IPv4                       │
│                            │
│ 使用：源IP、目标IP          │
├────────────────────────────┤
│ ② 数据链路层               │
│                            │
│ Ethernet                   │
│                            │
│ 使用：源MAC、目标MAC        │
├────────────────────────────┤
│ ① 物理层                   │
│                            │
│ 网卡PHY                    │
│ RJ45                       │
│ 双绞线                     │
│ 电信号                     │
└────────────────────────────┘
```

```text
浏览器
↓
HTTP / HTTPS
↓
TCP（或QUIC/UDP）
↓
IP
↓
Ethernet / Wi-Fi
↓
路由器
↓
互联网
```

### 2.4 Modbus TCP 的分层与客户端、服务器

```text
┌────────────────────────────┐
│ ⑤ 应用层                   │
│                            │
│ Modbus                     │
├────────────────────────────┤
│ ④ 传输层                   │
│                            │
│ TCP                        │
│ TCP端口502                 │
├────────────────────────────┤
│ ③ 网络层                   │
│                            │
│ IPv4                       │
├────────────────────────────┤
│ ② 数据链路层               │
│                            │
│ Ethernet                   │
├────────────────────────────┤
│ ① 物理层                   │
│                            │
│ Ethernet PHY               │
│ RJ45 / 网线                │
└────────────────────────────┘
```

```text
Modbus TCP
我要找哪台设备？
↓
IP地址

我要找设备里的哪个服务？
↓
TCP端口
```

只有客户端能发起请求，服务器只能被动应答、不能主动上报
任何设备都可以被配置成客户端或服务器
一个设备可以一边监听 502 端口接受别人的请求（当服务器），一边主动发起连接去读别的设备（当客户端）

### 2.5 EtherCAT 直接使用 Ethernet 帧

```text
┌────────────────────────────┐
│ ⑤ 应用层                   │
│                            │
│ EtherCAT设备的应用数据      │
│ PDO / SDO / CoE 等          │
├────────────────────────────┤
│ ④ 传输层                   │
│                            │
│ —— 没有TCP/UDP ——           │
├────────────────────────────┤
│ ③ 网络层                   │
│                            │
│ —— 没有IPv4 ——              │
├────────────────────────────┤
│ ② 数据链路层               │
│                            │
│ Ethernet帧                 │
│ + EtherCAT协议             │
│ EtherType = 0x88A4         │
├────────────────────────────┤
│ ① 物理层                   │
│                            │
│ Ethernet PHY               │
│ 网线                       │
└────────────────────────────┘
```

```text
EtherCAT 主站把一个 Ethernet 帧发出去
主站
 ↓
从站1
 ↓
从站2
 ↓
从站3
 ↓
再返回主站

EtherCAT 从站会在帧经过自己的时候直接读取或修改属于自己的数据。
一个帧沿着整条 EtherCAT 链路依次经过所有从站
不需要
IPv4地址
TCP
UDP
TCP/UDP端口

EtherCAT寻址
自动增量寻址
配置地址
逻辑地址
而不是靠IP寻址
```

### 2.6 EtherType：Ethernet 里装的是什么

```text
网络上的数据不一定有 TCP/UDP 端口
Ethernet
 ├─ IPv4
 │    └─ TCP
 │         └─ XDPro数据
 │
 ├─ ARP
 │
 ├─ LLDP
 │
 └─ EtherCAT
ARP、EtherCAT没有 TCP/UDP 端口号

Ethernet帧
┌─────────────────────────┐
│ 目标MAC                 │
│ 源MAC                   │
│ 类型EtherType           │
│                         │
│     内容 / Payload      │
│                         │
└─────────────────────────┘
```

如果EtherType = 0x88A4，则Ethernet → EtherCAT

### 2.7 普通以太网与 EtherCAT 的连接路径

```text
普通以太网
MAC → PHY → RJ45 → 网线 → RJ45 → PHY → MAC
EtherCAT
主站 → MAC → PHY → 变压器 → RJ45 → 网线 → RJ45 → 变压器 → PHY → ESC → MCU → 应用

EtherCAT从站不是靠MAC处理数据，而是靠ESC硬件实时转发
网线到PHY是电信号
PHY到MAC是数字信号
```

<a id="chapter-03"></a>

## 3. 主站：离线配置与在线运行怎样配合

### 3.1 配置工具负责规划，主站驱动负责运行

EtherCAT 配置工具（离线工具）
大脑在做作战规划，把总线上挂了哪些设备、每个设备输入输出多少数据、数据放在总线数据包的哪个字节算好。
典型软件：倍福的 TwinCAT

EtherCAT 主站驱动（在线运行时）：
类似战场上的执行部队，负责在实时操作系统（RTOS）上，按照规划好的时间节拍发送/接收以太网报文，并监控从站状态。典型协议栈：SOEM

### 3.2 ESI：从站给配置工具的设备说明

从站设备描述文件（ESI，EtherCAT Slave Information）XML格式

```text
内容：
该从站设备的 Vendor ID（厂商ID）、Product Code（产品型号）、支持的通信模式、默认的 PDO（过程数据对象） 映射定义、以及内置的 CoE 对象字典（CANopen over EtherCAT）。
作用：
告诉主站配置工具，“我是什么设备，我有哪些参数和控制信号”。
```

### 3.3 离线搭建：导入 ESI，排列设备

```text
离线手动搭建：
在工具中导入各厂家的 ESI 文件。
工程师在工具里，手动把“伺服驱动器 1”、“IO 模块 2”拖入网络拓扑，按物理接线顺序排列好。
```

### 3.4 在线扫描：读取 SII，匹配 ESI

```text
在线扫描拓扑：
将主站电脑插上网线，直连物理设备。
配置工具直接发送命令，读取每个物理从站芯片外挂的 EEPROM 信息（SII，Slave Information Interface）。
配置工具 拿 读到的设备 ID 到 ESI 文件库中自动匹配，几秒钟内自动还原出当前的物理拓扑连线。
```

### 3.5 配置工具计算周期、PDO、FMMU、SM 和 DC

```text
配置工具内部完成的核心运算：
确定总线周期（如 1ms、500µs）。
PDO 编排与内存映射（FMMU 计算）：将每个从站要输入输出的数据（例如伺服的位置给定、实际位置、IO输入输出位），拼接成一整包连续的以太网数据帧（确定偏移量 offset 和字节长度）。
配置 SyncManager 和 DC（分布式时钟）参数。
```

<a id="chapter-04"></a>

## 4. 物理连接与本地接口：数据从哪里进、从哪里出

### 4.1 主站硬件链路：MAC、MII、PHY 与网线

```text
主站 → MAC → PHY → RJ45 → 网线 → RJ45 → PHY → ESC → MCU → 应用

通信控制器 MAC:
- 发送数据时把 CPU 的纯数据打包成以太网帧，接收数据时拆包，只能拆到数据链路层（14字节的EtherNet头、4字节的FCS）
```

```text
MII 接口:
- 数据链路层 和 物理层进行数据交互的接口，传数字信号。
物理层芯片PHY:
- 0、1数字信号，变成能在百米网线里稳定传输的“高频模拟电信号”，并在接收时反向还原。
隔离变压器:
- 电 → 磁 → 电，两边没有物理导线连在一起，磁场感应来传信号，电气隔离
RJ45口:
- 网线插座
双绞线
- 两根发 TX，两根收 RX
```

### 4.2 PHY 与链路建立

```text
ESC ──数字信号── PHY ──电信号── 网线
```

所以 PHY 的角色很简单：

> **物理层收发器。**

ESC 并不会直接把普通数字电平扔到几十米网线上。

PHY 负责 100BASE-TX 等物理层工作，链路是否建立也由 PHY 告诉 ESC。

PHY-网线-PHY，RJ45插上网线，两个PHY会硬件自动互相发送信号，建立连接，如果成功，PHY告诉ESC“端口另一边有设备，并且通信条件正常”

### 4.3 MII：ESC 与 PHY 之间的接口

```text
ESC → PHY：TX_D、TX_ENA
PHY → ESC：RX_D、RX_DV、RX_ERR、RX_CLK
PHY → ESC：LINK_MII（链路状态）
ESC ↔ PHY：MI_CLK / MI_DATA（管理 PHY）
```

### 4.4 发送 FIFO、接收 FIFO 与时钟关系

ET1100 为降低延时，省掉了发送 FIFO。

FIFO 可以理解成“临时排队区”。

普通方案：

```text
MAC → FIFO 暂存（收完整个包）→ 校验 FCS → 按 PHY 节奏发送
```

ET1100 更追求低延迟：

```text
ESC → 更直接地跟着 PHY 时钟发出去
```

好处：延时更小。

代价：ESC 和 PHY 时钟关系要求更严格。

> **ET1100 用牺牲对异常情况的容忍和缓冲能力，换更低的转发延迟。**

这里很容易误解。

ET1100 没有发送 FIFO，但接收侧仍有 RX FIFO（主站发来）。

原因很实际：

> **接收端和内部处理端的时钟不是完美一致，总要有一点缓冲来吸收时钟偏差。**

更准确的理解：

```text
TX：尽量直接，降低转发等待
RX：保留必要缓冲，吸收时钟差
```

### 4.5 EBUS：另一种物理连接方式

**EBUS = EtherCAT Bus Interface。**

```text
MII 路线： ESC → MII → PHY → 网线
EBUS路线： ESC → LVDS 差分线 → 另一个 ESC
```

EBUS 不需要外接普通以太网 PHY，因此元件更少、延时更低，但最远传输距离只有约 10 m。

所以可以粗略理解成：

```text
MII：更像标准以太网物理口，能接 PHY 和网线
EBUS：更像设备内部/模块之间的短距离 EtherCAT 连接
```

典型延时数量级：MII 端口传输延时大约数百 ns，EBUS 更低，约百 ns 量级。

```text
EBUS(n)-TX+、EBUS(n)-TX-
EBUS(n)-RX+、EBUS(n)-RX-
```

每两对 低压差分信号 之间有100欧姆电阻（吸收信号反射、保证100 Mbit/s差分传输的信号完整性）
RBIAS接11k欧姆接GND（内部就会产生一个固定大小的电流，然后芯片里其他电路都拿这个电流当“标尺”来工作）

### 4.6 Port：端口、回环与拓扑

ET1100 最多有 4 个物理通信端口：Port 0~3。

> 哪个端口收到帧，哪个就是当前入口；端口都具有 RX/TX。

**ESC 内部有固定的逻辑处理顺序和回环机制。**

```text
             Port 0
                │
Port 3 ─── [ ESC / EPU ] ─── Port 1
                │
             Port 2
```

- 线型；
- 分支；
- 回返路径。

```text
主站
 │
 ①
 ├── ② ── ③ ── ④
 │
 └── ⑤
      ├── ⑥ ── ⑦ ── ⑧
      │              ├── ⑨
      │              ├── ⑩
      │              └── ⑪
      │
      └── ⑫ ── ⑬ ── ⑭
数据走到④这条支路末端后，会通过 EtherCAT 的返回机制回到前面的分叉位置，然后再进入⑤这条支路。
Port 0 → ESC → Port 3 → Port 1 → Port 2 → Port 0
主站 → ① → ② → ③ → ④ → ⑤ → ⑥ → ⑦ → ⑧ → ⑨ → ⑩ → ⑪ → ⑫ → ⑬ → ⑭ → 返回主站
```

### 4.7 PDI：MCU 访问 ESC 的接口总称

**PDI = Physical Device Interface。**

```text
PDI
├─ 直接数字 IO
├─ SPI
└─ 并行微处理器接口 MCI
```

直接IO信号接口，无需MCU，最多32位引脚:
DPRAM数据接口，使用MCU，支持并行和串行两种方式。

ET1100的PDI接口类型和相关特性，由寄存器0x0140、0x0141配置：

- 接口选择，SPI
- AL状态更新方式
以下是1使能
- 增强的链接检测
- 分布时钟 同步输出单元
- 分布时钟 锁存输入单元

### 4.8 ET1100 的 IO、并行 MCI 与 SPI 接口

ET1100有128个引脚

```text
1.PDI 主机接口引脚，PDI[0]～PDI[39]
- IO
- MCI（微控制器接口，并行）16位同步/异步、8位同步/异步
ADR[0]~ADR[15]地址总线（ET1100寄存器或者DPRAM地址）、DATA[0]~DATA[15]数据总线
*BHE（Byte High Enable，*低电平有效），本次总线操作要不要操作DATA[15:8]
（8 位 MCI，那么 DATA[15:8] 根本不用，BHE 也就不用了）
异步 MCI 用的是 BUSY：
BUSY前一次写操作还没在 ET1100 内部彻底处理完时，下一次访问可能会被延迟，因此 BUSY 会持续更久
同步 MCI 用的是 TA：
同步接口不是一直给你一个 BUSY 状态，而是在这一笔操作完成之后给一个 TA 脉冲

- SPI
SPI_SEL片选、SPI_IRQ（ET1100给MCU的中断）
```

<a id="chapter-05"></a>

## 5. ESC 内部地址空间：寄存器与 DPRAM

### 5.1 为什么 ESC 里面会有地址

因为主站和 MCU 最终都需要操作 ESC 里的东西。

比如：

- 主站想知道链路有没有连接；
- 主站想让从站从 INIT 进入 PRE-OP；
- 主站想写 PDO；
- MCU 想读取主站刚写来的目标位置；
- MCU 想把实际位置写回去；
- 主站想配置 DC；
- 主站想读错误计数器。

如果每一种功能都设计一根专门的控制线，芯片会乱成一团。

ESC 采用更统一的方式：

> **把控制信息、状态信息、过程数据都放到一个“有地址的空间”里。**

于是大家统一做一件事：**读某地址、写某地址。**

### 5.2 64 KB 地址空间与实际 RAM 容量

ESC 地址空间最大 64 KB：

```text
0x0000
  │
  │  0x0000 ~ 0x0FFF
  │  ESC 寄存器区
  │  ——控制 ESC 自己
  │
0x1000
  │
  │  0x1000 ~ 0xFFFF
  │  过程数据 / Mailbox 等数据空间
  │
0xFFFF
```

低地址里的寄存器控制 ESC 本身：状态、端口、FMMU、SM、DC、EEPROM 等。

高地址的数据区主要放真正要交换的数据：Mailbox、PDO、用户过程数据。

注意：ET1100 实际过程数据 RAM 是 8 KB，不代表 `0x1000~0xFFFF` 每个地址都真的有物理 RAM。这里先把它当统一的 ESC 地址空间来理解。

### 5.3 寄存器：固定地址、固定含义

**寄存器 = 芯片内部一个有固定含义的小地址。**

比如：

```text
0x0130  AL Status
```

> “这里专门写当前从站是什么状态。”

又比如：

```text
0x0300 附近  接收错误计数
```

> 读这个地址，我就把对应端口的错误计数给你。

### 5.4 按功能认识寄存器地址

```text
0x0000 附近   ESC 基本信息：型号、FMMU 数量、SM 数量、RAM 大小
0x0010 附近   从站设置地址 / 地址别名
0x0100 附近   Data Link：端口和数据链路状态
0x0120 附近   AL State Machine：从站状态机
0x0140 附近   PDI：ESC ↔ MCU 接口设置
0x0200 附近   Interrupt / Event：事件和中断
0x0300 附近   Error Counter：链路错误计数
0x0400 附近   Watchdog：看门狗
0x0500 附近   EEPROM
0x0600 附近   FMMU
0x0800 附近   SyncManager
0x0900 附近   Distributed Clocks
0x1000 以后   Mailbox / PDO / 过程数据 RAM
```

### 5.5 DPRAM：ECAT 与 PDI 共同访问的数据区

**DPRAM = Dual-Port RAM，双端口 RAM。**

```text
一边：EtherCAT 网络侧 / ECAT
另一边：本地 MCU / PDI
```

两边都要碰同一片数据。

比如目标位置：

```text
主站通过 EtherCAT 写进去
            ↓
          DPRAM
            ↑
MCU 通过 PDI / SPI 读出来
```

反方向的实际位置：

```text
MCU 通过 PDI 写进去
            ↓
          DPRAM
            ↑
主站通过 EtherCAT 读走
```

> **两个人同时碰同一片 RAM，一定会遇到“谁先写、谁在读、会不会读到一半新一半旧”的问题。**

SM 就是为这个问题出现的。

<a id="chapter-06"></a>

## 6. 帧、寻址与 FMMU：主站怎样高效找到数据

### 6.1 带宽利用率：一辆货车经过 100 个设备的例子

在一定时间内，使用的带宽，占总带宽的比例。

- 主站只发一辆超长的大货车（一个完整的以太网帧），这辆车里装有 100 个设备的数据。
- 这辆车开过 1 号设备时，车根本不减速、不停车，
- 把车里属于自己的新指令抓下来，一把扔进“收件箱”，
  在从“发件箱”里抓起早就准备好的旧数据，塞回车厢对应的空位里
  （发件箱的数据是上次从收件箱拿到数据后，计算出的数据）
- ESC 芯片在改动车厢对应的空位的同时，计算并替换帧末尾的 4 字节 CRC
- ESC 芯片如果发现进来的包本来就是坏的，它会故意在这个包后面加一个错误标记
- 车子接着高速驶过 2 号、3 号……100 号设备；
- 最后开回主站时，主站一次性收齐了所有设备返回的最新数据。

### 6.2 协议栈与堆栈处理延时

MCU读信、解析信件、按照协议规范处理各种状态，这种专门处理通信协议的软件代码，就叫协议栈（Stack），

协议栈运行所消耗的时间，就叫堆栈处理延时。

### 6.3 Ethernet 帧、EtherCAT 头与 Datagram

```text
Ethernet II → EtherCAT frame header → 6 个 EtherCAT datagram
```

目的MAC 6、源MAC 6、类型 2
- 对应Ethernet II
EtherCAT头 2
- 对应EtherCAT frame header的“Length（后面“EtherCAT数据”的长度） / Reserved（保留位） / Type（0x01意味着后面的“EtherCAT数据”是命令）”一共2字节
EtherCAT数据（若干子报文）
- “6cmd sumlen 30”=”6 个 EtherCAT 子报文，6 个子报文的数据区加起来是 30 字节“，30+6*（10子报文头+2WKC）=102
FCS 4B 
- 网卡/驱动抓包前已经把 4 字节的 FCS 去掉

（6+6+2）+2+102=118

### 6.4 数据链路层与物理层的传输开销

```text
数据链路层	以太网报头+数据载荷+校验码
物理层网线上  前导码+数据链路层+帧间距
```

```text
前导码，8 字节

14字节EtherNet头

数据载荷，最少 46 字节，最多 1500 字节
- 哪怕只有 1 个字节的数据，也必须塞入填充物强行凑满 46 字节。
- 如果是TCP/IP，这46字节，还包括 IP 报头（至少 20 字节） 和 TCP 报头（至少20字节）/UDP 报头（至少8字节）

校验码，这个字段叫做FCS，4 字节
- CRC-32

帧间距，12 字节
- 发完这包数据后，线路必须空闲 12 个字节的时间
```

```text
1538 × 8 / 100M(bit/s) = 123.04us
84 × 8 / 100M(bit/s) = 6.72us
```

### 6.5 Datagram 的 Header、Data 与 WKC

```text
Cmd（1字节）、Idx（1字节）、Address（4字节）、[Len（11 bit） R（3 bit） C（1 bit） M（1 bit）]（2字节）、IRQ（2字节）
Data（Length 指定字节） 
WKC（2字节）
```

ADP（Address Position）、ADO（Address Offset）
32 位地址区，不一定是Slave Addr + Offset Addr，由 Command 决定这 32 位到底怎么解释

寄存器值和 PDO 通常没有额外“协议头”

```text
EtherCAT Datagram
┌──────────────────────────────┐
│ Cmd（1字节）、Idx（1字节）、Address（4字节）、[Len（11 bit） R（3 bit） C（1 bit） M（1 bit）]（2字节）、IRQ（2字节）
│ Data                         │
│ WKC                          │
└──────────────────────────────┘
```

Mailbox的Data有自己的结构，甚至CoE也有自己的结构

```text
EtherCAT Datagram
┌──────────────────────────────┐
│ Cmd（1字节）、Idx（1字节）、Address（4字节）、[Len（11 bit） R（3 bit） C（1 bit） M（1 bit）]（2字节）、IRQ（2字节）
│ Data                         │
│   ┌──────────────────────┐   
│   │ Mailbox Header       
│   │ CoE
│   └──────────────────────┘  
│ WKC                          │
└──────────────────────────────┘
```

### 6.6 Datagram Index 与 Interrupt

Index范围00~FF，到FF后会回卷，识别重复Datagram或丢失Datagram
主站协议栈：每生成一个 Datagram，Index 加 1，FF溢出后从00继续，一帧里面有6个Datagram，就会消耗6个Index
相邻的一发一回里面，Datagram Index 完全一样（上面的TexasInstrum_36:c5:78和下面的ba:3d:f6:36:c5:78的index一致）

Interrupt：帧每经过一个从站，从站都可以往这 16 bit 里“添事件”，但是主站只能知道“网络里有人产生了这种事件”，不一定能仅凭 IRQ 判断究竟是哪一个从站产生的

### 6.7 on-the-fly：帧经过时直接处理

普通交换机可以粗略想成：

```text
收到一帧 → 缓存 → 判断 → 再发下一跳
```

EtherCAT ESC 更强调：

```text
帧一边进来
   ↓
ESC 一边查看 Datagram
   ↓
地址命中时，现场读/写自己的数据
   ↓
同时继续向后转发
```

也就是 **on-the-fly**。

### 6.8 FCS/CRC：先看到写命令，再确认整帧是否正确

部分低地址寄存器写入具有缓存机制：

```text
帧经过
  │
  ├─ 先看到“要写寄存器”
  │
  ├─ 数据先进入临时缓存
  │
  └─ 等整帧 FCS 确认正确
         │
         ├─ 正确 → 真正提交到寄存器
         └─ 错误 → 丢掉临时结果
```

原因也很直观：

> **ESC 是 on-the-fly 的，它在帧还没结束时就已经看见前面的数据，但 FCS 在帧尾。**

也就是说：

> **“我已经看到了写命令”不等于“我已经知道整帧没坏”。**

所以部分寄存器先暂存，最后再提交。

而过程数据区的处理机制与这类寄存器不同。

这一点以后调试“为什么 某写操作 没有最终生效”时很重要。

### 6.9 寻址总览：按位置、按配置地址、按逻辑地址

```text
EtherCAT 寻址
│
├── ① 设备寻址（直接找某台设备）
│    ├── 顺序寻址：按设备连接顺序位置寻址
│    └── 设置寻址：主站给每个从站发一个编号，作为地址
│
└── ② 逻辑寻址（不强调是哪台设备，访问一片“逻辑内存）
```

一个靠位置，一个靠编号：
顺序寻址：找“第三排第二个人”
设置寻址：找“学号 1003 的同学”
把所有从站的数据想象成一整块连续的“大内存”，把分散在不同从站里的数据，映射到一个统一的逻辑地址空间：
主站可以读逻辑地址 0x0000～0x0007，至于”0x0000～0x0001 → 实际属于从站A、0x0002～0x0003 → 实际属于从站B“，对应关系，EtherCAT 从站内部通过 FMMU 等机制来处理
一个 EtherCAT 帧里可以塞很多个 EtherCAT Datagram，每个子报文可以采用不同的寻址方式

### 6.10 Auto Increment：启动时按位置找从站

刚上电时，主站甚至还不知道有哪些从站，更不知道它们之后应该叫什么地址。

所以先靠物理连接顺序找。

可以理解成：

> “从我这里数过去，第 1 个、第 2 个、第 3 个……”

典型用途：

- 扫描有多少从站；
- 读取从站信息；
- 给每个从站写入固定地址；

所以 Auto Increment 更像：

> **“开机点名工具。”**

第一次确定“有多少台从站”，用一个 BRD Datagram，通过 WKC 数出有多少台从站
SOEM初始化时发BRD读取ESC Type寄存器（ESC内部0x0000地址，ESC的类型），然后直接把返回的WKC当作slavecount，给Datagram的ADP一次写入0、-1、-2……1-slavecount，
从站①内部并没有保存自己是0，从站②内部并没有保存自己是-1，

APRW按位置找一台，ADP 到该从站时为 0 的从站先读后写

### 6.11 Configured Address：按主站分配的地址找从站

主站给每个从站写一个配置地址。

之后就能：

```text
FPRD：去配置地址为 1 的从站，读内部 0x0300
FPWR：去配置地址为 1 的从站，写内部 0x1000
```

此时 32 位 Address 可以理解成：

```text
ADP：哪台从站
ADO：那台从站里面哪个地址
```

例子：

```text
FPRD
ADP = 0x0001
ADO = 0x0300
```

就是：

> **“去 1 号从站，读它 ESC 地址 0x0300。”**

FPRW按配置地址找一台，读写

### 6.12 Logical Address：访问统一的逻辑过程映像

主站可以把多个从站想成一整块连续虚拟内存：

```text
逻辑地址空间

0x0000 ┌──────── 从站 A 的 2 Byte ────────┐
0x0002 ├──────── 从站 B 的 2 Byte ────────┤
0x0004 ├──────── 从站 C 的 2 Byte ────────┤
0x0006 └─────────────────────────────────┘
```

于是主站可以一次发：

```text
LRD Addr=0x0000 Len=6
```

不用拆成：

```text
读 A 两字节
读 B 两字节
读 C 两字节
```

那谁负责把“逻辑 0x0002”翻译成“B 从站 ESC 的某地址”？

答案就是 **FMMU。**

逻辑寻址没有16bit从站+16bit内部地址，只有32 bit Logical Address，4GB 逻辑（虚拟）地址空间，一片连续地址，实际数据可以分散在不同地方

### 6.13 Broadcast：相同 ADO、读取结果按位 OR

BRD 是广播，不要把”Slave Addr: 0x0001“当成BRD只访问1号从站

所有从站读取相同 ADO，并把读取值与 Datagram 当前 Data 做按位 OR，所以最后得到的是所有设备读取结果的逻辑 OR。
读所有从站 AL Status 0x0130，如果所有从站都处于OP，则OR后0x0008，如果某台还是SAFE-OP，则OR后0x000C

BRW所有从站读写

### 6.14 Read Multiple Write：一个从站读，多个从站写

ARMW一个位置从站读，其余相关从站写

FRMW一个配置地址从站读，其余相关从站写

- FRMW Len:8, Adp 0x1, Ado 0x910，Adp是把从站1内，ESC内的地址为0x910处的变量，读出来，就是Dc SysTime
- 数据包经过从站的时候，ESC看自己的配置地址，是否是ADP，
- 是的话，读自己ADO地址的变量，写入Datagram Data
- 不是的话，把Datagram Data，写到自己ADO地址
- （Beckhoff 明确说明 ARMW/FRMW 会周期性地把参考时钟的 System Time 分发给其他 DC slave，用来做 drift compensation漂移补偿）
- 只有一个从站，就只读，不写了，因为没人拿这个 Data 去同步

### 6.15 FMMU：把逻辑地址翻译到 ESC 内部地址

**FMMU = Fieldbus Memory Management Unit。**

它位于 ESC 内部。

主站启动时，把映射规则写进去。

运行时，EtherCAT Datagram 带着 Logical Address 经过 ESC，FMMU 判断：

> **“这个逻辑地址范围有没有属于我？”**

如果属于我，就把它映射到本 ESC 内部某片地址。

没有 FMMU：

```text
主站必须知道：
数据 A 在从站 1 的 0x1000
数据 B 在从站 2 的 0x1100
数据 C 在从站 5 的 0x1230
```

有 FMMU：

```text
主站只看：
逻辑 0x0000 ~ 0x001F 是我的过程映像
```

各个 ESC 自己负责：

```text
“其中哪些逻辑地址属于我，应该落到我的哪个内部地址。”
```

所以 FMMU 的本质是：

> **把“网络上统一的大内存视图”翻译成“每个 ESC 自己的小内存”。**

### 6.16 FMMU 保存的是映射规则

一条 FMMU 映射大概描述这些信息：

```text
逻辑起始地址
逻辑长度
逻辑起始/结束 bit
↓
映射到 ESC 内部哪个物理地址
从哪个 bit 开始
↓
这块地址允许读？允许写？
↓
是否激活
```

只要看到 `0x0600` 附近，就想：

> **“主站正在给 ESC 写地址翻译规则。”**

```text
FMMU 配置	值
数据逻辑起始地址	0x00014711
数据长度	2 Byte
数据逻辑起始位	3
数据逻辑终止位	0
从站物理内存起始地址	0x0F01
物理内存起始位	1
操作类型	2，写
激活	1
0x14711、0x14712，从 0x14711的bit3 到 0x14712的bit0，一共6个bit（跨了两个字节）
映射到
0x0F01开始的位置
1 = 读映射
2 = 写映射
3 = 读写映射
激活 = 1表示这条 FMMU 映射有效
```

如果逻辑地址，无法映射到任何一个从站，就不做处理

### 6.17 WKC：检查 Datagram 的参与处理情况

**WKC = Working Counter。**

> **WKC 是这一个 Datagram 在穿过整个 EtherCAT 网络过程中，符合条件的从站成功参与处理了多少。**

因此：

- 广播可能有多个从站参与；
- 逻辑寻址可能多个从站参与；
- 特殊 Read Multiple Write 也可能多个从站参与；
- WKC 完全可能大于 1，甚至大于 3。

主站一般会有一个 **Expected WKC**。

运行时比较：

```text
Actual WKC == Expected WKC
```

它更像：

> **“本来应该参与这次工作的那些从站，这一轮都参与了吗？”**

<a id="chapter-07"></a>

## 7. SyncManager：主站与 MCU 怎样交换同一片数据

### 7.1 SM 管理哪些内容

**SyncManager = ESC 内部的“共享内存交通管理员”。**

它把某一段 DPRAM 变成一块“受管理的缓冲区”。

你可以配置：

```text
从哪个地址开始
长度多大

谁写谁读（一个 SM 通道，同一时刻只有一个方向）
使用哪种工作模式：3个缓存区模式（一个给发送方写新数据、一个给接收方读、一个保持空闲）、单个缓存区模式
是否产生中断
是否启用看门狗
```

所以 SM 和普通 RAM 最大区别是：

> **不是“你想什么时候碰就什么时候碰”，而是 ESC 硬件在管理双方访问顺序。**

### 7.2 SM 同步的是内存访问

SM 的“同步”是：

> **同步两个访问者对同一片内存的读写。**

两个访问者就是：

```text
ECAT 侧：主站通过 EtherCAT 帧访问
PDI 侧：MCU 通过 SPI/MCI 访问
```

### 7.3 FMMU 与 SM 的分工

**FMMU 解决“地址在哪里”。**

**SM 解决“双方怎样安全交换这一片数据”。**

可以用仓库比喻：

```text
FMMU = 导航地图
       “逻辑 0x100 对应仓库第 3 排第 2 格”

SM   = 仓库管理员
       “现在谁能写、谁能读、什么时候算一批数据完成（读最新的一份、还是有标志位）”
```

### 7.4 Buffer 模式：旧数据可以丢，读取完整的新数据

典型用途：**PDO / 周期过程数据。**

比如目标速度每 1 ms 更新：

```text
1000 rpm
1010 rpm
1020 rpm
1030 rpm
```

如果 MCU 某次慢了一点，没有读到 1010，而直接读到 1020，很多实时控制场景并不要求把 1010 补回来。

你更在意：

> **现在最新目标是什么？**

所以 Buffer 模式强调：

- 随时可以产生新数据；
- 接收方总能读到一份完整、最新的数据；
- 太旧的版本可以被覆盖。

### 7.5 三缓冲：WRITE、READ、NEXT

没有“完成”信号，永远读最新的一份

假设只有一块内存：

```text
主站正在写 4 字节目标位置：
AA BB CC DD

MCU 正好同时读：
AA BB 旧CC 旧DD
```

就可能出现“撕裂数据”。

三缓冲的思想是让三种角色分开：

```text
一块给写方写
一块给读方读
一块作为下一块可交换的缓冲
```

写完、读完时，由 SM 在背后交换角色。

```
刚写完的这块 → 变成可读
原来可读的那块 → 变成空闲
原来空闲的那块 → 变成可写
```

所以你可以把它理解成：

> **SM 不一定把数据复制三遍，而是在管理“哪一块现在是 WRITE、哪一块是 READ、哪一块 NEXT”。**

### 7.6 Mailbox 模式：一封信不能被随便覆盖

发送方把一整批数据全部写完后，SM 硬件会置一个标志位，表示这批数据完整了

Mailbox 的需求完全不同。

比如：

> “把对象字典 0x1C13:00 写成 2。”

这是一次明确命令。

如果 MCU 还没处理，主站直接用下一封命令覆盖掉它，就出大事了。

所以 Mailbox 模式的规则更像：

```text
发送方写完整一封信
↓
缓存区锁住
↓
接收方必须读走
↓
确认这封信已经消费
↓
才允许下一封写入
```

这就是“握手”。

因此：

```text
Buffer 模式：追求“最新”
Mailbox 模式：追求“不丢”
```

这是区分两种 SM 模式最简单的方法。

### 7.7 SM0、SM1、SM2、SM3 的常见分工

典型 EtherCAT 从站常见：

```text
SM0：Master → Slave Mailbox
SM1：Slave → Master Mailbox
SM2：过程数据一个方向
SM3：过程数据另一个方向
```

但不要把这当“宇宙硬编码”。

真正准确的是：

> **SM 通道由主站配置；SII 会描述推荐/默认的 SM 类型和地址。**

在你当前抓包例子里：

```text
0x1000 ~ 0x13FF：SM0 / Mailbox Out
0x1400 ~ 0x17FF：SM1 / Mailbox In
Mailbox 长度 1024 B
```

主站写请求：

```text
FPWR → 0x1000 → SM0
```

从站回响应：

```text
MCU/从站协议栈把响应放入 SM1
主站再 FPRD 读 0x1400
```

这样一来，“Mailbox 地址”就不再是一个孤立概念：

> **Mailbox 本质上就是被 SM 以 Mailbox 模式管理的一段 ESC RAM。**

| 你这个从站        | 方向           | 谁写     | 谁读     | 用途                  |
| ----------------- | -------------- | -------- | -------- | --------------------- |
| `0x1000 ~ 0x13FF` | Master → Slave | 主站     | 从站程序 | **SM0 / Mailbox Out** |
| `0x1400 ~ 0x17FF` | Slave → Master | 从站程序 | 主站     | **SM1 / Mailbox In**  |

<a id="chapter-08"></a>

## 8. Mailbox、CoE、SDO 与 PDO：配置和运行各走哪条通道

### 8.1 Mailbox 是运输箱，CoE 等是箱内协议

这个层次一定要清晰：

```text
EtherCAT Datagram
└─ Data
   └─ Mailbox
      └─ CoE
         └─ SDO Request / Response
```

也可以装：

```text
Mailbox
├─ CoE
├─ FoE
├─ SoE
├─ EoE
└─ ...
```

所以：

> **Mailbox 是一个可靠传输容器；CoE/SoE/FoE/EoE 是装在里面的上层协议。**

CoE（CANopen over EtherCAT）：把 CANopen 的机制搬到 EtherCAT 上。对象字典访问、SDO 参数读写、PDO 映射。
SoE（Servo Drive over EtherCAT）主要用于读取/配置伺服参数。
EoE（Ethernet over EtherCAT）在 EtherCAT 网络里传输普通以太网帧，从而支持 TCP/IP、UDP、HTTP、FTP 等协议。
FoE（File over EtherCAT）下载固件、上传文件，不依赖 TCP/IP。

### 8.2 PDO 与 SDO 的业务内容

过程数据对象（PDO）

周期性、实时过程数据（伺服位置、IO 信号等），生产者消费者模型，无应答：

比如：伺服电机的目标位置、实际位置、当前扭矩。

服务数据对象（SDO）

偶尔用一下的配置数据，EtherCAT 里 SDO 走 Mailbox，也就是邮箱数据：

非周期，用来读写设备对象字典、参数配置，请求 - 应答模式

### 8.3 SDO 配置参数，PDO 交换过程数据

**CoE = CANopen over EtherCAT。**

它把 CANopen 的一些对象字典、SDO、PDO 等机制带到 EtherCAT。

但“PDO”这个词在学习时很容易产生错觉。

最实用的区分：

```text
配置阶段：
Mailbox → CoE → SDO
用途：读写参数、配置 PDO 映射

运行阶段：
EtherCAT 过程数据通道
用途：周期传目标位置、实际位置、状态字等
```

也就是说：

> **SDO 更像“设置菜单”；PDO 更像“实时数据流”。**

```text
PDO：数据的业务意义/映射内容
DPRAM：这些字节实际落在 ESC 内部的存储位置
```

PDO 不是“一块 RAM”。

### 8.4 抓包拆解：一次 CoE SDO 写请求

```text
EtherCAT Datagram
┌─────────────────────────────────────────────┐
│ EtherCAT Datagram Header       10 bytes    │
│ ├─ Cmd: FPWR (5)               1 byte       │
│ ├─ Idx: 0xa5                   1 byte       │
│ ├─ Adp: 0x0001             	 2 bytes      │
│ ├─ Ado: 0x1000             	 2 bytes      │ Mailbox
│ ├─ Len: 1024 (0x400)           2 bytes      │
│ └─ IRQ: 0x0000                 2 bytes      │
├─────────────────────────────────────────────┤
│ Data (1024 bytes)                           │
│ 有效内容16字节                                │
│ ┌─ Mailbox Header (6 bytes) ──────────────┐ │
│ │ Length: 10 (0x000A)                     │ │ CoE Header + SDO Request Payload
│ │ Address: 0x0001                         │ │
│ │ Type: 3 (CoE)                           │ │ Mailbox只是一个“运输箱”，必须告诉接收方：箱子里面装的是什么协议
│ │ Counter: 0                              │ │ 
│ └─────────────────────────────────────────┘ │
│ ┌─ Mailbox Protocol Payload (CoE SDO) ────┐ │
│ │ CoE Header (2 bytes)                    │ │
│ │   Number: 0               
│ │   Type: 2 (SDO Request)                 │ │ 后面这 8 个字节是一个 SDO 请求
│ │                                         │ │
│ │ SDO Request Payload (8 bytes)           │ │
│ │   Command: 0x2F (Initiate Download)     │ │
│ │   Index: 0x1C13                         │ │
│ │   SubIndex: 0x00                        │ │
│ │   Data: 0x02			  │ │
│ └─────────────────────────────────────────┘ │
│                                             │
│ ┌─ Padding填充 (1008 bytes) ───────────────┐ │
│ │ 00 00 00 00 ... (全为 0)                │ │
│ └─────────────────────────────────────────┘ │
├─────────────────────────────────────────────┤
│ WKC                              2 bytes    │
└─────────────────────────────────────────────┘
```

0x2F = 写 + expedited + 长度有效 + 1 byte
Download
- 主站把数据下载到从站，写参数
Upload
- 主站从从站上传数据，读参数

### 8.5 Mailbox Counter：重发时的计数示例

```text
Mailbox 请求 A   Counter = 1
Mailbox 请求 B   Counter = 2
Mailbox 请求 C   Counter = 3
如果主站发送 C 后没收到正常结果
Mailbox 重新请求 C   Counter = 3
从站看Counter还是3，判断这可能不是新的命令，而是上一封 Mailbox 消息重发

邮箱是请求应答模式
PDO是请求消费模式
```

### 8.6 请求与响应：FPWR 写入、FPRD 读回

PLC，把信件放进 ESC 邮箱，MCU 发现邮箱里有信，MCU 取走，处理
MCU，先把信放进 ESC 邮箱，ESC 告诉主站：有东西可以取，主站发送 EtherCAT 读命令，把信取回来

真正的 CoE SDO Response 不是在这个 FPWR 里面返回的，之后主站还会再发一个 FPRD 去读从站的返回邮箱。
0x1000、0x1400 是这个设备当前配置出来的地址

```text
① Texas
   FPWR
   Datagram Index = 0xA5
   ADO = 0x1000
   主站把：
   CoE SDO Request
   0x1C13:00 = 0x02
   写进从站邮箱
② ba
   FPWR
   Datagram Index = 0xA5
   WKC: 0 → 1
   表示从站接收了这次写操作,但这还不是 SDO Response
③ Texas
   FPRD
   Datagram Index = 0xB0
   ADO = 0x1400
   Data = 00 00 00 ...
   主站问：“从站，你的返回邮箱里现在有什么？”
④ ba
   FPRD
   Datagram Index = 0xB0
   ADO = 0x1400
   Data 被从站填写成：
   CoE SDO Response
   0x1C13:00
   这才是真正的 SDO Response
```

### 8.7 WKC 成功不等于 SDO 已经执行成功

例如主站发：

```text
FPWR
ADO = 0x1000
Data = Mailbox + CoE SDO Request
```

回来后 WKC 从 0 变 1，只说明：

> **“这个 EtherCAT 写操作成功写进了从站的 Mailbox。”**

它没有证明：

> “MCU 已经解析了 CoE，并且对象字典真的把参数改成功了。”

真正的 SDO Response 往往是之后：

```text
主站 FPRD 读取返回 Mailbox
↓
从 SM1 读出 CoE SDO Response
```

所以这里存在两层成功：

```text
第 1 层：EtherCAT 数据链路访问成功 → 看 WKC
第 2 层：应用协议命令执行成功       → 看 Mailbox/CoE Response
```

<a id="chapter-09"></a>

## 9. AL 状态机：从站开放到哪一步

### 9.1 INIT → PRE-OP → SAFE-OP → OP

**AL = Application Layer。**

这里最重要的是：

> **AL 状态机规定从站从刚上电，到允许 Mailbox，再到允许过程数据，最后到正常运行，要逐级开放。**

核心状态：

```text
INIT
  ↓
PRE-OP
  ↓
SAFE-OP
  ↓
OP
```

有些资料还会出现 BOOT，但主线先放一边。

### 9.2 四个状态分别允许做什么

**INIT：只允许搞基础设施**

像工厂刚通电。

此时重点是：

- ESC 是否正常；
- 地址是否能访问；
- EEPROM/SII 是否正常；
- 基本通信结构是否建立。

不要期待已经正常跑控制数据。

**PRE-OP：可以“谈配置”了**

PRE-OP 最重要的变化：**Mailbox 可以工作。**

所以：

```text
Mailbox
  ↓
CoE / FoE / SoE / EoE
  ↓
参数配置、PDO 映射、文件传输……
```

可以做很多“开机准备”，但还不是正常实时控制。

**SAFE-OP：过程数据通道基本建好了，但输出仍受保护**

可以把它理解成：

> **“生产线已经接好，传感器可以看，控制输出还没完全放权。”**

这一阶段通常用来确认 PDO、SM、FMMU 等过程数据配置没问题。

**OP：真正实时运行**

到这里才是：

```text
主站周期写目标值
从站周期回实际值
电机/IO 按过程数据持续运行
```

### 9.3 AL Control、AL Status 与 AL Status Code

核心寄存器：

```text
0x0120  AL Control
0x0130  AL Status
0x0134  AL Status Code
```

翻译成人话：

```text
主站：写 0x0120
      “我想让你去 SAFE-OP。”

从站：检查配置
      能去 → 更新 0x0130
      不能 → 0x0130 报错误，并把原因写进 0x0134

主站：读 0x0130
      如果有错，再读 0x0134
```

所以调试“为什么进不了 OP”时，最重要的思维不是：

> “OP 失败了，好玄学。”

而是：

> **“是哪一次状态迁移失败？AL Status Code 说什么？”**

AL 错误码：无效 Mailbox 配置、无效 SM 配置、没有有效输入/输出、无效看门狗配置、DC Sync/Latch 配置等。

你只要知道：

> **AL 状态机就是整个从站配置正确性的总验收员。**

### 9.4 Link 与 AL State 的区别

```text
Link：物理链路通不通
AL State：从站功能开放到哪一步
```

Link up 不代表已经 OP。

<a id="chapter-10"></a>

## 10. 事件、中断与看门狗：怎样通知 MCU、怎样发现超时

### 10.1 SyncManager 事件 → AL Event Request → PDI IRQ → MCU

```text
当 SM 的缓冲区状态发生了“完整写入”或“完整读出”的变化时，硬件把对应的位置 1，用来触发中断，通知 MCU

发送方把一整批数据全部写完后，SM 硬件会置一个标志位
主站把数据写进三缓存区后，SM 硬件并不会置一个“Buffer Full”位，但它会触发一个“SM2 写入事件”

SyncManager 事件
      ↓
AL Event Request
      ↓
PDI IRQ（如果这个事件没有被 Mask 掉）
      ↓
MCU 中断
```

### 10.2 Event 与 IRQ：事件发生不一定产生中断

**Event = 事情发生了。**

**IRQ = ESC 选择用一根中断线提醒 MCU。**

事件可以存在，但 IRQ 被屏蔽。

所以：

```text
事件发生
≠
一定产生 MCU 中断
```

要经过 Event Mask。

这和 MCU 自己内部外设的中断逻辑很像：

```text
状态位 → 中断使能 → NVIC/IRQ
```

### 10.3 事件来源与 Event Mask

典型来源有：

- AL Control 改变；
- DL 状态改变；
- SyncManager 状态/激活变化；
- Latch 事件；
- SYNC0 / SYNC1；
- 某个 SM 缓存区完成读写。

所以以后你看到 `0x0200~0x0223`，不要背每一 bit，先想到：

> **“这是 ESC 的事件路由器和中断屏蔽区。”**

### 10.4 Process Data Watchdog：监控网络侧过程输出

它关心：

> **主站有没有持续更新过程输出？**

例如正常情况下主站每 1 ms 发一次目标转矩。

突然网线断了，ESC 里的最后一个目标转矩还留着。

如果没有 Watchdog，电机可能继续拿旧目标运行。

所以过程数据看门狗的逻辑是：

```text
过程数据持续更新
→ 看门狗不断重新计时

很久没有新过程数据
→ 超时
→ 标记异常
→ 输出进入保护行为
```

它检查的是：

> **“网络给我的输出数据还活着吗？”**

### 10.5 PDI Watchdog：监控本地 MCU 访问

它关心：

> **本地 MCU 还在正常访问 ESC 吗？**

正常 PDI 读写会重新触发计时。

如果很久没 PDI 操作，可能表示：

- MCU 卡死；
- SPI/PDI 有问题；
- 应用程序跑飞。

所以：

```text
Process Data Watchdog：盯网络侧
PDI Watchdog：盯本地 MCU 侧
```

<a id="chapter-11"></a>

## 11. SII、EEPROM 与 ESI：设备信息和启动配置从哪里来

### 11.1 EEPROM 是存储器，SII 是从站信息

不要把两个词混成一个东西。

```text
EEPROM = 物理存储器
SII    = 按 EtherCAT 规定组织在 EEPROM 里的从站信息
```

**SII = Slave Information Interface。**

> **“从站写给主站和 ESC 的自我说明书。”**

### 11.2 ESC 上电读取 EEPROM

因为 ESC 上电后不能什么都不知道。

它需要知道一些基础配置，比如：

- PDI 用什么模式；
- 物理端口怎样配置；
- 某些同步设置；
- 站点别名；
- Mailbox / SM 的默认描述；
- 厂商和产品信息。

所以启动时：

```text
上电 / Reset
    ↓
ESC 自动读取 EEPROM
    ↓
装载必要配置
    ↓
EEPROM_Loaded 有效
    ↓
PDI / 过程数据区才进入可正常工作的状态
```

这就是为什么 `EEPROM_Loaded` 不只是“读完了一个文件”的提示，它关系到 ESC 是否完成基础初始化。

EEPROM_Loaded（上电读取SII EEPROM，配置PDI、端口，EEPROM_LOADED有效则表示已完成启动加载，MCU可以访问PDI了）
EEPROM_SIZE（大小/配置选择）

### 11.3 SII 中的身份、硬件、通信和业务描述

**A. “我是谁”**

```text
Vendor ID厂商ID
Product Code产品代码
Revision修订
```

**B. “我的硬件怎么接”**

```text
PDI 配置
端口相关配置
部分延时信息
```

**C. “我怎么通信”**

```text
Mailbox 地址/大小
SyncManager 描述
FMMU 描述
```

**D. “我支持什么业务”**

分类信息里可描述：

```text
TxPDO
RxPDO
DC
字符串/设备信息等
```

所以 SII 不是一个“参数表”这么简单，而是：

> **ESC 和主站启动时构建这台从站身份与通信结构的重要依据。**

### 11.4 SII 与 ESI XML 的分工

```text
SII：在从站 EEPROM 里，跟着硬件走
ESI：主站电脑/工程软件侧的 XML 描述文件
```

可以粗略类比：

```text
SII = 设备自己带的身份证和简历摘要
ESI = 主站软件拿到的更完整产品说明书
```

两边需要对应，否则主站可能无法按预期配置这台设备。

如果 EEPROM 显示CoE = No，即使你的从站实际上写了 CoE 协议，主站也不知道应该按照 CoE 的方式访问它。

### 11.5 EEPROM 的访问控制与读写时序

只抓住一个现实问题：

> **主站和 PDI/MCU 都可能想访问同一颗 EEPROM。**

所以必须有所有权：

```text
现在由 ECAT/主站控制？
还是由 PDI/MCU 控制？
```

不能两边同时发命令。

EEPROM 操作还有：

- Busy；
- Read / Write / Reload；
- 校验和错误；
- 无应答/命令错误；
- 写使能保护。

EEPROM 写入内部存储可能需要毫秒级时间，如果前一笔还没彻底完成就立刻继续写，EEPROM 可能暂时不应答，从而触发错误。

> **EEPROM 比 ESC 内部寄存器慢很多，不要把它当普通 RAM 连续狂写。**

<a id="chapter-12"></a>

## 12. DC 与 SYNC：不同从站怎样在同一时刻执行

### 12.1 DC 要解决什么问题

**DC = Distributed Clocks，分布式时钟。**

它要解决的问题不是：

> **“不同从站怎样在几乎同一个绝对时刻做动作？”**

假设主站每 1 ms 发一次 PDO：

```text
0 ms
1 ms
2 ms
3 ms
```

但数据到每个从站的时刻有传播延时，而且 MCU 自己也可能用不同定时器节奏处理：

```text
从站 A：1.1 ms 处理一次
从站 B：1.2 ms 处理一次
```

即使收到的是同一批目标位置，真正更新电机的时刻也会散开。

多轴控制就会抖。

DC 的思路：

> **先把所有支持 DC 的 ESC 内部时钟校准到同一个时间轴，再让每台 ESC 在“同一个约定时刻”产生 SYNC0/SYNC1。**

于是：

```text
数据什么时候到，可以有微小先后
真正什么时候执行，由共同时间表决定
```

### 12.2 自由运行、数据事件同步与 DC + SYNC

```text
自由运行模式：
从站不关心EtherCAT帧什么时候正好到，而是定时器中断一到，才从ESC取数据，计算
同步于数据输入或输出事件：
ESC更新PDO，产生事件，通知MCU处理，
但是不是所有ESC同时收到数据，有Delay
DC+SYNC：
DC让所有从站的时钟同步，	
ESC内部的DC时钟，到了设定时间，ESC自己产生SYNC0（同步信号/同步事件）（主站设定SYNC0周期），
各从站的 DC Clock 已经被校准到几乎相同，所以不同从站的 SYNC0 会几乎同时出现
```

### 12.3 DC 寄存器总表：当前时间、接收时间与补偿量

| 地址            | 长度  | 内容                              | 谁产生                                                       |
| --------------- | ----- | --------------------------------- | ------------------------------------------------------------ |
| `0x0900~0x0903` | 4 B   | Port 0 Receive Time               | ESC 本地硬件时钟锁存                                         |
| `0x0904~0x0907` | 4 B   | Port 1 Receive Time               | ESC                                                          |
| `0x0908~0x090B` | 4 B   | Port 2 Receive Time               | ESC                                                          |
| `0x090C~0x090F` | 4 B   | Port 3 Receive Time               | ESC                                                          |
|                 |       |                                   |                                                              |
| `0x0910~0x0917` | 4/8 B | System Time                       | ESC 内部真正的 Local Time是一直增加的<br />System Time = Local Time + System Time Offset |
| `0x0918~0x091F` | 4/8 B | ECAT Processing Unit Receive Time | 某个测量帧到达 EPU 时，对内部 Local Time 的一次硬件锁存快照。只读，不连续增长。 |
|                 |       |                                   |                                                              |
| `0x0920~0x0927` | 4/8 B | System Time Offset                | 主站算后写，写完后基本不动                                   |
| `0x0928~0x092B` | 4 B   | System Time Delay                 | 主站算后写，写完后基本不动                                   |
| `0x092C~0x092F` | 4 B   | System Time Difference            | ESC 控制环产生的当前剩余的同步误差，只读                     |

### 12.4 Offset、Delay、Difference、Drift

**Offset：两块表一开始差多少**

例如：

```text
参考时钟：100000 ns
从站时钟：100350 ns
```

起点差了约 350 ns。

这类初始时间偏差，就是 Offset 概念。

相关寄存器：

```text
0x0920 ~ 0x0927  System Time Offset
```

**Delay：报文从参考位置传播到本从站需要多久**

两台时钟就算一模一样，报文从 A 跑到 B 也不是瞬移。

所以必须知道传播延时。

相关寄存器：

```text
0x0928 ~ 0x092B  System Time Delay
```

没有 Delay，你无法判断：

> “B 比 A 晚 100 ns，是因为 B 的表慢，还是因为信号本身走了 100 ns？”

**Difference：现在还差多少**

初始补偿完不代表永远完美。

ESC 需要知道当前还有多少残余时间误差。

```text
System Time Difference
≈ 本从站 System Time
 - 参考 System Time
 - 传播 Delay
```

相关寄存器：

```text
0x092C ~ 0x092F
```

它更像“现在这一下还偏多少”。

**Drift：不是“差了多少”，而是“两块表走速不一样”**

例如：

```text
t1：差 10 ns
t2：差 10 ns
t3：差 10 ns
```

有 Difference，但差值不继续扩大，说明走速可能已经很一致。

另一种：

```text
t1：差 10 ns
t2：差 20 ns
t3：差 30 ns
```

说明从站时钟在持续越跑越偏。

这就是 Drift。

所以：

```text
Offset     = 初始位置差
Delay      = 传播时间
Difference = 当前剩余误差
Drift      = 走速误差
```

Drift
- 时钟漂移，两块晶振走快走慢不完全一样，本地 clock period 和 Reference Clock clock period 的差（速率误差，不是时间差）
- 0x0932 的 Speed Counter Difference

### 12.5 Port Receive Time 与拓扑

每个 DC ESC 可以在帧经过端口时自动锁存时间。

ET1100 中：

```text
0x0900~0x0903  Port 0 Receive Time
0x0904~0x0907  Port 1 Receive Time
0x0908~0x090B  Port 2 Receive Time
0x090C~0x090F  Port 3 Receive Time
```

这些不是“当前时间一直在跳”。

而是：

> **某次特定帧经过端口那一刻，ESC 硬件拍的一张时间照片。**

**为什么必须先知道哪个端口真的连着设备？**

假如一个从站有 4 个 Port，但实际只用了 Port0 和 Port1。

那 DC 计算传播路径时就必须知道：

```text
帧从 Port0 进
从 Port1 出
之后又从 Port1 回来
最后 Port0 返回
```

如果你连拓扑路径都不知道，就没法根据各端口时间戳算传播 Delay。

所以 DC 初始化不是单独“写几组时钟寄存器”这么简单。

它依赖：

```text
先扫描从站
↓
读端口 Link / Loop 状态
↓
恢复拓扑
↓
触发 Receive Time 锁存
↓
计算 Delay / Offset
```

这就是为什么“Port 状态”和“DC”原本看似无关，最后却连起来了。

### 12.6 去程与回程时间戳：Delay 和时间误差

```text
                  去程
参考时钟 A  ─────────────→ 从站 n ─────→ 后面的从站 ───┐
   T1                       T2                      │
                                                    │
   T4                       T3                      │
参考时钟 A  ←───────────── 从站 n ←───────────────────┘
                  回程
假如 参考时钟对应的从站、从站n，他们的ESC都是port0进，port1出，
则T1是ESC（参考时钟对应的从站）的port0察觉来数据的时候的Local Time值（Port 0 Receive Time）
则T2是ESC（从站n）的port0察觉来数据的时候的Local Time值（Port 0 Receive Time）
则T3是ESC（从站n）的port1察觉来数据的时候的Local Time值（Port 1 Receive Time）
则T4是ESC（参考时钟对应的从站）的port1察觉来数据的时候的Local Time值（Port 1 Receive Time）
((T4-T1)-(T3-T2))/2=delay
System Time（从站n） = Local硬件时钟（从站n） + Offset
   System Time（参考时钟对应的从站） = Local硬件时钟（参考时钟对应的从站）
   System Time Difference = System Time（从站n） - System Time（参考时钟对应的从站） - delay
Difference > 0自己跑快了，稍微减速
Difference < 0自己跑慢了，稍微加速
Difference = 0，同步完成
```

delay可以多测几次求平均（真实网络测量有一点抖动），算出来后，如果网络拓扑没变，传播延迟基本不会突然大幅变化
2.
测 offset 前必须先知道 delay
参考时钟在T1 = 100，delay是5ns，如果两块时钟完全一样，那么到从站n的时候，从站应该显示105

### 12.7 参考时钟与 FRMW 同步输入

主站连接到的第一个支持 DC 的从站作为参考时钟

在当前学习框架里可以这样记：

> **整条 DC 网络必须有一块“大家都对着它校准的表”。**

这个 ESC 的 System Time 就充当参考。

后续从站不需要每次都直接和主站 CPU 时钟硬比，而是以这条 EtherCAT 时间基准做同步。

这也解释了 FRMW / ARMW 这类 Read Multiple Write 命令为什么在 DC 里很适合：

```text
一个参考从站把自己的时间读进 Datagram Data
↓
同一 Datagram 继续经过后面的从站
↓
后面的从站使用这个参考时间做同步输入
```

一趟帧就能完成“一个提供参考，多个消费参考”。

```text
FRMW Len: 8, Adp 0x1, Ado 0x910
1. 读取地址为0x1（主站设置的地址）的从站内容（ESC的0x910地址处）
2. 写入地址不匹配的从站的ESC，但不是覆盖原0x910地址处的内容
而是计算 Δt = System Time（从站n） - System Time（参考时钟对应的从站） - delay 
但是 Δt 不会直接写入 System Time Difference（0x092C），而是作为误差样本，
ESC 内部经过 Difference Filter 后，在写入System Time Difference（0x092C）
```

### 12.8 DC 初始化：谁支持 DC

主站启动早期用顺序寻址读 ESC 特征信息。

目标：

> **哪些从站具有 DC？是 32 位还是 64 位时钟？**

### 12.9 DC 初始化：网络拓扑长什么样

主站读各从站 Data Link 状态，判断哪些 Port：

```text
建立了链路
打开/闭合
实际参与拓扑
```

于是拼出整条 EtherCAT 路径。

### 12.10 DC 初始化：各端口的接收时间

主站触发所有从站锁存 Port Receive Time。

然后逐个读回 `0x0900~0x090F`。

这相当于在全网放很多“高速摄像头”，记录同一帧到达每个路口的时刻。

### 12.11 DC 初始化：计算 Delay 与 Offset

主站结合：

```text
拓扑
+ 去程时间戳
+ 回程时间戳
```

计算：

```text
Propagation Delay
Initial Offset
```

然后写入相应 DC 寄存器。

### 12.12 DC 初始化：检查对齐并周期维护

主站把参考时钟 System Time 传播到其他 DC 从站。

随后反复查看 System Time Difference。

如果误差仍大，就继续校正。

运行以后也不能完全撒手，因为晶振存在 Drift，因此还需要周期维护。

### 12.13 SYNC0：按共同 DC 时间表产生事件

如果 DC 只让 ESC 内部寄存器的时间很接近，却没有办法让应用“在同一时刻做事”，那意义还没落地。

所以有 **SYNC0 / SYNC1**。

一句话：

> **SYNC0 / SYNC1 是 ESC 根据统一 DC 时间表自动产生的同步事件/信号。**

假设：

```text
EtherCAT 周期：1 ms
SYNC0 周期：   1 ms
执行偏移：     约 0.3 ms
```

那么可以是：

```text
0.0 ms  EtherCAT 帧经过，各 ESC 收到新一轮目标
0.3 ms  所有 DC 从站几乎同时产生 SYNC0
1.0 ms  下一帧经过
1.3 ms  下一次 SYNC0
2.0 ms  下一帧
2.3 ms  SYNC0
```

关键不是：

> “每个从站收到帧后自己计时 0.3 ms。”

而是：

> **主站事先把共同的 DC 时间表写给各从站；每台 ESC 看同一类时间基准，在指定绝对时刻产生 SYNC。**

这就是“同步”的本质。

所有人都可以看到墙上的时钟
开考前五分钟发卷子，但卷子不是同时发到所有人的手上，你收到卷子了可以看题，在心里算，但只能等考试正式开始，才能往卷子上写

### 12.14 SYNC0 与 SYNC1 的关系

核心关系是：

```text
SYNC0：基础周期事件
SYNC1：相对 SYNC0 再延迟一个配置时间
```

相关配置大致分为：

```text
启动时间
SYNC0 周期
SYNC1 相对 SYNC0 的时间
脉冲宽度
是否激活周期运行
是否输出 SYNC0 / SYNC1
```

第一遍不用背 `0x0990、0x09A0...` 每个地址。

只要你看到这一片寄存器，知道主站是在给 ESC 写：

> **“从几点开始，每隔多久敲一次同步钟；SYNC1 比 SYNC0 晚多久。”**

### 12.15 Latch：外部事件到来时锁存时间

这对概念也很适合放一起理解。

**SYNC：ESC 根据时间，向外产生一个事件。**

**Latch：外部事件进来，ESC 给它记下时间。**

所以：

```text
SYNC：时间 → 事件
Latch：事件 → 时间
```

这句话非常好记。

**Latch 用来干什么？**

例如外部传感器来了一个非常关键的边沿：

```text
编码器索引
高速光电触发
测量探针
外部同步输入
```

如果让 MCU 中断以后再读软件时钟，存在中断延迟和抖动。

Latch 的方式是：

```text
外部引脚边沿到达
↓
ESC 硬件立即锁存 DC System Time
↓
MCU 以后再慢慢读这个时间戳
```

这样记录的是“事件真正发生时”的硬件时间，而不是“CPU 什么时候终于处理到它”。

ET1100 有 Latch0 / Latch1，可记录上升沿和下降沿，并支持单次或连续模式。

<a id="chapter-13"></a>





## 13. 完整流程：从上电、配置到周期运行

下面以 **ET1100 + 外部 SII EEPROM + MCU，支持 CoE，并按需使用 DC** 的从站为例，把各模块串起来。不同主站可以交错执行扫描、配置和 DC 初始化；章节顺序用于理解依赖关系，不表示协议要求每个软件都逐项按此顺序执行。

先记住两条线：

```text
通信状态：INIT → PRE-OP → SAFE-OP → OP
功能开放：基础访问 → 邮箱配置 → 输入周期更新、输出保持安全状态 → 正常应用输出

DC 时钟：测量、初始补偿 → 同步控制环收敛 → 运行期间持续维护
DC 不需要等进入 OP 才开始对时。
```

### 13.1 阶段 1：上电，ESC 装载自己的硬件启动配置

```text
电源、时钟和复位条件满足，ET1100 退出复位
↓
ESC 通过 EEPROM 接口自动读取 SII 的 ESC Configuration Area
即 EEPROM 字地址 0x0000～0x0007，共 8 个字、16 字节
↓
检查配置区校验，装载 PDI、ESC 功能、SYNC 脉宽、站别名等初始值
↓
相应配置生效，PDI 具备可用条件
↓
PDI Operational 状态有效；提供 EEPROM_Loaded 的引脚/接口给出加载成功信号
↓
MCU 通过已配置的 PDI 初始化从站应用和协议栈
```

**“装载 SII 基础配置”具体装载什么？**

这里的 EEPROM 地址按 **16 位字** 编号，右侧 ESC 寄存器地址按 **字节** 编号，两种地址不要混淆。

| EEPROM 字地址 | 配置内容 | 对应 ESC 寄存器 | 具体作用 |
| --- | --- | --- | --- |
| `0x0000` 低字节 | PDI Control | `0x0140` | 选择 MCU/应用怎样访问 ESC，例如数字 IO、SPI、并行微控制器接口 |
| `0x0000` 高字节 | ESC Configuration | `0x0141` | 配置增强链路检测、DC SYNC/LATCH 单元使能等 ESC 功能，具体位按芯片手册解释 |
| `0x0001` | PDI Configuration | `0x0150～0x0151` | 配置所选 PDI 的接口选项，以及 SYNC/LATCH 引脚相关选项；不同 PDI 模式含义不同 |
| `0x0002` | Pulse Length of SYNC Signals | `0x0982～0x0983` | 设置 SYNC 信号的初始脉宽；这不是 SYNC0 周期 |
| `0x0003` | Extended PDI Configuration | `0x0152～0x0153` | 所选 ESC/PDI 支持的扩展配置；未实现或保留位按手册处理 |
| `0x0004` | Configured Station Alias | `0x0012～0x0013` | 装载站别名初值；不等于主站后续分配的 Configured Station Address |
| `0x0005～0x0006` | 保留字 | — | 按规范填写保留值 |
| `0x0007` | Checksum | — | 保存配置区校验信息，供 ESC 检查启动配置 |

### 13.2 阶段 2：PHY 建链，主站能够访问从站

```text
PHY ↔ 网线 ↔ 对端 PHY
↓
建立符合要求的以太网链路
↓
PHY 向 ESC 提供 Link 状态
↓
ESC 按端口状态执行转发或内部回环
↓
主站发送 EtherCAT 帧，访问从站 ESC
```

这段描述的是使用外部 PHY 的以太网端口；EBUS 等接口的物理实现不同。PHY 建链与其他初始化可能部分并行。

> **Link 表示通信链路建立，不表示从站已经进入 PRE-OP/OP，更不表示电机已经使能。**

### 13.3 阶段 3：扫描、识别设备，分配配置站地址

冷启动时，主站尚未建立可依赖的配置站地址关系，通常先广播探测，再按自动增量位置访问。

```text
例如 BRD 读取所有从站支持的 ESC 寄存器，通过 WKC 统计成功参与的从站
↓
使用 APRD / APWR 等按位置访问，逐站分配配置站地址
↓
读取 ESC 能力、端口状态、DC 能力和 SII 设备身份等信息
↓
建立实际设备/拓扑信息，与工程配置及设备描述匹配
```

读取设备信息和分配地址的具体先后可以交错：一旦分配了地址，就可以使用 FPRD/FPWR 继续读该站的信息。

主站关注的信息包括：

- ESC 类型和功能、端口 Link/Loop 状态、是否支持 DC。
- SII 中的 Vendor ID、Product Code、Revision 等身份信息。
- 配置所需的 Mailbox、过程数据和同步能力描述。

常见配置站地址寄存器为 `0x0010～0x0011`，与 `0x0012～0x0013` 的站别名区分。

**自动增量按报文经过各 EPU 的顺序数站。** 伺服 IN/OUT 接反时，实际扫描顺序可能改变。设备顺序不匹配能暴露拓扑异常，但仅凭顺序或 WKC 不能唯一断定是哪个端口接反，还要结合端口和拓扑信息。

### 13.4 阶段 4：INIT → PRE-OP，建立邮箱并确认状态迁移

以下针对支持 Mailbox/CoE 的从站；没有邮箱的简单从站不需要照搬邮箱步骤。

主站在 INIT 中按设备要求配置邮箱 SyncManager 的地址、长度、方向和工作模式，常见分工是：

```text
SM0：主站写，从站应用读，Master → Slave Mailbox
SM1：从站应用写，主站读，Slave → Master Mailbox
```

这些 SM 给对应的 ESC RAM 区域提供邮箱握手和访问控制。之后并不会自动跳到 PRE-OP，而是：

```text
主站写 AL Control（0x0120），请求 PRE-OP
↓
从站状态机检查邮箱配置和相关初始化条件
↓
成功：从站更新 AL Status（0x0130）
失败：报告错误，提供 AL Status Code（0x0134）
↓
主站读取状态，确认迁移是否成功
```

进入 PRE-OP 后，支持 CoE 的设备可以通过 `Mailbox → CoE → SDO` 进行参数访问。正常过程数据通信尚未开放。

**后面的 PRE-OP → SAFE-OP、SAFE-OP → OP，同样需要“请求 → 从站检查 → 状态反馈 → 主站确认”。** 对带 MCU 的从站，应用协议栈参与检查，不能认为 ESC 硬件自动批准所有迁移。

### 13.5 阶段 5：确定 PDO 内容，按需通过 SDO 配置

以 CoE 伺服为例：

```text
RxPDO：目标位置、控制字……（从站接收，即主站输出）
TxPDO：实际位置、状态字……（从站发送，即主站输入）
```

`Rx/Tx` 是从站视角。主站需要确定过程数据中包含哪些对象、顺序和长度，以及它们分配给哪个过程数据 SM。

- 设备支持可配置 PDO 映射时，可通过 SDO 修改映射和分配，例如设备支持的 `0x1600`、`0x1A00`、`0x1C12`、`0x1C13` 等对象。
- 默认映射已经满足要求时，可以沿用默认配置。
- 固定映射设备按其规定使用，不能假定都允许改写。

```text
SDO：通过邮箱读写参数、配置映射规则
PDO：运行时通过过程数据通道交换目标值和反馈值
```

SDO 操作要检查响应是否成功；底层 FPWR 的 WKC 正常，不等于参数请求已经被应用程序接受。不支持 CoE 的设备采用其支持的配置方式。

### 13.6 阶段 6：配置过程数据 SM 和 FMMU

对于常见的逻辑过程数据访问，主站根据确定的 PDO 布局配置：

| 模块 | 配置什么 | 负责什么 |
| --- | --- | --- |
| 过程数据 SM | ESC RAM 起始地址、长度、方向、缓冲模式、使能及相关事件/看门狗设置 | 管理 ECAT 与 PDI 两侧对过程数据缓冲区的访问，保证规定范围的数据一致性 |
| FMMU | 逻辑起始地址、长度/位范围、ESC 内部地址、读写方向、激活状态 | 把主站逻辑地址映射到本从站的 ESC 地址空间 |

SM2 常用于主站输出、SM3 常用于从站反馈，但实际编号和布局以设备描述及配置为准。

```text
EtherCAT 逻辑 Datagram
        │
        ▼
FMMU：地址匹配、逻辑地址映射
        │
        ▼
由 SM 管理访问和缓冲切换的过程数据区
        ▲
        │
     PDI / MCU
```

**SM 是访问和缓冲管理逻辑，不是数据写完 RAM 后再经过的独立搬运站。** FMMU 则用于逻辑寻址；物理地址寄存器访问不依赖 FMMU。

### 13.7 阶段 7：需要精确时间同步时，初始化并持续维护 DC

DC 不限于多轴，也用于同步采样、精确定时输出和事件时间戳。是否使用 DC、SYNC0 或 SYNC1，应匹配从站支持的同步模式。

**A. 初始化：测路程、对齐时间、让控制环收敛。**

```text
根据实际拓扑识别 DC 从站，选定参考时钟
通常选处理顺序中第一个支持 DC 的从站
↓
触发同一次测量帧的端口/EPU 接收时间锁存
倍福 ESC 示例：BWR 写 0x0900 的至少首字节
↓
等帧返回，读取 Port Receive Time（0x0900～0x090F）
以及需要的 EPU Receive Time（0x0918 起）
↓
结合拓扑和 ESC 内部延迟，计算各站相对参考时钟的传播 Delay
写入 System Time Delay（0x0928）
↓
建立参考时间基准，计算各站的初始时间偏移 Offset
写入 System Time Offset（0x0920 起）
↓
按 ESC 要求初始化同步控制环
例如写 Speed Counter Start（0x0930）复位内部滤波器
↓
主站发送 ARMW / FRMW 等同步 Datagram，分发参考时间
各从站 ESC 比较时间并调整本地数字时钟走速，逐渐收敛
↓
主站检查同步误差是否满足应用要求
```

主站还需要按设备模式配置同步启动时间、周期和相位/偏移，并启用对应功能。例如：

| 地址 | 作用 |
| --- | --- |
| `0x0990` 起 | 周期同步启动时间 |
| `0x09A0` | SYNC0 周期 |
| `0x09A4` | SYNC1 相对 SYNC0 的相关时序，按 ESC 模式解释 |
| `0x0981` | 周期同步及相应 SYNC 信号激活控制 |

配置与收敛的先后、何时开始输出 SYNC 可以因主站和从站实现不同而交错，但必须满足所选模式的状态迁移和运行条件。EEPROM 中预设的 SYNC 脉宽不能代替这些运行期配置。

**B. 是否先算好 Delay/Offset，等 OP 后再计算 Difference、调整时钟？**

更准确地说：

> **Delay/Offset 通常在初始化阶段测算并写好；时钟误差比较和走速校正也在启动阶段开始，不需要等 OP。进入 OP 后，继续维持同一套同步机制。**

在这里“从站作为参考时钟”的常见方案中，主站与 ESC 的分工如下：

| 谁 | 初始化时做什么 | 持续运行时做什么 |
| --- | --- | --- |
| 主站 | 识别拓扑、选择参考时钟、计算并写入 Delay/Offset、配置同步时间表 | 持续发同步 Datagram，读取/检查误差、状态和 WKC，并把自己的通信任务安排到合适的时间窗口 |
| 参考时钟 ESC | 建立网络共同的时间基准 | 在同步 Datagram 中提供参考 System Time |
| 其他 DC ESC | 接收补偿参数，启动本地同步控制环 | 收到参考时间后计算误差、滤波，并自动校正本地数字时钟走速 |

误差原理可以这样理解：

```text
本站 System Time = 本站 Local Time + 配置的 Offset

误差 e = 本站 System Time -（收到的参考 System Time + 配置的 Delay）

e > 0：本站时间领先，控制环使其适当放慢
e < 0：本站时间落后，控制环使其适当加快
```

`System Time Difference（0x092C）` 是 ESC 对同步误差处理、滤波后的诊断结果。**不是主站每周期读出 Difference 后，再逐台下发“加速/减速”命令；本地校正由 ESC 内部控制环完成。** 寄存器的正负编码与数值解释按 ESC 手册处理。

你抓包里的两种操作正好对应这两件事：

```text
FRMW，ADO=0x0910：分发参考时间，供各 DC ESC 持续校准
BRD， ADO=0x092C：读取同步误差汇总，供主站监视校准结果
```

BRD 把各站读数按位 OR 合并，不会单独返回每台的误差；它适合按规定的有效位/阈值检查整体同步情况。要定位哪台偏了，用 FPRD 按配置地址逐站读。读取误差的频率由主站实现决定，不是协议强制每周期都读。

**C. Delay/Offset 以后还会不会变？**

在拓扑、链路和时间基准稳定、设备没有复位的情况下，通常沿用初始化时写入的 Delay/Offset，周期性分发参考时间来补偿晶振漂移；每次误差比较仍会使用这两个补偿量，并不是以后忽略它们。

如果换线、增减从站、发生重连/复位，或者参考时间基准发生变化，主站应按情况重新测量、配置或重新同步；一些主站也会周期性重新测量传播延迟。因此不能理解成“两个寄存器写一次就永远不动”。

### 13.8 阶段 8：进入 SAFE-OP，已经可以周期交换过程数据

```text
主站完成当前模式所需的过程数据、同步等配置
↓
主站请求 SAFE-OP
↓
从站检查过程数据 SM、长度、必要的 DC 配置及设备自身条件
准备有效输入数据，满足条件后确认 SAFE-OP
↓
主站读取 AL Status，确认迁移结果
```

这个阶段通常是：

- 支持的邮箱通信继续可用。
- 输入过程数据已经可以周期更新并被主站读取。
- 主站可以发送输出过程数据，但从站通常保持实际输出在设备规定的安全状态。
- 配置使用 DC 时，按该模式继续提供同步报文和所需同步事件。

“安全状态”的具体行为由设备和配置规定；SAFE-OP 本身不等于功能安全认证，也不意味着主站证明了机械系统安全。

**对于有输出的从站，主站在请求 OP 前应先发送有效输出数据。** 这些初始输出通常应选择符合应用要求的安全值。周期过程数据交换可以在 SAFE-OP 已经开始，不能说进入 OP 才开始发周期帧。

### 13.9 阶段 9：进入 OP，按配置模式正常应用输出

```text
主站已经提供有效输出数据，并持续满足所选同步/看门狗等条件
↓
主站请求 OP
↓
从站检查应用条件，满足后确认 OP
↓
主站确认 AL Status，继续周期通信
↓
从站按运行模式正常应用主站输出，并发布输入反馈
```

**EtherCAT 的 OP 只说明通信应用状态，不等于伺服已使能。** 采用 CiA 402 的驱动器还需要设置运行模式、控制字并满足其驱动状态机、故障与联锁条件，才能执行运动命令。

运行时应分开理解两条并行的工作线。

**通信线：报文经过 ESC，硬件处理后继续转发。**

```text
主站准备输出过程映像
↓
组织一个或多个 EtherCAT 帧；每帧含一个或多个 Datagram
↓
各 ESC 按命令寻址：逻辑访问由 FMMU 匹配，物理访问按站地址/内部地址匹配
↓
在相应访问条件满足时，写入输出或读出已准备好的输入
SM 管理相关过程数据缓冲区，ESC 更新 WKC 并继续转发
↓
帧返回主站，主站检查 WKC、状态并使用收到的数据
```

不是所有 Datagram 都经过 FMMU；也不是每周期必须把所有数据放进同一个帧。

**应用线：MCU 按所选模式读取、计算和更新。**

| 同步模式 | 应用处理节拍 |
| --- | --- |
| 自由运行 | 从站自己的任务/定时器节拍 |
| SM 事件同步 | 配置的输入或输出过程数据事件 |
| DC 同步 | 配置的 SYNC0/SYNC1 及设备规定的处理时序 |

```text
MCU 在约定的任务/事件时刻，经 PDI 读取一致的过程数据
↓
执行应用算法，按设备实现更新控制环/输出
↓
采集并准备反馈，经 PDI 发布到输入过程数据缓冲区
↓
后续相应读访问取走当时已经准备好的有效反馈
```

事件是否产生 IRQ 取决于使能和屏蔽配置，应用也可能轮询。DC 模式可以在 SYNC 时取数据和运算，也可以提前取数、到点更新，不能固定写成“SM 中断取数 → 等 SYNC0 → 执行”。

**ESC 不会停住报文等待 MCU 算完或电机到位。** “下一周期读回反馈”是某种具体任务安排，不是协议保证的固定一周期延迟。

运行期间，主站继续监视 WKC、AL 状态、必要的 DC 误差；主站和从站也按各自职责处理通信超时、看门狗和应用故障。

### 13.10 从一个目标位置追到三轴反馈

下面选一个具体例子：三台伺服均已进入 OP，驱动器已使能、模式正确、DC 已同步；主站提前送目标，各驱动器按共同的 SYNC0 节拍更新控制。其他设备实现可以采用不同的数据准备/采样时序。

```text
【提前传输目标】
主站控制算法计算 A、B、C 三轴目标
↓
写入输出过程映像，组织 LWR 或 LRW 等 Datagram
↓
帧经过 A、B、C，各 ESC 把自己的目标送入 SM 管理的输出缓冲区
如果是 LRW，同时把当时已经准备好的输入反馈带回
↓
帧返回主站，检查 WKC；报文不会等待这次目标执行完成

【按共同时间执行】
目标数据已在各站要求的截止时间之前到达并可用
↓
共同 DC 时间到达预先配置的同步时刻
↓
A、B、C 的 ESC 在同步精度范围内产生对应 SYNC0 事件
↓
各驱动器按实现的任务时序，将目标用于自己的控制环
↓
电机按驱动器控制结果运动

【准备并读取反馈】
各驱动器按规定时刻采样实际位置等数据
↓
MCU 将一份有效反馈发布到 SM 管理的输入缓冲区
↓
之后的 LRD / LRW 读取当时可用的反馈
↓
主站得到反馈，用于后续控制和诊断
```

需要区分三个时刻：**目标到达 ESC 的时刻、控制环应用目标的时刻、实际位置采样的时刻。** 它们由设备模式和周期安排关联，不会因为同在一个 EtherCAT 系统就自动成为同一时刻。

DC 对齐的是共同时间基准及约定事件；MCU 执行、PWM 更新和机械响应仍有各自的延迟。“三个电机绝对同时动作”和“下一帧就返回目标到位结果”都不能作为协议保证。

本章核对依据：

- [Beckhoff ESC Technology，Version 2.3](https://download.beckhoff.com/download/Document/io/ethercat-development-products/ethercat_esc_datasheet_sec1_technology_2i3.pdf)：第 11.1 节/Table 40 为自动加载配置区；第 11.2.1 节为 `0x0110[0]` 与 EEPROM 错误状态；第 16.1 节为 `EEPROM_Loaded` 信号；第 9.1 节为 DC 初始化和持续补偿。
- [Beckhoff EtherCAT State Machine](https://infosys.beckhoff.com/content/1033/el6201/1036980875.html)：各状态开放功能及 SAFE-OP → OP 前的有效输出要求。

<a id="chapter-14"></a>

## 14. Wireshark：从抓包原理到 EtherCAT 字段

### 14.1 XDPro 通信时，Wireshark 怎样同时抓包

马路有一堆汽车，Wireshark只是站在路边看，不参与运输

```text
                    ┌────────→ Wireshark（Npcap复制一份）
                    │           
                    │        
                    │
RJ45 → 网卡 → Windows底层网络驱动 → Windows网络协议栈 → XDPro
wireshark不需要 占用IP、占用TCP端口、UDP端口
```

```text
网卡收到：
┌─────────────────────────────────┐
│ Ethernet                        │
│                                 │
│   ┌─────────────────────────┐   │
│   │ IPv4                    │   │
│   │                         │   │
│   │   ┌─────────────────┐   │   │
│   │   │ TCP             │   │   │
│   │   │                 │   │   │
│   │   │   Xnet数据      │   │   │
│   │   └─────────────────┘   │   │
│   └─────────────────────────┘   │
└─────────────────────────────────┘
正常软件：
网卡 → Windows帮忙拆 → 软件
Ethernet看目标 MAC，是不是给我的 → IPv4看目标 IP，是不是给我的 → TCP看目标端口，该交给哪个程序 → TCP排序、重传、拼接 → 把TCP里的“数据内容”交给 XDPro（Windows只解析到TCP传输层这里） → XDPro按照Xnet规则解释
Wireshark：
网卡 → Npcap复制原始帧 → Wireshark自己拆（Ethernet → IPv4 → TCP → 应用数据）
```

### 14.2 EtherCAT 过滤与当前抓包中的其他通信

Ethernet就是以太网，Xnet是信捷的PLC通信协议（读取PLC数据、写PLC数据、下载程序、监控状态）

```text
1.电脑 ↔ PLC Ethernet（XDPro）
2.PLC EtherCAT ↔ LFC3-AP EtherCAT（EtherCAT）
3.其他广播
10000个数据包可能9000个XDPro，1000个EtherCAT，
所以设置ether proto 0x88a4，过滤以后Wireshark窗口只剩EtherCAT
```

### 14.3 按六层顺序看抓包

推荐按 6 层顺序看。

```text
第 1 层：这是谁发出的？主站原包还是回来的包？
第 2 层：这一 Ethernet 帧里有几个 EtherCAT Datagram？
第 3 层：每个 Datagram 的 Cmd 是什么？
第 4 层：这个 Cmd 下，32 位 Address 应该怎样解释？
第 5 层：Data 是寄存器/PDO，还是里面又套 Mailbox？
第 6 层：WKC 是否符合预期？
```

如果 Data 是 Mailbox，再继续往里拆：

```text
Mailbox Type
↓
CoE / FoE / SoE / EoE
↓
如果 CoE：SDO Request / Response？
↓
Index / SubIndex / Command / Data
```

如果是 DC：

```text
是 Port Receive Time？
System Time？
Offset？
Delay？
Difference？
SYNC 配置？
```

这样抓包字段就不再是平铺的一百个名字，而是一棵树。

### 14.4 当前抓包中的主站发出帧与返回帧

TexasInstrum_36:c5:78是b8:3d:f6:36:c5:78，b8:3d:f6:36:c5:78 与 ba:3d:f6:36:c5:78，b8 | 0x02 = ba
TexasInstrum_36:c5:78 是主站发出的原始包，ba:3d:f6:36:c5:78 是经过 EtherCAT 网络后回来的包（MAC 地址第一个字节的 bit 1，也就是掩码 0x02）
从站是在帧经过的时候 on-the-fly 读写数据，而不是像普通 Ethernet switch 那样每经过一个节点就重新构造一个带自己 MAC 的 Ethernet 帧

### 14.5 ecat_mailbox 过滤看到哪些包

过滤使用ecat_mailbox
这个捕获到的 Ethernet 帧里，目前是否存在能够被 Wireshark 解析成 EtherCAT Mailbox Protocol 的有效内容。

```text
┌────────────────────────────────────┐
│ ① Texas  FPWR  写 Mailbox Request │ ← 有 Mailbox，显示
│ ② ba     FPWR  返回               │ ← Mailbox 数据还在，显示
│                                    │
│ ③ Texas  FPRD  读 Mailbox Response│ ← Data全0，不显示
│ ④ ba     FPRD  返回               │ ← Data被填入Response，显示
└────────────────────────────────────┘
```

### 14.6 Mailbox 抓包：写请求与读响应

```text
EtherCAT datagram: Cmd: 'FPWR' (5), Len: 1024, Adp 0x1, Ado 0x1000, Cnt 0
    Header
        Command: Configured address Physical Write (0x05)
        Index: 0xa5
        Slave Addr: 0x0001
        Offset Addr: 0x1000
        Length      : 1024 (0x400) - No Roundtrip - More Follows...
        Interrupt: 0x0000
    EtherCAT Mailbox Protocol:CoE SDO Req : 'Initiate Download' (1) Idx=0x1c13 Sub=0
        Header
            Length: 10
            Address: 0x0001
            .... ..00 = Priority: 0
            Type: CoE (CANopen over EtherCAT) (3)
            Counter: 0
        CoE
            Number: 0
            Type: SDO Req (2)
            SDO Req : 'Initiate Download' (1) Idx=0x1c13 Sub=0
                Initiate Download: 0x2f
                Index: 0x1c13
                SubIndex: 0x00
                Data: 0x02
	Data [...]: 00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
	Working Cnt: 0
EtherCAT datagram: Cmd: 'FPWR' (5), Len: 1024, Adp 0x1, Ado 0x1000, Cnt 1
    Header
        Command: Configured address Physical Write (0x05)
        Index: 0xa5
        Slave Addr: 0x0001
        Offset Addr: 0x1000
        Length      : 1024 (0x400) - No Roundtrip - More Follows...
        Interrupt: 0x0000
    EtherCAT Mailbox Protocol:CoE SDO Req : 'Initiate Download' (1) Idx=0x1c13 Sub=0
        Header
            Length: 10
            Address: 0x0001
            .... ..00 = Priority: 0
            Type: CoE (CANopen over EtherCAT) (3)
            Counter: 0
        CoE
            Number: 0
            Type: SDO Req (2)
            SDO Req : 'Initiate Download' (1) Idx=0x1c13 Sub=0
                Initiate Download: 0x2f
                Index: 0x1c13
                SubIndex: 0x00
                Data: 0x02
    Data [...]: 00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
    Working Cnt: 1

EtherCAT datagram: Cmd: 'FPRD' (4), Len: 1024, Adp 0x1, Ado 0x1400, Cnt 0
  Header
	Command: Configured address Physical Read (0x04)
	Index: 0xb0
	Slave Addr: 0x0001
	Offset Addr: 0x1400
	Length    : 1024 (0x400) - No Roundtrip - More Follows...
	Interrupt: 0x0000
        上面完全一样
	Data [...]: 00000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
	Working Cnt: 0
EtherCAT datagram: Cmd: 'FPRD' (4), Len: 1024, Adp 0x1, Ado 0x1400, Cnt 1
    Header
        Command: Configured address Physical Read (0x04)
        Index: 0xb0
        Slave Addr: 0x0001
        Offset Addr: 0x1400
        Length      : 1024 (0x400) - No Roundtrip - More Follows...
        Interrupt: 0x0000
    上面完全一样
    EtherCAT Mailbox Protocol:CoE
        Header
            Length: 10
            Address: 0x0001
            .... ..00 = Priority: 0
            Type: CoE (CANopen over EtherCAT) (3)
            Counter: 4
        CoE
            Number: 0
            Type: SDO Res (3)
            SDO Res: Scs 3
            Index: 0x1c13
            SubIndex: 0x00	
    Data [...]: 000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
    Working Cnt: 1
```

### 14.7 周期抓包：DC、AL 状态和过程数据

检查DC → 同步DC时间 → 检查从站状态 → 读取过程数据 → 写入过程数据 → 再检查从站状态

```text
EtherCAT datagram: Cmd: 'BRD' (7), Len: 4, Adp 0x1, Ado 0x92c, Cnt 1
  Header
    Command: Broadcast Read (0x07)
    Index: 0x1d
    Slave Addr: 0x0001
    Offset Addr: 0x092c
    Length      : 4 (0x4) - No Roundtrip - More Follows...
    Interrupt: 0x0000
  DC CtrlError (0x92c): 0x00 00 00 00
  Working Cnt: 1

EtherCAT datagram: Cmd: 'FRMW' (14), 	, Cnt 1
  Header
    Command: Configured Address Physical Read Multiple Write (0x0e)
    Index: 0x1e
    Slave Addr: 0x0001
    Offset Addr: 0x0910
    Length      : 8 (0x8) - No Roundtrip - More Follows...
    Interrupt: 0x0000
  DC SysTime (0x910): 0x00000001bb84cc6b
  DC SysTime L (0x910): 0xbb84cc6b
  DC SysTime H (0x914): 0x00000001
  Working Cnt: 1
EtherCAT datagram: Cmd: 'BRD' (7), Len: 4, Adp 0x1, Ado 0x130, Cnt 1
  Header
    Command: Broadcast Read (0x07)
    Index: 0x1f
    Slave Addr: 0x0001
    Offset Addr: 0x0130
    Length      : 4 (0x4) - No Roundtrip - More Follows...
    Interrupt: 0x0000
  AL Status (0x130): 0x0008, AL Status: OP
    .... .... .... 1000 = AL Status: OP (0x8)
    .... .... ...0 .... = Error: False
    .... .... ..0. .... = Id: False
  Working Cnt: 1
EtherCAT datagram: Cmd: 'LRD' (10), Len: 6, Addr 0x0, Cnt 1
  Header
    Command: Logical Read (0x0a)
    Index: 0x20
    Log Addr: 0x00000000
    Length      : 6 (0x6) - No Roundtrip - More Follows...
    Interrupt: 0x0000
  Data: 00 00 00 00 00 00
  Working Cnt: 1
EtherCAT datagram: Cmd: 'LWR' (11), Len: 6, Addr 0x6, Cnt 1
  Header
    Command: Logical Write (0x0b)
    Index: 0x21
    Log Addr: 0x00000006
    Length      : 6 (0x6) - No Roundtrip - More Follows...
    Interrupt: 0x0000
  Data: 000000000000
  Working Cnt: 1
EtherCAT datagram: Cmd: 'BRD' (7), Len: 2, Adp 0x1, Ado 0x130, Cnt 1
  Header
    Command: Broadcast Read (0x07)
    Index: 0x22
    Slave Addr: 0x0001
    Offset Addr: 0x0130
    Length      : 2 (0x2) - No Roundtrip - Last Sub Command
    Interrupt: 0x0000
  AL Status (0x130): 0x0008, AL Status: OP
    .... .... .... 1000 = AL Status: OP (0x8)
    .... .... ...0 .... = Error: False
    .... .... ..0. .... = Id: False
  Working Cnt: 1
```

### 14.8 FPRD 抓包：读取端口接收错误计数

4. FPRD（Configured Address Physical Read按“已配置从站地址”读一个从站）
- Len:4,Adp 0x1,Ado 0x300 去配置地址为 1 的那个从站，从它 ESC 内部地址 0x300 开始，读取 4 个字节
- 0x0300 ~ 0x0307 是 RX Error Counter 区域。每个端口占两个字节CRC 0 = 0x0000，CRC 1 = 0x0000，从站1的 Port 0 和 Port 1 当前都没有 RX 错误
- port0和port1都既有TX又有RX，A 的 TX -> B 的 RX，真正能检测错误的是接收端，A 把数据发出去了，并不知道几十米外 B 到底收到的是不是正确波形

### 14.9 查看电脑网口的 MAC 地址

右键 以太网 → 状态 → 详细信息 → 在弹出的小窗口里找 "物理地址"

<a id="chapter-15"></a>


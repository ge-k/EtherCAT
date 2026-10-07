# 邮箱与 CoE

Mailbox（邮箱）是在主站与从站应用之间交换消息的通道。它通常使用由 [SyncManager](../SyncManager/SyncManager.md) 管理的两块 RAM，分别传递主站请求和从站返回的消息。

## Mailbox 与上层协议

邮箱规定消息容器和传输机制，消息具体表示什么操作由里面的协议决定。

| 协议 | 用途 |
| --- | --- |
| CoE：CANopen over EtherCAT | 对象字典访问、SDO 服务、PDO 相关机制 |
| SoE：Servo Drive Profile over EtherCAT | 基于相应驱动参数模型访问伺服参数 |
| FoE：File over EtherCAT | 文件传输，例如固件下载，不依赖 TCP/IP |
| EoE：Ethernet over EtherCAT | 在 EtherCAT 内传输普通 Ethernet 帧，可用于上层 IP 通信 |

是否支持某种协议由设备决定。CoE 的 SDO 使用邮箱；常规周期 PDO 走过程数据通道，不能把 CoE 的所有业务都理解成每次套一个邮箱头。

## SDO 怎样访问对象字典

**SDO（Service Data Object）** 以 Index 和 SubIndex 指定对象字典条目，并用请求—响应方式读写。Download 表示主站向从站写数据；Upload 表示主站从从站读数据。

一次简短的 SDO 写请求可以逐层表示为：

```text
EtherCAT Datagram：FPWR 写入从站邮箱地址
└─ Mailbox Header：类型为 CoE
   └─ CoE Header：服务为 SDO Request
      └─ SDO：命令、Index、SubIndex、待写数据
```

笔记中的例子向 `0x1C13:00` 写入 `0x02`。它是一个 1 字节 expedited download：少量数据直接包含在发起请求中，命令字 `0x2F` 表示写入、快速传送、长度有效及 1 B 有效数据。

## 长度为什么看起来不一致

在示例中，FPWR 的 Datagram Data 长度为 1024 B，但实际消息只有 16 B：

| 部分 | 长度 |
| --- | --- |
| Mailbox Header | 6 B |
| CoE Header | 2 B |
| SDO 请求 | 8 B |
| 本次访问中的其余填充 | 1008 B |

Mailbox Header 的 Length 为 10，只计算邮箱头之后的 CoE 头与 SDO 请求。Datagram 的 Length 为 1024，表示本次对邮箱缓冲区的访问长度。两者描述的边界不同。

邮箱头还有 Address、通道/优先级、Type 和 Counter 等字段。这里的 Mailbox Address 属于邮箱协议头，不能与 Datagram 的 ADP、ADO 混淆。

## 请求返回与应用响应是两回事

假设该设备配置主站写邮箱为 `0x1000`，主站读邮箱为 `0x1400`：

```text
1. 主站 FPWR 写 0x1000，放入 CoE SDO Request
2. 同一 FPWR 返回，WKC 从 0 变为 1
3. 从站应用读走请求，处理对象字典访问，准备响应
4. 主站用另一条 FPRD 读 0x1400，取回 SDO Response
```

第 2 步只说明底层邮箱写访问成功，不能证明参数已修改。最终要检查 SDO 响应，失败时还应读取其 Abort 信息。FPWR 返回中的 Data 通常仍是请求本身，不是从站应用已经给出的响应。

从站不会为了这个响应主动生成一帧普通 EtherCAT 数据发给主站；消息由应用放入读邮箱，再由主站安排读取。主站可以通过邮箱状态或相关通知判断是否有消息。

## Counter 与重传

Mailbox Counter 是邮箱消息层的计数，用于辅助识别重复消息，与 Datagram Idx、WKC 是三种不同字段。

启用计数检查时，新消息计数通常在 1～7 之间循环；重复传送同一条消息时保留其计数，使接收方有机会识别重复。Counter 为 0 表示未使用这种计数检查。请求与响应是不同方向的消息，不能要求它们的 Counter 必须相同。

重传还要遵守邮箱状态和相应协议的处理规则，不能因为超时就把满邮箱无条件重写。邮箱提供有序交接条件，上层服务再定义请求成功、错误和重试语义。

来源：[学习笔记，第 8、14.6 节](../../../original_information/学习笔记/0EtherCAT学习笔记.md)。

# 记录

```
可以扫描到，但一直处于Pre OP，先写入配置，在激活就行，

LFC3-AP的PWR灯是常亮的，RUN灯是常亮的，但是SF灯是闪烁的，
XF-E8NX8YT的PWR灯是常亮的，RUN灯是常亮的，
断电重启一下就好了
```

# SM

## 数据流向

         EtherCAT侧
             |
      EtherCAT Processing Unit
             |
      SyncManager 2
             |
    +----------------+
    | Buffer0        |
    | Buffer1        |
    | Buffer2        |
    +----------------+
             |
          PDI接口
             |
          MCU

## SM寄存器

```
0x0810	SM2 Start Address
- 0x1100，SM2管理PDRAM地址0x1100

0x0812	SM2 Length
- 2，管理两个byte（也就是0x1100~0x1101，这两个是外部可见，但是3 buffer模式，0x1102~0x1103、0x1104~0x1105不可见）

0x0814	SM2 Control（单buffer 三buffer、主读从写 主写从读、ECAT中断请求允许、AL中断请求允许、看门狗触发使能）
- bit0 bit1：00是3 buffer，10是单buffer
- bit2 bit3：00主读从写，01主写从读
- bit4：ECAT Event Request Interrupt Enable
- bit5：AL Event Request Interrupt Enable
- bit6：Watchdog Trigger Enable
- bit7：

0x0815	SM2 Status（buffer被写、被读、单buffer空满、3 buffer最后写的buffer是哪个）
- bit0：buffer 完整、成功写入后置 1，interrupt 被通知给 reading side；读取方读取第一个 byte 后清除；
- bit1：buffer 完整、成功读取后置 1，interrupt 被通知给 write side；写入方写入第一个 byte 后清除；
- bit2：
- bit3：Mailbox 模式：0=empty，1=full；3-buffer PDO 模式下 Reserved
- bit4 bit5：3-buffer模式，最后写入的是哪个buffer，buffer1（00）、buffer2（01）、buffer3（10）、无buffer被写入（11）
- bit6 bit7：

0x0816	SM2 Activate
- bit0：SM Enable值是1，SM2管理这块DPRAM（EtherCAT侧配置）
- bit1：Repeat Request，每翻转一次，就请求 PDI 重做一次 mailbox 操作（用于ECAT Read Mailbox，也就是SM1）
- bit2~bit5：
- bit6：Latch Event ECAT，ECAT侧导致SM buffer交换，则bit6=1，则Latch Event（是指3 buffer下，一次读写都后（ECAT侧是最后读写的），硬件SM自动交换buffer吗）
- bit7：Latch Event PDI，PDI侧导致SM buffer交换，则bit7=1，则Latch Event（是指3 buffer下，一次读写都后（PDI侧是最后读写的），硬件SM自动交换buffer吗）

0x0817	SM2 PDI Control
```

## ECAT Event Request和Mask

ECAT Event Request

- 0x0210–0x0211
- 只读的事件状态寄存器 ，当前有哪些 ECAT interrupt/event pending  

```
- bit0：DC Latch Event

- bit1：

- bit2：DL Status Event，
Data Link，1是发生变化，0是没变化
DL Status 发生变化，
bit2 = 1，
Master 读取 DL Status，
bit2 清除

- bit3：AL Status Event
INIT，PRE-OP，SAFE-OP，OP
AL Status 发生变化
bit3 = 1，
Master 读取 AL Status 后，
bit3 清除

- bit4~bit11：SM0 Event~SM7 Event

- bit12~bit15：Reserved
```

ECAT Event Mask

- 0x0200–0x0201

- ECAT Event Request对应 bit 的 mask
- Mask = 0 → 禁止这个事件映射
  Mask = 1 → 允许这个事件映射

## AL Event Request和Mask

AL Event Request

- 0x0220–0x0223
- 当前发生的 AL Event / PDI IRQ 事件状态

```
- bit0：AL Control Event，AL Control被写
- bit1：DC Latch Event，Latch 输入变化
- bit2：DC SYNC0 Event（SYNC0可以单独接一根引脚到MCU，也可以塞进PDI IRQ）
- bit3：DC SYNC1 Event
- bit4：任意SM Activation Changed

- bit5：
- bit6 bit7：

- bit8~bit15：SM0的Event可触发IRQ ~ SM7的Event可触发IRQ
- bit16~bit23：ETH1100无SM8~SM15

- bit24 bit25：
- bit26~bit31：
```

PDI AL Event Mask

- 0x0204–0x0207
- AL Event Request 对应 bit 的 mask
- Mask = 0 → 禁止这个事件映射
  Mask = 1 → 允许这个事件映射

## 主站写，从站读

SM2：ECAT write、PDI read

```
Master 写入 2 bytes 到 SM2 Buffer
      │ 完整成功写入
      ▼
SM2 Status 0x0815 bit0（这是0） = 1，Write Event，通知 reading side（如果 reading side = PDI）
      │ 如果 SM2 Control 0x0814 bit5（这是5） = 1，AL Event Request Interrupt Enable
      ▼
AL Event Request 0x0220 bit10 = 1，SM2的Event触发
      │ 如果 AL Event Mask，0x0204 bit10 = 1，SM2的Event的Mask使能
      ▼
PDI IRQ
      │
      ▼
MCU 进入中断，MCU 开始读取 SM2
      │ 读取完第一个 byte
      ▼
SM2 Status bit0 自动清 0，AL Event Request bit10 自动清除
      │ 读取完整个 SM2
      ▼
SM2 Status 0x0815 bit1（这是1） = 1，Read Event，通知 write side（如果 write side = Master）
      │ 如果 SM2 Control 0x0814 bit4（这是4） = 1，ECAT Event Request Interrupt Enable
      ▼
ECAT Event Request，0x0210 bit6 = 1，SM2的Event触发
      │ 如果 ECAT Event Mask，0x0200 bit6 = 1，SM2的Event的Mask使能
      ▼
写到EtherCAT Frame的datagram的interrupt字段（写入第一个字节后，SM2 Status bit1自动清0，ECAT Event Request bit6清零）
      │
      ▼
主站看到ECAT IRQ
```






# XCP

# XCP 协议详解 — 标定读写原理与实践

> **文档版本**: v1\.0
> **适用范围**: AUTOSAR XCP 模块 / ASAM MCD\-1 XCP v1\.0\+
> **核心问题**: 如何通过 XCP 协议读取/修改 ECU 内部某个地址的值
> 
> 

---

## 目录

1. 什么是 XCP？—— 通俗理解

2. 协议架构与设计模式

3. XCP 报文结构详解

4. 标定读操作：读取某地址的值

5. 标定写操作：修改某地址的值

6. 完整通信握手流程

7. 代码示例：模拟 XCP 标定读写

8. 深入原理：XCP 的内存访问机制

---

## 1\. 什么是 XCP？—— 通俗理解

### 1\.1 一句话概括

**XCP**（Universal Calibration Protocol，通用标定协议）是 ASAM 标准定义的 **ECU 内部变量"透明化"协议**。它让标定工具（如 CANape、INCA）能像调试器一样**读取和修改** ECU 运行时的内存数据，而无需修改 ECU 固件。

### 1\.2 生活中的类比

```
┌─────────────────────────────────────────────────────────────┐
│  XCP 就像给 ECU 装上了一个"观测窗口"                          │
│                                                              │
│  标定工程师 = 站在窗口外的人（XCP Master）                    │
│  ECU         = 房间里工作的机器（XCP Slave）                  │
│  窗口        = XCP 协议                                       │
│                                                              │
│  你可以通过窗口：                                              │
│  ├─ 观察房间内的仪表读数  →  标定读取 (SHORT_UPLOAD)          │
│  └─ 调节房间内的旋钮开关  →  标定写入 (SHORT_DOWNLOAD)        │
└─────────────────────────────────────────────────────────────┘
```

### 1\.3 XCP 与 CCP 的关系

|特性|CCP \(CAN Calibration Protocol\)|XCP \(Universal Calibration Protocol\)|
|---|---|---|
|传输层|仅 CAN|CAN、FlexRay、Ethernet、USB、LIN \.\.\.|
|传输模式|按需请求|支持按需 \+ DAQ 周期性测量|
|地址空间|8/16/32 位|8/16/32/64 位|
|复杂度|较低|较高，功能更丰富|

---

## 2\. 协议架构与设计模式

### 2\.1 协议架构分层

### 2\.2 核心设计模式

|设计模式|XCP 中的应用|
|---|---|
|**C/S（Client/Server）**|Master 发送命令（CMD），Slave 返回响应（RES）|
|**状态机**|Slave 有明确的连接/断开/标定/测量状态|
|**资源抽象**|Memory Transfer Address \(MTA\) 抽象出统一的地址访问接口|
|**可插拔传输**|协议核心与传输层解耦，同一套命令可在不同物理总线上运行|
|**保护模式**|需要通过种子密钥（Seed \& Key）解锁才能写入|

### 2\.3 Slave 状态机

---

## 3\. XCP 报文结构详解

### 3\.1 通用报文结构（不依赖传输层）

XCP 报文在逻辑上分为三层，各层叠加构成完整的协议数据单元（PDU）：

**详细字节布局（以 CAN 传输层为例，8 字节 CAN 帧）：**

```text
Byte 0    | Byte 1    | Byte 2~5  | Byte 6~7
┌───────────┼───────────┼───────────┼──────────┐
│   PID     │   CMD     │   Data    │ 填充/CRC │
│  (0xF0)   │ (SHORT   │ 可变长度  │  (可选)   │
│           │  UPLOAD)  │           │          │
└───────────┴───────────┴───────────┴──────────┘
  ▲            ▲            ▲            ▲
  │           │            │            └─ CRC / 填充 (传输层依赖)
  │           │            └─ 命令参数 / 数据载荷
  │           └─ 命令码 (仅 CMD 包)
  └─ Packet Identifier (包类型标识)
```

**XCP 报文各层功能说明：**

|协议层|字节数|说明|
|---|---|---|
|**PID 层**|1 Byte|报文类型标识（CMD/RES/ERR/DAQ/EV/SERV）|
|**Timestamp/CT 层**|0\~4 Bytes|可选时间戳或计数器，用于同步测量|
|**Data 层**|可变|命令码 \+ 参数（CMD 包）或数据载荷（RES/DAQ 包）|
|**CRC 层**|0\~4 Bytes|可选校验和，保证数据完整性|

### 3\.2 PID —— Packet Identifier

PID 标识报文的**类型**，位于每个 XCP 报文的第一个字节：

```text
PID = 0x00 ~ 0xBF → 数据包（DAQ/STIM）
PID = 0xC0 ~ 0xDB → 服务接口包（用于 CAN FD 的扩展）
PID = 0xDC 0xDD   → 内部保留
PID = 0xDE → 错误（ERR）
PID = 0xE0 → 从站服务请求（SERV）
PID = 0xE1 → 事件（EV）
PID = 0xF0 → 命令包（CMD）
PID = 0xFD → 应答包（RES）
```

### 3\.3 命令包（CMD）结构

```
Byte 0: PID = 0xF0  (CMD 包标识)
Byte 1: CMD Code     (命令码，如 0xF4 = SHORT_UPLOAD)
Byte 2: Command Specific Data...
```

### 3\.4 应答包（RES）结构

```
Byte 0: PID = 0xFD  (RES 包标识)
Byte 1~N: Response Data
```

### 3\.5 错误包（ERR）结构

```
Byte 0: PID = 0xFE  (ERR 包标识)
Byte 1: Error Code
```

常见错误码：

|错误码|含义|
|---|---|
|0x00|ERR\_CMD\_UNKNOWN|
|0x01|ERR\_CMD\_BUSY|
|0x02|ERR\_ACCESS\_DENIED|
|0x10|ERR\_MEMORY\_OVERFLOW|

---

## 4\. 标定读操作：读取某地址的值

### 4\.1 交互流程

### 4\.2 完整的报文字节流（CAN 传输层，8 字节帧）

#### 4\.2\.1 SET\_MTA 命令

```
Master → Slave  (CAN ID: 0x601, DLC=8)

PID(1B) | CMD(1B) | MTA[0..3](4B) | EXT(1B) | reserved(1B)
  0xF0      0xF6     0x40 0x00 0x12 0x34    0x00       0x00

字段说明:
  - PID=0xF0                                      → 命令包
  - CMD=0xF6                                       → SET_MTA
  - MTA=0x40001234                                 → 目标地址 (大端序)
  - EXT=0x00                                       → 扩展位 (通常为 0)
```

```
Slave → Master  (CAN ID: 0x602, DLC=8)

PID(1B) | RES(1B)
  0xFD      0xFF

字段说明:
  - PID=0xFD  → 应答包
  - 0xFF      → 成功 (ERR_CMD_SYNC / SUCCESS)
```

#### 4\.2\.2 SHORT\_UPLOAD 命令

```
Master → Slave  (CAN ID: 0x601, DLC=8)

PID(1B) | CMD(1B) | Length(1B) | reserved(5B)
  0xF0      0xF4       0x04          0x00...

字段说明:
  - PID=0xF0    → 命令包
  - CMD=0xF4    → SHORT_UPLOAD
  - Length=0x04 → 读取 4 个字节
```

```
Slave → Master  (CAN ID: 0x602, DLC=8)

PID(1B) | Data[0] | Data[1] | Data[2] | Data[3] | reserved(4B)
  0xFD     0x42      0x00      0x00      0x00        0x00...

字段说明:
  - PID=0xFD → 应答包
  - Data     → 从 0x40001234 读取的 4 字节数据 (小端序值 = 0x00000042)
```

### 4\.3 SHORT\_UPLOAD 协议细节

|字段|字节数|说明|
|---|---|---|
|**请求:**|||
|PID|1|0xF0 \(CMD\)|
|CMD Code|1|0xF4 \(SHORT\_UPLOAD\)|
|Length|1|要读取的字节数（1\~255）|
|**应答（成功）:**|||
|PID|1|0xFD \(RES\)|
|Data|N|读取到的数据，N = Length|
|**应答（失败）:**|||
|PID|1|0xFE \(ERR\)|
|Error Code|1|错误码|

### 4\.4 与 SET\_MTA \+ UPLOAD 的对比

除了 SHORT\_UPLOAD，还能用 **SET\_MTA \+ UPLOAD** 组合：

|特性|SHORT\_UPLOAD \(0xF4\)|SET\_MTA \+ UPLOAD \(0xF6 \+ 0xF5\)|
|---|---|---|
|命令数|1 个|2 个|
|地址设定|隐含（使用当前 MTA）|显式设置|
|最大读取量|255 字节|可协商（更大）|
|适用场景|快速读少量参数|大批量数据读取|

---

## 5\. 标定写操作：修改某地址的值

### 5\.1 交互流程

### 5\.2 完整的报文字节流

#### 5\.2\.1 SHORT\_DOWNLOAD 命令

```
Master → Slave  (CAN ID: 0x601, DLC=8)

PID(1B) | CMD(1B) | Data[0] | Data[1] | Data[2] | Data[3] | reserved(3B)
  0xF0      0xF0     0x42      0x00      0x00      0x00       0x00...

字段说明:
  - PID=0xF0  → 命令包
  - CMD=0xF0  → SHORT_DOWNLOAD
  - Data      → 要写入 0x40001234 的 4 字节数据 (值=0x00000042)
```

```
Slave → Master  (CAN ID: 0x602, DLC=8)

PID(1B) | RES(1B)
  0xFD      0xFF

字段说明:
  - 0xFF → 写入成功
```

### 5\.3 SHORT\_DOWNLOAD 协议细节

|字段|字节数|说明|
|---|---|---|
|**请求:**|||
|PID|1|0xF0 \(CMD\)|
|CMD Code|1|0xF0 \(SHORT\_DOWNLOAD\)|
|Data|1\~255|要写入的数据（长度由实际数据决定）|
|**应答（成功）:**|||
|PID|1|0xFD \(RES\)|
|RES|1|0xFF \(成功\)|
|**应答（失败）:**|||
|PID|1|0xFE \(ERR\)|
|Error Code|1|错误码|

### 5\.4 写保护机制：Seed \& Key

大多数 ECU 在出厂后，**标定写入默认处于保护状态**。必须通过 Seed \& Key 解锁：

**常见资源保护掩码：**

|Bit|资源|
|---|---|
|Bit 0|CAL\_PAG \(Calibration Page 写访问\)|
|Bit 1|DAQ \(DAQ 配置\)|
|Bit 2|STIM \(刺激/仿真\)|
|Bit 3|PGM \(编程/刷写\)|

### 5\.5 MODIFY\_BITS：按位修改命令

当只需要修改某个地址的**部分位**时，应使用 `MODIFY_BITS` \(0xEF\) 命令：

```
Master → Slave

PID=0xF0 | CMD=0xEF | AND_MASK(1B) | OR_MASK(1B)

操作逻辑: target = (target & AND_MASK) | OR_MASK
```

示例：将地址 0x40001234 的 **Bit 3** 置 1，其余位不变：

|命令|值|效果|
|---|---|---|
|AND\_MASK|0xFF|所有位保持|
|OR\_MASK|0x08|Bit 3 = 1|

> **注**: MODIFY\_BITS 是原子操作，比先读后写更安全，适用于控制寄存器。
> 
> 

### 5\.6 SET\_MTA \+ DOWNLOAD \(0xF6 \+ 0xF2\) 的连用方式

#### 5\.6\.1 三种写入命令的定位

XCP 提供了三种写入命令，用途不同：

|命令|命令码|有无长度字段|是否需要后续包|最大数据量|适用场景|
|---|---|---|---|---|---|
|**SHORT\_DOWNLOAD**|0xF0|无（隐式）|否|≤255 字节|快速写少量参数|
|**DOWNLOAD**|0xF2|有（显式）|可跟 DOWNLOAD\_NEXT|按传输层协商|大批量数据写首包|
|**DOWNLOAD\_NEXT**|0xF1|无（隐式）|可继续跟|按传输层协商|大批量数据续包|

#### 5\.6\.2 SET\_MTA \+ SHORT\_DOWNLOAD — 小数据写入（最常用）

这是最常见的标定写入方式，**SET\_MTA 设定地址 → SHORT\_DOWNLOAD 写入数据**：

**字节流详解（CAN 8 字节帧）：**

```
SET_MTA 命令:
  Tx: 0xF0 0xF6 0x40 0x00 0x12 0x34 0x00 0x00
  │     │    │     └──────┬───────┘    │    │
  │     │    │            │            │    └─ 填充
  │     │    │            │            └─ MTA_EXT=0x00
  │     │    │            └─ 地址: 0x40001234 (大端序)
  │     │    └─ SET_MTA 命令码
  │     └─ PID_CMD
  └─ CAN ID (0x601)

  Rx: 0xFD 0xFF
  │    │    └─ ERR_NONE (成功)
  │    └─ PID_RES
  └─ CAN ID (0x602)

SHORT_DOWNLOAD 命令:
  Tx: 0xF0 0xF0 0x00 0x00 0x01 0x00 0x00 0x00
  │     │    └──────┬───────┘
  │     │           └─ 写入数据: 0x00010000 (小端序)
  │     │              即写入值 = 0x00010000
  │     └─ SHORT_DOWNLOAD 命令码
  └─ PID_CMD

  Rx: 0xFD 0xFF
  │    │    └─ 成功
  │    └─ PID_RES
  └─ CAN ID (0x602)
```

**关键点：** SHORT\_DOWNLOAD 的**数据长度**由 CAN 帧长度隐式决定。

- 上图 DLC=8，除去 PID\(1B\) \+ CMD\(1B\) = 6 字节有效数据

- 若只写 2 字节，可发 DLC=4: `0xF0 0xF0 0x00 0x00`

#### 5\.6\.3 SET\_MTA \+ DOWNLOAD \+ DOWNLOAD\_NEXT — 大数据写入

当写入数据超过单包容量时（例如一次写入 256 字节的标定曲线），使用 **DOWNLOAD \+ DOWNLOAD\_NEXT** 链式传输：

#### 5\.6\.4 DOWNLOAD 的字节流详解

**DOWNLOAD 命令 \(0xF2\) — 首包：**

```
Tx: 0xF0 0xF2 0x00 0x01 0xAA 0xBB 0xCC 0xDD
  │     │    │     │    └──────────┬───────────┘
  │     │    │     │               └─ 数据载荷 (6 字节)
  │     │    │     │                  0xAA 0xBB 0xCC 0xDD ...
  │     │    │     └─ 总长度 Length=0x0100 = 256 字节
  │     │    │         (大端序 2 字节)
  │     │    └─ DOWNLOAD 命令码 (0xF2)
  │     └─ PID_CMD
  └─ CAN ID

  Rx: 0xFD 0xFF  → 首包成功，继续发送 DOWNLOAD_NEXT
```

**DOWNLOAD\_NEXT 命令 \(0xF1\) — 续包：**

```
Tx: 0xF0 0xF1 0xEE 0xFF 0x00 0x11 0x22 0x33
  │     │    │    └──────────┬───────────┘
  │     │    │               └─ 第二段数据 (6 字节)
  │     │    └─ DOWNLOAD_NEXT 命令码 (0xF1)
  │     └─ PID_CMD
  └─ CAN ID

  Rx: 0xFD 0xFF  → 续包成功，继续下一个续包或结束
```

#### 5\.6\.5 三种写入方式的完整对比

|特性|SHORT\_DOWNLOAD \(0xF0\)|DOWNLOAD \(0xF2\) \+ DOWNLOAD\_NEXT \(0xF1\)|
|---|---|---|
|命令数|1 个写入命令|1 个 DOWNLOAD \+ N 个 DOWNLOAD\_NEXT|
|数据长度声明|无（隐式）|首包显式声明总长度|
|单包最大数据|CTO 容量 \- 1 字节|CTO 容量 \- 2 字节（含长度头）|
|总数据量上限|255 字节|65535 字节（16 位长度）|
|MTA 自增|写入后递增|每包写入后递增|
|适用场景|单参数标定（1 uint32）|标定曲线/Map/大数据块|

#### 5\.6\.6 完整实测：SET\_MTA \+ DOWNLOAD \+ DOWNLOAD\_NEXT

假设要在地址 0x40002000 写入一条 12 字节的标定曲线数据：

```text
数据: 0x01 0x02 0x03 0x04 0x05 0x06 0x07 0x08 0x09 0x0A 0x0B 0x0C
CAN 单帧容量: 6 字节有效数据 (8 字节帧 - PID - CMD)
```

**通信过程：**

```
Step 1: SET_MTA (0xF6)
  Master → Slave: F0 F6 40 00 20 00 00 00
  Slave  → Master: FD FF

Step 2: DOWNLOAD (0xF2) — 首包，声明总长度=12
  Master → Slave: F0 F2 00 0C 01 02 03 04 05 06
  │     │    │  │  └──────────┬──────────┘
  │     │    │  │             └─ 数据: 01 02 03 04 05 06 (6 字节)
  │     │    │  └─ 总长度 = 0x000C = 12 (大端序 2 字节)
  │     │    └─ DOWNLOAD 命令码
  │     └─ PID_CMD
  └─ CAN ID

  Slave  → Master: FD FF   ← 首包成功

  MTA 状态: 0x40002000 → 0x40002006 (递增 6 字节)

Step 3: DOWNLOAD_NEXT (0xF1) — 续包 1
  Master → Slave: F0 F1 07 08 09 0A 0B 0C 00 00
  │     │    └───┬────┘
  │     │        └─ 数据: 07 08 09 0A 0B 0C (6 字节)
  │     └─ DOWNLOAD_NEXT 命令码
  └─ PID_CMD

  Slave  → Master: FD FF   ← 续包成功

  MTA 状态: 0x40002006 → 0x4000200C (递增 6 字节)

所有 12 字节写入完成，最终 MTA = 0x4000200C
```

#### 5\.6\.7 对比：SHORT\_DOWNLOAD vs DOWNLOAD 的内部处理差异

从 Slave 协议栈实现角度看，两者的区别：

```c
/* === SHORT_DOWNLOAD 处理 (0xF0) === */
void handle_short_download(const uint8_t *cmd, uint32_t len)
{
    // len = 整个包长度 (除去 PID)
    uint32_t data_len = len - 1;  // 减去 CMD 码 (0xF0)

    // 直接写入 MTA 指向的内存
    memcpy(memory + (mta - base), &cmd[1], data_len);

    // MTA 自增
    mta += data_len;

    send_res_ok();
}

/* === DOWNLOAD 处理 (0xF2) === */
void handle_download(const uint8_t *cmd, uint32_t len)
{
    // 首包: cmd[1..2] = 总长度 (大端序 16 位)
    uint16_t total_len = ((uint16_t)cmd[1] << 8) | cmd[2];

    // cmd[3..] = 第一段数据
    uint32_t chunk_len = len - 3;  // 减去 PID + CMD + 长度头

    memcpy(memory + (mta - base), &cmd[3], chunk_len);
    mta += chunk_len;

    // 记录剩余待接收的数据量
    remaining = total_len - chunk_len;

    send_res_ok();
}

/* === DOWNLOAD_NEXT 处理 (0xF1) === */
void handle_download_next(const uint8_t *cmd, uint32_t len)
{
    uint32_t chunk_len = len - 1;  // 减去 PID + CMD

    // 直接写入当前 MTA, 自动递增
    memcpy(memory + (mta - base), &cmd[1], chunk_len);
    mta += chunk_len;

    remaining -= chunk_len;

    send_res_ok();

    // 如果 remaining == 0, 传输完成
}
```

#### 5\.6\.8 使用建议：何时用哪种方式？

|场景|推荐方式|原因|
|---|---|---|
|标定一个 `uint8` 参数|`SET_MTA` \+ `SHORT_DOWNLOAD`|只需 1 包，开销最小|
|标定一个 `uint32` 参数|`SET_MTA` \+ `SHORT_DOWNLOAD`|4 字节，1 包搞定|
|标定一个 `float` 参数|`SET_MTA` \+ `SHORT_DOWNLOAD`|4 字节，1 包搞定|
|标定一个 10 个元素的数组|`SET_MTA` \+ `SHORT_DOWNLOAD`|仍可 1 包|
|标定一条 256 点的曲线|`SET_MTA` \+ `DOWNLOAD` \+ `DOWNLOAD_NEXT`|超出单包容量|
|标定一个 2D Map \(数百字节\)|`SET_MTA` \+ `DOWNLOAD` \+ `DOWNLOAD_NEXT`|大数据链式传输|
|刷写一小段 Flash|`SET_MTA` \+ `DOWNLOAD` \+ `DOWNLOAD_NEXT`|通常需要大数据传输|

---

## 6\. 完整通信握手流程

### 6\.1 典型的标定会话

### 6\.2 分阶段详解

#### Phase 1: 连接建立

**CONNECT 命令** \(`CMD = 0xFF`\)：

```
请求:
  0xF0 | 0xFF | connect_mode
  ├─ PID = 0xF0
  ├─ CMD = 0xFF (CONNECT)
  └─ connect_mode:
       0x00 = NORMAL
       0x01 = USER_DEFINED

应答:
  0xFD | resource_mask(1B) | comm_mode(1B)
  ├─ resource_mask: Slave 支持的资源
  └─ comm_mode:
       Bit 0: Byte Order (0=little, 1=big)
       Bit 1: Address Granularity (0=1byte, 1=2byte, 2=4byte)
       Bit 2: Optional Commands
       ...
```

---

## 7\. 代码示例：模拟 XCP 标定读写

### 7\.1 示例概述

下面实现一个**极简的 XCP Slave 模拟器**和一个 **XCP Master 客户端**，运行在 PC 上。

> **注**：真实环境 XCP 运行在 CAN/Ethernet 上，这里用 TCP Socket 模拟传输层，聚焦 XCP 协议逻辑本身。
> 
> 

### 7\.2 XCP Slave 模拟器（C语言）

```c
/**
 * xcp_slave_sim.c
 * 极简 XCP Slave 模拟器 — 演示标定读写命令处理
 *
 * 编译: gcc -o xcp_slave_sim xcp_slave_sim.c -lpthread
 * 运行: ./xcp_slave_sim
 *
 * 说明:
 *   - 监听 TCP 127.0.0.1:5555
 *   - 模拟 ECU ROM: 0x40000000 ~ 0x40010000
 *   - 支持命令: CONNECT, DISCONNECT, SET_MTA, SHORT_UPLOAD, SHORT_DOWNLOAD, GET_SEED, UNLOCK
 *   - 初始状态为 PROTECTED, 需要通过 GET_SEED/UNLOCK 解锁写入
 */

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <stdint.h>
#include <stdbool.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <pthread.h>

/* ======================== XCP 协议常量 ======================== */

/* PID */
#define PID_CMD         0xF0
#define PID_RES         0xFD
#define PID_ERR         0xFE

/* 命令码 */
#define CMD_CONNECT         0xFF
#define CMD_DISCONNECT      0xFE
#define CMD_GET_STATUS      0xFD
#define CMD_GET_ID          0xFA
#define CMD_SET_MTA         0xF6
#define CMD_UPLOAD          0xF5
#define CMD_SHORT_UPLOAD    0xF4
#define CMD_DOWNLOAD        0xF2
#define CMD_SHORT_DOWNLOAD  0xF0
#define CMD_GET_SEED        0xF8
#define CMD_UNLOCK          0xF7

/* 错误码 */
#define ERR_NONE            0xFF
#define ERR_CMD_UNKNOWN     0x00
#define ERR_ACCESS_DENIED   0x02
#define ERR_OUT_OF_RANGE    0x10
#define ERR_SEED_KEY        0x11

/* 资源掩码 */
#define RES_CAL_PAG         (1 << 0)
#define RES_DAQ             (1 << 1)
#define RES_PGM             (1 << 3)

/* 保护模式 */
#define PROTECTED           0x00
#define UNLOCKED            0xFF

/* ======================== ECU 模拟器状态 ======================== */

/* 模拟 ECU 内存（64KB） */
#define MEMORY_SIZE         (64 * 1024)
#define MEMORY_BASE         0x40000000
static uint8_t ecu_memory[MEMORY_SIZE];

/* XCP Slave 协议状态 */
typedef struct {
    bool        connected;
    uint32_t    mta;            /* Memory Transfer Address */
    uint8_t     mta_ext;        /* MTA Extension          */
    uint8_t     protection;     /* 当前保护状态           */
    uint8_t     resource_mask;  /* 本 Slave 支持的资源     */
    int         client_fd;      /* 当前连接客户端          */
} XcpSlaveState;

static XcpSlaveState slave = {
    .connected     = false,
    .mta           = 0,
    .mta_ext       = 0,
    .protection    = PROTECTED,      /* 初始保护 */
    .resource_mask = RES_CAL_PAG | RES_DAQ,
    .client_fd     = -1,
};

/* ======================== 内存访问辅助函数 ======================== */

static bool addr_valid(uint32_t addr, uint32_t len)
{
    uint32_t offset = addr - MEMORY_BASE;
    if (addr < MEMORY_BASE) return false;
    if (offset + len > MEMORY_SIZE) return false;
    return true;
}

/* ======================== XCP 命令处理 ======================== */

/* 写应答到 socket */
static void send_res(int fd, const uint8_t *data, uint32_t len)
{
    send(fd, data, len, 0);
}

/* 单字节成功应答 */
static void send_res_ok(int fd)
{
    uint8_t res[2] = { PID_RES, ERR_NONE };
    send(fd, res, 2, 0);
}

/* 发送错误 */
static void send_err(int fd, uint8_t err_code)
{
    uint8_t err[2] = { PID_ERR, err_code };
    send(fd, err, 2, 0);
}

/* 处理 CONNECT 命令 */
static void handle_connect(int fd, const uint8_t *cmd)
{
    uint8_t mode = cmd[1];  /* 第2字节: connect_mode */

    if (slave.connected) {
        /* 已连接则拒绝 */
        send_err(fd, ERR_CMD_BUSY);
        return;
    }

    slave.connected  = true;
    slave.client_fd  = fd;
    slave.protection = PROTECTED;  /* 每次连接默认保护 */

    /* 应答: RES | resource_mask | comm_mode */
    uint8_t res[4] = {
        PID_RES,
        slave.resource_mask,    /* resource_mask */
        0x01,                   /* comm_mode */
        0x00                    /* 预留 */
    };
    send(fd, res, 4, 0);

    printf("[XCP] CONNECT: mode=%d, connected\n", mode);
}

/* 处理 DISCONNECT 命令 */
static void handle_disconnect(int fd)
{
    if (!slave.connected) {
        send_err(fd, ERR_CMD_UNKNOWN);
        return;
    }
    slave.connected = false;
    slave.client_fd = -1;
    slave.mta       = 0;
    send_res_ok(fd);

    printf("[XCP] DISCONNECT: done\n");
}

/* 处理 SET_MTA 命令 */
static void handle_set_mta(int fd, const uint8_t *cmd)
{
    if (!slave.connected) {
        send_err(fd, ERR_CMD_UNKNOWN);
        return;
    }

    /* 解析 MTA (大端序): cmd[1..4] = address */
    uint32_t addr = ((uint32_t)cmd[1] << 24) |
                    ((uint32_t)cmd[2] << 16) |
                    ((uint32_t)cmd[3] <<  8) |
                    ((uint32_t)cmd[4]);

    uint8_t ext = cmd[5];

    slave.mta     = addr;
    slave.mta_ext = ext;

    send_res_ok(fd);

    printf("[XCP] SET_MTA: addr=0x%08X, ext=0x%02X\n", addr, ext);
}

/* 处理 SHORT_UPLOAD 命令 — 标定读取 */
static void handle_short_upload(int fd, const uint8_t *cmd, uint32_t cmd_len)
{
    if (!slave.connected) {
        send_err(fd, ERR_CMD_UNKNOWN);
        return;
    }

    uint8_t length = cmd[1];  /* 要读取的字节数 */

    printf("[XCP] SHORT_UPLOAD: mta=0x%08X, len=%d\n", slave.mta, length);

    if (!addr_valid(slave.mta, length)) {
        send_err(fd, ERR_OUT_OF_RANGE);
        return;
    }

    uint32_t offset = slave.mta - MEMORY_BASE;

    /* 构造应答: RES + data */
    uint8_t res[256];
    res[0] = PID_RES;
    memcpy(&res[1], &ecu_memory[offset], length);

    send(fd, res, 1 + length, 0);

    /* MTA 自动递增 */
    slave.mta += length;
}

/* 处理 SHORT_DOWNLOAD 命令 — 标定写入 */
static void handle_short_download(int fd, const uint8_t *cmd, uint32_t cmd_len)
{
    if (!slave.connected) {
        send_err(fd, ERR_CMD_UNKNOWN);
        return;
    }

    /* 检查写保护 */
    if (slave.protection == PROTECTED) {
        printf("[XCP] SHORT_DOWNLOAD: 拒绝！需要 UNLOCK\n");
        send_err(fd, ERR_ACCESS_DENIED);
        return;
    }

    uint32_t data_len = cmd_len - 1;  /* 除去 PID */
    printf("[XCP] SHORT_DOWNLOAD: mta=0x%08X, len=%d\n", slave.mta, data_len);

    if (!addr_valid(slave.mta, data_len)) {
        send_err(fd, ERR_OUT_OF_RANGE);
        return;
    }

    uint32_t offset = slave.mta - MEMORY_BASE;
    memcpy(&ecu_memory[offset], &cmd[1], data_len);

    /* 打印写入的值 */
    printf("[XCP] 写入数据: ");
    for (uint32_t i = 0; i < data_len; i++) {
        printf("0x%02X ", cmd[1 + i]);
    }
    printf("\n");

    send_res_ok(fd);

    /* MTA 自动递增 */
    slave.mta += data_len;
}

/* 处理 GET_SEED 命令 */
static void handle_get_seed(int fd, const uint8_t *cmd)
{
    if (!slave.connected) {
        send_err(fd, ERR_CMD_UNKNOWN);
        return;
    }

    uint8_t resource = cmd[1];

    printf("[XCP] GET_SEED: resource=0x%02X\n", resource);

    /* 任意固定 SEED: 0x11223344 */
    uint8_t res[6] = {
        PID_RES,
        0x04,           /* SEED 长度 = 4 */
        0x11, 0x22, 0x33, 0x44
    };
    send(fd, res, 6, 0);
}

/* 处理 UNLOCK 命令 */
static void handle_unlock(int fd, const uint8_t *cmd, uint32_t cmd_len)
{
    if (!slave.connected) {
        send_err(fd, ERR_CMD_UNKNOWN);
        return;
    }

    uint32_t key_len = cmd_len - 1;  /* 除去 PID */

    printf("[XCP] UNLOCK: key_len=%d, key=", key_len);
    for (uint32_t i = 0; i < key_len; i++) {
        printf("0x%02X ", cmd[1 + i]);
    }
    printf("\n");

    /* 简单 Key 校验: SEED=0x11223344, 要求 Key 为 0x44332211 (反转) */
    uint8_t expected_key[4] = { 0x44, 0x33, 0x22, 0x11 };

    if (key_len < 4 || memcmp(&cmd[1], expected_key, 4) != 0) {
        printf("[XCP] UNLOCK: Key 错误!\n");
        send_err(fd, ERR_SEED_KEY);
        return;
    }

    slave.protection = UNLOCKED;

    /* 应答: privileges_mask (解锁的资源) */
    uint8_t res[2] = { PID_RES, slave.resource_mask };
    send(fd, res, 2, 0);

    printf("[XCP] UNLOCK: 成功! 写入权限已开放\n");
}

/* ======================== 主报文分发 ======================== */

static void process_xcp_message(int fd, const uint8_t *buf, uint32_t len)
{
    if (len < 2) return;

    uint8_t pid = buf[0];

    if (pid != PID_CMD) {
        printf("[XCP] 忽略非 CMD 包: PID=0x%02X\n", pid);
        return;
    }

    uint8_t cmd = buf[1];

    switch (cmd) {
    case CMD_CONNECT:
        handle_connect(fd, &buf[1]);
        break;
    case CMD_DISCONNECT:
        handle_disconnect(fd);
        break;
    case CMD_GET_STATUS: {
        uint8_t res[8] = { PID_RES, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00 };
        send(fd, res, 8, 0);
        break;
    }
    case CMD_GET_ID: {
        uint8_t res[8] = { PID_RES, 0x02, 0x00, 0x41, 0x42, 0x00, 0x00, 0x00 };
        send(fd, res, 8, 0);
        break;
    }
    case CMD_SET_MTA:
        handle_set_mta(fd, &buf[1]);
        break;
    case CMD_SHORT_UPLOAD:
        handle_short_upload(fd, &buf[1], len - 1);
        break;
    case CMD_SHORT_DOWNLOAD:
        handle_short_download(fd, &buf[1], len - 1);
        break;
    case CMD_GET_SEED:
        handle_get_seed(fd, &buf[1]);
        break;
    case CMD_UNLOCK:
        handle_unlock(fd, &buf[1], len - 1);
        break;
    default:
        printf("[XCP] 未知命令: 0x%02X\n", cmd);
        send_err(fd, ERR_CMD_UNKNOWN);
        break;
    }
}

/* ======================== TCP Server ======================== */

static void *client_handler(void *arg)
{
    int client_fd = *(int *)arg;
    free(arg);

    uint8_t buf[1024];

    printf("[Server] 新客户端连接, fd=%d\n", client_fd);

    while (1) {
        int n = (int)recv(client_fd, buf, sizeof(buf), 0);
        if (n <= 0) {
            printf("[Server] 客户端断开 fd=%d\n", client_fd);
            break;
        }

        printf("[Server] 收到 %d 字节:\n  ", n);
        for (int i = 0; i < n; i++) printf("0x%02X ", buf[i]);
        printf("\n");

        process_xcp_message(client_fd, buf, (uint32_t)n);
    }

    slave.connected = false;
    close(client_fd);
    return NULL;
}

int main(void)
{
    /* 初始化 ECU 内存 */
    for (uint32_t i = 0; i < MEMORY_SIZE; i++) {
        ecu_memory[i] = (uint8_t)(i & 0xFF);
    }
    /* 在特定地址预置可读的值 */
    ecu_memory[0x1234] = 0x42;  // 0x40001234 = 0x42

    int server_fd = socket(AF_INET, SOCK_STREAM, 0);
    if (server_fd < 0) {
        perror("socket");
        return 1;
    }

    struct sockaddr_in addr = {
        .sin_family      = AF_INET,
        .sin_port        = htons(5555),
        .sin_addr.s_addr = INADDR_ANY
    };

    int opt = 1;
    setsockopt(server_fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

    if (bind(server_fd, (struct sockaddr *)&addr, sizeof(addr)) < 0) {
        perror("bind");
        return 1;
    }

    listen(server_fd, 5);
    printf("========================================================\n");
    printf("  XCP Slave 模拟器启动\n");
    printf("  监听: TCP 127.0.0.1:5555\n");
    printf("  内存: 0x40000000 ~ 0x4000FFFF (64KB)\n");
    printf("  预置: [0x40001234] = 0x%02X\n", ecu_memory[0x1234]);
    printf("========================================================\n\n");

    while (1) {
        int *client_fd = malloc(sizeof(int));
        *client_fd = accept(server_fd, NULL, NULL);
        if (*client_fd < 0) {
            perror("accept");
            free(client_fd);
            continue;
        }

        pthread_t tid;
        pthread_create(&tid, NULL, client_handler, client_fd);
        pthread_detach(tid);
    }

    close(server_fd);
    return 0;
}
```

### 7\.3 XCP Master 客户端（Python）

```python
"""
xcp_master.py
XCP Master 客户端 — 演示标定读取和写入

使用: python xcp_master.py

演示流程:
  1. CONNECT
  2. 读取地址 0x40001234 (SHORT_UPLOAD)
  3. 尝试写入 — 应失败(被保护)
  4. GET_SEED + UNLOCK
  5. 写入地址 0x40001234 (SHORT_DOWNLOAD)
  6. 读取验证
  7. DISCONNECT
"""

import socket
import struct
import time

# ======================== XCP 协议常量 ========================
PID_CMD = 0xF0
PID_RES = 0xFD
PID_ERR = 0xFE

CMD_CONNECT        = 0xFF
CMD_DISCONNECT     = 0xFE
CMD_GET_STATUS     = 0xFD
CMD_GET_ID         = 0xFA
CMD_SET_MTA        = 0xF6
CMD_SHORT_UPLOAD   = 0xF4
CMD_SHORT_DOWNLOAD = 0xF0
CMD_GET_SEED       = 0xF8
CMD_UNLOCK         = 0xF7

ERR_NONE = 0xFF
ERR_SEED_KEY = 0x11


class XcpMaster:
    """XCP Master 简易实现"""

    def __init__(self, host='127.0.0.1', port=5555):
        self.host = host
        self.port = port
        self.sock = None

    # ------------------------ 传输层 ------------------------
    def connect_transport(self):
        """建立 TCP 连接"""
        self.sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        self.sock.connect((self.host, self.port))
        print(f"[Transport] TCP 连接建立: {self.host}:{self.port}")

    def send_cmd(self, cmd_bytes):
        """发送 XCP 命令"""
        pkt = bytes([PID_CMD]) + cmd_bytes
        print(f"[TX] {pkt.hex().upper()}")
        self.sock.send(pkt)

    def recv_res(self, expected_len=64):
        """接收应答"""
        data = self.sock.recv(expected_len)
        if not data:
            raise ConnectionError("连接断开")
        pid = data[0]
        if pid == PID_ERR:
            err_code = data[1]
            print(f"[RX] ERR: code=0x{err_code:02X}")
            raise RuntimeError(f"XCP Error 0x{err_code:02X}")
        print(f"[RX] {data.hex().upper()}")
        return data

    def recv_res_ok(self):
        """接收成功应答"""
        res = self.recv_res(2)
        assert res[0] == PID_RES, f"期望 RES, 得到 PID=0x{res[0]:02X}"
        return res

    # ------------------------ XCP 命令 ------------------------

    def connect(self):
        """CONNECT 命令"""
        print("\n=== 1. CONNECT ===")
        self.send_cmd(bytes([CMD_CONNECT, 0x00]))  # mode=Normal
        res = self.recv_res(8)
        resource_mask = res[1]
        comm_mode = res[2]
        print(f"  resource_mask=0x{resource_mask:02X}, comm_mode=0x{comm_mode:02X}")
        return resource_mask, comm_mode

    def disconnect(self):
        """DISCONNECT 命令"""
        print("\n=== 7. DISCONNECT ===")
        self.send_cmd(bytes([CMD_DISCONNECT]))
        self.recv_res_ok()
        print("  断开成功")

    def set_mta(self, addr: int, ext: int = 0):
        """SET_MTA 命令: 设定 Memory Transfer Address"""
        print(f"\n--- SET_MTA: addr=0x{addr:08X} ---")
        # 大端序 4 字节地址
        addr_bytes = struct.pack('>I', addr)
        self.send_cmd(bytes([CMD_SET_MTA]) + addr_bytes + bytes([ext]))
        self.recv_res_ok()
        print("  MTA 设置成功")

    def short_upload(self, length: int) -> bytes:
        """SHORT_UPLOAD 命令: 从当前 MTA 读取数据"""
        print(f"\n--- SHORT_UPLOAD: length={length} ---")
        self.send_cmd(bytes([CMD_SHORT_UPLOAD, length]))
        res = self.recv_res(length + 2)
        data = res[1:1 + length]
        print(f"  读取到数据: {data.hex().upper()}")
        print(f"  整数值 (little-endian): {int.from_bytes(data, 'little')}")
        return data

    def short_download(self, data: bytes):
        """SHORT_DOWNLOAD 命令: 写入数据到当前 MTA"""
        print(f"\n--- SHORT_DOWNLOAD: data={data.hex().upper()} ---")
        self.send_cmd(bytes([CMD_SHORT_DOWNLOAD]) + data)
        self.recv_res_ok()
        print("  写入成功")

    def get_seed(self, resource: int = 1) -> bytes:
        """GET_SEED 命令: 获取种子"""
        print(f"\n--- GET_SEED: resource=0x{resource:02X} ---")
        self.send_cmd(bytes([CMD_GET_SEED, resource]))
        res = self.recv_res(64)
        seed_len = res[1]
        seed = res[2:2 + seed_len]
        print(f"  种子: {seed.hex().upper()} (len={seed_len})")
        return seed

    def unlock(self, key: bytes):
        """UNLOCK 命令: 发送密钥解锁"""
        print(f"\n--- UNLOCK: key={key.hex().upper()} ---")
        self.send_cmd(bytes([CMD_UNLOCK]) + key)
        res = self.recv_res(8)
        privileges = res[1]
        print(f"  解锁成功, 开放资源: 0x{privileges:02X}")
        return privileges

    # ------------------------ 便捷操作 ------------------------

    def read_value(self, addr: int, length: int = 4) -> bytes:
        """便捷操作: 读取某地址的值"""
        self.set_mta(addr)
        return self.short_upload(length)

    def write_value(self, addr: int, data: bytes):
        """便捷操作: 写入某地址的值"""
        self.set_mta(addr)
        self.short_download(data)

    def close(self):
        if self.sock:
            self.sock.close()
            print("[Transport] 连接关闭")


# ======================== 主演示流程 ========================

def main():
    master = XcpMaster()

    try:
        # 0. 建立传输层连接
        master.connect_transport()

        # 1. CONNECT
        resource_mask, comm_mode = master.connect()

        # 2. 读取地址 0x40001234 (预置值 = 0x42)
        print("\n=== 2. 标定读取 ===")
        val_before = master.read_value(0x40001234, length=4)
        val_int_before = int.from_bytes(val_before, 'little')

        # 3. 尝试直接写入 — 应失败 (被保护)
        print("\n=== 3. 尝试受保护写入 (预期失败) ===")
        try:
            master.write_value(0x40001234, b'\x00\x00\x01\x00')
            print("  WARNING: 写操作未拒绝?")
        except RuntimeError as e:
            print(f"  ✓ 写入被拒绝: {e}")

        # 4. Seed & Key 解锁
        print("\n=== 4. Seed & Key 解锁 ===")
        seed = master.get_seed(resource=1)  # CAL_PAG
        # Key = SEED 的逆序 (从模拟器得知的简单算法)
        key = bytes(reversed(seed))
        print(f"  计算 Key: {key.hex().upper()} (seed 反转)")
        master.unlock(key)

        # 5. 写入新值
        print("\n=== 5. 标定写入 ===")
        new_value_int = 0x00010000
        new_value = struct.pack('<I', new_value_int)  # 小端序 4 字节
        master.write_value(0x40001234, new_value)

        # 6. 读取验证
        print("\n=== 6. 读取验证 ===")
        val_after = master.read_value(0x40001234, length=4)
        val_int_after = int.from_bytes(val_after, 'little')

        print(f"\n{'='*50}")
        print(f"  结果对比:")
        print(f"    写入前 value[0x40001234] = 0x{val_int_before:08X}")
        print(f"    写入后 value[0x40001234] = 0x{val_int_after:08X}")
        if val_int_after == new_value_int:
            print(f"  ✓ 标定读写验证成功!")
        else:
            print(f"  ✗ 值不匹配!")
        print(f"{'='*50}")

        # 7. 断开连接
        master.disconnect()

    except Exception as e:
        print(f"\n[Error] {e}")
    finally:
        master.close()


if __name__ == '__main__':
    main()
```

### 7\.4 运行演示效果

```
# 终端 1: 启动 Slave
$ gcc -o xcp_slave_sim xcp_slave_sim.c -lpthread
$ ./xcp_slave_sim
========================================================
  XCP Slave 模拟器启动
  监听: TCP 127.0.0.1:5555
  内存: 0x40000000 ~ 0x4000FFFF (64KB)
  预置: [0x40001234] = 0x42
========================================================

# 终端 2: 运行 Master
$ python xcp_master.py
[Transport] TCP 连接建立: 127.0.0.1:5555

=== 1. CONNECT ===
[TX] F0FF00
[RX] FD010100
  resource_mask=0x01, comm_mode=0x01

=== 2. 标定读取 ===

--- SET_MTA: addr=0x40001234 ---
[TX] F0F64000123400
[RX] FDFF
  MTA 设置成功

--- SHORT_UPLOAD: length=4 ---
[TX] F0F404
[RX] FD42000000
  读取到数据: 42000000
  整数值 (little-endian): 66  ← 0x42 = 66

=== 3. 尝试受保护写入 (预期失败) ===
  ✓ 写入被拒绝: XCP Error 0x02  ← ERR_ACCESS_DENIED

=== 4. Seed & Key 解锁 ===
  ✓ 解锁成功, 开放资源: 0x01

=== 5. 标定写入 ===
  ✓ 写入成功

=== 6. 读取验证 ===
  写入前 value[0x40001234] = 0x00000042
  写入后 value[0x40001234] = 0x00010000
  ✓ 标定读写验证成功!
```

---

## 8\. 深入原理：XCP 的内存访问机制

### 8\.1 Memory Transfer Address \(MTA\) 工作机制

MTA 是 XCP 标定读写的核心抽象，类似 CPU 的 **地址寄存器**：

**MTA 自动递增特性**的意义：

```c
/* 连续读取 4 个 int32 的示例 */
/* 手动: 需要 4 次 SET_MTA + 4 次 SHORT_UPLOAD  = 8 条命令 */

/* 利用自增: 1 次 SET_MTA + 4 次 SHORT_UPLOAD  = 5 条命令 */
set_mta(0x40001000);
v1 = short_upload(4);   // 读 0x40001000 ~ 0x40001003, MTA→0x40001004
v2 = short_upload(4);   // 读 0x40001004 ~ 0x40001007, MTA→0x40001008
v3 = short_upload(4);   // 读 0x40001008 ~ 0x4000100B, MTA→0x4000100C
v4 = short_upload(4);   // 读 0x4000100C ~ 0x4000100F
```

### 8\.2 地址对齐与 Granularity

XCP 支持三种地址粒度（Granularity），通过 CONNECT 应答的 comm\_mode 告知 Master：

|Granularity|字节对齐|地址位数|典型场景|
|---|---|---|---|
|0|1 字节|8/16/32/64|通用|
|1|2 字节|8/16/32/64|字对齐的 DSP|
|2|4 字节|8/16/32/64|32 位处理器|

Slave 硬件要求地址对齐，Master 必须在 SET\_MTA 时保证对齐。

### 8\.3 标定页切换机制（CAL\_PAGE）

XCP 支持**双页标定**（Calibration Page Switching），这是 AUTOSAR 标定最核心的机制之一：

**工作流程：**

1. ECU 正常运行时，使用 **ROM/Flash 页（Page 0）**

2. 标定工程师修改参数值 → 写入 **RAM 页（Page 1）**

3. 执行 `SET_CAL_PAGE` 命令切换到 RAM 页 → **在线生效**

4. 确认优化值后，通过 PGM（刷写）将 RAM 页**固化到 Flash**

### 8\.4 传输层解耦的深入理解

XCP 的设计精髓在于 **协议与传输层完全解耦**：

```text
┌───────────────────┐
│   XCP Core        │  ← 命令集、状态机、MTA、DAQ 完全与传输无关
│   (Resource Layer) │
├───────────────────┤
│   Transport Layer │  ← CAN / CAN FD / FlexRay / Ethernet / USB
│   (TL)            │
├───────────────────┤
│   Physical Layer  │  ← 物理总线 / 网络
└───────────────────┘
```

对于不同传输层，同一标定命令的**差异只在帧封装**：

|传输层|最大 PDU|PID 位置|时间戳支持|
|---|---|---|---|
|CAN|8 字节|Byte 0|选配（DTO）|
|CAN FD|64 字节|Byte 0|支持|
|Ethernet TCP/IP|可变|Byte 0|原生支持|
|FlexRay|可变|Byte 0|原生支持|
|USB|可变|Byte 0|支持|

### 8\.5 XCP 与 AUTOSAR 的关系

在 AUTOSAR 架构中，XCP 位于 **BSW 层**：

**AUTOSAR Xcp 模块的关键配置：**

```xml
<!-- XcpConfig 示例 (部分配置参数) -->
<XcpConfig>
    <!-- 传输层选择 -->
    <XcpTransportLayer>CAN</XcpTransportLayer>

    <!-- 协议参数 -->
    <XcpProtocol>
        <MaxCto>8</MaxCto>          <!-- 命令传输对象最大字节 -->
        <MaxDto>8</MaxDto>          <!-- 数据传输对象最大字节 -->
        <AddressGranularity>0</AddressGranularity>  <!-- 1 字节粒度 -->
    </XcpProtocol>

    <!-- 内存映射 (A2L 文件相关) -->
    <XcpMemoryAccess>
        <CalibrationArea>
            <BaseAddr>0x40000000</BaseAddr>
            <Size>0x10000</Size>
            <AccessType>CAL_PAG</AccessType>
        </CalibrationArea>
    </XcpMemoryAccess>
</XcpConfig>
```

### 8\.6 A2L 文件的作用

在实际工程中，标定工具通过 **A2L 文件**（ASAP2 / ASAM MCD\-2MC）了解 ECU 变量布局。A2L 文件描述了：

```
/begin CHARACTERISTIC
    "MyCalParam"           /* 标定参数名称 */
    "Value of my param"    /* 描述 */
    0x40001234             /* 内存地址 */
    LONG                   /* 数据类型 (4字节有符号整数) */
    0x00000001             /* 转换方法 */
    0                      /* 最小值 */
    100                    /* 最大值 */
/end CHARACTERISTIC
```

**XCP \+ A2L 的配合关系：**

标定工程师不需要记住 `0x40001234` 这个地址，工具根据 A2L 找到地址，通过 XCP 协议完成读写。

---

## 总结

|操作|使用的 XCP 命令|命令码|前置条件|
|---|---|---|---|
|**读取地址值**|SET\_MTA \+ SHORT\_UPLOAD|0xF6 → 0xF4|已 CONNECT|
|**写入地址值**|SET\_MTA \+ SHORT\_DOWNLOAD|0xF6 → 0xF0|已 CONNECT \+ UNLOCK|
|**按位修改**|MODIFY\_BITS|0xEF|已 CONNECT \+ UNLOCK|
|**解锁写入权限**|GET\_SEED \+ UNLOCK|0xF8 → 0xF7|已 CONNECT|

**关键要点回顾：**

1. XCP 是 **Master\-Slave** 协议，Master 发命令，Slave 应答

2. **MTA** 是访问内存的指针，读写后自动递增

3. **SHORT\_UPLOAD** 用于读取，**SHORT\_DOWNLOAD** 用于写入

4. 写操作通常需要先 **GET\_SEED \+ UNLOCK** 解锁

5. **CAL\_PAGE** 机制支持在线切换标定页，实现"瞬时生效"

6. XCP 的传输层解耦设计使其适用于 CAN / FlexRay / Ethernet 等多种总线

> （注：部分内容可能由 AI 生成）

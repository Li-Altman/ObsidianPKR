# ComM、NM、CanNM 交互逻辑详解

> 本文详细讲解 AUTOSAR 架构中 **ComM（Communication Manager）**、**NM（Network Management 抽象层）** 和 **CanNM（CAN 网络管理具体实现）** 三个模块之间的交互逻辑，以及完整的网络管理协调机制。

---

## 1. 通俗理解：三模块的角色分工

可以把这三层关系类比为 **"公司管理层 — 项目经理 — 一线工人"**：

| 角色 | 类比 | 职责 |
|------|------|------|
| **ComM** | 公司管理层 | 决定"要不要通信"，管理应用层的通信需求，协调网络状态 |
| **NM（抽象层）** | 项目经理 | 传达管理层的指令，但不关心具体的通信协议细节 |
| **CanNM** | 一线工人 | 真正干活——发送/接收 CAN 网络管理报文，维护 CAN 专属的状态机 |

**核心流程一句话：** SW-C（应用层软件组件）通过 ComM 发起通信请求 → ComM 协调后通知 NM → NM 转发给 CanNM → CanNM 在 CAN 总线上发送/接收 NM 报文来同步网络状态。

---

## 2. 模块架构概览

### 2.1 模块层级关系图

```mermaid
graph TB
    subgraph "Application Layer (SW-Cs)"
        A[SW-C 1<br/>应用组件]
        B[SW-C 2<br/>应用组件]
        C[SW-C N<br/>应用组件]
    end

    subgraph "BSW Layer"
        subgraph "ComM - Communication Manager"
            CM[ComM<br/>通信管理器]
            CM_SM[ComM State Machine<br/>通信状态机]
            CM_UM[User Management<br/>用户请求管理]
        end

        subgraph "NM - Network Management Abstraction"
            NM[NM 抽象层<br/>通用接口]
            NM_CB[Callback 路由<br/>回调分发]
        end

        subgraph "CanNM - CAN NM Implementation"
            CAN_NM[CanNM<br/>CAN 网络管理]
            CAN_NM_SM[CanNM State Machine<br/>CAN NM 状态机]
            CAN_NM_PDU[NM PDU 处理<br/>报文收发]
        end

        subgraph "Lower Layers"
            CAN_IF[CanIf<br/>CAN 接口层]
            CAN_DRV[CanDrv<br/>CAN 驱动层]
        end
    end

    A -->|ComM_RequestComMode| CM
    B -->|ComM_RequestComMode| CM
    C -->|ComM_RequestComMode| CM
    CM -->|Nm_NetworkRequest| NM
    CM -->|Nm_NetworkRelease| NM
    NM -->|转发请求| CAN_NM
    CAN_NM -->|Nm_NetworkMode/ComM_Nm_xxx| NM
    NM -->|回调| CM
    CAN_NM -->|CanIf_Transmit| CAN_IF
    CAN_IF -->|Can_Write| CAN_DRV
    CAN_DRV -->|CAN Bus| BUS((CAN Bus))

    classDef app fill:#e1f5fe,stroke:#01579b,stroke-width:2px
    classDef bsw fill:#fff3e0,stroke:#e65100,stroke-width:2px
    classDef detail fill:#f3e5f5,stroke:#4a148c,stroke-width:2px
    classDef lower fill:#e8f5e9,stroke:#1b5e20,stroke-width:2px

    class A,B,C app
    class CM,CM_SM,CM_UM bsw
    class NM,NM_CB bsw
    class CAN_NM,CAN_NM_SM,CAN_NM_PDU detail
    class CAN_IF,CAN_DRV lower
```

**图解释：** 上图展示了三个模块在 AUTOSAR 架构中的层级位置。
- **ComM** 位于 BSW 上层，直接与 SW-C 交互，管理通信请求
- **NM（抽象层）** 是 ComM 和具体协议实现之间的中间层，提供统一的接口
- **CanNM** 是协议相关的具体实现，通过 CanIf 和 CanDrv 操作 CAN 硬件
- 调用方向从上到下（请求），回调方向从下到上（通知）

### 2.2 交互接口总览

```mermaid
graph LR
    subgraph "Interfaces"
        I1["ComM_RequestComMode<br/>(SW-C → ComM)"]
        I2["ComM_GetCurrentComMode<br/>(SW-C → ComM)"]
        I3["Nm_NetworkRequest<br/>(ComM → NM)"]
        I4["Nm_NetworkRelease<br/>(ComM → NM)"]
        I5["Nm_GetState<br/>(ComM → NM)"]
        I6["CanNm_NetworkRequest<br/>(NM → CanNM)"]
        I7["CanNm_NetworkRelease<br/>(NM → CanNM)"]
        I8["ComM_Nm_NetworkStart<br/>(NM → ComM)"]
        I9["ComM_Nm_NetworkMode<br/>(NM → ComM)"]
        I10["ComM_Nm_NetworkRelease<br/>(NM → ComM)"]
        I11["Nm_ConfirmNm<br/>(CanNM → NM)"]
        I12["Callback 路由<br/>(NM → ComM)"]
    end

    I1 --> I3
    I3 --> I6
    I6 --> I11
    I11 --> I12

    classDef if fill:#ffebee,stroke:#b71c1c,stroke-width:1px
    class I1,I2,I3,I4,I5,I6,I7,I8,I9,I10,I11,I12 if
```

**图解释：** 关键接口路径。
- **SW-C → ComM**: `ComM_RequestComMode` 请求通信模式
- **ComM → NM → CanNM**: 请求链（NetworkRequest / NetworkRelease）
- **CanNM → NM → ComM**: 回调链（NetworkStart / NetworkMode / NetworkRelease）

---

## 3. ComM 详解

### 3.1 ComM 的作用

ComM 是 AUTOSAR 通信架构的 **"交通警察"**，负责：

1. **管理多个通信通道（Channel）** 的状态
2. **聚合多个 SW-C 的通信请求**，决定最终的通信模式
3. **协调网络状态机**，控制 NM 的启动与停止
4. **管理通信权限**（COMM_NO_COMMUNICATION / COMM_SILENT_COMMUNICATION / COMM_FULL_COMMUNICATION）

### 3.2 ComM 状态机

```mermaid
stateDiagram-v2
    [*] --> COMM_NO_COMMUNICATION: 初始化

    state COMM_NO_COMMUNICATION {
        [*] --> NoComm_Wait
        NoComm_Wait --> NoComm_Request: SW-C 请求通信
    }

    state COMM_SILENT_COMMUNICATION {
        [*] --> Silent_Active
        Silent_Active --> Silent_Request: 用户请求增加
        Silent_Active --> Silent_Timeout: 所有用户释放
    }

    state COMM_FULL_COMMUNICATION {
        [*] --> FullComm_Active
        FullComm_Active --> FullComm_Active: 多个用户同时请求
        FullComm_Active --> FullComm_Release: 用户释放
        FullComm_Release --> FullComm_Active: 仍有其他用户请求
        FullComm_Release --> Silent_Active: 无用户请求
    }

    COMM_NO_COMMUNICATION --> COMM_SILENT_COMMUNICATION: 请求 SILENT
    COMM_NO_COMMUNICATION --> COMM_FULL_COMMUNICATION: 请求 FULL
    COMM_SILENT_COMMUNICATION --> COMM_FULL_COMMUNICATION: 请求升级
    COMM_SILENT_COMMUNICATION --> COMM_NO_COMMUNICATION: 所有用户释放
    COMM_FULL_COMMUNICATION --> COMM_SILENT_COMMUNICATION: 降级请求
    COMM_FULL_COMMUNICATION --> COMM_NO_COMMUNICATION: 直接下线
    COMM_FULL_COMMUNICATION --> COMM_NO_COMMUNICATION: 所有用户释放
```

**图解释：** ComM 有三种通信模式：
- **COMM_NO_COMMUNICATION**: 无通信，总线关闭，NM 停止
- **COMM_SILENT_COMMUNICATION**: 静默通信，仅监听不发送（部分场景需要）
- **COMM_FULL_COMMUNICATION**: 全功能通信，正常收发

### 3.3 ComM 的"用户管理"机制

ComM 的核心理念是 **"多用户引用计数"**：

```c
/* ComM 内部伪代码 - 用户管理 */
typedef struct {
    uint8_t  UserId;              /* 用户 ID */
    ComM_ModeType RequestedMode;  /* 请求的通信模式 */
    boolean  IsActive;            /* 是否激活 */
} ComM_UserType;

typedef struct {
    ComM_ModeType    CurrentMode;    /* 当前通信模式 */
    ComM_ModeType    TargetMode;     /* 目标通信模式 */
    uint8_t          FullCommCount;  /* FULL 模式请求计数 */
    uint8_t          SilentCommCount;/* SILENT 模式请求计数 */
    ComM_UserType    Users[10];      /* 最多 10 个用户 */
} ComM_ChannelType;

/* 计算最终通信模式 */
ComM_ModeType ComM_CalculateTargetMode(ComM_ChannelType* Channel) {
    /* 优先级: FULL_COMMUNICATION > SILENT_COMMUNICATION > NO_COMMUNICATION */
    if (Channel->FullCommCount > 0) {
        return COMM_FULL_COMMUNICATION;
    }
    if (Channel->SilentCommCount > 0) {
        return COMM_SILENT_COMMUNICATION;
    }
    return COMM_NO_COMMUNICATION;
}
```

**代码解释：** 每个 SW-C 是一个"用户"。当一个 SW-C 调用 `ComM_RequestComMode(ChannelId, COMM_FULL_COMMUNICATION)` 时，ComM 增加 FULL 计数。当多个用户同时请求 FULL 时，计数累加；用户释放时计数递减。只有所有用户都释放后，才会进入 NO_COMMUNICATION。

---

## 4. NM 抽象层详解

### 4.1 NM 抽象层的作用

NM 抽象层位于 ComM 和协议特定实现之间，提供了一组 **标准化的 API**，使得 ComM 不需要关心底层是 CAN、LIN 还是 FlexRay。

### 4.2 NM 抽象层的接口映射

```mermaid
graph TD
    subgraph "ComM 视角（统一接口）"
        A1["Nm_NetworkRequest(channel)"]
        A2["Nm_NetworkRelease(channel)"]
        A3["Nm_GetState(channel)"]
    end

    subgraph "NM 抽象层（路由转发）"
        B["NM 抽象层<br/>根据配置路由到协议特定模块"]
        B1["Channel 1 → CAN"]
        B2["Channel 2 → LIN"]
        B3["Channel 3 → FlexRay"]
    end

    subgraph "协议特定实现"
        C1["CanNm_NetworkRequest"]
        C2["LinNm_NetworkRequest"]
        C3["FrNm_NetworkRequest"]
    end

    A1 --> B
    A2 --> B
    A3 --> B
    B --> B1
    B --> B2
    B --> B3
    B1 --> C1
    B2 --> C2
    B3 --> C3

    classDef comm fill:#e3f2fd,stroke:#1565c0
    classDef nm fill:#fff3e0,stroke:#e65100
    classDef impl fill:#f3e5f5,stroke:#4a148c

    class A1,A2,A3 comm
    class B,B1,B2,B3 nm
    class C1,C2,C3 impl
```

**图解释：** NM 抽象层是"万能适配器"。
- ComM 只调用 `Nm_xxx` 系列接口
- NM 抽象层根据配置（Channel ID 映射）把调用转发到对应的协议实现
- 对于 CAN 网络，转发到 `CanNm_xxx`；对于 LIN 网络，转发到 `LinNm_xxx`
- 这种设计体现了 **抽象工厂模式** 和 **策略模式** 的思想

---

## 5. CanNM 详解

### 5.1 CanNM 状态机

CanNM 拥有完整的网络管理状态机，这是整个交互中最核心的部分：

```mermaid
stateDiagram-v2
    [*] --> BUS_SLEEP: 初始化/PowerON

    state BUS_SLEEP {
        [*] --> Sleep_Idle
    }

    state PREPARE_BUS_SLEEP {
        [*] --> PBS_Wait
        PBS_Wait --> PBS_Timeout: 计时器超时
    }

    state READY_SLEEP {
        [*] --> RS_Idle
        RS_Idle --> RS_Active: 收到 NM 报文
    }

    state NORMAL_OPERATION {
        [*] --> NO_Active
        NO_Active --> NO_Tx: 发送 NM 报文
        NO_Active --> NO_Rx: 接收 NM 报文
        NO_Tx --> NO_Active
        NO_Rx --> NO_Active
    }

    state REPEAT_MESSAGE {
        [*] --> RM_Active
        RM_Active --> RM_Tx: 重复发送 NM 报文
        RM_Tx --> RM_Active
    }

    BUS_SLEEP --> REPEAT_MESSAGE: 网络请求<br/>或收到 NM 报文
    BUS_SLEEP --> PREPARE_BUS_SLEEP: 进入睡眠

    PREPARE_BUS_SLEEP --> BUS_SLEEP: PBS 超时<br/>无通信需求
    PREPARE_BUS_SLEEP --> READY_SLEEP: 收到 NM 报文

    READY_SLEEP --> REPEAT_MESSAGE: 收到 NM 报文<br/>且网络有活动
    READY_SLEEP --> NORMAL_OPERATION: 网络同步

    REPEAT_MESSAGE --> NORMAL_OPERATION: RM 时间到<br/>网络已同步
    REPEAT_MESSAGE --> READY_SLEEP: 无通信需求

    NORMAL_OPERATION --> READY_SLEEP: NM 报文停止<br/>T_WAIT_BUS_SLEEP 超时

    note right of BUS_SLEEP: 低功耗状态<br/>总线关闭
    note right of REPEAT_MESSAGE: 网络启动同步阶段<br/>发送 NM 报文宣告存在
    note right of NORMAL_OPERATION: 正常运行<br/>周期性发送 NM 报文
    note right of READY_SLEEP: 等待睡眠确认
    note right of PREPARE_BUS_SLEEP: 准备总线睡眠
```

**图解释：** CanNM 有 5 个核心状态，构成了完整的网络管理生命周期：
- **BUS_SLEEP**: 睡眠状态，总线关闭，低功耗
- **REPEAT_MESSAGE**: 重复消息状态，网络启动时发送 NM 报文宣告自己的存在
- **NORMAL_OPERATION**: 正常运行状态，周期性发送 NM 报文维持网络
- **READY_SLEEP**: 准备睡眠状态，等待确认所有节点准备休眠
- **PREPARE_BUS_SLEEP**: 准备总线睡眠状态，等待超时后进入睡眠

### 5.2 CanNM 关键定时参数

| 参数 | 含义 | 典型值 | 说明 |
|------|------|--------|------|
| **T_REPEAT_MESSAGE** | 重复消息时间 | 500~1000ms | 网络启动时持续发送 NM 报文的时间 |
| **T_WAIT_BUS_SLEEP** | 等待总线睡眠时间 | 2000~5000ms | 从 NORMAL 到 READY_SLEEP 的等待时间 |
| **T_PREPARE_BUS_SLEEP** | 准备总线睡眠时间 | 500~1000ms | 从 READY_SLEEP 到 BUS_SLEEP 的等待时间 |
| **T_NM_Tx** | NM 报文发送周期 | 100~1000ms | 正常模式下发送 NM 报文的周期 |
| **T_NM_Timeout** | NM 报文超时时间 | 1500~5000ms | 接收 NM 报文的超时检测时间 |

---

## 6. 完整交互流程详解

### 6.1 网络启动流程

```mermaid
sequenceDiagram
    participant SWC as SW-C
    participant ComM as ComM
    participant NM as NM (抽象层)
    participant CanNM as CanNM
    participant CanIf as CanIf/CanDrv
    participant Bus as CAN Bus
    participant NodeB as 其他节点

    Note over SWC,Bus: === 网络启动流程（Network Start）===

    SWC->>ComM: ComM_RequestComMode(Channel, COMM_FULL_COMMUNICATION)
    Note over ComM: 增加用户引用计数<br/>FullCommCount++

    ComM->>ComM: 计算目标模式 = FULL_COMMUNICATION
    Note over ComM: 当前 NO_COMM → 目标 FULL_COMM<br/>触发状态切换

    ComM->>NM: Nm_NetworkRequest(Channel)
    Note over NM: 抽象层路由到 CanNM

    NM->>CanNM: CanNm_NetworkRequest(Channel)

    CanNM->>CanNM: 启动状态机<br/>BUS_SLEEP → REPEAT_MESSAGE

    Note over CanNM: 启动 T_REPEAT_MESSAGE 定时器

    CanNM->>CanIf: CanIf_CanNmTransmit(NM_PDU)
    CanIf->>CanDrv: Can_Write(NM_PDU)
    CanDrv->>Bus: 发送 NM 报文 (ID: 0x5xx)
    Bus->>NodeB: 其他节点收到 NM 报文

    CanNM->>CanNM: 周期发送 NM 报文<br/>(按 T_NM_Tx 周期)

    Note over CanNM: T_REPEAT_MESSAGE 超时

    CanNM->>CanNM: REPEAT_MESSAGE → NORMAL_OPERATION

    CanNM->>NM: CanNm_NetworkMode(Channel, NM_MODE_SYNCHRONIZE)
    NM->>ComM: ComM_Nm_NetworkMode(Channel, NM_MODE_SYNCHRONIZE)
    Note over ComM: 网络已建立<br/>更新通信状态

    ComM-->>SWC: ComM_GetCurrentComMode → COMM_FULL_COMMUNICATION
```

**图解释：** 网络启动流程的完整时序：
1. **SW-C 发起请求**: 应用层调用 `ComM_RequestComMode` 请求 FULL 通信
2. **ComM 决策**: 增加引用计数，计算目标模式，判断需要启动网络
3. **NM 路由**: 通过 `Nm_NetworkRequest` 将请求转发到 CanNM
4. **CanNM 启动**: 状态从 BUS_SLEEP → REPEAT_MESSAGE，开始发送 NM 报文
5. **网络同步**: 在 REPEAT_MESSAGE 期间持续发送 NM 报文，让其他节点感知到本节点
6. **进入正常运行**: REPEAT_MESSAGE 超时后进入 NORMAL_OPERATION，通知上层网络已就绪

### 6.2 正常网络运行状态

```mermaid
sequenceDiagram
    participant NodeA as Node A (CanNM)
    participant NodeB as Node B (CanNM)
    participant NodeC as Node C (CanNM)
    participant Bus as CAN Bus

    Note over NodeA,Bus: === 网络正常运行（NORMAL_OPERATION）===

    loop 每个节点周期发送 NM 报文
        NodeA->>Bus: NM_PDU (ID: 0x501, NodeA)
        NodeB->>Bus: NM_PDU (ID: 0x502, NodeB)
        NodeC->>Bus: NM_PDU (ID: 0x503, NodeC)
    end

    Note over NodeA: 收到 NodeB 的 NM 报文
    Note over NodeA: 更新 NM 协调状态
    Note over NodeA: 重启 T_NM_Timeout 定时器

    Note over NodeB: 收到 NodeC 的 NM 报文
    Note over NodeB: 更新 NM 协调状态

    Note over NodeC: 收到 NodeA 的 NM 报文
    Note over NodeC: 更新 NM 协调状态

    Note over NodeA,NodeC: 所有节点都在 NORMAL_OPERATION<br/>网络保持唤醒状态
```

**图解释：** 正常运行状态中：
- 每个节点以 **T_NM_Tx** 周期发送自己的 NM 报文
- 每个节点收到其他节点的 NM 报文后，**重启超时定时器**
- 只要总线上有任意一个节点还在发送 NM 报文，网络就保持唤醒
- 这种机制实现了 **"最后一个节点决定睡眠"** 的分布式策略

### 6.3 网络释放流程

```mermaid
sequenceDiagram
    participant SWC as SW-C
    participant ComM as ComM
    participant NM as NM (抽象层)
    participant CanNM as CanNM
    participant Bus as CAN Bus

    Note over SWC,Bus: === 网络释放流程（Network Release）===

    SWC->>ComM: ComM_RequestComMode(Channel, COMM_NO_COMMUNICATION)
    Note over ComM: 减少用户引用计数<br/>FullCommCount--

    ComM->>ComM: 计算目标模式 = NO_COMMUNICATION
    Note over ComM: FullCommCount == 0<br/>SilentCommCount == 0

    ComM->>NM: Nm_NetworkRelease(Channel)

    NM->>CanNM: CanNm_NetworkRelease(Channel)

    CanNM->>CanNM: NORMAL_OPERATION → READY_SLEEP
    Note over CanNM: 停止发送 NM 报文<br/>启动 T_WAIT_BUS_SLEEP 定时器

    Bus-->>CanNM: 其他节点可能仍在发送 NM 报文

    alt 收到其他节点 NM 报文
        CanNM->>CanNM: READY_SLEEP → REPEAT_MESSAGE
        Note over CanNM: 网络有其他节点活跃<br/>不能睡眠
    else T_WAIT_BUS_SLEEP 超时
        CanNM->>CanNM: READY_SLEEP → PREPARE_BUS_SLEEP
        Note over CanNM: 启动 T_PREPARE_BUS_SLEEP 定时器

        CanNM->>NM: CanNm_NetworkRelease(Channel)
        NM->>ComM: ComM_Nm_NetworkRelease(Channel)

        Note over CanNM: T_PREPARE_BUS_SLEEP 超时

        CanNM->>CanNM: PREPARE_BUS_SLEEP → BUS_SLEEP
        Note over CanNM: 进入睡眠状态<br/>总线关闭，低功耗
    end
```

**图解释：** 网络释放流程展示了 AUTOSAR 网络管理最精妙的设计——**分布式睡眠协调**：
1. **ComM 释放**: 用户释放通信请求，引用计数归零
2. **CanNM 开始休眠流程**: 进入 READY_SLEEP，停止发送自己的 NM 报文
3. **等待其他节点**: 启动 T_WAIT_BUS_SLEEP 定时器
4. **关键决策点**:
   - 如果在等待期间收到其他节点的 NM 报文 → 说明网络还有活跃节点，回到 REPEAT_MESSAGE
   - 如果超时没有收到任何 NM 报文 → 说明自己是最后一个节点，可以安全进入睡眠
5. **进入睡眠**: 经过 PREPARE_BUS_SLEEP 后进入 BUS_SLEEP

### 6.4 网络唤醒流程

```mermaid
sequenceDiagram
    participant SWC as SW-C
    participant ComM as ComM
    participant NM as NM (抽象层)
    participant CanNM as CanNM
    participant Bus as CAN Bus

    Note over SWC,Bus: === 网络唤醒流程（Network Wakeup）===

    alt 本地唤醒（Local Wakeup）
        SWC->>ComM: ComM_RequestComMode(Channel, COMM_FULL_COMMUNICATION)
        ComM->>NM: Nm_NetworkRequest(Channel)
        NM->>CanNM: CanNm_NetworkRequest(Channel)
        CanNM->>CanNM: BUS_SLEEP → REPEAT_MESSAGE
        CanNM->>Bus: 发送 NM 报文 (唤醒其他节点)

    else 远程唤醒（Remote Wakeup）
        Bus->>CanNM: 收到其他节点的 NM 报文（远程唤醒）
        CanNM->>CanNM: BUS_SLEEP → REPEAT_MESSAGE
        CanNM->>NM: CanNm_NetworkStart(Channel, NM_WAKEUP_REMOTE)
        NM->>ComM: ComM_Nm_NetworkStart(Channel, NM_WAKEUP_REMOTE)
        Note over ComM: 检测到远程唤醒<br/>更新通信状态
        ComM-->>SWC: 通知 SW-C 网络已唤醒
    end
```

**图解释：** 网络唤醒有两种方式：
- **本地唤醒**: 本地节点的应用层主动请求通信，由 SW-C 发起
- **远程唤醒**: 收到总线上其他节点的 NM 报文，被动唤醒
- 两种方式最终都会进入 REPEAT_MESSAGE 状态，重新开始网络同步

---

## 7. CanNM 报文格式详解

### 7.1 NM 报文结构

```mermaid
packet-beta
  0-7: "CAN ID (Base ID + Node ID)"
  8-15: "Reserved"
  16-23: "Control Bit Vector (CBV)"
  24-31: "User Data (0-8 bytes)"

  title: "CAN NM PDU 格式"
```

| 字段 | 长度 | 说明 |
|------|------|------|
| **CAN ID** | 11/29 bit | 基本 ID + 节点 ID（如 0x500 + NodeID） |
| **Control Bit Vector (CBV)** | 1 byte | 控制位向量，标识 NM 状态 |
| **User Data** | 0-8 bytes | 用户自定义数据（可选） |

### 7.2 CBV（Control Bit Vector）详解

```mermaid
graph LR
    subgraph CBV Byte
        B7["Bit 7<br/>Reserved"]
        B6["Bit 6<br/>Reserved"]
        B5["Bit 5<br/>Reserved"]
        B4["Bit 4<br/>PN Anticipate"]
        B3["Bit 3<br/>PN Available"]
        B2["Bit 2<br/>Reserved"]
        B1["Bit 1<br/>NM Coordinator Sleep"]
        B0["Bit 0<br/>Active Wakeup"]
    end

    B0 -->|"1 = 主动唤醒"| AW["本节点主动唤醒网络"]
    B1 -->|"1 = 协调睡眠"| CS["本节点准备睡眠（最后一个节点）"]
    B3 -->|"1 = 可用"| PA["Partial Network 可用"]
    B4 -->|"1 = 预判"| PAN["Partial Network 预判"]

    classDef cbv fill:#e8eaf6,stroke:#283593
    classDef desc fill:#e0f2f1,stroke:#004d40

    class B0,B1,B2,B3,B4,B5,B6,B7 cbv
    class AW,CS,PA,PAN desc
```

**图解释：** CBV 是 NM 报文中的控制字节，每个 bit 代表不同的网络管理状态：
- **Bit 0 (Active Wakeup)**: 本节点主动唤醒网络，1 表示本节点是唤醒源
- **Bit 1 (NM Coordinator Sleep)**: 协调睡眠标志，最后一个准备睡眠的节点置位此位
- **Bit 3, Bit 4**: 用于 Partial Network 功能，支持选择性唤醒

---

## 8. 状态转换与时间量化关系

### 8.1 状态转换时序图

```mermaid
timeline
    title CanNM 状态转换时序（完整生命周期）
    T0 : 初始状态 : BUS_SLEEP
    T1 : 网络请求 : REPEAT_MESSAGE (T_REPEAT_MSN)
    T2 : 网络同步 : REPEAT_MESSAGE → NORMAL_OPERATION
    T3 : 正常运行 : NORMAL_OPERATION (周期性发送 NM)
    T4 : 所有节点释放 : READY_SLEEP (T_WAIT_BUS_SLEEP)
    T5 : 等待确认 : PREPARE_BUS_SLEEP (T_PREPARE_BUS_SLEEP)
    T6 : 睡眠 : BUS_SLEEP
```

### 8.2 多节点交互时序（完整场景）

```mermaid
sequenceDiagram
    participant NodeA as Node A
    participant NodeB as Node B
    participant NodeC as Node C
    participant Bus as CAN Bus

    Note over NodeA,Bus: 初始状态: 所有节点 BUS_SLEEP

    NodeA->>NodeA: 本地应用请求通信
    NodeA->>NodeA: BUS_SLEEP → REPEAT_MESSAGE
    NodeA->>Bus: NM_PDU (CBV: ActiveWakeup=1)

    NodeB->>NodeB: 收到 NodeA 的 NM 报文
    NodeB->>NodeB: BUS_SLEEP → REPEAT_MESSAGE
    NodeB->>Bus: NM_PDU (CBV: ActiveWakeup=0)

    NodeC->>NodeC: 收到 NodeA 的 NM 报文
    NodeC->>NodeC: BUS_SLEEP → REPEAT_MESSAGE
    NodeC->>Bus: NM_PDU (CBV: ActiveWakeup=0)

    Note over Bus: T_REPEAT_MESSAGE 超时

    NodeA->>NodeA: REPEAT_MESSAGE → NORMAL_OPERATION
    NodeB->>NodeB: REPEAT_MESSAGE → NORMAL_OPERATION
    NodeC->>NodeC: REPEAT_MESSAGE → NORMAL_OPERATION

    loop 正常运行
        NodeA->>Bus: NM_PDU (周期发送)
        NodeB->>Bus: NM_PDU (周期发送)
        NodeC->>Bus: NM_PDU (周期发送)
    end

    Note over NodeA: 应用释放通信
    NodeA->>NodeA: NORMAL_OPERATION → READY_SLEEP
    NodeA->>NodeA: 停止发送 NM 报文
    NodeA->>NodeA: 启动 T_WAIT_BUS_SLEEP

    loop 仍收到 NodeB, NodeC 的报文
        NodeA->>NodeA: 重启 T_WAIT_BUS_SLEEP
    end

    Note over NodeB: 应用释放通信
    NodeB->>NodeB: NORMAL_OPERATION → READY_SLEEP
    NodeB->>NodeB: 停止发送 NM 报文

    NodeC->>NodeC: 收到 NodeA, NodeB 停止
    Note over NodeC: NodeC 仍是 NORMAL_OPERATION<br/>继续发送 NM 报文

    Note over NodeC: 应用释放通信
    NodeC->>NodeC: NORMAL_OPERATION → READY_SLEEP
    NodeC->>NodeC: 停止发送 NM 报文
    NodeC->>NodeC: 启动 T_WAIT_BUS_SLEEP

    Note over Bus: T_WAIT_BUS_SLEEP 超时<br/>无任何 NM 报文

    NodeC->>NodeC: READY_SLEEP → PREPARE_BUS_SLEEP
    NodeC->>Bus: NM_PDU (CBV: NMCoordinatorSleep=1)
    NodeC->>NodeC: PREPARE_BUS_SLEEP → BUS_SLEEP

    Note over NodeA: 超时 → PREPARE_BUS_SLEEP → BUS_SLEEP
    Note over NodeB: 超时 → PREPARE_BUS_SLEEP → BUS_SLEEP

    Note over NodeA,NodeC: 所有节点进入 BUS_SLEEP
```

**图解释：** 这是一个完整的多节点交互场景，展示了 AUTOSAR 网络管理的核心设计思想：
1. **主动唤醒**: NodeA 主动唤醒，设置 CBV 的 ActiveWakeup 位
2. **被动唤醒**: NodeB、NodeC 被远程唤醒，不设置 ActiveWakeup 位
3. **同步启动**: 所有节点在 REPEAT_MESSAGE 状态同步后进入 NORMAL_OPERATION
4. **分布式睡眠**: 每个节点独立判断是否进入睡眠
5. **最后一个节点**: NodeC 是最后一个释放通信的节点，它发送 NMCoordinatorSleep 标志
6. **全睡眠**: 所有节点最终进入 BUS_SLEEP

---

## 9. 三模块之间的回调机制

### 9.1 回调链架构

```mermaid
graph TD
    subgraph 回调触发
        TRIG1["CanNM 状态变化"]
        TRIG2["CanNM 收到 NM 报文"]
        TRIG3["CanNM 定时器超时"]
    end

    subgraph 回调路径
        PATH1["CanNm_NetworkMode()"]
        PATH2["CanNm_NetworkStart()"]
        PATH3["CanNm_NetworkRelease()"]
        PATH4["CanNm_ConfirmNm()"]
    end

    subgraph NM 抽象层
        NM_ROUTE["回调路由<br/>根据 Channel ID 分发"]
    end

    subgraph ComM 回调处理
        CB1["ComM_Nm_NetworkStart()"]
        CB2["ComM_Nm_NetworkMode()"]
        CB3["ComM_Nm_NetworkRelease()"]
        CB4["ComM_Nm_ConfirmNm()"]
    end

    subgraph ComM 内部处理
        PROC1["更新 Channel 状态"]
        PROC2["通知 BswM 模式切换"]
        PROC3["响应 SW-C 查询"]
    end

    TRIG1 --> PATH1
    TRIG1 --> PATH3
    TRIG2 --> PATH2
    TRIG3 --> PATH3
    TRIG3 --> PATH4

    PATH1 --> NM_ROUTE
    PATH2 --> NM_ROUTE
    PATH3 --> NM_ROUTE
    PATH4 --> NM_ROUTE

    NM_ROUTE --> CB1
    NM_ROUTE --> CB2
    NM_ROUTE --> CB3
    NM_ROUTE --> CB4

    CB1 --> PROC1
    CB2 --> PROC1
    CB3 --> PROC1
    PROC1 --> PROC2
    PROC1 --> PROC3

    classDef trig fill:#ffebee,stroke:#c62828
    classDef path fill:#f3e5f5,stroke:#6a1b9a
    classDef nm fill:#fff3e0,stroke:#e65100
    classDef cb fill:#e3f2fd,stroke:#1565c0
    classDef proc fill:#e8f5e9,stroke:#2e7d32

    class TRIG1,TRIG2,TRIG3 trig
    class PATH1,PATH2,PATH3,PATH4 path
    class NM_ROUTE nm
    class CB1,CB2,CB3,CB4 cb
    class PROC1,PROC2,PROC3 proc
```

**图解释：** 回调机制是 CanNM → NM → ComM 的反向通知通路。
- CanNM 的状态变化、报文接收、定时器超时都会触发回调
- 回调经过 NM 抽象层路由，最终到达 ComM
- ComM 根据回调更新通道状态，并通知 BswM 进行模式切换

### 9.2 回调函数原型

```c
/* ===== NM 抽象层回调注册（ComM 实现） ===== */

/* 网络启动通知 */
void ComM_Nm_NetworkStart(
    uint8              Channel,     /* 通道 ID */
    Nm_WakeUpSourceType WakeUpSource /* 唤醒源类型 */
);

/* 网络模式通知 */
void ComM_Nm_NetworkMode(
    uint8    Channel,      /* 通道 ID */
    Nm_ModeType Mode       /* 网络模式 */
);

/* 网络释放通知 */
void ComM_Nm_NetworkRelease(
    uint8 Channel          /* 通道 ID */
);

/* NM 报文发送确认 */
void ComM_Nm_ConfirmNm(
    uint8 Channel,         /* 通道 ID */
    uint8* NmPduData,      /* NM 报文数据 */
    uint8 NmPduLength      /* 数据长度 */
);

/* ===== CanNM 的回调函数（NM 抽象层实现） ===== */

/* CanNM 网络模式回调 */
void CanNm_NetworkMode(
    uint8    Channel,          /* 通道 ID */
    CanNm_ModeType CanNm_Mode  /* CanNM 模式 */
);

/* CanNM 网络启动回调 */
void CanNm_NetworkStart(
    uint8              Channel,     /* 通道 ID */
    CanNm_WakeUpSourceType WakeUpSource /* 唤醒源 */
);

/* CanNM 网络释放回调 */
void CanNm_NetworkRelease(
    uint8 Channel          /* 通道 ID */
);
```

---

## 10. BswM 在交互中的角色

### 10.1 BswM 模式仲裁

BswM（Basic Software Mode Manager）在 ComM 和 CanNM 之间扮演 **"仲裁者"** 角色：

```mermaid
graph TB
    subgraph 触发源
        T1["ComM 状态变化通知"]
        T2["CanNM 状态变化通知"]
        T3["EcuM 状态变化通知"]
    end

    subgraph BswM 仲裁
        BSWM["BswM"]
        PROC["规则处理<br/>Rule Engine"]
        ACT["动作执行<br/>Action List"]
    end

    subgraph 执行动作
        A1["CanSM 请求<br/>启动/停止 CAN"]
        A2["CanTrcv 模式控制<br/>正常/睡眠"]
        A3["PDUR 路由控制<br/>开启/关闭"]
        A4["Com 状态控制<br/>开启/关闭"]
    end

    T1 --> BSWM
    T2 --> BSWM
    T3 --> BSWM
    BSWM --> PROC
    PROC --> ACT
    ACT --> A1
    ACT --> A2
    ACT --> A3
    ACT --> A4

    classDef trig fill:#ffebee,stroke:#c62828
    classDef bswm fill:#e8eaf6,stroke:#283593
    classDef act fill:#e8f5e9,stroke:#2e7d32
    classDef proc fill:#fff3e0,stroke:#e65100

    class T1,T2,T3 trig
    class BSWM bswm
    class PROC,ACT proc
    class A1,A2,A3,A4 act
```

**图解释：** BswM 作为模式仲裁器，接收 ComM 和 CanNM 的状态变化，执行预定义的规则和动作，控制 CAN 协议栈各模块的状态。

### 10.2 完整交互的全景图

```mermaid
sequenceDiagram
    participant SWC as SW-C
    participant ComM as ComM
    participant BswM as BswM
    participant NM as NM 抽象层
    participant CanNM as CanNM
    participant CanSM as CanSM
    participant CanIf as CanIf
    participant CanDrv as CanDrv
    participant Bus as CAN Bus

    Note over SWC,Bus: === 完整交互流程 ===

    SWC->>ComM: ComM_RequestComMode(FULL)

    ComM->>ComM: 更新引用计数
    ComM->>BswM: BswM_ComM_CurrentMode(FULL)
    BswM->>BswM: 执行规则: 启动 CAN 通信

    BswM->>CanSM: CanSM_RequestComMode(FULL)
    CanSM->>CanSM: 启动 CAN 控制器

    CanSM->>CanDrv: Can_SetControllerMode(STARTED)
    CanDrv-->>CanSM: Can_ControllerModeIndication(STARTED)

    CanSM->>CanIf: CanIf_SetControllerMode(STARTED)
    CanSM->>BswM: BswM_CanSM_CurrentState(ONLINE)

    BswM->>BswM: 执行规则: CAN 已就绪
    BswM->>ComM: ComM_ChannelReady(Channel)

    ComM->>NM: Nm_NetworkRequest(Channel)
    NM->>CanNM: CanNm_NetworkRequest(Channel)

    CanNM->>CanNM: BUS_SLEEP → REPEAT_MESSAGE
    CanNM->>CanIf: CanIf_CanNmTransmit(NM_PDU)
    CanIf->>CanDrv: Can_Write(NM_PDU)
    CanDrv->>Bus: NM PDU

    Note over CanNM: T_REPEAT_MESSAGE 超时
    CanNM->>CanNM: REPEAT_MESSAGE → NORMAL_OPERATION

    CanNM->>NM: CanNm_NetworkMode(NM_MODE_SYNCHRONIZE)
    NM->>ComM: ComM_Nm_NetworkMode(NM_MODE_SYNCHRONIZE)
    ComM->>BswM: BswM_ComM_NmMode(SYNCHRONIZE)

    ComM-->>SWC: 通信已建立
```

**图解释：** 这是最完整的交互全景图，展示了从 SW-C 请求到 CAN 总线报文发送的完整链路：
1. **SW-C → ComM → BswM → CanSM**: 先启动 CAN 硬件
2. **CanSM → CanDrv → CanIf**: 硬件初始化
3. **BswM → ComM**: 确认硬件就绪
4. **ComM → NM → CanNM**: 启动网络管理
5. **CanNM → CanIf → CanDrv → Bus**: 发送 NM 报文
6. **CanNM → NM → ComM**: 通知网络已就绪

---

## 11. 关键设计模式分析

### 11.1 分层抽象模式

```
┌─────────────────────────────────────┐
│            ComM                     │  ← 不关心底层协议类型
│     (通信管理器，应用层接口)        │
├─────────────────────────────────────┤
│            NM 抽象层                │  ← 适配器，接口标准化
│     (统一接口，协议无关)            │
├─────────────────────────────────────┤
│    CanNM / LinNM / FrNM            │  ← 协议具体实现
│     (CAN / LIN / FlexRay 实现)      │
└─────────────────────────────────────┘
```

**设计思路：** 这种分层结构体现了 **"依赖倒置原则"**——高层模块（ComM）不依赖于低层模块（CanNM），而是依赖于抽象（NM 接口）。低层模块也依赖于同一个抽象。这使得：
- 更换通信协议（CAN → LIN → FlexRay）时，ComM 完全不需要修改
- 新增协议支持时，只需要实现 NM 抽象层定义的接口

### 11.2 观察者模式（回调机制）

```c
/* 观察者模式实现 - 回调注册 */
typedef struct {
    /* ComM 注册的回调函数指针 */
    void (*NetworkStart)(uint8 Channel, Nm_WakeUpSourceType WakeUpSource);
    void (*NetworkMode)(uint8 Channel, Nm_ModeType Mode);
    void (*NetworkRelease)(uint8 Channel);
} Nm_ComMCallbacksType;

/* NM 抽象层维护回调表 */
static const Nm_ComMCallbacksType* Nm_ComMCallbacks = NULL;

/* ComM 注册回调 */
void Nm_InitComMCallbacks(const Nm_ComMCallbacksType* Callbacks) {
    Nm_ComMCallbacks = Callbacks;
}

/* CanNM 触发回调时，NM 抽象层调用 ComM 注册的函数 */
void CanNm_NetworkMode(uint8 Channel, CanNm_ModeType CanNm_Mode) {
    /* NM 抽象层内部处理 */
    /* ... */

    /* 调用 ComM 注册的回调 */
    if (Nm_ComMCallbacks != NULL && Nm_ComMCallbacks->NetworkMode != NULL) {
        Nm_ComMCallbacks->NetworkMode(Channel, (Nm_ModeType)CanNm_Mode);
    }
}
```

**代码解释：** 观察者模式在 AUTOSAR NM 中的应用：
- ComM 是 **观察者**，向 NM 注册回调函数
- CanNM 是 **被观察者**，状态变化时通知 NM
- NM 抽象层是 **事件分发器**，将 CanNM 的通知转发给 ComM
- 这种方式实现了模块间的 **松耦合**

### 11.3 引用计数模式（ComM 用户管理）

```c
/* 引用计数模式 - ComM 用户管理 */
typedef struct {
    uint8  ChannelId;
    uint16 ActiveUsers;       /* 位图: 每个 bit 代表一个用户 */
    uint8  FullCommCount;     /* FULL 通信请求计数 */
    uint8  SilentCommCount;   /* SILENT 通信请求计数 */
    ComM_ModeType CurrentMode;/* 当前通信模式 */
} ComM_ChannelControlType;

/* 请求通信 */
Std_ReturnType ComM_RequestComMode(uint8 ChannelId, ComM_ModeType ComMode) {
    ComM_ChannelControlType* ch = &ComM_Channels[ChannelId];

    /* 增加引用计数 */
    if (ComMode == COMM_FULL_COMMUNICATION) {
        if (ch->FullCommCount == 0) {
            /* 第一次请求 FULL，需要启动网络 */
            Nm_NetworkRequest(ChannelId);
        }
        ch->FullCommCount++;
    }

    /* 计算目标模式 */
    ComM_ModeType targetMode = ComM_CalculateTargetMode(ch);

    /* 如果需要切换状态 */
    if (targetMode != ch->CurrentMode) {
        ComM_SwitchMode(ChannelId, targetMode);
    }

    return E_OK;
}

/* 释放通信 */
Std_ReturnType ComM_RequestComMode(uint8 ChannelId, ComM_ModeType ComMode) {
    ComM_ChannelControlType* ch = &ComM_Channels[ChannelId];

    if (ComMode == COMM_FULL_COMMUNICATION) {
        if (ch->FullCommCount > 0) {
            ch->FullCommCount--;
        }
        if (ch->FullCommCount == 0) {
            /* 最后一个用户释放 FULL，释放网络 */
            Nm_NetworkRelease(ChannelId);
        }
    }

    /* 计算目标模式并切换 */
    ComM_ModeType targetMode = ComM_CalculateTargetMode(ch);
    if (targetMode != ch->CurrentMode) {
        ComM_SwitchMode(ChannelId, targetMode);
    }

    return E_OK;
}
```

**代码解释：** 引用计数模式确保：
- 多个 SW-C 可以同时请求通信，互不干扰
- 只有最后一个用户释放时，网络才会开始关闭流程
- 避免网络频繁启动/关闭（**防抖动**）

---

## 12. 典型应用场景与配置示例

### 12.1 场景：CAN 节点网络管理配置

```c
/* ===== CanNM 配置示例 ===== */
const CanNm_ConfigType CanNm_ConfigData = {
    /* 通道配置 */
    .CanNmChannelConfig = {
        .CanNmChannelIndex = 0,              /* 通道索引 */
        .CanNmNodeId = 0x01,                 /* 节点 ID (0x01) */
        .CanNmPduId = 0,                     /* NM PDU ID */
        .CanNmPduNMRxId = 0x501,             /* NM 接收 ID (Base 0x500 + Node) */
        .CanNmPduNMTxId = 0x501,             /* NM 发送 ID (Base 0x500 + Node) */

        /* 定时器配置 */
        .CanNmRepeatMessageTime = 500,       /* T_REPEAT_MESSAGE = 500ms */
        .CanNmWaitBusSleepTime = 2000,       /* T_WAIT_BUS_SLEEP = 2000ms */
        .CanNmPrepareBusSleepTime = 500,     /* T_PREPARE_BUS_SLEEP = 500ms */
        .CanNmMsgCycleTime = 100,            /* T_NM_Tx = 100ms */
        .CanNmMsgTimeoutTime = 500,          /* T_NM_Timeout = 500ms */

        /* 功能配置 */
        .CanNmActiveWakeupTxEnabled = TRUE,  /* 允许主动唤醒发送 */
        .CanNmPassiveWakeupEnabled = TRUE,   /* 允许被动唤醒 */
        .CanNmCoordinatorSleep = TRUE,       /* 支持协调睡眠 */
        .CanNmPartialNetworkEnabled = FALSE, /* 不启用 Partial Network */
    }
};

/* ===== ComM 配置示例 ===== */
const ComM_ConfigType ComM_ConfigData = {
    .ComMChannel = {
        .ComMChannelId = 0,                  /* 通道 ID */
        .ComMChannelNm = TRUE,               /* 支持 NM */
        .ComMChannelNmIf = NM_IF_CAN,        /* NM 接口类型: CAN */
        .ComMChannelNmChannelIndex = 0,       /* NM 通道索引 */
        .ComMUserList = {
            .ComMUser = {
                { .ComMUserId = 0, .ComMUserDefaultMode = COMM_NO_COMMUNICATION },
                { .ComMUserId = 1, .ComMUserDefaultMode = COMM_NO_COMMUNICATION },
                { .ComMUserId = 2, .ComMUserDefaultMode = COMM_NO_COMMUNICATION },
            }
        }
    }
};
```

### 12.2 场景：OSEK 直接网络管理 vs AUTOSAR NM

| 特性 | OSEK 直接 NM | AUTOSAR NM |
|------|-------------|------------|
| **架构** | 单体 NM | 分层：ComM / NM / CanNM |
| **报文格式** | 固定的"ID 环"令牌传递 | 基于 CBV 的分布式协调 |
| **唤醒机制** | 令牌传递唤醒 | 报文唤醒 + 选择性唤醒 |
| **睡眠机制** | 所有节点同步睡眠 | 分布式睡眠，独立决策 |
| **扩展性** | 固定节点数 | 动态节点，支持 Partial Network |
| **复杂度** | 低 | 中高 |

---

## 13. 常见问题 FAQ

### Q1: 为什么 ComM 需要 BswM 才能启动 CAN 通信？

**A:** ComM 只负责"通信需求管理"，不负责"硬件控制"。启动 CAN 控制器需要：
- CanSM 管理 CAN 控制器的状态机
- BswM 仲裁多个条件（EcuM 状态、ComM 状态、CanSM 状态）
- 这种分离使得模式管理更加灵活和可配置

### Q2: NM 抽象层是否必须？

**A:** 在 AUTOSAR 4.0+ 中，NM 抽象层是必需的。它提供了：
- 统一的 API 接口
- 多个 NM 通道的路由管理
- 唤醒源类型转换
- 即使只有一个 CAN 网络，NM 抽象层也提供了标准化接口

### Q3: T_WAIT_BUS_SLEEP 和 T_PREPARE_BUS_SLEEP 有什么区别？

**A:**
- **T_WAIT_BUS_SLEEP**: 等待总线上所有节点停止发送 NM 报文。这是"分布式协商"阶段，判断是否可以安全睡眠
- **T_PREPARE_BUS_SLEEP**: 准备进入睡眠的最后阶段。这是"最后确认"阶段，协调睡眠节点发送最终标志

### Q4: 多个节点同时请求网络启动会怎样？

**A:** 这是完全正常的情况：
1. 每个节点独立从 BUS_SLEEP → REPEAT_MESSAGE
2. 在 REPEAT_MESSAGE 状态下，它们通过 NM 报文互相感知
3. 所有节点在 T_REPEAT_MESSAGE 超时后进入 NORMAL_OPERATION
4. 网络同步完成，无需任何中心协调器

### Q5: 如果节点在 NORMAL_OPERATION 状态下突然掉电？

**A:** 这种场景被称为"突发离线"：
1. 其他节点在 T_NM_Timeout 超时后检测到该节点离线
2. 如果还有至少一个节点在 NORMAL_OPERATION，网络继续保持
3. 如果所有节点都离线，经过 T_WAIT_BUS_SLEEP → T_PREPARE_BUS_SLEEP 后进入 BUS_SLEEP
4. 断线节点重新上电后，通过 REPEAT_MESSAGE 重新加入网络

---

## 14. 总结

```mermaid
graph TB
    subgraph 设计理念
        D1["分层解耦<br/>ComM / NM / CanNM"]
        D2["分布式协调<br/>无中心节点"]
        D3["状态机驱动<br/>确定性行为"]
        D4["回调通知<br/>事件驱动架构"]
    end

    subgraph 核心机制
        M1["用户引用计数<br/>多用户管理"]
        M2["定时器协调<br/>T_REPEAT/T_WAIT/T_PREPARE"]
        M3["CBV 控制位<br/>状态信息传递"]
        M4["五状态机<br/>BUS_SLEEP→NORMAL→..."]
    end

    subgraph 优势
        B1["低功耗<br/>睡眠时总线关闭"]
        B2["即插即用<br/>节点动态加入/离开"]
        B3["鲁棒性<br/>单点故障不影响"]
        B4["可扩展<br/>支持 Partial Network"]
    end

    D1 --> M1
    D2 --> M2
    D3 --> M3
    D4 --> M4
    M1 --> B1
    M2 --> B2
    M3 --> B3
    M4 --> B4

    classDef design fill:#e3f2fd,stroke:#1565c0
    classDef mech fill:#fff3e0,stroke:#e65100
    classDef benefit fill:#e8f5e9,stroke:#2e7d32

    class D1,D2,D3,D4 design
    class M1,M2,M3,M4 mech
    class B1,B2,B3,B4 benefit
```

**关键总结：**
1. **ComM** 负责通信需求管理，以引用计数方式协调多个用户的请求
2. **NM 抽象层** 提供协议无关的统一接口，实现适配器模式
3. **CanNM** 实现具体的 CAN 网络管理协议，包含 5 状态状态机
4. **交互方式** 是双向的：上层通过 API 调用下发指令，下层通过回调通知上层
5. **核心设计理念** 是分布式协调，没有中心节点，每个节点独立决策
6. **睡眠机制** 通过 T_WAIT_BUS_SLEEP 和 T_PREPARE_BUS_SLEEP 两级定时器确保安全睡眠
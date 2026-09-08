# COMM_NM_FULL 与 COMM_NM_LIGHT 详解

> 结合 AUTOSAR 规范 ECUC_ComM_00568 和 ECUC_ComM_00607 解释

---

## 1. 先说结论

| 配置值 | 本质 | 一句话概括 |
|--------|------|-----------|
| **COMM_NM_FULL** | 完整网络管理变体 | 节点参与完整的 NM 协调，发送/接收 NM 报文，参与分布式睡眠 |
| **COMM_NM_LIGHT** | 轻量网络管理变体 | 节点仅做基本的唤醒/睡眠管理，不参与 NM 协调，不发送 NM 报文 |

这两个值不是 ComM 的模式，而是 **`ComMNmVariant` 配置参数的可选值**，用于告诉 ComM：**这个通道底层的 NM 实现是"完整版"还是"轻量版"**，从而决定 ComM 对该通道的行为策略。

---

## 2. AUTOSAR 规范出处

### 2.1 ECUC_ComM_00568 — `ComMNmVariant`

```xml
<!-- AUTOSAR ECU Configuration 参数定义 -->
<ECUC_PARAMETER_DEF>
    <SHORT-NAME>ComMNmVariant</SHORT-NAME>
    <PARAMETER-NAME>ComMNmVariant</PARAMETER-NAME>
    <PARENT-REF>ComMChannel</PARENT-REF>
    <TYPE>ECUC_ENUMERATION</TYPE>
    <IMPLEMENTATION-TYPE>ComMNmVariantType</IMPLEMENTATION-TYPE>
    <ECUC-VALUE-REF>COMM_NM_FULL</ECUC-VALUE-REF>
    <ECUC-VALUE-REF>COMM_NM_LIGHT</ECUC-VALUE-REF>
    <ECUC-VALUE-REF>COMM_NM_PASSIVE</ECUC-VALUE-REF>
    <ECUC-VALUE-REF>COMM_NM_NO_COMMUNICATION</ECUC-VALUE-REF>
</ECUC_PARAMETER_DEF>
```

**官方定义**：该参数指定了通道所使用的网络管理变体。ComM 根据此参数决定通道的 NM 行为策略。

### 2.2 ECUC_ComM_00607 — `ComMUserIdentifier`

```xml
<ECUC_PARAMETER_DEF>
    <SHORT-NAME>ComMUserIdentifier</SHORT-NAME>
    <PARENT-REF>ComMUser</PARENT-REF>
    <TYPE>ECUC_INTEGER</TYPE>
    <IMPLEMENTATION-TYPE>ComM_UserHandleType</IMPLEMENTATION-TYPE>
    <LOWER-MULTIPLE>1</LOWER-MULTIPLE>
    <UPPER-MULTIPLE>1</UPPER-MULTIPLE>
</ECUC_PARAMETER_DEF>
```

**官方定义**：该参数定义了 ComM 用户（SW-C）的唯一标识符，用于 ComM 内部管理用户请求时区分不同的用户。

---

## 3. `ComMNmVariant` 的完整枚举值

```c
/* AUTOSAR 标准定义 */
typedef enum {
    COMM_NM_NO_COMMUNICATION = 0,  /* 无网络管理 */
    COMM_NM_LIGHT            = 1,  /* 轻量网络管理 */
    COMM_NM_FULL             = 2,  /* 完整网络管理 */
    COMM_NM_PASSIVE          = 3   /* 被动网络管理 */
} ComMNmVariantType;
```

```mermaid
graph TB
    subgraph ComMNmVariantType 四种变体
        direction TB

        subgraph FULL["COMM_NM_FULL (2)"]
            F1["发送 NM 报文 ✓"]
            F2["接收 NM 报文 ✓"]
            F3["参与分布式睡眠协调 ✓"]
            F4["主动唤醒能力 ✓"]
            F5["协调睡眠（Co-Sleep）✓"]
            F6["Partial Network 支持 ✓"]
        end

        subgraph LIGHT["COMM_NM_LIGHT (1)"]
            L1["发送 NM 报文 ✗"]
            L2["接收 NM 报文 ✓（仅唤醒检测）"]
            L3["分布式睡眠协调 ✗"]
            L4["主动唤醒能力 ✗"]
            L5["协调睡眠 ✗"]
            L6["Partial Network 支持 ✗"]
        end

        subgraph PASSIVE["COMM_NM_PASSIVE (3)"]
            P1["发送 NM 报文 ✗"]
            P2["接收 NM 报文 ✓"]
            P3["被动响应唤醒 ✓"]
            P4["不参与协调"]
            P5["仅监听总线活动"]
        end

        subgraph NO_COMM["COMM_NM_NO_COMMUNICATION (0)"]
            N1["NM 完全禁用"]
            N2["无 NM 报文收发"]
            N3["无唤醒管理"]
            N4["ComM 直接控制通信"]
        end
    end

    classDef full fill:#e3f2fd,stroke:#1565c0
    classDef light fill:#fff3e0,stroke:#e65100
    classDef passive fill:#f3e5f5,stroke:#4a148c
    classDef none fill:#f5f5f5,stroke:#9e9e9e

    class F1,F2,F3,F4,F5,F6 full
    class L1,L2,L3,L4,L5,L6 light
    class P1,P2,P3,P4 passive
    class N1,N2,N3,N4 none
```

**图解释：** 四种 NM 变体的能力对比。`COMM_NM_FULL` 具备全部能力；`COMM_NM_LIGHT` 仅保留基本的唤醒检测；`COMM_NM_PASSIVE` 只能被动响应；`COMM_NM_NO_COMMUNICATION` 完全禁用 NM。

---

## 4. COMM_NM_FULL（完整网络管理）

### 4.1 配置场景

```xml
<!-- 配置示例：COMM_NM_FULL -->
<ComMChannel>
    <SHORT-NAME>ComMChannel_CAN0</SHORT-NAME>
    <ComMNmVariant>COMM_NM_FULL</ComMNmVariant>  <!-- ECUC_ComM_00568 -->
    <ComMChannelNmIf>NM_IF_CAN</ComMChannelNmIf>
    <ComMChannelNmChannelIndex>0</ComMChannelNmChannelIndex>
    <ComMUserList>
        <ComMUser>
            <ComMUserIdentifier>0</ComMUserIdentifier>  <!-- ECUC_ComM_00607 -->
            <ComMUserDefaultMode>COMM_NO_COMMUNICATION</ComMUserDefaultMode>
        </ComMUser>
    </ComMUserList>
</ComMChannel>
```

### 4.2 行为特征

当 `ComMNmVariant = COMM_NM_FULL` 时，ComM 对该通道的完整行为如下：

```c
/* ComM 内部行为 - FULL 变体 */
void ComM_ChannelMainFunction(ComM_ChannelType* ch) {
    /* 只有 FULL 变体才会调用 NM 接口 */
    if (ch->NmVariant == COMM_NM_FULL) {
        /* 用户请求 → 启动 NM */
        if (ch->FullCommCount > 0 && !ch->NmActive) {
            Nm_NetworkRequest(ch->ChannelId);  /* 调用 NM 启动网络 */
            ch->NmActive = TRUE;
        }

        /* 用户释放 → 停止 NM */
        if (ch->FullCommCount == 0 && ch->NmActive) {
            Nm_NetworkRelease(ch->ChannelId);  /* 调用 NM 释放网络 */
            ch->NmActive = FALSE;
        }
    }
}

/* FULL 变体对回调的处理 */
void ComM_Nm_NetworkMode(uint8 Channel, Nm_ModeType Mode) {
    ComM_ChannelType* ch = &ComM_Channels[Channel];

    /* FULL 变体：处理 NM 回调 */
    if (ch->NmVariant == COMM_NM_FULL) {
        if (Mode == NM_MODE_SYNCHRONIZE) {
            /* 网络同步完成，可以开启通信 */
            ch->CurrentMode = COMM_FULL_COMMUNICATION;
            BswM_ComM_CurrentMode(Channel, COMM_FULL_COMMUNICATION);

            /* BswM 执行规则:
             * → 开启 Com 路由
             * → 开启 PduR 路由
             * → 通知 CanSM 保持 ONLINE
             */
        }
    }
}
```

### 4.3 适用场景

- **需要参与网络协调的 ECU**（多个 ECU 之间需要同步睡眠/唤醒）
- **网关节点**（必须参与多个网络的协调）
- **需要 Partial Network 支持的 ECU**
- **需要主动唤醒网络的 ECU**（如诊断工具、主控节点）

---

## 5. COMM_NM_LIGHT（轻量网络管理）

### 5.1 配置场景

```xml
<!-- 配置示例：COMM_NM_LIGHT -->
<ComMChannel>
    <SHORT-NAME>ComMChannel_LIN0</SHORT-NAME>
    <ComMNmVariant>COMM_NM_LIGHT</ComMNmVariant>  <!-- ECUC_ComM_00568 -->
    <ComMChannelNmIf>NM_IF_LIN</ComMChannelNmIf>
    <ComMChannelNmChannelIndex>0</ComMChannelNmChannelIndex>
    <ComMUserList>
        <ComMUser>
            <ComMUserIdentifier>0</ComMUserIdentifier>  <!-- ECUC_ComM_00607 -->
            <ComMUserDefaultMode>COMM_NO_COMMUNICATION</ComMUserDefaultMode>
        </ComMUser>
    </ComMUserList>
</ComMChannel>
```

### 5.2 行为特征

```c
/* ComM 内部行为 - LIGHT 变体 */
void ComM_LightNmChannelMainFunction(ComM_ChannelType* ch) {
    /* LIGHT 变体：不调用 Nm_NetworkRequest/Release */
    if (ch->NmVariant == COMM_NM_LIGHT) {
        /* 用户请求 → 直接唤醒总线（不经过 NM 协调） */
        if (ch->FullCommCount > 0 && ch->CurrentMode == COMM_NO_COMMUNICATION) {
            /* 直接请求 CanSM 唤醒总线，不经过 NM 抽象层 */
            CanSM_RequestComMode(ch->ChannelId, CANSM_FULL_COMMUNICATION);
            ch->CurrentMode = COMM_FULL_COMMUNICATION;
            BswM_ComM_CurrentMode(ch->ChannelId, COMM_FULL_COMMUNICATION);
        }

        /* 用户释放 → 直接关闭总线 */
        if (ch->FullCommCount == 0 && ch->CurrentMode != COMM_NO_COMMUNICATION) {
            CanSM_RequestComMode(ch->ChannelId, CANSM_NO_COMMUNICATION);
            ch->CurrentMode = COMM_NO_COMMUNICATION;
            BswM_ComM_CurrentMode(ch->ChannelId, COMM_NO_COMMUNICATION);
        }
    }
}

/* LIGHT 变体：不处理 NM 回调（因为没有 NM 协调） */
/* ComM_Nm_NetworkStart / ComM_Nm_NetworkMode / ComM_Nm_NetworkRelease
 * 这些回调函数对 LIGHT 变体通道不会被调用 */
```

### 5.3 适用场景

- **不需要网络协调的简单 ECU**（如传感器节点、执行器节点）
- **LIN 网络**（LIN 通常使用 LIGHT NM）
- **仅需本地唤醒的 ECU**
- **资源受限的 ECU**（减少代码量和内存占用）
- **网络中的叶子节点**（不参与协调，仅响应主节点）

---

## 6. FULL vs LIGHT 的完整对比

### 6.1 功能对比表

```mermaid
graph LR
    subgraph 对比维度
        DIM1["NM 报文发送"]
        DIM2["NM 报文接收"]
        DIM3["分布式睡眠"]
        DIM4["主动唤醒"]
        DIM5["协调睡眠"]
        DIM6["Partial Network"]
        DIM7["Nm_NetworkRequest 调用"]
        DIM8["Nm_NetworkRelease 调用"]
        DIM9["代码量/内存"]
        DIM10["通信延迟"]
    end

    subgraph FULL
        F1["✓ 周期性发送"]
        F2["✓ 监听所有节点"]
        F3["✓ 参与协调"]
        F4["✓ 主动唤醒"]
        F5["✓ 支持"]
        F6["✓ 可配置"]
        F7["✓ 调用"]
        F8["✓ 调用"]
        F9["较大"]
        F10["较低（主动同步）"]
    end

    subgraph LIGHT
        L1["✗ 不发送"]
        L2["✓ 仅检测唤醒"]
        L3["✗ 不参与"]
        L4["✗ 仅被动"]
        L5["✗ 不支持"]
        L6["✗ 不支持"]
        L7["✗ 不调用"]
        L8["✗ 不调用"]
        L9["较小"]
        L10["较高（依赖主节点）"]
    end

    DIM1 --> F1
    DIM1 --> L1
    DIM2 --> F2
    DIM2 --> L2
    DIM3 --> F3
    DIM3 --> L3
    DIM4 --> F4
    DIM4 --> L4
    DIM5 --> F5
    DIM5 --> L5
    DIM6 --> F6
    DIM6 --> L6
    DIM7 --> F7
    DIM7 --> L7
    DIM8 --> F8
    DIM8 --> L8
    DIM9 --> F9
    DIM9 --> L9
    DIM10 --> F10
    DIM10 --> L10

    classDef dim fill:#e8eaf6,stroke:#283593,stroke-width:1px
    classDef full fill:#e3f2fd,stroke:#1565c0,stroke-width:2px
    classDef light fill:#fff3e0,stroke:#e65100,stroke-width:2px

    class DIM1,DIM2,DIM3,DIM4,DIM5,DIM6,DIM7,DIM8,DIM9,DIM10 dim
    class F1,F2,F3,F4,F5,F6,F7,F8,F9,F10 full
    class L1,L2,L3,L4,L5,L6,L7,L8,L9,L10 light
```

### 6.2 状态机差异

```mermaid
stateDiagram-v2
    state COMM_NM_FULL {
        [*] --> BUS_SLEEP
        BUS_SLEEP --> REPEAT_MESSAGE
        REPEAT_MESSAGE --> NORMAL_OPERATION
        NORMAL_OPERATION --> READY_SLEEP
        READY_SLEEP --> PREPARE_BUS_SLEEP
        PREPARE_BUS_SLEEP --> BUS_SLEEP
        READY_SLEEP --> REPEAT_MESSAGE
        PREPARE_BUS_SLEEP --> REPEAT_MESSAGE

        note right of BUS_SLEEP
            CanNM 完整 5 状态机
            T_REPEAT_MESSAGE
            T_WAIT_BUS_SLEEP
            T_PREPARE_BUS_SLEEP
            三种定时器完整运行
        end note
    }

    state COMM_NM_LIGHT {
        [*] --> SLEEP
        SLEEP --> AWAKE: 收到报文/本地请求
        AWAKE --> SLEEP: 通信结束

        note right of SLEEP
            简化 2 状态机
            无 T_REPEAT_MESSAGE
            无 T_WAIT_BUS_SLEEP
            无 NM 报文发送
            仅检测总线活动
        end note
    }
```

**图解释：** FULL 使用 CanNM 完整的 5 状态状态机，而 LIGHT 使用简化的 2 状态状态机（SLEEP ↔ AWAKE），没有网络协调的复杂状态。

### 6.3 时序对比

**FULL 变体的通信时序：**

```mermaid
sequenceDiagram
    participant SWC as SW-C
    participant ComM as ComM
    participant NM as NM 抽象层
    participant CanNM as CanNM
    participant Bus as CAN Bus

    SWC->>ComM: RequestComMode(FULL)
    ComM->>NM: Nm_NetworkRequest()
    NM->>CanNM: CanNm_NetworkRequest()
    CanNM->>CanNM: BUS_SLEEP → RM → NO
    CanNM->>NM: CanNm_NetworkMode(SYNC)
    NM->>ComM: ComM_Nm_NetworkMode(SYNC)
    Note over ComM: 网络同步后才返回
    Note over Bus: 有 NM 报文交互
    ComM-->>SWC: 通信已建立（延迟较大）
```

**LIGHT 变体的通信时序：**

```mermaid
sequenceDiagram
    participant SWC as SW-C
    participant ComM as ComM
    participant CanSM as CanSM
    participant CanDrv as CAN 驱动
    participant Bus as CAN Bus

    SWC->>ComM: RequestComMode(FULL)
    Note over ComM: 不经过 NM 抽象层
    ComM->>CanSM: CanSM_RequestComMode(FULL)
    CanSM->>CanDrv: Can_SetControllerMode(STARTED)
    CanDrv->>Bus: 直接通信
    Note over Bus: 无 NM 报文交互
    ComM-->>SWC: 通信已建立（延迟较小）
```

---

## 7. ECUC_ComM_00607（ComMUserIdentifier）的作用

### 7.1 配置定义

`ComMUserIdentifier` 是 `ComMUser` 配置容器的参数，用于唯一标识一个 ComM 用户。

### 7.2 核心作用

```c
/* 用户标识符在 ComM 内部的使用 */
typedef struct {
    ComM_UserHandleType  UserId;           /* = ComMUserIdentifier */
    ComM_ModeType        RequestedMode;    /* 当前请求的通信模式 */
    ComM_ModeType        DefaultMode;      /* 默认通信模式 */
    uint8                ChannelId;        /* 所属通道 */
    boolean              IsActive;         /* 用户是否活跃 */
} ComM_InternalUserType;

/* 用户管理 - 使用 UserId 区分请求者 */
Std_ReturnType ComM_RequestComMode(ComM_UserHandleType UserId,
                                    uint8 ChannelId,
                                    ComM_ModeType ComMode) {
    ComM_ChannelType* ch = &ComM_Channels[ChannelId];

    /* 查找用户 */
    for (int i = 0; i < ch->UserCount; i++) {
        if (ch->Users[i].UserId == UserId) {
            /* 更新该用户的请求模式 */
            ComM_ModeType oldMode = ch->Users[i].RequestedMode;
            ch->Users[i].RequestedMode = ComMode;

            /* 更新引用计数 */
            if (oldMode == COMM_FULL_COMMUNICATION &&
                ComMode != COMM_FULL_COMMUNICATION) {
                ch->FullCommCount--;
            }
            if (oldMode != COMM_FULL_COMMUNICATION &&
                ComMode == COMM_FULL_COMMUNICATION) {
                ch->FullCommCount++;
            }

            /* 重新计算目标模式 */
            ComM_ChannelRequestEvaluation(ch);
            return E_OK;
        }
    }
    return E_NOT_OK;
}
```

### 7.3 与 ECUC_ComM_00568 的关系

```mermaid
graph TB
    subgraph ComM 通道配置
        CHANNEL["ComMChannel<br/>（一个通信通道）"]
        VARIANT["ComMNmVariant (ECUC_ComM_00568)<br/>= COMM_NM_FULL / LIGHT / PASSIVE / NO_COMM"]
    end

    subgraph ComM 用户配置
        USER1["ComMUser 1<br/>UserIdentifier (ECUC_ComM_00607) = 0"]
        USER2["ComMUser 2<br/>UserIdentifier (ECUC_ComM_00607) = 1"]
        USER3["ComMUser 3<br/>UserIdentifier (ECUC_ComM_00607) = 2"]
    end

    subgraph 运行时行为
        BEHAVIOR["根据 NmVariant 决定<br/>FULL: 调用 NM 接口<br/>LIGHT: 直接调用 CanSM"]
    end

    CHANNEL --> VARIANT
    CHANNEL --> USER1
    CHANNEL --> USER2
    CHANNEL --> USER3
    VARIANT --> BEHAVIOR
    USER1 --> BEHAVIOR
    USER2 --> BEHAVIOR
    USER3 --> BEHAVIOR

    classDef chan fill:#e3f2fd,stroke:#1565c0
    classDef user fill:#fff3e0,stroke:#e65100
    classDef bhv fill:#f3e5f5,stroke:#4a148c

    class CHANNEL,VARIANT chan
    class USER1,USER2,USER3 user
    class BEHAVIOR bhv
```

**图解释：** `ComMNmVariant`（ECUC_ComM_00568）定义通道的 NM 行为策略，`ComMUserIdentifier`（ECUC_ComM_00607）定义该通道上每个用户的 ID。两者共同决定 ComM 如何处理该通道上的用户请求。

---

## 8. 实际工程中的选择建议

### 8.1 选择决策树

```mermaid
graph TD
    START["选择 NM 变体"] --> Q1{"需要网络协调？<br/>（分布式睡眠/唤醒）"}

    Q1 -->|"是"| Q2{"需要主动唤醒？<br/>（本地触发网络启动）"}
    Q1 -->|"否"| Q3{"需要监听 NM 报文？"}

    Q2 -->|"是"| FULL["COMM_NM_FULL<br/>完整 NM 协调"]
    Q2 -->|"否"| PASSIVE["COMM_NM_PASSIVE<br/>被动 NM"]

    Q3 -->|"是"| LIGHT["COMM_NM_LIGHT<br/>轻量 NM"]
    Q3 -->|"否"| NO_COM["COMM_NM_NO_COMMUNICATION<br/>无 NM"]

    FULL --> EX1["网关、主控、诊断节点<br/>CAN 骨干网络"]
    PASSIVE --> EX2["仅响应唤醒的节点<br/>监测节点"]
    LIGHT --> EX3["传感器节点、执行器<br/>LIN 从节点"]
    NO_COM --> EX4["纯应用节点<br/>无网络管理需求"]

    classDef start fill:#e8eaf6,stroke:#283593
    classDef q fill:#fff3e0,stroke:#e65100
    classDef choice fill:#e3f2fd,stroke:#1565c0
    classDef ex fill:#f5f5f5,stroke:#9e9e9e

    class START start
    class Q1,Q2,Q3 q
    class FULL,PASSIVE,LIGHT,NO_COM choice
    class EX1,EX2,EX3,EX4 ex
```

### 8.2 典型应用场景

| 场景 | 推荐变体 | 理由 |
|------|---------|------|
| **CAN 骨干网络**（多个 ECU 需要同步睡眠） | `COMM_NM_FULL` | 需要完整的 NM 协调 |
| **CAN 网关** | `COMM_NM_FULL` | 必须在多个网络间协调睡眠 |
| **LIN 从节点**（传感器） | `COMM_NM_LIGHT` | 不需要发送 NM 报文，仅响应主节点 |
| **LIN 主节点** | `COMM_NM_FULL` 或 `COMM_NM_LIGHT` | 取决于是否需要在 LIN 上做 NM 协调 |
| **简单执行器**（如车窗电机） | `COMM_NM_NO_COMMUNICATION` | 不需要 NM，直接控制通信 |
| **诊断工具** | `COMM_NM_FULL` | 需要主动唤醒网络 |
| **低功耗传感器**（电池供电） | `COMM_NM_PASSIVE` | 仅被动响应，不主动发送 |
| **FlexRay 节点** | `COMM_NM_FULL` | FlexRay 通常需要完整的 NM 协调 |

---

## 9. 总结

```mermaid
graph TB
    subgraph 核心概念
        C1["ECUC_ComM_00568 = ComMNmVariant<br/>通道的 NM 变体配置"]
        C2["ECUC_ComM_00607 = ComMUserIdentifier<br/>用户的唯一标识符"]
    end

    subgraph COMM_NM_FULL
        FULL["ComM 行为:<br/>调用 Nm_NetworkRequest/Release<br/>等待 NM 同步回调<br/>参与分布式睡眠协调<br/>适用: 需要 NM 协调的 ECU"]
    end

    subgraph COMM_NM_LIGHT
        LIGHT["ComM 行为:<br/>不调用 NM 接口<br/>直接控制 CanSM<br/>无分布式协调<br/>适用: 简单 ECU 和 LIN 从节点"]
    end

    C1 --> FULL
    C1 --> LIGHT
    C2 --> FULL
    C2 --> LIGHT

    classDef core fill:#e8eaf6,stroke:#283593,stroke-width:2px
    classDef full fill:#e3f2fd,stroke:#1565c0,stroke-width:2px
    classDef light fill:#fff3e0,stroke:#e65100,stroke-width:2px

    class C1,C2 core
    class FULL full
    class LIGHT light
```

**一句话总结：**
- **`COMM_NM_FULL`**（ECUC_ComM_00568 的值）= 完整 NM → ComM 调用 `Nm_NetworkRequest/Release`，参与完整的 NM 协调
- **`COMM_NM_LIGHT`**（ECUC_ComM_00568 的值）= 轻量 NM → ComM 跳过 NM 抽象层，直接控制 CanSM
- **`ComMUserIdentifier`**（ECUC_ComM_00607）= 用户的唯一 ID，ComM 内部用它来区分和管理多个用户的请求
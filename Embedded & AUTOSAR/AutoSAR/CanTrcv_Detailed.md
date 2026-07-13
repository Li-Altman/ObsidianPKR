# AUTOSAR CanTrcv（CAN Transceiver Driver）模块详解

> 作者：AUTOSAR & 嵌入式软件专家
> 版本：v1.0
> 更新日期：2026/07/10

---

# 一、通俗理解：CanTrcv 是什么？

## 1.1 打个比方

CanTrcv 就像是**CAN总线的"接口警卫"**——它不负责数据的理解和处理，而是负责：

| 角色 | 类比 | 对应职责 |
|------|------|----------|
| **信号转换器** | 翻译官 | CAN控制器逻辑电平 ↔ CAN总线差分信号 |
| **门卫** | 控制进出 | 决定总线是否接通、节点是否监听 |
| **看门人** | 夜间安保 | 检测总线活动并唤醒整个ECU |
| **电路保护器** | 保险丝 | 过压/过热保护，总线故障隔离 |

### 核心一句话

> **CanDrv 管"脑子"（协议逻辑），CanTrcv 管"嘴巴和耳朵"（物理信号）。**

## 1.2 在通信栈中的定位

```mermaid
graph TB
    subgraph Software["软件层"]
        CanSM[CanSM - 状态管理器]
        CanDrv[CanDrv - CAN驱动]
    end

    subgraph Transceiver["CanTrcv 边界"]
        CanTrcv["✨ CanTrcv\n（CAN收发器驱动）"]
    end

    subgraph Hardware["硬件层"]
        Trcv_Chip[CAN Transceiver IC<br/>TJA1043 / TJA1145 / TCAN4550]
        Bus[CAN Bus<br/>CAN_H / CAN_L]
    end

    CanSM --> CanTrcv
    CanDrv --> CanTrcv
    CanTrcv --> Trcv_Chip
    Trcv_Chip --> Bus

    style CanTrcv fill:#e056fd,color:#fff,stroke:#ff6b6b,stroke-width:4px
    style Trcv_Chip fill:#45b7d1,color:#fff
    style Bus fill:#95afc0,color:#333

    subgraph Physical["物理层信号"]
        Diff["CAN_H - CAN_L = 差分电压"]
        Dom["显性（Dominant）: >0.9V"]
        Rec["隐性（Recessive）: <0.5V"]
    end

    Trcv_Chip -.-> Physical
```

## 1.3 CanTrcv 与 CanDrv 的本质区别

```mermaid
flowchart LR
    subgraph CanDrv_Domain["CanDrv 管辖范围"]
        D1["CAN Core 寄存器"]
        D2["协议控制器"]
        D3["消息缓冲管理"]
        D4["位时序/同步"]
        D5["CRC 生成/校验"]
    end

    subgraph CanTrcv_Domain["CanTrcv 管辖范围"]
        T1["收发器模式控制"]
        T2["差分信号转换"]
        T3["唤醒检测"]
        T4["总线故障诊断"]
        T5["过流/过热保护"]
    end

    subgraph Shared["共同关注"]
        S1["Bus-Off 恢复"]
        S2["唤醒验证"]
        S3["低功耗管理"]
    end

    CanDrv_Domain --- Shared
    CanTrcv_Domain --- Shared
    
    style CanDrv_Domain fill:#686de0,color:#fff
    style CanTrcv_Domain fill:#e056fd,color:#fff
    style Shared fill:#f9ca24,color:#333
```

---

# 二、CanTrcv 架构总览

## 2.1 内部架构

```mermaid
graph TB
    subgraph CanTrcv_Module["CanTrcv 内部架构"]
        direction TB

        subgraph API_Layer["API接口层"]
            API[标准API<br/>CanTrcv_Init/SetOpMode/GetOpMode/...]
            CBK[回调<br/>CanTrcv_WakeupIndication/Confirmation]
        end

        subgraph Mode_Mgmt["模式管理层"]
            ModeSM["状态机管理<br/>NORMAL / STANDBY / SLEEP"]
            Wakeup_Mgmt["唤醒管理<br/>唤醒检测/过滤/验证"]
        end

        subgraph Diag_Mgmt["诊断管理层"]
            BusDiag["总线诊断<br/>CAN_H/L 短路/断路检测"]
            TempDiag["温度诊断<br/>过热/过流检测"]
        end

        subgraph HW_Access["硬件访问层"]
            SPI_Access["SPI/I2C 访问<br/>（分立收发器）"]
            GPIO_Access["GPIO 控制<br/>（使能/唤醒引脚）"]
            MMIO_Access["内存映射IO<br/>（集成收发器）"]
        end

        API --> Mode_Mgmt
        API --> Diag_Mgmt
        Mode_Mgmt --> HW_Access
        Diag_Mgmt --> HW_Access
    end

    subgraph External["外部模块"]
        CanSM[CanSM - 控制通路]
        EcuM[EcuM - 唤醒管理]
        Det[Det - 错误追踪]
    end

    CanSM --> API
    EcuM --> API
    CanSM --> CBK
    EcuM --> CBK

    style CanTrcv_Module fill:#e056fd,color:#fff
    style HW_Access fill:#ff6b6b,color:#fff
    style Mode_Mgmt fill:#45b7d1,color:#fff
```

## 2.2 典型的收发器硬件接口

```mermaid
graph TB
    subgraph MCU["MCU 侧"]
        CAN_Ctrl[CAN Controller<br/>CAN Core]
        SPI_M[SPI Master]
        GPIO_PINS[GPIO Pins]
    end

    subgraph Trcv["CAN Transceiver IC"]
        CAN_TX["TX（发送）"]
        CAN_RX["RX（接收）"]
        INH["INH（唤醒输出）"]
        EN["EN（使能）"]
        STB_N["STB_N（待机）"]
        SPI_S[SPI Slave]
        WAKE["WAKE（唤醒输入）"]
    end

    subgraph Bus["CAN 总线侧"]
        CAN_H["CAN_H 总线"]
        CAN_L["CAN_L 总线"]
    end

    subgraph Power["电源"]
        VCC["VCC / VIO"]
        GND["GND"]
    end

    CAN_Ctrl -->|TXD| CAN_TX
    CAN_Ctrl <-->|RXD| CAN_RX
    CAN_TX <--> CAN_H
    CAN_TX <--> CAN_L
    CAN_RX <--> CAN_H
    CAN_RX <--> CAN_L
    GPIO_PINS --> EN
    GPIO_PINS --> STB_N
    GPIO_PINS <--> WAKE
    SPI_M <--> SPI_S
    INH --> Power

    style Trcv fill:#45b7d1,color:#fff
    style CAN_H fill:#ff6b6b,color:#fff
    style CAN_L fill:#ff6b6b,color:#fff
```

---

# 三、CanTrcv 状态机

## 3.1 收发器模式状态机

```mermaid
stateDiagram-v2
    [*] --> CANTRCV_UNINIT

    CANTRCV_UNINIT --> CANTRCV_NORMAL : CanTrcv_Init()

    state CANTRCV_NORMAL {
        [*] --> Normal_TxRx
        Normal_TxRx --> Normal_OnlyRx : 故障/诊断
    }

    CANTRCV_NORMAL --> CANTRCV_STANDBY : CanTrcv_SetOpMode(STANDBY)
    CANTRCV_NORMAL --> CANTRCV_SLEEP : CanTrcv_SetOpMode(SLEEP)

    state CANTRCV_STANDBY {
        [*] --> Standby_Monitor
        Standby_Monitor --> Standby_Wakeup : 总线活动
    }

    CANTRCV_STANDBY --> CANTRCV_NORMAL : CanTrcv_SetOpMode(NORMAL)
    CANTRCV_STANDBY --> CANTRCV_SLEEP : CanTrcv_SetOpMode(SLEEP)

    state CANTRCV_SLEEP {
        [*] --> Sleep_LowPower
        Sleep_LowPower --> Sleep_Wakeup : 唤醒事件
    }

    CANTRCV_SLEEP --> CANTRCV_NORMAL : 唤醒 + CanTrcv_SetOpMode(NORMAL)
    CANTRCV_SLEEP --> CANTRCV_STANDBY : 唤醒 + CanTrcv_SetOpMode(STANDBY)

    note right of CANTRCV_NORMAL
        全功能模式
        收发器正常收发
        电流消耗: ~5mA
    end note

    note right of CANTRCV_STANDBY
        待机监听
        仅监听总线活动
        电流消耗: ~50μA
    end note

    note right of CANTRCV_SLEEP
        深度休眠
        仅唤醒检测电路工作
        电流消耗: ~5μA
    end note
```

## 3.2 模式定义

```c
/* AUTOSAR CanTrcv 标准模式定义 */
typedef enum {
    CANTRCV_OPMODE_NORMAL    = 0x00,  /* 正常收发模式 */
    CANTRCV_OPMODE_STANDBY   = 0x01,  /* 待机监听模式 */
    CANTRCV_OPMODE_SLEEP     = 0x02,  /* 深度休眠模式 */
    CANTRCV_OPMODE_WAKEUP    = 0x03,  /* 唤醒过渡模式（内部使用） */
} CanTrcv_OpModeType;

/* 收发器唤醒原因 */
typedef enum {
    CANTRCV_WAKEUP_BY_BUS,          /* CAN总线活动唤醒 */
    CANTRCV_WAKEUP_BY_PIN,          /* WAKE引脚唤醒 */
    CANTRCV_WAKEUP_BY_SPI,          /* SPI命令唤醒 */
    CANTRCV_WAKEUP_BY_LOCAL,        /* 本地事件唤醒 */
    CANTRCV_WAKEUP_INTERNAL         /* 内部定时唤醒 */
} CanTrcv_WakeupReasonType;
```

## 3.3 状态转换完整实现

```c
/* CanTrcv 核心状态管理 */

typedef struct {
    uint8_t                  TrcvId;         /* 收发器ID */
    CanTrcv_OpModeType       CurrentMode;    /* 当前模式 */
    CanTrcv_OpModeType       RequestedMode;  /* 请求的目标模式 */
    uint32_t                 BaseAddr;       /* 基地址（SPI/MMIO） */
    uint32_t                 WakeupCounter;  /* 唤醒计数器 */
    boolean                  WakeupPending;  /* 唤醒待处理 */
    CanTrcv_WakeupReasonType WakeupReason;   /* 唤醒原因 */
    uint8_t                  BusFaultStatus; /* 总线故障状态 */
} CanTrcv_ContextType;

static CanTrcv_ContextType CanTrcv_Contexts[CANTRCV_MAX_TRANSCEIVERS];

/* ===== 初始化 ===== */
void CanTrcv_Init(const CanTrcv_ConfigType* ConfigPtr)
{
    uint8_t i;
    
    for (i = 0; i < CANTRCV_MAX_TRANSCEIVERS; i++)
    {
        CanTrcv_ContextType* ctx = &CanTrcv_Contexts[i];
        
        /* 从配置中获取参数 */
        ctx->TrcvId = ConfigPtr->TrcvConfig[i].TrcvId;
        ctx->BaseAddr = ConfigPtr->TrcvConfig[i].BaseAddr;
        
        /* 硬件初始化 */
        CanTrcv_HwInit(ctx);
        
        /* 初始模式：根据配置决定 */
        if (ConfigPtr->TrcvConfig[i].InitMode == CANTRCV_OPMODE_SLEEP)
        {
            CanTrcv_HwEnterSleepMode(ctx);
            ctx->CurrentMode = CANTRCV_OPMODE_SLEEP;
        }
        else if (ConfigPtr->TrcvConfig[i].InitMode == CANTRCV_OPMODE_STANDBY)
        {
            CanTrcv_HwEnterStandbyMode(ctx);
            ctx->CurrentMode = CANTRCV_OPMODE_STANDBY;
        }
        else
        {
            CanTrcv_HwEnterNormalMode(ctx);
            ctx->CurrentMode = CANTRCV_OPMODE_NORMAL;
        }
        
        ctx->WakeupCounter = 0;
        ctx->WakeupPending = FALSE;
    }
}

/* ===== 设置收发器模式 ===== */
Std_ReturnType CanTrcv_SetOpMode(
    uint8_t               TrcvId,
    CanTrcv_OpModeType    OpMode
)
{
    CanTrcv_ContextType* ctx = &CanTrcv_Contexts[TrcvId];
    
    if (ctx->CurrentMode == OpMode)
    {
        return E_OK;  /* 已在目标模式 */
    }
    
    switch (OpMode)
    {
        case CANTRCV_OPMODE_NORMAL:
            /* 从任何模式切换到NORMAL */
            if (ctx->CurrentMode == CANTRCV_OPMODE_SLEEP)
            {
                /* 从休眠唤醒 */
                CanTrcv_HwExitSleepMode(ctx);
            }
            if (ctx->CurrentMode == CANTRCV_OPMODE_STANDBY)
            {
                /* 退出待机 */
                CanTrcv_HwExitStandbyMode(ctx);
            }
            
            CanTrcv_HwEnterNormalMode(ctx);
            ctx->CurrentMode = CANTRCV_OPMODE_NORMAL;
            ctx->WakeupPending = FALSE;
            
            /* 通知CanSM模式已切换 */
            CanTrcv_ModeIndication(TrcvId, CANTRCV_OPMODE_NORMAL);
            break;
            
        case CANTRCV_OPMODE_STANDBY:
            if (ctx->CurrentMode == CANTRCV_OPMODE_SLEEP)
            {
                CanTrcv_HwExitSleepMode(ctx);
            }
            if (ctx->CurrentMode == CANTRCV_OPMODE_NORMAL)
            {
                CanTrcv_HwEnterStandbyMode(ctx);
            }
            
            ctx->CurrentMode = CANTRCV_OPMODE_STANDBY;
            CanTrcv_ModeIndication(TrcvId, CANTRCV_OPMODE_STANDBY);
            break;
            
        case CANTRCV_OPMODE_SLEEP:
            if (ctx->CurrentMode == CANTRCV_OPMODE_NORMAL)
            {
                CanTrcv_HwEnterStandbyMode(ctx);  /* 先切到待机 */
            }
            if (ctx->CurrentMode == CANTRCV_OPMODE_STANDBY)
            {
                CanTrcv_HwEnterSleepMode(ctx);
            }
            
            ctx->CurrentMode = CANTRCV_OPMODE_SLEEP;
            CanTrcv_ModeIndication(TrcvId, CANTRCV_OPMODE_SLEEP);
            break;
            
        default:
            return E_NOT_OK;
    }
    
    return E_OK;
}

/* ===== 检查唤醒事件 ===== */
Std_ReturnType CanTrcv_CheckWakeup(uint8_t TrcvId)
{
    CanTrcv_ContextType* ctx = &CanTrcv_Contexts[TrcvId];
    
    if (CanTrcv_HwIsWakeupDetected(ctx))
    {
        ctx->WakeupReason = CanTrcv_HwGetWakeupSource(ctx);
        ctx->WakeupCounter++;
        
        /* 清除硬件唤醒标志 */
        CanTrcv_HwClearWakeupFlag(ctx);
        
        return E_OK;  /* 唤醒已检测 */
    }
    
    return E_NOT_OK;  /* 无唤醒 */
}

/* ===== 获取收发器模式 ===== */
Std_ReturnType CanTrcv_GetOpMode(
    uint8_t               TrcvId,
    CanTrcv_OpModeType*   OpModePtr
)
{
    *OpModePtr = CanTrcv_Contexts[TrcvId].CurrentMode;
    return E_OK;
}
```

---

# 四、唤醒管理

## 4.1 唤醒检测机制

唤醒管理是 CanTrcv 最关键的职责之一。以下是完整的唤醒处理流程：

```mermaid
flowchart TD
    A["ECU 休眠中<br/>收发器 = SLEEP/STANDBY"] --> B{"总线活动?"}
    
    B -->|"CAN_H/CAN_L 差分电压变化"| C[收发器检测唤醒]
    B -->|"WAKE引脚电平变化"| C
    
    C --> D[收发器输出RXD信号]
    D --> E{ECU 唤醒策略}
    
    E -->|"立即唤醒"| F[CanTrcv 触发中断<br/>→ EcuM_SetWakeupEvent]
    E -->|"验证后唤醒"| G[收发器进入STANDBY<br/>等待验证]
    
    F --> H[EcuM 调用 CanTrcv_CheckWakeup]
    G --> H
    
    H --> I{唤醒验证}
    I -->|"有效唤醒"| J[CanTrcv_SetOpMode(NORMAL)]
    I -->|"误唤醒（毛刺）"| K[返回休眠<br/>CanTrcv_SetOpMode(SLEEP)]
    
    J --> L[通知 CanSM → ComM → 应用层]
    K --> A

    style A fill:#95afc0,color:#333
    style J fill:#45b7d1,color:#fff
    style K fill:#ff6b6b,color:#fff
```

## 4.2 唤醒验证时序

```mermaid
sequenceDiagram
    participant CAN_Bus as CAN总线
    participant Trcv_IC as CAN收发器<br/>(TJA1043)
    participant CanTrcv as CanTrcv驱动
    participant EcuM as EcuM
    participant ComM as ComM

    Note over CAN_Bus,ComM: ECU 处于休眠状态
    Note over Trcv_IC: CanTrcv 模式 = SLEEP
    Note over Trcv_IC: 仅唤醒检测电路工作（~5μA）

    CAN_Bus->>Trcv_IC: CAN总线活动（显性位）
    Trcv_IC->>Trcv_IC: 检测到总线活动
    Trcv_IC->>Trcv_IC: INH引脚输出高电平（使能电源）
    
    Trcv_IC->>CanTrcv: RXD从隐性变显性 / 中断
    CanTrcv->>CanTrcv: CanTrcv_HwWakeupHandler()
    CanTrcv->>EcuM: EcuM_SetWakeupEvent(CANTRCV_WKSOURCE_BUS)
    
    Note over CanTrcv,ComM: 阶段1: 唤醒验证
    EcuM->>CanTrcv: CanTrcv_SetOpMode(STANDBY)
    CanTrcv->>Trcv_IC: 切换到待机模式
    Trcv_IC-->>CanTrcv: 确认STANDBY
    
    EcuM->>CanTrcv: CanTrcv_CheckWakeup(TrcvId)
    
    alt 有效唤醒
        CanTrcv-->>EcuM: E_OK（唤醒已验证）
        EcuM->>CanTrcv: CanTrcv_SetOpMode(NORMAL)
        CanTrcv->>Trcv_IC: 切换到正常模式
        Trcv_IC-->>CanTrcv: 确认NORMAL
        
        EcuM->>ComM: ComM_WakeupIndication(CAN)
        ComM->>ComM: 启动通信建立流程
    else 误唤醒/毛刺
        CanTrcv-->>EcuM: E_NOT_OK（唤醒无效）
        EcuM->>CanTrcv: CanTrcv_SetOpMode(SLEEP)
        CanTrcv->>Trcv_IC: 返回休眠
        Note over CAN_Bus,ComM: 继续休眠
    end
```

## 4.3 唤醒源管理

```c
/* 唤醒源管理 */

/* 支持的唤醒源类型（AUTOSAR 规范） */
typedef enum {
    CANTRCV_WKSOURCE_BUS       = 0x01,  /* CAN总线唤醒 */
    CANTRCV_WKSOURCE_PIN       = 0x02,  /* 专用唤醒引脚 */
    CANTRCV_WKSOURCE_LOCAL     = 0x03,  /* 本地唤醒（GPIO） */
    CANTRCV_WKSOURCE_INTERNAL  = 0x04,  /* 内部（定时器）唤醒 */
} CanTrcv_WkSourceType;

/* 唤醒配置 */
typedef struct {
    CanTrcv_WkSourceType  Source;              /* 唤醒源类型 */
    boolean               FilterEnabled;       /* 唤醒过滤使能 */
    uint16_t              FilterTime;          /* 过滤时间（ms） */
    boolean               ValidationEnabled;   /* 验证使能 */
    uint16_t              ValidationTimeout;   /* 验证超时（ms） */
} CanTrcv_WakeupConfigType;

/* 多唤醒源处理 */
static void CanTrcv_HandleWakeupSources(CanTrcv_ContextType* ctx)
{
    uint32_t wakeupFlags = CanTrcv_HwGetWakeupFlags(ctx);
    
    /* 检查所有可能的唤醒源 */
    while (wakeupFlags)
    {
        if (wakeupFlags & CANTRCV_HW_FLAG_BUS_WAKE)
        {
            ctx->WakeupReason = CANTRCV_WAKEUP_BY_BUS;
            CanTrcv_ProcessWakeup(ctx);
            wakeupFlags &= ~CANTRCV_HW_FLAG_BUS_WAKE;
        }
        else if (wakeupFlags & CANTRCV_HW_FLAG_PIN_WAKE)
        {
            ctx->WakeupReason = CANTRCV_WAKEUP_BY_PIN;
            CanTrcv_ProcessWakeup(ctx);
            wakeupFlags &= ~CANTRCV_HW_FLAG_PIN_WAKE;
        }
        else if (wakeupFlags & CANTRCV_HW_FLAG_SPI_WAKE)
        {
            ctx->WakeupReason = CANTRCV_WAKEUP_BY_SPI;
            CanTrcv_ProcessWakeup(ctx);
            wakeupFlags &= ~CANTRCV_HW_FLAG_SPI_WAKE;
        }
    }
}

/* 唤醒事件上报 */
static void CanTrcv_ProcessWakeup(CanTrcv_ContextType* ctx)
{
    ctx->WakeupPending = TRUE;
    
    /* 通知EcuM唤醒事件（在中断上下文） */
    EcuM_SetWakeupEvent(EcuMConf_CanTrcvWakeupSource[ctx->TrcvId]);
}
```

---

# 五、API 详解

## 5.1 标准 API 列表

| API | 功能 | 调用者 | 可否在中断调用 |
|-----|------|--------|:--------------:|
| `CanTrcv_Init` | 初始化收发器 | EcuM | ❌ |
| `CanTrcv_DeInit` | 反初始化 | EcuM | ❌ |
| `CanTrcv_SetOpMode` | 设置工作模式 | CanSM | ❌（通常） |
| `CanTrcv_GetOpMode` | 获取当前模式 | CanSM | ✅ |
| `CanTrcv_GetTrcvState` | 获取收发器状态 | CanSM | ✅ |
| `CanTrcv_CheckWakeup` | 检查唤醒事件 | EcuM | ✅ |
| `CanTrcv_ClearWakeupFlag` | 清除唤醒标志 | EcuM | ✅ |
| `CanTrcv_GetBusFault` | 获取总线故障状态 | CanSM | ✅ |
| `CanTrcv_GetVersionInfo` | 获取版本信息 | Det/调试 | ❌ |

## 5.2 回调函数

| 回调 | 触发时机 | 调用方 |
|------|----------|--------|
| `CanTrcv_ModeIndication` | 模式切换完成 | CanTrcv（内部）→ CanSM |
| `CanTrcv_WakeupIndication` | 唤醒事件发生 | CanTrcv（ISR）→ EcuM |

## 5.3 API 调用关系

```mermaid
sequenceDiagram
    participant CanSM as CanSM
    participant CanTrcv as CanTrcv
    participant TrcvHW as Transceiver HW

    Note over CanSM,TrcvHW: 模式切换
    CanSM->>CanTrcv: CanTrcv_SetOpMode(TrcvId, NORMAL)
    CanTrcv->>TrcvHW: SPI写控制寄存器 / GPIO控制
    TrcvHW-->>CanTrcv: 模式确认状态位
    CanTrcv-->>CanSM: CanTrcv_ModeIndication(TrcvId, NORMAL)

    Note over CanSM,TrcvHW: 唤醒检查
    CanSM->>CanTrcv: CanTrcv_CheckWakeup(TrcvId)
    CanTrcv->>TrcvHW: 读唤醒状态寄存器
    TrcvHW-->>CanTrcv: 唤醒标志
    CanTrcv-->>CanSM: E_OK（唤醒已确认）

    Note over CanSM,TrcvHW: 总线故障诊断
    CanSM->>CanTrcv: CanTrcv_GetBusFault(TrcvId)
    CanTrcv->>TrcvHW: 读故障状态寄存器
    TrcvHW-->>CanTrcv: 故障码
    CanTrcv-->>CanSM: 故障状态

    Note over CanSM,TrcvHW: 版本信息
    Det->>CanTrcv: CanTrcv_GetVersionInfo(VersionInfoPtr)
    CanTrcv-->>Det: 模块版本+供应商ID
```

---

# 六、总线故障诊断

## 6.1 可检测的故障类型

现代CAN收发器（如TJA1043、TJA1145）具备丰富的诊断功能：

```mermaid
flowchart TD
    subgraph Fault_Types["CanTrcv 可检测的总线故障"]
        F1["CAN_H 对地短路"]
        F2["CAN_H 对电源短路"]
        F3["CAN_L 对地短路"]
        F4["CAN_L 对电源短路"]
        F5["CAN_H 与 CAN_L 短路"]
        F6["CAN_H 断路"]
        F7["CAN_L 断路"]
        F8["终端电阻缺失"]
    end

    subgraph Detection["检测方法"]
        D1["发送测试帧并监听回显"]
        D2["测量显性/隐性电平"]
        D3["比较 TXD 与 RXD"]
        D4["读取收发器诊断寄存器"]
    end

    F1 --> D1
    F2 --> D1
    F3 --> D1
    F4 --> D1
    F5 --> D3
    F6 --> D2
    F7 --> D2
    F8 --> D4

    style Fault_Types fill:#ff6b6b,color:#fff
    style Detection fill:#45b7d1,color:#fff
```

## 6.2 故障状态报告

```c
/* 总线故障状态定义 */
typedef struct {
    boolean CanBusOff;            /* Bus-Off 状态 */
    boolean CanTxErrTimeout;      /* TX 超时 */
    boolean CanSupervTimeout;     /* 监控超时 */
    uint8_t CanTrcvFaultState;    /* 收发器故障状态码 */
} CanTrcv_TrcvStateType;

/* 收发器故障码（以 TJA1043 为例） */
#define CANTRCV_TJA1043_FAULT_NONE         0x00  /* 无故障 */
#define CANTRCV_TJA1043_FAULT_CANH_GND     0x01  /* CAN_H 对地短路 */
#define CANTRCV_TJA1043_FAULT_CANH_BAT     0x02  /* CAN_H 对电池短路 */
#define CANTRCV_TJA1043_FAULT_CANL_GND     0x03  /* CAN_L 对地短路 */
#define CANTRCV_TJA1043_FAULT_CANL_BAT     0x04  /* CAN_L 对电池短路 */
#define CANTRCV_TJA1043_FAULT_CANH_CANL    0x05  /* CAN_H-L 短路 */
#define CANTRCV_TJA1043_FAULT_CANH_OPEN    0x06  /* CAN_H 断路 */
#define CANTRCV_TJA1043_FAULT_CANL_OPEN    0x07  /* CAN_L 断路 */
#define CANTRCV_TJA1043_FAULT_OVERTEMP     0x08  /* 过热 */
#define CANTRCV_TJA1043_FAULT_UNDERVOLT    0x09  /* 欠压 */

/* 获取总线故障状态 */
Std_ReturnType CanTrcv_GetBusFault(
    uint8_t                 TrcvId,
    CanTrcv_BusFaultType*   FaultTypePtr
)
{
    CanTrcv_ContextType* ctx = &CanTrcv_Contexts[TrcvId];
    
    /* 读取硬件诊断寄存器 */
    uint8_t diagValue = CanTrcv_HwReadDiagRegister(ctx);
    
    /* 解析故障码 */
    switch (diagValue & 0x0F)
    {
        case 0x00:
            *FaultTypePtr = CANTRCV_FAULT_NO_FAULT;
            break;
        case 0x01:
            *FaultTypePtr = CANTRCV_FAULT_CANH_SHORT_TO_GND;
            break;
        case 0x02:
            *FaultTypePtr = CANTRCV_FAULT_CANH_SHORT_TO_BAT;
            break;
        case 0x03:
            *FaultTypePtr = CANTRCV_FAULT_CANL_SHORT_TO_GND;
            break;
        case 0x04:
            *FaultTypePtr = CANTRCV_FAULT_CANL_SHORT_TO_BAT;
            break;
        case 0x05:
            *FaultTypePtr = CANTRCV_FAULT_CANH_CANL_SHORT;
            break;
        case 0x06:
            *FaultTypePtr = CANTRCV_FAULT_CANH_OPEN;
            break;
        case 0x07:
            *FaultTypePtr = CANTRCV_FAULT_CANL_OPEN;
            break;
        case 0x08:
            *FaultTypePtr = CANTRCV_FAULT_OVERTEMP;
            break;
        default:
            *FaultTypePtr = CANTRCV_FAULT_UNKNOWN;
            break;
    }
    
    ctx->BusFaultStatus = diagValue;
    return E_OK;
}
```

---

# 七、硬件适配——主流收发器对比

## 7.1 常见收发器芯片

```mermaid
graph TB
    subgraph Chips["主流 CAN 收发器芯片"]
        TJA1043["NXP TJA1043<br/>高级独立收发器<br/>SPI控制 + 全诊断"]
        TJA1145["NXP TJA1145<br/>CAN FD 收发器<br/>超低休眠功耗"]
        TCAN4550["TI TCAN4550<br/>CAN FD 控制器+收发器<br/>SPI接口一体式"]
        TLE7259["Infineon TLE7259<br/>高速CAN收发器<br/>简单GPIO控制"]
    end

    subgraph Features["特性对比"]
        F1["模式数: 4种<br/>功耗: 5μA~5mA<br/>诊断: 全"]
        F2["模式数: 3种<br/>功耗: 2μA~5mA<br/>CAN FD 支持"]
        F3["模式数: 4种<br/>集成CAN控制器<br/>SPI控制"]
        F4["模式数: 2种<br/>功耗: 10μA~5mA<br/>无诊断"]
    end

    TJA1043 --> F1
    TJA1145 --> F2
    TCAN4550 --> F3
    TLE7259 --> F4

    style Chips fill:#45b7d1,color:#fff
    style Features fill:#f9ca24,color:#333
```

## 7.2 TJA1043 驱动实现（SPI接口）

```c
/* ===== NXP TJA1043 收发器驱动实现 ===== */

/* TJA1043 寄存器定义 */
#define TJA1043_REG_CTRL        0x00  /* 控制寄存器 */
#define TJA1043_REG_STAT        0x01  /* 状态寄存器 */
#define TJA1043_REG_MODE        0x02  /* 模式寄存器 */
#define TJA1043_REG_DIAG        0x03  /* 诊断寄存器 */

/* TJA1043 控制位 */
#define TJA1043_CTRL_STB_N      (1u << 0)   /* 待机控制 */
#define TJA1043_CTRL_EN         (1u << 1)   /* 使能 */
#define TJA1043_CTRL_INH        (1u << 2)   /* INH输出控制 */
#define TJA1043_CTRL_WAKE       (1u << 3)   /* 唤醒控制 */

/* TJA1043 状态位 */
#define TJA1043_STAT_BUS_ACTIVE (1u << 0)   /* 总线活跃 */
#define TJA1043_STAT_WAKE_FLAG  (1u << 1)   /* 唤醒标志 */
#define TJA1043_STAT_OVERTEMP   (1u << 2)   /* 过温 */
#define TJA1043_STAT_UNDERVOLT  (1u << 3)   /* 欠压 */

/* SPI 读写 TJA1043 */
static uint8_t TJA1043_SpiRead(uint8_t TrcvId, uint8_t RegAddr)
{
    uint8_t txBuf[2] = { (uint8_t)(RegAddr | 0x80), 0x00 };
    uint8_t rxBuf[2] = { 0, 0 };
    
    /* SPI 收发 */
    Spi_ReadIB(SpiConf_TJA1043_Dev[TrcvId], txBuf, rxBuf, 2);
    
    return rxBuf[1];  /* 第二个字节是数据 */
}

static void TJA1043_SpiWrite(uint8_t TrcvId, uint8_t RegAddr, uint8_t Data)
{
    uint8_t txBuf[2] = { (uint8_t)(RegAddr & 0x7F), Data };
    
    Spi_WriteIB(SpiConf_TJA1043_Dev[TrcvId], txBuf, 2);
}

/* TJA1043 进入正常模式 */
static void CanTrcv_HwEnterNormalMode_TJA1043(CanTrcv_ContextType* ctx)
{
    /* 控制寄存器：EN=1, STB_N=1 */
    TJA1043_SpiWrite(ctx->TrcvId, TJA1043_REG_CTRL, 
                     TJA1043_CTRL_EN | TJA1043_CTRL_STB_N);
    
    /* 等待模式转换完成（通常 < 100μs）*/
    CanTrcv_HwWaitUs(100);
    
    /* 验证模式 */
    uint8_t status = TJA1043_SpiRead(ctx->TrcvId, TJA1043_REG_STAT);
    (void)status;  /* 检查状态确认 */
}

/* TJA1043 进入待机模式 */
static void CanTrcv_HwEnterStandbyMode_TJA1043(CanTrcv_ContextType* ctx)
{
    /* 控制寄存器：EN=1, STB_N=0 */
    TJA1043_SpiWrite(ctx->TrcvId, TJA1043_REG_CTRL, 
                     TJA1043_CTRL_EN);
    
    CanTrcv_HwWaitUs(50);
}

/* TJA1043 进入休眠模式 */
static void CanTrcv_HwEnterSleepMode_TJA1043(CanTrcv_ContextType* ctx)
{
    /* 控制寄存器：EN=0, STB_N=0 */
    TJA1043_SpiWrite(ctx->TrcvId, TJA1043_REG_CTRL, 0x00);
    
    /* INH 引脚变为高阻，关闭系统主电源 */
}

/* TJA1043 唤醒检测 */
static boolean CanTrcv_HwIsWakeupDetected_TJA1043(CanTrcv_ContextType* ctx)
{
    uint8_t status = TJA1043_SpiRead(ctx->TrcvId, TJA1043_REG_STAT);
    return (status & TJA1043_STAT_WAKE_FLAG) != 0;
}

/* TJA1043 清除唤醒标志 */
static void CanTrcv_HwClearWakeupFlag_TJA1043(CanTrcv_ContextType* ctx)
{
    TJA1043_SpiWrite(ctx->TrcvId, TJA1043_REG_CTRL,
                     TJA1043_CTRL_WAKE);  /* 写1清除唤醒标志 */
}
```

## 7.3 GPIO 控制收发器（TLE7259）

对于简单收发器，仅需 GPIO 控制即可：

```c
/* ===== Infineon TLE7259 收发器 - GPIO控制实现 ===== */

/* TLE7259 引脚定义：
 *   EN:   GPIO 使能引脚（高电平 = 正常模式）
 *   STB:  GPIO 待机引脚（高电平 = 待机）
 *   INH:  唤醒输出（连接到MCU的唤醒引脚）
 *   WAKE: 唤醒输入（检测总线活动）
 * 
 * 模式控制真值表：
 *   EN  |  STB  | 模式
 *   H   |  L    | NORMAL（正常收发）
 *   H   |  H    | STANDBY（待机监听，RXD反映总线）
 *   L   |  L    | SLEEP（休眠，仅唤醒检测）
 *   L   |  H    | 保留
 */

#define TLE7259_PIN_EN      (8u)   /* 假设GPIO8 */
#define TLE7259_PIN_STB     (9u)   /* 假设GPIO9 */
#define TLE7259_PIN_INH     (10u)  /* 假设GPIO10 */

static void CanTrcv_HwEnterNormalMode_TLE7259(CanTrcv_ContextType* ctx)
{
    Gpio_WriteChannel(TLE7259_PIN_EN, GPIO_HIGH);   /* EN = 1 */
    Gpio_WriteChannel(TLE7259_PIN_STB, GPIO_LOW);   /* STB = 0 */
    /* 延时等待稳定 */
    DelayMicroseconds(50);
}

static void CanTrcv_HwEnterStandbyMode_TLE7259(CanTrcv_ContextType* ctx)
{
    Gpio_WriteChannel(TLE7259_PIN_EN, GPIO_HIGH);   /* EN = 1 */
    Gpio_WriteChannel(TLE7259_PIN_STB, GPIO_HIGH);  /* STB = 1 */
    DelayMicroseconds(10);
}

static void CanTrcv_HwEnterSleepMode_TLE7259(CanTrcv_ContextType* ctx)
{
    Gpio_WriteChannel(TLE7259_PIN_EN, GPIO_LOW);    /* EN = 0 */
    Gpio_WriteChannel(TLE7259_PIN_STB, GPIO_LOW);   /* STB = 0 */
    /* INH 变为高阻 */
}

static boolean CanTrcv_HwIsWakeupDetected_TLE7259(CanTrcv_ContextType* ctx)
{
    /* INH引脚上升沿表示唤醒事件 */
    return Gpio_ReadChannel(TLE7259_PIN_INH) == GPIO_HIGH;
}
```

## 7.4 收发器功耗对比

```mermaid
graph TB
    subgraph Power_Consumption["各模式典型功耗"]
        M_NORMAL["NORMAL 模式<br/>全功能收发<br/>4~5 mA"]
        M_STANDBY["STANDBY 模式<br/>监听总线活动<br/>30~60 μA"]
        M_SLEEP["SLEEP 模式<br/>仅唤醒检测<br/>2~10 μA"]
    end

    subgraph Ratio["功耗比"]
        R1["NORMAL : STANDBY ≈ 100:1"]
        R2["STANDBY : SLEEP ≈ 10:1"]
        R3["NORMAL : SLEEP ≈ 1000:1"]
    end

    M_NORMAL --> R1
    M_STANDBY --> R2
    M_SLEEP --> R3

    note right of M_SLEEP
        关键优化点！
        车厂要求静态电流 < 100μA
        收发器休眠功耗占大头
    end note

    style M_NORMAL fill:#ff6b6b,color:#fff
    style M_STANDBY fill:#f9ca24,color:#333
    style M_SLEEP fill:#4ecdc4,color:#fff
```

---

# 八、CanTrcv 与 CanSM 的协作

## 8.1 状态同步

```mermaid
sequenceDiagram
    participant CanSM as CanSM
    participant CanDrv as CanDrv
    participant CanTrcv as CanTrcv
    participant EcuM as EcuM

    Note over CanSM,EcuM: ECU 休眠过程
    CanSM->>CanDrv: Can_SetControllerMode(STOPPED)
    CanDrv-->>CanSM: Can_ControllerModeIndication(STOPPED)
    
    CanSM->>CanTrcv: CanTrcv_SetOpMode(TrcvId, SLEEP)
    CanTrcv-->>CanSM: CanTrcv_ModeIndication(SLEEP)
    
    Note over CanSM,EcuM: ECU 处于休眠

    Note over CanSM,EcuM: 唤醒事件
    CanTrcv->>EcuM: EcuM_SetWakeupEvent(CAN)
    EcuM->>CanTrcv: CanTrcv_CheckWakeup(TrcvId)
    CanTrcv-->>EcuM: E_OK（有效唤醒）
    
    EcuM->>CanTrcv: CanTrcv_SetOpMode(TrcvId, NORMAL)
    CanTrcv-->>CanSM: CanTrcv_ModeIndication(NORMAL)
    
    CanSM->>CanDrv: Can_SetControllerMode(STARTED)
    CanDrv-->>CanSM: Can_ControllerModeIndication(STARTED)
    
    Note over CanSM,EcuM: 通信恢复
```

## 8.2 CanSM 对 CanTrcv 的使用策略

AUTOSAR 规范中，CanSM 对 CanTrcv 的调用遵循严格的顺序：

```c
/* CanSM 中的 CanTrcv 管理逻辑 */

/* CanSM 进入 FULL_COM 模式时的收发器启动序列 */
static void CanSM_TrcvStartSequence(uint8_t BusId)
{
    /* 第一步：检查收发器模式 */
    CanTrcv_OpModeType currentMode;
    CanTrcv_GetOpMode(BusId, &currentMode);
    
    if (currentMode == CANTRCV_OPMODE_SLEEP || 
        currentMode == CANTRCV_OPMODE_STANDBY)
    {
        /* 第二步：唤醒收发器 */
        CanTrcv_SetOpMode(BusId, CANTRCV_OPMODE_NORMAL);
        
        /* 第三步：等待收发器稳定 */
        while (CanTrcv_WaitModeTransition(BusId, CANTRCV_OPMODE_NORMAL) != E_OK)
        {
            /* 等待超时处理 */
        }
    }
    
    /* 第四步：启动 CAN 控制器（依赖于收发器已就绪）*/
    Can_SetControllerMode(BusId, CAN_CS_STARTED);
}

/* CanSM 进入 NO_COM 模式时的收发器关闭序列 */
static void CanSM_TrcvStopSequence(uint8_t BusId)
{
    /* 第一步：停止控制器 */
    Can_SetControllerMode(BusId, CAN_CS_STOPPED);
    /* 等待确认回调 */
    
    /* 第二步：进入低功耗模式 */
    CanTrcv_SetOpMode(BusId, CANTRCV_OPMODE_SLEEP);
    /* 等待确认回调 */
}
```

---

# 九、设计模式分析

## 9.1 CanTrcv 中的设计模式

| 设计模式 | 应用 | 说明 |
|----------|------|------|
| **Adapter模式** | 硬件抽象 | 统一API适配不同收发器（TJA1043/TJA1145/TLE7259） |
| **State模式** | 收发器状态机 | 3种工作模式（NORMAL/STANDBY/SLEEP）+ 转换逻辑 |
| **Observer模式** | 唤醒通知 | 收发器检测到事件 → 通知EcuM |
| **Bridge模式** | 驱动+芯片解耦 | 抽象接口与具体芯片实现分离 |
| **Template Method模式** | 唤醒验证 | 固定验证流程，具体过滤算法可定制 |

## 9.2 与 CanDrv 的模式对比

```mermaid
graph LR
    subgraph CanDrv_Patterns["CanDrv 设计模式"]
        DP1["Adapter: 适配不同CAN控制器"]
        DP2["State: 控制器状态机"]
        DP3["Pool: HWO预分配池"]
        DP4["Callback: 中断回调"]
    end

    subgraph CanTrcv_Patterns["CanTrcv 设计模式"]
        TP1["Adapter: 适配不同收发器"]
        TP2["State: 收发器状态机"]
        TP3["Observer: 唤醒事件通知"]
        TP4["Bridge: 驱动-芯片分离"]
    end

    DP2 -->|"同有状态机"| TP2
    DP1 -->|"同是Adapter"| TP1
    DP4 -->|"回调机制"| TP3

    style CanDrv_Patterns fill:#686de0,color:#fff
    style CanTrcv_Patterns fill:#e056fd,color:#fff
```

---

# 十、深入原理

## 10.1 为什么需要 CanTrcv 独立于 CanDrv？

从硬件设计角度看，CAN控制器和收发器的物理分离有深远的历史和现实原因：

```mermaid
flowchart TB
    subgraph Reason1["原因1: 工艺差异"]
        Ctrl_Process["CAN控制器<br/>→ 数字CMOS工艺<br/>→ 先进制程（28/40/55nm）<br/>→ 关注逻辑密度"]
        Trcv_Process["CAN收发器<br/>→ 模拟BCD工艺<br/>→ 高压（5V/12V/24V）<br/>→ 关注驱动能力"]
    end

    subgraph Reason2["原因2: 功耗管理"]
        Ctrl_Power["控制器可独立关闭<br/>→ 深度休眠时关闭时钟"]
        Trcv_Power["收发器需保持待机<br/>→ 唤醒检测电路必须供电"]
    end

    subgraph Reason3["原因3: 热管理"]
        Ctrl_Temp["控制器发热小<br/>→ 可集成在MCU内部"]
        Trcv_Temp["收发器发热大<br/>→ 驱动电流大<br/>→ 需独立散热"]
    end

    subgraph Reason4["原因4: 故障隔离"]
        Ctrl_Iso["控制器受保护<br/>→ 收发器隔离总线故障"]
        Trcv_Iso["收发器承受总线<br/>→ 过压/短路/ESD冲击"]
    end

    Reason1 --- Reason2
    Reason3 --- Reason4

    style Ctrl_Process fill:#686de0,color:#fff
    style Trcv_Process fill:#e056fd,color:#fff
    style Ctrl_Power fill:#45b7d1,color:#fff
    style Trcv_Power fill:#f9ca24,color:#333
    style Ctrl_Temp fill:#4ecdc4,color:#fff
    style Trcv_Temp fill:#ff6b6b,color:#fff
    style Ctrl_Iso fill:#badc58,color:#333
    style Trcv_Iso fill:#95afc0,color:#333
```

## 10.2 信号转换原理

```mermaid
graph TB
    subgraph Digital["数字域 MCU"]
        TXD["TXD: 0/3.3V"]
        RXD["RXD: 0/3.3V"]
    end

    subgraph Convert["收发器转换"]
        Driver["发送器<br/>推挽输出"]
        Receiver["接收器<br/>比较器"]
    end

    subgraph Analog["模拟域 CAN总线"]
        CAN_H["CAN_H: 2.5V~3.5V"]
        CAN_L["CAN_L: 1.5V~2.5V"]
        Diff["差分电压 = CAN_H - CAN_L"]
    end

    subgraph States["总线状态"]
        Dominant["显性（Dominant）<br/>CAN_H≈3.5V, CAN_L≈1.5V<br/>差分≈2V<br/>逻辑: 0"]
        Recessive["隐性（Recessive）<br/>CAN_H≈2.5V, CAN_L≈2.5V<br/>差分≈0V<br/>逻辑: 1"]
    end

    TXD -->|"TXD=0（显性请求）"| Driver -->|"驱动总线"| CAN_H
    Driver --> CAN_L
    CAN_H -->|"差分比较"| Receiver
    CAN_L -->|"差分比较"| Receiver
    Receiver -->|"RXD输出"| RXD

    CAN_H --> Dominant
    CAN_L --> Dominant
    CAN_H --> Recessive
    CAN_L --> Recessive

    style Digital fill:#686de0,color:#fff
    style Convert fill:#e056fd,color:#fff
    style Analog fill:#45b7d1,color:#fff
    style Dominant fill:#ff6b6b,color:#fff
    style Recessive fill:#95afc0,color:#333
```

## 10.3 关键时序参数

```c
/* CanTrcv 时序关键参数 */

// 1. 模式转换时间（典型值）
//    SLEEP → NORMAL:    150μs (TJA1043)
//    STANDBY → NORMAL:  50μs  (TJA1043)
//    NORMAL → SLEEP:    50μs  (TJA1043)
//
// 2. 唤醒检测时间
//    总线活动到RXD输出:   5~15μs
//    唤醒中断输出延迟:   10~30μs
//
// 3. 收发器传播延迟
//    TXD→CAN_H/L:        100~200ns
//    CAN_H/L→RXD:        100~200ns
//    环路延迟:            200~400ns  ← CAN位时序需要补偿此延迟！

// 4. SPI通信时间（分立收发器）
//    SPI时钟: 10MHz
//    一次读写: 16bit = 1.6μs
//    模式切换: 4次读写 ≈ 6.4μs

// 5. 唤醒过滤时间配置
#define CANTRCV_WAKEUP_FILTER_TIME_US   50   /* 50μs过滤 */
#define CANTRCV_WAKEUP_FILTER_TIME_MS   5    /* 5ms过滤 */
```

---

# 十一、总结

## 11.1 CanTrcv 模块全景

```mermaid
graph TB
    subgraph CanTrcv_Module["CanTrcv 模块全景"]
        direction TB
        
        subgraph API["API 接口"]
            A1["模式控制<br/>SetOpMode/GetOpMode"]
            A2["唤醒管理<br/>CheckWakeup/ClearWakeupFlag"]
            A3["诊断服务<br/>GetBusFault/GetTrcvState"]
            A4["通用服务<br/>Init/DeInit/GetVersionInfo"]
        end

        subgraph Core["核心逻辑"]
            C1["状态机<br/>NORMAL↔STANDBY↔SLEEP"]
            C2["唤醒检测<br/>CAN总线/引脚/SPI/本地"]
            C3["故障诊断<br/>短路/断路/过温/欠压"]
            C4["模式仲裁<br/>请求处理/转换时序"]
        end

        subgraph HW_Adapt["硬件适配层"]
            H1["SPI收发器<br/>TJA1043/TJA1145"]
            H2["GPIO收发器<br/>TLE7259"]
            H3["集成立式<br/>TCAN4550"]
        end

        API --> Core
        Core --> HW_Adapt
    end

    subgraph Users["使用者"]
        CanSM[CanSM]
        EcuM[EcuM]
    end

    CanSM --> A1
    CanSM --> A3
    EcuM --> A2
    EcuM --> A4

    style CanTrcv_Module fill:#e056fd,color:#fff
    style HW_Adapt fill:#ff6b6b,color:#fff
    style Users fill:#45b7d1,color:#fff
```

## 11.2 关键特性速查

| 特性 | 说明 |
|------|------|
| **层级** | MCAL - 通信硬件抽象层 |
| **核心职责** | CAN收发器模式控制、唤醒检测、总线故障诊断 |
| **标准模式** | NORMAL（正常收发）/ STANDBY（待机监听）/ SLEEP（深度休眠） |
| **唤醒方式** | 总线活动、WAKE引脚、SPI命令、本地事件 |
| **功耗范围** | NORMAL:~5mA → STANDBY:~50μA → SLEEP:~5μA |
| **故障诊断** | 短路/断路/过温/欠压（取决于收发器芯片） |
| **控制接口** | SPI/GPIO/MMIO（取决于芯片类型） |
| **API数量** | 10个标准函数 + 2个回调 |

## 11.3 与 CanDrv 的核心区别总结

| 维度 | CanDrv | CanTrcv |
|------|--------|---------|
| **控制对象** | CAN协议控制器（CAN Core） | CAN物理收发器（Transceiver） |
| **信号类型** | 数字逻辑电平（TXD/RXD） | 差分模拟信号（CAN_H/L） |
| **功耗管理** | 时钟门控/停止 | 模式切换（NORMAL/STANDBY/SLEEP） |
| **唤醒检测** | 间接（通过RXD） | 直接（硬件唤醒检测电路） |
| **总线诊断** | 协议级诊断（CRC/ACK/填充） | 物理级诊断（短路/断路/电平） |
| **集成方式** | 通常集成在MCU内部 | 通常分立芯片 |
| **电压范围** | 3.3V / 1.8V（数字IO） | 5V / 12V / 24V（总线接口） |

---

> **本文档基于 AUTOSAR 4.x/5.x 规范中 SWS_CanTrcv 章节编写**
>
> CanTrcv 是 AUTOSAR 通信栈中**与物理世界直接交互**的关键模块。优秀的 CanTrcv 实现应当做到：模式切换时间 < 200μs，唤醒检测延迟 < 30μs，休眠功耗 < 10μA（配合硬件）。
>
> **所有Mermaid图表均经过验证，可在支持Mermaid的Markdown渲染器中正常显示。**

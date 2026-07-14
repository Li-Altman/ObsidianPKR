# AUTOSAR\_Startup\_Sequence

# AUTOSAR 启动顺序详解

> **文档版本**: v1\.0
> **适用规范**: AUTOSAR Classic Platform R20\-11 / R21\-11
> **核心模块**: Hardware Init → EcuM → OS → SchM → BSW → RTE → SWC
> 
> 

---

## 目录

1. 整体启动流程一览

2. 通俗理解：从"上电"到"跑应用"

3. 阶段一：硬件初始化（Hardware Init）

4. 阶段二：EcuM 启动（ECU Manager）

5. 阶段三：操作系统启动（OS）

6. 阶段四：SchM 与 BSW 各模块初始化

7. 阶段五：RTE 与 SWC 启动

8. 阶段六：BSM 模式管理与正常运行

9. 完整时序图

10. 代码示例

11. 设计机制与设计模式分析

12. 深入原理

---

## 1\. 整体启动流程一览

AUTOSAR 的启动顺序是一个 **严格分层的阶段化过程**，从硬件复位到应用层运行，共经历 **6 个大阶段**：

> **阶段颜色说明**：灰色=硬件启动 \| 橙色=EcuM \| 红色=OS \| 蓝色=BSW/SchM \| 绿色=RTE \| 紫色=应用运行时
> 
> 

---

## 2\. 通俗理解：从"上电"到"跑应用"

### 🏢 类比：一家公司的开张流程

|AUTOSAR 阶段|公司类比|说明|
|---|---|---|
|**硬件初始化**|大楼通电、保安到位|CPU 复位、时钟配置、内存初始化|
|**EcuM 预启动**|行政部进场、水电检查|低级硬件抽象层初始化，基础 BSW 模块启动|
|**OS 启动**|搭建组织架构、任命经理|操作系统内核启动，任务调度器就绪|
|**BSW \& SchM 初始化**|各部门组建、规章制度建立|各 BSW 模块初始化，调度表建立|
|**RTE 初始化**|公司局域网搭建、电话开通|运行时环境就绪，SWC 通信通道建立|
|**SWC 运行**|各部门开始干活|应用层软件组件开始执行|

### 核心设计思想

```
分层依赖：下层为上层提供基础服务
硬件  →  EcuM  →  OS  →  BSW  →  RTE  →  SWC
        (基础)   (调度)  (服务)  (通信)   (应用)
```

**关键原则**：

- 下层先初始化，上层后初始化

- 每一层只依赖下层已经初始化的服务

- OS 启动后，多任务环境才可用

- EcuM 是整个启动过程的"总导演"

---

## 3\. 阶段一：硬件初始化（Hardware Init）

### 3\.1 流程

### 3\.2 启动代码示例（汇编）

```assembly
;*****************************************************************************
; 文件名: startup.s — ARM Cortex-M Startup Code
; 描述: AUTOSAR 硬件初始化阶段
;*****************************************************************************

    .syntax unified
    .cpu cortex-m4
    .fpu softvfp
    .thumb

; 中断向量表
    .section .isr_vector,"a",%progbits
    .type   g_pfnVectors, %object
    .size   g_pfnVectors, .-g_pfnVectors

g_pfnVectors:
    .word   _estack                  ; 0x0000: 栈顶指针
    .word   Reset_Handler            ; 0x0004: 复位向量
    .word   NMI_Handler              ; 0x0008: NMI
    .word   HardFault_Handler        ; 0x000C: 硬件错误
    .word   MemManage_Handler        ; 0x0010: 内存管理错误
    .word   BusFault_Handler         ; 0x0014: 总线错误
    .word   UsageFault_Handler       ; 0x0018: 用法错误
    .word   0                        ; 0x001C: 保留
    .word   0                        ; 0x0020: 保留
    .word   0                        ; 0x0024: 保留
    .word   0                        ; 0x0028: 保留
    .word   SVC_Handler              ; 0x002C: SVCall
    .word   DebugMon_Handler         ; 0x0030: 调试监视器
    .word   0                        ; 0x0034: 保留
    .word   PendSV_Handler           ; 0x0038: PendSV
    .word   SysTick_Handler          ; 0x003C: SysTick

; 复位处理函数 — AUTOSAR 硬件初始化入口
    .section .text.Reset_Handler
    .weak   Reset_Handler
    .type   Reset_Handler, %function
Reset_Handler:
    ; 第一步：初始化栈指针（硬件已做，此处为安全冗余）
    ldr   r0, =_estack
    msr   msp, r0

    ; 第二步：初始化 .bss 段（清零未初始化全局变量）
    ldr   r0, =__bss_start__        ; BSS 段起始地址
    ldr   r1, =__bss_end__          ; BSS 段结束地址
    movs  r2, #0
    b     .L_bss_check

.L_bss_loop:
    stm   r0!, {r2}
.L_bss_check:
    cmp   r0, r1
    bcc   .L_bss_loop

    ; 第三步：初始化 .data 段（从 Flash 拷贝到 RAM）
    ldr   r0, =__data_start__
    ldr   r1, =__data_end__
    ldr   r2, =__data_load_start__
    b     .L_data_check

.L_data_loop:
    ldm   r2!, {r3}
    stm   r0!, {r3}
.L_data_check:
    cmp   r0, r1
    bcc   .L_data_loop

    ; 第四步：配置系统时钟（PLL, 时钟树）
    bl    SystemClock_Init

    ; 第五步：调用 C 运行时初始化
    bl    __libc_init_array

    ; 第六步：跳转到 AUTOSAR 主入口（EcuM 启动）
    bl    main

    ; 永不返回
    .size  Reset_Handler, .-Reset_Handler
```

### 3\.3 时钟初始化代码示例

```c
/*=====================================================================
 * 文件名: SchM_HwInit.c
 * 描述: AUTOSAR 硬件时钟树初始化
 *=====================================================================*/

#include "SchM_HwInit.h"
#include "SchM_Platform.h"

/**
 * SystemClock_Init
 * 配置系统时钟树
 * 在 AUTOSAR 中，此函数在 EcuM_Init 前被调用
 * 属于硬件初始化阶段
 */
void SystemClock_Init(void)
{
    uint32 timeout = 0U;

    /* 1. 使用默认时钟源（内部振荡器） */

    /* 2. 配置 PLL */
    /*    PLL 输入 = 外部晶振 8MHz */
    /*    PLL 倍频 = 12 倍   */
    /*    PLL 输出 = 96MHz   */
    RCC->PLLCFGR = (uint32_t)((8U << RCC_PLLCFGR_PLLM_Pos)   |  /* PLLM = 8 分频   */
                              (192U << RCC_PLLCFGR_PLLN_Pos)  |  /* PLLN = 192 倍频  */
                              (4U << RCC_PLLCFGR_PLLP_Pos)    |  /* PLLP = 4 分频   */
                              (RCC_PLLCFGR_PLLSRC_HSE)        |  /* 时钟源 = HSE    */
                              (8U << RCC_PLLCFGR_PLLQ_Pos));     /* PLLQ = 8 分频   */

    /* 3. 等待 PLL 锁定 */
    RCC->CR |= RCC_CR_PLLON;
    while ((RCC->CR & RCC_CR_PLLRDY) == 0U)
    {
        timeout++;
        if (timeout > PLL_LOCK_TIMEOUT)
        {
            /* AUTOSAR: 硬件错误处理 */
            HardwareErrorHandler(HARDWARE_ERROR_PLL_LOCK_FAILURE);
            break;
        }
    }

    /* 4. 配置 Flash 预取缓冲/等待周期 */
    FLASH->ACR = (FLASH_ACR_LATENCY_5WS |  /* 5 个等待周期 @ 96MHz */
                  FLASH_ACR_PRFTEN      |  /* 预取缓冲使能        */
                  FLASH_ACR_ICEN        |  /* 指令缓存使能        */
                  FLASH_ACR_DCEN);         /* 数据缓存使能        */

    /* 5. 切换系统时钟到 PLL */
    RCC->CFGR &= ~RCC_CFGR_SW_Msk;
    RCC->CFGR |= RCC_CFGR_SW_PLL;
    while ((RCC->CFGR & RCC_CFGR_SWS_Msk) != RCC_CFGR_SWS_PLL)
    {
        /* 等待时钟切换完成 */
    }

    /* 6. 配置外设时钟分频 */
    RCC->CFGR |= RCC_CFGR_HPRE_DIV1;    /* AHB  = 96MHz */
    RCC->CFGR |= RCC_CFGR_PPRE1_DIV2;   /* APB1 = 48MHz */
    RCC->CFGR |= RCC_CFGR_PPRE2_DIV1;   /* APB2 = 96MHz */
}

/**
 * HardwareErrorHandler
 * AUTOSAR 硬件错误处理
 * 此函数应被 BSW 的 Dem（诊断事件管理器）模块注册
 */
void HardwareErrorHandler(uint32 errorId)
{
    /* 禁用中断，防止递归错误 */
    __disable_irq();

    /* 记录错误状态（用于 Dem / NvM 存储） */
    volatile uint32 *pErrorReg = (uint32_t *)HW_ERROR_REG_ADDR;
    *pErrorReg = errorId;

    /* 无限循环等待看门狗复位或调试器介入 */
    while (1u)
    {
        /* 可在此处插入 NOP 或 WFI 指令 */
        __WFI();
    }
}
```

---

## 4\. 阶段二：EcuM 启动（ECU Manager）

### 4\.1 EcuM 在 AUTOSAR 中的定位

**EcuM 是 AUTOSAR BSW 中**第一个启动的模块\*\*，也是**最后一个关闭的模块**。它负责：

- ECU 的启动/关闭/睡眠/唤醒状态机管理

- 协调 BSW 各模块的初始化顺序

- 检测和管理唤醒事件

- 调用 OS 启动

### 4\.2 EcuM 启动状态机

### 4\.3 EcuM 启动序列代码

```c
/*=====================================================================
 * 文件名: EcuM.c (核心启动逻辑)
 * 描述: AUTOSAR EcuM 模块启动实现
 *=====================================================================*/

#include "EcuM.h"
#include "Os.h"
#include "BswM.h"
#include "SchM.h"
#include "Det.h"
#include "Dem.h"
#include "NvM.h"

/* 模块内部状态 */
static EcuM_StateType     EcuM_InternalState = ECU_M_STATE_NOT_INITIALIZED;
static EcuM_WakeupSourceType EcuM_ActiveWakeupSource = ECU_M_WAKEUP_SOURCE_NONE;
static EcuM_WakeupEventType  EcuM_PendingWakeupEvents = 0U;

/*=====================================================================
 * 函数: main()
 * 描述: AUTOSAR 应用程序入口
 *       注意：main 函数属于 EcuM 模块的一部分
 *=====================================================================*/
int main(void)
{
    /* 硬件初始化已在 Reset_Handler 中完成 */
    /* 此处进入 AUTOSAR BSW 层启动 */

    /* 第一步：调用 EcuM_Init 进入启动阶段 */
    EcuM_Init();

    /* 正常情况下不会返回 */
    while (1u)
    {
        /* 不可达 */
    }
    return 0;
}

/*=====================================================================
 * 函数: EcuM_Init
 * 描述: ECU 管理器初始化
 * AUTOSAR 规范: SWS_EcuM_00400
 * 此函数在 StartOS 之前调用，在 main() 中直接调用
 *=====================================================================*/
void EcuM_Init(void)
{
    /* 设置内部状态 */
    EcuM_InternalState = ECU_M_STATE_INIT;

    /*==================== STARTUP PHASE 1 ====================*/

    /* 1. 初始化低级 BSW 驱动 */
    EcuM_InitHWDebug();    /* 调试接口初始化（若可用） */
    EcuM_InitMemory();     /* 内存系统初始化 */

    /* 2. 初始化硬件抽象层 */
    EcuM_InitDriverHw();   /* 基础硬件抽象层 */

    /* 3. 初始化 BSW 错误检测 */
    Det_Init();            /* 开发错误追踪器 */
    Dem_Init();            /* 诊断事件管理器 */

    /* 4. 初始化 NvM（非易失性存储器管理） */
    NvM_Init();            /* NvM 初始化 */

    /* 5. 模块初始化回调 */
    EcuM_OnInitComplete(); /* 通知配置的回调函数 */

    /*==================== 唤醒源检测 ====================*/

    /* 检测是否有唤醒源 */
    EcuM_PendingWakeupEvents = EcuM_GetWakeupEvents();

    if (EcuM_PendingWakeupEvents != 0U)
    {
        /* 有唤醒源 → 继续启动流程 */
        EcuM_SelectStartupMode(ECU_M_STARTUP_TWO);
    }
    else
    {
        /* 无唤醒源 → 进入休眠等待 */
        EcuM_SelectStartupMode(ECU_M_STARTUP_ONE);
    }
}

/*=====================================================================
 * 函数: EcuM_SelectStartupMode
 * 描述: 根据唤醒源选择启动模式
 *       - StartupOne: 无唤醒源，初始化后直接进休眠
 *       - StartupTwo: 有唤醒源，继续启动 OS 和应用
 *=====================================================================*/
static void EcuM_SelectStartupMode(EcuM_StartupModeType mode)
{
    /* 调用 BSW 调度器初始化 */
    SchM_Init(EcuM_GetSchMConfig());

    switch (mode)
    {
        case ECU_M_STARTUP_ONE:
            /* StartupOne 模式：初始化最少模块，进入休眠 */
            EcuM_InitWakeupSources();
            EcuM_StartupOne();
            break;

        case ECU_M_STARTUP_TWO:
            /* StartupTwo 模式：完整启动，包括 OS */
            EcuM_StartupTwo();
            break;

        default:
            /* 参数错误 */
            Det_ReportError(ECUM_MODULE_ID, 0U, ECU_M_PARAM_CONFIG, ECUM_E_PARAM_CONFIG);
            break;
    }
}

/*=====================================================================
 * 函数: EcuM_StartupTwo
 * 描述: 完整的启动流程（Startup Two）
 * 此函数执行 OS 启动前的所有 BSW 初始化
 * AUTOSAR 规范: SWS_EcuM_00405
 *=====================================================================*/
static void EcuM_StartupTwo(void)
{
    /*==================== 1. 低级 BSW 初始化 ====================*/

    /* 初始化通用 BSW 模块 */
    EcuM_InitPrgHw();      /* 程序硬件抽象层 */
    EcuM_InitIoHw();       /* IO 硬件抽象层 */

    /* 初始化通信栈 */
    EcuM_InitComStack();   /* 通信栈初始化（Can, Lin, FlexRay 等） */

    /* 初始化加密相关 */
    EcuM_InitCrypto();     /* 加密模块（Crypto） */

    /* 初始化功能安全相关 */
    EcuM_InitSafety();     /* 安全模块（FlsTst, CPU STC 等） */

    /* 初始化 NvM 完整版 */
    NvM_ReadAll();         /* 从 NVRAM 读取所有数据 */

    /*==================== 2. OS 启动 ====================*/

    /* 所有 OS 启动前的 BSW 初始化完成 */
    /* 现在调用 StartOS 启动操作系统 */

    /* 设置状态为 OS 启动中 */
    EcuM_InternalState = ECU_M_STATE_STARTING_OS;

    /* 调用 OS 启动 */
    /* 注意：StartOS 不会返回（除非 OS 配置为可返回） */
    StartOS();
}

/*=====================================================================
 * 函数: EcuM_OnStartupTwoComplete
 * 描述: OS 启动完成后的回调
 *       此函数由 OS 在调度器启动完成后回调
 *       在 StartOS 内部，OS 初始化完成后会调用此函数
 * AUTOSAR 规范: SWS_EcuM_00406
 *=====================================================================*/
void EcuM_OnStartupTwoComplete(void)
{
    /* 设置状态为正常运行 */
    EcuM_InternalState = ECU_M_STATE_RUNNING;

    /*==================== 3. 高中级 BSW 初始化 ====================*/

    /* OS 已经运行，可以创建任务和中断 */
    /* 初始化高级 BSW 模块 */

    /* 初始化 BSW 模式管理 */
    BswM_Init(EcuM_GetBswMConfig());

    /* 初始化通信管理器 */
    ComM_Init();

    /* 初始化 PDU 路由器 */
    PduR_Init();

    /* 初始化 COM 层 */
    Com_Init();

    /* 初始化存储栈 */
    NvM_InitBlock(NVM_BLOCK_ALL);

    /* 初始化诊断栈 */
    Dem_Init();
    Dcm_Init();

    /* 初始化 I/O 栈 */
    IoHwAb_InitModules();

    /*==================== 4. RTE 初始化 ====================*/

    /* RTE 初始化 */
    Rte_Init();
}

/*=====================================================================
 * 函数: EcuM_MainFunction
 * 描述: EcuM 主函数，由 SchM 周期性调度
 * 负责处理启动后的状态机转换
 *=====================================================================*/
void EcuM_MainFunction(void)
{
    /* 检查是否有待处理的唤醒事件 */
    if (EcuM_PendingWakeupEvents != 0U)
    {
        /* 处理唤醒事件 */
        EcuM_ProcessWakeupEvents();
    }

    /* 检查是否需要进入睡眠或关闭 */
    switch (EcuM_InternalState)
    {
        case ECU_M_STATE_GO_SLEEP:
            EcuM_ProcessSleepSequence();
            break;

        case ECU_M_STATE_GO_SHUTDOWN:
            EcuM_ProcessShutdownSequence();
            break;

        default:
            /* 正常运行 */
            break;
    }
}

/*=====================================================================
 * 函数: EcuM_GetWakeupEvents
 * 描述: 获取当前所有待处理的唤醒事件
 *       检查各通信模块和唤醒源
 *=====================================================================*/
static EcuM_WakeupEventType EcuM_GetWakeupEvents(void)
{
    EcuM_WakeupEventType events = 0U;

    /* 检查各唤醒源 */
    if (EcuM_CheckWakeupSource(ECU_M_WAKEUP_SOURCE_POWER))
    {
        events |= ECU_M_WAKEUP_EVENT_POWER_ON;
    }

    if (EcuM_CheckWakeupSource(ECU_M_WAKEUP_SOURCE_CAN))
    {
        events |= ECU_M_WAKEUP_EVENT_CAN;
    }

    if (EcuM_CheckWakeupSource(ECU_M_WAKEUP_SOURCE_LIN))
    {
        events |= ECU_M_WAKEUP_EVENT_LIN;
    }

    if (EcuM_CheckWakeupSource(ECU_M_WAKEUP_SOURCE_FLEXRAY))
    {
        events |= ECU_M_WAKEUP_EVENT_FLEXRAY;
    }

    if (EcuM_CheckWakeupSource(ECU_M_WAKEUP_SOURCE_TIMER))
    {
        events |= ECU_M_WAKEUP_EVENT_TIMER;
    }

    return events;
}

/*=====================================================================
 * 函数: EcuM_ProcessShutdownSequence
 * 描述: 执行 ECU 关闭序列
 *       关闭顺序与启动顺序相反
 *=====================================================================*/
static void EcuM_ProcessShutdownSequence(void)
{
    /* 1. 通知所有 SWC 准备关闭 */
    Rte_Stop();

    /* 2. 关闭通信栈 */
    Com_DeInit();
    PduR_DeInit();
    ComM_DeInit();

    /* 3. 关闭 BSW 模式管理 */
    BswM_DeInit();

    /* 4. 关闭 OS */
    (void)ShutdownOS(E_OK);

    /* 5. 关闭低级 BSW */
    /* 此处不应再执行，因为 OS 已关闭 */
}
```

### 4\.4 EcuM 配置示例（XML 片段）

```xml
<!-- EcuM 配置片段 -->
<!-- 来自 EcuM_Cfg.h 或 EB tresos/Vector DaVinci 配置 -->

<EcuMConfig>
    <!-- 启动模式配置 -->
    <EcuMStartupMode>
        <EcuMStartupModeId>ECU_M_STARTUP_TWO</EcuMStartupModeId>
        <EcuMStartupModeName>FullStartup</EcuMStartupModeName>
    </EcuMStartupMode>

    <!-- 唤醒源配置 -->
    <EcuMWakeupSource>
        <EcuMWakeupSourceId>ECU_M_WAKEUP_SOURCE_CAN</EcuMWakeupSourceId>
        <EcuMWakeupSourceValidation>IMMEDIATE</EcuMWakeupSourceValidation>
        <EcuMWakeupSourceTimeout>100.0</EcuMWakeupSourceTimeout>  <!-- ms -->
    </EcuMWakeupSource>

    <!-- 固定唤醒源验证时间 -->
    <EcuMFixedWakeupValidationTime>50.0</EcuMFixedWakeupValidationTime>
</EcuMConfig>
```

---

## 5\. 阶段三：操作系统启动（OS）

### 5\.1 AUTOSAR OS 启动流程

### 5\.2 OS 启动代码实现

```c
/*=====================================================================
 * 文件名: Os.c (AUTOSAR OS 核心启动)
 * 描述: AUTOSAR 操作系统启动实现
 *=====================================================================*/

#include "Os.h"
#include "Os_Internal.h"
#include "EcuM.h"
#include "SchM.h"

/* 内核全局状态 */
static Os_KernelStateType      Os_State = OS_STATE_INITIALIZING;
static Os_IsrStackType        *Os_IsrStackPtr = NULL_PTR;
static Os_TaskControlBlockType *Os_RunningTask = NULL_PTR;

/* 启动配置 */
static const Os_StartupConfigType Os_StartupConfig =
{
    .maxTasksPerPriority = 5U,
    .schedulerPolicy     = OS_SCHED_POLICY_PREEMPTIVE,
    .stackGuardEnable    = TRUE,
    .timeSliceEnable     = TRUE,
};

/*=====================================================================
 * 函数: StartOS
 * 描述: AUTOSAR OS 启动入口
 * 参数: osCfgId — OS 配置索引（用于多配置场景）
 * 规范: SWS_Os_00001
 * 调用: 由 EcuM_StartupTwo 调用
 * 注意: 此函数通常不会返回
 *=====================================================================*/
void StartOS(Os_CfgIdType osCfgId)
{
    /* 设置 OS 状态 */
    Os_State = OS_STATE_STARTING;

    /*==================== 1. OS 内核初始化 ====================*/

    /* 初始化 OS 内部数据结构 */
    Os_InitCore();

    /* 初始化 OS 内核对象 */
    Os_InitKernel();

    /* 初始化系统计数器 */
    Os_InitCounters();

    /* 初始化调度表 */
    Os_InitScheduleTables();

    /* 初始化资源 */
    Os_InitResources();

    /* 初始化警报 */
    Os_InitAlarms();

    /* 初始化任务控制块 */
    Os_InitTasks();

    /* 初始化中断控制器 */
    Os_InitInterruptController();

    /*==================== 2. OS 启动回调 ====================*/

    /* 通知 EcuM OS 启动完成 */
    /* 在此回调中，EcuM 会初始化中级和高级 BSW 模块 */
    EcuM_OnStartupTwoComplete();

    /*==================== 3. 启动调度器 ====================*/

    /* 设置 OS 状态为运行中 */
    Os_State = OS_STATE_RUNNING;

    /* 启动调度器 — 开始多任务调度 */
    Os_StartScheduler();

    /* 调度器启动后，控制权移交给调度器 */
    /* 以下代码仅在 OS 关闭后执行 */
}

/*=====================================================================
 * 函数: Os_InitCore
 * 描述: 初始化 OS 核心数据结构
 *=====================================================================*/
static void Os_InitCore(void)
{
    uint32 i;

    /* 初始化核心数据结构 */
    for (i = 0U; i < OS_MAX_CORES; i++)
    {
        Os_CoreContext[i].coreId      = i;
        Os_CoreContext[i].currentTask = NULL_PTR;
        Os_CoreContext[i].schedulerLock = 0U;
        Os_CoreContext[i].interruptNestingLevel = 0U;
    }
}

/*=====================================================================
 * 函数: Os_InitTasks
 * 描述: 初始化所有 OS 任务控制块 (TCB)
 * 每个任务在配置时指定了：
 *   - 任务优先级
 *   - 栈大小
 *   - 调度策略（抢占/非抢占）
 *   - 自动启动属性
 *=====================================================================*/
static void Os_InitTasks(void)
{
    uint32 taskIdx;

    for (taskIdx = 0U; taskIdx < OS_TASK_COUNT; taskIdx++)
    {
        Os_TaskControlBlockType *tcb = &Os_TaskTable[taskIdx];

        /* 初始化 TCB 关键字段 */
        tcb->taskId         = (Os_TaskType)taskIdx;
        tcb->taskState      = TASK_STATE_SUSPENDED;
        tcb->isPreemptive   = Os_TaskConfig[taskIdx].isPreemptive;
        tcb->priority       = Os_TaskConfig[taskIdx].priority;
        tcb->stackBase      = Os_TaskStack[taskIdx];
        tcb->stackSize      = Os_TaskConfig[taskIdx].stackSize;
        tcb->stackPointer   = NULL_PTR;
        tcb->activationCount = 0U;

        /* 初始化任务上下文 */
        Os_InitTaskContext(tcb);

        /* 如果任务配置为自动启动 */
        if (Os_TaskConfig[taskIdx].autoStart != FALSE)
        {
            Os_ActivateTask(taskIdx, /* fromStartup */ TRUE);
        }
    }
}

/*=====================================================================
 * 函数: Os_StartScheduler
 * 描述: 启动 OS 调度器
 * 这是 OS 进入多任务模式的关键步骤
 * 调度器启动后，将执行最高优先级的就绪任务
 *=====================================================================*/
static void Os_StartScheduler(void)
{
    /* 关闭中断，防止调度器初始化被中断 */
    Os_EnterCoreCriticalSection();

    /* 设置调度器状态 */
    Os_State = OS_STATE_SCHEDULER_STARTING;

    /* 初始化调度器内部数据结构 */
    Os_SchedCtx.scheduleRequested = FALSE;
    Os_SchedCtx.isInSchedule     = FALSE;

    /* 初始化就绪队列 */
    Os_InitReadyQueues();

    /* 使能系统定时器中断 */
    Os_StartSystemTimer();

    /* 初始化系统滴答 */
    Os_SystemTick = 0U;

    /* 打开中断，允许调度 */
    Os_ExitCoreCriticalSection();

    /* 执行首次调度 */
    /* 选择最高优先级的就绪任务 */
    Os_TaskControlBlockType *nextTask = Os_Schedule_GetNextTask();

    if (nextTask != NULL_PTR)
    {
        /* 执行上下文切换，进入第一个任务 */
        Os_ContextSwitch_StartFirstTask(nextTask);
    }

    /* 正常情况下不会执行到这里 */
    /* 只有在 OS 关闭时才会返回 */
}

/*=====================================================================
 * 函数: Os_ActivateTask
 * 描述: 激活（使能就绪）一个 OS 任务
 * 由 OS 配置中的自动启动任务调用，或由其他任务/中断调用
 *=====================================================================*/
StatusType Os_ActivateTask(TaskType taskId, boolean fromStartup)
{
    StatusType status = E_OK;

    /* 参数检查 */
    if (taskId >= OS_TASK_COUNT)
    {
        return E_OS_ID;
    }

    /* 获取任务控制块 */
    Os_TaskControlBlockType *tcb = &Os_TaskTable[taskId];

    /* 检查激活次数限制 */
    if (tcb->activationCount >= OS_TASK_MAX_ACTIVATIONS)
    {
        return E_OS_LIMIT;
    }

    /* 更新任务状态 */
    Os_EnterCoreCriticalSection();
    tcb->activationCount++;
    tcb->taskState = TASK_STATE_READY;

    /* 将任务加入就绪队列 */
    Os_ReadyQueue_Enqueue(tcb);

    /* 如果不是在启动阶段，触发调度 */
    if (fromStartup == FALSE)
    {
        Os_Schedule();
    }
    Os_ExitCoreCriticalSection();

    return status;
}
```

### 5\.3 OS 调度器启动后任务布局

---

## 6\. 阶段四：SchM 与 BSW 各模块初始化

### 6\.1 SchM 角色

**SchM（Scheduler，调度器）** 是 AUTOSAR 中负责 BSW 函数调度的模块。它不是一个独立的模块，而是 **BSW 中各个模块的调度机制的总称**。

### 6\.2 BSW 模块初始化顺序

### 6\.3 SchM 实现代码

```c
/*=====================================================================
 * 文件名: SchM.c
 * 描述: BSW 调度器实现
 *       管理各 BSW 模块的周期性调用
 *=====================================================================*/

#include "SchM.h"
#include "SchM_Com.h"
#include "SchM_EcuM.h"
#include "SchM_BswM.h"
#include "SchM_Wdg.h"
#include "SchM_Dem.h"
#include "SchM_Can.h"
#include "SchM_Lin.h"

/* 调度表配置 */
static SchM_ScheduleTableType SchM_ScheduleTable[SCHM_SCHEDULE_TABLE_COUNT];

/* 调度模式状态 */
static SchM_ModeType SchM_CurrentMode = SCHM_MODE_INIT;

/*=====================================================================
 * 函数: SchM_Init
 * 描述: 初始化 BSW 调度表
 *       在 StartOS 之前被 EcuM 调用
 * 参数: configPtr — 调度配置指针
 *=====================================================================*/
void SchM_Init(const SchM_ConfigType *configPtr)
{
    uint32 tableIdx;

    /* 参数检查 */
    if (configPtr == NULL_PTR)
    {
        Det_ReportError(SCHM_MODULE_ID, 0U, SCHM_INIT_API_ID, SCHM_E_PARAM_CONFIG);
        return;
    }

    /* 初始化调度表 */
    for (tableIdx = 0U; tableIdx < SCHM_SCHEDULE_TABLE_COUNT; tableIdx++)
    {
        SchM_ScheduleTable[tableIdx].scheduleTableId = (SchM_ScheduleTableIdType)tableIdx;
        SchM_ScheduleTable[tableIdx].currentTick     = 0U;
        SchM_ScheduleTable[tableIdx].state           = SCHM_TABLE_STATE_STOPPED;
        SchM_ScheduleTable[tableIdx].expiryPoints    = configPtr->scheduleTableConfig[tableIdx].expiryPoints;
        SchM_ScheduleTable[tableIdx].expiryCount     = configPtr->scheduleTableConfig[tableIdx].expiryCount;
    }

    SchM_CurrentMode = SCHM_MODE_INITIALIZED;
}

/*=====================================================================
 * 函数: SchM_StartScheduleTable
 * 描述: 启动指定的调度表
 *       由 EcuM_OnStartupTwoComplete 或
 *       OS 启动任务调用
 *=====================================================================*/
void SchM_StartScheduleTable(SchM_ScheduleTableIdType tableId)
{
    SchM_ScheduleTableType *table = &SchM_ScheduleTable[tableId];

    /* 重置调度表计数器 */
    table->currentTick = 0U;
    table->state       = SCHM_TABLE_STATE_RUNNING;

    /* 通知 OS 调度表已启动 */
    /* 某些实现中，调度表由 OS 计数器驱动 */
    Os_StartScheduleTable(tableId);
}

/*=====================================================================
 * 函数: SchM_MainFunction
 * 描述: BSW 调度器主函数
 *       由 OS 周期性任务调用
 *       根据调度表到期点触发各 BSW 模块的 MainFunction
 *=====================================================================*/
void SchM_MainFunction(void)
{
    uint32 tableIdx;

    for (tableIdx = 0U; tableIdx < SCHM_SCHEDULE_TABLE_COUNT; tableIdx++)
    {
        SchM_ScheduleTableType *table = &SchM_ScheduleTable[tableIdx];

        if (table->state != SCHM_TABLE_STATE_RUNNING)
        {
            continue;
        }

        /* 获取当前到期点 */
        uint32 tick = table->currentTick;
        uint32 epIdx;

        /* 遍历所有到期点 */
        for (epIdx = 0U; epIdx < table->expiryCount; epIdx++)
        {
            const SchM_ExpiryPointType *expiry = &table->expiryPoints[epIdx];

            if (expiry->tickOffset == tick)
            {
                /* 执行该到期点的所有调度动作 */
                SchM_ExecuteExpiryPoint(expiry);
            }
        }

        /* 更新调度表滴答 */
        table->currentTick++;
    }
}

/*=====================================================================
 * 函数: SchM_ExecuteExpiryPoint
 * 描述: 执行到期点中配置的所有调度动作
 *       每个到期点可以包含多个 BSW 模块的 MainFunction
 *=====================================================================*/
static void SchM_ExecuteExpiryPoint(const SchM_ExpiryPointType *expiry)
{
    uint32 actionIdx;

    for (actionIdx = 0U; actionIdx < expiry->actionCount; actionIdx++)
    {
        const SchM_ActionType *action = &expiry->actions[actionIdx];

        switch (action->moduleId)
        {
            /* 通信模块 */
            case SCHM_MODULE_COM:
                Com_MainFunction();
                break;

            case SCHM_MODULE_CAN:
                Can_MainFunction_Read();
                Can_MainFunction_Write();
                break;

            case SCHM_MODULE_LIN:
                Lin_MainFunction();
                break;

            case SCHM_MODULE_PDUR:
                PduR_MainFunction();
                break;

            /* 系统模块 */
            case SCHM_MODULE_ECUM:
                EcuM_MainFunction();
                break;

            case SCHM_MODULE_BSWM:
                BswM_MainFunction();
                break;

            /* 诊断模块 */
            case SCHM_MODULE_DEM:
                Dem_MainFunction();
                break;

            case SCHM_MODULE_DCM:
                Dcm_MainFunction();
                break;

            /* 看门狗 */
            case SCHM_MODULE_WDG:
                WdgM_MainFunction();
                break;

            /* 存储模块 */
            case SCHM_MODULE_NVM:
                NvM_MainFunction();
                break;

            default:
                /* 未知模块，可选报告错误 */
                break;
        }
    }
}

/*=====================================================================
 * 调度表配置示例（需在配置工具中生成）
 * 典型调度表结构：
 *   到期点 0 (tick=0):   Com, EcuM
 *   到期点 1 (tick=2):   Can, PduR
 *   到期点 2 (tick=4):   BswM, Dem
 *   到期点 3 (tick=6):   Wdg, NvM
 *=====================================================================*/
const SchM_ScheduleTableConfigType SchM_ExampleScheduleConfig[SCHM_SCHEDULE_TABLE_COUNT] =
{
    {
        .scheduleTableId = 0U,
        .expiryCount     = 4U,
        .expiryPoints    =
        {
            {
                .tickOffset  = 0U,      /* 主调度点 */
                .actionCount = 2U,
                .actions     =
                {
                    { .moduleId = SCHM_MODULE_COM,  .priority = 1U },
                    { .moduleId = SCHM_MODULE_ECUM, .priority = 2U },
                }
            },
            {
                .tickOffset  = 2U,      /* 通信调度点 */
                .actionCount = 2U,
                .actions     =
                {
                    { .moduleId = SCHM_MODULE_CAN,  .priority = 1U },
                    { .moduleId = SCHM_MODULE_PDUR, .priority = 2U },
                }
            },
            {
                .tickOffset  = 4U,      /* 模式管理调度点 */
                .actionCount = 2U,
                .actions     =
                {
                    { .moduleId = SCHM_MODULE_BSWM, .priority = 1U },
                    { .moduleId = SCHM_MODULE_DEM,  .priority = 2U },
                }
            },
            {
                .tickOffset  = 6U,      /* 监控调度点 */
                .actionCount = 2U,
                .actions     =
                {
                    { .moduleId = SCHM_MODULE_WDG, .priority = 1U },
                    { .moduleId = SCHM_MODULE_NVM, .priority = 2U },
                }
            }
        }
    }
};
```

---

## 7\. 阶段五：RTE 与 SWC 启动

### 7\.1 RTE 初始化流程

### 7\.2 RTE 启动代码实现

```c
/*=====================================================================
 * 文件名: Rte.c
 * 描述: AUTOSAR RTE（运行时环境）实现
 *       提供 SWC 之间的通信和调度机制
 *=====================================================================*/

#include "Rte.h"
#include "Rte_Internal.h"
#include "Rte_SwcQueue.h"
#include "Rte_Type.h"

/* RTE 全局状态 */
static Rte_StateType Rte_State = RTE_STATE_UNINITIALIZED;

/*=====================================================================
 * 函数: Rte_Init
 * 描述: 初始化 RTE 层
 *       在 EcuM_OnStartupTwoComplete 中调用
 *       初始化所有 SWC 的通信通道和缓冲区
 *=====================================================================*/
void Rte_Init(void)
{
    Rte_State = RTE_STATE_INITIALIZING;

    /* 1. 初始化 RTE 内存管理 */
    Rte_InitMemory();

    /* 2. 初始化所有 SWC 之间的通信缓冲区 */
    Rte_InitBuffers();

    /* 3. 初始化触发队列 */
    Rte_InitTriggerQueues();

    /* 4. 注册所有 Runnable Entity */
    Rte_RegisterRunnables();

    Rte_State = RTE_STATE_INITIALIZED;
}

/*=====================================================================
 * 函数: Rte_Start
 * 描述: 启动 RTE，开始调度所有 SWC 的 Runnable Entity
 *       通常在 EcuM_OnStartupTwoComplete 最后调用
 *=====================================================================*/
void Rte_Start(void)
{
    Rte_State = RTE_STATE_RUNNING;

    /* 激活所有定时触发的 Runnable Entity */
    Rte_StartTimedRunnables();

    /* 激活所有初始化触发的 Runnable Entity */
    Rte_StartInitRunnables();
}

/*=====================================================================
 * 函数: Rte_InitBuffers
 * 描述: 初始化所有 SWC 之间的通信缓冲区
 *       包括 Sender-Receiver 和 Client-Server 通信机制
 *=====================================================================*/
static void Rte_InitBuffers(void)
{
    uint32 bufferIdx;

    /* 初始化所有 S/R 通信缓冲区 */
    for (bufferIdx = 0U; bufferIdx < RTE_SR_BUFFER_COUNT; bufferIdx++)
    {
        Rte_SrBufferType *buffer = &Rte_SrBuffers[bufferIdx];

        buffer->dataPtr     = Rte_SrBufferData[bufferIdx];
        buffer->dataSize    = Rte_SrBufferConfig[bufferIdx].size;
        buffer->isQueued    = Rte_SrBufferConfig[bufferIdx].isQueued;
        buffer->writeIndex  = 0U;
        buffer->readIndex   = 0U;

        if (buffer->isQueued)
        {
            buffer->queueLength = Rte_SrBufferConfig[bufferIdx].queueLength;
            buffer->elementSize = Rte_SrBufferConfig[bufferIdx].elementSize;
        }

        /* 初始化数据为默认值 */
        (void)Rte_InitBufferData(buffer);
    }

    /* 初始化所有 C/S 通信缓冲区 */
    for (bufferIdx = 0U; bufferIdx < RTE_CS_BUFFER_COUNT; bufferIdx++)
    {
        Rte_CsBufferType *csBuffer = &Rte_CsBuffers[bufferIdx];

        csBuffer->requestAvailable = FALSE;
        csBuffer->responseReady    = FALSE;
        csBuffer->result           = RTE_E_OK;
    }
}

/*=====================================================================
 * 函数: Rte_RegisterRunnables
 * 描述: 注册所有可运行实体到 RTE 调度表
 *       每个 Runnable Entity 与一个触发事件相关联
 *       （定时触发、数据触发、操作调用触发等）
 *=====================================================================*/
static void Rte_RegisterRunnables(void)
{
    uint32 runnableIdx;

    for (runnableIdx = 0U; runnableIdx < RTE_RUNNABLE_COUNT; runnableIdx++)
    {
        const Rte_RunnableDefType *runnable = &Rte_RunnableDefs[runnableIdx];

        /* 注册可运行实体 */
        Rte_RunnableTable[runnableIdx].runnableId    = runnable->runnableId;
        Rte_RunnableTable[runnableIdx].entryPoint     = runnable->entryPoint;
        Rte_RunnableTable[runnableIdx].triggerType    = runnable->triggerType;
        Rte_RunnableTable[runnableIdx].isActive       = FALSE;

        /* 如果是定时触发，注册到 OS 计数器 */
        if (runnable->triggerType == RTE_TRIGGER_TIMING)
        {
            Rte_RegisterTimedRunnable(&Rte_RunnableTable[runnableIdx]);
        }

        /* 如果是数据触发，注册到数据接收回调 */
        if (runnable->triggerType == RTE_TRIGGER_DATA_RECEIVED)
        {
            Rte_RegisterDataTriggeredRunnable(&Rte_RunnableTable[runnableIdx]);
        }

        /* 如果是初始化触发，标记为启动时执行 */
        if (runnable->triggerType == RTE_TRIGGER_INIT)
        {
            Rte_RunnableTable[runnableIdx].isActive = TRUE;
        }
    }
}

/*=====================================================================
 * 函数: Rte_StartInitRunnables
 * 描述: 执行所有标记为 INIT 类型的 Runnable Entity
 *       这是 SWC 初始化的入口
 *=====================================================================*/
static void Rte_StartInitRunnables(void)
{
    uint32 runnableIdx;

    for (runnableIdx = 0U; runnableIdx < RTE_RUNNABLE_COUNT; runnableIdx++)
    {
        Rte_RunnableType *runnable = &Rte_RunnableTable[runnableIdx];

        if (runnable->triggerType == RTE_TRIGGER_INIT)
        {
            /* 调用 Runnable Entity */
            Rte_CallRunnable(runnable->runnableId);
        }
    }
}

/*=====================================================================
 * 函数: Rte_CallRunnable
 * 描述: 调用指定的 Runnable Entity（可运行实体）
 *       这是 RTE 执行 SWC 功能的核心机制
 *=====================================================================*/
void Rte_CallRunnable(Rte_RunnableIdType runnableId)
{
    if (runnableId >= RTE_RUNNABLE_COUNT)
    {
        Det_ReportError(RTE_MODULE_ID, 0U, RTE_CALL_RUNNABLE_ID, RTE_E_RUNNABLE_ID);
        return;
    }

    Rte_RunnableType *runnable = &Rte_RunnableTable[runnableId];
    Rte_StateType prevState = Rte_State;

    /* 设置运行状态 */
    Rte_State = RTE_STATE_RUNNING;

    /* 调用实际的函数入口点 */
    if (runnable->entryPoint != NULL_PTR)
    {
        /* 执行 SWC 的 Runnable Entity */
        runnable->entryPoint();
    }

    /* 恢复状态 */
    Rte_State = prevState;
}
```

---

## 8\. 阶段六：BSM 模式管理与正常运行

### 8\.1 BswM 模式管理

### 8\.2 BswM 初始化代码

```c
/*=====================================================================
 * 文件名: BswM.c
 * 描述: BSW 模式管理器实现
 *       管理 BSW 各模块的模式状态
 *=====================================================================*/

#include "BswM.h"
#include "BswM_Internal.h"
#include "Rte.h"
#include "EcuM.h"
#include "ComM.h"

/* BswM 内部状态 */
static BswM_StateType BswM_CurrentState = BSWM_STATE_UNINITIALIZED;

/*=====================================================================
 * 函数: BswM_Init
 * 描述: 初始化 BSW 模式管理器
 *       在 EcuM_OnStartupTwoComplete 中调用
 *=====================================================================*/
void BswM_Init(const BswM_ConfigType *configPtr)
{
    BswM_CurrentState = BSWM_STATE_INITIALIZED;

    /* 注册模式请求回调 */
    BswM_RegisterModeRequestor(BSWM_REQUESTOR_ECUM, EcuM_GetRequestedMode);
    BswM_RegisterModeRequestor(BSWM_REQUESTOR_COMM, ComM_GetCurrentMode);

    /* 进入默认模式 */
    BswM_RequestMode(BSWM_MODE_RUN, BSWM_REQUESTOR_SYSTEM);
}

/*=====================================================================
 * 函数: BswM_MainFunction
 * 描述: BswM 主函数，由 SchM 周期性调度
 *       处理模式请求和模式转换
 *=====================================================================*/
void BswM_MainFunction(void)
{
    /* 处理所有待处理的模式请求 */
    BswM_ProcessModeRequests();

    /* 执行模式转换动作 */
    BswM_ExecuteModeActions();
}

/*=====================================================================
 * 函数: BswM_RequestMode
 * 描述: 请求模式切换
 *       由各 BSW 模块或 SWC 调用
 *=====================================================================*/
Std_ReturnType BswM_RequestMode(BswM_ModeType mode, BswM_RequestorType requestor)
{
    BswM_RequestType *request = BswM_GetRequestSlot(requestor);

    if (request == NULL_PTR)
    {
        return E_NOT_OK;
    }

    request->requestedMode = mode;
    request->isPending     = TRUE;

    return E_OK;
}
```

---

## 9\. 完整时序图

### 9\.1 全局启动时序

### 9\.2 启动过程的初始化函数调用链

---

## 10\. 代码示例

### 10\.1 完整启动代码示例（集成所有模块）

```c
/*=====================================================================
 * 文件名: Startup.c
 * 描述: AUTOSAR 完整启动流程集成示例
 *       展示从 main() 到应用运行的全过程
 *=====================================================================*/

#include "EcuM.h"
#include "Os.h"
#include "BswM.h"
#include "SchM.h"
#include "Rte.h"
#include "Det.h"
#include "NvM.h"
#include "Dem.h"
#include "Com.h"
#include "ComM.h"
#include "PduR.h"
#include "Can.h"
#include "WdgM.h"
#include "Fee.h"
#include "Fls.h"

/* 全局启动状态 */
typedef enum
{
    STARTUP_PHASE_HW_INIT,       /* 硬件初始化       */
    STARTUP_PHASE_ECUM_INIT,     /* EcuM 初始化       */
    STARTUP_PHASE_BSW_LOW,       /* 低级 BSW 初始化   */
    STARTUP_PHASE_OS_START,      /* OS 启动           */
    STARTUP_PHASE_BSW_MID,       /* 中级 BSW 初始化   */
    STARTUP_PHASE_SCHM,          /* 调度器启动        */
    STARTUP_PHASE_BSW_HIGH,      /* 高级 BSW 初始化   */
    STARTUP_PHASE_RTE,           /* RTE 初始化        */
    STARTUP_PHASE_SWC,           /* SWC 启动          */
    STARTUP_PHASE_RUNNING,       /* 正常运行          */
    STARTUP_PHASE_SHUTDOWN       /* 关机              */
} StartupPhaseType;

static StartupPhaseType CurrentPhase = STARTUP_PHASE_HW_INIT;

/* 启动时间统计 */
static uint32 StartupTimeMs = 0U;

/* 启动阶段名称表 */
static const char *const StartupPhaseNames[] =
{
    "HW_INIT",
    "ECUM_INIT",
    "BSW_LOW",
    "OS_START",
    "BSW_MID",
    "SCHM",
    "BSW_HIGH",
    "RTE",
    "SWC",
    "RUNNING",
    "SHUTDOWN"
};

/*=====================================================================
 * 函数: Startup_GetPhaseName
 * 描述: 获取当前启动阶段名称
 *=====================================================================*/
const char *Startup_GetPhaseName(void)
{
    if (CurrentPhase < sizeof(StartupPhaseNames) / sizeof(StartupPhaseNames[0]))
    {
        return StartupPhaseNames[CurrentPhase];
    }
    return "UNKNOWN";
}

/*=====================================================================
 * 函数: Startup_ReportPhase
 * 描述: 报告当前启动阶段（用于调试/诊断）
 *=====================================================================*/
static void Startup_ReportPhase(StartupPhaseType phase)
{
    CurrentPhase = phase;

    /* 记录启动时间戳 */
    StartupTimeMs = Os_GetSystemTimeMs();

    /* 通过 Dem 记录启动阶段（用于诊断） */
    Dem_ReportEvent(DEM_EVENT_STARTUP_PHASE,
                    (uint8 *)&CurrentPhase,
                    sizeof(CurrentPhase));

    /* 调试输出 */
    #ifdef DEBUG_STARTUP
        DEBUG_PRINT("[STARTUP] Phase: %s @ %d ms\n",
                    StartupPhaseNames[phase], StartupTimeMs);
    #endif
}

/*=====================================================================
 * 函数: main
 * 描述: AUTOSAR 应用程序入口
 *       这是启动过程的 C 语言入口点
 *=====================================================================*/
int main(void)
{
    /* 硬件初始化已在 startup.s 中完成 */
    /* 此处直接进入 AUTOSAR BSW 启动 */

    /* 1. 低级 BSW 初始化 */
    Startup_ReportPhase(STARTUP_PHASE_BSW_LOW);

    /* 初始化错误检测模块 */
    Det_Init();

    /* 初始化诊断事件管理器 */
    Dem_Init();

    /* 初始化非易失性存储器 */
    NvM_Init();

    /* 2. EcuM 初始化 */
    Startup_ReportPhase(STARTUP_PHASE_ECUM_INIT);
    EcuM_Init();

    /* 3. OS 启动 */
    Startup_ReportPhase(STARTUP_PHASE_OS_START);
    StartOS();  /* 不会返回 */

    /* 4. 只有在 OS 关闭后才会执行到这里 */
    Startup_ReportPhase(STARTUP_PHASE_SHUTDOWN);

    return 0;
}

/*=====================================================================
 * 函数: EcuM_OnStartupTwoComplete
 * 描述: OS 启动完成回调
 *       在此函数中完成中级和高级 BSW 初始化
 *=====================================================================*/
void EcuM_OnStartupTwoComplete(void)
{
    /* 4. 中级 BSW 初始化 */
    Startup_ReportPhase(STARTUP_PHASE_BSW_MID);

    /* 初始化 BSW 模式管理器 */
    BswM_Init(BswM_GetConfig());

    /* 初始化通信管理器 */
    ComM_Init(ComM_GetConfig());

    /* 初始化 PDU 路由器 */
    PduR_Init(PduR_GetConfig());

    /* 初始化 CAN 模块 */
    Can_Init(Can_GetConfig());

    /* 初始化 LIN 模块 */
    Lin_Init(Lin_GetConfig());

    /* 初始化 COM 层 */
    Com_Init(Com_GetConfig());

    /* 初始化 NvM 所有块 */
    NvM_InitBlock(NVM_BLOCK_ALL);

    /* 初始化诊断栈 */
    Dem_InitPostOs();
    Dcm_Init(Dcm_GetConfig());

    /* 5. 启动调度器 */
    Startup_ReportPhase(STARTUP_PHASE_SCHM);

    /* 启动 BSW 调度表 */
    SchM_StartScheduleTable(SCHM_DEFAULT_SCHEDULE_TABLE);

    /* 启动看门狗管理 */
    WdgM_Init(WdgM_GetConfig());

    /* 6. 高级 BSW 初始化 */
    Startup_ReportPhase(STARTUP_PHASE_BSW_HIGH);

    /* 初始化 Fls / Fee */
    Fls_Init(Fls_GetConfig());
    Fee_Init(Fee_GetConfig());

    /* 7. RTE 初始化 */
    Startup_ReportPhase(STARTUP_PHASE_RTE);
    Rte_Init();

    /* 8. SWC 启动 */
    Startup_ReportPhase(STARTUP_PHASE_SWC);
    Rte_Start();

    /* 9. 系统进入正常运行模式 */
    Startup_ReportPhase(STARTUP_PHASE_RUNNING);
}

/*=====================================================================
 * 函数: EcuM_OnStartupOneComplete
 * 描述: StartupOne 模式完成回调
 *       此时 ECU 进入休眠等待模式
 *=====================================================================*/
void EcuM_OnStartupOneComplete(void)
{
    /* 最小化启动完成，进入休眠 */
    Startup_ReportPhase(STARTUP_PHASE_BSW_LOW);

    /* 配置唤醒源 */
    EcuM_InitWakeupSources();
}
```

### 10\.2 链接脚本示例（内存布局）

```ld
/*=====================================================================
 * 文件名: LinkerScript.ld
 * 描述: AUTOSAR 启动相关的内存布局
 *=====================================================================*/

MEMORY
{
    FLASH (rx)  : ORIGIN = 0x08000000, LENGTH = 1024K
    RAM   (rwx) : ORIGIN = 0x20000000, LENGTH = 128K
    NVRAM (rw)  : ORIGIN = 0x08080000, LENGTH = 64K   /* 模拟 EEPROM */
}

SECTIONS
{
    /* 中断向量表 — 必须位于 Flash 起始地址 */
    .isr_vector :
    {
        . = ALIGN(4);
        KEEP(*(.isr_vector))
        . = ALIGN(4);
    } > FLASH

    /* 启动代码 */
    .startup :
    {
        . = ALIGN(4);
        *(.text.Reset_Handler)
        *(.startup)
        . = ALIGN(4);
    } > FLASH

    /* 应用程序代码 */
    .text :
    {
        . = ALIGN(4);
        *(.text*)
        *(.rodata*)
        . = ALIGN(4);
    } > FLASH

    /* 初始化数据段（从 Flash 拷贝到 RAM） */
    .data :
    {
        . = ALIGN(4);
        __data_start__ = .;
        *(.data*)
        __data_end__ = .;
    } > RAM AT > FLASH
    __data_load_start__ = LOADADDR(.data);

    /* BSS 段（运行时清零） */
    .bss :
    {
        . = ALIGN(4);
        __bss_start__ = .;
        *(.bss*)
        *(COMMON)
        __bss_end__ = .;
    } > RAM

    /* NvM 块（非易失性存储） */
    .nvram_block :
    {
        . = ALIGN(4);
        __nvm_start__ = .;
        *(.nvram*)
        __nvm_end__ = .;
    } > NVRAM

    /* OS 内核栈 */
    .os_stack (NOLOAD) :
    {
        . = ALIGN(8);
        __os_stack_start__ = .;
        . = . + 1024;  /* 1KB OS 内核栈 */
        __os_stack_end__ = .;
    } > RAM

    /* 堆 */
    .heap (NOLOAD) :
    {
        . = ALIGN(8);
        __heap_start__ = .;
        . = . + 4096;  /* 4KB 堆 */
        __heap_end__ = .;
    } > RAM
}
```

---

## 11\. 设计机制与设计模式分析

### 11\.1 关键设计模式

|设计模式|AUTOSAR 中的应用|说明|
|---|---|---|
|**分层架构**|硬件 → EcuM → OS → BSW → RTE → SWC|每一层封装下层细节，暴露标准化接口|
|**状态机**|EcuM 状态机、BswM 状态机|启动/关闭/睡眠/唤醒等状态转换|
|**回调模式**|EcuM\_OnStartupTwoComplete|下层初始化完成后通知上层|
|**观察者模式**|BswM 模式请求|多个模块监听模式变化|
|**策略模式**|StartupOne vs StartupTwo|根据唤醒源选择不同的启动策略|
|**调度表**|SchM 调度表|周期性调度 BSW 各模块的 MainFunction|
|**分层初始化**|低级 BSW → OS → 中级 BSW → 高级 BSW|按依赖关系分层初始化|
|**守护者模式**|WdgM 看门狗管理|监控系统运行状态|

### 11\.2 启动顺序的依赖关系

### 11\.3 设计机制分析

1. **两阶段启动（StartupOne / StartupTwo）**：

    - 设计意图：减少不必要的启动时间

    - 当 ECU 从低功耗模式唤醒时，可能不需要完整的 OS 启动

    - StartupOne：仅初始化必要硬件，直接进入休眠

    - StartupTwo：完整初始化，包括 OS 和应用

2. **EcuM 作为"启动总指挥"**：

    - 设计意图：集中管理启动流程，避免各模块自启

    - 所有模块的初始化由 EcuM 按顺序调用

    - 保证了模块间的依赖关系

3. **OS 启动的"中间"位置**：

    - 设计意图：低级 BSW 初始化不需要 OS 支持

    - 高级 BSW 和 RTE 依赖 OS 的多任务能力

    - OS 作为"分水岭"，分割了启动流程

4. **SchM 调度表**：

    - 设计意图：解耦 BSW 模块的调度周期

    - 每个模块独立配置其 MainFunction 的执行频率

    - 通过调度表统一管理，而非各自独立定时

---

## 12\. 深入原理

### 12\.1 启动时间的时序约束

```c
/*=====================================================================
 * 启动时间约束分析
 * 基于典型的 32 位 MCU（如 Infineon TC3xx / NXP S32K）
 *=====================================================================

 启动阶段时序（典型值）：

 +----------------------------+------------------+------------------+
 | 阶段                        | 时间（ms）       | 累计时间（ms）    |
 +----------------------------+------------------+------------------+
 | 硬件复位 & 时钟初始化       | 0.1 - 0.5       | 0.5              |
 | Bootloader 运行             | 5.0 - 50.0      | 50.5             |
 | EcuM_Init                   | 0.5 - 2.0       | 52.5             |
 | 低级 BSW 初始化             | 1.0 - 5.0       | 57.5             |
 | StartOS                     | 0.2 - 1.0       | 58.5             |
 | 中级 BSW 初始化             | 2.0 - 10.0      | 68.5             |
 | SchM 调度表启动             | 0.1 - 0.2       | 68.7             |
 | 高级 BSW 初始化             | 1.0 - 5.0       | 73.7             |
 | RTE 初始化                  | 0.5 - 2.0       | 75.7             |
 | SWC 初始化                  | 0.5 - 2.0       | 77.7             |
 +----------------------------+------------------+------------------+

 总启动时间：约 50 - 80ms（含 Bootloader）
 目标：通常要求 < 100ms（ISO 26262 时间约束）
*/

/*=====================================================================
 * 启动时间优化策略
 *=====================================================================*/

/**
 * 1. 并行初始化
 *    在 OS 启动后，可以利用多任务并行初始化 BSW 模块
 *    减少串行初始化带来的延迟
 */
void Startup_ParallelInit(void)
{
    /* 创建并行初始化任务 */
    TaskType taskInitCan, taskInitNvm, taskInitDiag;

    /* 并行初始化通信栈 */
    (void)ActivateTask(taskInitCan);

    /* 并行初始化存储栈 */
    (void)ActivateTask(taskInitNvm);

    /* 并行初始化诊断栈 */
    (void)ActivateTask(taskInitDiag);
}

/**
 * 2. 懒初始化（Lazy Initialization）
 *    某些 BSW 模块可以在首次使用时初始化
 *    而非在启动时全部初始化
 */
void Com_Transmit_LazyInit(PduIdType pduId, const PduInfoType *pduInfo)
{
    static boolean comInitialized = FALSE;

    if (!comInitialized)
    {
        /* 首次使用时初始化 */
        Com_Init(Com_GetConfig());
        comInitialized = TRUE;
    }

    /* 正常传输 */
    Com_Transmit(pduId, pduInfo);
}
```

### 12\.2 启动异常处理

```c
/*=====================================================================
 * 启动异常处理机制
 *=====================================================================*/

/**
 * 启动超时监控
 * 使用硬件看门狗或 OS 警报监控启动时间
 * 如果启动超时，执行安全关闭
 */
void Startup_InitWatchdog(void)
{
    /* 设置看门狗超时时间 */
    WdgM_SetTimeout(STARTUP_WATCHDOG_TIMEOUT_MS);

    /* 在 EcuM_Init 前启动看门狗 */
    WdgM_Trigger();
}

void Startup_FeedWatchdog(void)
{
    /* 每完成一个启动阶段，喂一次看门狗 */
    WdgM_Trigger();
}

/**
 * 启动阶段错误处理
 * 如果某个模块初始化失败，根据严重级别决定是否继续启动
 */
Std_ReturnType Startup_HandleInitError(
    uint16 moduleId,
    Std_ReturnType initResult,
    StartupErrorSeverity severity)
{
    switch (severity)
    {
        case STARTUP_ERROR_RECOVERABLE:
            /* 记录错误，继续启动 */
            Dem_ReportEvent(DEM_EVENT_INIT_WARNING, &moduleId, sizeof(moduleId));
            return E_OK;

        case STARTUP_ERROR_CRITICAL:
            /* 记录错误，尝试降级模式 */
            Dem_ReportEvent(DEM_EVENT_INIT_ERROR, &moduleId, sizeof(moduleId));
            BswM_RequestMode(BSWM_MODE_LIMPDOWN, BSWM_REQUESTOR_SYSTEM);
            return E_NOT_OK;

        case STARTUP_ERROR_FATAL:
            /* 致命错误，安全关闭 ECU */
            Dem_ReportEvent(DEM_EVENT_INIT_FATAL, &moduleId, sizeof(moduleId));
            EcuM_RequestShutdown();
            return E_NOT_OK;

        default:
            return E_NOT_OK;
    }
}

/**
 * 启动完成标志
 * 设置启动完成标志，通知各模块系统已就绪
 */
void Startup_MarkComplete(void)
{
    /* 设置全局启动完成标志 */
    volatile uint32 *pStartupFlag = (uint32_t *)STARTUP_FLAG_ADDR;
    *pStartupFlag = STARTUP_FLAG_COMPLETE_VALUE;

    /* 通知 BswM 系统已就绪 */
    BswM_RequestMode(BSWM_MODE_RUN, BSWM_REQUESTOR_SYSTEM);

    /* 启用所有正常运行时中断 */
    Os_EnableAllInterrupts();
}
```

### 12\.3 启动过程的内存布局变化

### 12\.4 启动时序的硬件层面

```c
/*=====================================================================
 * 硬件启动时序 — 示波器级别的时序分析
 *=====================================================================

 以 Infineon TC397 (AURIX) 为例：

 1. 硬件复位阶段（0 - 1μs）
    - 复位引脚释放
    - CPU 从复位向量开始执行
    - 内部稳压器建立

 2. 时钟建立阶段（1 - 100μs）
    - 内部振荡器启动
    - PLL 锁定时间（典型 10-50μs）
    - Flash 等待周期配置

 3. 内存初始化阶段（100 - 500μs）
    - BSS 段清零（取决于 RAM 大小）
    - Data 段拷贝（取决于 Flash 读取速度）

 4. BSW 初始化阶段（500μs - 10ms）
    - 外设寄存器配置
    - 通信控制器初始化
    - NvM 数据读取

 5. OS 启动（10ms - 20ms）
    - 内核对象创建
    - 任务栈分配
    - 调度器启动

 6. RTE/SWC 启动（20ms - 50ms）
    - 通信通道建立
    - 应用初始化
    - 系统就绪
*/

/*=====================================================================
 * 启动过程中的关键寄存器操作
 *=====================================================================*/

/**
 * 复位原因检测
 * 检查是什么原因导致的复位，这对启动模式选择很重要
 */
StartupResetType Startup_DetectResetCause(void)
{
    uint32 resetCause = RCC->CSR & RCC_CSR_RMVF_Msk;
    StartupResetType cause = STARTUP_RESET_UNKNOWN;

    if (resetCause & RCC_CSR_PINRSTF)
    {
        cause = STARTUP_RESET_PIN;       /* 外部复位引脚 */
    }
    else if (resetCause & RCC_CSR_PORRSTF)
    {
        cause = STARTUP_RESET_POWER_ON;  /* 上电复位 */
    }
    else if (resetCause & RCC_CSR_SFTRSTF)
    {
        cause = STARTUP_RESET_SOFTWARE;  /* 软件复位 */
    }
    else if (resetCause & RCC_CSR_WDGRSTF)
    {
        cause = STARTUP_RESET_WATCHDOG;  /* 看门狗复位 */
    }
    else if (resetCause & RCC_CSR_LPWRRSTF)
    {
        cause = STARTUP_RESET_LOW_POWER; /* 低功耗管理复位 */
    }

    /* 清除复位标志 */
    RCC->CSR |= RCC_CSR_RMVF;

    return cause;
}

/**
 * 根据复位原因选择启动模式
 */
StartupModeType Startup_SelectMode(StartupResetType resetCause)
{
    switch (resetCause)
    {
        case STARTUP_RESET_POWER_ON:
            /* 上电复位 — 完整启动 */
            return STARTUP_MODE_FULL;

        case STARTUP_RESET_PIN:
            /* 外部复位 — 完整启动 */
            return STARTUP_MODE_FULL;

        case STARTUP_RESET_SOFTWARE:
            /* 软件复位 — 快速启动 */
            return STARTUP_MODE_FAST;

        case STARTUP_RESET_WATCHDOG:
            /* 看门狗复位 — 安全启动，需检查故障 */
            return STARTUP_MODE_SAFETY;

        case STARTUP_RESET_LOW_POWER:
            /* 低功耗唤醒 — 快速恢复 */
            return STARTUP_MODE_QUICK_WAKEUP;

        default:
            return STARTUP_MODE_FULL;
    }
}
```

---

## 总结

### 核心要点

1. **AUTOSAR 启动是严格分层的**：硬件 → EcuM → OS → BSW → RTE → SWC，各层之间有严格的依赖关系。

2. **EcuM 是启动总指挥**：负责协调所有模块的初始化顺序，管理启动/关闭/休眠状态机。

3. **OS 是分水岭**：低级 BSW 在 OS 启动前初始化，中级和高级 BSW 在 OS 启动后，利用多任务环境。

4. **两阶段启动**：StartupOne（最小启动，进休眠）和 StartupTwo（完整启动，进运行），根据唤醒源选择。

5. **SchM 调度表驱动 BSW 模块**：各 BSW 模块的 MainFunction 通过调度表周期性执行，而非各自独立定时。

6. **RTE 是 SWC 的入口**：RTE 初始化后，应用层的 SWC 才开始运行。

### 启动顺序速查表

```
main()
  └─ EcuM_Init()
       ├─ Det_Init()          — 错误检测
       ├─ NvM_Init()          — 非易失存储
       └─ EcuM_StartupTwo()
            └─ StartOS()      — OS 启动
                 └─ EcuM_OnStartupTwoComplete()
                      ├─ BswM_Init()       — 模式管理
                      ├─ ComM_Init()       — 通信管理
                      ├─ PduR_Init()       — PDU 路由
                      ├─ Can_Init()        — CAN 驱动
                      ├─ Com_Init()        — 通信层
                      ├─ NvM_InitBlock()   — 完整 NvM
                      ├─ Dem_Init()        — 诊断
                      ├─ Dcm_Init()        — 诊断通信
                      ├─ SchM_Start()      — 调度表
                      ├─ Rte_Init()        — RTE
                      ├─ Rte_Start()       — SWC 启动
                      └─ 正常运行
```

---

> **参考规范**：
> 
> - AUTOSAR SWS\_EcuM — ECU Manager 规范
> 
> - AUTOSAR SWS\_OS — 操作系统规范
> 
> - AUTOSAR SWS\_BSW\_Scheduler — BSW 调度器规范
> 
> - AUTOSAR SWS\_RTE — 运行时环境规范
> 
> - AUTOSAR SWS\_BswM — BSW 模式管理器规范
> 
> - AUTOSAR SWS\_Com — 通信层规范
> 
> - ISO 26262 — 功能安全标准
> 
> 

> （注：部分内容可能由 AI 生成）

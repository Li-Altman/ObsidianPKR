# FLASH 与 SRAM 的地址映射关系详解

> 理解嵌入式 MCU 的**内存映射**——CPU 如何看到 FLASH 和 SRAM，它们如何在同一个地址空间中共存

---

## 1. 通俗理解：CPU 的"地址簿"

把 CPU 的地址空间想象成一个**大型仓库**，每个地址是一个**货架位置**：

```mermaid
graph TB
    subgraph CPU_ADDR_SPACE["CPU 视角的地址空间"]
        SPACE["0x0000_0000 ~ 0xFFFF_FFFF<br/>32-bit 地址空间 = 4GB<br/>CPU 的"视野范围""]

        subgraph 地址分区["地址分区"]
            CODE["0x0000_0000 ~ 0x000F_FFFF<br/>FLASH 区域<br/>1MB<br/>存放教科书（代码）"]
            DATA["0x1FF8_0000 ~ 0x1FFF_FFFF<br/>SRAM 区域<br/>128KB<br/>存放草稿纸（变量）"]
            PERI["0x4000_0000 ~ 0x400F_FFFF<br/>外设区域<br/>存放工具（寄存器）"]
            OTHER["其他地址<br/>未使用/保留"]
        end
    end

    subgraph 类比["类比"]
        BOOK["FLASH 区域 = 书架<br/>书（代码）写好后长期放在那里"]
        PAPER["SRAM 区域 = 桌面<br/>草稿纸（变量）随手写画"]
        TOOL["外设区域 = 工具箱<br/>随用随取"]
    end

    CODE --- BOOK
    DATA --- PAPER
    PERI --- TOOL

    classDef space fill:#e8eaf6,stroke:#283593,stroke-width:2px
    classDef flash fill:#e3f2fd,stroke:#1565c0,stroke-width:2px
    classDef sram fill:#f3e5f5,stroke:#4a148c,stroke-width:2px
    classDef peri fill:#fff3e0,stroke:#e65100,stroke-width:2px
    classDef other fill:#f5f5f5,stroke:#9e9e9e,stroke-dasharray:5 5
    classDef analog fill:#e8f5e9,stroke:#2e7d32,stroke-width:1px

    class SPACE space
    class CODE,BOOK flash
    class DATA,PAPER sram
    class PERI,TOOL peri
    class OTHER other
```

**图解释：** CPU 通过**统一地址空间**（Unified Address Space）来访问 FLASH 和 SRAM。FLASH 和 SRAM 被映射到不同的地址范围，CPU 通过地址总线的地址来区分访问的是哪个存储器。

**核心思想：** CPU 不关心地址背后的物理设备是什么，它只知道"地址 X 上存着数据"。芯片设计者把 FLASH、SRAM、外设寄存器映射到不同的地址范围，CPU 通过地址总线选择对应的设备。

---

## 2. 地址映射的物理基础

### 2.1 总线架构

```mermaid
graph TB
    subgraph ARCH["总线架构与地址映射"]
        CPU["CPU 核心<br/>Cortex-M/AURIX/..."]

        I_BUS["I-Bus<br/>指令总线"]
        D_BUS["D-Bus<br/>数据总线"]
        S_BUS["S-Bus<br/>系统总线"]

        MATRIX["总线矩阵<br/>地址解码器<br/>负责将地址路由到对应的从设备"]

        FLASH_CTRL["FLASH 控制器<br/>地址范围: 0x0000_0000 ~ 0x000F_FFFF<br/>响应: 代码读取和常量访问"]
        SRAM_CTRL["SRAM 控制器<br/>地址范围: 0x1FF8_0000 ~ 0x1FFF_FFFF<br/>响应: 变量读写"]
        PERI_CTRL["外设总线<br/>地址范围: 0x4000_0000 ~ 0x400F_FFFF<br/>响应: 寄存器读写"]

        CPU --> I_BUS
        CPU --> D_BUS
        CPU --> S_BUS
        I_BUS --> MATRIX
        D_BUS --> MATRIX
        S_BUS --> MATRIX
        MATRIX -->|"地址 0x0008xxxx<br/>命中 FLASH 范围"| FLASH_CTRL
        MATRIX -->|"地址 0x1FFCxxxx<br/>命中 SRAM 范围"| SRAM_CTRL
        MATRIX -->|"地址 0x4004xxxx<br/>命中外设范围"| PERI_CTRL
    end

    classDef core fill:#ffebee,stroke:#c62828,stroke-width:2px
    classDef bus fill:#e8eaf6,stroke:#283593,stroke-width:2px
    classDef matrix fill:#fff3e0,stroke:#e65100,stroke-width:2px
    classDef slave fill:#e8f5e9,stroke:#2e7d32,stroke-width:2px

    class CPU core
    class I_BUS,D_BUS,S_BUS bus
    class MATRIX matrix
    class FLASH_CTRL,SRAM_CTRL,PERI_CTRL slave
```

**图解释：** 总线矩阵是地址映射的核心：
1. CPU 发出地址（如 `0x0008xxxx`）
2. 总线矩阵中的**地址解码器**判断地址落在哪个范围内
3. 将访问路由到对应的从设备（FLASH 控制器 / SRAM 控制器 / 外设总线）
4. 从设备响应数据，返回给 CPU

### 2.2 地址解码原理

```mermaid
graph LR
    subgraph ADDR_DECODE["地址解码过程"]
        ADDR["CPU 发出的地址: 0x1FFC_1234<br/>二进制: 0001 1111 1111 1100 0001 0010 0011 0100"]
        DECODE["地址解码器"]
        RANGE["范围判断:<br/>0x1FF8_0000 ~ 0x1FFF_FFFF<br/>= SRAM 区域"]
        SELECT["选中: SRAM 控制器<br/>片选信号 CS# 有效"]
        RESP["SRAM 控制器<br/>读取地址 0x1FFC_1234<br/>返回数据给 CPU"]

        ADDR --> DECODE
        DECODE --> RANGE
        RANGE --> SELECT
        SELECT --> RESP
    end

    classDef addr fill:#ffebee,stroke:#c62828,stroke-width:2px
    classDef decode fill:#e8eaf6,stroke:#283593,stroke-width:2px
    classDef resp fill:#e8f5e9,stroke:#2e7d32,stroke-width:2px

    class ADDR addr
    class DECODE,RANGE,SELECT decode
    class RESP resp
```

**图解释：** 地址解码器的核心是**范围比较器**：
- CPU 发出的 32 位地址中，高位地址（如 `0x1FFC`）决定了访问哪个设备
- 地址解码器将高位地址与预配置的地址范围比较
- 命中后，对应的设备片选信号（CS#）被激活，设备响应访问

---

## 3. 典型 MCU 的地址映射

### 3.1 S32K148 内存映射

```mermaid
graph TB
    subgraph S32K148 地址空间
        direction TB

        subgraph FLASH_REGION["FLASH 区域"]
            F_BOOT["0x0000_0000 - 0x0000_7FFF<br/>Bootloader (32KB)"]
            F_APP["0x0000_8000 - 0x000F_FFFF<br/>Application Code (992KB)"]
            F_CAL["0x0010_0000 - 0x0010_FFFF<br/>Calibration Constants (64KB)<br/>实际是 PFLASH 高端"]
        end

        subgraph DFLASH_REGION["DFLASH 区域（独立 FLASH）"]
            DF_EE["0x1000_0000 - 0x1000_7FFF<br/>EEPROM Emulation (32KB)"]
            DF_NV["0x1000_8000 - 0x1000_BFFF<br/>NVRAM (16KB)"]
            DF_CFG["0x1000_C000 - 0x1000_FFFF<br/>Configuration (16KB)"]
        end

        subgraph SRAM_REGION["SRAM 区域"]
            S_CACHE["0x1FF8_0000 - 0x1FF8_7FFF<br/>Flash Cache + ECC (32KB)"]
            S_DATA["0x1FF8_8000 - 0x1FFF_FFFF<br/>SRAM Data (96KB)"]
        end

        subgraph PERI_REGION["外设区域"]
            P_GPIO["0x4000_0000 - 0x400F_FFFF<br/>GPIO / PORT / PINS"]
            P_FTFC["0x4002_0000 - 0x4002_FFFF<br/>FTFC (Flash Controller)"]
            P_CAN["0x4004_0000 - 0x4004_FFFF<br/>CAN / FlexCAN"]
            P_ADC["0x400B_0000 - 0x400B_FFFF<br/>ADC"]
        end
    end

    classDef flash fill:#e3f2fd,stroke:#1565c0,stroke-width:2px
    classDef dflash fill:#fff3e0,stroke:#e65100,stroke-width:2px
    classDef sram fill:#f3e5f5,stroke:#4a148c,stroke-width:2px
    classDef peri fill:#e8f5e9,stroke:#2e7d32,stroke-width:2px

    class F_BOOT,F_APP,F_CAL flash
    class DF_EE,DF_NV,DF_CFG dflash
    class S_CACHE,S_DATA sram
    class P_GPIO,P_FTFC,P_CAN,P_ADC peri
```

**图解释：** S32K148 的地址映射展示了典型的嵌入式 MCU 布局：
- **0x0000_0000 ~ 0x0010_FFFF**: PFLASH（代码 + 常量）
- **0x1000_0000 ~ 0x1000_FFFF**: DFLASH（数据）
- **0x1FF8_0000 ~ 0x1FFF_FFFF**: SRAM（变量 + 堆栈 + Cache）
- **0x4000_0000 ~ 0x400F_FFFF**: 外设寄存器

### 3.2 STM32F103C8T6 内存映射

```mermaid
graph TB
    subgraph STM32F103 地址空间
        direction TB

        subgraph CODE_REGION["Code 区域"]
            C_FLASH["0x0800_0000 - 0x0800_FFFF<br/>Main Flash (64KB)<br/>代码 + 数据共用"]
            C_SYSTEM["0x1FFF_F000 - 0x1FFF_F7FF<br/>System Memory (2KB)<br/>Bootloader"]
        end

        subgraph SRAM_REGION["SRAM 区域"]
            S_RAM["0x2000_0000 - 0x2000_4FFF<br/>SRAM (20KB)<br/>变量 + 堆栈 + 堆"]
        end

        subgraph PERI_REGION["外设区域"]
            P_APB1["0x4000_0000 - 0x4000_FFFF<br/>APB1 外设<br/>（TIM / USART / I2C）"]
            P_APB2["0x4001_0000 - 0x4001_FFFF<br/>APB2 外设<br/>（GPIO / ADC / SPI）"]
            P_AHB["0x4002_0000 - 0x4002_3FFF<br/>AHB 外设<br/>（RCC / CRC）"]
        end

        subgraph ALIAS_REGION["别名区域"]
            A_BITBAND["0x2200_0000 - 0x2203_FFFF<br/>SRAM Bit-band 别名<br/>20KB → 1MB 位寻址"]
            A_PERI_BB["0x4200_0000 - 0x420F_FFFF<br/>外设 Bit-band 别名<br/>1MB 外设位寻址"]
        end
    end

    classDef code fill:#e3f2fd,stroke:#1565c0,stroke-width:2px
    classDef sram fill:#f3e5f5,stroke:#4a148c,stroke-width:2px
    classDef peri fill:#e8f5e9,stroke:#2e7d32,stroke-width:2px
    classDef alias fill:#fff3e0,stroke:#e65100,stroke-width:2px

    class C_FLASH,C_SYSTEM code
    class S_RAM sram
    class P_APB1,P_APB2,P_AHB peri
    class A_BITBAND,A_PERI_BB alias
```

**图解释：** STM32F103 的地址映射展示了 Cortex-M3 的典型布局：
- **0x0800_0000**: Flash 映射地址（实际物理 Flash 在 0x0800_0000）
- **0x1FFF_F000**: System Memory（内置 Bootloader）
- **0x2000_0000**: SRAM
- **0x4000_0000**: 外设
- **0x2200_0000**: Bit-band 别名区域（SRAM 位寻址扩展）

### 3.3 三种 MCU 地址映射对比

| MCU | FLASH 地址 | SRAM 地址 | FLASH 大小 | SRAM 大小 | 特殊之处 |
|-----|-----------|-----------|-----------|----------|---------|
| **S32K148** | `0x0000_0000` | `0x1FF8_0000` | 1MB | 128KB | 独立 DFLASH 在 `0x1000_0000` |
| **TC3xx** | `0x8000_0000` | `0x5000_0000` | 16MB | 4MB | 多核独立局部 SRAM，DFLASH 在 `0xAF00_0000` |
| **STM32F103** | `0x0800_0000` | `0x2000_0000` | 64KB | 20KB | 无独立 DFLASH，代码和数据共用 Flash |
| **S32K116** | `0x0000_0000` | `0x1FFF_0000` | 128KB | 16KB | 小容量 MCU，映射结构类似 S32K148 |

---

## 4. 地址映射的硬件实现

### 4.1 物理地址 vs 总线地址

```mermaid
graph TB
    subgraph ARCH_MAP["物理地址 vs 总线地址"]
        FLASH_PHY["FLASH 物理阵列<br/>实际存储单元<br/>地址: 0x0000_0000 ~ 0x000F_FFFF"]
        SRAM_PHY["SRAM 物理阵列<br/>实际存储单元<br/>地址: 0x0000_0000 ~ 0x0001_FFFF"]

        MAP1["FLASH 映射到 CPU 地址空间<br/>0x0000_0000 ~ 0x000F_FFFF<br/>（1:1 映射）"]
        MAP2["SRAM 映射到 CPU 地址空间<br/>0x1FF8_0000 ~ 0x1FFF_FFFF<br/>（偏移映射）"]

        CPU_ADDR1["读 FLASH: 地址 0x0008_1234<br/>→ 总线矩阵 → FLASH 控制器<br/>→ FLASH 物理地址 0x0008_1234"]
        CPU_ADDR2["读 SRAM: 地址 0x1FFC_5678<br/>→ 总线矩阵 → SRAM 控制器<br/>→ SRAM 物理地址 0x0004_5678"]

        FLASH_PHY --> MAP1
        SRAM_PHY --> MAP2
        MAP1 --> CPU_ADDR1
        MAP2 --> CPU_ADDR2
    end

    classDef phy fill:#e8f5e9,stroke:#2e7d32,stroke-width:2px
    classDef map fill:#fff3e0,stroke:#e65100,stroke-width:2px
    classDef cpu fill:#e3f2fd,stroke:#1565c0,stroke-width:2px

    class FLASH_PHY,SRAM_PHY phy
    class MAP1,MAP2 map
    class CPU_ADDR1,CPU_ADDR2 cpu
```

**图解释：** 物理地址和总线地址不一定相同：
- **FLASH** 通常 1:1 映射：FLASH 物理地址 0x0008_1234 = CPU 地址 0x0008_1234
- **SRAM** 通常偏移映射：SRAM 物理地址 0x0004_5678 → CPU 地址 0x1FFC_5678（偏移 +0x1FF8_0000）
- 地址映射由芯片设计时的**总线矩阵**决定，对软件透明

### 4.2 Cortex-M 的地址映射架构

```mermaid
graph TB
    subgraph CM_ADDR_SPACE["ARM Cortex-M 预定义地址映射"]
        CODE_REGION["0x0000_0000 - 0x1FFF_FFFF<br/>Code 区域 (512MB)<br/>代码 + 异常向量 + 常量"]
        SRAM_REGION["0x2000_0000 - 0x3FFF_FFFF<br/>SRAM 区域 (512MB)<br/>变量 + 堆栈 + 堆"]
        PERI_REGION["0x4000_0000 - 0x5FFF_FFFF<br/>外设区域 (512MB)<br/>所有外设寄存器"]
        RAM2_REGION["0x6000_0000 - 0x9FFF_FFFF<br/>外部 RAM 区域 (1GB)<br/>外部存储器"]
        DEVICE_REGION["0xA000_0000 - 0xDFFF_FFFF<br/>外部设备区域 (1GB)<br/>外部设备"]
        SYS_REGION["0xE000_0000 - 0xFFFF_FFFF<br/>系统区域 (512MB)<br/>PPB 系统寄存器"]
    end

    CODE_REGION --> FLASH_EXAMPLE["芯片厂商决定 FLASH 的具体位置<br/>STM32: 0x0800_0000<br/>S32K: 0x0000_0000<br/>Kinetis: 0x0000_0000"]
    SRAM_REGION --> SRAM_EXAMPLE["芯片厂商决定 SRAM 的具体位置<br/>STM32: 0x2000_0000<br/>S32K: 0x1FF8_0000<br/>Kinetis: 0x1FFF_0000"]

    classDef cortex fill:#e8eaf6,stroke:#283593,stroke-width:2px
    classDef example fill:#f5f5f5,stroke:#9e9e9e,stroke-dasharray:5 5

    class CODE_REGION,SRAM_REGION,PERI_REGION,RAM2_REGION,DEVICE_REGION,SYS_REGION cortex
    class FLASH_EXAMPLE,SRAM_EXAMPLE example
```

**图解释：** ARM Cortex-M 规定了**粗粒度的地址空间分区**：
- **0x0000_0000 ~ 0x1FFF_FFFF**: Code 区域（512MB），用于代码和异常向量
- **0x2000_0000 ~ 0x3FFF_FFFF**: SRAM 区域（512MB），用于变量
- **0x4000_0000 ~ 0x5FFF_FFFF**: 外设区域（512MB）
- **0xE000_0000 ~ 0xFFFF_FFFF**: 系统区域（系统寄存器、NVIC、MPU 等）

**芯片厂商**可以在这些大区域内自由选择 FLASH 和 SRAM 的具体映射地址。这就是为什么不同 MCU 的 FLASH 地址不同（STM32 在 0x0800_0000，S32K 在 0x0000_0000）。

---

## 5. 链接脚本中的地址映射

### 5.1 链接脚本如何定义映射

```ld
/* ============================================
 * 链接脚本中的地址映射定义
 * 链接器根据这些地址生成最终的可执行文件
 * ============================================ */

MEMORY
{
    /* FLASH 区域: 物理地址 0x0000_0000, 1MB */
    /* 所有代码和常量放在这里 */
    FLASH (rx)  : ORIGIN = 0x00000000, LENGTH = 0x00100000

    /* DFLASH 区域: 物理地址 0x1000_0000, 64KB */
    /* 数据存储（FEE 模拟 EEPROM）*/
    DFLASH (rw) : ORIGIN = 0x10000000, LENGTH = 0x00010000

    /* SRAM 区域: 物理地址 0x1FF80000, 128KB */
    /* 所有变量、堆栈、堆放在这里 */
    SRAM (rw)   : ORIGIN = 0x1FF80000, LENGTH = 0x00040000
}

SECTIONS
{
    /* ---- 放在 FLASH 中的段 ---- */

    /* 中断向量表: 必须从 FLASH 起始地址开始 */
    .vectors : {
        __vector_table = .;
        KEEP(*(.vectors))           /* 中断向量表 */
        . = ALIGN(4);
    } > FLASH

    /* 程序代码 */
    .text : {
        . = ALIGN(4);
        *(.text)                    /* 所有 .text 段 */
        *(.text*)                   /* 所有 .text.* 段 */
        *(.glue_7)
        *(.glue_7t)
        . = ALIGN(4);
    } > FLASH

    /* 只读常量（字符串、查表数据）*/
    .rodata : {
        . = ALIGN(4);
        *(.rodata)                  /* 所有 .rodata 段 */
        *(.rodata*)
        . = ALIGN(4);
    } > FLASH

    /* ---- 放在 SRAM 中的段 ---- */

    /* 初始化数据（.data 段）*/
    /* 初始值在 FLASH 中，启动时复制到 SRAM */
    .data : {
        . = ALIGN(4);
        __data_start = .;
        *(.data)                    /* 所有 .data 段 */
        *(.data*)
        . = ALIGN(4);
        __data_end = .;
    } > SRAM AT > FLASH             /* 加载地址在 FLASH，运行时在 SRAM */

    /* 未初始化数据（.bss 段）*/
    /* 启动时在 SRAM 中清零 */
    .bss : {
        . = ALIGN(4);
        __bss_start = .;
        *(.bss)                     /* 所有 .bss 段 */
        *(.bss*)
        *(COMMON)
        . = ALIGN(4);
        __bss_end = .;
    } > SRAM

    /* 堆栈 */
    .stack : {
        . = ALIGN(8);
        __stack_end = .;
        . = . + 0x4000;             /* 16KB 堆栈 */
        . = ALIGN(8);
        __stack_start = .;
    } > SRAM

    /* 堆 */
    .heap : {
        . = ALIGN(8);
        __heap_start = .;
        . = . + 0x2000;             /* 8KB 堆 */
        . = ALIGN(8);
        __heap_end = .;
    } > SRAM

    /* ---- 放在 DFLASH 中的段 ---- */

    /* NVRAM 数据（FEE 管理）*/
    .nvram (NOLOAD) : {
        *(.nvram)
    } > DFLASH
}
```

**代码解释：** 链接脚本中的 `MEMORY` 命令定义了 FLASH 和 SRAM 的地址范围，`SECTIONS` 命令决定了每个段放在哪个存储器中：
- `.vectors`、`.text`、`.rodata` → **FLASH**（0x0000_0000）
- `.data`、`.bss`、`.stack`、`.heap` → **SRAM**（0x1FF8_0000）
- `.nvram` → **DFLASH**（0x1000_0000）

### 5.2 变量地址与存储器的对应

```c
/* ============================================
 * 变量定义 → 链接器分配 → 存储位置
 * ============================================ */

/* ---- 以下变量在 SRAM 中 ---- */

/* 全局变量（初始化）→ .data 段 → SRAM */
uint32_t systemTick = 0;          /* 地址: 0x1FF8_8000（SRAM 中）*/
CanNm_ChannelType canNmChannel;   /* 地址: 0x1FF8_8010（SRAM 中）*/

/* 全局变量（未初始化）→ .bss 段 → SRAM */
static uint8_t rxBuffer[256];     /* 地址: 0x1FF9_0000（SRAM 中）*/
volatile uint32_t irqFlags;       /* 地址: 0x1FF9_0100（SRAM 中）*/

/* 局部变量 → 栈 → SRAM */
void SomeFunction(void) {
    uint32_t temp = 0;            /* 地址: 栈指针 - 4（SRAM 中）*/
    uint8_t  data[64];            /* 地址: 栈指针 - 64（SRAM 中）*/
}

/* ---- 以下变量在 FLASH 中 ---- */

/* 常量 → .rodata 段 → FLASH */
const uint8_t canNmCbvTable[256] = {
    /* 地址: 0x0001_0000（FLASH 中）*/
    /* 掉电不丢失 */
};

/* 字符串常量 → .rodata 段 → FLASH */
const char* versionStr = "V1.2.3";
/* 字符串 "V1.2.3" 在 FLASH 中（0x0001_1000）*/
/* 指针变量 versionStr 在 SRAM 中（0x1FF8_9000）*/

/* 函数代码 → .text 段 → FLASH */
void CanNm_MainFunction(void) {
    /* 函数代码在 FLASH 中（0x0000_8000）*/
    /* 函数内的局部变量在 SRAM 栈中 */
}
```

---

## 6. CPU 如何访问 FLASH 和 SRAM

### 6.1 取指令（Instruction Fetch）

```mermaid
sequenceDiagram
    participant CPU as CPU Core
    participant BIU as 总线接口单元
    participant MATRIX as 总线矩阵
    participant FLASH as FLASH 控制器
    participant SRAM as SRAM 控制器
    participant CACHE as FLASH Cache

    Note over CPU,SRAM: 场景: CPU 执行代码（取指令）

    CPU->>CPU: PC = 0x0000_8000（指向 FLASH 中的函数）

    CPU->>BIU: 取指令请求 (地址: 0x0000_8000)

    BIU->>MATRIX: 地址 0x0000_8000
    Note over MATRIX: 地址解码: 0x0000_8000<br/>命中 FLASH 区域

    MATRIX->>CACHE: 是否命中 Cache？

    alt Cache 命中
        CACHE-->>MATRIX: 返回缓存指令
        Note over CACHE: 命中延迟: 1~2 周期
    else Cache 未命中
        MATRIX->>FLASH: 读 FLASH 请求 (地址: 0x0000_8000)

        FLASH->>FLASH: 内存映射读取
        Note over FLASH: FLASH 读延迟: 2~8 周期<br/>（取决于频率和等待状态）

        FLASH-->>MATRIX: 返回指令数据
        MATRIX->>CACHE: 写入 Cache 行
    end

    MATRIX-->>BIU: 返回指令
    BIU-->>CPU: 指令数据

    Note over CPU: CPU 执行指令
    Note over CPU,SRAM: 取指令总是从 FLASH 或 Cache 中读取<br/>SRAM 不参与取指令过程
```

**图解释：** CPU 取指令的路径：
1. CPU 的 PC 指向 FLASH 地址（0x0000_8000）
2. 总线矩阵将地址路由到 FLASH 控制器
3. 如果有 Cache，优先从 Cache 读取
4. FLASH 的读取延迟通常为 2~8 个 CPU 周期（取决于频率和等待状态）
5. **SRAM 不参与取指令**（除非代码在 RAM 中执行）

### 6.2 数据访问（Data Access）

```mermaid
sequenceDiagram
    participant CPU as CPU Core
    participant BIU as 总线接口单元
    participant MATRIX as 总线矩阵
    participant FLASH as FLASH 控制器
    participant SRAM as SRAM 控制器

    Note over CPU,SRAM: 场景 1: 访问全局变量（读/写 SRAM）

    CPU->>CPU: 执行: systemTick++

    CPU->>BIU: 读内存请求 (地址: 0x1FF8_8000)
    BIU->>MATRIX: 地址 0x1FF8_8000
    Note over MATRIX: 地址解码: 0x1FF8_8000<br/>命中 SRAM 区域

    MATRIX->>SRAM: 读 SRAM 请求
    SRAM-->>MATRIX: 返回 systemTick 值
    MATRIX-->>BIU: 返回数据
    BIU-->>CPU: systemTick = 0

    CPU->>CPU: systemTick = 1

    CPU->>BIU: 写内存请求 (地址: 0x1FF8_8000)
    BIU->>MATRIX: 地址 0x1FF8_8000
    MATRIX->>SRAM: 写 SRAM 请求 (数据: 1)
    Note over SRAM: SRAM 写延迟: 1 周期
    SRAM-->>MATRIX: 写入完成

    Note over CPU,SRAM: 场景 2: 访问常量（读 FLASH）

    CPU->>CPU: 执行: val = canNmCbvTable[0x42]

    CPU->>BIU: 读内存请求 (地址: 0x0001_0042)
    BIU->>MATRIX: 地址 0x0001_0042
    Note over MATRIX: 地址解码: 0x0001_0042<br/>命中 FLASH 区域

    MATRIX->>FLASH: 读 FLASH 请求
    FLASH-->>MATRIX: 返回常量数据
    MATRIX-->>BIU: 返回数据
    BIU-->>CPU: val = 查表结果
```

**图解释：** CPU 的数据访问路径取决于地址：
- **访问 SRAM 地址**（0x1FF8_xxxx）：经过总线矩阵 → SRAM 控制器，延迟 1 个周期
- **访问 FLASH 地址**（0x0000_xxxx）：经过总线矩阵 → FLASH 控制器，延迟 2~8 个周期
- CPU 通过**地址值**自动区分访问哪个存储器，无需软件干预

### 6.3 FLASH 和 SRAM 的访问延迟对比

```mermaid
graph TB
    subgraph FLASH_READ_LATENCY["FLASH 读延迟（带 Cache）"]
        F1["CPU 频率: 120MHz<br/>周期时间: 8.33ns"]
        F2["FLASH 读延迟: 3 周期<br/>= 25ns"]
        F3["Cache 命中: 1 周期<br/>= 8.33ns"]
        F4["带 Cache 的平均延迟:<br/>命中率 90% → 1.2 周期"]
    end

    subgraph SRAM_READ_LATENCY["SRAM 读延迟"]
        S1["CPU 频率: 120MHz<br/>周期时间: 8.33ns"]
        S2["SRAM 读延迟: 1 周期<br/>= 8.33ns"]
        S3["SRAM 写延迟: 1 周期<br/>= 8.33ns"]
        S4["SRAM 零等待<br/>与 CPU 同频"]
    end

    classDef flash fill:#e3f2fd,stroke:#1565c0,stroke-width:2px
    classDef sram fill:#f3e5f5,stroke:#4a148c,stroke-width:2px

    class F1,F2,F3,F4 flash
    class S1,S2,S3,S4 sram
```

**图解释：** FLASH 和 SRAM 的访问延迟差异：
- **SRAM**：零等待（1 周期），与 CPU 同频
- **FLASH**：2~8 周期，需要 Flash Cache 来弥补速度差距
- 带 Cache 的 FLASH 平均延迟可以接近 SRAM（命中率 90%+）

---

## 7. FLASH 和 SRAM 地址映射的"对齐"要求

### 7.1 访问对齐

```c
/* ============================================
 * FLASH 和 SRAM 的访问对齐要求
 * ============================================ */

/* ---- SRAM 访问（对齐宽松）---- */
/* SRAM 支持任意对齐的字节/半字/字访问 */
uint8_t  byte;     /* 任意地址 */
uint16_t half;     /* 任意地址（但推荐 2 字节对齐）*/
uint32_t word;     /* 任意地址（但推荐 4 字节对齐）*/

/* SRAM 中的非对齐访问（支持但性能下降）*/
uint32_t* ptr = (uint32_t*)0x1FF8_8001;  /* 非对齐地址 */
uint32_t val = *ptr;  /* CPU 内部拆分为两次访问，效率低 */

/* ---- FLASH 访问（对齐严格）---- */
/* FLASH 通常要求固定对齐访问 */
/* 32-bit 总线宽度: 必须 4 字节对齐 */
/* 64-bit 总线宽度: 必须 8 字节对齐 */

/* FLASH 中的非对齐访问（可能导致 fault）*/
const uint32_t* flashPtr = (const uint32_t*)0x0000_8001;  /* 非对齐地址 */
uint32_t val = *flashPtr;
/* 可能触发: 
 * 1. 硬件 fault（取决于 MCU 配置）
 * 2. 或拆分为两次访问（效率低）
 * 3. 或返回错误数据
 */

/* 正确的做法：始终对齐访问 FLASH */
const uint32_t* flashPtr = (const uint32_t*)0x0000_8000;  /* 对齐地址 */
uint32_t val = *flashPtr;  /* 正常访问 */
```

### 7.2 地址空间的"空洞"和"别名"

```mermaid
graph TB
    subgraph 地址别名["地址别名（Aliasing）"]
        ALIAS1["物理 FLASH 0x0000_0000<br/>可以通过多个地址访问"]
        ALIAS2["别名 1: 0x0000_0000（直接映射）"]
        ALIAS3["别名 2: 0x0100_0000（某些 MCU 的别名区域）"]
        ALIAS4["别名 3: 0x0200_0000（某些 MCU 的别名区域）"]
    end

    subgraph 地址空洞["地址空洞（Hole）"]
        HOLE1["0x0010_0000 ~ 0x0FFF_FFFF<br/>FLASH 结束 ~ DFLASH 开始<br/>未使用区域"]
        HOLE2["0x1001_0000 ~ 0x1FF7_FFFF<br/>DFLASH 结束 ~ SRAM 开始<br/>未使用区域"]
        HOLE3["访问空洞区域 → 触发 BusFault"]
    end

    ALIAS1 --> ALIAS2
    ALIAS1 --> ALIAS3
    ALIAS1 --> ALIAS4

    classDef alias fill:#fff3e0,stroke:#e65100,stroke-width:2px
    classDef hole fill:#ffebee,stroke:#c62828,stroke-width:2px

    class ALIAS1,ALIAS2,ALIAS3,ALIAS4 alias
    class HOLE1,HOLE2,HOLE3 hole
```

**图解释：** 地址映射中的两个特殊现象：
- **地址别名**：同一个物理 FLASH 可以通过多个地址范围访问（某些 MCU 的特性）
- **地址空洞**：FLASH 和 SRAM 之间的地址范围没有映射设备，访问会触发 BusFault

---

## 8. FLASH 和 SRAM 的地址映射在启动过程中的作用

### 8.1 启动时地址映射的切换

```mermaid
sequenceDiagram
    participant CPU as CPU Core
    participant FLASH as FLASH (0x0000_0000)
    participant SRAM as SRAM (0x1FF8_0000)
    participant BOOT as Boot ROM (0xFFFF_0000)

    Note over CPU,BOOT: 上电复位

    CPU->>CPU: 读取复位向量
    
    alt 正常启动模式
        Note over CPU: 从 FLASH 的 0x0000_0000 读取<br/>SP = [0x0000_0000]<br/>PC = [0x0000_0004]
        CPU->>FLASH: 读取向量表
        FLASH-->>CPU: SP = 0x1FFC_4000, PC = 0x0000_8000

        CPU->>FLASH: 执行 Startup Code（在 FLASH 中）
        FLASH-->>CPU: 指令

        CPU->>SRAM: 复制 .data 段（FLASH → SRAM）
        CPU->>SRAM: 清零 .bss 段
        CPU->>SRAM: 初始化堆栈指针

        CPU->>FLASH: 跳转到 main()（在 FLASH 中）
        Note over CPU,SRAM: 正常运行

    else 启动加载模式
        Note over CPU: 从 Boot ROM 启动<br/>地址映射被重映射
        CPU->>BOOT: 读取向量表
        BOOT-->>CPU: SP, PC

        Note over CPU: Boot ROM 中的代码<br/>可以重新映射 FLASH 地址

        BOOT->>BOOT: 执行 Bootloader
        BOOT->>FLASH: 擦除/写入 FLASH（固件升级）
    end
```

**图解释：** 启动过程中地址映射的作用：
- 复位后 CPU 从 **0x0000_0000** 读取向量表（SP 和 PC）
- Startup Code 在 FLASH 中执行，负责初始化 SRAM
- 某些 MCU 支持**地址重映射**（Remap），可以改变 FLASH 和 Boot ROM 的映射关系

### 8.2 地址重映射（Remap）

```c
/* ============================================
 * 地址重映射 - 改变 FLASH/SRAM 的映射关系
 * ============================================ */

/* 某些 MCU 支持地址重映射 */
/* 例如: STM32F1 的 BOOT0/BOOT1 引脚决定启动地址 */

/* BOOT0=0, BOOT1=0: 从 FLASH 启动 (0x0800_0000 → 0x0000_0000) */
/* BOOT0=1, BOOT1=0: 从 System Memory 启动 (0x1FFF_F000 → 0x0000_0000) */
/* BOOT0=1, BOOT1=1: 从 SRAM 启动 (0x2000_0000 → 0x0000_0000) */

/* 内存映射寄存器（以 STM32F1 为例）*/
#define SYSCFG_MEMRMP  (*((volatile uint32_t*)0x4001_0000))
/* MEMRMP[1:0]:
 * 00 = FLASH mapped at 0x0000_0000
 * 01 = System memory mapped at 0x0000_0000
 * 10 = SRAM mapped at 0x0000_0000
 * 11 = reserved
 */

/* 运行时重映射到 SRAM（将代码复制到 SRAM 后执行）*/
void RemapToSRAM(void) {
    /* 1. 将代码从 FLASH 复制到 SRAM */
    memcpy((void*)0x2000_0000, (void*)0x0800_0000, CODE_SIZE);

    /* 2. 设置向量表偏移为 SRAM 地址 */
    SCB->VTOR = 0x2000_0000;

    /* 3. 重映射: SRAM 映射到 0x0000_0000 */
    SYSCFG_MEMRMP = 0x02;  /* SRAM at 0x0000_0000 */

    /* 4. 跳转到 SRAM 中的代码执行 */
    /* 此后所有中断向量从 SRAM 读取 */
    /* 代码在 SRAM 中执行（速度更快，但掉电丢失）*/
}
```

**代码解释：** 地址重映射允许将 SRAM 映射到地址 0x0000_0000：
- 将代码从 FLASH 复制到 SRAM
- 通过重映射寄存器，让 CPU 认为 SRAM 就是起始地址
- 这样可以实现**代码在 RAM 中执行**（速度更快，但掉电丢失）
- 常用于 Bootloader 或需要高速执行的关键代码

---

## 9. 地址映射相关的性能优化

### 9.1 FLASH 等待状态（Wait State）

```c
/* ============================================
 * FLASH 等待状态配置
 * ============================================ */

/* FLASH 的读速度比 CPU 慢，需要插入等待状态 */
/* 等待状态数取决于 CPU 频率和 FLASH 速度 */

/* FLASH 等待状态配置寄存器（以 S32K148 为例）*/
#define FLASH_RDCFG  (*((volatile uint32_t*)0x40020004))

/* 等待状态配置:
 * CPU 频率  | 等待状态 | FLASH 读延迟
 * 0~40MHz   | 0 周期  | 12.5ns
 * 40~80MHz  | 1 周期  | 25ns
 * 80~120MHz | 2 周期  | 37.5ns
 */

void Fls_ConfigureWaitState(uint32_t cpuFreqHz) {
    uint32_t ws;

    if (cpuFreqHz <= 40000000) {
        ws = 0;  /* 零等待 */
    } else if (cpuFreqHz <= 80000000) {
        ws = 1;  /* 1 个等待周期 */
    } else {
        ws = 2;  /* 2 个等待周期 */
    }

    FLASH_RDCFG = (FLASH_RDCFG & ~0x03) | ws;
}

/* ============================================
 * FLASH Cache 配置
 * ============================================ */

/* FLASH Cache 可以显著减少平均访问延迟 */
/* Cache 命中时: 1 周期（与 SRAM 相同）*/
/* Cache 未命中: 等待状态 + 读取时间 */

/* 启用 FLASH Cache */
void Fls_EnableCache(void) {
    /* 以 S32K148 为例 */
    FLASH_CACHECR = 0x01;  /* 启用 Cache */
    FLASH_CACHECR |= 0x02; /* 启用预取（Prefetch）*/
}

/* Cache 命中率估算 */
/* 代码顺序执行: 命中率 > 90% */
/* 跳转/中断: 命中率降低 */
/* 大循环: 首次执行未命中，之后命中率 100% */
```

### 9.2 FLASH 和 SRAM 的地址映射对性能的影响

```mermaid
graph TB
    subgraph 性能影响因素
        F1["FLASH 读延迟: 2~8 周期<br/>需要 Cache 和预取弥补"]
        F2["SRAM 读延迟: 1 周期<br/>零等待，与 CPU 同频"]
        F3["代码在 FLASH 中执行: 正常"]
        F4["代码在 SRAM 中执行: 更快<br/>但需要复制 + 占用 SRAM 空间"]
        F5["常量在 FLASH 中: 读延迟 2~8 周期"]
        F6["变量在 SRAM 中: 读延迟 1 周期"]
    end

    subgraph 优化策略
        O1["启用 FLASH Cache + 预取"]
        O2["关键代码放到 SRAM 执行"]
        O3["频繁访问的常量复制到 SRAM"]
        O4["合理安排代码布局，减少 Cache miss"]
    end

    F1 --> O1
    F4 --> O2
    F5 --> O3
    F1 --> O4

    classDef factor fill:#e3f2fd,stroke:#1565c0,stroke-width:2px
    classDef opt fill:#e8f5e9,stroke:#2e7d32,stroke-width:2px

    class F1,F2,F3,F4,F5,F6 factor
    class O1,O2,O3,O4 opt
```

---

## 10. 常见问题

### Q1: 为什么 FLASH 的起始地址通常是 0x0000_0000？

**A:** 复位后 CPU 从 0x0000_0000 读取向量表（SP 和 PC）。将 FLASH 映射到 0x0000_0000 可以确保：
1. 上电后 CPU 能直接读取代码
2. 不需要额外的 Boot ROM 跳转
3. ARM Cortex-M 的架构设计如此

STM32 的 FLASH 在 0x0800_0000，但通过别名映射到 0x0000_0000，所以 CPU 仍然能从 0x0000_0000 读取向量表。

### Q2: 为什么 SRAM 和 FLASH 的地址不连续？

**A:** 这是芯片设计的有意安排：
1. ARM Cortex-M 把地址空间预分区（Code / SRAM / Peripheral / System）
2. FLASH 在 Code 区，SRAM 在 SRAM 区
3. 不连续的地址空间便于地址解码器快速判断访问目标
4. 预留的地址空间可以用于扩展存储器或别名映射

### Q3: 访问 FLASH 和 SRAM 的速度差多少？

**A:** 差别很大：

| 操作 | FLASH | SRAM | 比率 |
|------|-------|------|------|
| 读延迟 | 2~8 周期 | 1 周期 | 2~8x 慢 |
| 写延迟 | 10~100μs（需擦除） | 1 周期 | 10,000x 以上 |
| 带 Cache 读 | 1~2 周期 | 1 周期 | 接近 |

### Q4: 代码可以放在 SRAM 中执行吗？

**A:** 可以，但需要特殊处理：
1. 将代码从 FLASH 复制到 SRAM
2. 设置向量表偏移（VTOR）指向 SRAM
3. 进行地址重映射（如果需要）
4. 跳转到 SRAM 中的代码执行

**优点**：执行速度更快（零等待），不受 FLASH 等待状态影响
**缺点**：占用 SRAM 空间，掉电丢失，启动时需要复制

### Q5: 地址空洞（Hole）访问会发生什么？

**A:** 访问未映射的地址区域会触发：
1. **BusFault**（Cortex-M）：进入 HardFault 处理
2. **异常返回**：取决于 MCU 的错误处理机制
3. 系统中的地址空洞是设计预留的，**软件不应访问**

---

## 11. 总结

```mermaid
graph TB
    subgraph 核心要点
        P1["FLASH 和 SRAM 在同一个地址空间中<br/>CPU 通过地址值区分访问哪个存储器"]
        P2["地址映射由芯片设计决定<br/>总线矩阵中的地址解码器实现路由"]
        P3["FLASH 通常从 0x0000_0000 开始<br/>SRAM 从 0x1FFF_xxxx 或 0x2000_0000 开始"]
        P4["FLASH 读延迟 2~8 周期（需 Cache）<br/>SRAM 读延迟 1 周期（零等待）"]
        P5["链接脚本定义变量和函数的存储位置<br/>编译器生成地址，链接器分配地址"]
    end

    subgraph 地址映射的作用
        R1["决定了系统的启动行为"]
        R2["决定了代码和数据的访问性能"]
        R3["决定了存储器的利用率"]
        R4["决定了系统的可扩展性"]
    end

    P1 --> R1
    P2 --> R2
    P3 --> R3
    P4 --> R4
    P5 --> R1

    classDef point fill:#e3f2fd,stroke:#1565c0,stroke-width:2px
    classDef role fill:#fff3e0,stroke:#e65100,stroke-width:2px

    class P1,P2,P3,P4,P5 point
    class R1,R2,R3,R4 role
```

### 一句话总结

**FLASH 和 SRAM 的地址映射**是芯片设计者为 CPU 定义的一张"地址-设备对应表"：CPU 发出地址 → 总线矩阵解码 → 自动路由到 FLASH 或 SRAM → 读取/写入数据。**整个过程对软件透明**，但理解这个映射关系对于编写链接脚本、优化性能、调试问题至关重要。

| 存储器 | 地址范围（典型） | 访问延迟 | 内容 |
|--------|----------------|----------|------|
| **FLASH** | 0x0000_0000 ~ 0x000F_FFFF | 2~8 周期（带 Cache 1~2） | 代码、常量、向量表 |
| **SRAM** | 0x1FF8_0000 ~ 0x1FFF_FFFF | 1 周期 | 变量、堆栈、堆 |
| **DFLASH** | 0x1000_0000 ~ 0x1000_FFFF | 2~8 周期 | 持久化数据 |
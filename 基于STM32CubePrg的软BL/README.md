# 基于 STM32CubeProgrammer 的软 Bootloader

> **版本**: 1.0.0 | **平台**: STM32F407VETx | **协议**: ST Open Bootloader V3.1（与 CubeProgrammer "Open Bootloader" 模式兼容）

## 目录

- [1. 概述](#1-概述)
- [2. 分区与资源](#2-分区与资源)
- [3. 架构与上电逻辑](#3-架构与上电逻辑)
- [4. CubeMX 前置配置](#4-cubemx-前置配置)
- [5. 快速开始（拷入工程）](#5-快速开始拷入工程)
- [6. 构建与烧录](#6-构建与烧录)
- [7. 固件升级工作流](#7-固件升级工作流)
- [8. 上层 APP 接口（app_boot_interface.h）](#8-上层-app-接口app_boot_interfaceh)
- [9. 与 ROM Bootloader 的区别](#9-与-rom-bootloader-的区别)
- [10. 常见问题](#10-常见问题)
- [11. 设计说明](#11-设计说明)

---

## 1. 概述

这是一个**自包含的软 Bootloader**，烧录在 STM32F407 的 Flash 前 32 KB（0x08000000）。
基于 ST 官方 `stm32-mw-openbl`（BSD 许可）+ 本模块自写的 STM32F4 接口层。

- 上电自动运行，**无需 BOOT0 跳线**
- 用 CubeProgrammer（**Open Bootloader** 模式）通过 USART1 下载/擦除/运行 App
- Bootloader 自带自保护：扇区 0..1 不可被主机擦除/写入（防变砖）
- 全部代码可拷入任意 F407 工程直接编译，无需看本模块的工程

### 核心特性

| 特性 | 说明 |
|------|------|
| **零依赖接口** | USART 用寄存器级代码（无需 LL 驱动）；Flash 用 HAL |
| **自保护** | 扇区 0..1 不可擦/不可写；整体擦除自动跳过 BL 扇区 |
| **协议兼容** | ST Open Bootloader V3.1，115200-8E1，同步字节 0x7F |
| **上电智能跳转** | App 有效→等 1 s 主机后跳 App；无 App→一直等主机 |
| **App 可主动进 BL** | `APP_JumpToBootloader()` 写 RAM 标志 + 软复位 |
| **选项字节可读** | 注册 0x1FFFC000 区，CubeProgrammer 连接时能读 OB |

### 文件

```
基于STM32CubePrg的软BL/
├── README.md                    # 本说明
└── openbl/                      # BL 模块全部源码（整体拷入你的工程）
    ├── main.c                   # BL 主程序（替换工程 main.c 或参考其逻辑）
    ├── STM32F407xx_BOOT.ld      # BL 链接脚本（32 KB @ 0x08000000）
    ├── Core/                    # openbl_core.c/.h  （MW，原样）
    ├── Modules/Mem/             # openbl_mem.c/.h   （MW，原样）
    ├── Modules/USART/           # openbl_usart_cmd.c/.h（MW，原样）
    ├── Inc/                     # 配置：platform.h / openbootloader_conf.h / interfaces_conf.h
    │                            #      app_boot_interface.h（App 侧接口，拷给 App 用）
    └── Interfaces/              # F4 接口层：usart/flash/ram/optionbytes/common/app_openbootloader
```

### 资源占用

```
FLASH: ~17.5 KB  (32 KB 分区，占 53%)
RAM:   ~3.2 KB
```

---

## 2. 分区与资源

| 区域 | 地址 | 大小 |
|---|---|---|
| Bootloader（本模块） | `0x08000000` | 32 KB（扇区 0..1） |
| 用户 App | `0x08008000` | 480 KB（扇区 2..11） |
| 选项字节（只读） | `0x1FFFC000` | 16 B |
| 跳转标志（RAM） | `0x2001FFFC` | 4 B |

---

## 3. 架构与上电逻辑

```
复位
 ├─ ① App 请求进 BL？(RAM 0x2001FFFC == 0xDEADBEEF)  → 是 → 清标志 → 一直等主机
 ├─ ② App 有效？(0x08008000 处 SP∈RAM 且 Reset∈[0x08008000,0x08080000))
 │     ├─ 是 → 等主机 1 s
 │     │      ├─ 有主机 → 进入命令循环
 │     │      └─ 无主机 → 跳转 App(0x08008000)
 │     └─ 否 → 一直等主机（不限时）
 └─ 命令循环: 0x7F 同步 → 0x79 ACK → 逐命令处理 (GET/GET_ID/读/写/擦/GO...)
```

BL 自保护逻辑（`flash_interface.c`）：
- 写地址 < 0x08008000 → 拒绝
- 擦除扇区 0..1 → 跳过；整体擦除 → 只擦扇区 2..11

---

## 4. CubeMX 前置配置

模块**不依赖** CubeMX 生成任何 USART/DMA 代码（USART 由接口层寄存器级自建）。
宿主工程只需：

1. **RCC**：HSE（8 MHz）
2. **SYS**：Serial Wire（SWD）、SysTick
3. 时钟：HSE → PLL → 168 MHz（与 App 一致，见 `main.c` 的 `SystemClock_Config`）
4. 其他外设按 App 需要配置

> 不要使能 USART1/DMA（BL 自己管 PA9/PA10）。
> 主机工程需保留 CubeMX 生成的 `stm32f4xx_it.c`（含 `SysTick_Handler → HAL_IncTick`，
> BL 的 1 s 等待依赖 `HAL_GetTick`）和 `stm32f4xx_hal_msp.c`（`HAL_MspInit`）。

---

## 5. 快速开始（拷入工程）

1. **拷贝**：把 `openbl/` 整个文件夹拷入工程（建议放 `Core/` 同级，或按你的 Lib 规范放 `Core/Lib/`）。
2. **main.c**：用 `openbl/main.c` 替换 CubeMX 生成的 `Core/Src/main.c`（BL 代码都在 `USER CODE` 段内，CubeMX 再生成不丢）。
3. **编译设置**（CMake 示例）：

   ```cmake
   # 源文件
   openbl/Core/openbl_core.c
   openbl/Modules/Mem/openbl_mem.c
   openbl/Modules/USART/openbl_usart_cmd.c
   openbl/Interfaces/usart_interface.c
   openbl/Interfaces/flash_interface.c
   openbl/Interfaces/ram_interface.c
   openbl/Interfaces/optionbytes_interface.c
   openbl/Interfaces/common_interface.c
   openbl/Interfaces/app_openbootloader.c

   # 头文件路径
   openbl/Inc  openbl/Interfaces  openbl/Core  openbl/Modules/Mem  openbl/Modules/USART

   # 宏定义
   USE_HAL_DRIVER  STM32F407xx

   # HAL 模块（本机 STM32Cube FW_F4 即可）
   hal  hal_cortex  hal_rcc  hal_rcc_ex  hal_gpio  hal_flash  hal_flash_ex
   hal_flash_ramfunc  hal_pwr  hal_pwr_ex
   ```

4. **链接脚本**：用 `openbl/STM32F407xx_BOOT.ld` 替换工程的 `STM32F407xx_FLASH.ld`
   （FLASH: ORIGIN=0x08000000, LENGTH=32K；也可在 CubeMX Linker Settings 里设）。
5. **构建烧录**：编译出 `.elf` → SWD 烧到 0x08000000（一次）。

---

## 6. 构建与烧录

```bash
# 参考工程（STM32CubeIDE 或 CMake）
cmake --preset Debug && cmake --build --preset Debug
# 产物 build/Debug/STM32_OpenBootLoader.elf (~17.5 KB)

# 转换 bin（可选）
arm-none-eabi-objcopy -O binary xxx.elf bootloader.bin

# SWD 烧录（首次，一次即可）
STM32_Programmer_CLI -c port=SWD mode=UR -w bootloader.bin 0x08000000 -v
```

---

## 7. 固件升级工作流

**硬件**：USB-TTL → PA9(RX)、PA10(TX)、GND 共地，**115200-8E1**。

**方式 A：App 运行中升级（推荐）**

1. App 串口发 `boot` → `APP_JumpToBootloader()` → 软复位
2. BL 检测到跳转标志 → 停在"等主机"
3. CubeProgrammer：UART → COM → 115200 → **Open Bootloader** → Connect
4. 握手：发 `0x7F` → 回 `0x79`，读 ID `0x413`，读选项字节成功
5. 加载 App `.bin`（起始地址 `0x08008000`）→ Download（自动擦扇区 2..11）
6. Disconnect → 复位 → BL 自动跳 App

**方式 B：App 无法运行 / 首次**

1. CubeProgrammer 连接（同上）
2. 若 App 已存在：断电 → 点 Connect → 上电，1 s 内完成握手
3. 无 App 时 BL 一直等，不限时

**固件文件**：`.elf` 转 `.bin`：

```bash
arm-none-eabi-objcopy -O binary xxx.elf xxx.bin
```

---

## 8. 上层 APP 接口（app_boot_interface.h）

拷贝 `openbl/Inc/app_boot_interface.h` 到 App 工程，然后：

```c
#include "app_boot_interface.h"

int main(void) {
    APP_SetVectorTable();          /* ① main 第一行：SCB->VTOR = 0x08008000 */
    HAL_Init();
    ...
}

/* ② 需要升级时 */
APP_JumpToBootloader();            /* 写 RAM 标志 + 软复位，进 BL 等主机 */
```

App 工程还需：
- 链接偏移：`FLASH: ORIGIN = 0x08008000, LENGTH = 0x78000`
- 不要占用 RAM 末 4 字节 `0x2001FFFC`（跳转标志）

---

## 9. 与 ROM Bootloader 的区别

| | ROM Bootloader (AN2606) | 本软 BL |
|---|---|---|
| 位置 | 出厂 ROM | Flash 0x08000000 |
| 触发 | BOOT0=3.3V | 上电自动运行 |
| 同步字节 | 0x7F | 0x7F |
| 波特率 | 自动识别 | 固定 115200 |
| CubeProgrammer 模式 | ST Bootloader | **Open Bootloader** |

---

## 10. 常见问题

### Q: CubeProgrammer 连不上？
- 确认连接方式选 **Open Bootloader**（不是 ST Bootloader）
- 确认 8E1（CubeProgrammer 右侧 Parity=Even）；F4 上必须是 M=1+PCE=1
- 确认 BL 在运行：无 App 时应一直等主机，随时可连

### Q: 复位后跳 App 了，怎么进 BL？
- 用 App 的 `boot` 命令（方式 A），或复位后 1 s 内连 CubeProgrammer

### Q: 能升级 BL 自己吗？
- 不能走 UART（BL 拒绝擦除自身扇区）。用 SWD 直接重写 0x08000000。

### Q: 适配其他 STM32 系列？
- 需改：`platform.h`（HAL 头）、`openbootloader_conf.h`（内存/设备ID/页定义）、
  `flash_interface.c`（扇区/页擦写 API）、`usart_interface.c`（寄存器）、链接脚本。

---

## 11. 设计说明

### 11.1 为什么 USART 接口用寄存器级？
BL 只有收发单字节 + 波特率，寄存器级代码最精简、不依赖 LL/中间层，
拷入任何 HAL 工程都无需额外驱动。

### 11.2 8E1 的坑
STM32F4 要得到 **8 数据 + 1 偶校验**，字长必须配 9 位（M=1）+ 使能校验（PCE=1）。
只开 PCE 而 M=0 实际是 7 数据+校验，波形不匹配、字节被丢弃。

### 11.3 同步字节
本协议 USART 同步字节是 **0x7F**（与 ROM 一致）。注意 MW 里 `SYNC_BYTE` 宏是
0xA5，那是给 SPI/I3C 用的，USART 必须用 0x7F。

### 11.4 自保护原理
`flash_interface.c` 在写入/擦除入口统一判断地址/扇区，小于 0x08008000（扇区 0..1）
一律拒绝，主机无法破坏 BL。

### 11.5 跳转标志
App 在 `0x2001FFFC` 写 `0xDEADBEEF` 后软复位；BL 上电检查该值（RAM 复位不清），
命中则清标志并停在主机等待。该地址位于 RAM 顶，栈从 0x20020000 向下、.bss 向上，
正常不会用到。

---

## 版本历史

| 版本 | 日期 | 变更 |
|------|------|------|
| 1.0.0 | 2026-08-17 | 基于 stm32-mw-openbl 的 F407 自包含软 BL；8E1、0x7F、OB 读取、自保护、上电等待 1 s |

## 许可

ST MW 部分为 BSD-3-Clause；本模块接口层 MIT License — 可自由用于商业和开源项目。

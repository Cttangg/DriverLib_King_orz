# (2.8寸) TFT-LCD 触摸屏驱动 (FSMC + 软件 SPI)

STM32F407 2.8 寸 TFT-LCD (ILI9341) + 电阻触摸屏 (XPT2046) 可移植驱动组件, 版本 v1.3.0.

## 文件清单

| 文件 | 说明 |
|------|------|
| `lcd.c/h` | LCD 驱动: FSMC 并口, 多驱动 IC 自适应 (9341/9325/9328/9320/6804/5310/5510/4531/4535 等); 版本宏 `LCD_VER_*`; `LCD_InitEx()` 错误码 |
| `lcd_conf.h` | LCD 板级配置: 背光引脚 / FSMC 引脚组与读写时序 / 默认方向 / 面板尺寸覆盖 |
| `touch.c/h` | 电阻触摸驱动: 软件模拟 SPI, PEN 中断事件驱动 (EXTI9_5), 手势状态机 `TP_GetGesture()` (单击/长按/滑动) |
| `touch_conf.h` | 触摸板级配置: 引脚 / 校准开关 `TP_CAL_PRESET_ENABLE` / 滤波与手势参数 |
| `delay.c/h` | 基于 DWT 的微秒延时 (`delay_init`/`delay_us`) + `delay_ms` |
| `FONT.H` | ASCII 12x12 / 16x16 / 24x24 点阵字库 |
| `self_test.c/h` | 自检: LCD 色条+读点校验 / 触摸有效点 / 串口回环 (`LIB_SelfTest`) |

## 接线 (默认, 见 conf 头)

- LCD: FSMC NE1=PD7, A18=PD13, RD=PD4, WR=PD5, D0~D15 (PD/PE), 背光 PB1
- 触摸: T_PEN=PC5, T_MISO=PB14, T_MOSI=PB15, T_SCK=PB13, T_CS=PB12

## 初始化顺序

```c
LCD_Init();          /* 或 LCD_InitEx() 取错误码; 依赖 delay 模块 */
TP_Init();           /* 依赖 LCD; 预存校准值, 新屏设 TP_CAL_PRESET_ENABLE=0 后跑 TP_Adjust() */
```

主循环: `UART_Task()` + `TP_GetGesture()` (返回 `TOUCH_EVENT_SINGLE_CLICK/LONG_PRESS/SWIPE`).

## 移植

改 `lcd_conf.h` / `touch_conf.h` 即可; conf 版本与驱动不匹配会触发编译期 `#error`.

## 与源工程同步

源: `2.8-inch_LCD_Driver/Core/Lib` (该目录为唯一权威版本, 修改后复制回本目录).

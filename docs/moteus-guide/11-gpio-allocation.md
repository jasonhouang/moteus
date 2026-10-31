# Moteus GPIO 引脚分配

## 1. 概述

Moteus 控制器基于 STM32G474 MCU，使用约 40+ 个 GPIO 引脚实现电机控制、电源管理、通信和辅助扩展功能。不同板族（Family 0/1/2/3）的引脚分配略有差异，主要是为了优化 PCB 布线和 ADC 性能。

**板族说明**：
- **Family 0**: moteus r4.x（原版，10-44V，900W）
- **Family 1**: moteus-n1（高功率，10-54V，2kW）
- **Family 2**: moteus-c1（紧凑型，10-51V，250W）
- **Family 3**: moteus-x1（最新高功率，10-54V，1.3kW）

## 2. 硬件检测引脚（启动时使用）

这些引脚在系统启动时用于检测板族和硬件版本，之后不再使用。

| 引脚 | 功能 | 说明 |
|------|------|------|
| PB10 | Family Detect A | 板族检测（上拉/下拉组合判断） |
| PB11 | Family Detect B | 板族检测（上拉/下拉组合判断） |
| PC6 | HW Rev Detect 0 | Family 0 硬件版本检测（3位编码） |
| PA15 | HW Rev Detect 1 | Family 0 硬件版本检测（3位编码） |
| PA10 | HW Rev Detect 2 | Family 0 硬件版本检测（3位编码） |

**检测逻辑**：
- Family 0/1: PB10/PB11 悬空（通过检测 DRV8323 位置区分）
- Family 2 (c1): PB10 外部下拉，PB11 悬空
- Family 3 (x1): PB10 悬空，PB11 外部下拉

## 3. 电机控制（PWM 输出）

3 路 PWM 输出驱动三相电机，使用 TIM1 定时器，中心对齐模式。

### 所有板族（Family 0/1/2/3）

| 引脚 | 功能 | 定时器 | 说明 |
|------|------|--------|------|
| PA0 | PWM1 (Phase U) | TIM1_CH1 | 电机相 U PWM 输出 |
| PA1 | PWM2 (Phase V) | TIM1_CH2 | 电机相 V PWM 输出 |
| PA2 | PWM3 (Phase W) | TIM1_CH3 | 电机相 W PWM 输出 |

**PWM 频率**：
- Family 0 (r4.11): 60 kHz
- Family 1 (n1): 20 kHz
- Family 2 (c1): 30 kHz
- Family 3 (x1): 15 kHz

## 4. 电流采样（ADC 输入）

3 路电流采样用于 FOC 控制，需要快速 ADC 通道。

### Family 0 (moteus r4.x)

| 引脚 | 功能 | ADC | 说明 |
|------|------|-----|------|
| PB0 | Current1 (Phase U) | ADC1 | 相 U 电流采样（慢速通道） |
| PB1 | Current2 (Phase V) | ADC2 | 相 V 电流采样（慢速通道） |
| PB2 | Current3 (Phase W) | ADC3 | 相 W 电流采样（慢速通道） |

### Family 1/2/3 (n1/c1/x1)

| 引脚 | 功能 | ADC | 说明 |
|------|------|-----|------|
| PA3 | Current1 (Phase U) | ADC1_IN1 | 相 U 电流采样（快速通道） |
| PA6 | Current2 (Phase V) | ADC2_IN3 | 相 V 电流采样（快速通道） |
| PB1 | Current3 (Phase W) | ADC3_IN1 | 相 W 电流采样（快速通道） |

**注意**：Family 1/2/3 使用快速 ADC 通道，采样时间更短，支持更高的 PWM 频率。

## 5. 电压/温度采样（ADC 输入）

### 5.1 母线电压采样

#### Family 0 (moteus r4.x)

| 引脚 | 功能 | ADC | 说明 |
|------|------|-----|------|
| PA8 (r4.1-r4.3) / PB12 (r4.5+) | Vsense | ADC5 | 母线电压采样，ADC 比例因子 0.00884 (r4.5前) / 0.017947 (r4.5+) |

#### Family 1/2/3 (n1/c1/x1)

| 引脚 | 功能 | ADC | 说明 |
|------|------|-----|------|
| PA9 | Vsense | ADC5_IN2 | 母线电压采样，ADC 比例因子 0.017947 |

### 5.2 FET 温度采样

#### Family 0 (moteus r4.x)

| 引脚 | 功能 | ADC | 说明 |
|------|------|-----|------|
| PA9 | Tsense | ADC4/5 | FET 温度采样（板载热敏电阻） |

#### Family 1/2/3 (n1/c1/x1)

| 引脚 | 功能 | ADC | 说明 |
|------|------|-----|------|
| PB12 | Tsense | ADC4_IN3 | FET 温度采样（板载热敏电阻） |

### 5.3 电机温度采样

#### Family 0 (moteus r4.x)

| 引脚 | 功能 | ADC | 说明 |
|------|------|-----|------|
| PA8 (r4.5+) / PB12 (r4.4) / PA9 (r4.1-r4.3) | Msense | ADC4/5 | 电机温度采样（电机内置热敏电阻） |

**注意**：r4.1-r4.3 没有独立的电机温度采样，复用 FET 温度引脚。

#### Family 1/2/3 (n1/c1/x1)

| 引脚 | 功能 | ADC | 说明 |
|------|------|-----|------|
| PA8 | Msense | ADC5_IN1 | 电机温度采样（电机内置热敏电阻） |

## 6. 栅极驱动控制（DRV8323/DRV8353）

SPI 接口控制栅极驱动器，用于配置和保护。

### Family 0 (moteus r4.x)

| 引脚 | 功能 | 方向 | 说明 |
|------|------|------|------|
| PA3 | DRV Enable | 输出 | 栅极驱动使能（高电平有效） |
| PB7 | DRV HiZ | 输出 | 高阻态控制（低电平进入高阻） |
| PC4 | DRV CS | 输出 | SPI 片选（低电平有效） |
| PA5 | DRV SCK | 输出 | SPI 时钟 |
| PA6 | DRV MISO | 输入 | SPI 主入从出 |
| PA7 | DRV MOSI | 输出 | SPI 主出从入 |
| PB6 | DRV Fault | 输入 | 故障指示（低电平故障） |

### Family 1/2/3 (n1/c1/x1)

| 引脚 | 功能 | 方向 | 说明 |
|------|------|------|------|
| PC14 | DRV Enable | 输出 | 栅极驱动使能（高电平有效） |
| PC15 | DRV HiZ | 输出 | 高阻态控制（低电平进入高阻） |
| PB0 | DRV CS | 输出 | SPI 片选（低电平有效） |
| PC10 | DRV SCK | 输出 | SPI 时钟 |
| PC11 | DRV MISO | 输入 | SPI 主入从出 |
| PC13 | DRV MOSI | 输出 | SPI 主出从入 |
| PB13 | DRV Fault | 输入 | 故障指示（低电平故障） |

**SPI 参数**：
- 时钟频率：500 kHz（配置时）
- 数据格式：11位，MSB first
- 寄存器地址：0x00-0x0F

## 7. 编码器接口（AS5047 绝对值编码器）

SPI 接口读取 onboard 绝对值编码器。

### Family 0 (moteus r4.x)

| 引脚 | 功能 | 方向 | 说明 |
|------|------|------|------|
| PB11 | AS5047 CS | 输出 | 编码器片选（低电平有效） |
| PA5 | AS5047 SCK | 输出 | 编码器时钟（与 DRV 复用） |
| PA6 | AS5047 MISO | 输入 | 编码器数据（与 DRV 复用） |

### Family 1/2/3 (n1/c1/x1)

| 引脚 | 功能 | 方向 | 说明 |
|------|------|------|------|
| PB2 | AS5047 CS | 输出 | 编码器片选（低电平有效） |
| PC10 | AS5047 SCK | 输出 | 编码器时钟（与 DRV 复用） |
| PC11 | AS5047 MISO | 输入 | 编码器数据（与 DRV 复用） |

**注意**：编码器和栅极驱动共享 SPI 总线（MOSI/MISO/SCK），通过不同的 CS 引脚区分。

**SPI 参数**：
- 时钟频率：最高 10 MHz
- 数据格式：14位角度值 + 2位 CRC
- 分辨率：14位（16384 counts/rev）

## 8. CAN-FD 通信

CAN-FD 收发器接口，支持最高 5 Mbps 通信速率。

### Family 0 (moteus r4.x)

| 引脚 | 功能 | 方向 | 说明 |
|------|------|------|------|
| PA11 | CAN RD | 输入 | CAN 接收 |
| PA12 | CAN TD | 输出 | CAN 发送 |

### Family 1/2/3 (n1/c1/x1)

| 引脚 | 功能 | 方向 | 说明 |
|------|------|------|------|
| PB5 | CAN RD | 输入 | CAN 接收 |
| PB6 | CAN TD | 输出 | CAN 发送 |

**CAN-FD 参数**：
- 仲裁段波特率：1 Mbps
- 数据段波特率：5 Mbps（BRS 模式）
- 帧格式：CAN-FD（最高 64 字节）
- 滤波器：硬件过滤器，支持标准帧和扩展帧

## 9. LED 指示

### Family 0 (moteus r4.x)

| 引脚 | 功能 | 方向 | 说明 |
|------|------|------|------|
| PF0 | Debug LED1 | 输出 | 调试 LED（高电平点亮） |
| PF1 | Power LED | 输出 | 电源指示灯（高电平点亮） |

### Family 1/2/3 (n1/c1/x1)

| 引脚 | 功能 | 方向 | 说明 |
|------|------|------|------|
| PB15 | Debug LED1 | 输出 | 调试 LED（高电平点亮） |
| PC6 | Power LED | 输出 | 电源指示灯（高电平点亮） |

## 10. 调试接口

### Family 0 (moteus r4.x)

| 引脚 | 功能 | 方向 | 说明 |
|------|------|------|------|
| PA4 | Debug DAC | 输出 | 调试用 DAC 输出（12位，0-3.3V） |
| PC14 | Debug1 | 输出 | 调试 GPIO 1（可配置为 PWM/UART） |
| PC15 | Debug2 | 输出 | 调试 GPIO 2（可配置为 PWM/UART） |

### Family 1/2/3 (n1/c1/x1)

| 引脚 | 功能 | 方向 | 说明 |
|------|------|------|------|
| PA4 | Debug DAC | 输出 | 调试用 DAC 输出（12位，0-3.3V） |
| NC | Debug1 | - | 未使用 |
| NC | Debug2 | - | 未使用 |

**DAC 用途**：
- 可用于输出内部变量（电流、电压、PID 输出等）
- 通过 `servo.emit_debug` 配置输出内容
- 采样率与 PWM 频率相同

## 11. 辅助端口（Aux Port）

辅助端口提供灵活的 I/O 扩展能力，支持多种外设接口。

### Aux1 引脚分配（Family 1/2/3）

| 引脚 | 默认功能 | 可选功能 | 说明 |
|------|----------|----------|------|
| PA5 | GPIO | SPI_SCK, UART_TX | 可配置 |
| PA6 | GPIO | SPI_MISO, UART_RX | 可配置 |
| PA7 | GPIO | SPI_MOSI, ADC | 可配置 |
| PA8 | GPIO | ADC, PWM | 可配置 |
| PA15 | GPIO | UART_CTS | 可配置 |
| PB3 | GPIO | SPI_SCK, UART_RTS | 可配置 |
| PB4 | GPIO | SPI_MISO | 可配置 |

### Aux2 引脚分配（Family 1/2/3）

| 引脚 | 默认功能 | 可选功能 | 说明 |
|------|----------|----------|------|
| PA10 | GPIO | UART_TX, I2C_SDA | 可配置 |
| PA11 | GPIO | UART_RX, I2C_SCL | 可配置 |
| PA12 | GPIO | UART_RTS | 可配置 |
| PA13 | GPIO | SWDIO | 调试接口（默认禁用） |
| PA14 | GPIO | SWCLK | 调试接口（默认禁用） |

### 支持的外设类型

通过 `aux?.sources` 配置，支持以下外设：

1. **SPI 编码器**
   - AS5047, MA732, AKSIM-2
   - 最高 10 MHz 时钟
   - 14-16位分辨率

2. **UART 编码器**
   - BiSS-C, CUI AMT21/22, Orbis
   - 波特率：115200 - 10 Mbps
   - 支持 RS422 差分信号

3. **I2C 设备**
   - 支持标准模式（100 kHz）和快速模式（400 kHz）
   - 可连接外部传感器

4. **正交编码器**
   - A/B/Z 相信号
   - 最高计数率取决于 MCU 定时器

5. **Hall 传感器**
   - 3 路 Hall 输入
   - 用于电机换相

6. **模拟输入**
   - 12位 ADC
   - 采样率：最高 200 kSPS

7. **GPIO**
   - 数字输入/输出
   - 可配置上拉/下拉

## 12. SWD 调试接口

| 引脚 | 功能 | 说明 |
|------|------|------|
| PA13 | SWDIO | 串行调试数据（默认禁用，可作为 GPIO） |
| PA14 | SWCLK | 串行调试时钟（默认禁用，可作为 GPIO） |

**注意**：SWD 接口在正常运行时默认禁用，引脚可作为普通 GPIO 使用。需要调试时通过配置启用。

## 13. 引脚冲突和复用说明

### 13.1 SPI 总线复用

栅极驱动（DRV8323）和编码器（AS5047）共享 SPI 总线：
- 共享引脚：SCK, MISO, MOSI
- 通过不同的 CS 引脚区分
- 固件自动管理总线访问，无需用户干预

### 13.2 ADC 通道分配

STM32G474 有 5 个 ADC，通道分配如下：

**ADC1**（快速）：
- IN1: PA3 (Current1, Family 1/2/3)
- IN2: PA4 (Debug DAC)
- IN3: PA6 (Current2, Family 1/2/3)

**ADC2**（快速）：
- IN1: PA0 (PWM1)
- IN2: PA1 (PWM2)
- IN3: PA2 (PWM3)
- IN17: 内部温度传感器

**ADC3**（快速）：
- IN1: PB1 (Current3, Family 1/2/3)
- IN2: PB2 (AS5047 CS, Family 1/2/3)

**ADC4**：
- IN1: PA8 (Msense, Family 1/2/3)
- IN2: PA9 (Vsense, Family 1/2/3)
- IN3: PB12 (Tsense, Family 1/2/3)

**ADC5**：
- IN1: PA8 (Msense, Family 1/2/3)
- IN2: PA9 (Vsense, Family 1/2/3)
- IN3: PB12 (Tsense, Family 1/2/3)

### 13.3 定时器分配

**TIM1**（高级定时器）：
- CH1-CH3: PA0-PA2 (PWM1-3)
- 用于电机 PWM 输出
- 中心对齐模式，死区时间可配置

**TIM2/TIM3/TIM4**：
- 用于辅助功能（正交编码器、PWM 输出等）
- 可通过 Aux Port 配置使用

## 14. 未使用的引脚

以下引脚在固件中未使用（NC = Not Connected）：

### Family 0 (moteus r4.x)
- 大部分引脚已使用
- 部分辅助功能引脚可通过配置启用

### Family 1/2/3 (n1/c1/x1)
- Debug1, Debug2: 未使用
- 部分辅助引脚可通过 Aux Port 配置

## 15. 电源和地

### 电源引脚

| 电压 | 用途 | 说明 |
|------|------|------|
| 3.3V | MCU 核心供电 | STM32G474 主电源 |
| 5V | 辅助端口供电 | Aux1/Aux2 对外供电（最大 200mA） |
| 12V | 栅极驱动供电 | DRV8323 电荷泵 |
| VBUS | 母线电压 | 电机供电（10-54V，取决于板族） |

### 地引脚

| 标识 | 用途 | 说明 |
|------|------|------|
| GND | 数字地 | MCU 和数字电路参考地 |
| PGND | 功率地 | 电机和功率级参考地 |
| AGND | 模拟地 | ADC 和模拟电路参考地 |

**注意**：GND、PGND、AGND 在单点连接，避免地环路干扰。

## 16. 引脚配置示例

### 16.1 默认配置

固件启动时自动检测板族和版本，配置对应的引脚：

```cpp
// fw/moteus_hw.cc
MoteusHwPins FindHardwarePins(FamilyAndVersion fv) {
  MoteusHwPins result;
  
  if (fv.family == 0) {
    // Family 0 引脚配置
    result.pwm1 = PA_0_ALT0;
    result.pwm2 = PA_1_ALT0;
    result.pwm3 = PA_2_ALT0;
    result.drv8323_enable = PA_3;
    // ... 其他引脚
  } else if (fv.family == 1 || fv.family == 2 || fv.family == 3) {
    // Family 1/2/3 引脚配置
    result.pwm1 = PA_0_ALT0;
    result.pwm2 = PA_1_ALT0;
    result.pwm3 = PA_2_ALT0;
    result.drv8323_enable = PC_14;
    // ... 其他引脚
  }
  
  return result;
}
```

### 16.2 Aux Port 配置

通过 CAN-FD 命令配置辅助端口：

```python
# Python 示例
import moteus

controller = moteus.Controller()

# 配置 Aux1 为 SPI 编码器模式
await controller.set_config({
    'aux1.sources.0.type': 'spi',
    'aux1.sources.0.cpr': 16384,
})

# 配置 Aux2 为 UART 编码器模式
await controller.set_config({
    'aux2.sources.0.type': 'uart',
    'aux2.sources.0.uart.baud_rate': 1000000,
})
```

## 17. 电气特性

### 17.1 GPIO 电气参数

| 参数 | 值 | 说明 |
|------|-----|------|
| 工作电压 | 3.3V | MCU GPIO 电压 |
| 最大输出电流 | 25 mA | 单个 GPIO 引脚 |
| 总输出电流限制 | 120 mA | 所有 GPIO 总和 |
| 输入漏电流 | ±1 µA | 最大 |
| 上拉/下拉电阻 | 40 kΩ | 内部可配置 |

### 17.2 ADC 电气参数

| 参数 | 值 | 说明 |
|------|-----|------|
| 分辨率 | 12位 | 0-4095 |
| 参考电压 | 3.3V | VREF+ |
| 输入范围 | 0-3.3V | 单端输入 |
| 采样率 | 最高 4.27 MSPS | 取决于通道数 |
| 输入阻抗 | >50 kΩ | 采样期间 |

### 17.3 PWM 电气参数

| 参数 | 值 | 说明 |
|------|-----|------|
| 逻辑高电平 | 3.3V | GPIO 输出 |
| 逻辑低电平 | 0V | GPIO 输出 |
| 最大输出电流 | ±25 mA | 直接驱动能力 |
| 死区时间 | 50-200 ns | 可配置 |
| 频率范围 | 15-60 kHz | 取决于板族 |

## 18. 常见问题

### Q1: 为什么不同板族的引脚分配不同？

主要考虑因素：
1. **PCB 布线优化**：不同功率等级的板子，布局不同
2. **ADC 性能**：Family 1/2/3 使用快速 ADC 通道，支持更高 PWM 频率
3. **外设兼容性**：保持与旧版本的软件兼容性

### Q2: 可以同时使用 DRV8323 和 AS5047 吗？

可以。它们共享 SPI 总线（SCK/MISO/MOSI），但使用不同的 CS 引脚。固件会自动管理总线访问，无需用户干预。

### Q3: Aux Port 可以连接哪些设备？

支持的设备类型：
- SPI 编码器（AS5047, MA732, AKSIM-2, BiSS-C）
- UART 编码器（CUI AMT21/22, Orbis）
- I2C 传感器
- 正交编码器
- Hall 传感器
- 模拟传感器
- 数字 GPIO

### Q4: 如何查看当前引脚配置？

使用 `moteus_tool` 查看：

```bash
# 查看所有配置
python3 utils/moteus_tool.py --target 1 --dump-config

# 查看 Aux Port 配置
python3 utils/moteus_tool.py --target 1 --read aux1
python3 utils/moteus_tool.py --target 1 --read aux2
```

### Q5: 可以修改引脚分配吗？

**固定功能**（不可修改）：
- PWM1-3（电机输出）
- Current1-3（电流采样）
- Vsense/Tsense/Msense（电压/温度采样）
- CAN-FD（通信）

**可配置功能**：
- Aux1/Aux2（辅助端口）
- Debug DAC/GPIO（调试接口）

## 19. 参考资料

- STM32G474 数据手册：https://www.st.com/en/microcontrollers-microprocessors/stm32g474.html
- DRV8323 数据手册：https://www.ti.com/product/DRV8323
- AS5047 数据手册：https://ams.com/as5047
- Moteus 硬件设计文件：`hw/` 目录
- 固件源码：`fw/moteus_hw.cc`, `fw/moteus_hw.h`

## 20. 版本历史

| 版本 | 日期 | 变更 |
|------|------|------|
| 1.0 | 2026-10 | 初始版本，覆盖 Family 0/1/2/3 所有引脚 |

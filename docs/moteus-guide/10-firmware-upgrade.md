# Moteus 固件升级指南

## 1. 升级原理

### 1.1 Flash 分区结构

Moteus 控制器的 STM32G474 MCU 具有 512KB Flash，分为两个主要区域：

```
┌─────────────────────────────────────────┐
│  Flash 总容量: 512KB                    │
├─────────────────────────────────────────┤
│  0x0800_0000 - 0x0800_7FFF  (32KB)      │
│  Bootloader 区域                         │
│  - CAN 引导加载程序                       │
│  - 所有型号共用                           │
├─────────────────────────────────────────┤
│  0x0800_8000 - 0x0807_FFFF  (480KB)     │
│  Application 区域                        │
│  - 主固件                                │
│  - 运行时自动检测硬件型号                  │
│  - 支持 family 0/1/2/3                   │
└─────────────────────────────────────────┘
```

### 1.2 Bootloader 工作原理

Bootloader 是一个独立的小型程序，位于 Flash 的起始位置，负责：

1. **启动检测**：
   - 上电后首先运行 Bootloader
   - 检查是否需要进入升级模式
   - 如果无需升级，跳转到主固件

2. **升级模式触发条件**：
   - 收到特定的 CAN 命令（`d flash`）
   - 主固件主动请求重启到 Bootloader
   - SWD 调试器强制进入

3. **升级过程**：
   ```
   接收固件数据 (CAN-FD)
       ↓
   验证数据完整性
       ↓
   擦除 Application 区域
       ↓
   写入新固件
       ↓
   验证写入结果
       ↓
   复位并跳转到新固件
   ```

4. **协议支持**：
   - 使用 Multiplex Bootloader Protocol 1
   - 支持 CAN-FD 高速传输
   - 内置 CRC 校验确保数据完整性

### 1.3 硬件型号自动检测

主固件在启动时自动检测硬件型号（family）：

| Family | 型号 | Gate Driver | 检测方式 |
|--------|------|-------------|----------|
| 0 | moteus (r4.x) | DRV8323 | 默认 |
| 1 | moteus-n1 | DRV8353 | GPIO 电平检测 |
| 2 | moteus-c1 | DRV8323 | GPIO 电平检测 |
| 3 | moteus-x1 | - | GPIO 电平检测 |

**关键点**：
- Bootloader 不检测硬件型号，所有型号共用同一个 Bootloader
- 主固件是统一的二进制文件，运行时根据检测结果配置不同的硬件参数
- 升级时不需要指定硬件型号

## 2. CAN-FD 升级（推荐方式）

### 2.1 准备工作

**所需工具**：
- Python 3.8+
- moteus Python 库（已安装）
- CAN-FD 接口（SocketCAN 或 fdcanusb）
- 待升级的固件文件（`.elf` 格式）

**检查环境**：
```bash
# 确认 CAN 接口已启动
ip link show can0

# 如果未启动，配置 CAN 接口
sudo ip link set can0 up type can bitrate 1000000 dbitrate 5000000 fd on

# 确认设备在线
python3 -m moteus.moteus_tool --list
```

### 2.2 获取固件文件

**方式 1：从 GitHub Releases 下载（推荐）**

访问 https://github.com/mjbots/moteus/releases，下载最新版本的固件：

- `moteus-fw-<version>+g<sha>.elf` - 主固件（**需要这个**）
- `moteus-bl-<version>+g<sha>.elf` - Bootloader（通常不需要）

其中：
- `<version>` 是语义化版本号（如 `1.0.0` 或 `1.0.0-rc.1`）
- `<sha>` 是构建时的 git commit short hash（10 位）

**方式 2：从源码构建**

```bash
# 克隆仓库
git clone https://github.com/mjbots/moteus.git
cd moteus

# 构建固件
tools/bazel build --config=target //:target

# 生成的固件位置
ls -lh bazel-bin/fw/moteus
```

### 2.3 执行升级

**基本升级命令**：
```bash
python3 -m moteus.moteus_tool --target 1 --flash path/to/moteus-fw.elf
```

**参数说明**：
- `--target 1`：目标设备 ID（CAN 地址）
- `--flash`：执行固件升级
- `path/to/moteus-fw.elf`：固件文件路径

**升级过程输出示例**：
```
Entering bootloader
Erasing flash...
Writing firmware...
[========================================] 100%
Verifying...
Resetting device...
Firmware upgrade complete!
```

### 2.4 升级过程中的注意事项

1. **不要断电**：
   - 升级过程中绝对不能断开电源
   - 如果升级中断，设备可能变砖，需要通过 SWD 恢复

2. **保持 CAN 连接稳定**：
   - 确保 CAN 总线终端电阻正确（120Ω）
   - 避免在升级过程中有其他设备大量发送 CAN 数据

3. **等待自动重启**：
   - 升级完成后设备会自动重启
   - 重启时间约 1-2 秒
   - 重启后会自动运行新固件

4. **验证升级结果**：
```bash
# 查询固件版本
python3 -m moteus.moteus_tool --target 1 --query

# 查看详细信息
python3 -m moteus.moteus_tool --target 1 --info
```

### 2.5 批量升级

如果需要升级多个设备，可以编写脚本：

```python
#!/usr/bin/env python3
import subprocess
import sys

# 设备 ID 列表
device_ids = [1, 2, 3, 4, 5]
firmware_path = "path/to/moteus-fw.elf"

for device_id in device_ids:
    print(f"\n=== Upgrading device {device_id} ===")
    
    cmd = [
        "python3", "-m", "moteus.moteus_tool",
        "--target", str(device_id),
        "--flash", firmware_path
    ]
    
    result = subprocess.run(cmd, capture_output=True, text=True)
    
    if result.returncode == 0:
        print(f"Device {device_id}: Success")
    else:
        print(f"Device {device_id}: Failed")
        print(result.stderr)
        sys.exit(1)

print("\n=== All devices upgraded successfully ===")
```

**注意事项**：
- 批量升级时建议逐个进行，避免 CAN 总线冲突
- 每个设备升级完成后验证再升级下一个
- 可以并行升级不同 CAN 总线上的设备

## 3. SWD 升级（备用/恢复方式）

### 3.1 何时使用 SWD 升级

- CAN-FD 接口损坏
- Bootloader 损坏
- 升级过程中断电导致变砖
- 需要同时更新 Bootloader 和主固件
- 开发调试时需要频繁烧录

### 3.2 硬件准备

**所需设备**：
- [STM32 Programmer](https://mjbots.com/products/stm32-programmer) 或兼容的 SWD 调试器
- SWD 连接线（4 线：SWDIO, SWCLK, GND, 3.3V）
- 目标 moteus 控制器

**连接方式**：
```
STM32 Programmer          moteus 控制器
┌─────────────┐          ┌─────────────┐
│ SWDIO       │──────────│ SWDIO       │
│ SWCLK       │──────────│ SWCLK       │
│ GND         │──────────│ GND         │
│ 3.3V (可选) │──────────│ 3.3V (可选) │
└─────────────┘          └─────────────┘
```

**注意**：
- 如果 moteus 已有电源供应，不需要连接 3.3V
- 确保 SWD 接口接触良好
- SWD 线缆长度不宜超过 20cm

### 3.3 安装依赖

```bash
# Ubuntu/Debian
sudo apt install openocd binutils-arm-none-eabi

# 验证安装
openocd --version
arm-none-eabi-objcopy --version
```

### 3.4 使用 SWD 升级

**方式 1：构建并烧录**

```bash
# 构建固件
tools/bazel build --config=target //:target

# 烧录（同时更新 Bootloader 和主固件）
tools/bazel build --config=target //fw:flash

# 或者直接运行 flash 脚本
./fw/flash.py
```

**方式 2：烧录预编译的固件**

```bash
# 只烧录主固件
./fw/flash.py path/to/moteus-fw.elf

# 同时烧录 Bootloader 和主固件
./fw/flash.py path/to/moteus-fw.elf path/to/moteus-bl.elf
```

**方式 3：使用 OpenOCD 手动烧录**

```bash
# 启动 OpenOCD 服务
./run_openocd_noreset.sh

# 在另一个终端使用 GDB
gdb-multiarch -x moteus-debug.gdb bazel-bin/fw/moteus.elf

# 在 GDB 中执行
(gdb) load
(gdb) monitor reset run
(gdb) quit
```

### 3.5 SWD 升级输出示例

```
Open On-Chip Debugger 0.11.0
For bug reports, read
    http://openocd.org/doc/doxygen/bugs.html
Info : STLINK V2J37S7 (API v2) VID:PID 0483:3748
Info : Target voltage: 3.25 V
Info : stm32g4x.cpu: hardware has 6 breakpoints, 4 watchpoints
Info : starting gdb server for stm32g4x.cpu on 3333
Info : Listening on port 3333 for gdb connections
target halted due to debug-request, current mode: Thread
xPSR: 0x01000000 pc: 0x08000188 msp: 0x20000400
Info : device identity: STM32G474
Info : flash size = 512 kbytes
auto erase enabled
wrote 524288 bytes from file bazel-bin/fw/moteus.elf in 12.345678s (41.474 KiB/s)
verified 524288 bytes in 12.345678s
```

## 4. 升级失败恢复

### 4.1 常见失败场景

**场景 1：升级过程中断电**
- 症状：设备无响应，CAN 无法连接
- 原因：Flash 数据不完整，Bootloader 或主固件损坏
- 解决：使用 SWD 重新烧录

**场景 2：CAN 通信中断**
- 症状：升级进度卡住，超时错误
- 原因：CAN 总线干扰或连接不稳定
- 解决：
  1. 等待超时（通常 30 秒）
  2. 重新尝试升级
  3. 如果多次失败，使用 SWD 恢复

**场景 3：固件不兼容**
- 症状：升级成功但设备无法正常工作
- 原因：固件版本与硬件不匹配
- 解决：降级到兼容版本

### 4.2 恢复步骤

**步骤 1：尝试软恢复**

```bash
# 尝试进入 Bootloader 模式
python3 -m moteus.moteus_tool --target 1 --bootloader

# 重新升级
python3 -m moteus.moteus_tool --target 1 --flash path/to/moteus-fw.elf
```

**步骤 2：使用 SWD 完全恢复**

```bash
# 连接 SWD 调试器

# 擦除整个 Flash
openocd -f interface/stlink.cfg -f target/stm32g4x.cfg -c "init; reset halt; stm32g4x mass_erase 0; exit"

# 重新烧录 Bootloader 和主固件
./fw/flash.py path/to/moteus-fw.elf path/to/moteus-bl.elf
```

**步骤 3：验证恢复结果**

```bash
# 查询设备状态
python3 -m moteus.moteus_tool --target 1 --query

# 检查固件版本
python3 -m moteus.moteus_tool --target 1 --info
```

## 5. 版本管理

### 5.1 固件版本号

Moteus 使用语义化版本号（Semantic Versioning）：

```
MAJOR.MINOR.PATCH[-prerelease]+build-metadata

示例：
1.0.0              # 正式版本
1.0.0-rc.1         # 候选版本
1.0.0+g1234567890  # 带构建元数据（git commit hash）
```

**版本含义**：
- **MAJOR**：不兼容的 API 或配置变更
- **MINOR**：向后兼容的功能新增
- **PATCH**：向后兼容的问题修复

### 5.2 固件 ABI 版本

除了语义化版本号，固件还有内部 ABI 版本号（`FIRMWARE_VERSION`），用于配置兼容性管理：

```cpp
// fw/moteus_hw.h
#define MOTEUS_FIRMWARE_VERSION 0x010100
```

**ABI 版本变更历史**：
- `0x0100`：初始版本
- `0x0101`：修复前馈电压符号错误
- `0x0102`：移除 `servo.feedforward_scale`
- `0x0103`：添加 PWM 补偿配置
- `0x0104`：支持可配置 PWM 频率
- `0x0105`：新的编码器和位置子系统
- `0x0106`：I2C 轮询配置变更
- `0x0107`：修复输出符号问题
- `0x0108`：反电动势前馈默认值变更
- `0x0109`：换向补偿插值，电机热敏电阻配置
- `0x010a`：功率限制默认值变更
- `0x010b`：温度裕量配置重命名
- `0x010c`：PLL 滤波器配置变更
- `0x010d`：RS422 配置路径变更
- `0x010e`：电流环 PID 配置变更
- `0x010000`：ABI 1.0.0 发布
- `0x010100`：Gate driver 配置修正，ABI 1.1.0 发布

### 5.3 配置兼容性

**升级时的配置迁移**：

当 ABI 版本变更时，`moteus_tool` 会自动迁移配置：

```bash
# 升级固件
python3 -m moteus.moteus_tool --target 1 --flash moteus-fw.elf

# 自动迁移配置
# - 删除已废弃的参数
# - 添加新参数的默认值
# - 转换格式变更的参数
```

**降级时的注意事项**：

- **同 MAJOR 版本降级**：通常安全，配置会自动适配
- **跨 MAJOR 版本降级**：可能丢失配置，建议先备份

**备份配置**：
```bash
# 导出当前配置
python3 -m moteus.moteus_tool --target 1 --dump-config > config_backup.txt

# 恢复配置
python3 -m moteus.moteus_tool --target 1 --load-config config_backup.txt
```

### 5.4 版本查询

**查询固件版本**：
```bash
# 完整信息
python3 -m moteus.moteus_tool --target 1 --info

# 输出示例：
# Serial number: 0x12345678
# Firmware version: 1.1.0+g1234567890
# Model number: 0
# Hardware version: 2 (moteus-c1)
```

**通过代码查询**：
```python
import moteus
import asyncio

async def query_version():
    transport = moteus.TransportFactory.create()
    controller = moteus.Controller(id=1, transport=transport)
    
    info = await controller.query()
    print(f"Firmware version: {info['firmware_version']}")
    print(f"Hardware family: {info['hw_family']}")

asyncio.run(query_version())
```

## 6. 高级主题

### 6.1 自定义 Bootloader

如果需要修改 Bootloader（通常不需要）：

```bash
# 构建 Bootloader
tools/bazel build --config=target //fw:can_bootloader

# 生成的文件
ls -lh bazel-bin/fw/can_bootloader
```

**注意**：
- 修改 Bootloader 需要非常谨慎
- 错误的 Bootloader 会导致设备完全变砖
- 只能通过 SWD 恢复

### 6.2 固件签名验证

当前版本的 Bootloader **不验证固件签名**，这意味着：

- 可以烧录任何兼容的固件
- 需要确保固件来源可靠
- 生产环境建议添加签名验证

### 6.3 双 Bank 升级（未来特性）

STM32G474 支持双 Bank Flash，可以实现：

- 在 Bank 1 运行当前固件
- 在 Bank 2 写入新固件
- 验证成功后切换启动 Bank
- 失败时自动回滚

**当前状态**：Moteus 尚未实现双 Bank 升级，所有升级都是直接覆盖。

### 6.4 OTA 升级（无线升级）

通过外部无线模块（如 WiFi、蓝牙）实现 OTA 升级：

```
无线模块 ←→ 外部 MCU ←→ CAN-FD ←→ moteus 控制器
                                      ↓
                                  Bootloader
                                      ↓
                                  固件升级
```

**实现要点**：
- 外部 MCU 作为 CAN-FD 网关
- 使用 `moteus_tool` 的 API 进行升级
- 需要实现固件下载和缓存
- 添加升级进度反馈

## 7. 最佳实践

### 7.1 生产环境升级流程

1. **测试环境验证**：
   - 在测试设备上验证新固件
   - 运行完整测试套件
   - 确认功能正常

2. **备份配置**：
   ```bash
   python3 -m moteus.moteus_tool --target 1 --dump-config > config_backup.txt
   ```

3. **通知相关人员**：
   - 告知升级时间和影响范围
   - 准备回滚方案

4. **执行升级**：
   - 选择低负载时段
   - 逐个设备升级
   - 每个设备升级后验证

5. **升级后验证**：
   ```bash
   # 检查固件版本
   python3 -m moteus.moteus_tool --target 1 --info
   
   # 运行基本功能测试
   python3 test_basic_functionality.py
   
   # 监控运行状态
   python3 -m moteus_gui.tview --devices=1
   ```

6. **记录升级日志**：
   - 记录升级时间、设备 ID、固件版本
   - 记录升级过程中的问题
   - 保存配置备份

### 7.2 开发环境升级流程

开发时频繁升级固件的建议：

1. **使用 SWD 调试器**：
   - 更快的烧录速度
   - 支持断点调试
   - 可以查看寄存器状态

2. **自动化脚本**：
   ```bash
   #!/bin/bash
   # build_and_flash.sh
   
   tools/bazel build --config=target //:target
   if [ $? -eq 0 ]; then
       ./fw/flash.py
       echo "Flash complete!"
   else
       echo "Build failed!"
       exit 1
   fi
   ```

3. **增量构建**：
   ```bash
   # 只重新编译修改的文件
   tools/bazel build --config=target //:target
   ```

### 7.3 故障排查清单

**升级失败时检查**：

- [ ] CAN 接口是否正确配置（波特率、FD 模式）
- [ ] 设备 ID 是否正确
- [ ] 固件文件是否完整（检查文件大小）
- [ ] CAN 总线终端电阻是否正确（120Ω）
- [ ] 设备供电是否稳定
- [ ] 是否有其他设备干扰 CAN 总线
- [ ] 尝试降低 CAN 波特率

**设备无响应时检查**：

- [ ] 电源电压是否正常
- [ ] SWD 接口连接是否可靠
- [ ] 尝试 SWD 恢复
- [ ] 检查是否有硬件损坏

## 8. 总结

Moteus 固件升级提供了两种方式：

| 方式 | 优点 | 缺点 | 适用场景 |
|------|------|------|----------|
| **CAN-FD 升级** | 方便快捷，无需额外硬件 | 依赖 CAN 通信 | 日常升级、批量升级 |
| **SWD 升级** | 可靠，可恢复变砖设备 | 需要专用硬件 | 开发调试、故障恢复 |

**关键要点**：
- Bootloader 和主固件都是统一的，所有型号共用
- 升级过程中绝对不能断电
- 升级前建议备份配置
- 生产环境建议先在测试环境验证
- 遇到问题时优先尝试 CAN 恢复，失败再用 SWD

掌握固件升级流程是 Moteus 开发和维护的基础技能，务必在实际上手前充分理解。

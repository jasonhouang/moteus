# Moteus 项目概述

## 1. 项目简介

Moteus 是一个高性能无刷直流电机伺服控制器系统，专为机器人应用设计。该项目包含完整的硬件设计、固件、客户端软件库和工具链。

### 1.1 核心特性

- **高性能电机控制**：支持位置、速度、扭矩多种控制模式
- **集成位置反馈**：内置编码器支持，提供精确的位置反馈
- **CAN-FD通信**：使用CAN-FD协议进行高速、可靠的通信
- **多硬件版本支持**：支持r4.11、c1、n1、x1等多个硬件版本
- **完整的软件栈**：从底层固件到上层客户端库的完整实现

### 1.2 应用场景

- 机器人关节执行器
- 精密运动控制系统
- 力控机器人
- 工业自动化设备

### 1.3 技术栈

- **固件**：C++，基于STM32G4微控制器，使用mbed-os
- **构建系统**：Bazel（主要），CMake（客户端库）
- **客户端库**：Python、C++、Rust
- **通信协议**：自定义CAN-FD协议
- **测试框架**：Google Test（C++），pytest（Python）

## 2. 项目结构

```
moteus/
├── fw/              # 固件源代码
│   ├── test/        # 固件单元测试
│   └── *.h/*.cc     # 核心控制算法和硬件驱动
├── lib/             # 客户端库
│   ├── python/      # Python客户端库
│   ├── cpp/         # C++客户端库
│   └── rust/        # Rust客户端库
├── hw/              # 硬件设计文件
│   ├── controller/  # r4.11版本PCB设计
│   ├── c1/          # 紧凑型控制器
│   ├── n1/          # 大电流控制器
│   └── x1/          # 最新大功率控制器
├── utils/           # 工具和实用程序
│   ├── gui/         # 图形界面工具
│   └── test/        # 工具测试
├── docs/            # 文档
│   ├── reference/   # 参考文档
│   ├── guides/      # 使用指南
│   └── protocol/    # 协议文档
├── tools/           # 构建工具和配置
└── configs/         # 配置文件
```

## 3. 硬件版本

### 3.1 r4.11（原始版本）
- 电压范围：10-44V
- 峰值功率：900W
- 适用场景：通用机器人关节

### 3.2 c1（紧凑型）
- 电压范围：10-51V
- 峰值功率：250W
- 特点：体积更小，适合空间受限的应用

### 3.3 n1（大电流型）
- 电压范围：10-54V
- 峰值功率：2kW
- 特点：支持更大电流，适合高扭矩应用

### 3.4 x1（最新大功率型）
- 电压范围：10-54V
- 峰值功率：1.3kW
- 特点：最新设计，性能优化

## 4. 快速开始

### 4.1 环境准备

安装必要的Ubuntu软件包：

```bash
sudo apt update
sudo apt install -y python3-build python3-can python3-serial python3-setuptools \
    python3-pyelftools python3-qtpy python3-wheel python3-importlib-metadata \
    python3-scipy python3-usb mypy nodejs
```

### 4.2 编译固件

```bash
# 使用项目自带的bazel wrapper
tools/bazel build --config=target //:target
```

### 4.3 运行测试

```bash
# 运行快速测试
tools/bazel test --config=host //:host

# 运行完整测试套件（包括仿真回归测试）
tools/bazel test --test_size_filters= --test_tag_filters=-manual --config=host //:host
```

### 4.4 使用命令行工具

```bash
# 配置和诊断工具
utils/moteus_tool.py

# 遥测查看器
utils/tview.py
```

## 5. 学习路径建议

1. **第一阶段：基础理解**
   - 阅读本文档，了解项目整体架构
   - 查看 `docs/reference.md` 了解参数配置
   - 阅读 `docs/quick-start.md` 了解基本使用

2. **第二阶段：协议和通信**
   - 学习CAN-FD通信基础
   - 阅读 `docs/protocol/` 目录下的协议文档
   - 理解multiplex协议格式

3. **第三阶段：固件架构**
   - 阅读 `fw/moteus_controller.h` 了解主控制器
   - 学习 `fw/bldc_servo.h` 了解电机控制核心
   - 理解 `fw/motor_position.h` 了解位置反馈系统

4. **第四阶段：客户端开发**
   - 学习Python客户端库 `lib/python/moteus/moteus.py`
   - 了解C++客户端库 `lib/cpp/mjbots/moteus/moteus.h`
   - 实践基本的控制命令

5. **第五阶段：高级主题**
   - 研究PID调参
   - 了解编码器校准流程
   - 探索硬件设计和PCB布局

## 6. 重要概念

### 6.1 控制模式

- **位置模式（Position Mode）**：集成位置/速度控制，主要工作模式
- **速度模式（Velocity Mode）**：纯速度控制（kp_scale = 0）
- **扭矩模式（Torque Mode）**：直接扭矩控制
- **电流模式（Current Mode）**：直接电流控制（底层）

### 6.2 关键配置参数

- `servopos.position_min/max`：位置限制
- `servo.max_current_A`：电流限制
- `servo.pid_position`：位置环PID参数
- `motor_position.rotor_to_output_ratio`：齿轮比
- `id.id`：CAN-FD设备ID

### 6.3 通信架构

系统使用CAN-FD总线进行通信，支持：
- 阻塞式命令（同步）
- 异步命令（非阻塞）
- 多设备寻址
- 遥测数据流

## 7. 开发规范

### 7.1 代码风格

- C++代码遵循Google C++风格指南
- Python代码遵循PEP8
- 不允许尾随空格
- 空行不能包含任何空白字符

### 7.2 提交规范

- Commit message和代码注释必须使用英文
- Commit message必须准确反映所有修改
- 只提交相关文件，不要提交所有未跟踪文件

### 7.3 测试要求

- 新功能必须包含相应的单元测试
- 修改后必须运行相关测试确保不破坏现有功能
- 固件修改需要运行仿真测试

## 8. 常见问题

### 8.1 构建失败

**问题**：Bazel无法下载依赖
**解决**：使用离线构建方法
```bash
bash utils/download_bazel_deps.sh
tools/bazel build --config=target //:target \
    --repository_cache=/tmp/repo_cache \
    --distdir=/tmp/bazel_cache \
    --sandbox_writable_path=/dev
```

### 8.2 Python导入错误

**问题**：系统Python版本冲突
**解决**：检查并移除冲突的Python链接
```bash
# 如果 /usr/local/bin/python3 指向非系统Python
sudo mv /usr/local/bin/python3 /usr/local/bin/python3.bak
```

### 8.3 设备连接失败

**问题**：无法连接到moteus设备
**解决**：
1. 检查CAN-FD接口是否正确配置
2. 确认设备ID设置正确
3. 检查电源和接线

## 9. 进一步阅读

- `docs/reference.md`：完整的参数参考
- `docs/protocol/`：通信协议详细说明
- `docs/guides/`：各种使用指南
- `docs/troubleshooting/`：故障排除指南
- `docs/integration/`：集成指南

## 10. 联系和支持

- 官方网站：https://mjbots.com
- 技术支持：info@mjbots.com
- 项目仓库：查看项目根目录的README获取最新信息

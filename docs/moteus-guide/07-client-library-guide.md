# Moteus 客户端库使用指南

## 1. Python客户端库

### 1.1 安装

```bash
# 从源码安装（开发环境）
cd lib/python
pip3 install -e .

# 或者使用pip安装发布版本
pip3 install moteus
```

### 1.2 基本使用

#### 1.2.1 初始化和连接

```python
import asyncio
import moteus

async def main():
    # 创建传输层（使用SocketCAN）
    transport = moteus.TransportFactory.create(
        can_bus='socketcan',
        channel='can0'
    )
    
    # 创建控制器实例（设备ID=1）
    controller = moteus.Controller(id=1, transport=transport)
    
    # 查询设备状态
    result = await controller.query()
    print(f"Position: {result['position']}")
    print(f"Velocity: {result['velocity']}")
    print(f"Mode: {result['mode']}")

asyncio.run(main())
```

#### 1.2.2 位置控制

```python
import asyncio
import moteus

async def main():
    transport = moteus.TransportFactory.create(
        can_bus='socketcan',
        channel='can0'
    )
    controller = moteus.Controller(id=1, transport=transport)
    
    # 停止电机
    await controller.set_stop()
    
    # 位置控制
    result = await controller.set_position(
        position=1.57,           # 目标位置（弧度）
        velocity=0.0,            # 目标速度（弧度/秒）
        feedforward_torque=0.0,  # 前馈扭矩（Nm）
        kp_scale=1.0,            # 比例增益缩放
        kd_scale=1.0,            # 微分增益缩放
        maximum_torque=2.0,      # 最大扭矩限制（Nm）
    )
    
    print(f"Current position: {result['position']}")
    print(f"Current velocity: {result['velocity']}")
    print(f"Current torque: {result['torque']}")

asyncio.run(main())
```

#### 1.2.3 速度控制

```python
import asyncio
import moteus

async def main():
    transport = moteus.TransportFactory.create(
        can_bus='socketcan',
        channel='can0'
    )
    controller = moteus.Controller(id=1, transport=transport)
    
    # 速度控制
    result = await controller.set_velocity(
        velocity=10.0,           # 目标速度（弧度/秒）
        feedforward_torque=0.0,  # 前馈扭矩（Nm）
        kp_scale=0.0,            # 位置环禁用
        kd_scale=1.0,            # 微分增益缩放
        maximum_torque=2.0,      # 最大扭矩限制（Nm）
    )
    
    print(f"Current velocity: {result['velocity']}")

asyncio.run(main())
```

#### 1.2.4 扭矩控制

```python
import asyncio
import moteus

async def main():
    transport = moteus.TransportFactory.create(
        can_bus='socketcan',
        channel='can0'
    )
    controller = moteus.Controller(id=1, transport=transport)
    
    # 扭矩控制
    result = await controller.set_torque(
        torque=0.5,              # 目标扭矩（Nm）
        maximum_torque=2.0,      # 最大扭矩限制（Nm）
    )
    
    print(f"Current torque: {result['torque']}")

asyncio.run(main())
```

### 1.3 配置管理

#### 1.3.1 读取配置

```python
import asyncio
import moteus

async def main():
    transport = moteus.TransportFactory.create(
        can_bus='socketcan',
        channel='can0'
    )
    controller = moteus.Controller(id=1, transport=transport)
    
    # 读取单个配置
    value = await controller.read_config('servopos.position_max')
    print(f"Position max: {value}")
    
    # 读取多个配置
    configs = await controller.read_config([
        'servopos.position_max',
        'servopos.position_min',
        'servo.max_current_A',
    ])
    
    for key, value in configs.items():
        print(f"{key}: {value}")

asyncio.run(main())
```

#### 1.3.2 写入配置

```python
import asyncio
import moteus

async def main():
    transport = moteus.TransportFactory.create(
        can_bus='socketcan',
        channel='can0'
    )
    controller = moteus.Controller(id=1, transport=transport)
    
    # 写入单个配置
    await controller.write_config('servopos.position_max', 10.0)
    
    # 写入多个配置
    await controller.write_config({
        'servopos.position_max': 10.0,
        'servopos.position_min': -10.0,
        'servo.max_current_A': 20.0,
    })
    
    # 保存配置到Flash
    await controller.write_config('motor_position.rotor_to_output_ratio', 1.0)
    await controller.write_config('id.id', 1)

asyncio.run(main())
```

### 1.4 高级功能

#### 1.4.1 多设备控制

```python
import asyncio
import moteus

async def main():
    transport = moteus.TransportFactory.create(
        can_bus='socketcan',
        channel='can0'
    )
    
    # 创建多个控制器
    controllers = [
        moteus.Controller(id=i, transport=transport)
        for i in range(1, 5)  # 设备ID 1-4
    ]
    
    # 并行查询所有设备
    results = await asyncio.gather(*[
        controller.query() for controller in controllers
    ])
    
    for i, result in enumerate(results, start=1):
        print(f"Device {i}: position={result['position']:.3f}")

asyncio.run(main())
```

#### 1.4.2 周期性控制

```python
import asyncio
import moteus
import time

async def main():
    transport = moteus.TransportFactory.create(
        can_bus='socketcan',
        channel='can0'
    )
    controller = moteus.Controller(id=1, transport=transport)
    
    # 周期性位置控制（1kHz）
    period = 0.001  # 1ms
    start_time = time.time()
    
    try:
        while True:
            # 生成正弦轨迹
            t = time.time() - start_time
            position = 2.0 * 3.14159 * 0.5 * (1 - (t % 2.0))
            
            # 发送位置命令
            result = await controller.set_position(
                position=position,
                velocity=0.0,
            )
            
            # 等待下一个周期
            elapsed = time.time() - start_time
            next_time = (int(elapsed / period) + 1) * period
            await asyncio.sleep(next_time - elapsed)
    
    except KeyboardInterrupt:
        await controller.set_stop()

asyncio.run(main())
```

#### 1.4.3 遥测数据流

```python
import asyncio
import moteus

async def main():
    transport = moteus.TransportFactory.create(
        can_bus='socketcan',
        channel='can0'
    )
    controller = moteus.Controller(id=1, transport=transport)
    
    # 配置遥测数据
    await controller.write_config('telemetry.rate', 100)  # 100Hz
    
    # 订阅遥测数据
    async for telemetry in controller.telemetry_stream():
        print(f"Position: {telemetry['position']:.3f}, "
              f"Velocity: {telemetry['velocity']:.3f}, "
              f"Torque: {telemetry['torque']:.3f}")

asyncio.run(main())
```

### 1.5 错误处理

```python
import asyncio
import moteus

async def main():
    transport = moteus.TransportFactory.create(
        can_bus='socketcan',
        channel='can0'
    )
    controller = moteus.Controller(id=1, transport=transport)
    
    try:
        result = await controller.query()
    except moteus.CommunicationTimeout:
        print("Communication timeout - check connection")
    except moteus.DeviceFault as e:
        print(f"Device fault: {e.fault_code}")
    except Exception as e:
        print(f"Error: {e}")

asyncio.run(main())
```

### 1.6 完整示例：轨迹跟踪

```python
import asyncio
import moteus
import time
import math

async def trajectory_tracking():
    """正弦轨迹跟踪示例"""
    
    # 初始化
    transport = moteus.TransportFactory.create(
        can_bus='socketcan',
        channel='can0'
    )
    controller = moteus.Controller(id=1, transport=transport)
    
    # 配置参数
    amplitude = 2.0  # 弧度
    frequency = 0.5  # Hz
    duration = 10.0  # 秒
    
    # 停止电机
    await controller.set_stop()
    await asyncio.sleep(0.5)
    
    # 开始轨迹跟踪
    start_time = time.time()
    control_rate = 1000  # Hz
    
    try:
        while True:
            elapsed = time.time() - start_time
            
            if elapsed > duration:
                break
            
            # 计算目标位置和速度
            position = amplitude * math.sin(2 * math.pi * frequency * elapsed)
            velocity = amplitude * 2 * math.pi * frequency * math.cos(2 * math.pi * frequency * elapsed)
            
            # 发送位置命令
            result = await controller.set_position(
                position=position,
                velocity=velocity,
                kp_scale=1.0,
                kd_scale=1.0,
            )
            
            # 打印状态（每100ms）
            if int(elapsed * 10) % 1 == 0:
                print(f"t={elapsed:.2f}s: pos_cmd={position:.3f}, "
                      f"pos_act={result['position']:.3f}, "
                      f"vel_act={result['velocity']:.3f}")
            
            # 等待下一个控制周期
            await asyncio.sleep(1.0 / control_rate)
    
    except KeyboardInterrupt:
        pass
    finally:
        # 停止电机
        await controller.set_stop()
        print("Trajectory tracking completed")

if __name__ == '__main__':
    asyncio.run(trajectory_tracking())
```

## 2. C++客户端库

### 2.1 编译

C++库是header-only的，只需包含头文件：

```cpp
#include "mjbots/moteus/moteus.h"
```

### 2.2 基本使用

#### 2.2.1 初始化和连接

```cpp
#include "mjbots/moteus/moteus.h"
#include <iostream>

int main() {
    // 创建传输层
    mjbots::moteus::SocketCanTransport transport("can0");
    
    // 创建控制器
    mjbots::moteus::Controller::Options options;
    options.id = 1;
    options.transport = &transport;
    
    mjbots::moteus::Controller controller(options);
    
    // 查询状态
    auto result = controller.Query();
    
    std::cout << "Position: " << result.position << std::endl;
    std::cout << "Velocity: " << result.velocity << std::endl;
    std::cout << "Mode: " << result.mode << std::endl;
    
    return 0;
}
```

#### 2.2.2 位置控制

```cpp
#include "mjbots/moteus/moteus.h"
#include <iostream>

int main() {
    mjbots::moteus::SocketCanTransport transport("can0");
    
    mjbots::moteus::Controller::Options options;
    options.id = 1;
    options.transport = &transport;
    
    mjbots::moteus::Controller controller(options);
    
    // 停止电机
    controller.SetStop();
    
    // 位置控制
    mjbots::moteus::Controller::PositionCommand cmd;
    cmd.position = 1.57;           // 弧度
    cmd.velocity = 0.0;            // 弧度/秒
    cmd.feedforward_torque = 0.0;  // Nm
    cmd.kp_scale = 1.0;
    cmd.kd_scale = 1.0;
    cmd.maximum_torque = 2.0;      // Nm
    
    auto result = controller.SetPosition(cmd);
    
    std::cout << "Current position: " << result.position << std::endl;
    std::cout << "Current velocity: " << result.velocity << std::endl;
    
    return 0;
}
```

#### 2.2.3 速度控制

```cpp
#include "mjbots/moteus/moteus.h"

int main() {
    mjbots::moteus::SocketCanTransport transport("can0");
    
    mjbots::moteus::Controller::Options options;
    options.id = 1;
    options.transport = &transport;
    
    mjbots::moteus::Controller controller(options);
    
    // 速度控制
    mjbots::moteus::Controller::VelocityCommand cmd;
    cmd.velocity = 10.0;           // 弧度/秒
    cmd.feedforward_torque = 0.0;
    cmd.kp_scale = 0.0;            // 禁用位置环
    cmd.kd_scale = 1.0;
    cmd.maximum_torque = 2.0;
    
    auto result = controller.SetVelocity(cmd);
    
    return 0;
}
```

#### 2.2.4 扭矩控制

```cpp
#include "mjbots/moteus/moteus.h"

int main() {
    mjbots::moteus::SocketCanTransport transport("can0");
    
    mjbots::moteus::Controller::Options options;
    options.id = 1;
    options.transport = &transport;
    
    mjbots::moteus::Controller controller(options);
    
    // 扭矩控制
    mjbots::moteus::Controller::TorqueCommand cmd;
    cmd.torque = 0.5;              // Nm
    cmd.maximum_torque = 2.0;
    
    auto result = controller.SetTorque(cmd);
    
    return 0;
}
```

### 2.3 异步接口

```cpp
#include "mjbots/moteus/moteus.h"
#include <iostream>

void QueryCallback(const mjbots::moteus::Controller::Result& result) {
    std::cout << "Position: " << result.position << std::endl;
    std::cout << "Velocity: " << result.velocity << std::endl;
}

int main() {
    mjbots::moteus::SocketCanTransport transport("can0");
    
    mjbots::moteus::Controller::Options options;
    options.id = 1;
    options.transport = &transport;
    
    mjbots::moteus::Controller controller(options);
    
    // 异步查询
    controller.AsyncQuery(&QueryCallback);
    
    // 等待完成（实际应用中需要事件循环）
    while (true) {
        transport.Poll();
    }
    
    return 0;
}
```

### 2.4 配置管理

```cpp
#include "mjbots/moteus/moteus.h"
#include <iostream>

int main() {
    mjbots::moteus::SocketCanTransport transport("can0");
    
    mjbots::moteus::Controller::Options options;
    options.id = 1;
    options.transport = &transport;
    
    mjbots::moteus::Controller controller(options);
    
    // 读取配置
    float position_max = controller.ReadConfig<float>("servopos.position_max");
    std::cout << "Position max: " << position_max << std::endl;
    
    // 写入配置
    controller.WriteConfig("servopos.position_max", 10.0f);
    controller.WriteConfig("servopos.position_min", -10.0f);
    controller.WriteConfig("servo.max_current_A", 20.0f);
    
    return 0;
}
```

### 2.5 多设备控制

```cpp
#include "mjbots/moteus/moteus.h"
#include <vector>
#include <iostream>

int main() {
    mjbots::moteus::SocketCanTransport transport("can0");
    
    // 创建多个控制器
    std::vector<mjbots::moteus::Controller> controllers;
    
    for (int id = 1; id <= 4; id++) {
        mjbots::moteus::Controller::Options options;
        options.id = id;
        options.transport = &transport;
        
        controllers.emplace_back(options);
    }
    
    // 查询所有设备
    for (auto& controller : controllers) {
        auto result = controller.Query();
        std::cout << "Device " << controller.options().id 
                  << ": position=" << result.position << std::endl;
    }
    
    return 0;
}
```

### 2.6 完整示例：轨迹跟踪

```cpp
#include "mjbots/moteus/moteus.h"
#include <iostream>
#include <cmath>
#include <chrono>
#include <thread>

int main() {
    // 初始化
    mjbots::moteus::SocketCanTransport transport("can0");
    
    mjbots::moteus::Controller::Options options;
    options.id = 1;
    options.transport = &transport;
    
    mjbots::moteus::Controller controller(options);
    
    // 配置参数
    const double amplitude = 2.0;  // 弧度
    const double frequency = 0.5;  // Hz
    const double duration = 10.0;  // 秒
    const int control_rate = 1000; // Hz
    
    // 停止电机
    controller.SetStop();
    std::this_thread::sleep_for(std::chrono::milliseconds(500));
    
    // 开始轨迹跟踪
    auto start_time = std::chrono::high_resolution_clock::now();
    
    try {
        while (true) {
            auto current_time = std::chrono::high_resolution_clock::now();
            double elapsed = std::chrono::duration<double>(current_time - start_time).count();
            
            if (elapsed > duration) {
                break;
            }
            
            // 计算目标位置和速度
            double position = amplitude * std::sin(2 * M_PI * frequency * elapsed);
            double velocity = amplitude * 2 * M_PI * frequency * 
                             std::cos(2 * M_PI * frequency * elapsed);
            
            // 发送位置命令
            mjbots::moteus::Controller::PositionCommand cmd;
            cmd.position = position;
            cmd.velocity = velocity;
            cmd.kp_scale = 1.0;
            cmd.kd_scale = 1.0;
            
            auto result = controller.SetPosition(cmd);
            
            // 打印状态（每100ms）
            if (static_cast<int>(elapsed * 10) % 1 == 0) {
                std::cout << "t=" << elapsed << "s: "
                          << "pos_cmd=" << position << ", "
                          << "pos_act=" << result.position << ", "
                          << "vel_act=" << result.velocity << std::endl;
            }
            
            // 等待下一个控制周期
            std::this_thread::sleep_for(
                std::chrono::microseconds(1000000 / control_rate));
        }
    } catch (const std::exception& e) {
        std::cerr << "Error: " << e.what() << std::endl;
    }
    
    // 停止电机
    controller.SetStop();
    std::cout << "Trajectory tracking completed" << std::endl;
    
    return 0;
}
```

## 3. 命令行工具

### 3.1 moteus_tool.py

`moteus_tool.py`是主要的命令行工具，用于配置和诊断。

#### 3.1.1 基本用法

```bash
# 查看帮助
python3 utils/moteus_tool.py --help

# 列出所有设备
python3 utils/moteus_tool.py --list

# 查询设备状态
python3 utils/moteus_tool.py --target 1 --query

# 停止电机
python3 utils/moteus_tool.py --target 1 --stop
```

#### 3.1.2 配置管理

```bash
# 读取配置
python3 utils/moteus_tool.py --target 1 --read-config servopos.position_max

# 写入配置
python3 utils/moteus_tool.py --target 1 --write-config servopos.position_max=10.0

# 读取所有配置
python3 utils/moteus_tool.py --target 1 --dump-config

# 导出配置到文件
python3 utils/moteus_tool.py --target 1 --dump-config > config.txt

# 从文件导入配置
python3 utils/moteus_tool.py --target 1 --load-config config.txt
```

#### 3.1.3 校准

```bash
# 执行编码器校准
python3 utils/moteus_tool.py --target 1 --calibrate

# 执行索引校准
python3 utils/moteus_tool.py --target 1 --reindex

# 查看校准状态
python3 utils/moteus_tool.py --target 1 --query
```

#### 3.1.4 固件更新

```bash
# 更新固件
python3 utils/moteus_tool.py --target 1 --flash --firmware bazel-bin/fw/moteus.bin

# 更新引导加载程序
python3 utils/moteus_tool.py --target 1 --flash --bootloader bazel-bin/fw/can_bootloader.bin
```

#### 3.1.5 诊断

```bash
# 查看设备信息
python3 utils/moteus_tool.py --target 1 --info

# 查看故障信息
python3 utils/moteus_tool.py --target 1 --fault

# 清除故障
python3 utils/moteus_tool.py --target 1 --clear-fault

# 查看遥测数据
python3 utils/moteus_tool.py --target 1 --telemetry
```

### 3.2 tview.py

`tview.py`是图形化的遥测查看器。

```bash
# 启动遥测查看器
python3 utils/tview.py --devices=1

# 监控多个设备
python3 utils/tview.py --devices=1,2,3,4

# 指定CAN接口
python3 utils/tview.py --devices=1 --can=can0
```

**功能**：
- 实时显示位置、速度、扭矩
- 显示电流、电压、温度
- 绘制实时曲线
- 记录数据到文件

## 4. 最佳实践

### 4.1 性能优化

1. **使用异步接口**：避免阻塞
2. **批量操作**：减少通信开销
3. **合理设置控制频率**：通常1kHz足够
4. **使用紧凑格式**：减少数据量

### 4.2 错误处理

1. **设置超时**：避免无限等待
2. **重试机制**：处理临时通信错误
3. **故障检测**：监控设备状态
4. **安全停止**：异常时立即停止电机

### 4.3 安全考虑

1. **参数验证**：发送前检查参数范围
2. **速度限制**：限制最大速度
3. **扭矩限制**：限制最大扭矩
4. **看门狗**：通信超时自动停止

### 4.4 调试技巧

1. **使用日志**：记录通信数据
2. **使用tview**：实时监控状态
3. **使用decode_can_frame**：分析CAN帧
4. **逐步测试**：从简单命令开始

## 5. 常见问题

### 5.1 连接失败

**问题**：无法连接到设备

**解决方案**：
1. 检查CAN接口是否正确配置
2. 确认设备ID设置正确
3. 检查电源和接线
4. 使用`--list`查看设备

### 5.2 通信超时

**问题**：命令发送后无响应

**解决方案**：
1. 增加超时时间
2. 检查CAN总线负载
3. 检查波特率配置
4. 使用示波器检查信号

### 5.3 控制不稳定

**问题**：电机振荡或不稳定

**解决方案**：
1. 调整PID参数
2. 降低增益
3. 检查编码器校准
4. 检查机械连接

## 6. 总结

Moteus客户端库提供了：

- ✅ **Python库**：易于使用，适合快速开发
- ✅ **C++库**：高性能，适合嵌入式应用
- ✅ **命令行工具**：强大的配置和诊断功能
- ✅ **图形工具**：直观的遥测查看

选择合适的客户端库和工具，可以高效地开发和控制moteus设备。

# Moteus 快速开始指南

## 1. 5分钟快速上手

### 1.1 环境检查

```bash
# 检查Python版本
python3 --version  # 应该 >= 3.8

# 检查CAN接口
ip link show can0  # 应该显示CAN接口

# 如果CAN接口未启动
sudo ip link set can0 up type can bitrate 1000000 dbitrate 5000000 fd on
```

### 1.2 第一个程序

创建 `hello_moteus.py`：

```python
#!/usr/bin/env python3
import asyncio
import moteus

async def main():
    # 创建传输层
    transport = moteus.TransportFactory.create(
        can_bus='socketcan',
        channel='can0'
    )
    
    # 创建控制器（设备ID=1）
    controller = moteus.Controller(id=1, transport=transport)
    
    # 查询状态
    result = await controller.query()
    
    print(f"设备状态:")
    print(f"  位置: {result['position']:.3f} 弧度")
    print(f"  速度: {result['velocity']:.3f} 弧度/秒")
    print(f"  模式: {result['mode']}")

if __name__ == '__main__':
    asyncio.run(main())
```

运行：
```bash
python3 hello_moteus.py
```

### 1.3 控制电机

创建 `move_motor.py`：

```python
#!/usr/bin/env python3
import asyncio
import moteus

async def main():
    transport = moteus.TransportFactory.create(
        can_bus='socketcan',
        channel='can0'
    )
    controller = moteus.Controller(id=1, transport=transport)
    
    print("停止电机...")
    await controller.set_stop()
    await asyncio.sleep(1.0)
    
    print("移动到位置 1.0 弧度...")
    result = await controller.set_position(
        position=1.0,
        velocity=0.0,
        kp_scale=1.0,
        kd_scale=1.0,
    )
    
    print(f"当前位置: {result['position']:.3f} 弧度")
    
    await asyncio.sleep(2.0)
    
    print("移动到位置 -1.0 弧度...")
    result = await controller.set_position(
        position=-1.0,
        velocity=0.0,
    )
    
    print(f"当前位置: {result['position']:.3f} 弧度")
    
    await asyncio.sleep(2.0)
    
    print("停止电机...")
    await controller.set_stop()

if __name__ == '__main__':
    asyncio.run(main())
```

运行：
```bash
python3 move_motor.py
```

## 2. 命令行工具快速上手

### 2.1 使用moteus_tool

```bash
# 列出所有设备
python3 utils/moteus_tool.py --list

# 查询设备状态
python3 utils/moteus_tool.py --target 1 --query

# 读取配置
python3 utils/moteus_tool.py --target 1 --read-config servopos.position_max

# 写入配置
python3 utils/moteus_tool.py --target 1 --write-config servopos.position_max=10.0

# 停止电机
python3 utils/moteus_tool.py --target 1 --stop
```

### 2.2 使用tview查看遥测

```bash
# 启动遥测查看器
python3 utils/tview.py --devices=1
```

## 3. 校准电机

### 3.1 编码器校准

```bash
# 执行编码器校准
python3 utils/moteus_tool.py --target 1 --calibrate

# 校准过程会自动完成，大约需要30秒
# 校准完成后会显示结果
```

### 3.2 索引校准

```bash
# 如果需要绝对位置参考
python3 utils/moteus_tool.py --target 1 --reindex
```

## 4. 常见问题快速解决

### 4.1 无法连接到设备

**检查清单**：
1. CAN接口是否启动？
```bash
sudo ip link set can0 up type can bitrate 1000000 dbitrate 5000000 fd on
```

2. 设备ID是否正确？
```bash
python3 utils/moteus_tool.py --list
```

3. 电源是否正常？
- 检查电机电源
- 检查CAN总线终端电阻

### 4.2 电机不转动

**检查清单**：
1. 是否已校准？
```bash
python3 utils/moteus_tool.py --target 1 --query
# 查看calibrated状态
```

2. 是否有故障？
```bash
python3 utils/moteus_tool.py --target 1 --fault
# 如果有故障，清除故障
python3 utils/moteus_tool.py --target 1 --clear-fault
```

3. 参数是否正确？
```bash
python3 utils/moteus_tool.py --target 1 --read-config servo.max_current_A
# 确保电流限制不为0
```

### 4.3 控制不稳定

**调整PID参数**：
```bash
# 降低比例增益
python3 utils/moteus_tool.py --target 1 --write-config servo.pid_position.kp=1.0

# 降低微分增益
python3 utils/moteus_tool.py --target 1 --write-config servo.pid_position.kd=0.01
```

## 5. 下一步学习

完成快速上手后，建议按以下顺序学习：

1. **阅读项目概述** → `01-overview.md`
2. **理解软件架构** → `02-architecture.md`
3. **学习通信协议** → `06-communication-protocol.md`
4. **深入客户端库** → `07-client-library-guide.md`
5. **开始开发** → `08-development-guide.md`

## 6. 参考资源

- **官方文档**: `docs/reference.md`
- **协议文档**: `docs/protocol/`
- **使用指南**: `docs/guides/`
- **故障排除**: `docs/troubleshooting/`

## 7. 获取帮助

- **GitHub Issues**: https://github.com/mjbots/moteus/issues
- **邮件支持**: info@mjbots.com
- **社区讨论**: 查看项目README

祝你使用愉快！

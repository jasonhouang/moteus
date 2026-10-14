# Moteus 通信协议

## 1. 协议概述

Moteus使用基于CAN-FD的自定义通信协议，实现高效、可靠的设备间通信。

### 1.1 协议特点

- **基于CAN-FD**：利用CAN-FD的高速和大帧特性
- **多路复用**：单个CAN帧可包含多个寄存器读写操作
- **紧凑编码**：使用二进制编码，减少数据量
- **请求-响应模式**：支持同步和异步通信
- **广播支持**：支持遥测数据广播

### 1.2 协议层次

```
┌─────────────────────────────────┐
│   应用层 (Application Layer)    │  用户命令、状态查询
├─────────────────────────────────┤
│   协议层 (Protocol Layer)       │  Multiplex协议
├─────────────────────────────────┤
│   传输层 (Transport Layer)      │  CAN-FD
├─────────────────────────────────┤
│   物理层 (Physical Layer)       │  CAN总线
└─────────────────────────────────┘
```

## 2. CAN-FD基础

### 2.1 CAN-FD特性

相比传统CAN，CAN-FD具有以下优势：

| 特性 | 传统CAN | CAN-FD |
|------|---------|--------|
| 最大数据长度 | 8字节 | 64字节 |
| 最大波特率 | 1Mbps | 8Mbps（数据段） |
| 仲裁段波特率 | 最高1Mbps | 最高1Mbps |
| 错误检测 | CRC-15 | CRC-17/CRC-21 |

### 2.2 CAN帧结构

**标准CAN帧**：
```
┌─────┬──────────┬─────┬─────────┬─────┬─────┬─────┐
│ SOF │ ID(11bit)│ RTR │ DLC(4)  │DATA │ CRC │ EOF │
└─────┴──────────┴─────┴─────────┴─────┴─────┴─────┘
```

**CAN-FD帧**：
```
┌─────┬──────────┬─────┬─────┬─────────┬──────┬─────┬─────┐
│ SOF │ ID(11bit)│ RRS │ FDF │ DLC(4)  │ DATA │ CRC │ EOF │
└─────┴──────────┴─────┴─────┴─────────┴──────┴─────┴─────┘
```

**关键差异**：
- **FDF位**：标识CAN-FD帧
- **数据长度**：支持0-64字节（8, 12, 16, 20, 24, 32, 48, 64）
- **双波特率**：仲裁段和数据段可以使用不同波特率

### 2.3 Moteus CAN配置

**仲裁段波特率**：1Mbps
**数据段波特率**：5Mbps（可配置）

```python
# 配置示例
can_config = {
    'arbitration_bitrate': 1000000,  # 1Mbps
    'data_bitrate': 5000000,         # 5Mbps
}
```

## 3. Multiplex协议

### 3.1 协议帧格式

Moteus在CAN帧基础上定义了Multiplex协议层：

```
CAN帧结构：
┌──────────┬──────────┬──────────┬──────────┬──────────┐
│ CAN ID   │ Source   │ Dest     │ Command  │ Payload  │
│ (11bit)  │ (8bit)   │ (8bit)   │ (8bit)   │ (变长)    │
└──────────┴──────────┴──────────┴──────────┴──────────┘
```

**CAN ID编码**：
```
CAN ID = (Destination << 8) | Source

示例：
- Source = 1 (客户端)
- Destination = 5 (设备)
- CAN ID = (5 << 8) | 1 = 0x501
```

### 3.2 命令类型

Multiplex协议定义了多种命令类型：

| 命令 | 值 | 说明 |
|------|-----|------|
| kWriteRegisters | 0x00 | 写入寄存器 |
| kReadRegisters | 0x01 | 读取寄存器 |
| kReply | 0x02 | 响应 |
| kError | 0x03 | 错误 |
| kNOP | 0x04 | 空操作 |
| kQuery | 0x05 | 查询状态 |
| kPosition | 0x06 | 位置控制 |
| kVelocity | 0x07 | 速度控制 |
| kTorque | 0x08 | 扭矩控制 |
| kStayWithinBounds | 0x09 | 边界内保持 |

### 3.3 寄存器系统

Moteus使用寄存器系统管理配置和状态。

**寄存器地址空间**：
```
0x000 - 0x0FF: 系统寄存器（版本、ID等）
0x100 - 0x1FF: 电机配置寄存器
0x200 - 0x2FF: 控制参数寄存器
0x300 - 0x3FF: 状态寄存器
0x400 - 0x4FF: 遥测寄存器
0x500 - 0x5FF: 辅助端口寄存器
```

**寄存器数据类型**：
- `uint8`: 8位无符号整数
- `uint16`: 16位无符号整数
- `uint32`: 32位无符号整数
- `int8`: 8位有符号整数
- `int16`: 16位有符号整数
- `int32`: 32位有符号整数
- `float32`: 32位浮点数

### 3.4 写入寄存器命令

**命令格式**：
```
┌─────────┬──────────┬──────────┬──────────┬──────────┬──────────┐
│ Command │ Start    │ Length   │ Type     │ Data     │ ...      │
│ (0x00)  │ Addr(16) │ (8)      │ (8)      │ (变长)    │          │
└─────────┴──────────┴──────────┴──────────┴──────────┴──────────┘
```

**示例：写入设备ID**：
```python
# 寄存器地址：0x000 (id.id)
# 数据类型：uint8
# 新值：5

payload = bytes([
    0x00,        # Command: kWriteRegisters
    0x00, 0x00,  # Start address: 0x0000 (little-endian)
    0x01,        # Length: 1 register
    0x00,        # Type: uint8
    0x05,        # Data: new ID = 5
])
```

### 3.5 读取寄存器命令

**命令格式**：
```
┌─────────┬──────────┬──────────┐
│ Command │ Start    │ Length   │
│ (0x01)  │ Addr(16) │ (8)      │
└─────────┴──────────┴──────────┘
```

**示例：读取设备ID**：
```python
payload = bytes([
    0x01,        # Command: kReadRegisters
    0x00, 0x00,  # Start address: 0x0000
    0x01,        # Length: 1 register
])
```

**响应格式**：
```
┌─────────┬──────────┬──────────┬──────────┐
│ Command │ Length   │ Data     │ ...      │
│ (0x02)  │ (8)      │ (变长)    │          │
└─────────┴──────────┴──────────┴──────────┘
```

### 3.6 位置控制命令

**命令格式**：
```
┌─────────┬──────────┬──────────┬──────────┬──────────┬──────────┬──────────┐
│ Command │ Position │ Velocity │ Feedfwd  │ KP Scale │ KD Scale │ Max Tq   │
│ (0x06)  │ (f32)    │ (f32)    │ (f32)    │ (f32)    │ (f32)    │ (f32)    │
└─────────┴──────────┴──────────┴──────────┴──────────┴──────────┴──────────┘
```

**参数说明**：
- `Position`: 目标位置（弧度）
- `Velocity`: 目标速度（弧度/秒）
- `Feedforward`: 前馈扭矩（Nm）
- `KP Scale`: 比例增益缩放（0.0-1.0）
- `KD Scale`: 微分增益缩放（0.0-1.0）
- `Max Torque`: 最大扭矩限制（Nm）

**示例**：
```python
import struct

payload = struct.pack('<Bffffff',
    0x06,        # Command: kPosition
    1.5708,      # Position: π/2 弧度
    0.0,         # Velocity: 0
    0.0,         # Feedforward: 0
    1.0,         # KP Scale: 100%
    1.0,         # KD Scale: 100%
    1.0,         # Max Torque: 1.0 Nm
)
```

### 3.7 速度控制命令

**命令格式**：
```
┌─────────┬──────────┬──────────┬──────────┬──────────┬──────────┐
│ Command │ Velocity │ Feedfwd  │ KP Scale │ KD Scale │ Max Tq   │
│ (0x07)  │ (f32)    │ (f32)    │ (f32)    │ (f32)    │ (f32)    │
└─────────┴──────────┴──────────┴──────────┴──────────┴──────────┘
```

**注意**：速度控制模式下，KP Scale通常为0（禁用位置环）。

### 3.8 扭矩控制命令

**命令格式**：
```
┌─────────┬──────────┬──────────┐
│ Command │ Torque   │ Max Tq   │
│ (0x08)  │ (f32)    │ (f32)    │
└─────────┴──────────┴──────────┘
```

### 3.9 查询命令

**命令格式**：
```
┌─────────┐
│ Command │
│ (0x05)  │
└─────────┘
```

**响应**：返回当前状态，包括位置、速度、扭矩、电流等。

## 4. 数据编码

### 4.1 数值编码

**整数编码**：
- 使用小端序（Little-Endian）
- 示例：`uint16(0x1234)` → `bytes([0x34, 0x12])`

**浮点数编码**：
- IEEE 754单精度浮点（32位）
- 小端序
- 示例：`float32(3.14)` → `bytes([...])`

```python
import struct

# 整数编码
value = 0x1234
encoded = struct.pack('<H', value)  # Little-endian uint16

# 浮点数编码
value = 3.14159
encoded = struct.pack('<f', value)  # Little-endian float32
```

### 4.2 紧凑格式

为减少数据量，协议支持紧凑格式：

**位置/速度紧凑格式**：
- 使用`int32`表示
- 分辨率：2^-16 弧度/位
- 范围：±32768 弧度

```python
# 标准格式（float32）
position = 1.5708  # 弧度
encoded = struct.pack('<f', position)  # 4字节

# 紧凑格式（int32）
position_int = int(position * 65536)  # 转换为整数
encoded = struct.pack('<i', position_int)  # 4字节
```

### 4.3 位域编码

某些寄存器使用位域编码多个值：

```python
# 示例：状态寄存器
# Bit 0-3: 模式
# Bit 4-7: 故障代码
# Bit 8: 使能状态
# Bit 9: 校准状态

status = 0x1234
mode = status & 0x0F         # 提取位0-3
fault = (status >> 4) & 0x0F # 提取位4-7
enabled = (status >> 8) & 0x01
calibrated = (status >> 9) & 0x01
```

## 5. 传输层实现

### 5.1 SocketCAN（Linux）

**特点**：
- Linux原生支持
- 高性能
- 支持CAN-FD

**使用示例**：
```python
import socket
import struct

# 创建SocketCAN套接字
sock = socket.socket(socket.AF_CAN, socket.SOCK_RAW, socket.CAN_RAW)
sock.bind(('can0',))

# 启用CAN-FD
sock.setsockopt(socket.SOL_CAN_RAW, socket.CAN_RAW_FD_FRAMES, 1)

# 发送CAN帧
can_id = 0x501  # Destination=5, Source=1
data = bytes([0x05])  # Query command
can_frame = struct.pack('<IB8s', can_id, 8, data)
sock.send(can_frame)

# 接收CAN帧
can_frame = sock.recv(72)  # CAN-FD最大帧大小
can_id, dlc, data = struct.unpack('<IB8s', can_frame[:16])
```

### 5.2 fdcanusb（USB适配器）

**特点**：
- mjbots USB-CAN适配器
- 批量传输
- 跨平台支持

**使用示例**：
```python
import serial

# 打开串口
ser = serial.Serial('/dev/ttyACM0', baudrate=115200)

# 发送命令
can_id = 0x501
data = bytes([0x05])
frame = f"can send {can_id:x} {data.hex()}\n".encode()
ser.write(frame)

# 接收响应
response = ser.readline()
# 解析响应...
```

### 5.3 python-can（通用）

**特点**：
- 支持多种硬件
- 统一的API
- 易于使用

**使用示例**：
```python
import can

# 创建总线
bus = can.interface.Bus(channel='can0', interface='socketcan')

# 发送消息
msg = can.Message(
    arbitration_id=0x501,
    data=[0x05],
    is_fd=True
)
bus.send(msg)

# 接收消息
msg = bus.recv(timeout=1.0)
if msg:
    print(f"Received: {msg.arbitration_id:x}, {msg.data.hex()}")
```

## 6. 协议实现细节

### 6.1 命令编码流程

```python
def encode_position_command(controller_id, client_id, position, velocity):
    """编码位置控制命令"""
    
    # 1. 构建CAN ID
    can_id = (controller_id << 8) | client_id
    
    # 2. 构建命令payload
    payload = struct.pack('<Bffffff',
        0x06,        # Command: kPosition
        position,    # Position
        velocity,    # Velocity
        0.0,         # Feedforward
        1.0,         # KP Scale
        1.0,         # KD Scale
        0.0,         # Max Torque (0=无限制)
    )
    
    # 3. 构建CAN帧
    can_frame = {
        'id': can_id,
        'data': payload,
        'is_fd': True,
    }
    
    return can_frame
```

### 6.2 响应解码流程

```python
def decode_response(can_frame):
    """解码响应帧"""
    
    # 1. 提取CAN ID
    can_id = can_frame['id']
    source = can_id & 0xFF
    dest = (can_id >> 8) & 0xFF
    
    # 2. 提取命令类型
    data = can_frame['data']
    command = data[0]
    
    # 3. 根据命令类型解码
    if command == 0x02:  # kReply
        # 解码响应数据
        response = {}
        offset = 1
        
        # 解析寄存器值
        while offset < len(data):
            reg_addr = struct.unpack_from('<H', data, offset)[0]
            offset += 2
            
            reg_type = data[offset]
            offset += 1
            
            # 根据类型解码
            if reg_type == 0x00:  # uint8
                value = data[offset]
                offset += 1
            elif reg_type == 0x04:  # float32
                value = struct.unpack_from('<f', data, offset)[0]
                offset += 4
            
            response[reg_addr] = value
        
        return response
    
    return None
```

### 6.3 错误处理

**错误类型**：
- 通信超时
- CRC错误
- 无效命令
- 设备故障

**错误处理示例**：
```python
def send_command_with_retry(transport, command, max_retries=3):
    """带重试的命令发送"""
    
    for attempt in range(max_retries):
        try:
            transport.send(command)
            response = transport.receive(timeout=1.0)
            
            if response and response['command'] != 0x03:  # 非错误响应
                return response
            
        except TimeoutError:
            if attempt == max_retries - 1:
                raise
            continue
    
    return None
```

## 7. 高级特性

### 7.1 批量操作

可以在单个CAN帧中执行多个寄存器操作：

```python
def encode_batch_write(registers):
    """编码批量写入命令"""
    
    payload = bytearray([0x00])  # kWriteRegisters
    
    for reg_addr, reg_type, value in registers:
        payload.extend(struct.pack('<H', reg_addr))  # 地址
        payload.append(reg_type)                      # 类型
        
        # 编码值
        if reg_type == 0x00:  # uint8
            payload.append(value)
        elif reg_type == 0x04:  # float32
            payload.extend(struct.pack('<f', value))
    
    return bytes(payload)
```

### 7.2 广播通信

使用特殊的CAN ID进行广播：

```python
# 广播地址
BROADCAST_ID = 0x7FF

# 发送广播命令
can_id = (BROADCAST_ID << 8) | client_id
# 所有设备都会接收此命令
```

### 7.3 遥测数据流

设备可以主动发送遥测数据：

```python
# 遥测帧格式
telemetry_frame = {
    'id': (0x7FF << 8) | device_id,  # 广播
    'data': encode_telemetry_data(),
    'is_fd': True,
}

def encode_telemetry_data():
    """编码遥测数据"""
    return struct.pack('<Bfff',
        0x0A,        # 遥测命令
        position,    # 当前位置
        velocity,    # 当前速度
        torque,      # 当前扭矩
    )
```

## 8. 协议调试

### 8.1 CAN帧解码工具

项目提供CAN帧解码工具：

```bash
# 使用decode_can_frame.py
python3 utils/decode_can_frame.py --frame "501#060000C842000000000000803F0000803F00000000"
```

**输出示例**：
```
CAN Frame:
  ID: 0x501 (Source=1, Dest=5)
  Command: kPosition (0x06)
  Position: 100.0 rad
  Velocity: 0.0 rad/s
  Feedforward: 0.0 Nm
  KP Scale: 1.0
  KD Scale: 1.0
  Max Torque: 0.0 Nm
```

### 8.2 协议分析

使用Wireshark分析CAN通信：

1. 安装Wireshark和CAN插件
2. 捕获CAN接口流量
3. 应用Moteus协议解码器

### 8.3 日志记录

```python
import logging

# 配置日志
logging.basicConfig(level=logging.DEBUG)
logger = logging.getLogger('moteus.protocol')

# 记录命令
logger.debug(f"Sending command: {command.hex()}")

# 记录响应
logger.debug(f"Received response: {response.hex()}")
```

## 9. 性能优化

### 9.1 减少通信延迟

- 使用CAN-FD数据段高波特率
- 批量操作减少帧数
- 使用紧凑格式减少数据量

### 9.2 提高吞吐量

- 异步通信
- 流水线操作
- 减少等待时间

### 9.3 带宽优化

```python
# 使用紧凑格式
def encode_compact_position(position):
    """使用紧凑格式编码位置"""
    # 分辨率: 2^-16 rad/bit
    # 范围: ±32768 rad
    position_int = int(position * 65536)
    return struct.pack('<i', position_int)  # 4字节 vs float32 4字节
    # 但整数运算更快
```

## 10. 安全考虑

### 10.1 命令验证

- 参数范围检查
- 速率限制
- 权限验证

### 10.2 错误恢复

- 通信超时自动停止
- 故障自动保护
- 看门狗机制

### 10.3 数据完整性

- CRC校验
- 序列号检查
- 重传机制

## 11. 总结

Moteus通信协议提供了：

- ✅ **高效传输**：CAN-FD高速大帧
- ✅ **灵活命令**：多种控制模式
- ✅ **可靠通信**：错误检测和恢复
- ✅ **易于扩展**：寄存器系统
- ✅ **完善工具**：调试和分析工具

理解协议是实现自定义客户端和高级应用的基础。

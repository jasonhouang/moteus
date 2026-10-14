# Moteus 软件架构

## 1. 整体架构概览

Moteus系统采用分层架构设计，从底层硬件抽象到上层应用接口，层次清晰，模块化程度高。

```
┌─────────────────────────────────────────────────────────┐
│              应用层 (Application Layer)                  │
│         (用户程序、机器人控制算法)                         │
└────────────────┬────────────────────────────────────────┘
                 │
┌────────────────▼────────────────────────────────────────┐
│           客户端库层 (Client Library Layer)              │
│    Python / C++ / Rust 客户端库                          │
│    - 协议编解码                                           │
│    - 传输层抽象                                           │
│    - 异步/同步接口                                        │
└────────────────┬────────────────────────────────────────┘
                 │
┌────────────────▼────────────────────────────────────────┐
│           通信层 (Communication Layer)                   │
│         CAN-FD / fdcanusb / socketcan                   │
│         - 多路复用协议                                     │
│         - 设备发现                                         │
│         - 数据传输                                         │
└────────────────┬────────────────────────────────────────┘
                 │
┌────────────────▼────────────────────────────────────────┐
│           固件层 (Firmware Layer)                        │
│    ┌──────────────────────────────────────────────┐    │
│    │  MoteusController (主控制器)                  │    │
│    │  ├─ BldcServo (电机伺服控制)                  │    │
│    │  ├─ MotorPosition (位置反馈)                  │    │
│    │  ├─ MotorDriver (电机驱动)                    │    │
│    │  ├─ AuxPort (辅助端口)                        │    │
│    │  └─ Communication (通信处理)                  │    │
│    └──────────────────────────────────────────────┘    │
└────────────────┬────────────────────────────────────────┘
                 │
┌────────────────▼────────────────────────────────────────┐
│        硬件抽象层 (Hardware Abstraction Layer)           │
│    ┌──────────────────────────────────────────────┐    │
│    │  MoteusHw (硬件定义)                          │    │
│    │  ├─ 引脚配置                                   │    │
│    │  ├─ ADC配置                                    │    │
│    │  ├─ PWM配置                                    │    │
│    │  └─ 外设初始化                                  │    │
│    └──────────────────────────────────────────────┘    │
└────────────────┬────────────────────────────────────────┘
                 │
┌────────────────▼────────────────────────────────────────┐
│              硬件层 (Hardware Layer)                     │
│         STM32G4 MCU + 功率级 + 传感器                    │
└─────────────────────────────────────────────────────────┘
```

## 2. 固件架构详解

### 2.1 核心组件

#### 2.1.1 MoteusController（主控制器）

**文件位置**：`fw/moteus_controller.h`

MoteusController是整个固件的核心协调器，负责：
- 初始化所有子系统
- 协调各模块之间的交互
- 管理配置和遥测
- 处理通信协议

**主要职责**：
```cpp
class MoteusController {
 public:
  MoteusController(Pool*, PersistentConfig*, CommandManager*,
                   TelemetryManager*, MicroServer*,
                   MultiTransportDatagramServer*,
                   ClockManager*, SystemInfo*,
                   MillisecondTimer*, FirmwareInfo*, Uuid*);
  
  void Start();           // 启动控制器
  void Poll();            // 主循环轮询
  void PollMillisecond(); // 毫秒级轮询
  
  BldcServo* bldc_servo();  // 获取电机伺服实例
};
```

**工作流程**：
1. 构造时接收所有依赖组件
2. Start()初始化所有子系统
3. Poll()在主循环中持续调用，处理实时任务
4. PollMillisecond()处理毫秒级定时任务

#### 2.1.2 BldcServo（电机伺服控制）

**文件位置**：`fw/bldc_servo.h`, `fw/bldc_servo.cc`

BldcServo是电机控制的核心实现，负责：
- 实现FOC（磁场定向控制）算法
- 执行PID控制循环
- 处理多种控制模式
- 管理电机状态和故障

**关键组件**：
```cpp
class BldcServo {
 public:
  struct Options {
    PinName pwm1, pwm2, pwm3;      // PWM输出引脚
    PinName current1, current2, current3;  // 电流采样引脚
    PinName vsense, tsense, msense;  // 电压、温度采样
    // ...
  };
  
  void Start();
  void Command(const CommandData&);  // 接收控制命令
  
  const Status& status() const;      // 获取当前状态
  const Config& config() const;      // 获取配置
  const Control& control() const;    // 获取控制参数
};
```

**控制模式**：
- **STOPPED**：电机停止，不输出扭矩
- **FAULT**：故障状态，需要清除
- **ENABLING**：正在启用
- **CALIBRATING**：校准中
- **CALIBRATION_COMPLETE**：校准完成
- **OPERATING**：正常运行

#### 2.1.3 MotorPosition（位置反馈系统）

**文件位置**：`fw/motor_position.h`

MotorPosition负责处理所有位置反馈相关功能：
- 编码器读取和处理
- 位置、速度计算
- 多编码器融合
- 绝对编码器支持

**支持的编码器类型**：
- AS5047（SPI绝对编码器）
- AKIM2（模拟编码器）
- BISS-C（数字绝对编码器）
- CUI AMT21/22（增量/绝对编码器）
- Hall传感器

**关键功能**：
```cpp
class MotorPosition {
 public:
  struct Status {
    float position;      // 当前位置（弧度）
    float velocity;      // 当前速度（弧度/秒）
    uint32_t index_count; // 索引计数
    bool index_found;    // 是否找到索引
  };
  
  const Status& status() const;
  void Poll();  // 轮询更新位置
};
```

#### 2.1.4 MotorDriver（电机驱动）

**文件位置**：`fw/motor_driver.h`, `fw/drv8323.h`

MotorDriver负责：
-  gate driver控制（DRV8323）
- PWM信号生成
- 电流采样
- 保护功能（过流、过温）

**DRV8323 Gate Driver**：
```cpp
class Drv8323 {
 public:
  struct Status {
    bool fault;
    bool overtemperature;
    bool overcurrent;
    // ...
  };
  
  void Configure();
  void Enable();
  void Disable();
  const Status& status() const;
};
```

#### 2.1.5 AuxPort（辅助端口）

**文件位置**：`fw/aux_port.h`

AuxPort提供灵活的I/O扩展能力：
- GPIO输入/输出
- I2C通信
- SPI通信
- UART通信
- 模拟输入

**配置选项**：
```cpp
struct AuxPortConfig {
  enum class Function {
    NONE,
    GPIO,
    I2C,
    SPI,
    UART,
    // ...
  };
  
  Function function;
  // 具体配置参数...
};
```

### 2.2 控制算法架构

#### 2.2.1 FOC（磁场定向控制）

**文件位置**：`fw/foc.h`, `fw/foc.cc`

FOC是电机控制的核心算法，实现：
- Clarke变换（三相→两相）
- Park变换（静止→旋转坐标系）
- 电流环PI控制
- 逆Park变换
- SVPWM（空间矢量调制）

**算法流程**：
```
三相电流 (ia, ib, ic)
    ↓ Clarke变换
两相电流 (iα, iβ)
    ↓ Park变换
旋转坐标系电流 (id, iq)
    ↓ PI控制
电压指令 (vd, vq)
    ↓ 逆Park变换
两相电压 (vα, vβ)
    ↓ SVPWM
PWM占空比 (duty_a, duty_b, duty_c)
```

#### 2.2.2 PID控制器

**文件位置**：`fw/pid.h`

实现标准的PID控制，支持：
- 位置环PID
- 速度环PID
- 电流环PI
- 积分限幅
- 微分滤波

**PID结构**：
```cpp
struct PidConfig {
  float kp;          // 比例增益
  float ki;          // 积分增益
  float kd;          // 微分增益
  float iratelimit;  // 积分变化率限制
  float ilimit;      // 积分限幅
  float kff;         // 前馈增益
  // ...
};

class Pid {
 public:
  float Apply(float error, float rate, float dt);
  void Reset();
};
```

#### 2.2.3 位置控制模式

**文件位置**：`fw/bldc_servo_position.h`

位置控制模式是最常用的控制模式，实现：
- 位置环 → 速度环 → 电流环的级联控制
- 轨迹规划（梯形/S曲线）
- 位置限制
- 速度限制
- 扭矩限制

**控制流程**：
```
目标位置 (position_cmd)
    ↓ 位置环PID
速度指令 (velocity_cmd)
    ↓ 速度环PID
电流指令 (current_cmd)
    ↓ 电流环PI
电压指令 (voltage_cmd)
    ↓ FOC
PWM输出
```

### 2.3 通信架构

#### 2.3.1 CAN-FD通信层

**文件位置**：`fw/fdcan.h`, `fw/fdcan.cc`

实现CAN-FD协议栈：
- 消息收发
- 过滤器配置
- 错误处理
- 时间戳管理

**关键特性**：
- 支持标准CAN和CAN-FD格式
- 最高8字节数据（CAN）或64字节数据（CAN-FD）
- 硬件过滤器减少CPU负载
- DMA传输提高效率

#### 2.3.2 Multiplex协议

**文件位置**：`lib/cpp/mjbots/moteus/moteus_multiplex.h`

Multiplex是moteus的自定义高层协议，提供：
- 设备寻址
- 命令/响应格式
- 遥测数据流
- 配置读写

**协议帧格式**：
```
┌─────────┬─────────┬─────────┬─────────┬─────────┐
│ CAN ID  │ Source  │ Dest    │ Command │ Payload │
│ (11bit) │ (8bit)  │ (8bit)  │ (8bit)  │ (变长)   │
└─────────┴─────────┴─────────┴─────────┴─────────┘
```

**命令类型**：
- `kWriteRegisters`：写入配置寄存器
- `kReadRegisters`：读取配置寄存器
- `kQuery`：查询状态
- `kPosition`：位置控制命令
- `kVelocity`：速度控制命令
- `kTorque`：扭矩控制命令

#### 2.3.3 传输层抽象

**Python客户端**：`lib/python/moteus/transport_*.py`

支持多种传输方式：
- `fdcanusb`：通过USB-CAN适配器
- `socketcan`：Linux SocketCAN接口
- `pythoncan`：python-can库支持的各种接口

**传输层接口**：
```python
class Transport:
    async def send(self, data: bytes) -> None:
        """发送数据"""
        pass
    
    async def receive(self) -> bytes:
        """接收数据"""
        pass
```

### 2.4 配置管理系统

#### 2.4.1 PersistentConfig

**文件位置**：`mjlib/micro/persistent_config.h`（外部库）

负责管理持久化配置：
- 从Flash读取配置
- 写入配置到Flash
- 配置版本管理
- 默认值处理

**使用方式**：
```cpp
class MyComponent {
 public:
  struct Config {
    float param1 = 1.0f;
    int param2 = 10;
    
    template <typename Visitor>
    void Accept(Visitor& v) {
      v.Visit(MJ_NVP(param1), &param1);
      v.Visit(MJ_NVP(param2), &param2);
    }
  };
  
  MyComponent(PersistentConfig* config) {
    config->Register("my_component", &config_);
  }
  
 private:
  Config config_;
};
```

#### 2.4.2 TelemetryManager

**文件位置**：`mjlib/micro/telemetry_manager.h`（外部库）

管理遥测数据流：
- 注册遥测通道
- 数据序列化
- 周期性发送

**使用方式**：
```cpp
class MyComponent {
 public:
  MyComponent(TelemetryManager* telemetry) {
    telemetry->Register("my_telemetry", &telemetry_data_);
  }
  
 private:
  struct TelemetryData {
    float value1;
    int value2;
    
    template <typename Visitor>
    void Accept(Visitor& v) {
      v.Visit(MJ_NVP(value1), &value1);
      v.Visit(MJ_NVP(value2), &value2);
    }
  } telemetry_data_;
};
```

### 2.5 任务调度架构

#### 2.5.1 主循环调度

固件采用协作式多任务调度，主循环结构：

```cpp
int main() {
  // 初始化所有组件
  MoteusController controller(...);
  controller.Start();
  
  // 主循环
  while (true) {
    controller.Poll();           // 处理实时任务
    controller.PollMillisecond(); // 处理毫秒级任务
  }
}
```

#### 2.5.2 中断驱动任务

**高优先级中断**：
- ADC采样中断（~20kHz）
- PWM更新中断
- 电流环控制

**中优先级中断**：
- CAN消息接收
- 编码器读取

**低优先级中断**：
- UART通信
- I2C通信

#### 2.5.3 时间管理

**文件位置**：`fw/millisecond_timer.h`

提供精确的时间管理：
- 毫秒级计时
- 微秒级计时
- 超时检测
- 周期任务调度

```cpp
class MillisecondTimer {
 public:
  uint32_t ms() const;      // 当前毫秒时间戳
  uint32_t us() const;      // 当前微秒时间戳
  bool IsExpired(uint32_t deadline) const;
  uint32_t MakeDeadline(uint32_t ms_from_now) const;
};
```

## 3. 客户端库架构

### 3.1 Python客户端架构

**文件位置**：`lib/python/moteus/moteus.py`

#### 3.1.1 核心类

**Controller类**：
```python
class Controller:
    def __init__(self, id: int, transport: Transport):
        """
        id: CAN设备ID
        transport: 传输层实例
        """
        pass
    
    async def set_stop(self) -> dict:
        """停止电机"""
        pass
    
    async def set_position(
        self,
        position: float = 0.0,
        velocity: float = 0.0,
        feedforward_torque: float = 0.0,
        kp_scale: float = 1.0,
        kd_scale: float = 1.0,
        maximum_torque: float = 0.0,
    ) -> dict:
        """位置控制命令"""
        pass
    
    async def set_velocity(
        self,
        velocity: float,
        feedforward_torque: float = 0.0,
        kp_scale: float = 0.0,
        kd_scale: float = 1.0,
        maximum_torque: float = 0.0,
    ) -> dict:
        """速度控制命令"""
        pass
    
    async def set_torque(
        self,
        torque: float,
        maximum_torque: float = 0.0,
    ) -> dict:
        """扭矩控制命令"""
        pass
    
    async def query(self) -> dict:
        """查询当前状态"""
        pass
```

**返回数据结构**：
```python
{
    'mode': int,              # 当前模式
    'position': float,        # 当前位置（ revolutions）
    'velocity': float,        # 当前速度（revolutions/s）
    'torque': float,          # 当前扭矩（Nm）
    'q_current': float,       # q轴电流（A）
    'd_current': float,       # d轴电流（A）
    'reeadcode': int,         # 读编码器值
    'servo_stats': {...},     # 伺服统计数据
    # ...
}
```

#### 3.1.2 传输层

**TransportFactory**：
```python
class TransportFactory:
    @staticmethod
    def create(
        can_bus: str = 'socketcan',
        channel: str = 'can0',
        bitrate: int = 1000000,
        fdcanusb: str = None,
    ) -> Transport:
        """创建传输层实例"""
        pass
```

**支持的传输类型**：
- `socketcan`：Linux原生SocketCAN
- `fdcanusb`：mjbots USB-CAN适配器
- `pythoncan`：python-can库支持的各种接口

### 3.2 C++客户端架构

**文件位置**：`lib/cpp/mjbots/moteus/moteus.h`

#### 3.2.1 核心类

**Controller类**：
```cpp
class Controller {
 public:
  struct Options {
    uint32_t id = 1;
    Transport* transport = nullptr;
  };
  
  Controller(const Options& options);
  
  // 阻塞式接口
  Result Query();
  Result SetStop();
  Result SetPosition(const PositionCommand& cmd);
  Result SetVelocity(const VelocityCommand& cmd);
  Result SetTorque(const TorqueCommand& cmd);
  
  // 异步接口
  void AsyncQuery(Callback callback);
  void AsyncSetStop(Callback callback);
  void AsyncSetPosition(const PositionCommand& cmd, Callback callback);
  
 private:
  class Impl;
  std::unique_ptr<Impl> impl_;
};
```

**命令结构**：
```cpp
struct PositionCommand {
  float position = 0.0;
  float velocity = 0.0;
  float feedforward_torque = 0.0;
  float kp_scale = 1.0;
  float kd_scale = 1.0;
  float maximum_torque = 0.0;
};

struct VelocityCommand {
  float velocity = 0.0;
  float feedforward_torque = 0.0;
  float kp_scale = 0.0;
  float kd_scale = 1.0;
  float maximum_torque = 0.0;
};

struct TorqueCommand {
  float torque = 0.0;
  float maximum_torque = 0.0;
};
```

#### 3.2.2 传输层

**Transport接口**：
```cpp
class Transport {
 public:
  virtual ~Transport() = default;
  
  virtual void Send(const CanFrame& frame) = 0;
  virtual bool Receive(CanFrame* frame, uint32_t timeout_ms) = 0;
};

// SocketCAN实现
class SocketCanTransport : public Transport {
 public:
  SocketCanTransport(const std::string& interface);
  void Send(const CanFrame& frame) override;
  bool Receive(CanFrame* frame, uint32_t timeout_ms) override;
};

// fdcanusb实现
class FdcanusbTransport : public Transport {
 public:
  FdcanusbTransport(const std::string& device);
  void Send(const CanFrame& frame) override;
  bool Receive(CanFrame* frame, uint32_t timeout_ms) override;
};
```

### 3.3 客户端库对比

| 特性 | Python | C++ | Rust |
|------|--------|-----|------|
| 异步支持 | ✅ asyncio | ✅ 回调 | ✅ async/await |
| 阻塞接口 | ✅ | ✅ | ✅ |
| 类型安全 | ❌ | ✅ | ✅ |
| 性能 | 中 | 高 | 高 |
| 易用性 | 高 | 中 | 中 |
| 平台支持 | 全平台 | Linux为主 | 全平台 |

## 4. 数据流架构

### 4.1 控制命令流

```
用户程序
    ↓
客户端库 (Controller.set_position)
    ↓
协议编解码 (Multiplex编码)
    ↓
传输层 (Transport.send)
    ↓
CAN-FD总线
    ↓
固件CAN驱动 (FDCAN接收中断)
    ↓
协议解析 (Multiplex解码)
    ↓
命令分发 (MoteusController)
    ↓
电机控制 (BldcServo.Command)
    ↓
PID控制 (位置环→速度环→电流环)
    ↓
FOC算法 (Park/Clarke/SVPWM)
    ↓
PWM输出 (定时器中断)
    ↓
功率级 (Gate Driver + MOSFET)
    ↓
电机
```

### 4.2 状态反馈流

```
电机
    ↓
电流采样 (ADC中断)
    ↓
位置采样 (编码器读取)
    ↓
状态计算 (BldcServo)
    ↓
遥测数据打包 (TelemetryManager)
    ↓
协议编解码 (Multiplex编码)
    ↓
CAN-FD发送
    ↓
传输层接收 (Transport.receive)
    ↓
协议解析 (Multiplex解码)
    ↓
客户端库 (Controller.query返回)
    ↓
用户程序
```

### 4.3 配置管理流

```
用户配置 (moteus_tool.py或客户端库)
    ↓
协议编解码 (WriteRegisters命令)
    ↓
传输层发送
    ↓
CAN-FD接收
    ↓
配置解析 (PersistentConfig)
    ↓
配置分发 (各组件接收配置)
    ↓
Flash存储 (持久化)
    ↓
应用新配置
```

## 5. 模块依赖关系

### 5.1 固件模块依赖

```
MoteusController
    ├─ BldcServo
    │   ├─ Foc (FOC算法)
    │   ├─ Pid (PID控制器)
    │   ├─ MotorDriver (电机驱动)
    │   ├─ MotorPosition (位置反馈)
    │   └─ AuxPort (辅助端口)
    ├─ ClockManager (时钟管理)
    ├─ SystemInfo (系统信息)
    ├─ FirmwareInfo (固件信息)
    └─ Uuid (设备唯一ID)
```

### 5.2 客户端库依赖

**Python**：
```
moteus.Controller
    ├─ moteus.Protocol (协议编解码)
    ├─ moteus.Transport (传输层抽象)
    │   ├─ FdcanusbDevice
    │   ├─ SocketCanDevice
    │   └─ PythonCanDevice
    └─ moteus.Reader (数据解析)
```

**C++**：
```
mjbots::moteus::Controller
    ├─ mjbots::moteus::Protocol (协议编解码)
    ├─ mjbots::moteus::Transport (传输层抽象)
    │   ├─ SocketCanTransport
    │   └─ FdcanusbTransport
    └─ mjbots::base (基础工具)
```

## 6. 扩展点

### 6.1 添加新的控制模式

1. 在 `fw/bldc_servo_structs.h` 定义新的命令结构
2. 在 `fw/bldc_servo.cc` 实现控制逻辑
3. 在协议层添加命令支持
4. 在客户端库添加对应接口

### 6.2 添加新的编码器类型

1. 在 `fw/motor_position.h` 添加编码器接口
2. 实现编码器驱动（SPI/I2C/UART等）
3. 在 `AuxPort` 中注册新编码器
4. 添加配置参数支持

### 6.3 添加新的传输层

1. 实现 `Transport` 接口
2. 处理CAN帧的序列化和反序列化
3. 在 `TransportFactory` 中注册
4. 添加配置参数

## 7. 性能优化点

### 7.1 固件优化

- **中断优先级**：合理设置中断优先级，确保关键任务及时响应
- **DMA使用**：ADC采样、CAN通信使用DMA减少CPU负载
- **内存池**：使用内存池避免动态内存分配
- **零拷贝**：CAN消息处理尽量使用零拷贝

### 7.2 客户端优化

- **批量查询**：多设备时使用批量查询减少通信开销
- **异步处理**：使用异步接口避免阻塞
- **连接池**：复用传输层连接
- **数据压缩**：遥测数据使用压缩格式

## 8. 安全机制

### 8.1 固件安全

- **看门狗**：硬件看门狗防止死机
- **过流保护**：硬件和软件双重过流保护
- **过温保护**：温度监控和降额
- **过压保护**：电压监控和关断
- **通信超时**：通信超时自动停止

### 8.2 客户端安全

- **参数验证**：发送前验证参数范围
- **超时处理**：通信超时重试和错误处理
- **异常处理**：完善的异常捕获和处理
- **线程安全**：多线程环境下的安全访问

## 9. 调试支持

### 9.1 固件调试

- **Telemetry流**：实时遥测数据输出
- **Board Debug**：板级调试接口
- **GDB支持**：SWD调试接口
- **日志系统**：分级日志输出

### 9.2 客户端调试

- **详细日志**：通信日志记录
- **帧解码工具**：CAN帧解码工具
- **遥测查看器**：tview.py实时查看
- **性能分析**：延迟和吞吐量分析

## 10. 架构演进

### 10.1 当前架构优势

- 模块化设计，易于理解和维护
- 清晰的层次结构，依赖关系明确
- 良好的抽象，支持多种硬件和传输方式
- 完善的测试覆盖

### 10.2 未来演进方向

- 支持更多编码器类型
- 支持更多控制模式
- 改进的通信协议（更高效率）
- 更强大的诊断和调试功能
- 支持多电机协同控制

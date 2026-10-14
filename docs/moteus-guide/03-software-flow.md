# Moteus 软件流程

## 1. 固件启动流程

### 1.1 启动序列

固件启动遵循严格的初始化顺序，确保所有硬件和软件组件正确初始化。

```
Reset Vector (STM32G4)
    ↓
SystemInit() - 系统初始化
    ├─ 配置时钟系统 (HSE → PLL → SYSCLK)
    ├─ 配置Flash等待周期
    └─ 配置电源管理
    ↓
main() - 主函数入口
    ↓
HAL初始化
    ├─ HAL_Init() - HAL库初始化
    ├─ SystemClock_Config() - 系统时钟配置
    └─ 外设时钟使能
    ↓
内存池初始化
    ├─ Pool初始化 - 动态内存管理
    └─ 内存区域划分
    ↓
核心服务初始化
    ├─ PersistentConfig - 持久化配置管理
    │   └─ 从Flash加载配置
    ├─ CommandManager - 命令管理器
    ├─ TelemetryManager - 遥测管理器
    └─ MillisecondTimer - 毫秒计时器
    ↓
硬件抽象层初始化
    ├─ MoteusHw - 硬件引脚配置
    │   ├─ GPIO配置
    │   ├─ ADC配置
    │   ├─ PWM定时器配置
    │   └─ DMA配置
    ├─ AuxADC - 辅助ADC
    ├─ AuxPort - 辅助端口
    └─ MotorDriver - 电机驱动
    ↓
传感器初始化
    ├─ MotorPosition - 位置反馈
    │   ├─ 编码器初始化 (AS5047/AKIM2/BISS-C等)
    │   ├─ Hall传感器初始化
    │   └─ 索引信号配置
    └─ 温度传感器
    ↓
控制算法初始化
    ├─ BldcServo - 电机伺服
    │   ├─ FOC初始化
    │   ├─ PID控制器初始化
    │   └─ 控制模式设置
    └─ 安全限制配置
    ↓
通信系统初始化
    ├─ FDCAN - CAN-FD控制器
    │   ├─ 过滤器配置
    │   ├─ 波特率设置
    │   └─ 中断配置
    ├─ Multiplex协议 - 多路复用协议
    └─ DatagramServer - 数据报服务器
    ↓
主控制器初始化
    ├─ MoteusController构造
    │   └─ 组装所有组件
    └─ controller.Start() - 启动控制器
    ↓
进入主循环
    └─ while(true) { Poll(); PollMillisecond(); }
```

### 1.2 配置加载流程

```
PersistentConfig初始化
    ↓
从Flash读取配置数据
    ↓
配置版本检查
    ├─ 版本匹配 → 使用Flash配置
    └─ 版本不匹配 → 使用默认配置
    ↓
配置验证
    ├─ 参数范围检查
    ├─ 依赖关系检查
    └─ 硬件兼容性检查
    ↓
分发配置到各组件
    ├─ BldcServo接收电机配置
    ├─ MotorPosition接收编码器配置
    ├─ PID接收控制参数
    └─ 其他组件接收各自配置
    ↓
应用配置
    └─ 各组件根据配置初始化
```

### 1.3 错误处理流程

```
错误检测
    ├─ 硬件错误 (过流、过温、过压)
    ├─ 软件错误 (配置错误、通信超时)
    └─ 传感器错误 (编码器故障、通信失败)
    ↓
错误分类
    ├─ 可恢复错误 → 记录日志，进入FAULT状态
    └─ 致命错误 → 停止电机，进入安全状态
    ↓
FAULT状态处理
    ├─ 停止PWM输出
    ├─ 禁用Gate Driver
    ├─ 记录错误代码
    └─ 发送错误遥测
    ↓
等待错误清除
    ├─ 用户发送清除命令
    ├─ 错误条件消失
    └─ 重新初始化
    ↓
恢复运行
    └─ 返回OPERATING状态
```

## 2. 电机控制流程

### 2.1 位置控制模式流程

位置控制是最常用的控制模式，实现精确的位置跟踪。

```
用户发送位置命令
    ↓
命令接收 (BldcServo::Command)
    ↓
命令解析
    ├─ 目标位置 (position)
    ├─ 目标速度 (velocity)
    ├─ 前馈扭矩 (feedforward_torque)
    ├─ KP缩放 (kp_scale)
    ├─ KD缩放 (kd_scale)
    └─ 最大扭矩限制 (maximum_torque)
    ↓
轨迹规划
    ├─ 位置限制检查 (position_min/max)
    ├─ 速度限制检查 (velocity_limit)
    ├─ 加速度限制 (acceleration_limit)
    └─ 生成平滑轨迹
    ↓
位置环控制
    ├─ 计算位置误差: error = position_cmd - position_fb
    ├─ 位置PID计算: velocity_cmd = Kp * error + Ki * ∫error + Kd * d(error)/dt
    ├─ 应用前馈: velocity_cmd += velocity_ff
    └─ 速度限制
    ↓
速度环控制
    ├─ 计算速度误差: error = velocity_cmd - velocity_fb
    ├─ 速度PID计算: current_cmd = Kp * error + Ki * ∫error
    ├─ 应用前馈: current_cmd += current_ff
    └─ 电流限制
    ↓
电流环控制 (FOC)
    ├─ Clarke变换: (ia, ib, ic) → (iα, iβ)
    ├─ Park变换: (iα, iβ) → (id, iq)
    ├─ d轴电流环: vd = Kp_d * (id_ref - id) + Ki_d * ∫(id_ref - id)
    ├─ q轴电流环: vq = Kp_q * (iq_ref - iq) + Ki_q * ∫(iq_ref - iq)
    ├─ 逆Park变换: (vd, vq) → (vα, vβ)
    └─ SVPWM: (vα, vβ) → (duty_a, duty_b, duty_c)
    ↓
PWM输出
    ├─ 更新PWM占空比
    ├─ 中心对齐模式
    └─ 死区时间插入
    ↓
功率级驱动
    ├─ Gate Driver (DRV8323)
    ├─ MOSFET开关
    └─ 电机绕组电流
```

### 2.2 速度控制模式流程

速度控制模式用于需要精确速度控制的场景。

```
用户发送速度命令
    ↓
命令接收 (BldcServo::Command)
    ↓
命令解析
    ├─ 目标速度 (velocity)
    ├─ 前馈扭矩 (feedforward_torque)
    ├─ KP缩放 (kp_scale = 0, 禁用位置环)
    ├─ KD缩放 (kd_scale)
    └─ 最大扭矩限制 (maximum_torque)
    ↓
速度环控制
    ├─ 计算速度误差: error = velocity_cmd - velocity_fb
    ├─ 速度PID计算: current_cmd = Kp * error + Ki * ∫error
    ├─ 应用前馈: current_cmd += current_ff
    ├─ 电流限制
    └─ 扭矩限制
    ↓
电流环控制 (FOC)
    ├─ Clarke变换
    ├─ Park变换
    ├─ d轴电流环
    ├─ q轴电流环
    ├─ 逆Park变换
    └─ SVPWM
    ↓
PWM输出
    ↓
功率级驱动
    ↓
电机旋转
```

### 2.3 扭矩控制模式流程

扭矩控制模式用于力控应用，直接控制输出扭矩。

```
用户发送扭矩命令
    ↓
命令接收 (BldcServo::Command)
    ↓
命令解析
    ├─ 目标扭矩 (torque)
    └─ 最大扭矩限制 (maximum_torque)
    ↓
扭矩到电流转换
    ├─ 使用电机扭矩常数: iq = torque / kt
    ├─ 应用扭矩限制
    └─ id = 0 (最大扭矩/电流比)
    ↓
电流环控制 (FOC)
    ├─ Clarke变换
    ├─ Park变换
    ├─ d轴电流环: id_ref = 0
    ├─ q轴电流环: iq_ref = calculated_iq
    ├─ 逆Park变换
    └─ SVPWM
    ↓
PWM输出
    ↓
功率级驱动
    ↓
电机输出扭矩
```

### 2.4 电流控制模式流程

电流控制是最底层的控制模式，直接控制相电流。

```
用户发送电流命令
    ↓
命令接收 (BldcServo::Command)
    ↓
命令解析
    ├─ d轴电流目标 (id_ref)
    └─ q轴电流目标 (iq_ref)
    ↓
电流环控制 (FOC)
    ├─ ADC采样三相电流 (ia, ib, ic)
    ├─ Clarke变换: (ia, ib, ic) → (iα, iβ)
    ├─ Park变换: (iα, iβ) → (id, iq)
    ├─ d轴电流环: vd = Kp_d * (id_ref - id) + Ki_d * ∫(id_ref - id)
    ├─ q轴电流环: vq = Kp_q * (iq_ref - iq) + Ki_q * ∫(iq_ref - iq)
    ├─ 电压限制
    ├─ 逆Park变换: (vd, vq) → (vα, vβ)
    └─ SVPWM: (vα, vβ) → (duty_a, duty_b, duty_c)
    ↓
PWM输出
    ↓
功率级驱动
    ↓
电机绕组电流
```

## 3. 通信流程

### 3.1 命令发送流程（客户端到固件）

```
用户程序调用
    ├─ Python: controller.set_position(...)
    ├─ C++: controller.SetPosition(...)
    └─ Rust: controller.set_position(...)
    ↓
协议编码 (Multiplex)
    ├─ 确定命令类型 (kPosition = 0x01)
    ├─ 编码源地址 (source_id)
    ├─ 编码目标地址 (dest_id)
    ├─ 编码命令参数
    │   ├─ position (float32)
    │   ├─ velocity (float32)
    │   ├─ feedforward_torque (float32)
    │   ├─ kp_scale (float32)
    │   ├─ kd_scale (float32)
    │   └─ maximum_torque (float32)
    └─ 计算校验和
    ↓
CAN帧封装
    ├─ CAN ID = (dest_id << 8) | source_id
    ├─ 数据长度 = 8 (CAN) 或 64 (CAN-FD)
    └─ 数据 = 编码后的命令
    ↓
传输层发送
    ├─ SocketCAN: sendto()
    ├─ fdcanusb: USB批量传输
    └─ pythoncan: bus.send()
    ↓
物理层传输
    ├─ CAN收发器
    ├─ CAN总线 (差分信号)
    └─ 终端电阻匹配
    ↓
固件CAN接收
    ├─ CAN中断触发
    ├─ DMA传输到缓冲区
    └─ 消息入队
    ↓
协议解码 (Multiplex)
    ├─ 解析CAN ID
    ├─ 提取源/目标地址
    ├─ 解码命令类型
    ├─ 解码命令参数
    └─ 验证校验和
    ↓
命令分发
    ├─ 根据目标地址路由
    └─ 传递给BldcServo
    ↓
命令执行
    └─ BldcServo::Command()
```

### 3.2 状态查询流程（固件到客户端）

```
用户程序调用
    ├─ Python: result = await controller.query()
    ├─ C++: result = controller.Query()
    └─ Rust: result = controller.query().await
    ↓
查询命令编码
    ├─ 命令类型: kQuery
    ├─ 源地址: 客户端ID
    └─ 目标地址: 设备ID
    ↓
CAN帧发送
    ↓
固件接收查询命令
    ↓
状态收集
    ├─ 电机状态
    │   ├─ mode (当前模式)
    │   ├─ position (当前位置)
    │   ├─ velocity (当前速度)
    │   └─ torque (当前扭矩)
    ├─ 电流状态
    │   ├─ q_current (q轴电流)
    │   └─ d_current (d轴电流)
    ├─ 编码器状态
    │   ├─ raw_position
    │   ├─ velocity
    │   └─ index_count
    └─ 系统状态
        ├─ voltage (供电电压)
        ├─ temperature (温度)
        └─ fault_code (故障代码)
    ↓
响应编码 (Multiplex)
    ├─ 编码响应类型
    ├─ 编码状态数据
    │   ├─ 使用紧凑格式 (float16/float32)
    │   └─ 只包含请求的字段
    └─ 计算校验和
    ↓
CAN帧封装
    ├─ CAN ID = (source_id << 8) | dest_id
    └─ 数据 = 编码后的响应
    ↓
CAN发送
    ↓
客户端接收
    ├─ CAN中断/轮询
    ├─ 消息入队
    └─ 消息出队
    ↓
响应解码 (Multiplex)
    ├─ 解析CAN ID
    ├─ 提取响应类型
    ├─ 解码状态数据
    └─ 验证校验和
    ↓
返回用户程序
    └─ 返回状态字典/结构体
```

### 3.3 遥测数据流（固件主动上报）

```
遥测数据产生
    ├─ 周期性采样 (如1kHz)
    ├─ 事件触发 (如故障发生)
    └─ 用户请求
    ↓
遥测数据收集
    ├─ 电机状态
    ├─ 控制参数
    ├─ 传感器数据
    └─ 系统信息
    ↓
遥测打包 (TelemetryManager)
    ├─ 序列化数据
    │   ├─ 使用Visitor模式
    │   └─ 二进制编码
    ├─ 添加时间戳
    └─ 添加通道ID
    ↓
CAN帧封装
    ├─ 使用广播地址
    ├─ 数据长度 = 64 (CAN-FD)
    └─ 分片传输（如果数据过大）
    ↓
CAN发送
    ↓
客户端接收
    ├─ 过滤器匹配
    ├─ 消息入队
    └─ 消息处理
    ↓
遥测解析
    ├─ 根据通道ID查找解析器
    ├─ 反序列化数据
    └─ 更新本地状态
    ↓
数据展示
    ├─ tview.py实时显示
    ├─ 日志记录
    └─ 数据分析
```

### 3.4 配置读写流程

```
配置写入流程:

用户修改配置
    ├─ moteus_tool.py --config
    ├─ 客户端库 write_config()
    └─ 直接CAN命令
    ↓
配置命令编码
    ├─ 命令类型: kWriteRegisters
    ├─ 寄存器地址
    ├─ 寄存器数据
    └─ 数据长度
    ↓
CAN帧发送
    ↓
固件接收
    ↓
配置解析 (PersistentConfig)
    ├─ 查找寄存器
    ├─ 验证数据
    ├─ 更新内存中的配置
    └─ 通知相关组件
    ↓
配置应用
    ├─ BldcServo更新电机参数
    ├─ PID更新控制参数
    └─ MotorPosition更新编码器参数
    ↓
Flash写入
    ├─ 擦除Flash扇区
    ├─ 写入新配置
    ├─ 验证写入
    └─ 更新配置版本
    ↓
确认响应
    └─ 发送成功/失败响应

配置读取流程:

用户请求配置
    ├─ moteus_tool.py --read-config
    └─ 客户端库 read_config()
    ↓
读取命令编码
    ├─ 命令类型: kReadRegisters
    ├─ 寄存器地址范围
    └─ 数据长度
    ↓
CAN帧发送
    ↓
固件接收
    ↓
配置读取 (PersistentConfig)
    ├─ 从内存读取配置
    ├─ 序列化为二进制
    └─ 打包响应
    ↓
CAN响应发送
    ↓
客户端接收
    ↓
配置解析
    ├─ 反序列化数据
    ├─ 转换为可读格式
    └─ 显示或返回
```

## 4. 校准流程

### 4.1 编码器校准流程

编码器校准用于建立电机电气角度和机械角度的对应关系。

```
启动校准
    ├─ moteus_tool.py --calibrate
    └─ 发送校准命令
    ↓
进入校准模式
    ├─ BldcServo状态 → CALIBRATING
    ├─ 禁用正常控制
    └─ 启用校准算法
    ↓
Phase 1: 电阻测量
    ├─ 施加已知电压
    ├─ 测量电流响应
    ├─ 计算相电阻
    └─ 验证电阻范围
    ↓
Phase 2: 电感测量
    ├─ 施加脉冲电压
    ├─ 测量电流上升时间
    ├─ 计算相电感
    └─ 验证电感范围
    ↓
Phase 3: 编码器偏移校准
    ├─ 锁定转子位置
    ├─ 读取编码器值
    ├─ 计算电气角度偏移
    ├─ 多次测量取平均
    └─ 存储偏移值
    ↓
Phase 4: 旋转方向验证
    ├─ 低速旋转电机
    ├─ 检测旋转方向
    ├─ 与编码器方向对比
    └─ 必要时反转
    ↓
Phase 5: 扭矩常数测量
    ├─ 施加已知电流
    ├─ 测量输出扭矩
    ├─ 计算扭矩常数 kt
    └─ 验证kt范围
    ↓
校准完成
    ├─ 保存校准数据到Flash
    ├─ 状态 → CALIBRATION_COMPLETE
    └─ 发送完成响应
    ↓
用户确认
    └─ 可以开始正常运行
```

### 4.2 索引校准流程

索引校准用于建立绝对位置参考点。

```
启动索引校准
    ├─ 发送RequireReindex命令
    └─ 进入索引搜索模式
    ↓
低速旋转
    ├─ 以低速旋转电机
    ├─ 监测索引信号
    └─ 等待索引边沿
    ↓
检测索引
    ├─ 检测索引上升沿
    ├─ 记录编码器值
    ├─ 验证索引质量
    └─ 多次验证
    ↓
设置位置参考
    ├─ 将索引位置设为0
    ├─ 更新位置偏移
    └─ 标记index_found = true
    ↓
校准完成
    └─ 可以正常使用绝对位置
```

## 5. 故障处理流程

### 5.1 过流保护流程

```
电流采样 (ADC中断)
    ↓
电流值计算
    ├─ Clarke变换
    └─ 计算电流幅值: I = sqrt(id^2 + iq^2)
    ↓
过流检测
    ├─ 比较: I > overcurrent_threshold
    └─ 去抖: 持续时间 > overcurrent_timeout
    ↓
触发过流保护
    ├─ 立即禁用PWM
    ├─ 禁用Gate Driver
    ├─ 记录故障代码: errc::overcurrent
    └─ 发送故障遥测
    ↓
进入FAULT状态
    ├─ 停止所有控制
    ├─ 记录故障时间
    └─ 等待故障清除
    ↓
故障清除
    ├─ 用户发送清除命令
    ├─ 检查电流是否恢复正常
    ├─ 重新初始化Gate Driver
    └─ 返回OPERATING状态
```

### 5.2 过温保护流程

```
温度采样 (周期性)
    ├─ 读取温度传感器
    ├─ ADC采样
    └─ 转换为温度值
    ↓
温度检测
    ├─ 比较: T > overtemperature_threshold
    └─ 去抖: 持续时间 > overtemperature_timeout
    ↓
降额保护 (第一阶段)
    ├─ 降低电流限制
    ├─ 降低速度限制
    ├─ 发送警告遥测
    └─ 继续运行
    ↓
温度继续上升
    ├─ T > critical_temperature
    └─ 进入严重过温
    ↓
触发过温保护
    ├─ 禁用PWM
    ├─ 禁用Gate Driver
    ├─ 记录故障代码: errc::overtemperature
    └─ 发送故障遥测
    ↓
进入FAULT状态
    ↓
温度下降
    ├─ T < recovery_temperature
    └─ 可以清除故障
    ↓
故障清除
    └─ 恢复正常运行
```

### 5.3 通信超时流程

```
通信监控 (周期性)
    ├─ 记录最后通信时间
    └─ 计算通信间隔
    ↓
超时检测
    ├─ 间隔 > communication_timeout
    └─ 超时次数 > timeout_count
    ↓
触发超时保护
    ├─ 停止电机 (进入STOPPED模式)
    ├─ 记录故障代码: errc::communication_timeout
    └─ 发送故障遥测
    ↓
等待通信恢复
    ├─ 接收新的命令
    └─ 重置超时计数器
    ↓
恢复正常
    └─ 继续接受命令
```

## 6. 主循环流程

### 6.1 主循环结构

```cpp
int main() {
    // 初始化阶段
    InitializeHardware();
    InitializeServices();
    InitializeControllers();
    
    MoteusController controller(...);
    controller.Start();
    
    // 主循环
    while (true) {
        // 高优先级任务 (微秒级)
        controller.Poll();
        
        // 低优先级任务 (毫秒级)
        controller.PollMillisecond();
    }
}
```

### 6.2 Poll() 任务流程

Poll()在主循环中持续调用，处理实时任务。

```
Poll() 调用
    ↓
CAN消息处理
    ├─ 检查CAN接收队列
    ├─ 处理接收到的消息
    │   ├─ 解析命令
    │   ├─ 执行命令
    │   └─ 发送响应
    └─ 处理CAN发送队列
    ↓
电机控制更新
    ├─ 读取编码器位置
    ├─ 读取电流采样
    ├─ 执行控制算法
    │   ├─ 位置环
    │   ├─ 速度环
    │   └─ 电流环 (FOC)
    └─ 更新PWM输出
    ↓
辅助端口处理
    ├─ 读取AuxPort输入
    ├─ 处理I2C/SPI/UART
    └─ 更新AuxPort输出
    ↓
遥测数据更新
    ├─ 收集遥测数据
    ├─ 打包遥测帧
    └─ 加入发送队列
    ↓
错误检查
    ├─ 检查故障条件
    ├─ 更新故障状态
    └─ 执行保护动作
```

### 6.3 PollMillisecond() 任务流程

PollMillisecond()每秒调用约1000次，处理毫秒级任务。

```
PollMillisecond() 调用 (每毫秒)
    ↓
时间更新
    ├─ 更新毫秒计数器
    └─ 检查定时器
    ↓
周期性任务
    ├─ 温度采样 (每10ms)
    ├─ 电压采样 (每10ms)
    ├─ 遥测发送 (每1ms或可配置)
    └─ 看门狗喂狗 (每100ms)
    ↓
状态监控
    ├─ 检查通信超时
    ├─ 检查过温条件
    └─ 检查系统状态
    ↓
日志记录
    ├─ 记录重要事件
    └─ 更新统计信息
```

## 7. 中断处理流程

### 7.1 ADC采样中断

ADC采样中断是最高优先级的中断，用于电流环控制。

```
ADC触发 (PWM中心对齐)
    ↓
ADC中断入口
    ├─ 保存上下文
    └─ 清除中断标志
    ↓
采样三相电流
    ├─ 读取ADC1 (current1)
    ├─ 读取ADC2 (current3)
    ├─ 读取ADC3 (current2)
    └─ DMA传输完成
    ↓
电流计算
    ├─ 转换为安培
    ├─ 滤波处理
    └─ 存储到缓冲区
    ↓
FOC执行
    ├─ Clarke变换
    ├─ Park变换
    ├─ 电流环PI控制
    ├─ 逆Park变换
    └─ SVPWM计算
    ↓
更新PWM占空比
    ├─ 写入定时器寄存器
    └─ 下一个PWM周期生效
    ↓
恢复上下文
    └─ 返回主程序
```

### 7.2 CAN接收中断

CAN接收中断处理 incoming CAN消息。

```
CAN消息到达
    ↓
CAN中断入口
    ├─ 保存上下文
    └─ 清除中断标志
    ↓
读取CAN消息
    ├─ 读取CAN ID
    ├─ 读取数据长度
    ├─ 读取数据内容
    └─ DMA传输（如果使用）
    ↓
消息入队
    ├─ 检查队列空间
    ├─ 写入接收队列
    └─ 更新队列指针
    ↓
过滤器匹配
    ├─ 检查是否匹配过滤器
    └─ 不匹配则丢弃
    ↓
恢复上下文
    └─ 返回主程序
```

### 7.3 编码器读取中断

编码器读取中断用于高速读取编码器位置。

```
编码器触发 (SPI/I2C/定时器)
    ↓
编码器中断入口
    ├─ 保存上下文
    └─ 清除中断标志
    ↓
读取编码器数据
    ├─ SPI: 读取寄存器
    ├─ I2C: 读取数据
    └─ 转换为角度值
    ↓
数据处理
    ├─  unwrap处理 (处理360°跳变)
    ├─ 滤波处理
    └─ 计算速度
    ↓
存储结果
    ├─ 更新位置值
    ├─ 更新速度值
    └─ 更新索引计数
    ↓
恢复上下文
    └─ 返回主程序
```

## 8. 数据流时序

### 8.1 控制命令时序

```
时间轴 →

T0: 用户发送位置命令
    │
    ├─ 协议编码 (100μs)
    ├─ CAN传输 (50μs)
    │
T1: 固件接收命令 (T0 + 150μs)
    │
    ├─ 协议解码 (50μs)
    ├─ 命令分发 (10μs)
    │
T2: 开始执行控制 (T1 + 60μs)
    │
    ├─ 位置环计算 (10μs)
    ├─ 速度环计算 (10μs)
    ├─ 电流环计算 (20μs)
    ├─ FOC计算 (30μs)
    │
T3: PWM更新 (T2 + 70μs)
    │
    ├─ 等待下一个PWM周期
    │
T4: PWM生效 (下一个PWM周期开始)
    │
    ├─ 电机响应
    │
T5: 状态反馈 (用户查询)
    │
    ├─ 状态采集 (50μs)
    ├─ 协议编码 (100μs)
    ├─ CAN传输 (50μs)
    │
T6: 客户端接收 (T5 + 200μs)

总延迟: T6 - T0 ≈ 500μs - 1ms (取决于PWM频率和通信负载)
```

### 8.2 遥测数据时序

```
时间轴 →

T0: 遥测采样触发 (周期性，如1ms)
    │
    ├─ 数据采集 (20μs)
    ├─ 数据打包 (30μs)
    │
T1: CAN发送 (T0 + 50μs)
    │
    ├─ CAN传输 (50μs)
    │
T2: 客户端接收 (T1 + 50μs)
    │
    ├─ 数据解析 (20μs)
    ├─ 数据更新 (10μs)
    │
T3: 显示/记录 (T2 + 30μs)

遥测周期: 1ms (可配置)
端到端延迟: ~130μs
```

## 9. 状态机流程

### 9.1 BldcServo状态机

```
                    ┌──────────┐
                    │ STOPPED  │ ← 初始状态
                    └────┬─────┘
                         │
                    启用命令 │
                         ↓
                    ┌──────────┐
                    │ ENABLING │ → 启用Gate Driver
                    └────┬─────┘
                         │
                    启用完成 │
                         ↓
                    ┌──────────┐
         ┌─────────│ OPERATING│ ← 正常运行
         │         └────┬─────┘
         │              │
    故障检测        校准命令
         │              │
         ↓              ↓
    ┌──────────┐   ┌─────────────┐
    │  FAULT   │   │ CALIBRATING │
    └────┬─────┘   └──────┬──────┘
         │                │
    清除命令         校准完成
         │                │
         └────────→┌──────────────────┐
                   │CALIBRATION_      │
                   │COMPLETE          │
                   └──────────────────┘
```

### 9.2 状态转换条件

**STOPPED → ENABLING**
- 条件: 收到启用命令
- 动作: 初始化Gate Driver

**ENABLING → OPERATING**
- 条件: Gate Driver启用成功
- 动作: 开始控制循环

**OPERATING → FAULT**
- 条件: 检测到故障（过流、过温、过压等）
- 动作: 禁用PWM，记录故障

**OPERATING → CALIBRATING**
- 条件: 收到校准命令
- 动作: 进入校准模式

**CALIBRATING → CALIBRATION_COMPLETE**
- 条件: 校准成功完成
- 动作: 保存校准数据

**FAULT → STOPPED**
- 条件: 收到清除命令且故障条件消失
- 动作: 重新初始化

## 10. 性能指标

### 10.1 控制环频率

- **电流环**: 20kHz (50μs周期)
- **速度环**: 20kHz (与电流环同步)
- **位置环**: 20kHz (与电流环同步)

### 10.2 通信延迟

- **命令延迟**: 150-300μs (客户端到固件)
- **响应延迟**: 150-300μs (固件到客户端)
- **总往返延迟**: 300-600μs

### 10.3 采样频率

- **电流采样**: 20kHz
- **电压采样**: 20kHz
- **温度采样**: 100Hz
- **编码器采样**: 20kHz (SPI/I2C)

### 10.4 控制精度

- **位置精度**: < 0.01° (取决于编码器分辨率)
- **速度精度**: < 0.1% (取决于控制参数)
- **电流精度**: < 1% (取决于ADC分辨率)
- **扭矩精度**: < 2% (取决于电机参数)

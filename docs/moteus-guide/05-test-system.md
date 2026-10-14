# Moteus 测试系统

## 1. 测试系统概述

Moteus项目采用多层次的测试策略，确保软件质量和可靠性。测试系统包括：

- **单元测试**：测试单个函数或类
- **集成测试**：测试模块间的交互
- **仿真测试**：在主机上模拟硬件行为
- **回归测试**：确保修改不破坏现有功能
- **性能测试**：验证性能指标

### 1.1 测试框架

- **C++测试**：Google Test (gtest)
- **Python测试**：pytest + unittest
- **仿真测试**：自定义仿真框架

### 1.2 测试目标

```bash
# 运行所有快速测试（推荐日常开发）
tools/bazel test --config=host //:host

# 运行完整测试套件（包括仿真回归测试）
tools/bazel test --test_size_filters= --test_tag_filters=-manual --config=host //:host
```

## 2. 固件测试

### 2.1 测试位置

固件测试位于 `fw/test/` 目录。

### 2.2 测试文件结构

```
fw/test/
├── foc_test.cc                    # FOC算法测试
├── motor_position_test.cc         # 位置反馈测试
├── torque_model_test.cc           # 扭矩模型测试
├── stm32_i2c_timing_test.cc       # I2C时序测试
├── encoder_bandwidth_test.cc      # 编码器带宽测试
├── hall_filter_test.cc            # Hall滤波器测试
├── bissc_extract_test.cc          # BISS-C提取测试
├── simulation_dynamometer_test.cc # 测功机仿真测试
├── simulation_regen_test.cc       # 再生仿真测试
├── simulation_hall_misalign_test.cc # Hall错位仿真测试
└── BUILD                          # 测试构建配置
```

### 2.3 运行固件测试

```bash
# 运行所有固件测试
tools/bazel test --config=host //fw:host

# 运行特定测试
tools/bazel test --config=host //fw/test:foc_test
tools/bazel test --config=host //fw/test:motor_position_test

# 查看详细输出
tools/bazel test --config=host //fw/test:foc_test --test_output=all
```

### 2.4 测试示例：FOC测试

**文件**：`fw/test/foc_test.cc`

**测试内容**：
- Clarke变换正确性
- Park变换正确性
- 逆Park变换正确性
- SVPWM计算正确性
- 电流环PI控制

**测试结构**：
```cpp
#include <gtest/gtest.h>
#include "fw/foc.h"

namespace moteus {
namespace test {

class FocTest : public ::testing::Test {
 protected:
  void SetUp() override {
    // 初始化测试环境
  }
};

TEST_F(FocTest, ClarkeTransform) {
  // 测试Clarke变换
  Foc::ClarkeTransform input = {1.0f, 2.0f, 3.0f};
  auto output = Foc::Clarke(input);
  
  EXPECT_NEAR(output.alpha, expected_alpha, 1e-6);
  EXPECT_NEAR(output.beta, expected_beta, 1e-6);
}

TEST_F(FocTest, ParkTransform) {
  // 测试Park变换
  // ...
}

}  // namespace test
}  // namespace moteus
```

### 2.5 测试示例：电机位置测试

**文件**：`fw/test/motor_position_test.cc`

**测试内容**：
- 编码器读取
- 位置计算
- 速度计算
- 索引处理
- 多编码器融合

**测试方法**：
```cpp
TEST_F(MotorPositionTest, EncoderReading) {
  // 模拟编码器输入
  MockEncoder encoder;
  encoder.SetPosition(1.5f);
  
  MotorPosition position(config, &encoder);
  position.Poll();
  
  EXPECT_NEAR(position.status().position, 1.5f, 1e-6);
}

TEST_F(MotorPositionTest, VelocityCalculation) {
  // 测试速度计算
  // ...
}
```

### 2.6 仿真测试

仿真测试在主机上模拟硬件行为，用于验证控制算法。

**文件**：`fw/test/simulation_*.cc`

**仿真框架**：
```cpp
class SimulationEnvironment {
 public:
  // 模拟电机模型
  MotorModel motor;
  
  // 模拟传感器
  EncoderModel encoder;
  CurrentSensorModel current_sensor;
  
  // 模拟控制器
  BldcServo controller;
  
  // 运行仿真步骤
  void Step(float dt) {
    // 1. 读取传感器
    encoder.Update(motor.position);
    current_sensor.Update(motor.current);
    
    // 2. 执行控制
    controller.Poll();
    
    // 3. 更新电机状态
    motor.Update(controller.pwm_output, dt);
  }
};
```

**仿真测试示例**：
```cpp
TEST(SimulationTest, PositionControl) {
  SimulationEnvironment env;
  
  // 设置目标位置
  BldcServo::CommandData cmd;
  cmd.mode = BldcServoMode::kPosition;
  cmd.position = 1.0f;
  
  // 运行仿真
  for (int i = 0; i < 1000; i++) {
    env.controller.Command(cmd);
    env.Step(0.001f);  // 1ms步长
  }
  
  // 验证结果
  EXPECT_NEAR(env.motor.position, 1.0f, 0.01f);
}
```

### 2.7 测试覆盖率

```bash
# 生成覆盖率报告（需要配置）
tools/bazel coverage --config=host //fw:host

# 查看覆盖率报告
# 通常在 bazel-out/_coverage/_coverage_report.dat
```

## 3. Python库测试

### 3.1 测试位置

Python测试位于 `lib/python/moteus/test/` 目录。

### 3.2 测试文件结构

```
lib/python/moteus/test/
├── protocol_test.py       # 协议编解码测试
├── moteus_test.py         # 主库功能测试
├── moteus_tool_test.py    # 命令行工具测试
├── transport_test.py      # 传输层测试
├── multiplex_test.py      # 多路复用协议测试
├── reader_test.py         # 数据读取测试
├── calibrate_encoder_test.py  # 编码器校准测试
├── ld_saturation_test.py  # 电感饱和测试
├── fdcanusb_test.py       # fdcanusb测试
├── mock_serial.py         # 模拟串口（测试辅助）
└── BUILD                  # 测试构建配置
```

### 3.3 运行Python测试

```bash
# 运行所有Python测试
tools/bazel test --config=host //lib/python:host

# 运行特定测试
tools/bazel test --config=host //lib/python/test:protocol_test
tools/bazel test --config=host //lib/python/test:moteus_test

# 使用pytest直接运行（开发时更方便）
cd lib/python/moteus
python3 -m pytest test/ -v
```

### 3.4 测试示例：协议测试

**文件**：`lib/python/moteus/test/protocol_test.py`

**测试内容**：
- 命令编码
- 响应解码
- 数据类型处理
- 边界条件

**测试结构**：
```python
import unittest
from moteus import protocol

class ProtocolTest(unittest.TestCase):
    def test_position_command_encoding(self):
        """测试位置命令编码"""
        cmd = protocol.PositionCommand(
            position=1.5,
            velocity=0.5,
            feedforward_torque=0.1
        )
        
        encoded = cmd.encode()
        
        # 验证编码结果
        self.assertEqual(len(encoded), expected_length)
        self.assertEqual(encoded[0], expected_command_id)
    
    def test_response_decoding(self):
        """测试响应解码"""
        data = bytes([...])  # 模拟响应数据
        response = protocol.decode_response(data)
        
        self.assertAlmostEqual(response.position, expected_position)
        self.assertAlmostEqual(response.velocity, expected_velocity)

if __name__ == '__main__':
    unittest.main()
```

### 3.5 测试示例：传输层测试

**文件**：`lib/python/moteus/test/transport_test.py`

**测试内容**：
- SocketCAN传输
- fdcanusb传输
- 数据发送和接收
- 错误处理

**测试方法**：
```python
class TransportTest(unittest.TestCase):
    def test_socketcan_send_receive(self):
        """测试SocketCAN发送和接收"""
        transport = SocketCanTransport('vcan0')
        
        # 发送数据
        transport.send(b'\x01\x02\x03\x04')
        
        # 接收数据
        received = transport.receive()
        
        self.assertEqual(received, b'\x01\x02\x03\x04')
    
    def test_timeout_handling(self):
        """测试超时处理"""
        transport = SocketCanTransport('vcan0')
        transport.set_timeout(1.0)
        
        with self.assertRaises(TimeoutError):
            transport.receive()
```

### 3.6 测试示例：主库功能测试

**文件**：`lib/python/moteus/test/moteus_test.py`

**测试内容**：
- Controller类功能
- 命令发送和接收
- 状态查询
- 错误处理

**测试方法**：
```python
class MoteusTest(unittest.TestCase):
    def setUp(self):
        """设置测试环境"""
        self.transport = MockTransport()
        self.controller = Controller(id=1, transport=self.transport)
    
    def test_set_position(self):
        """测试位置控制"""
        # 模拟响应
        self.transport.mock_response = create_mock_response(
            position=1.5,
            velocity=0.0
        )
        
        # 发送命令
        result = await self.controller.set_position(position=1.5)
        
        # 验证结果
        self.assertAlmostEqual(result['position'], 1.5)
    
    def test_query(self):
        """测试状态查询"""
        # 模拟响应
        self.transport.mock_response = create_mock_response(
            mode=Mode.OPERATING,
            position=2.0,
            velocity=1.0
        )
        
        result = await self.controller.query()
        
        self.assertEqual(result['mode'], Mode.OPERATING)
        self.assertAlmostEqual(result['position'], 2.0)
```

## 4. C++库测试

### 4.1 测试位置

C++测试位于 `lib/cpp/mjbots/moteus/test/` 目录。

### 4.2 测试文件结构

```
lib/cpp/mjbots/moteus/test/
├── moteus_test.cc          # 主库功能测试
├── protocol_test.cc        # 协议测试
├── transport_test.cc       # 传输层测试
└── BUILD                   # 测试构建配置
```

### 4.3 运行C++测试

```bash
# 运行所有C++测试
tools/bazel test --config=host //lib/cpp:host

# 运行特定测试
tools/bazel test --config=host //lib/cpp/test:moteus_test
```

### 4.4 测试示例

**文件**：`lib/cpp/mjbots/moteus/test/moteus_test.cc`

```cpp
#include <gtest/gtest.h>
#include "mjbots/moteus/moteus.h"

namespace mjbots {
namespace moteus {
namespace test {

class MoteusTest : public ::testing::Test {
 protected:
  void SetUp() override {
    // 初始化测试环境
  }
};

TEST_F(MoteusTest, PositionCommand) {
  // 测试位置命令
  Controller::Options options;
  options.id = 1;
  options.transport = std::make_unique<MockTransport>();
  
  Controller controller(options);
  
  Controller::PositionCommand cmd;
  cmd.position = 1.5f;
  cmd.velocity = 0.5f;
  
  auto result = controller.SetPosition(cmd);
  
  EXPECT_NEAR(result.position, 1.5f, 1e-6);
}

}  // namespace test
}  // namespace moteus
}  // namespace mjbots
```

## 5. 工具测试

### 5.1 测试位置

工具测试位于 `utils/test/` 目录。

### 5.2 测试文件结构

```
utils/test/
├── moteus_tool_test.py     # moteus_tool测试
├── firmware_validate_test.py  # 固件验证测试
└── BUILD
```

### 5.3 运行工具测试

```bash
# 运行所有工具测试
tools/bazel test --config=host //utils:host

# 运行特定测试
tools/bazel test --config=host //utils/test:moteus_tool_test
```

## 6. 测试最佳实践

### 6.1 单元测试原则

1. **单一职责**：每个测试只测试一个功能
2. **独立性**：测试之间不相互依赖
3. **可重复**：测试结果可重复
4. **快速**：单元测试应该快速执行
5. **隔离**：使用mock隔离外部依赖

### 6.2 测试命名规范

**C++测试**：
```cpp
TEST_F(TestSuiteName, TestName) {
  // 测试名称应该描述测试的内容
}

// 好的命名
TEST_F(FocTest, ClarkeTransformWithBalancedCurrents) { }
TEST_F(MotorPositionTest, VelocityCalculationAtHighSpeed) { }

// 不好的命名
TEST_F(FocTest, Test1) { }
TEST_F(MotorPositionTest, TestVelocity) { }
```

**Python测试**：
```python
def test_function_name_scenario(self):
    """测试名称应该描述测试的场景"""
    pass

# 好的命名
def test_position_command_with_zero_velocity(self):
    pass

def test_response_decoding_with_invalid_data(self):
    pass

# 不好的命名
def test_position(self):
    pass

def test_something(self):
    pass
```

### 6.3 测试数据管理

**使用测试夹具**：
```python
class TestFixture:
    """测试夹具，提供通用的测试数据"""
    
    @staticmethod
    def create_motor_config():
        """创建标准的电机配置"""
        return MotorConfig(
            pole_pairs=7,
            torque_constant=0.1,
            phase_resistance=0.5,
            phase_inductance=0.001
        )
    
    @staticmethod
    def create_encoder_config():
        """创建标准的编码器配置"""
        return EncoderConfig(
            resolution=8192,
            index_enabled=True
        )
```

### 6.4 Mock和Stub

**使用Mock隔离依赖**：
```python
class MockTransport(Transport):
    """模拟传输层"""
    
    def __init__(self):
        self.sent_data = []
        self.mock_response = None
    
    def send(self, data):
        self.sent_data.append(data)
    
    def receive(self):
        return self.mock_response

# 使用Mock
transport = MockTransport()
controller = Controller(id=1, transport=transport)

# 验证发送的数据
controller.set_position(position=1.5)
assert len(transport.sent_data) == 1
assert transport.sent_data[0] == expected_data
```

### 6.5 测试覆盖率

**目标**：
- 核心算法：>90%覆盖率
- 控制逻辑：>80%覆盖率
- 工具代码：>70%覆盖率

**生成覆盖率报告**：
```bash
# Python覆盖率
cd lib/python/moteus
python3 -m pytest test/ --cov=moteus --cov-report=html

# 打开报告
xdg-open htmlcov/index.html
```

## 7. 持续集成

### 7.1 CI配置

项目使用GitHub Actions进行持续集成。

**工作流文件**：`.github/workflows/ci.yml`

**CI流程**：
1. 代码检出
2. 安装依赖
3. 构建固件
4. 运行测试
5. 代码质量检查
6. 上传构建产物

### 7.2 本地CI脚本

```bash
# 运行完整的CI测试
./travis-ci.sh
```

**脚本内容**：
```bash
#!/bin/bash
set -e

echo "=== Building firmware ==="
tools/bazel build --config=target //:target

echo "=== Running fast tests ==="
tools/bazel test --config=host //:host

echo "=== Running full test suite ==="
tools/bazel test --test_size_filters= --test_tag_filters=-manual --config=host //:host

echo "=== All tests passed ==="
```

### 7.3 预提交检查

建议在提交前运行：

```bash
# 快速检查
tools/bazel test --config=host //:host

# 完整检查
./travis-ci.sh
```

## 8. 测试调试

### 8.1 查看测试输出

```bash
# 查看所有输出
tools/bazel test --config=host //fw/test:foc_test --test_output=all

# 查看错误输出
tools/bazel test --config=host //fw/test:foc_test --test_output=errors

# 流式输出
tools/bazel test --config=host //fw/test:foc_test --test_output=streamed
```

### 8.2 调试失败的测试

**C++测试**：
```bash
# 使用GDB调试
tools/bazel test --config=host //fw/test:foc_test --run_under=gdb

# 在GDB中
(gdb) break test_name
(gdb) run
```

**Python测试**：
```bash
# 使用pdb调试
cd lib/python/moteus
python3 -m pytest test/protocol_test.py::ProtocolTest::test_position_command -s --pdb
```

### 8.3 测试日志

测试日志保存在：
```
bazel-testlogs/fw/test/foc_test/test.log
bazel-testlogs/lib/python/test/protocol_test/test.log
```

查看日志：
```bash
cat bazel-testlogs/fw/test/foc_test/test.log
```

## 9. 性能测试

### 9.1 性能测试位置

性能测试位于各模块的test目录中。

### 9.2 性能测试示例

**固件性能测试**：
```cpp
TEST(PerformanceTest, ControlLoopTiming) {
  // 测试控制环执行时间
  auto start = std::chrono::high_resolution_clock::now();
  
  for (int i = 0; i < 10000; i++) {
    controller.Poll();
  }
  
  auto end = std::chrono::high_resolution_clock::now();
  auto duration = std::chrono::duration_cast<std::chrono::microseconds>(end - start);
  
  double avg_time_us = duration.count() / 10000.0;
  
  // 验证平均执行时间 < 50μs
  EXPECT_LT(avg_time_us, 50.0);
  
  std::cout << "Average control loop time: " << avg_time_us << " μs" << std::endl;
}
```

**Python性能测试**：
```python
import time

def test_command_throughput(self):
    """测试命令吞吐量"""
    num_commands = 1000
    
    start = time.time()
    for _ in range(num_commands):
        await self.controller.query()
    end = time.time()
    
    duration = end - start
    throughput = num_commands / duration
    
    print(f"Command throughput: {throughput:.2f} commands/s")
    
    # 验证吞吐量 > 1000 commands/s
    self.assertGreater(throughput, 1000)
```

## 10. 测试工具

### 10.1 测试辅助工具

**Mock串口**：`lib/python/moteus/test/mock_serial.py`
```python
class MockSerial:
    """模拟串口，用于测试"""
    
    def __init__(self):
        self.input_buffer = bytearray()
        self.output_buffer = bytearray()
    
    def write(self, data):
        self.output_buffer.extend(data)
    
    def read(self, size):
        data = self.input_buffer[:size]
        self.input_buffer = self.input_buffer[size:]
        return data
    
    def inject_response(self, data):
        """注入模拟响应"""
        self.input_buffer.extend(data)
```

### 10.2 测试数据生成

**生成测试数据**：
```python
def generate_test_waveform(num_samples, frequency, amplitude):
    """生成测试波形"""
    t = np.linspace(0, 1.0, num_samples)
    waveform = amplitude * np.sin(2 * np.pi * frequency * t)
    return waveform

# 使用
waveform = generate_test_waveform(1000, 50.0, 1.0)
```

### 10.3 测试报告生成

**生成HTML报告**：
```bash
# Python测试报告
python3 -m pytest test/ --html=report.html --self-contained-html

# C++测试报告
tools/bazel test --config=host //fw:host --test_output=all 2>&1 | tee test_output.txt
```

## 11. 常见问题

### 11.1 测试失败：找不到测试

**问题**：
```
ERROR: No tests found
```

**解决方案**：
```bash
# 列出所有测试
tools/bazel query 'tests(//...)'

# 检查测试文件是否在BUILD中注册
```

### 11.2 测试失败：依赖缺失

**问题**：
```
ImportError: No module named 'xxx'
```

**解决方案**：
```bash
# 安装依赖
sudo apt install python3-pytest

# 检查BUILD文件中的依赖
```

### 11.3 测试失败：超时

**问题**：
```
TIMEOUT: Test timed out
```

**解决方案**：
```bash
# 增加超时时间
tools/bazel test --config=host //fw/test:long_test --test_timeout=300
```

### 11.4 测试失败：资源竞争

**问题**：测试结果不稳定

**解决方案**：
- 确保测试独立性
- 使用锁或队列管理共享资源
- 避免使用全局状态

## 12. 测试清单

### 12.1 新功能测试清单

- [ ] 编写单元测试
- [ ] 编写集成测试
- [ ] 更新仿真测试
- [ ] 验证测试覆盖率
- [ ] 运行完整测试套件
- [ ] 更新测试文档

### 12.2 Bug修复测试清单

- [ ] 编写回归测试
- [ ] 验证测试能捕获bug
- [ ] 修复bug
- [ ] 验证测试通过
- [ ] 运行完整测试套件

### 12.3 重构测试清单

- [ ] 确保测试覆盖率不降低
- [ ] 运行完整测试套件
- [ ] 验证所有测试通过
- [ ] 检查性能测试

## 13. 总结

Moteus的测试系统提供了：

- ✅ **全面的测试覆盖**：单元测试、集成测试、仿真测试
- ✅ **多语言支持**：C++、Python测试框架
- ✅ **自动化CI**：GitHub Actions持续集成
- ✅ **性能测试**：验证性能指标
- ✅ **测试工具**：Mock、数据生成、报告生成

良好的测试是软件质量的保证，开发新功能时务必编写相应的测试。

# Moteus 开发指南

## 1. 开发环境设置

### 1.1 系统要求

**操作系统**：Ubuntu 20.04/22.04/24.04（推荐）

**硬件要求**：
- CPU：多核处理器（推荐4核以上）
- 内存：8GB以上（推荐16GB）
- 存储：20GB可用空间
- CAN接口：SocketCAN兼容的CAN-FD接口

### 1.2 安装依赖

```bash
# 更新系统
sudo apt update
sudo apt upgrade

# 安装基础工具
sudo apt install -y \
    build-essential \
    git \
    curl \
    wget \
    python3 \
    python3-pip \
    python3-venv

# 安装Python依赖
sudo apt install -y \
    python3-build \
    python3-can \
    python3-serial \
    python3-setuptools \
    python3-pyelftools \
    python3-qtpy \
    python3-wheel \
    python3-importlib-metadata \
    python3-scipy \
    python3-usb \
    mypy \
    nodejs

# 安装CAN工具
sudo apt install -y \
    can-utils \
    python3-can
```

### 1.3 克隆项目

```bash
# 克隆仓库
git clone https://github.com/mjbots/moteus.git
cd moteus

# 或者使用SSH
git clone git@github.com:mjbots/moteus.git
cd moteus
```

### 1.4 验证安装

```bash
# 构建固件
tools/bazel build --config=target //:target

# 运行测试
tools/bazel test --config=host //:host

# 如果都成功，环境设置完成
```

## 2. 代码规范

### 2.1 C++代码规范

遵循Google C++风格指南，关键点：

#### 2.1.1 命名规范

```cpp
// 类名：PascalCase
class MoteusController {
  // ...
};

// 函数名：PascalCase
void InitializeHardware();

// 变量名：snake_case
int motor_position = 0;

// 常量：kPrefix
const int kMaxRetries = 3;

// 枚举值：kPrefix
enum class MotorMode {
  kStopped,
  kOperating,
  kFault,
};

// 命名空间：小写
namespace moteus {
  // ...
}
```

#### 2.1.2 文件格式

```cpp
// 版权头
// Copyright 2023 mjbots Robotic Systems, LLC.  info@mjbots.com
//
// Licensed under the Apache License, Version 2.0 (the "License");
// you may not use this file except in compliance with the License.
// You may obtain a copy of the License at
//
//     http://www.apache.org/licenses/LICENSE-2.0
//
// Unless required by applicable law or agreed to in writing, software
// distributed under the License is distributed on an "AS IS" BASIS,
// WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
// See the License for the specific language governing permissions and
// limitations under the License.

#pragma once  // 头文件保护

// 包含顺序：
// 1. 相关头文件
// 2. C系统头文件
// 3. C++标准库头文件
// 4. 其他库头文件
// 5. 项目头文件

#include "fw/moteus_controller.h"

#include <cstdint>
#include <vector>

#include "mjlib/base/visitor.h"

#include "fw/bldc_servo.h"
```

#### 2.1.3 缩进和空格

```cpp
// 使用2个空格缩进
class MyClass {
 public:
  void MyFunction() {
    if (condition) {
      DoSomething();
    }
  }
  
 private:
  int member_variable_;  // 私有成员以下划线结尾
};

// 不允许尾随空格
// 空行不能包含任何空白字符
```

#### 2.1.4 注释规范

```cpp
// 类注释
/// 这是类的说明文档
/// 可以有多行
class MoteusController {
  // ...
};

// 函数注释
/// 初始化硬件
/// @param config 配置参数
/// @return 成功返回true
bool InitializeHardware(const Config& config);

// 行内注释
int value = 42;  // 魔术数字，需要解释
```

### 2.2 Python代码规范

遵循PEP8规范：

#### 2.2.1 命名规范

```python
# 类名：PascalCase
class MoteusController:
    pass

# 函数名：snake_case
def initialize_hardware():
    pass

# 变量名：snake_case
motor_position = 0

# 常量：UPPER_SNAKE_CASE
MAX_RETRIES = 3

# 私有成员：_prefix
class MyClass:
    def __init__(self):
        self._private_var = 0
```

#### 2.2.2 文件格式

```python
#!/usr/bin/env python3

# 版权头
# Copyright 2023 mjbots Robotic Systems, LLC.  info@mjbots.com
#
# Licensed under the Apache License, Version 2.0 (the "License");
# ...

"""模块说明文档"""

import os
import sys

from moteus import protocol
from moteus import transport

# 代码...
```

#### 2.2.3 类型提示

```python
from typing import Optional, Dict, List

def query_controller(
    controller_id: int,
    timeout: float = 1.0
) -> Optional[Dict[str, float]]:
    """查询控制器状态
    
    Args:
        controller_id: 控制器ID
        timeout: 超时时间（秒）
    
    Returns:
        状态字典，失败返回None
    """
    pass
```

### 2.3 代码格式化

**C++格式化**：
```bash
# 使用clang-format
clang-format -i fw/*.h fw/*.cc

# 或者使用Bazel目标
tools/bazel run //:clang-format
```

**Python格式化**：
```bash
# 使用yapf
yapf -i lib/python/moteus/*.py

# 或者使用black
black lib/python/moteus/
```

## 3. 开发工作流

### 3.1 分支策略

```bash
# 主分支
main          # 稳定版本
develop       # 开发分支

# 功能分支
feature/xxx   # 新功能
bugfix/xxx    # Bug修复
hotfix/xxx    # 紧急修复

# 发布分支
release/v1.0  # 发布准备
```

### 3.2 开发流程

1. **创建功能分支**
```bash
git checkout develop
git pull origin develop
git checkout -b feature/my-feature
```

2. **开发和测试**
```bash
# 编写代码
# 运行测试
tools/bazel test --config=host //:host

# 构建固件
tools/bazel build --config=target //:target
```

3. **提交代码**
```bash
git add .
git commit -m "feat: add new feature"
```

4. **推送和创建PR**
```bash
git push origin feature/my-feature
# 在GitHub上创建Pull Request
```

5. **代码审查**
- 等待审查
- 根据反馈修改
- 审查通过后合并

### 3.3 Commit Message规范

使用Conventional Commits格式：

```
<type>(<scope>): <subject>

<body>

<footer>
```

**类型**：
- `feat`: 新功能
- `fix`: Bug修复
- `docs`: 文档更新
- `style`: 代码格式（不影响功能）
- `refactor`: 重构
- `test`: 测试相关
- `chore`: 构建/工具相关

**示例**：
```
feat(fw): add field weakening support

Implement field weakening algorithm to extend motor speed range.
The algorithm reduces q-axis current and injects d-axis current
when back-EMF approaches supply voltage.

Closes #123
```

## 4. 固件开发

### 4.1 修改固件

#### 4.1.1 添加新功能

1. **定义接口**
```cpp
// fw/my_feature.h
#pragma once

namespace moteus {

class MyFeature {
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
  
  MyFeature(PersistentConfig* config);
  
  void Start();
  void Poll();
  
 private:
  Config config_;
};

}  // namespace moteus
```

2. **实现功能**
```cpp
// fw/my_feature.cc
#include "fw/my_feature.h"

namespace moteus {

MyFeature::MyFeature(PersistentConfig* config) {
  config->Register("my_feature", &config_);
}

void MyFeature::Start() {
  // 初始化
}

void MyFeature::Poll() {
  // 周期性任务
}

}  // namespace moteus
```

3. **集成到主控制器**
```cpp
// fw/moteus_controller.cc
#include "fw/my_feature.h"

class MoteusController::Impl {
  // ...
  MyFeature my_feature_;
};

MoteusController::Impl::Impl(...) 
    : my_feature_(persistent_config) {
}

void MoteusController::Poll() {
  // ...
  my_feature_.Poll();
}
```

4. **更新BUILD文件**
```python
# fw/BUILD
cc_library(
    name = "my_feature",
    hdrs = ["my_feature.h"],
    srcs = ["my_feature.cc"],
    deps = [
        ":common",
        "@com_github_mjbots_mjlib//mjlib/micro:persistent_config",
    ],
)
```

#### 4.1.2 添加新寄存器

```cpp
// fw/my_registers.h
#pragma once

#include "mjlib/base/visitor.h"

namespace moteus {

struct MyRegisters {
  uint32_t reg1 = 0;
  float reg2 = 0.0f;
  
  template <typename Visitor>
  void Accept(Visitor& v) {
    v.Visit(MJ_NVP(reg1), &reg1);
    v.Visit(MJ_NVP(reg2), &reg2);
  }
};

}  // namespace moteus
```

### 4.2 调试固件

#### 4.2.1 使用GDB

```bash
# 启动GDB服务器（在目标板上）
# 使用SWD调试器连接

# 在主机上启动GDB
arm-none-eabi-gdb bazel-bin/fw/moteus

# 连接到目标
(gdb) target remote localhost:3333

# 设置断点
(gdb) break MoteusController::Poll

# 运行
(gdb) continue
```

#### 4.2.2 使用遥测

```cpp
// 添加遥测数据
struct TelemetryData {
  float value1;
  int value2;
  
  template <typename Visitor>
  void Accept(Visitor& v) {
    v.Visit(MJ_NVP(value1), &value1);
    v.Visit(MJ_NVP(value2), &value2);
  }
} telemetry_data_;

// 注册遥测
telemetry_manager->Register("my_telemetry", &telemetry_data_);

// 更新遥测数据
telemetry_data_.value1 = calculate_value();
```

#### 4.2.3 使用串口调试

```cpp
// 启用调试串口
PinName debug_uart_out = PA2;

// 输出调试信息
if (debug_uart_out != NC) {
  char buffer[128];
  snprintf(buffer, sizeof(buffer), "pos=%.3f vel=%.3f\n",
           position, velocity);
  debug_uart->write(buffer);
}
```

### 4.3 测试固件

#### 4.3.1 单元测试

```cpp
// fw/test/my_feature_test.cc
#include <gtest/gtest.h>
#include "fw/my_feature.h"

namespace moteus {
namespace test {

class MyFeatureTest : public ::testing::Test {
 protected:
  void SetUp() override {
    // 初始化测试环境
  }
};

TEST_F(MyFeatureTest, BasicFunctionality) {
  MyFeature feature;
  
  // 测试基本功能
  EXPECT_EQ(feature.Calculate(1.0f), 2.0f);
}

}  // namespace test
}  // namespace moteus
```

#### 4.3.2 仿真测试

```cpp
// fw/test/simulation_my_feature_test.cc
#include "fw/test/simulation_test.h"

TEST(SimulationMyFeatureTest, IntegrationTest) {
  SimulationEnvironment env;
  
  // 配置仿真环境
  env.config.my_feature.enabled = true;
  
  // 运行仿真
  for (int i = 0; i < 1000; i++) {
    env.Step(0.001f);
  }
  
  // 验证结果
  EXPECT_TRUE(env.my_feature.is_active());
}
```

## 5. 客户端库开发

### 5.1 Python库开发

#### 5.1.1 添加新功能

```python
# lib/python/moteus/moteus.py

class Controller:
    async def my_new_feature(self, param1: float) -> dict:
        """新功能说明
        
        Args:
            param1: 参数说明
        
        Returns:
            返回值说明
        """
        # 编码命令
        payload = self._encode_command(param1)
        
        # 发送并接收响应
        response = await self.transport.send_and_receive(payload)
        
        # 解码响应
        return self._decode_response(response)
```

#### 5.1.2 添加测试

```python
# lib/python/moteus/test/my_feature_test.py
import unittest
from moteus import Controller

class MyFeatureTest(unittest.TestCase):
    def test_basic_functionality(self):
        """测试基本功能"""
        controller = Controller(id=1, transport=MockTransport())
        
        result = await controller.my_new_feature(1.0)
        
        self.assertEqual(result['status'], 'success')

if __name__ == '__main__':
    unittest.main()
```

### 5.2 C++库开发

#### 5.2.1 添加新功能

```cpp
// lib/cpp/mjbots/moteus/moteus.h

class Controller {
 public:
  struct MyFeatureResult {
    float value1;
    int value2;
  };
  
  MyFeatureResult MyNewFeature(float param1) {
    // 编码命令
    auto payload = EncodeCommand(param1);
    
    // 发送并接收响应
    auto response = transport_->SendAndReceive(payload);
    
    // 解码响应
    return DecodeResponse(response);
  }
};
```

## 6. 文档编写

### 6.1 代码注释

```cpp
/// 计算目标位置
///
/// 使用梯形轨迹规划算法，根据当前位置和目标位置计算
/// 平滑的运动轨迹。考虑速度和加速度限制。
///
/// @param current_position 当前位置（弧度）
/// @param target_position 目标位置（弧度）
/// @param max_velocity 最大速度（弧度/秒）
/// @param max_acceleration 最大加速度（弧度/秒²）
///
/// @return 规划后的目标位置
///
/// @note 该函数在控制循环中调用，必须保证实时性
/// @warning 如果参数无效，返回当前位置
float PlanTrajectory(
    float current_position,
    float target_position,
    float max_velocity,
    float max_acceleration);
```

### 6.2 README更新

```markdown
## 新功能说明

### 功能名称

简要描述功能。

#### 使用方法

```python
# Python示例
controller.my_new_feature(param1=1.0)
```

```cpp
// C++示例
controller.MyNewFeature(1.0f);
```

#### 参数说明

- `param1`: 参数说明

#### 返回值

返回值说明。

#### 示例

完整示例代码。
```

### 6.3 文档站点

项目使用MkDocs生成文档站点：

```bash
# 安装MkDocs
pip3 install mkdocs mkdocs-material

# 本地预览
cd docs
mkdocs serve

# 构建文档
mkdocs build
```

## 7. 性能优化

### 7.1 固件优化

#### 7.1.1 代码优化

```cpp
// 使用const引用避免拷贝
void ProcessData(const Data& data) {
  // ...
}

// 使用inline减少函数调用开销
inline float Calculate(float x) {
  return x * x;
}

// 使用位运算
int value = flags & 0x0F;  // 提取低4位
flags |= 0x10;              // 设置第5位
flags &= ~0x10;             // 清除第5位
```

#### 7.1.2 内存优化

```cpp
// 使用内存池
class MyObject {
  static mjlib::micro::Pool pool_;
};

// 避免动态内存分配
// ❌ 不好
std::vector<float> data;
data.push_back(value);

// ✅ 好
float data[100];
int count = 0;
data[count++] = value;
```

#### 7.1.3 中断优化

```cpp
// 最小化中断处理时间
void ADC_IRQHandler() {
  // 只读取数据
  uint16_t adc_value = ADC->DR;
  
  // 设置标志，在主循环中处理
  adc_ready = true;
  adc_data = adc_value;
}

// 主循环中处理
void Poll() {
  if (adc_ready) {
    ProcessAdcData(adc_data);
    adc_ready = false;
  }
}
```

### 7.2 客户端优化

#### 7.2.1 异步编程

```python
# ✅ 使用异步
async def query_all(controllers):
    tasks = [c.query() for c in controllers]
    return await asyncio.gather(*tasks)

# ❌ 避免阻塞
def query_all(controllers):
    results = []
    for c in controllers:
        results.append(c.query())  # 阻塞
    return results
```

#### 7.2.2 批量操作

```python
# ✅ 批量读取
configs = await controller.read_config([
    'servopos.position_max',
    'servopos.position_min',
    'servo.max_current_A',
])

# ❌ 逐个读取
config1 = await controller.read_config('servopos.position_max')
config2 = await controller.read_config('servopos.position_min')
config3 = await controller.read_config('servo.max_current_A')
```

## 8. 常见问题

### 8.1 编译错误

**问题**：编译失败，找不到头文件

**解决方案**：
```bash
# 清理缓存
tools/bazel clean --expunge

# 重新构建
tools/bazel build --config=target //:target
```

### 8.2 测试失败

**问题**：测试失败

**解决方案**：
```bash
# 查看详细输出
tools/bazel test --config=host //fw:test --test_output=all

# 调试测试
tools/bazel test --config=host //fw:test --run_under=gdb
```

### 8.3 代码审查

**问题**：代码审查未通过

**解决方案**：
1. 仔细阅读审查意见
2. 修改代码
3. 推送更新
4. 请求重新审查

## 9. 发布流程

### 9.1 版本管理

使用语义化版本：`MAJOR.MINOR.PATCH`

- `MAJOR`: 不兼容的API变更
- `MINOR`: 向后兼容的功能新增
- `PATCH`: 向后兼容的问题修复

### 9.2 发布步骤

1. **更新版本号**
```bash
# 修改版本文件
echo "1.2.3" > VERSION
```

2. **更新CHANGELOG**
```markdown
# Changelog

## [1.2.3] - 2023-10-14

### Added
- 新功能A
- 新功能B

### Fixed
- 修复问题C
```

3. **创建发布分支**
```bash
git checkout -b release/v1.2.3
```

4. **测试和验证**
```bash
# 运行完整测试
tools/bazel test --test_size_filters= --test_tag_filters=-manual --config=host //:host

# 构建固件
tools/bazel build --config=target //:target
```

5. **合并到主分支**
```bash
git checkout main
git merge release/v1.2.3
git tag v1.2.3
git push origin main --tags
```

6. **创建GitHub Release**
- 上传构建产物
- 编写发布说明

## 10. 总结

开发Moteus需要：

- ✅ **熟悉C++和Python**
- ✅ **理解嵌入式系统**
- ✅ **掌握电机控制原理**
- ✅ **遵循代码规范**
- ✅ **编写测试**
- ✅ **编写文档**

遵循本指南，可以高效地开发和贡献代码。

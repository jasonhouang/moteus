# Moteus 编译系统

## 1. 构建系统概述

Moteus项目使用**Bazel**作为主要构建系统，这是一个由Google开发的现代构建工具，具有以下优势：

- **增量构建**：只重新构建修改的部分
- **依赖管理**：自动处理复杂的依赖关系
- **跨平台支持**：支持多种目标平台
- **可重复构建**：确保构建的一致性
- **并行构建**：充分利用多核CPU

### 1.1 为什么选择Bazel

1. **大规模项目支持**：适合包含固件、库、工具的复杂项目
2. **严格的依赖管理**：避免依赖冲突
3. **远程缓存**：加速团队构建
4. **沙箱构建**：确保构建的可重复性
5. **多语言支持**：C++、Python、Rust统一构建

### 1.2 Bazel版本管理

项目使用自带的Bazel wrapper（`tools/bazel`），确保所有开发者使用相同版本的Bazel。

```bash
# 使用项目自带的bazel wrapper
tools/bazel build ...

# 不要使用系统安装的bazel
bazel build ...  # ❌ 不推荐
```

## 2. 构建目标

### 2.1 主要构建目标

项目定义了三个主要的构建目标：

#### 2.1.1 host（主机测试）

用于在开发主机上运行测试，不需要实际硬件。

```bash
# 运行快速测试（推荐日常开发使用）
tools/bazel test --config=host //:host

# 运行完整测试套件（包括仿真回归测试）
tools/bazel test --test_size_filters= --test_tag_filters=-manual --config=host //:host
```

**包含的测试**：
- `//fw:host`：固件单元测试
- `//utils:host`：工具测试
- `//lib/python:host`：Python库测试
- `//lib/cpp:host`：C++库测试
- `//lib/rust:host`：Rust库测试

#### 2.1.2 target（目标固件）

用于构建目标硬件的固件。

```bash
# 构建固件
tools/bazel build --config=target //:target
```

**生成的产物**：
- `//fw:moteus`：主固件ELF文件
- `//fw:bin`：固件二进制文件（.bin）
- `//fw:can_bootloader`：CAN引导加载程序

#### 2.1.3 单独构建组件

```bash
# 只构建固件
tools/bazel build --config=target //fw:moteus

# 只构建引导加载程序
tools/bazel build --config=target //fw:can_bootloader

# 只运行固件测试
tools/bazel test --config=host //fw:host

# 只运行Python测试
tools/bazel test --config=host //lib/python:host
```

## 3. 构建配置

### 3.1 配置文件

构建配置定义在 `tools/bazel.rc` 和各个 `BUILD` 文件中。

#### 3.1.1 主要配置选项

**--config=host**
- 目标平台：主机（x86_64 Linux）
- 用途：运行测试
- 编译器：系统GCC/Clang

**--config=target**
- 目标平台：STM32G4（ARM Cortex-M4）
- 用途：构建固件
- 编译器：ARM GCC交叉编译器
- 优化级别：-O2

### 3.2 编译选项

固件编译使用以下关键选项：

```python
COPTS = [
    "-Werror",              # 警告视为错误
    "-Wdouble-promotion",   # 检测double到float的隐式转换
    "-gdwarf-4",           # 生成DWARF-4调试信息
]
```

### 3.3 链接器脚本

固件使用专门的链接器脚本定义内存布局：

```
fw/stm32g474.ld          # 主固件链接器脚本
fw/stm32g474_bootloader.ld  # 引导加载程序链接器脚本
```

**内存布局**：
```
Flash (512KB):
  0x08000000 - 0x08007FFF: Bootloader (32KB)
  0x08008000 - 0x0807FFFF: Main Firmware (480KB)

SRAM (128KB):
  0x20000000 - 0x2001FFFF: Main SRAM (128KB)

CCM SRAM (32KB):
  0x10000000 - 0x10007FFF: Core-Coupled Memory (32KB)
```

## 4. 依赖管理

### 4.1 外部依赖

项目通过Bazel的工作空间规则管理外部依赖，定义在 `WORKSPACE` 文件中。

#### 4.1.1 主要依赖

**mbed-os (STM32G4)**
```python
mbed_g4_repository()  # STM32G4 mbed-os
```
- 用途：STM32G4硬件抽象层
- 位置：`@com_github_ARMmbed_mbed-g4`

**mbed-os (Bootloader)**
```python
mbed_bootloader_g4_repository()  # 引导加载程序mbed-os
```
- 用途：引导加载程序硬件抽象层
- 位置：`@com_github_ARMmbed_mbed-g4-bootloader`

**mjlib**
```python
mjlib_repository()  # mjbots核心库
```
- 用途：基础工具、协议、微控制器抽象
- 位置：`@com_github_mjbots_mjlib`
- 主要模块：
  - `mjlib/base`：基础工具（流、tokenizer等）
  - `mjlib/micro`：微控制器抽象（PersistentConfig、TelemetryManager等）
  - `mjlib/multiplex`：多路复用协议

**Boost**
```python
boost_repository()  # Boost C++库
```
- 用途：C++标准库补充
- 位置：`@boost`

#### 4.1.2 依赖版本管理

所有依赖版本都通过Git commit hash或版本标签固定，确保构建的可重复性。

```python
http_archive(
    name = "com_github_mjbots_mjlib",
    sha256 = "...",
    strip_prefix = "mjlib-...",
    urls = ["https://github.com/mjbots/mjlib/archive/....tar.gz"],
)
```

### 4.2 离线构建

当网络不可用或下载失败时，可以使用离线构建方法。

#### 4.2.1 预下载依赖

```bash
# 下载所有依赖到本地缓存
bash utils/download_bazel_deps.sh
```

**下载内容**：
- LLVM工具链：~414MB
- Boost库：~124MB
- mbed-os：~88MB
- 其他依赖：~124MB
- **总计**：~650MB

**下载时间**：10-15分钟（取决于网络速度）

#### 4.2.2 使用本地缓存构建

```bash
tools/bazel build --config=target //:target \
    --repository_cache=/tmp/repo_cache \
    --distdir=/tmp/bazel_cache \
    --sandbox_writable_path=/dev
```

**参数说明**：
- `--repository_cache`：指定仓库缓存目录
- `--distdir`：指定分发包目录
- `--sandbox_writable_path=/dev`：解决沙箱对/dev/null的限制（仅在特定VM环境需要）

### 4.3 依赖更新

更新依赖需要修改 `WORKSPACE` 文件：

```python
# 1. 修改版本号或commit hash
http_archive(
    name = "com_github_mjbots_mjlib",
    sha256 = "新的sha256值",
    strip_prefix = "mjlib-新版本",
    urls = ["https://github.com/mjbots/mjlib/archive/新版本.tar.gz"],
)

# 2. 清理缓存
tools/bazel clean --expunge

# 3. 重新构建
tools/bazel build --config=target //:target
```

## 5. BUILD文件结构

### 5.1 根目录BUILD

```python
# 测试套件
test_suite(
    name = "host",
    tests = [
        "//fw:host",
        "//utils:host",
        "//lib/python:host",
        "//lib/cpp:host",
        "//lib/rust:host",
    ],
)

# 目标固件
filegroup(
    name = "target",
    srcs = [
        "//fw:moteus",
        "//fw:bin",
        "//fw:can_bootloader",
    ],
)
```

### 5.2 固件BUILD文件

固件的BUILD文件（`fw/BUILD`）是最复杂的，定义了多个构建目标。

#### 5.2.1 公共库

```python
cc_library(
    name = "common",
    hdrs = [
        "bldc_servo_control.h",
        "bldc_servo_position.h",
        # ... 更多头文件
    ],
    srcs = [
        "foc.cc",
    ],
    deps = [
        "@com_github_mjbots_mjlib//mjlib/base:limit",
        "@com_github_mjbots_mjlib//mjlib/base:visitor",
        # ... 更多依赖
    ],
    copts = COPTS,
)
```

#### 5.2.2 主固件

```python
g4_mbed_binary(
    name = "moteus",
    srcs = MOTEUS_SOURCES + [
        "main.cc",
        "moteus_controller.cc",
        "bldc_servo.cc",
        # ... 更多源文件
    ],
    deps = [
        ":common",
        ":git_info",
        "@com_github_mjbots_mjlib//mjlib/micro:persistent_config",
        # ... 更多依赖
    ],
    linker_script = "stm32g474.ld",
    copts = COPTS,
)
```

#### 5.2.3 引导加载程序

```python
g4_bootloader_mbed_binary(
    name = "can_bootloader",
    srcs = [
        "bootloader.h",
        "can_bootloader.cc",
        "stm32g4xx_fdcan_typedefs.h",
    ],
    deps = [
        ":git_info",
        "@com_github_mjbots_mjlib//mjlib/base:buffer_stream",
        # ... 更多依赖
    ],
    linker_script = "stm32g474_bootloader.ld",
    enable_wrappers = False,
    copts = COPTS,
)
```

#### 5.2.4 测试目标

```python
cc_test(
    name = "host",
    size = "small",
    srcs = [
        "test/foc_test.cc",
        "test/motor_position_test.cc",
        # ... 更多测试文件
    ],
    deps = [
        ":common",
        "@com_github_mjbots_mjlib//mjlib/base:limit",
        # ... 更多依赖
    ],
    copts = COPTS,
)
```

### 5.3 Python库BUILD文件

```python
py_library(
    name = "moteus",
    srcs = glob(["*.py"]),
    deps = [
        # Python依赖
    ],
    visibility = ["//visibility:public"],
)

py_test(
    name = "host",
    srcs = glob(["test/*_test.py"]),
    deps = [":moteus"],
)
```

### 5.4 C++库BUILD文件

```python
cc_library(
    name = "moteus",
    hdrs = [
        "moteus.h",
        "moteus_protocol.h",
        "moteus_transport.h",
        # ... 更多头文件
    ],
    visibility = ["//visibility:public"],
)

cc_test(
    name = "host",
    srcs = ["test/moteus_test.cc"],
    deps = [":moteus"],
)
```

## 6. 构建规则

### 6.1 自定义规则

项目使用自定义Bazel规则来简化构建配置。

#### 6.1.1 mbed_binary规则

用于构建基于mbed-os的STM32固件。

```python
load("@com_github_ARMmbed_mbed-g4//:rules.bzl", "mbed_binary")

mbed_binary(
    name = "moteus",
    srcs = [...],
    deps = [...],
    linker_script = "stm32g474.ld",
)
```

**功能**：
- 自动配置ARM交叉编译器
- 应用mbed-os编译选项
- 生成ELF、BIN、HEX文件
- 计算校验和

#### 6.1.2 mbed_bootloader规则

专门用于构建引导加载程序。

```python
load("@com_github_ARMmbed_mbed-g4-bootloader//:rules.bzl", "mbed_binary")

g4_bootloader_mbed_binary(
    name = "can_bootloader",
    srcs = [...],
    deps = [...],
    linker_script = "stm32g474_bootloader.ld",
    enable_wrappers = False,
)
```

### 6.2 规则参数

**srcs**：源文件列表
```python
srcs = [
    "main.cc",
    "moteus_controller.cc",
    # ...
]
```

**hdrs**：头文件列表
```python
hdrs = [
    "moteus_controller.h",
    "bldc_servo.h",
    # ...
]
```

**deps**：依赖列表
```python
deps = [
    ":common",  # 本地依赖
    "@com_github_mjbots_mjlib//mjlib/base:limit",  # 外部依赖
]
```

**copts**：编译选项
```python
copts = [
    "-Werror",
    "-Wdouble-promotion",
    "-gdwarf-4",
]
```

**linker_script**：链接器脚本
```python
linker_script = "stm32g474.ld"
```

## 7. 构建产物

### 7.1 固件产物

构建完成后生成的文件：

```
bazel-bin/fw/
├── moteus              # ELF文件（包含调试信息）
├── moteus.bin          # 二进制文件（用于烧录）
├── moteus.hex          # Intel HEX文件（用于烧录）
├── moteus.map          # 内存映射文件
├── can_bootloader      # 引导加载程序ELF
├── can_bootloader.bin  # 引导加载程序二进制
└── can_bootloader.hex  # 引导加载程序HEX
```

### 7.2 查看构建产物

```bash
# 查看构建输出目录
ls -la bazel-bin/fw/

# 查看固件大小
arm-none-eabi-size bazel-bin/fw/moteus

# 查看固件符号
arm-none-eabi-nm bazel-bin/fw/moteus | sort

# 反汇编
arm-none-eabi-objdump -d bazel-bin/fw/moteus > disassembly.txt
```

### 7.3 固件大小分析

```bash
# 查看各段大小
arm-none-eabi-size bazel-bin/fw/moteus
   text    data     bss     dec     hex filename
 123456    1234    5678  130368   1fd40 bazel-bin/fw/moteus

# text: 代码段（Flash）
# data: 已初始化数据（Flash + RAM）
# bss: 未初始化数据（RAM）
```

## 8. 构建优化

### 8.1 增量构建

Bazel自动跟踪文件变化，只重新构建受影响的部分。

```bash
# 修改源文件后重新构建
tools/bazel build --config=target //:target
# 只重新编译修改的文件和依赖它的文件
```

### 8.2 并行构建

Bazel默认使用所有可用CPU核心进行并行构建。

```bash
# 限制并行任务数
tools/bazel build --config=target //:target --jobs=4

# 使用所有核心（默认）
tools/bazel build --config=target //:target
```

### 8.3 构建缓存

```bash
# 清理构建缓存（保留下载的依赖）
tools/bazel clean

# 完全清理（包括下载的依赖）
tools/bazel clean --expunge

# 查看缓存大小
du -sh ~/.cache/bazel
```

### 8.4 构建分析

```bash
# 分析构建性能
tools/bazel build --config=target //:target --profile=profile.gz

# 使用Chrome打开性能分析
chrome://tracing/  # 加载 profile.gz
```

## 9. 常见问题

### 9.1 构建失败：无法下载依赖

**问题**：
```
ERROR: Error downloading ...
```

**解决方案**：

1. **使用离线构建**：
```bash
bash utils/download_bazel_deps.sh
tools/bazel build --config=target //:target \
    --repository_cache=/tmp/repo_cache \
    --distdir=/tmp/bazel_cache
```

2. **检查网络连接**：
```bash
# 测试GitHub连接
curl -I https://github.com
```

3. **配置代理**（如果需要）：
```bash
export http_proxy=http://proxy:port
export https_proxy=http://proxy:port
```

### 9.2 构建失败：沙箱权限错误

**问题**：
```
ERROR: ... Permission denied: /dev/null
```

**解决方案**：
```bash
tools/bazel build --config=target //:target --sandbox_writable_path=/dev
```

### 9.3 构建失败：编译器错误

**问题**：
```
error: ...
```

**解决方案**：

1. **检查编译器版本**：
```bash
arm-none-eabi-gcc --version
```

2. **清理并重新构建**：
```bash
tools/bazel clean --expunge
tools/bazel build --config=target //:target
```

3. **查看详细错误**：
```bash
tools/bazel build --config=target //:target --verbose_failures
```

### 9.4 构建失败：内存不足

**问题**：
```
error: out of memory
```

**解决方案**：
```bash
# 限制并行任务数
tools/bazel build --config=target //:target --jobs=2

# 增加交换空间
sudo swapon --show
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
```

### 9.5 测试失败：找不到测试

**问题**：
```
ERROR: No tests found
```

**解决方案**：
```bash
# 列出所有测试
tools/bazel query 'tests(//...)'

# 运行特定测试
tools/bazel test --config=host //fw/test:foc_test
```

## 10. 高级主题

### 10.1 自定义构建配置

创建自定义配置文件 `tools/custom.rc`：

```python
build:custom --compilation_mode=opt
build:custom --copt=-O3
build:custom --copt=-mcpu=cortex-m4
build:custom --copt=-mfloat-abi=hard
build:custom --copt=-mfpu=fpv4-sp-d16
```

使用自定义配置：
```bash
tools/bazel build --config=custom //:target
```

### 10.2 交叉编译

项目已经配置好ARM交叉编译，无需额外配置。

**交叉编译器**：ARM GCC（通过mbed规则自动配置）

**目标架构**：
- CPU: Cortex-M4
- FPU: FPv4-SP-D16
- ABI: Hard float

### 10.3 调试构建

创建调试构建配置：

```bash
# 使用调试优化级别
tools/bazel build --config=target //:target \
    --compilation_mode=dbg \
    --copt=-O0 \
    --copt=-g3
```

### 10.4 发布构建

创建发布构建配置：

```bash
# 使用最高优化级别
tools/bazel build --config=target //:target \
    --compilation_mode=opt \
    --copt=-O3 \
    --copt=-DNDEBUG \
    --linkopt=-s
```

### 10.5 构建统计

```bash
# 查看构建统计
tools/bazel build --config=target //:target --stats

# 查看依赖图
tools/bazel query --output=graph //fw:moteus > deps.dot
dot -Tpng deps.dot -o deps.png
```

## 11. CI/CD集成

### 11.1 GitHub Actions

项目使用GitHub Actions进行持续集成，配置在 `.github/workflows/` 目录。

**主要工作流**：
- 构建固件
- 运行测试
- 代码质量检查
- 发布版本

### 11.2 本地CI脚本

```bash
# 运行完整的CI测试
./travis-ci.sh
```

该脚本执行：
1. 构建固件
2. 运行所有测试
3. 代码格式检查
4. 静态分析

### 11.3 预提交检查

建议在提交前运行：

```bash
# 运行快速测试
tools/bazel test --config=host //:host

# 构建固件
tools/bazel build --config=target //:target
```

## 12. 最佳实践

### 12.1 日常开发

1. **使用快速测试**：
```bash
tools/bazel test --config=host //:host
```

2. **增量构建**：
```bash
# 只构建修改的部分
tools/bazel build --config=target //fw:moteus
```

3. **定期清理**：
```bash
# 每周清理一次
tools/bazel clean
```

### 12.2 发布构建

1. **完全清理**：
```bash
tools/bazel clean --expunge
```

2. **使用离线缓存**：
```bash
bash utils/download_bazel_deps.sh
tools/bazel build --config=target //:target \
    --repository_cache=/tmp/repo_cache \
    --distdir=/tmp/bazel_cache
```

3. **验证构建**：
```bash
# 运行所有测试
tools/bazel test --test_size_filters= --test_tag_filters=-manual --config=host //:host

# 检查固件大小
arm-none-eabi-size bazel-bin/fw/moteus
```

### 12.3 团队协作

1. **统一Bazel版本**：始终使用 `tools/bazel` wrapper
2. **共享缓存**：使用远程缓存服务器
3. **依赖锁定**：使用精确的版本号
4. **文档更新**：更新依赖时更新文档

## 13. 总结

Moteus的Bazel构建系统提供了：

- ✅ **可靠的构建**：可重复、增量、并行
- ✅ **完整的依赖管理**：自动下载、版本锁定
- ✅ **多平台支持**：主机测试 + 目标固件
- ✅ **灵活的配置**：支持多种构建模式
- ✅ **强大的工具**：分析、调试、优化

掌握Bazel构建系统是高效开发Moteus的关键。

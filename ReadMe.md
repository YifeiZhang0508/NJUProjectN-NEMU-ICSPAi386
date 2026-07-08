# NEMU — x86 模拟编程实践

## 项目简介

本仓库是 MOOC 课程《计算机系统基础（五）：x86 模拟器编程实践》的代码仓库。

PA 实验的目标是实现 **NEMU**（NJU Emulator），一个用 C 语言编写的简化 i386 模拟器。它以用户态软件的形式运行，能够执行 i386 指令集程序。通过分阶段实现 CPU 模拟、内存管理、I/O 设备访问等功能，深入理解计算机系统的底层原理。

请先获取对应的实验手册再开展实验，实验手册地址：

* GitHub: http://github.com/ics-nju-wl/icspa-public-guide
* Gitee: https://gitee.com/wlicsnju/icspa-public-guide

本代码源仓库及其镜像地址：

* GitHub: http://github.com/ics-nju-wl/icspa-public
* Gitee: https://gitee.com/wlicsnju/icspa-public

## 项目结构

```
pa_nju/
├── game/               // 游戏相关代码
├── include/            // PA 整体依赖的公共头文件
│   ├── config.h        // 配置宏
│   └── trap.h          // trap 相关定义（不可改动）
├── kernel/             // 微型操作系统内核
├── libs/               // 框架代码使用的库（不可改动）
├── scripts/            // 框架代码功能脚本（不可改动）
├── testcase/           // 测试用例
├── nemu/               // NEMU 模拟器核心
│   ├── include/        // NEMU 头文件
│   ├── src/            // NEMU 源代码
│   │   ├── cpu/        // CPU 模拟（ALU、FPU、指令解码等）
│   │   ├── device/     // I/O 设备模拟
│   │   ├── memory/     // 内存与缓存模拟
│   │   ├── monitor/    // 调试监视器（断点、表达式求值、ELF 加载）
│   │   ├── main.c      // NEMU 入口
│   │   └── parse_args.c// 命令行参数解析
│   └── Makefile
├── Makefile            // 顶层 Makefile
├── Makefile.git        // Git 提交相关辅助
└── ReadMe.md
```

## 在 x86_64 WSL 中编译运行

原始项目为 32 位 (i386) 环境设计。在 x86_64 WSL 中编译需要额外的适配工作。

### 一、安装依赖

#### 1. 启用 i386 架构支持

```bash
sudo dpkg --add-architecture i386
sudo apt-get update
```

#### 2. 安装 32 位开发库

```bash
sudo apt-get install -y \
    gcc-multilib \
    g++-multilib \
    libc6-dev-i386 \
    libreadline-dev:i386 \
    libsdl1.2-dev:i386
```

### 二、编译与运行

```bash
make clean       # 清理旧文件
make nemu        # 编译 nemu、testcase、kernel
make test_pa-1   # 运行 PA-1 阶段测试
```

### 三、GCC 库路径问题排查

`nemu/Makefile` 中的 `LDFLAGS` 包含 `-L/usr/lib/gcc/x86_64-linux-gnu/15/32`，其中版本号 `15` 对应 GCC 15。如果你的系统安装了不同版本的 GCC，链接时会出现类似以下错误：

```
cannot find -lgcc: No such file or directory
```

**排查方法：**

```bash
# 查看当前 GCC 版本
gcc --version

# 查找实际的 32 位 GCC 库路径
find /usr/lib/gcc -name 'libgcc*' -type f 2>/dev/null
```

输出类似：

```
/usr/lib/gcc/x86_64-linux-gnu/13/libgcc.a
/usr/lib/gcc/x86_64-linux-gnu/13/32/libgcc.a
```

其中 `/usr/lib/gcc/x86_64-linux-gnu/13/32` 就是你的系统上 32 位 GCC 库的实际路径。

**修复方式：**

编辑 `nemu/Makefile`，将 `LDFLAGS` 中的版本号替换为你的实际版本：

```makefile
# 如果你的 GCC 版本是 13，将 15 改为 13
LDFLAGS := -m32 -Wl,-m,elf_i386 -L/usr/lib/i386-linux-gnu -L/usr/lib/gcc/x86_64-linux-gnu/13/32 -lreadline -lSDL
```

也可以用一条命令自动获取版本号：

```bash
# 自动检测 GCC 主版本号并替换
GCC_VER=$(gcc -dumpversion | cut -d. -f1)
sed -i "s|x86_64-linux-gnu/[0-9]*/32|x86_64-linux-gnu/${GCC_VER}/32|g" nemu/Makefile
```

---

## 修改说明

以下列出为适配 x86_64 WSL + GCC 15 环境所做的全部修改。

### Makefile 修改

#### 1. `nemu/Makefile`

```makefile
# 修改前
CFLAGS := -ggdb3 -MMD -MP -Wall -Werror -O2 -I./include ...
LDFLAGS := -lreadline -lSDL

# 修改后
CFLAGS := -std=gnu11 -m32 -ggdb3 -MMD -MP -Wall -O2 -I./include ...
LDFLAGS := -m32 -Wl,-m,elf_i386 -L/usr/lib/i386-linux-gnu -L/usr/lib/gcc/x86_64-linux-gnu/15/32 -lreadline -lSDL
```

| 改动 | 说明 |
|------|------|
| `CFLAGS` 添加 `-std=gnu11` | GCC 15 默认 C23 标准中 `bool` 是关键字，与 `nemu.h` 中 `typedef uint8_t bool;` 冲突 |
| `CFLAGS` 添加 `-m32` | 生成 32 位代码 |
| `CFLAGS` 移除 `-Werror` | 新版编译器新增的警告不应阻断编译 |
| `LDFLAGS` 添加 `-m32` | 链接时指定 32 位模式 |
| `LDFLAGS` 添加 `-Wl,-m,elf_i386` | 指定 ELF 格式为 i386 |
| `LDFLAGS` 添加 `-L/usr/lib/i386-linux-gnu` | 指定 32 位系统库路径 |
| `LDFLAGS` 添加 `-L/.../15/32` | 指定 32 位 GCC 运行时库路径（版本号需按实际调整） |

#### 2. `kernel/Makefile`

```makefile
# 修改前
CFLAGS = -m32 -MMD -Wall -Werror -march=i386 ...

# 修改后
CFLAGS = -std=gnu11 -m32 -MMD -Wall -march=i386 ...
```

| 改动 | 说明 |
|------|------|
| 添加 `-std=gnu11` | 解决 `bool` 类型定义冲突 |
| 移除 `-Werror` | 提升编译兼容性 |

#### 3. `testcase/Makefile`

```makefile
# 修改前
CFLAGS := -ggdb3 -MMD -MP -Wall -m32 -march=i386 ...

# 修改后
CFLAGS := -std=gnu11 -ggdb3 -MMD -MP -Wall -m32 -march=i386 ...
```

| 改动 | 说明 |
|------|------|
| 添加 `-std=gnu11` | 解决 `bool` 类型定义和 `asm` 关键字问题 |

#### 4. `game/Makefile`

```makefile
# 修改前
CFLAGS = -m32 -MMD -Wall -Werror \ ...

# 修改后
CFLAGS = -m32 -MMD -Wall \ ...
```

| 改动 | 说明 |
|------|------|
| 移除 `-Werror` | 提升编译兼容性 |

### 源代码修改

#### 1. `nemu/src/parse_args.c`（第 23 行）

```c
// 修改前
static bool main_arg_expr_score()

// 修改后
static bool main_arg_expr_score(char *noUse)
```

**原因：** 该函数通过函数指针 `bool (*handler)(char *)` 调用，签名必须匹配。

#### 2. `nemu/src/cpu/cpu.c`（第 10-12 行）

```c
// 修改前
CPU_STATE cpu;
FPU fpu;
int nemu_state;

// 修改后
CPU_STATE cpu;
int nemu_state;
```

**原因：** `FPU fpu` 已在 `nemu/src/cpu/fpu.c` 中定义，头文件 `cpu/fpu.h` 中已有 `extern FPU fpu` 声明，此处重复定义会导致链接时 multiple definition 错误。

---

## 问题分析

### 1. 32 位与 64 位兼容

原始项目面向 32 位 (i386) 环境。在 x86_64 系统上编译需要：
- 编译选项添加 `-m32`，链接选项添加 `-m32` 和 `-Wl,-m,elf_i386`
- 安装 32 位开发库（`gcc-multilib`、`libc6-dev-i386` 等）
- 指定 32 位链接器库路径

### 2. C 标准版本

GCC 15 默认使用 C23 标准，其中 `bool` 成为保留关键字，与项目中 `typedef uint8_t bool;` 的定义冲突。使用 `-std=gnu11` 指定 GNU C11 标准可解决此问题。

> 注意：使用 `-std=c11` 虽然也能避免 `bool` 冲突，但会导致 `asm` 关键字不可用（需改写为 `__asm__`），因此推荐使用 `-std=gnu11`。

### 3. `-Werror` 兼容性

`-Werror` 将所有编译警告视为错误。新版 GCC 会引入新的警告类型（如 `-Wincompatible-pointer-types`），导致原本可编译的代码编译失败。移除 `-Werror` 可提升跨版本兼容性。

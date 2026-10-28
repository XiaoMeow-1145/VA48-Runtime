> ⚠️ **注意：本项目目前仅为概念型项目，尚未经过实际构建与实验验证。所有设计和功能设想均以项目正式构建及实际实验结果为准。**

# VA48 Runtime

> Android 用户态 VA48 兼容运行时  
> A pluggable userspace VA48 compatibility runtime for Android PC gaming environments.

## 项目简介

VA48 Runtime 是一个面向 Android PC 游戏运行环境的实验性用户态虚拟地址空间兼容项目。

项目的目标是在**不修改 Android 内核、不刷入专用内核、不依赖特定 Snapdragon 平台**的情况下，为运行在 Android 上的 PC 游戏兼容环境探索一种扩展虚拟地址空间的方案。

目前，部分 Android 设备的 Linux/Android Kernel 使用 39-bit userspace virtual address，而部分 PC 游戏、运行库、模拟器或兼容层可能需要更大的虚拟地址空间，例如 47-bit 或 48-bit VA。

传统解决方案通常需要修改 ARM64 Kernel 配置：

```text
CONFIG_ARM64_VA_BITS_48=y
CONFIG_ARM64_VA_BITS=48
```

这种方式通常需要：

- 设备拥有可用的 Kernel Source。
- 修改 Kernel 配置。
- 编译适配设备的 Kernel。
- 制作并刷入专用 Kernel。
- 处理 Bootloader、AVB、Vendor Boot 等相关问题。
- 针对不同 SoC、厂商和 Android 版本分别适配。

VA48 Runtime 希望探索一种不同的思路：在用户态建立 Guest VA48 到 Host VA39 的地址空间兼容层，让支持该机制的运行环境能够管理超出宿主系统原生地址范围的逻辑虚拟地址。

项目不以修改内核为前提，而是尝试通过软件地址映射、地址转换和运行时集成实现更广泛的兼容性。

---

## 核心思想

假设 Android Kernel 实际提供的用户态虚拟地址空间为 39-bit：

```text
Host VA
0x0000000000
        ↓
0x7FFFFFFFFF
```

而某个运行环境希望使用更大的虚拟地址空间：

```text
Guest VA48
0x000000000000
        ↓
0xFFFFFFFFFFFF
```

VA48 Runtime 尝试建立一个独立的软件地址空间，将 Guest 地址映射到宿主系统能够实际使用的地址。

基本结构：

```text
Application / Guest Runtime
          │
          │ Guest Virtual Address
          ▼
┌─────────────────────────┐
│      VA48 Runtime       │
│                         │
│ Address Space Manager   │
│ Software Page Table     │
│ Software TLB            │
│ Mapping Manager         │
│ Address Translator      │
└────────────┬────────────┘
             │
             │ Guest VA → Host VA
             ▼
┌─────────────────────────┐
│ Android/Linux Kernel    │
│                         │
│ Real Host Address Space │
└─────────────────────────┘
```

逻辑过程：

```text
Guest VA48
    ↓
软件地址映射
    ↓
Host VA
    ↓
真实内存
```

需要注意，Guest VA48 是运行时管理的逻辑地址空间，并不意味着 Android Kernel 或 ARM64 MMU 获得了真正的 48-bit 用户态地址支持。

实际实现需要让执行环境参与地址转换，不能仅依靠普通内存映射 API 创建一个虚假的高地址。

---

## 为什么需要这种方案？

传统 Kernel VA48 的工作方式：

```text
程序
  ↓
真实 48-bit ARM64 VA
  ↓
MMU
  ↓
Kernel Page Table
  ↓
Physical Memory
```

这种方式能够提供真正的硬件虚拟地址空间支持，但通常需要修改并适配 Kernel。

VA48 Runtime 则尝试采用：

```text
程序 / Guest Runtime
         ↓
   VA48 Runtime
         ↓
   Software Mapping
         ↓
   Host VA Space
         ↓
   Android Kernel
```

这种方案可能带来额外的软件开销，但有机会减少对特定内核配置和刷机操作的依赖。

项目希望探索的方向是：

- 减少对专用 Kernel 的依赖。
- 尽可能支持更多 Android 设备。
- 为支持地址转换的 PC 兼容运行环境提供统一接口。
- 允许用户按游戏或运行环境选择是否启用。
- 在兼容性与运行效率之间取得平衡。

---

## 工作模式

VA48 Runtime 计划采用混合地址管理模式，而不是让所有内存操作都经过复杂的软件模拟。

### 1. 低地址快速路径

当 Guest 地址能够直接使用宿主地址，或者能够通过安全、有效的映射机制处理时，尽量采用快速路径。

例如：

```text
Guest VA
0x000000100000
       ↓
Host VA
0x000000100000
```

如果地址能够直接对应：

```text
Guest VA == Host VA
```

就尽量避免不必要的地址转换。

在其他情况下，也可以探索固定偏移映射：

```text
Host VA = Guest VA + Base
```

但这种方式只有在地址范围、映射关系和宿主内存布局满足要求时才能使用，不能保证适用于所有地址。

目标是尽可能保持普通内存操作的效率。

### 2. 高地址兼容路径

当程序需要访问超出 Host Kernel 实际支持范围的地址时，进入软件地址管理机制。

例如：

```text
Guest VA:
0x9000000000
       ↓
Software Address Mapping
       ↓
Host VA:
0x2000000000
```

Guest 地址与 Host 地址不再要求相同。

VA48 Runtime 负责记录两者之间的对应关系，并由集成的执行环境在需要时完成转换。

**重要：普通 ARM64 程序不能仅凭软件页表就直接使用虚假的高地址。** 实际实现必须通过动态二进制翻译器、专门的内存访问机制或其他能够参与地址转换的执行环境完成集成。

---

## Software Page Table

VA48 Runtime 计划维护自己的软件页表，用于管理 Guest 地址空间。

逻辑结构：

```text
Guest VA48
    │
    ├── Address Level
    │
    ├── Address Level
    │
    ├── Address Level
    │
    └── Guest Page
             │
             ▼
        Host Mapping
```

具体数据结构和层级将在实际开发中确定，不要求完全照搬硬件页表。

每个 Guest Page 可以记录：

- Guest Address
- Host Address
- Page Size
- Protection
- Present
- Dirty
- Executable
- Read
- Write
- Private
- Shared

通过这些信息，Runtime 可以独立管理 Guest Address Space，并记录每个逻辑内存区域的状态。

---

## Software TLB

如果每次地址转换都需要完整查询软件页表，可能产生较大的性能开销。

因此，项目计划探索 Software TLB，用于缓存最近使用的地址映射。

```text
Guest VA
    │
    ▼
Software TLB
    │
    ├── Hit → 获取缓存的映射
    │
    └── Miss
          ↓
      Software Page Table
          ↓
      Mapping Manager
```

通过缓存常用映射，减少重复查询。

实际性能收益需要通过基准测试验证。

---

## Lazy Mapping

项目计划采用延迟映射，避免一开始就为整个逻辑 VA48 地址空间分配实际内存。

例如：

```text
Guest VA48
┌──────────────────────────────┐
│                              │
│       逻辑地址空间           │
│                              │
│         实际使用区域         │
│              ↓               │
│         按需建立映射         │
│                              │
└──────────────────────────────┘
```

当程序申请一段内存时，Runtime 可以先记录逻辑地址范围，再根据具体需求分配 Host Memory。

基本流程：

```text
mmap()
   ↓
记录 Guest 地址范围
   ↓
需要实际内存时
   ↓
分配 Host Memory
   ↓
建立映射关系
```

具体实现取决于 Backend 的执行方式。延迟映射并不意味着可以不经过真实内存分配就完成任意读写操作。

---

## Large Page

在宿主内核、内存分配器和运行环境允许的情况下，项目可以探索使用更大的内存映射粒度。

例如：

- 4 KiB
- 16 KiB
- 64 KiB
- 2 MiB

实际支持的大小取决于宿主系统与具体实现。

潜在目标包括减少：

- 页表管理开销。
- 映射操作次数。
- 地址转换管理成本。
- 部分内存访问场景中的管理开销。

大页不一定在所有设备和场景下都能使用，也不保证一定提高性能。

---

## Memory Protection

Runtime 需要尽可能兼容常见内存保护操作。

例如：

```text
PROT_READ
PROT_WRITE
PROT_EXEC
```

以及：

```text
MAP_PRIVATE
MAP_SHARED
MAP_ANONYMOUS
```

这些标志需要结合 Guest 与 Host 的实际内存行为进行处理。

Guest 的保护状态可以由软件层记录，但真正的读写和执行权限仍然需要宿主系统及执行环境配合。

项目还需要处理共享内存、可执行内存、写时复制和内存映射生命周期等问题。

---

## Runtime API

VA48 Core 可以提供类似以下的接口：

```c
va48_init();

va48_mmap();

va48_munmap();

va48_mprotect();

va48_mremap();

va48_translate();

va48_register_process();

va48_get_layout();
```

这些接口目前属于设计设想，最终名称、参数和实现方式需要根据实际架构确定。

其中：

```c
va48_translate()
```

用于查询 Guest 地址与 Host 映射之间的关系。

例如：

```text
Guest:
0x9000000000
       ↓
VA48 Runtime
       ↓
Host:
0x3000000000
```

这只是逻辑示例，不代表当前已经实现了对应映射。

---

## 重要技术限制

VA48 Runtime 不能简单地通过：

```text
LD_PRELOAD=libva48.so
```

就让任意 ARM64 程序直接访问：

```text
0x9000000000
```

原因是 ARM64 CPU 的普通 load/store 指令仍然会访问真实的 Host 虚拟地址。

例如：

```asm
ldr x0, [x1]
```

如果：

```text
x1 = 0x9000000000
```

而宿主内核不支持这个地址，那么访问仍然可能触发 Address Fault。

普通的 C 库拦截、mmap 包装或软件页表无法自动改变 CPU 对内存指令的解释方式。

因此，真正的 VA48 兼容机制必须与执行环境集成，使其能够处理 Guest 地址与 Host 地址之间的差异。

这也是项目计划研究 FEX 和 Box64 等动态二进制翻译器的原因。

---

## Backend 架构

项目计划采用 Backend 架构，将核心地址空间管理逻辑与具体运行环境分离。

```text
VA48 Core
    │
    ├── FEX Backend
    │
    ├── Box64 Backend
    │
    └── PRoot Backend
```

不同 Backend 的能力和限制并不相同。

并非所有 Backend 都能独立提供完整的 Guest VA48 支持。具体需要修改哪些接口、哪些内存操作能够被正确处理，都需要在开发过程中确认。

---

## FEX

FEX 是一种面向 ARM64 Linux 的 x86/x86-64 动态二进制翻译器，可以与 Wine 等 Windows 兼容层配合使用。

项目计划研究 FEX 的：

- JIT。
- Memory Manager。
- Guest Address Space。
- Syscall Translation。
- 内存映射管理机制。

希望通过 FEX Backend，让 VA48 Runtime 能够参与 Guest 地址空间管理。

预期架构：

```text
x86/x86-64 Program
         ↓
        FEX
         ↓
    VA48 Backend
         ↓
     Guest VA48
         ↓
     Host Mapping
         ↓
   Android Kernel
```

FEX 是项目优先研究的集成目标之一，但目前尚未证明该方案能够直接适配 FEX 的现有架构。

---

## Box64

Box64 是一种能够在 ARM64 Linux 上运行 x86-64 Linux 程序的动态重编译器。

项目计划研究 Box64 的：

- Dynamic Recompiler。
- Memory Management。
- Guest Address Handling。
- mmap 相关处理。
- 运行时配置机制。

预期架构：

```text
x86-64 Linux Program
         ↓
       Box64
         ↓
    VA48 Backend
         ↓
     Host Mapping
         ↓
   Android Kernel
```

Box64 的实际集成方式需要根据其内部内存管理机制确定。

---

## PRoot

PRoot 可以参与 Linux 用户空间环境的构建，并提供路径、进程和部分系统调用相关的兼容处理。

项目可以研究 PRoot Backend，用于协助管理相关的运行环境和内存映射操作。

但是：

**PRoot 本身不能让 ARM64 CPU 的普通 load/store 指令直接访问虚假的高地址。**

因此，PRoot 更适合作为辅助集成目标，而不是直接实现完整 VA48 地址转换的唯一组件。

对于真正依赖高虚拟地址的程序，仍然需要能够参与地址转换的执行环境。

---

## Wine

Wine 通常不是 VA48 Runtime 的底层 Backend，而是上层 Windows 兼容环境的一部分。

预期结构：

```text
Windows Game
      ↓
     Wine
      ↓
  FEX / Box64
      ↓
  VA48 Runtime
      ↓
  Android Kernel
```

实际运行链取决于具体的 CPU 架构、Wine 构建方式、运行时组合和游戏需求。

VA48 Runtime 的目标是探索地址空间兼容，而不是替代 Wine 或 CPU 动态二进制翻译器。

---

## DroidDeck 集成

DroidDeck 是本项目计划优先支持的 Android PC 游戏运行环境。

但 DroidDeck 本身不应该负责实现 VA48 的核心地址转换机制。

预期架构：

```text
DroidDeck
     ↓
VA48 Plugin / Runtime
     ↓
  VA48 Core
     ↓
FEX / Box64 / PRoot
     ↓
    Wine
     ↓
 Windows Game
```

DroidDeck 主要负责：

- 发现插件。
- 读取 Runtime 配置。
- 选择兼容运行环境。
- 启动 Runtime。
- 传递环境变量。
- 传递启动参数。
- 管理游戏级别的启用状态。

真正的地址空间管理逻辑由 VA48 Runtime 负责。

具体插件接口需要根据 DroidDeck 实际提供的扩展能力进一步设计，不能假设其已经支持本文设想的插件机制。

---

## DroidDeck 用户界面设想

理想情况下，用户可以在游戏设置中看到类似选项：

```text
Game Settings

Compatibility
    Runtime:
        FEX

    Wine:
        Proton Experimental

VA48 Runtime:
    Enabled: ✓

    Mode:
        Balanced

    Lazy Mapping:
        ✓

    Large Page:
        ✓
```

以上为界面设计设想，并不代表 DroidDeck 目前已经支持这些选项。

---

## Per-Game 配置

项目计划支持针对不同游戏分别配置。

例如：

```text
Game A
VA48: OFF
```

```text
Game B
VA48: ON
Mode: Balanced
```

```text
Game C
VA48: ON
Mode: Compatibility
```

这样可以避免所有游戏都强制使用同一种地址空间管理方式。

具体配置文件格式和 DroidDeck 集成方式需要在实际开发阶段确定。

---

## Runtime Profile

项目计划提供不同的运行配置。

### Performance

尽可能减少软件转换与额外管理开销。

```text
Low Address
     ↓
  Fast Path
```

适用于不需要扩展地址空间的程序，或者经过验证可以使用快速路径的场景。

### Balanced

默认目标模式。

```text
Low Address
     ↓
  Fast Path

Extended Address
     ↓
Software Mapping
```

尝试在兼容性与运行效率之间取得平衡。

### Compatibility

优先保证地址映射和内存操作的正确性。

更多内存操作可能经过 Runtime 管理，例如：

```text
mmap
munmap
mprotect
mremap
Address Translation
```

该模式可能带来更大的性能开销。

以上模式目前均为设计设想，实际效果需要通过实验确定。

---

## 项目结构

初期可以采用以下目录结构：

```text
va48-runtime/
│
├── include/
│   └── va48.h
│
├── src/
│   ├── address_space.c
│   ├── memory_manager.c
│   ├── page_table.c
│   ├── address_translate.c
│   ├── mapping.c
│   └── syscall_bridge.c
│
├── backends/
│   ├── fex/
│   ├── box64/
│   └── proot/
│
├── android/
│   ├── libva48.so
│   └── va48-helper
│
├── launcher/
│   └── va48-wrapper
│
├── config/
│   └── default.json
│
├── manifest.json
│
└── tests/
    ├── mmap_test
    ├── page_table_test
    ├── translation_test
    └── va48_test
```

该结构仅为初始参考，最终需要根据实际代码和 Backend 接口调整。

---

## Plugin Manifest

为了让 DroidDeck 或其他兼容运行环境识别 Runtime，可以设计一个 Manifest。

例如：

```json
{
    "name": "VA48 Runtime",
    "version": "0.1.0",
    "type": "compatibility-runtime",
    "architecture": "arm64",
    "backends": [
        "fex",
        "box64",
        "proot"
    ],
    "features": [
        "userspace-va48",
        "lazy-mapping",
        "software-tlb",
        "large-page"
    ]
}
```

其中列出的功能代表计划支持的能力，不应在功能尚未实现时被视为已经可用。

---

## 第一阶段：VA48 Core

第一阶段不直接接入 DroidDeck，而是先在 Android VA39 设备上验证核心设计。

主要目标：

```text
Guest VA48
     ↓
Software Page Table
     ↓
Host VA39 Mapping
```

计划完成：

- Address Space Manager。
- Guest → Host Mapping。
- Software Page Table。
- Software TLB。
- Lazy Mapping。
- Memory Protection。
- mmap / munmap abstraction。
- mprotect abstraction。
- mremap abstraction。
- Address Translation。
- 基础压力测试。

首先需要证明软件地址空间管理逻辑能够正确处理 Guest 地址，而不是直接尝试运行大型 Windows 游戏。

---

## 第一阶段测试

选择一些超出宿主设备常规用户态地址范围的 Guest 地址进行测试。

例如：

```text
0x8000000000
0x9000000000
0x10000000000
0x40000000000
```

这些地址可以作为逻辑 Guest VA48 地址进行测试，但能否实际使用取决于 Guest 地址空间设计与具体实现。

计划测试：

```text
allocate
write
read
translate
protect
unmap
```

例如：

```text
Guest Address:
0x9000000000

Write:
Hello VA48

       ↓

VA48 Runtime

       ↓

Host Mapping:
0x2000000000

       ↓

Read:
Hello VA48
```

这只是预期测试流程，不代表当前已经完成了真实的 Guest → Host 地址转换。

如果这一阶段能够通过测试，就可以继续研究执行环境集成。

**需要特别区分：软件映射测试通过，并不等于普通 ARM64 程序已经能够直接访问虚假的高地址。**

---

## 第二阶段：Android 集成

在核心逻辑得到验证后，继续开发 Android 侧组件。

计划完成：

- ARM64 构建。
- Android 运行环境集成。
- Host VA 检测。
- Runtime 打包。
- 配置管理。
- 日志输出。
- 运行时错误处理。

启动时可以尝试检测：

```text
Host VA Bits
Page Size
Memory Layout
Available Address Range
```

例如：

```text
Host VA_BITS = 39
PAGE_SIZE = 4096
```

然后根据实际检测结果选择适当的映射策略。

具体检测方法需要考虑 Android 权限限制、内核配置和实际进程地址空间布局。

---

## 第三阶段：PC Runtime 集成

在核心映射逻辑和 Android 侧组件经过验证后，再逐步集成 PC 兼容运行环境。

计划研究：

```text
PRoot
  ↓
Box64
  ↓
FEX
```

其中，FEX 是优先研究的集成目标之一。

原因是动态二进制翻译器能够参与 Guest 指令执行和内存访问处理，比单纯拦截系统调用更有机会实现 Guest 地址与 Host 地址之间的转换。

不过，实际开发顺序可以根据各项目的源码结构、修改难度和测试结果调整。

---

## 第四阶段：DroidDeck

当 Runtime 的核心功能和至少一个 Backend 得到验证后，再开发 DroidDeck 集成。

计划实现：

- Plugin Discovery。
- Runtime Selection。
- Per-Game Configuration。
- Enable / Disable。
- Runtime Profiles。
- Logging。
- Compatibility Detection。

预期使用流程：

```text
DroidDeck
     ↓
游戏设置
     ↓
VA48 Runtime
     ↓
开启或关闭
     ↓
启动游戏
```

用户不需要为了使用该 Runtime 而专门刷入修改过的 Kernel。

但项目能否在特定设备上运行，仍然取决于实际的 Android 环境、权限、运行时架构和实现结果。

---

## 第五阶段：兼容性测试

最终计划测试以下运行链：

```text
Android Device
      ↓
   DroidDeck
      ↓
  VA48 Runtime
      ↓
  FEX / Box64
      ↓
     Wine
      ↓
  Windows Game
```

重点测试：

- VA allocation。
- Large memory mapping。
- High virtual address。
- JIT。
- Thread。
- mmap。
- mprotect。
- mremap。
- Shared memory。
- Executable memory。
- DLL loading。
- Wine memory layout。

这些测试用于确认实际实现能够处理哪些内存操作，以及是否存在性能、稳定性或兼容性问题。

即使地址空间兼容功能通过测试，也不能保证所有游戏都能够正常运行。游戏还可能依赖图形驱动、CPU 指令集、系统调用、反作弊兼容性以及其他运行环境特性。

---

## 与 Kernel VA48 的区别

| 项目 | Kernel VA48 | VA48 Runtime |
|---|---|---|
| 修改 Kernel | 通常需要 | 设计目标是不需要 |
| 刷入专用 Kernel | 视设备而定 | 设计目标是不需要 |
| 真正硬件 VA48 | 支持时可以 | 不提供 |
| 用户态实现 | 不是主要方式 | 是 |
| 性能 | 通常更接近原生 | 预计存在额外开销 |
| 兼容设备 | 受内核和硬件限制 | 希望覆盖更多设备 |
| FEX 集成 | 可选 | 计划支持 |
| Box64 集成 | 可选 | 计划支持 |
| DroidDeck | 可以集成 | 第一目标平台 |
| QEMU | 与本项目目标无关 | 初期不纳入范围 |
| 临时 Root | 取决于修改方式 | 不以 Root 为核心前提 |

表格描述的是项目目标与两种设计方向的区别，不代表 VA48 Runtime 已经实现相应功能。

---

## 项目定位

VA48 Runtime 的定位是：

```text
Userspace Virtual Address Compatibility Layer
```

即：

**用户态虚拟地址空间兼容层。**

它不是：

- Kernel Module。
- 完整 CPU Emulator。
- QEMU。
- 能够自动修改所有 Android 程序地址空间的通用工具。
- 能够直接将 VA39 Kernel 转变为真正 VA48 Kernel 的软件。

项目希望解决的问题是：

```text
Host Kernel VA39
       ↓
如何让支持软件地址转换的运行环境
       ↓
管理逻辑上的 Guest VA48
```

本项目的重点是探索地址空间管理与运行时集成，而不是通过简单包装 mmap 就声称实现完整 VA48。

---

## 性能目标

项目不追求：

```text
VA48 Runtime = Native VA48
```

真正的硬件 VA48 可以由 CPU MMU 和内核页表直接完成地址转换，而用户态软件方案可能需要额外的映射管理、地址转换和执行环境配合。

本项目更关注：

```text
兼容性
   ↓
地址空间扩展
   ↓
减少对专用 Kernel 的依赖
```

同时计划通过以下方式降低潜在开销：

- Fast Path。
- Software TLB。
- Lazy Mapping。
- Large Page。
- Mapping Cache。

实际性能目标需要在核心功能实现后，通过基准测试和真实应用测试确定。

---

## 设计原则

### 1. 不修改 Kernel

项目的核心设计目标是不要求用户修改 Android Kernel 配置，例如：

```text
CONFIG_ARM64_VA_BITS_48
```

也不要求用户为了使用该项目而刷入专用 Kernel。

### 2. Runtime 独立

VA48 Core 不应该绑定 DroidDeck。

理想情况下，它可以通过 Backend 接口集成到：

```text
DroidDeck
FEX
Box64
PRoot
```

等相关运行环境。

### 3. DroidDeck 只是前端

DroidDeck 主要负责：

```text
选择
配置
启动
```

而不是实现地址转换核心。

### 4. 尽可能降低开销

普通地址尽量使用快速路径。

需要特殊处理的地址则由兼容层和 Backend 管理。

### 5. 可扩展

核心设计应尽可能与具体 Backend 分离，以便后续增加或调整运行环境支持。

### 6. 以实际实验为准

所有架构设想、性能目标和兼容性目标都需要通过实际构建、设备运行和测试来验证。

如果实验结果证明某项设计不可行，就应当修改实现，而不是将理论方案当作已经成立的技术事实。

---

## 当前项目目标

第一阶段不直接追求运行大型 Windows 游戏。

当前最优先的目标是：

```text
Android VA39 Kernel
         ↓
创建逻辑 Guest VA48 Address Space
         ↓
管理 0x8000000000 以上的 Guest Address
         ↓
实现 Guest → Host Mapping
         ↓
完成读写、保护和释放测试
         ↓
研究与执行环境的集成
```

在核心地址空间管理逻辑通过验证后，再继续推进：

```text
FEX / Box64
      ↓
   DroidDeck
      ↓
实际游戏测试
```

---

## 项目愿景

最终希望探索出以下运行架构：

```text
Android Phone
      │
      │ 不修改 Kernel
      │ 不刷专用 Kernel
      ▼
   DroidDeck
      │
      ▼
  VA48 Runtime
      │
      ├──────── FEX
      │
      ├──────── Box64
      │
      └──────── PRoot
      │
      ▼
Wine / Windows Compatibility
      │
      ▼
   PC Games
```

让受到 39-bit userspace VA 限制的 Android PC 游戏运行环境，有机会通过用户态软件机制获得逻辑上的扩展虚拟地址空间能力。

最终能支持哪些设备、运行环境和游戏，需要由后续实际构建和实验决定。

---

## 项目状态

当前状态：

```text
Experimental / Research
概念设计与技术可行性研究阶段
```

目前最优先的工作：

```text
1. VA48 Core
2. Guest → Host Address Mapping
3. 基础测试与验证
4. FEX Backend 研究
5. Box64 Backend 研究
6. DroidDeck 集成
```

项目目前尚未证明可以在真实 Android VA39 Kernel 上实现完整的 Guest VA48 兼容环境。

因此，当前文档主要用于说明项目目标、设计思路和开发计划。

最终功能、技术路线、性能与兼容性均以实际构建和实验结果为准。

---

## 项目标语

**English**

A pluggable userspace VA48 compatibility runtime for Android PC gaming environments, designed for DroidDeck, FEX, Box64 and related runtimes without requiring a custom kernel.

**中文**

面向 Android PC 游戏运行环境的可插拔用户态 VA48 兼容运行时，旨在无需专用内核的情况下，为 DroidDeck、FEX、Box64 等运行环境探索扩展虚拟地址空间的可能性。

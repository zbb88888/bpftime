# bpftime：用户空间 eBPF 运行时框架

## 项目定位

**bpftime 是一个用户空间 eBPF 运行时开发框架**，而非开箱即用的命令行工具。

它提供的是**基础设施层**，让开发者能够在用户空间编译、运行和部署 eBPF 程序，无需依赖 Linux 内核的 eBPF 子系统。

```
bpftime = 用户空间 eBPF 完整运行时
         ├── 虚拟机（VM）          ── 执行 eBPF 字节码
         ├── 运行时库（Runtime）    ── Maps、Helpers、Ufuncs
         ├── 事件挂载（Attach）     ── Uprobe、Syscall、GPU、XDP
         ├── 安全验证（Verifier）   ── 程序安全性检查
         └── 开发接口（API）        ── 构建自定义工具
```

## 与现有工具的区别

| 项目 | 类型 | 说明 |
|------|------|------|
| **bpftime** | 运行时框架 | 提供 eBPF 运行环境，开发者基于它构建工具 |
| bpftrace | 命令行工具 | 开箱即用的追踪工具，一行脚本即可使用 |
| BCC | 工具集 | 现成的性能分析和追踪工具集合 |
| perf | 性能分析器 | 直接使用的性能统计工具 |

**类比**：bpftime 相当于「Linux 内核的 eBPF 子系统」，而 bpftrace/BCC 相当于「基于该子系统构建的工具」。

## 核心价值

### 1. 突破内核限制

- 无需 root 权限
- 无需内核 eBPF 支持
- 可在旧版 Linux 甚至其他操作系统上运行

### 2. 性能优势

- Uprobe 性能比内核实现快 **10 倍**
- 用户空间内存读写更快
- 支持 LLVM JIT/AOT 优化编译

### 3. 完全兼容现有生态

- 直接使用 clang、libbpf 编译 eBPF 程序
- 支持 CO-RE（Compile Once, Run Everywhere）
- 无需修改现有 eBPF 代码

### 4. 扩展性强

- 模块化设计，易于添加新的事件源
- 支持 GPU（CUDA）事件挂载
- 支持 XDP/DPDK 网络场景

## 架构设计

```
┌─────────────────────────────────────────────────────────────┐
│                      用户应用程序                            │
├─────────────────────────────────────────────────────────────┤
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │  Uprobe  │  │ Syscall  │  │   GPU    │  │   XDP    │   │
│  │  事件源  │  │  追踪    │  │  事件    │  │  网络    │   │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘   │
│       │             │             │             │           │
│  ┌────┴─────────────┴─────────────┴─────────────┴────┐     │
│  │              Attach 层（事件挂载机制）              │     │
│  └────────────────────────┬──────────────────────────┘     │
│                           │                                 │
│  ┌────────────────────────┴──────────────────────────┐     │
│  │              Runtime 层（运行时核心）               │     │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐           │     │
│  │  │  Maps   │  │ Helpers │  │ Ufuncs  │           │     │
│  │  └─────────┘  └─────────┘  └─────────┘           │     │
│  └────────────────────────┬──────────────────────────┘     │
│                           │                                 │
│  ┌────────────────────────┴──────────────────────────┐     │
│  │              VM 层（虚拟机后端）                    │     │
│  │  ┌─────────────┐  ┌─────────────┐                 │     │
│  │  │  LLVM JIT   │  │    ubpf     │                 │     │
│  │  └─────────────┘  └─────────────┘                 │     │
│  └───────────────────────────────────────────────────┘     │
│                                                             │
│  ┌───────────────────────────────────────────────────┐     │
│  │           Verifier（安全验证）                     │     │
│  │        PREVAIL 用户空间验证 / 内核验证             │     │
│  └───────────────────────────────────────────────────┘     │
└─────────────────────────────────────────────────────────────┘
```

## 使用场景

### 场景一：构建用户空间追踪工具

**需求**：在没有 root 权限或内核 eBPF 支持的环境中追踪程序行为。

**方案**：使用 bpftime 作为运行时，开发自定义的 eBPF 追踪程序。

```c
// 示例：追踪 malloc 调用
SEC("uprobe/libc.so.6:malloc")
int trace_malloc(struct pt_regs *ctx) {
    u32 pid = bpf_get_current_pid_tgid() >> 32;
    // 记录调用信息
    increment_counter(&call_map, &pid, 1);
    return 0;
}
```

### 场景二：高性能动态插桩

**需求**：对生产环境进行低开销的函数追踪。

**方案**：利用 bpftime 的用户空间 Uprobe，比内核实现快 10 倍。

```bash
# 动态 attach 到运行中的进程
sudo bpftime attach <pid>

# 加载 eBPF 程序
bpftime load ./my_tracer
```

### 场景三：跨平台 eBPF 部署

**需求**：在不支持内核 eBPF 的系统上运行 eBPF 程序。

**方案**：bpftime 提供完整的用户空间 eBPF 运行时，无需内核支持。

- 旧版 Linux 内核（< 4.x）
- 容器环境（受限的内核能力）
- 非 Linux 系统（实验性支持）

### 场景四：GPU 内核插桩

**需求**：追踪和分析 CUDA GPU 程序的执行。

**方案**：bpftime 的 `nv_attach_impl` 可将 eBPF 转换为 PTX 注入 GPU 内核。

```
eBPF 程序 → PTX 代码 → GPU 内核注入
```

性能比 NVbit 快 **10 倍**。

### 场景五：用户空间网络处理

**需求**：高性能网络数据包处理。

**方案**：集成 AF_XDP 或 DPDK，在用户空间运行 XDP 程序。

## 组件说明

| 组件 | 路径 | 功能 |
|------|------|------|
| **vm** | `vm/` | eBPF 虚拟机，支持 LLVM JIT 和 ubpf |
| **runtime** | `runtime/` | 核心运行时，Maps、Helpers、Handler 管理 |
| **attach** | `attach/` | 事件挂载机制，Uprobe、Syscall、GPU |
| **verifier** | `bpftime-verifier/` | 程序安全验证 |
| **daemon** | `daemon/` | 内核 eBPF 监控和协作 |
| **tools** | `tools/` | CLI 工具和 AOT 编译器 |
| **example** | `example/` | 示例程序和使用演示 |

## 快速开始

### 构建项目

```bash
# 基础构建（Debug 模式）
make build

# Release 构建
make release

# 启用 LLVM JIT
make release-with-llvm-jit

# 启用 GPU 支持
make build-gpu
```

### 运行示例

```bash
# 构建示例
make -C example/malloc

# 加载 eBPF 程序
bpftime load ./example/malloc/malloc

# 在另一个终端运行目标程序
bpftime start ./example/malloc/victim
```

### 动态 Attach

```bash
# 启动目标程序
./example/malloc/victim &

# 动态注入 eBPF 运行时
sudo bpftime attach $!
```

## 开发指南

### 基于 bpftime 开发工具

1. **编写 eBPF 程序**：使用标准的 clang/libbpf 工具链
2. **选择事件源**：Uprobe、Syscall、GPU 等
3. **实现数据处理**：通过 Maps 进行数据聚合和传递
4. **构建和测试**：使用 bpftime 运行时运行和调试

### 核心 API

```cpp
// 创建 eBPF 程序
struct ebpf_vm *vm = ebpf_create("");

// 注册 Helper 函数
ebpf_register(vm, helper_id, "helper_name", helper_func);

// 加载和执行
ebpf_load(vm, code, code_len);
uint64_t result = ebpf_exec(vm, mem, mem_len);
```

## 技术论文

bpftime 的设计理念和实现细节在 OSDI '25 论文中有详细描述：

> **Extending Applications Safely and Efficiently**
> Yusheng Zheng, Tong Yu, Yiwei Yang, Yanpeng Hu, Xiaozheng Lai, Dan Williams, Andi Quinn
> *19th USENIX Symposium on Operating Systems Design and Implementation (OSDI '25)*

## 总结

bpftime 是一个**面向开发者的 eBPF 用户空间运行时框架**，适用于：

- ✅ 构建自定义的 eBPF 工具和应用
- ✅ 在受限环境中部署 eBPF 程序
- ✅ 追求更高性能的动态插桩
- ✅ 探索 eBPF 在 GPU、网络等新领域的应用

它**不是**：
- ❌ 开箱即用的命令行工具
- ❌ 现成的性能分析工具集
- ❌ bpftrace/BCC 的替代品（而是它们的运行时基础）

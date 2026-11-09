# 依赖评估与决策

> 日期：2026-11-09  
> 范围：Rust 与 C++ 依赖  
> 原则：持续维护、近期发布、Windows/Linux 双平台、与 C++23/26 和 Rust 1.99.1 兼容  
> 完整架构：[`ARCHITECTURE_REPORT.md`](ARCHITECTURE_REPORT.md)

---

## 1. 评估标准

一个库进入项目，需要同时满足：

1. 最近 12 个月有发布或实质维护；
2. Windows x64 与 Linux x86_64 都能构建；
3. 与 C++23/C++26 或 Rust 1.99.1 兼容；
4. 能通过 xmake/Conan 或 Cargo 稳定获取；
5. 职责不与其他库重复；
6. 不引入无法控制的构建依赖（Python、Boost 全家桶等）。

“活跃”不等于“应该用”。测试框架、调度器、格式化库尤其容易重复。

---

## 2. Rust 依赖

### 2.1 指定库

| crate | 最新版本 | 更新时间 | 决策 | 用途 |
|---|---|---|---|---|
| `tokio` | 1.54.1 | 2026-10-21 | 采用 | FFmpeg 进程监督、超时、取消、异步 IO |
| `crossbeam` | 0.8.5 | 2026-09-05 | 采用 | 有界 channel、scoped thread、CPU 数据面 |
| `parking_lot` | 0.13.0 | 2026-10-21 | 采用 | 共享状态锁；避免 poisoning |
| `dashmap` | 6.2.1 | 2026-05-17 | 采用 6.2.1 | 能力缓存、stream registry |
| `cargo-nextest` | 0.9.151 | 2026-11-03 | 采用（CI） | 测试运行器 |
| `loom` | 0.7.3 | 2026-11-08 | 采用（dev） | 自定义同步原语/状态机模型检查 |
| `insta` | 1.49.0 | 2026-10-03 | 采用（dev） | CLI JSON、manifest、plan dump 快照 |
| `rstest` | 0.28.0 | 2026-10-18 | 采用（dev） | fixture、参数化测试 |

#### dashmap

- `7.0.0-rc2` 从 2025 年起一直 RC，不适合生产；
- `6.2.1` 是稳定版，满足能力缓存/registry；
- 不把 `dashmap` 放进每秒百万次的热路径；热路径用 per-thread 结构或无锁队列。

#### loom

- 只用于自定义同步结构、状态机、队列的模型检查；
- 不模拟 tokio runtime；
- 作为 dev-dependency，不进入发布物。

#### nextest

- 是测试运行器，不是运行时依赖；
- Windows/Linux CI 都可用；
- 与 `rstest`、`insta` 兼容。

### 2.2 补充库

| crate | 最新版本 | 决策 | 用途 |
|---|---|---|---|
| `clap` | 4.6.7 | 采用 | CLI、completions、man page |
| `serde` | 1.0.229 | 采用 | TOML/JSON 序列化 |
| `toml` | 1.2.1+spec-1.1.0 | 采用 | TOML 1.1 解析 |
| `thiserror` | 2.0.21 | 采用 | 错误类型 |
| `tracing` | 0.1.44 | 采用 | 结构化日志 |
| `tracing-subscriber` | 0.3.23 | 采用 | 日志输出 |
| `rayon` | 1.12.0 | 采用 | CPU 扫描/批量处理 |
| `cxx` | 1.0.205 | 采用 | Rust/C++ 桥 |
| `proptest` | 1.12.0 | 采用（dev） | 属性测试 |
| `assert_cmd` | 2.2.2 | 采用（dev） | CLI 集成测试 |
| `criterion` | 0.8.2 | 采用（dev） | Rust 基准 |
| `tempfile` | 3.27.0 | 采用 | 临时文件 |
| `rustix` | 1.1.6 | 采用 | Linux 文件/进程原语 |
| `windows-sys` | 0.61.2 | 采用 | Windows 文件/进程/Job Object |

不采用：

- `miette`：2025-04 后更新较慢；诊断用 `thiserror` + 自有错误码足够；
- nightly-only SIMD 库：AVX2 内核放在 C++；
- 第二套 CLI 解析器：`clap` 已覆盖；
- 第二套日志框架：`tracing` 已覆盖。

### 2.3 TOML 1.1

`toml = "=1.2.1"`，对应 `1.2.1+spec-1.1.0`，2026-11-06 发布。

- 只在 Rust 侧解析；
- C++ 不引入 TOML parser；
- `schema` 必填；
- `deny_unknown_fields`；
- 旧 LAE/v7 配置键直接拒绝；
- 不提供迁移层。

注意：TOML 1.1 基本是 1.0 的超集，语法层无法拒绝 1.0 写法；“不向下兼容”落实为 schema/字段级拒绝。

---

## 3. C++ 依赖

### 3.1 结论表

| 库 | 最新版本 | 更新时间 | 决策 | 用途 |
|---|---|---|---|---|
| `fmt` | 12.2.1 | 2026-11-07 | 采用 | 统一格式化/诊断 |
| `doctest` | 2.5.3 | 2026-07-06 | 采用 | 单元测试 |
| `oneTBB` | 2023.2.0 | 2026-10-28 | 采用 | AVX2 内核并行 |
| `GoogleTest` | 1.18.0 | 2026-08-10 | 不采用 | 与 doctest 重复 |
| `Catch2` | 3.16.1 | 2026-10-08 | 不采用 | 与 doctest 重复 |
| `taskflow` | 4.1.0 | 2026-06-20 | 不采用 | 与 oneTBB 重复 |
| `folly` | 2026.10.26.00 | 2026-10-26 | 不采用 | 过重/Windows 测试禁用 |
| `moodycamel::ConcurrentQueue` | 1.0.5 | 2026-04-03 | 条件采用 | C++ 自有 MPMC 队列 |
| `google/benchmark` | 1.10.0 | 2026-10-14 | 采用（dev） | C++ 内核微基准 |

### 3.2 fmt

采用理由：

- 2026-11-07 发布 12.2.1，维护活跃；
- 格式串编译期检查；
- Windows/Linux 行为一致；
- 与 oneTBB、doctest 无冲突；
- 可作为 C++ 侧唯一格式化层。

注意：

- C++23 已有 `std::format`/`std::print`；
- 如果团队坚持标准库优先，可以去掉 fmt，改用 `std::format`；
- 一旦选择 fmt，C++ 侧不要混用两套格式化。

### 3.3 doctest

采用理由：

- 2026-07-06 发布 2.5.3；
- header-only，编译快；
- 足够覆盖 AVX2 内核、buffer、错误路径；
- 不需要 mock；
- C++ 集成测试由 Rust `assert_cmd` 驱动 CLI。

不选 GoogleTest：

- 1.18.0 很成熟，但编译重、依赖多；
- C++ 侧是内核库，不是大型服务；
- GTest 的 death test/mock 优势用不上。

不选 Catch2：

- 3.16.1 功能强，但与 doctest 重叠；
- 多一个编译库和测试 DSL；
- 如果未来需要 generators/benchmarks，再单独评估 Catch2。

### 3.4 oneTBB

采用理由：

- 2023.2.0，2026-10-28；
- Intel 维护，Windows/Linux 支持成熟；
- `parallel_for`、`task_group`、`concurrent_queue` 足够；
- 适合 AVX2 内核的批量处理。

注意：

- 用 `tbb::global_control` 限制并行度；
- 每个 job 用独立 `task_arena`；
- 不要和 tokio/crossbeam/FFmpeg 线程叠加到超订。

不选 taskflow：

- 4.1.0，header-only，DAG 表达好；
- 但 Rust 已负责文件级/阶段级 DAG；
- C++ 再引入 taskflow 会出现两套调度器；
- oneTBB 的并行算法足够覆盖内核级并行。

### 3.5 folly

不采用，理由：

- 2026.10.26 发布，维护活跃；
- 体量巨大，依赖多：Boost、glog、gflags、double-conversion、libevent、OpenSSL 等；
- README 明确 Windows 上大量测试被禁用；
- 主要支持 C++20，C++23/26 支持不确定；
- 构建脚本依赖 Python；
- 与“只 Rust/C++、xmake + Conan、精简依赖”的目标冲突；
- 所需能力可用 oneTBB、crossbeam、moodycamel、std 替代。

重新评估条件：

- 确实需要 `folly::IOBuf` 做零拷贝网络；
- 或需要 folly 的某个无法替代的组件；
- 且 Windows 构建/测试通过。

### 3.6 moodycamel::ConcurrentQueue

- 1.0.5，2026-04-03；
- header-only，跨平台；
- 如果 C++ 数据面自己拥有 packet/frame 队列，建议采用；
- 如果队列仍在 Rust/crossbeam，则不需要。

结论：**条件采用**，不默认引入。

### 3.7 Google Benchmark

- 1.10.0，2026-10-14；
- 只用于 C++ AVX2 内核微基准；
- 不进入产品运行时；
- Rust 侧用 criterion 测调度/IO。

---

## 4. 最终依赖集

### Rust 运行时

```text
tokio
crossbeam
parking_lot
dashmap 6.2.1
clap
serde
toml 1.2.1
thiserror
tracing
tracing-subscriber
rayon
cxx
tempfile
rustix / windows-sys
```

### Rust 开发/测试

```text
cargo-nextest
loom
insta
rstest
proptest
assert_cmd
criterion
```

### C++ 运行时

```text
oneTBB
fmt
```

### C++ 开发/测试

```text
doctest
google/benchmark
```

### 条件依赖

```text
moodycamel::ConcurrentQueue
```

---

## 5. 不采用的库与原因

| 库 | 原因 |
|---|---|
| `folly` | 过重、Windows 测试禁用、C++20、Python 构建、依赖爆炸 |
| `taskflow` | 与 oneTBB 重复；Rust 已负责 DAG |
| `GoogleTest` | 与 doctest 重复；编译重 |
| `Catch2` | 与 doctest 重复；更重 |
| `toml++` | Rust 单点解析 TOML，C++ 不需要 |
| `spdlog` | fmt + 简单日志足够 |
| `miette` | 更新较慢；thiserror 足够 |
| `range-v3` | C++23 `std::ranges` 足够 |
| `expected-lite` | C++23 `std::expected` 足够 |
| `nlohmann/json` | manifest 由 Rust 输出；C++ 不解析 JSON |

---

## 6. 维护策略

- 所有依赖固定到精确版本，提交 `Cargo.lock` 和 Conan lock；
- 每月检查一次上游安全公告和 release；
- 每季度评估一次“是否还需要这个依赖”；
- C++ 依赖通过 Conan 锁定版本与构建选项；
- Rust 依赖通过 `cargo-deny` / `cargo-audit` 检查；
- 不在热路径引入动态分配/锁，除非基准证明必要。

---

## 7. 相关文档

- 结论摘要：[`EXECUTIVE_SUMMARY.md`](EXECUTIVE_SUMMARY.md)
- 完整架构：[`ARCHITECTURE_REPORT.md`](ARCHITECTURE_REPORT.md)
- 文档索引：[`README.md`](README.md)

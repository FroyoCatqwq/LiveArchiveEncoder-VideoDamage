# LiveArchiveEncoder-VideoDamage 技术评估与架构报告 v2

> 日期：2026-11-09  
> 参考基线：`LAE-Damager` / `frame_damage v3` 0.2.0  
> 目标：LiveArchiveEncoder v8 的视频损伤扩展包  
> 约束：仅 Rust + C++ 实现；Windows + Linux；仅 CLI；可独立运行，也可作为 v8 插件；使用 FFmpeg 三件套；TOML 1.1；x86-64-v3

---

## 1. 结论

这一版约束和上一版有一个根本变化：

> **FFmpeg / ffprobe / ffplay 三件套重新成为核心执行层，C++ 不再承担 libav 解封装/封装，而是承担高性能数据面。**

这会显著降低媒体格式支持的实现成本，也让“更多视频编码”成为可交付目标；但它同时把四个问题重新推到台前：

1. **进程管理**：FFmpeg CLI 的 stderr/stdout、超时、取消、进程树、临时文件必须由 Rust 统一管住。
2. **确定性边界**：同 seed 同输入不等于跨 FFmpeg 构建同输出；`packet-loss` 这类依赖 FFmpeg 内部随机数的操作仍然是 D3。
3. **全流保留**：旧项目只映射第一条视频/音频；新项目必须显式 `-map 0`、保留字幕/多音轨/章节/附件。
4. **C++ 的职责**：如果只是“用 C++ 再包一层 FFmpeg 命令”，C++ 没有存在价值。C++ 必须承担 AVX2/SSE4 数据面内核、并行执行和 buffer 管理。

**推荐架构：**

```text
Rust 控制面
  CLI / TOML 1.1 / Recipe / Plan / Scheduler
  FFmpeg 三件套发现、版本校验、进程监督
  Manifest / 输出事务 / 插件 C ABI
        │ cxx / C ABI
        ▼
C++ 数据面
  AVX2/SSE4 扫描与改写内核
  oneTBB 并行执行
  buffer / 队列 / 批处理
        │
        ▼
FFmpeg / ffprobe / ffplay（LAE 自构建三件套）
```

一句话：

> **Rust 管“做什么、什么时候做、失败怎么办”；C++ 管“怎么在内存里最快地做”；FFmpeg 管“格式与编解码”。**

---

## 2. 相比上一版的变化

| 项 | 上一版建议 | 本版 |
|---|---|---|
| 媒体层 | C++ libavformat/libavcodec | **FFmpeg CLI 三件套** |
| C++ 职责 | 解封装/封装/编解码 | **AVX2/SSE4 内核 + oneTBB 并行 + buffer** |
| 确定性 | 追求 packet 级 D1 | **按 FFmpeg 能力分层：D1/D2/D3 如实标注** |
| 编码覆盖 | 以 libav 能力为上限 | **以 FFmpeg 三件套能力为上限，分 tier 实现** |
| TOML | 未定 | **TOML 1.1 only，Rust 单点解析** |
| CPU | 可选 v3 | **强制 x86-64-v3，无降级** |
| 依赖 | 尽量少 | **按约束评估，拒绝冗余库** |

旧项目仍然只作为参考：

- 保留 `scan → plan → execute → verify`；
- 保留 PCG32、每帧独立随机流、整数概率；
- 保留错误码、夹具、容器矩阵、Supervisor 模式；
- **不复制**只映射第一条流的逻辑、同步阻塞 apply、过度确定性标签。

---

## 3. 四条约束的落地

### 3.1 只使用 Rust 和 C++

产品代码只允许 Rust 和 C++。

构建描述（`xmake.lua`、`conanfile.txt`）是工具 DSL，不是产品语言；如果要求连构建 DSL 也不能出现第三种文本，则改用 Cargo-first：

- `build.rs` 调用 `cc` / `cxx-build` 编译 C++；
- FFmpeg 通过 `pkg-config` / `vcpkg` / 预生成 Conan 包提供；
- 不引入 xmake/CMake。

本报告推荐 **xmake + Conan + Cargo + Rust `xtask`**，因为：

- 你已在 Windows/Linux 安装 xmake + Conan；
- xmake 对 MSVC/Clang、C++26、静态库/动态库、AVX2 开关更直接；
- Rust `xtask` 作为唯一入口，避免 shell 脚本；
- Conan 负责 FFmpeg 三件套之外的 C++ 依赖（fmt、oneTBB、doctest）。

### 3.2 Rust 1.99.1

- 固定 `rust-toolchain.toml`：`channel = "1.99.1"`；
- edition 2024，`rust-version = "1.99"`；
- 只用 stable；
- 参考：https://blog.rust-lang.org/2026/10/15/Rust-1.99.1/

建议使用的稳定能力：

- edition 2024 的 `let` chains、`impl Trait` in traits、`#[expect]`、C 字符串字面量；
- `std::sync::OnceLock` / `LazyLock`；
- `std::thread::scope`；
- `std::io::ErrorKind` 的细化匹配；
- 不引入 nightly 的 `std::simd`、`gen` blocks 等。

### 3.3 C++26，最低 C++23

- 目标：C++26。
- 下限：C++23。
- Windows：MSVC 2026，`/std:c++latest`（若 `/std:c++26` 可用则优先）。
- Linux：Clang 23.1.6，`-std=c++26`。
- CI 同时构建 C++23 和 C++26。

C++26 特性不能假设全部可用。所有 C++26 特性必须用 feature-test macro 守卫。

建议：

- C++23 基线：`std::expected`、`std::print`、`std::flat_map`、`std::generator`、`std::stacktrace`、`std::mdspan`、`std::byteswap`、`std::to_underlying`。
- C++20 基线：`std::span`、`std::ranges`、`std::format`、`std::jthread`、`std::stop_token`、`std::bit_cast`、`std::source_location`。
- C++26 可选：`std::execution`、`std::simd`、`std::inplace_vector`、`std::hive`、contracts。
- 核心路径不依赖 reflection、pattern matching 等尚未普遍实现的特性。
- C++ 异常不跨越 FFI；插件 C ABI 必须 `noexcept` 并捕获所有异常。

### 3.4 Windows + Linux

主目标：Windows x64、Linux x86_64。Linux aarch64 暂不作为一等目标。

| 问题 | Windows | Linux |
|---|---|---|
| 路径 | UTF-16 / `std::filesystem::path` | UTF-8 bytes |
| 原子替换 | `ReplaceFileW` | `renameat2` |
| 目录同步 | `FlushFileBuffers`（权限受限时用 journal 补足） | `fsync(dirfd)` |
| 进程树 | Job Object | `setsid` + `killpg` |
| 动态库 | `.dll` | `.so` |
| 编译器 | MSVC 2026 | Clang 23 |
| CPU 基线 | `/arch:AVX2` | `-march=x86-64-v3` |
| FFmpeg | MSVC 构建的 trio | Clang 兼容构建的 trio |

### 3.5 仅 CLI

不做 CUI、GUI、TUI。CLI 是唯一产品界面。

- `clap` 子命令；
- `--json` / `--jsonl`；
- 人类可读信息走 stderr，机器输出走 stdout；
- 稳定退出码：0 成功、1 运行期错误、2 用法/配置错误、130 中断；
- `--help`、`--version`、`--explain`、shell completions、man page；
- 纯文本进度；
- `ffplay` 只作为显式 `--play` 的外部播放器，不是本项目的 GUI。

### 3.6 独立 + v8 插件

同一核心，三种形态：

1. `lae-videodamage` CLI；
2. Rust crate；
3. C ABI `lae_videodamage.dll` / `.so` 插件。

不允许 CLI 与插件各写一套逻辑。

---

## 4. 总体架构

### 4.1 控制面 / 数据面

```text
┌────────────────────────────────────────────────────────────┐
│                        LAE v8 Host                         │
│  Job / Artifact / Config / Logging / Cancellation           │
└───────────────┬──────────────────────────────┬─────────────┘
                │ Rust crate                   │ C ABI
                ▼                              ▼
┌────────────────────────────┐   ┌───────────────────────────┐
│ lae-videodamage-core (Rust)│   │ lae-videodamage-plugin    │
│ TOML 1.1 / Recipe / Plan   │   │ cdylib + C header         │
│ Scheduler / Supervisor     │   │ 同一核心的薄适配          │
│ Manifest / Transaction     │   └───────────────────────────┘
└───────────────┬────────────┘
                │ cxx
                ▼
┌────────────────────────────────────────────────────────────┐
│                  lae-videodamage-kernels (C++)             │
│  AVX2/SSE4 扫描、bitflip、destructive、hash、buffer        │
│  oneTBB 并行执行、batch 调度                               │
└───────────────┬────────────────────────────────────────────┘
                │
                ▼
┌────────────────────────────────────────────────────────────┐
│           FFmpeg / ffprobe / ffplay（LAE 自构建三件套）      │
│  demux / mux / encode / decode / filters / probe / play     │
└────────────────────────────────────────────────────────────┘
```

### 4.2 为什么 C++ 不负责启动 FFmpeg

- Rust 的 `tokio::process` 在 Windows/Linux 上更成熟；
- Rust 统一处理 stdout/stderr drain、超时、进程树、取消；
- C++ 只接收 buffer/range，返回处理结果，避免 C++ 再写一套跨平台进程管理；
- 避免 Rust 和 C++ 两套调度器互相抢线程。

C++ 的职责是“数据面”：

- AVX2/SSE4 字节扫描与改写；
- 批量 buffer 处理；
- oneTBB 并行；
- 可选：frame/packet 级预处理。

### 4.3 FFmpeg 进程图

每个操作由 Rust 编排为进程图：

```text
ffprobe
  │ 流信息 / codec / pix_fmt / disposition
  ▼
ffmpeg 解封装/抽流
  │
  ▼
C++ AVX2 内核（按 plan 改写）
  │
  ▼
ffmpeg 重封装/重编码
  │
  ▼
ffprobe / ffmpeg -f null 验证
  │
  ▼
ffplay（可选 --play）
```

进程间通过临时文件或有界管道连接；每个子进程都必须：

- `-nostdin`、`-hide_banner`、`-v error`；
- stdout/stderr 都有消费者；
- 超时 kill；
- 输出到同目录临时文件；
- 失败清理。

---

## 5. FFmpeg 三件套

### 5.0 自构建策略

三件套不是直接取上游 floating `latest`，而是由 LAE / VideoDamage 共同维护的**自构建产物**：

- fork `FFmpeg-Builds/FFmpeg-Builds`，固定 FFmpeg tag/commit 与全部依赖 commit；
- 基于 `nonfree + gpl` 变体，覆盖尽可能全的编解码器、硬件加速、字幕与滤镜；
- Linux 使用最新 Clang 23 + `x86-64-v3` + ThinLTO/PGO 评估；
- Windows 先保留 MinGW-w64 基线，再评估 `clang-cl` + VS2026 SDK；VS2026 用于本项目本体；
- 强制 `x86-64-v3`，不向下兼容；
- nonfree 产物仅私有共享，不随 GPL 项目公开发行；
- 每次构建生成 build ID、`buildconf.txt`、`configure.txt`、`sources.lock`、`capabilities.toml`、`licenses.toml`、`SHA256SUMS`。

完整评估与路线图见 [`FFMPEG_BUILD_REPORT.md`](FFMPEG_BUILD_REPORT.md)。

### 5.1 共享 LAE 自构建三件套

解析顺序：

1. `--ffmpeg-dir`；
2. `LAE_FFMPEG_DIR`；
3. LAE v8 配置中的 FFmpeg 路径；
4. 出错，不静默回退到 PATH。

三件套必须来自同一目录/同一构建：

```text
ffmpeg
ffprobe
ffplay
```

启动时校验：

- 三个文件都存在且可执行；
- `ffmpeg -version` / `ffprobe -version` / `ffplay -version` 的版本核心一致；
- 记录完整 banner 到 manifest；
- 若版本不一致，直接报错，不尝试混用。

### 5.2 只支持自构建最新构建

- 不维护旧版本兼容矩阵；
- 不做“低于某版本则降级”；
- 以 LAE v8 / VideoDamage 共享的自构建 `build ID` 为唯一支持版本；
- 启动时校验三件套 banner、build ID、`ffmpeg -buildconf` 关键开关与 `SHA256SUMS`；不同构建直接 `LAE-VD-E1004`；
- 能力探测结果按 build ID + 路径 hash 缓存；
- 旧版本或非预期构建报错信息必须告诉用户“请切换到 LAE 自构建的最新 FFmpeg 三件套”。

### 5.3 能力探测

启动/首次运行探测：

```text
ffmpeg -version
ffmpeg -decoders
ffmpeg -encoders
ffmpeg -filters
ffmpeg -bsfs
ffmpeg -muxers
ffmpeg -demuxers
ffprobe -version
ffplay -version
```

缓存到：

```text
%LOCALAPPDATA%/LAE/VideoDamage/caps/<trio-hash>.toml
```

注意：能力缓存也是 TOML，统一 TOML 1.1。

### 5.4 ffplay 的定位

- 显式 `--play` 才启动；
- 使用 `-autoexit -loglevel error`；
- 视频会打开外部窗口，这不是本项目的 GUI；
- CI/无显示环境禁用 `--play`；
- 播放结果只作为人工验证，不作为自动验收的唯一依据。

---

## 6. 编码与容器覆盖

### 6.1 按层级定义能力

| Tier | 能力 | 依赖 | 确定性 |
|---|---|---|---|
| A | 码流级改写（bitflip/destructive） | 自己的 parser | D1 |
| B | packet 级丢包/重排 | FFmpeg packet + BSF | D3（FFmpeg 内部随机数） |
| C | 解码帧操作（drop/freeze/repeat） | FFmpeg decode/encode | 决策确定，字节依赖编码器 |
| D | 像素/滤镜操作 | FFmpeg filters | 滤镜图确定，字节依赖编码器 |
| E | 容器/文件级（truncate/segment cut） | 字节/容器 | D1（纯字节） |

### 6.2 码流级（Tier A）

第一批：

- H.264
- HEVC
- MPEG-4 Part 2
- MPEG-1
- MPEG-2
- H.263 / H.263+

第二批：

- AV1（OBU 解析）
- VP9（superframe 解析）

不建议承诺：

- 所有 FFmpeg 能解码的编码都做码流级；
- ProRes / DNxHD / FFV1 等先走 frame/pixel 级，除非有明确需求。

### 6.3 packet 级（Tier B）

- 任何 FFmpeg 能 `-c copy` 的编码；
- 丢包/重排/复制；
- 通过 FFmpeg BSF 或临时文件；
- 必须标注 D3，因为随机数/包边界在 FFmpeg 内部；
- 不要把它宣传成 D1。

### 6.4 frame/pixel 级（Tier C/D）

- 所有 FFmpeg 能解码/编码的编码；
- 包括 HEVC、VP8/VP9、AV1、ProRes、DNxHD、FFV1、MJPEG、Theora、WMV、RV 等；
- 决策确定，编码字节依赖编码器和版本；
- 保留 `pix_fmt`、color primaries/transfer/matrix/range；10-bit/HDR 默认保留；
- 强制 `yuv420p` 只能作为显式选项。

### 6.5 容器

通过 FFmpeg 支持：

- MP4 / MOV / MKV / WebM / AVI / MPEG-TS / FLV / ASF / Ogg / RM 等；
- 容器兼容性由 FFmpeg 能力 + 实测矩阵决定；
- 不做“mux 成功但内容错”的静默输出；
- 未知组合必须明确警告或拒绝。

### 6.6 全流保留

默认：

```text
-map 0
-c copy
```

只对目标流做处理，其他流原样复制。

保留：

- 多音轨；
- 字幕；
- 数据流；
- 附件；
- 章节；
- disposition；
- language/title；
- metadata。

只有显式 `--drop-stream` 才丢弃。旧项目只保留第一条视频/音频的行为不允许继承。

---

## 7. TOML 1.1

### 7.1 解析器

Rust 使用：

```toml
toml = "=1.2.1"
```

`toml 1.2.1+spec-1.1.0` 于 2026-11-06 发布，明确对应 TOML 1.1.0 规范。

- 单点解析：只在 Rust 侧解析 TOML；
- C++ 不引入 TOML 解析器；
- 解析结果通过 `cxx`/C ABI 以结构化配置传给 C++；
- 避免 C++ 侧 `toml++` 对 1.1 支持不确定的问题。

### 7.2 Schema

```toml
schema = "lae.videodamage/v1"

[ffmpeg]
trio_dir = "C:/Tools/LAE/ffmpeg"

[run]
jobs = 4
max_memory_mb = 512
verify = "decode"

[damage]
scenario = "dvb-2008"
seed = 42
playability = "glitchy"

[streams]
video = "v:0"
keep = ["a:*", "s:*", "d:*"]

[stages.packet]
drop_percent = 5.0
protect_keyframes = true
```

规则：

- `schema` 必填；
- `deny_unknown_fields`；
- 旧 LAE/v7 配置键不识别，直接报错；
- 不提供迁移层；
- 默认值集中在一处，不在 CLI/C++/插件里重复；
- 配置错误退出码 2。

### 7.3 “不向下兼容”的准确含义

TOML 1.1 基本是 1.0 的超集，**无法从语法层拒绝 1.0 写法**。因此“不向下兼容”落实为：

- 只支持 TOML 1.1 解析器；
- 只接受 `lae.videodamage/v1` schema；
- 旧字段名、旧 section、旧默认值全部拒绝；
- 不读取 LAE v7 的 TOML；
- 不提供自动迁移。

如果要求在语法层也拒绝 1.0，只能自己写严格 parser，不建议。

---

## 8. x86-64-v3

### 8.1 强制基线

- Rust：`.cargo/config.toml` 设置 `target-cpu=x86-64-v3`；
- C++ MSVC：`/arch:AVX2`；
- C++ Clang/GCC：`-march=x86-64-v3`；
- 不使用 AVX-512；
- 不提供 scalar fallback；
- 启动时检查 CPUID：AVX2、BMI2、FMA、MOVBE、SSE4.2；
- 不满足时直接报错，不静默降级。

### 8.2 为什么

- 字节扫描、bitflip、hash、filter 内核都吃 AVX2；
- v3 覆盖 Haswell 起（2013+），对归档工作站足够；
- 不维护两套内核，减少测试矩阵。

### 8.3 风险

- CI runner 必须支持 AVX2；
- 老机器直接无法运行，必须在 README/`--version` 明确；
- 插件宿主也必须满足；
- 跨平台打包要注明。

---

## 9. Rust / C++ 分工

### 9.1 Rust 控制面

- CLI（clap）；
- TOML 1.1 解析与校验；
- Recipe / Plan / Manifest；
- FFmpeg 三件套发现、版本校验、能力探测；
- 进程监督（tokio::process）；
- 调度（文件级 + segment 级）；
- 输出事务、journal、quarantine；
- 报告、日志、错误码；
- 插件 C ABI；
- C++ 内核 FFI。

### 9.2 C++ 数据面

- AVX2/SSE4 扫描；
- bitflip / destructive 内核；
- 批量 buffer 处理；
- oneTBB 并行；
- 可选：packet/frame 预处理；
- 不解析 TOML；
- 不启动 FFmpeg；
- 不管理输出事务；
- 不持有全局状态。

### 9.3 边界

- `cxx` 桥接：Rust 持有 plan/range，C++ 接收 `rust::Slice<const uint8_t>` / `std::span`，返回处理结果；
- 插件 C ABI：只用 C 兼容类型、opaque handle、UTF-8、`noexcept`；
- 不跨边界传 Rust `String`/`Vec`/`Result`；
- 不跨边界抛 C++ 异常。

---

## 10. Rust 依赖评估

### 10.1 指定库

| crate | 最新版本 | 结论 | 用途 |
|---|---|---|---|
| `tokio` | 1.54.1（2026-10-21） | **采用** | 进程监督、超时、取消、异步 IO |
| `crossbeam` | 0.8.5（2026-09-05） | **采用** | 有界 channel、scoped thread、CPU 数据面 |
| `parking_lot` | 0.13.0（2026-10-21） | **采用** | 共享状态锁；避免 poisoning |
| `dashmap` | 6.2.1（2026-05-17） | **采用 6.2.1，不用 7.0 RC** | 能力缓存、stream registry |
| `cargo-nextest` | 0.9.151（2026-11-03） | **采用（CI）** | 测试运行器 |
| `loom` | 0.7.3（2026-11-08） | **采用（dev）** | 自定义同步原语/状态机模型检查 |
| `insta` | 1.49.0（2026-10-03） | **采用（dev）** | CLI JSON、manifest、plan dump 快照 |
| `rstest` | 0.28.0（2026-10-18） | **采用（dev）** | fixture、参数化测试 |

说明：

- `dashmap` 7.0.0-rc2 从 2025 年起一直 RC，不建议生产使用；6.2.1 是稳定版。
- `loom` 只用于自定义并发结构，不模拟 tokio runtime。
- `nextest` 是测试运行器，不是运行时依赖。
- `tokio` 只用于进程/异步控制面；CPU 密集工作走 `spawn_blocking`、crossbeam、rayon。

### 10.2 补充库

| crate | 最新版本 | 结论 | 用途 |
|---|---|---|---|
| `clap` | 4.6.7 | 采用 | CLI、completions、man |
| `serde` | 1.0.229 | 采用 | TOML/JSON 序列化 |
| `toml` | 1.2.1+spec-1.1.0 | 采用 | TOML 1.1 |
| `thiserror` | 2.0.21 | 采用 | 错误类型 |
| `tracing` / `tracing-subscriber` | 0.1.44 / 0.3.23 | 采用 | 结构化日志 |
| `rayon` | 1.12.0 | 采用 | CPU 扫描/批量处理 |
| `cxx` | 1.0.205 | 采用 | Rust/C++ 桥 |
| `proptest` | 1.12.0 | 采用（dev） | 属性测试 |
| `assert_cmd` | 2.2.2 | 采用（dev） | CLI 集成测试 |
| `criterion` | 0.8.2 | 采用（dev） | Rust 基准 |
| `tempfile` | 3.27.0 | 采用 | 临时文件 |
| `rustix` / `windows-sys` | 1.1.6 / 0.61.2 | 采用 | 平台文件/进程原语 |

不采用：

- `miette`（2025-04 后更新较慢，诊断用 `thiserror` + 自有错误码足够）；
- 任何 nightly-only SIMD 库；AVX2 内核放在 C++。

---

## 11. C++ 依赖评估

### 11.1 结论表

| 库 | 最新版本 | 结论 | 理由 |
|---|---|---|---|
| `fmt` | 12.2.1（2026-11-07） | **采用** | 统一 C++ 格式化/日志；避免 MSVC/libc++ 的 `std::format` 差异 |
| `doctest` | 2.5.3（2026-07-06） | **采用** | 单元测试；header-only、编译快、足够 |
| `oneTBB` | 2023.2.0（2026-10-28） | **采用** | CPU 并行内核；成熟、Windows/Linux |
| `GoogleTest` | 1.18.0（2026-08-10） | **不采用** | 与 doctest 重复；编译重；本项目不需要 mock |
| `Catch2` | 3.16.1（2026-10-08） | **不采用** | 与 doctest 重复；功能更强但更重 |
| `taskflow` | 4.1.0（2026-06-20） | **不采用** | 与 oneTBB 重复；Rust 已负责文件级 DAG |
| `folly` | 2026.10.26.00（2026-10-26） | **不采用** | 过重；Windows 大量测试禁用；C++20；依赖 Python/大量第三方；与精简栈冲突 |
| `moodycamel::ConcurrentQueue` | 1.0.5（2026-04-03） | **条件采用** | 仅当 C++ 数据面自己拥有 MPMC 队列时 |
| `google/benchmark` | 1.10.0（2026-10-14） | **采用（dev）** | C++ AVX2 内核基准 |

### 11.2 fmt

优点：

- 12.2.1，2026-11-07，维护活跃；
- 格式串编译期检查；
- Windows/Linux 一致；
- 可作为日志、错误、CLI 诊断的统一格式化层；
- 与 oneTBB/doctest 无冲突。

注意：

- C++23 已有 `std::format`/`std::print`；
- 如果团队坚持“标准库优先”，可以去掉 fmt，改用 `std::format`；
- 但一旦选择 fmt，C++ 侧统一用 fmt，不混用两套格式化。

### 11.3 doctest

选择理由：

- 2.5.3，2026-07-06；
- header-only，编译快；
- 足够覆盖 AVX2 内核、buffer、错误路径；
- 不需要 mock；
- C++ 集成测试可由 Rust `assert_cmd` 驱动 CLI，不需要 GTest/Catch2。

不选 GoogleTest：

- 1.18.0 很成熟，但编译重、依赖多；
- 本项目 C++ 侧是内核库，不是大型服务；
- GTest 的 death test/mock 优势用不上。

不选 Catch2：

- 3.16.1 功能强，但与 doctest 重叠；
- 多一个编译库和测试 DSL；
- 如果未来需要 generators/benchmarks，再评估引入 Catch2，而不是现在同时装三套。

### 11.4 oneTBB

选择理由：

- 2023.2.0，2026-10-28；
- Intel 维护，Windows/Linux 支持成熟；
- `parallel_for`、`task_group`、`concurrent_queue` 足够；
- 适合 AVX2 内核的批量处理；
- 与 FFmpeg CLI 进程模型不冲突。

注意：

- 必须限制 `tbb::global_control` 的并行度，避免和 tokio/crossbeam/FFmpeg 线程叠加导致超订；
- 每个 job 用独立 `task_arena`，避免全局池被长任务占满；
- 如果 C++ 只做很小的内核，oneTBB 可能过度；但按“高性能并发”目标，保留。

不选 taskflow：

- 4.1.0，header-only，DAG 表达好；
- 但 Rust 已经负责文件级/阶段级 DAG；
- C++ 再引入 taskflow 会出现两套调度器；
- oneTBB 的并行算法足够覆盖内核级并行；
- 避免“为了 DAG 而 DAG”。

### 11.5 folly

不采用，理由具体：

- 2026.10.26 发布，维护活跃；
- 但体量巨大，依赖多（Boost、glog、gflags、double-conversion、libevent、OpenSSL 等）；
- README 明确 Windows 上大量测试被禁用；
- 主要支持 C++20，C++23/26 支持不确定；
- 构建脚本依赖 Python；
- 与“只 Rust/C++、xmake + Conan、精简依赖”的目标冲突；
- 我们需要的东西（Future/Executor/MPMC/IOBuf）可以用 oneTBB、crossbeam、moodycamel、std 替代。

如果未来确实需要 `folly::IOBuf` 做零拷贝网络，再单独评估，不现在引入。

### 11.6 moodycamel::ConcurrentQueue

- 1.0.5，2026-04-03；
- header-only，跨平台；
- 如果 C++ 数据面自己拥有 packet/frame 队列，建议采用；
- 如果队列仍在 Rust/crossbeam，则不需要。

结论：**条件采用**，不默认引入。

### 11.7 Google Benchmark

- 1.10.0，2026-10-14；
- 只用于 C++ AVX2 内核的微基准；
- 不进入产品运行时；
- 与 Rust criterion 分工：Rust 测调度/IO，C++ 测内核。

---

## 12. CLI

### 12.1 命令

```text
lae-videodamage probe <input> [--json]
lae-videodamage plan <recipe.toml> --input <input> --out <plan.fdpl>
lae-videodamage run --input <input> --recipe <recipe.toml> --out <output>
lae-videodamage run --input <input> --plan <plan.fdpl> --out <output>
lae-videodamage verify <output> [--json] [--decode]
lae-videodamage manifest <output> [--json]
lae-videodamage capabilities [--json]
lae-videodamage scenario list
lae-videodamage scenario show <name>
lae-videodamage explain <code>
lae-videodamage version
```

### 12.2 参数

```text
--config <PATH>
--ffmpeg-dir <PATH>
--json / --jsonl
--quiet
--progress plain|off
--jobs <N>
--max-memory <MB>
--seed <U64>
--plan-in/out <PATH>
--manifest-out <PATH>
--verify off|probe|decode|full
--play
--force / --no-clobber
--timeout <SECS>
```

### 12.3 输出

- stdout：JSON/manifest 路径；
- stderr：人类可读摘要、进度、错误；
- 退出码：0/1/2/130；
- `--play` 调用 ffplay，默认关闭；
- 不实现任何菜单、向导、TUI、GUI。

---

## 13. 插件

### 13.1 C ABI

```c
#define LAE_VD_ABI_VERSION 1

typedef struct lae_vd_handle lae_vd_handle;

typedef struct {
    const char* input_path;
    const char* output_path;
    const char* recipe_toml;
    const char* plan_path;
    uint64_t seed;
    uint32_t flags;
} lae_vd_request;

typedef struct {
    void (*on_progress)(void* user, const char* stage, uint64_t done, uint64_t total);
    void (*on_log)(void* user, int32_t level, const char* message);
    void* user;
} lae_vd_callbacks;

uint32_t lae_vd_abi_version(void);
lae_vd_handle* lae_vd_create(const char* config_toml, char* err, size_t err_len);
int32_t lae_vd_run(lae_vd_handle*, const lae_vd_request*, const lae_vd_callbacks*);
void lae_vd_cancel(lae_vd_handle*);
void lae_vd_destroy(lae_vd_handle*);
```

要求：

- `extern "C"`、`noexcept`；
- 内部 `catch (...)`；
- opaque handle；
- UTF-8；
- 不传 Rust 类型；
- 版本不匹配直接拒绝；
- 每个 handle 独立；
- 取消线程安全；
- 不依赖 CWD；
- 三件套路径由配置/宿主提供。

### 13.2 三种形态

| 形态 | 场景 |
|---|---|
| CLI | 独立使用、脚本、CI |
| Rust crate | LAE v8 同仓库集成 |
| C ABI plugin | 独立发布、C++ 宿主、其他语言 |

核心 `DamageService` 只有一份。

---

## 14. 完整性与可播放性

### 14.1 输出事务

```text
temp file → write → fsync(file) → replace → fsync(dir) → manifest
```

- Windows `ReplaceFileW`；
- Linux `renameat2`；
- 保留 ACL/xattr/mode；
- journal + recovery；
- 失败 quarantine；
- `kill -9` 残留可识别；
- 默认 no-clobber。

FFmpeg 写临时文件时，Rust 必须检查退出码和文件大小；不能只看 `ffmpeg` 返回 0。

### 14.2 全流验证

- ffprobe 检查所有流；
- 检查 stream count、codec、duration、disposition；
- 检查字幕/多音轨/章节/附件是否保留；
- 解码验证 `ffmpeg -f null`；
- `ffplay` 可选人工播放；
- `playability`：clean / glitchy / damaged / unplayable。

### 14.3 确定性标注

- Tier A（自己的 parser + 自己的改写）：D1；
- Tier C/D（重编码/滤镜）：决策确定，字节依赖编码器；
- Tier B（FFmpeg BSF 随机数）：D3；
- 容器输出：同一 FFmpeg 三件套构建内可复现，跨构建不承诺；
- manifest 记录 trio 版本、编码器、muxer、CPU 基线。

---

## 15. 并发与性能

### 15.1 调度

- Rust：
  - tokio 管 FFmpeg 子进程、stderr drain、超时、取消；
  - crossbeam 管 CPU 队列；
  - rayon 管扫描/批量；
  - parking_lot/dashmap 管共享状态。
- C++：
  - oneTBB 管 AVX2 内核并行；
  - 每个 job 独立 `task_arena`；
  - 限制 `global_control`，避免和 tokio/FFmpeg 线程超订。
- FFmpeg：
  - 每个进程 `-threads` 受控；
  - `--jobs` 同时限制文件数、进程数、估算内存；
  - 不能简单 `min(核数, 8)`。

### 15.2 目标

| 指标 | 目标 |
|---|---|
| 2 GB H.264 扫描 + plan | ≤ 3 s |
| 2 GB damage 全流程 | ≤ 8 s |
| 峰值 RSS | ≤ 256 MB（默认） |
| Ctrl-C 响应 | ≤ 200 ms |
| 4K 单帧处理 | 不 OOM |

### 15.3 进程监督

- `tokio::process::Command`；
- `kill_on_drop(true)`；
- Windows Job Object；Linux `setsid` + `killpg`；
- stdout/stderr 各自 drain；
- `-progress pipe:1` 解析进度；
- 超时 kill；
- 临时文件 Drop 清理；
- 不允许 stderr 管道无人读。

---

## 16. Windows / Linux

### 16.1 Windows

- MSVC 2026；
- `/std:c++latest`、`/Zc:__cplusplus`、`/permissive-`、`/utf-8`；
- `/arch:AVX2`；
- `/MD` 插件、`/MT` 可选 CLI；
- `ReplaceFileW`；
- Job Object 杀进程树；
- UNC/SMB 路径测试；
- FFmpeg trio 使用 MSVC 构建。

### 16.2 Linux

- Clang 23.1.6 + libc++ 23；
- `-std=c++26` / `-std=c++23`；
- `-march=x86-64-v3`；
- `renameat2`、`fsync(dirfd)`；
- `setsid` + `killpg`；
- FFmpeg trio 使用 clang 兼容构建；
- Debian 12 glibc 2.36 测试。

---

## 17. 测试与 CI

### 17.1 Rust

- `cargo nextest`；
- `rstest` fixtures；
- `insta` 快照（JSON/manifest/plan dump）；
- `loom` 并发状态机；
- `proptest` 属性测试；
- `assert_cmd` CLI 集成；
- `criterion` 基准；
- 真实 FFmpeg 三件套集成测试；
- parser fuzz（cargo-fuzz / libFuzzer）。

### 17.2 C++

- `doctest` 单测；
- AVX2 内核基准（Google Benchmark）；
- C++23/C++26 双标准；
- MSVC/Clang 双编译器；
- sanitizers（ASAN/UBSAN）在 Linux CI；
- oneTBB 并行正确性测试。

### 17.3 CI 矩阵

- Windows Server 2025 + VS2026；
- Ubuntu 24.04 + Clang 23；
- Debian 12 + Clang 23；
- AVX2 runner；
- `cargo fmt --check`、`clippy -D warnings`；
- C++ warnings-as-errors；
- `cargo-deny` / `cargo-audit`；
- 性能阈值；
- 发布物校验和。

### 17.4 语料

- 真实 30 分钟以上归档；
- 多音轨 + 字幕 + 章节 + 附件；
- H.264 / HEVC / AV1 / VP9 / MPEG-2 / ProRes / FFV1；
- VFR、10-bit、HDR；
- 截断/损坏容器；
- 进程超时/取消/半成品清理。

---

## 18. 风险

| 风险 | 影响 | 对策 |
|---|---|---|
| FFmpeg CLI 进程开销 | 短文件慢 | 批量复用/合并进程；接受启动成本 |
| FFmpeg 版本变化 | 输出字节变化 | 只共享 LAE 最新 trio；manifest 记录；不跨版本承诺 D1 |
| BSF 随机数 | packet 操作 D3 | 如实标注；不假装 D1 |
| 全流保留复杂 | 字幕/多音轨丢失 | `-map 0` + 逐流验证 + 真实语料 |
| HEVC/AV1 parser 复杂 | 码流级覆盖慢 | 先 H.264/HEVC；AV1 后置 |
| TOML 1.1 解析器 | 生态支持 | `toml 1.2.1+spec-1.1.0`；Rust 单点解析 |
| AVX2 强制 | 老机器无法运行 | 启动 CPUID 检查；文档明确 |
| 多调度器超订 | 性能下降 | tokio/crossbeam/oneTBB/FFmpeg 线程配额 |
| 插件 ABI | 宿主崩溃 | 版本化 C ABI、opaque handle、catch |
| folly/taskflow 引入 | 构建/维护爆炸 | 已拒绝 |
| GPL/nonfree | 发布合规 | 明确许可证与 FFmpeg 构建来源 |

---

## 19. 路线图

### M0：三件套 + TOML + 全流复制

- FFmpeg 三件套发现/版本校验/能力探测；
- TOML 1.1 schema；
- CLI `probe` / `copy`；
- C++ 内核骨架 + doctest；
- 全流复制，4 流进 4 流出；
- 输出事务 + manifest；
- Windows/Linux 构建。

**验收：** 4 流输入 4 流输出；逐流 hash 一致；C++23/C++26 都能编译；三件套不一致直接报错。

### M1：frame/pixel + 更多编码

- `frame-drop` / `freeze` / `repeat`；
- 12+ 视觉 effect；
- HEVC/AV1/VP9/ProRes/FFV1 等 frame/pixel 操作；
- 全流保留；
- 真实语料。

### M2：码流级 H.264/HEVC

- Rust parser；
- C++ AVX2 改写内核；
- plan v5；
- fuzz；
- D1 验证。

### M3：packet 级 + AV1/VP9

- packet 级 drop/reorder（D3 标注）；
- AV1 OBU/VP9 superframe parser；
- 性能优化。

### M4：插件 + CI + 发布

- C ABI plugin；
- LAE v8 集成；
- CI 矩阵；
- 性能门禁；
- 发布。

---

## 20. 关键决策（ADR 摘要）

| 编号 | 决策 | 理由 |
|---|---|---|
| D1 | 产品代码只 Rust/C++ | 约束 |
| D2 | Rust 控制面 + C++ AVX2 数据面 | 分工清晰，避免 C++ 进程管理重复 |
| D3 | FFmpeg 三件套为媒体层 | 约束，格式覆盖最大化 |
| D4 | 只用 LAE 共享最新 trio | 版本一致性、可复现边界 |
| D5 | 默认全流保留 | 归档完整性 |
| D6 | TOML 1.1，Rust 单点解析 | 约束，避免 C++ parser 分裂 |
| D7 | x86-64-v3 强制，无降级 | 性能目标 |
| D8 | Rust 1.99.1 edition 2024 | 约束 |
| D9 | C++26 目标、C++23 下限 | 编译器现实 |
| D10 | CLI only，clap | 约束 |
| D11 | 独立 CLI + Rust crate + C ABI plugin | 约束 |
| D12 | 依赖：tokio/crossbeam/parking_lot/dashmap 6.2.1/nextest/loom/insta/rstest + oneTBB/doctest/fmt | 精简可用 |
| D13 | 拒绝 folly/taskflow/GTest/Catch2 | 过重/重复 |
| D14 | 输出事务 + journal + quarantine | 文件完整性 |
| D15 | packet 级 D3 如实标注 | 不假装 D1 |

---

## 21. 结论

这版约束下，项目的正确形态不是“Rust 重写 + libav 替代 FFmpeg”，而是：

> **Rust 做可复现的控制面，C++ 做 AVX2 数据面，FFmpeg 三件套做格式与编解码底座。**

最重要的三条工程红线：

1. **全流保留**：默认 `-map 0`，只处理目标流，字幕/多音轨/章节/附件不能静默丢。
2. **三件套一致**：只使用 LAE 共享的自构建 FFmpeg 构建；build ID/`buildconf` 不一致直接失败。
3. **确定性诚实**：码流级 D1、重编码决策确定、packet BSF D3；容器输出只承诺同 trio 构建内可复现。

C++ 依赖只保留真正有职责的：

- `oneTBB`：AVX2 内核并行；
- `doctest`：单元测试；
- `fmt`：统一格式化/诊断；
- 可选：`moodycamel::ConcurrentQueue`、Google Benchmark。

不引入 `folly`、`taskflow`、`GoogleTest`、`Catch2`：它们不是“不好”，而是与本项目的规模、构建方式、Windows 支持和职责划分不匹配。

下一步应先做 M0：三件套发现/校验、TOML 1.1 schema、全流复制、C++ 内核骨架、输出事务。M0 通过后，再写任何一种 damage。

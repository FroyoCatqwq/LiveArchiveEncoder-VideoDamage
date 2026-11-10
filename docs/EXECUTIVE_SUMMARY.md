# 结论摘要

> 日期：2026-11-10
> 状态：编码前第二轮核查完成，尚未开始实现
> 第二轮核查：[`PRE_CODING_ROUND2_CHECK.md`](PRE_CODING_ROUND2_CHECK.md)
> Go/No-Go 基线：[`PRE_CODING_FINAL_REVIEW.md`](PRE_CODING_FINAL_REVIEW.md)
> 分工与准备：[`WORK_BREAKDOWN.md`](WORK_BREAKDOWN.md)
> FFmpeg 契约：[`FFMPEG_PROVIDER_REVIEW.md`](FFMPEG_PROVIDER_REVIEW.md)
> 完整架构：[`ARCHITECTURE_REPORT.md`](ARCHITECTURE_REPORT.md)

---

## 1. 一句话结论

`LiveArchiveEncoder-VideoDamage` 应采用：

> **Rust 控制面 + C++ AVX2 数据面 + 外部已安装的 BtbN `n9.0` 稳定版 FFmpeg 三件套。**

- **Rust** 管：CLI、TOML 1.1、Recipe/Plan、trio 锁文件、进程监督、调度、输出事务、Manifest、插件 C ABI。
- **C++** 管：AVX2/SSE4 扫描与改写内核、oneTBB 并行、buffer/批处理。
- **FFmpeg / ffprobe / ffplay** 管：解封装、封装、编解码、滤镜、探测、可选播放。

Core 只消费**已安装**的 FFmpeg，不下载、不安装、不部署。

---

## 2. 旧项目怎么处理

`LAE-Damager` / `frame_damage v3` 是研究原型，不是新项目的代码基座。

保留：

- `scan → plan → execute → verify` 四段式；
- FDPL 的“计划即凭证”思想；
- PCG32 + 每帧独立随机流 + 整数概率；
- 码流扫描算法、夹具、错误码、容器矩阵；
- 不 mock FFmpeg 的 e2e 方法；
- 性能归因与审计方法。

不继承：

- 只映射第一条视频/音频；
- 只保证进程内不污染目标的“原子写”；
- 把容器输出标成 `determinism = full`；
- 同步阻塞的 `apply`；
- 无 CI、无真实长归档语料、无 parser fuzz 的交付方式。

---

## 3. 目标架构

```text
LAE v8 Host
  │ Rust crate / C ABI
  ▼
Rust 控制面
  CLI / TOML 1.1 / Recipe / Plan
  FFmpeg trio 发现、锁文件校验、进程监督
  Manifest / 输出事务 / 插件 C ABI
  │ cxx
  ▼
C++ 数据面
  AVX2/SSE4 扫描与改写内核
  oneTBB 并行
  buffer / 批处理
  │
  ▼
FFmpeg / ffprobe / ffplay
（BtbN n9.0 最新稳定版，ffmpeg-trio.lock 锁定）
```

---

## 4. 硬约束

| 约束 | 决策 |
|---|---|
| 仅 Rust + C++ | 产品代码只 Rust/C++；Zig/Python 不进入新仓库 |
| Rust 版本 | 1.99.1，edition 2024，只用 stable |
| C++ 标准 | 目标 C++26，最低 C++23；C++26 特性用 feature-test macro 守卫 |
| 平台 | Windows x64 + Linux x86_64；MSVC 2026 / Clang 23 |
| 界面 | 仅 CLI；不做 CUI/GUI/TUI |
| 形态 | 独立 CLI + Rust crate + C ABI 插件 |
| 媒体层 | Core 只消费已安装的 BtbN `n9.0` 稳定版 GPL 三件套；`ffmpeg-trio.lock` 锁定 |
| FFmpeg 安装 | Core 不下载、不安装、不部署、不更新；Extension Pack 负责 |
| 配置 | TOML 1.1 only；Rust 单点解析；不向下兼容 |
| CPU | 本项目 Rust/C++ 强制 x86-64-v3；FFmpeg 预构建不承诺编译期 v3 |
| 确定性 | D1 只在同 `provider_fingerprint` 内成立；跨版本/跨资产不承诺 |

---

## 5. 编码覆盖

按层级定义，不承诺“所有编码都能码流级 damage”。

| Tier | 能力 | 覆盖 |
|---|---|---|
| A | 码流级改写 | H.264、HEVC、MPEG-4、MPEG-1/2、H.263 |
| B | packet 级丢包/重排 | 所有 FFmpeg 能 `-c copy` 的编码 |
| C | 解码帧操作 | HEVC、VP9、AV1、ProRes、DNxHD、FFV1、MJPEG、Theora、WMV、RV 等 |
| D | 像素/滤镜 | 所有 FFmpeg 可解码编码 |
| E | 容器/文件级 | MP4、MKV、AVI、TS、FLV、ASF、Ogg、RM 等 |

第一批码流级：H.264、HEVC。  
第二批：AV1、VP9。  
其他编码先走 frame/pixel 级，不假装支持码流级。

---

## 6. 依赖结论

### Rust

采用：

`tokio`、`crossbeam`、`parking_lot`、`dashmap 6.2.1`、`cargo-nextest`、`loom`、`insta`、`rstest`。

补充：

`clap`、`serde`、`toml 1.2.1+spec-1.1.0`、`thiserror`、`tracing`、`rayon`、`cxx`、`proptest`、`assert_cmd`、`criterion`、`tempfile`、`rustix` / `windows-sys`。

注意：

- `dashmap` 用 6.2.1，不用 7.0 RC；
- `loom` 只用于自定义并发结构；
- `tokio` 只用于控制面。

### C++

采用：

- `oneTBB`：AVX2 内核并行；
- `doctest`：单元测试；
- `fmt`：统一格式化/诊断。

条件采用：

- `moodycamel::ConcurrentQueue`：仅当 C++ 数据面自己拥有 MPMC 队列；
- `google/benchmark`：仅用于 C++ 内核微基准。

不采用：

- `folly`：过重、Windows 大量测试禁用、C++20、Python 构建、依赖爆炸；
- `taskflow`：与 oneTBB 重复；
- `GoogleTest` / `Catch2`：与 doctest 重复；
- `toml++`：Rust 单点解析 TOML；
- `spdlog`：fmt + 简单日志足够。

---

## 7. FFmpeg Core / Extension 契约

### Core 只消费

Core 负责：

- 发现已安装的 `ffmpeg` / `ffprobe` / `ffplay`；
- 读取并校验 `ffmpeg-trio.lock`；
- 校验版本、`buildconf`、二进制 SHA-256；
- 探测并缓存能力；
- 编排 FFmpeg 进程；
- 生成 Manifest。

Core 不做：

- 不下载；
- 不解压；
- 不安装；
- 不部署；
- 不更新；
- 不创建 PATH 软链接；
- 不处理 nonfree；
- 不自构建。

### Extension Pack 负责安装部署

FFmpeg Provider Extension Pack 负责：

- 解析 BtbN `n9.0` 最新稳定版；
- 下载 GPL 资产；
- 校验 `checksums.sha256`；
- 解压、安装、部署；
- 生成 `ffmpeg-trio.lock`；
- 更新/回滚；
- 未来支持 nonfree / 自构建。

### 当前支持

```text
provider     = "btbn"
repository   = "FFmpeg-Builds/FFmpeg-Builds"
release_line = "n9.0"
channel      = "stable"
variant      = "gpl"
target       = "win64" | "linux64"
```

当前不接受：

- `master-latest`；
- `n9.1-dev`；
- `n8.1`；
- `nonfree`；
- `lgpl`；
- 非 BtbN 构建。

### 锁文件

Core 默认读取：

```text
<ffmpeg-dir>/ffmpeg-trio.lock
```

`run` / `verify` 默认要求有效锁文件；缺失或哈希不匹配返回 `LAE-VD-E1004`。

`probe` / `capabilities` 可以无锁运行，但必须标记：

```text
provenance = "unverified"
```

---

## 8. 确定性、CPU 与许可证

### 确定性

| 操作 | 口径 |
|---|---|
| 码流级 damage / truncate | D1：同输入 + 同 plan + 同 `provider_fingerprint` |
| frame/pixel | 决策确定；编码字节依赖 encoder/buildconf |
| packet BSF | D3：FFmpeg 内部随机数 |
| 容器输出 | 同 provider + release + asset hash + buildconf 内可复现 |

Manifest 记录：

- 输入/输出 hash；
- plan hash；
- provider / release tag / asset SHA-256；
- 三件套二进制 SHA-256；
- buildconf / capabilities hash；
- 编码器/muxer；
- 项目 CPU 基线与 FFmpeg CPU 基线；
- 验证结果。

### CPU

- 本项目 Rust/C++：强制 x86-64-v3，无 scalar fallback；
- BtbN 预构建 FFmpeg：不承诺编译期 v3；
- FFmpeg SIMD 以 `runtime-cpudetect` / `buildconf` 为准；
- Manifest 分开记录两个 CPU 基线。

### 许可证

- Core 只接受 BtbN GPL 变体；
- nonfree 不属于 core；
- Extension Pack 未来引入 nonfree 时，不得进入 public release；
- LICENSES / provenance 记录 provider、release tag、资产名、哈希。

---

## 9. 编码前状态

当前仍是“文档完整、工程为零”。

允许：

- 冻结契约；
- 搭仓库骨架；
- 写 `probe` / `copy`；
- 手工安装 BtbN `n9.0`；
- 做全流复制。

不允许：

- 写 damage 算法；
- 把 FFmpeg 安装器写进 core；
- 支持 nonfree / 自构建；
- 无锁直接 `run`；
- 跳过 4 流门禁。

详细 Go/No-Go 见 [`PRE_CODING_FINAL_REVIEW.md`](PRE_CODING_FINAL_REVIEW.md)。

---

## 10. 先做什么

```text
Phase 0  冻结 C1–C10
   ↓
Phase 1  搭仓库骨架
   ↓
Phase 2  手工安装 BtbN n9.0 + 生成 trio.lock
   ↓
Phase 3  夹具 / CI / 性能基线
   ↓
M0       全流复制硬门禁
   ↓
M1       frame/pixel + 更多编码
   ↓
M2       码流级 H.264/HEVC
   ↓
M3       packet 级 + AV1/VP9
   ↓
M4       插件 / CI / 发布
```

M0 验收：

- 4 流输入 → 4 流输出；
- 每流 hash 一致；
- 字幕/第二音轨/章节/附件不丢；
- 锁文件/版本/buildconf/SHA-256 不匹配直接失败；
- Ctrl-C 清理临时文件；
- Windows/Linux 构建；
- C++23/C++26 构建。

---

## 11. 主要风险

| 风险 | 对策 |
|---|---|
| FFmpeg 进程开销 | 批量合并、控制 `-threads` |
| 锁文件漂移 | Extension Pack 生成具体 autobuild tag + SHA-256 |
| FFmpeg 版本变化 | `provider_fingerprint` + Manifest；不跨版本承诺 D1 |
| packet BSF 随机数 | 如实标注 D3 |
| 全流保留复杂 | `-map 0` + 逐流验证 + 真实语料 |
| HEVC/AV1 parser 复杂 | 先 H.264/HEVC，AV1 后置 |
| TOML 1.1 生态 | `toml 1.2.1+spec-1.1.0`；Rust 单点解析 |
| 项目 x86-64-v3 | 启动 CPUID 检查；FFmpeg CPU 基线单独记录 |
| 多调度器超订 | tokio/crossbeam/oneTBB/FFmpeg 线程配额 |
| 插件 ABI | 版本化 C ABI、opaque handle、catch |
| GPL/nonfree | nonfree 仅 Extension Pack 私有；不进 public release |

---

## 12. 第二轮核查修正

- Core 从“双来源”收敛为**只消费 BtbN `n9.0` 稳定版 GPL 三件套**；
- FFmpeg 下载/安装/部署/更新从 core 移出，转 **Extension Pack**；
- BnqDzj nonfree / 自构建降级为 Extension Pack 未来能力；
- `run` / `verify` 默认要求 `ffmpeg-trio.lock`；
- `probe` 无锁可诊断，但必须标记 `provenance=unverified`；
- 明确 C1–C10 是唯一冻结清单；
- 明确所有 TOML / CLI / C ABI / 接口片段均为草案；
- 明确文档权威顺序与冲突裁决规则；
- 明确 Phase 0–4、M0 硬门禁与 M1–M4 功能里程碑；
- 明确 M0 通过前不写 damage 算法。

---

## 13. 相关文档

- 第二轮核查：[`PRE_CODING_ROUND2_CHECK.md`](PRE_CODING_ROUND2_CHECK.md)
- Go/No-Go 基线：[`PRE_CODING_FINAL_REVIEW.md`](PRE_CODING_FINAL_REVIEW.md)
- 分工与准备：[`WORK_BREAKDOWN.md`](WORK_BREAKDOWN.md)
- FFmpeg 契约：[`FFMPEG_PROVIDER_REVIEW.md`](FFMPEG_PROVIDER_REVIEW.md)
- 完整架构：[`ARCHITECTURE_REPORT.md`](ARCHITECTURE_REPORT.md)
- 自构建 FFmpeg（暂缓）：[`FFMPEG_BUILD_REPORT.md`](FFMPEG_BUILD_REPORT.md)
- 依赖评估：[`LIBRARY_DECISIONS.md`](LIBRARY_DECISIONS.md)
- 文档索引（docs/README.md）：[`README.md`](README.md)

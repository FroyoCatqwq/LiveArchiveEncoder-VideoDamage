# 结论摘要

> 日期：2026-11-10  
> 状态：架构评估阶段，尚未开始实现  
> 完整报告：[`ARCHITECTURE_REPORT.md`](ARCHITECTURE_REPORT.md)  
> 当前 FFmpeg 策略：[`FFMPEG_PROVIDER_REVIEW.md`](FFMPEG_PROVIDER_REVIEW.md)  
> 自构建 FFmpeg 评估（已暂缓）：[`FFMPEG_BUILD_REPORT.md`](FFMPEG_BUILD_REPORT.md)  
> 依赖决策：[`LIBRARY_DECISIONS.md`](LIBRARY_DECISIONS.md)

---

## 1. 一句话结论

`LiveArchiveEncoder-VideoDamage` 应采用：

> **Rust 控制面 + C++ AVX2 数据面 + BtbN / BnqDzj 双来源预构建 FFmpeg 三件套。**

- **Rust** 管：CLI、TOML 1.1、Recipe/Plan、FFmpeg provider 解析、进程监督、调度、输出事务、Manifest、插件 C ABI。
- **C++** 管：AVX2/SSE4 扫描与改写内核、oneTBB 并行、buffer/批处理。
- **FFmpeg / ffprobe / ffplay** 管：解封装、封装、编解码、滤镜、探测、可选播放。

自构建 FFmpeg 暂缓；当前直接使用两家上游预构建，通过 `provider_fingerprint` 锁定来源与指纹。

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
  FFmpeg provider 解析、版本/指纹校验、进程监督
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
（BtbN / BnqDzj 双来源预构建，provider_fingerprint 锁定）
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
| 媒体层 | 外部预构建 FFmpeg / ffprobe / ffplay；当前支持 `btbn` 与 `bnqdzj-nonfree`；release tag + asset SHA-256 + buildconf hash 锁定 |
| 配置 | TOML 1.1 only；Rust 单点解析；不向下兼容 |
| CPU | 本项目 Rust/C++ 强制 x86-64-v3；FFmpeg 上游预构建不承诺编译期 v3，以 runtime cpudetect 为准 |
| 确定性 | 码流级 D1；重编码决策确定；packet BSF D3；容器只承诺同 `provider_fingerprint` 内可复现 |

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

不同 provider 的 `buildconf` 和能力矩阵可能不同；实际能力以 `capabilities.toml` 为准，不按 provider 名称硬编码。

---

## 6. 依赖结论

### Rust

采用：

`tokio`、`crossbeam`、`parking_lot`、`dashmap 6.2.1`、`cargo-nextest`、`loom`、`insta`、`rstest`。

补充：

`clap`、`serde`、`toml 1.2.1+spec-1.1.0`、`thiserror`、`tracing`、`rayon`、`cxx`、`proptest`、`assert_cmd`、`criterion`、`tempfile`、`rustix` / `windows-sys`。

注意：

- `dashmap` 用 6.2.1，不用 7.0 RC；
- `loom` 只用于自定义并发结构，不模拟 tokio；
- `tokio` 只用于控制面，CPU 密集任务走 crossbeam/rayon。

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
- `taskflow`：与 oneTBB 重复，Rust 已负责文件级 DAG；
- `GoogleTest` / `Catch2`：与 doctest 重复，编译重；
- `toml++`：Rust 单点解析 TOML，C++ 不需要；
- `spdlog`：fmt + 简单日志足够。

### FFmpeg 来源策略

当前支持两家上游预构建：

| 简称 | 实际仓库 | 当前角色 |
|---|---|---|
| `btbn` | `FFmpeg-Builds/FFmpeg-Builds`（`BtbN/FFmpeg-Builds` 指向同一仓库） | GPL / LGPL；适合公开/合规场景 |
| `bnqdzj-nonfree` | `BnqDzj/FFmpeg-Builds-nonfree` | nonfree；适合 LAE/VideoDamage 内部 |

事实：

- `bnqdzj/nonfree` 不是有效 GitHub 仓库路径；实际仓库是 `BnqDzj/FFmpeg-Builds-nonfree`。
- BtbN 当前 `latest` release 不发布 `nonfree` 资产；最近 37 个 release 的资产中 nonfree 数量为 0。
- BnqDzj 的 `latest` release 提供 `nonfree` / `nonfree-shared` 静态与共享资产。
- `nonfree` 二进制不可再分发，不能进入 public GitHub Release。
- 自构建 FFmpeg 暂缓；详见 [`FFMPEG_PROVIDER_REVIEW.md`](FFMPEG_PROVIDER_REVIEW.md) 与 [`FFMPEG_BUILD_REPORT.md`](FFMPEG_BUILD_REPORT.md)。

### Provider 指纹

不用 floating `latest` 当运行时基线；必须记录：

- provider；
- channel；
- release tag；
- asset name；
- asset SHA-256；
- 三件套二进制 SHA-256；
- `ffmpeg -version` banner；
- `ffmpeg -buildconf` hash；
- capabilities hash；
- `provider_fingerprint`。

能力缓存按 `provider_fingerprint` 索引；配置错误、buildconf 不匹配或 SHA-256 不匹配直接 `LAE-VD-E1004`。

---

## 7. 确定性口径

| 操作 | 口径 |
|---|---|
| 码流级 damage / truncate | D1：同输入 + 同 plan + 同 `provider_fingerprint` → 同输出 |
| frame-drop / visual-loss | 决策确定；编码字节依赖 encoder/buildconf |
| visual-glitch | 滤镜图确定；像素/编码字节依赖 encoder/buildconf |
| packet-loss / audio-loss | D3：FFmpeg BSF 内部随机数，同 seed + 同 buildconf 尽力复现 |
| 容器输出 | 同 provider + release + asset hash + buildconf 内可复现；跨 provider / 跨 release 不承诺 |

manifest 必须记录：

- 输入/输出 hash；
- plan hash；
- provider、channel、release tag；
- asset SHA-256 与三件套二进制 SHA-256；
- `buildconf` hash、capabilities hash；
- 编码器/muxer；
- 项目 CPU 基线与 FFmpeg CPU 基线；
- 验证结果。

---

## 8. 先做什么

### M0：双来源三件套 + TOML + 全流复制

0. 实现 `FfmpegProvider` / `TrioDescriptor` / `provider_fingerprint`，建立 BtbN 与 BnqDzj provider 注册表。
1. 固定 provider 的 release tag、asset SHA-256 与能力矩阵；floating `latest` 只作为更新入口。
2. FFmpeg 三件套发现、版本/`buildconf`/SHA-256 一致性校验、能力探测。
3. TOML 1.1 schema + Rust 解析 + `deny_unknown_fields`。
4. CLI `probe` / `copy`，增加 `--ffmpeg-provider`、`--ffmpeg-release`、`--ffmpeg-variant`。
5. C++ 内核骨架 + doctest + oneTBB。
6. 全流复制：4 流输入 → 4 流输出，逐流 hash 一致。
7. 输出事务 + manifest。
8. Windows/Linux 构建，C++23/C++26 双标准。

**M0 验收：**

- BtbN GPL/LGPL 与 BnqDzj nonfree 都能独立通过；
- 4 流输入 4 流输出；
- 每流 hash 一致；
- 三件套版本不一致直接报错；
- provider 声明与 `buildconf` 不匹配直接 `LAE-VD-E1004`；
- release tag / asset SHA-256 / binary SHA-256 不匹配直接失败；
- floating `latest` 不进入 manifest；
- nonfree 构建不进入公开发布资产清单；
- C++23/C++26 都能编译；
- AVX2 CPUID 检查生效；
- Ctrl-C 能清理临时文件。

M0 通过后再写 packet/frame/bitstream damage。

---

## 9. 主要风险

| 风险 | 对策 |
|---|---|
| FFmpeg 进程开销 | 批量合并、接受启动成本、控制 `-threads` |
| 双来源能力差异 | 按 `buildconf` 与 capabilities 决策；不按 provider 名称硬编码 |
| floating `latest` 不可复现 | 只作为更新入口；运行时 pin release tag + SHA-256 |
| FFmpeg 版本变化 | provider_fingerprint + manifest 记录；不跨 fingerprint 承诺 D1 |
| packet BSF 随机数 | 如实标注 D3 |
| 全流保留复杂 | `-map 0` + 逐流验证 + 真实语料 |
| HEVC/AV1 parser 复杂 | 先 H.264/HEVC，AV1 后置 |
| TOML 1.1 生态 | `toml 1.2.1+spec-1.1.0`；Rust 单点解析 |
| 项目 x86-64-v3 | 启动 CPUID 检查；文档明确；FFmpeg 上游预构建单独标注 |
| 多调度器超订 | tokio/crossbeam/oneTBB/FFmpeg 线程配额 |
| 插件 ABI | 版本化 C ABI、opaque handle、catch |
| GPL/nonfree | BtbN GPL 可公开；BnqDzj nonfree 仅私有；明确许可证与来源 |

---

## 10. 相关文档

- 完整架构报告：[`ARCHITECTURE_REPORT.md`](ARCHITECTURE_REPORT.md)
- 当前 FFmpeg 双来源审查：[`FFMPEG_PROVIDER_REVIEW.md`](FFMPEG_PROVIDER_REVIEW.md)
- 自构建 FFmpeg 评估（已暂缓）：[`FFMPEG_BUILD_REPORT.md`](FFMPEG_BUILD_REPORT.md)
- 依赖评估：[`LIBRARY_DECISIONS.md`](LIBRARY_DECISIONS.md)
- 文档索引：[`README.md`](README.md)

---

## 11. 本次审查修正

- 自构建 FFmpeg 从“当前方案”降级为“未来方案”；
- `bnqdzj/nonfree` 校正为 `BnqDzj/FFmpeg-Builds-nonfree`；
- BtbN 当前只提供 GPL/LGPL，nonfree 由 BnqDzj 提供；
- `build ID` 改为 `provider_fingerprint`；
- “FFmpeg 三件套也强制 v3”改为“仅本项目 Rust/C++ 强制 v3，FFmpeg 上游预构建不承诺”；
- “同 trio 构建内可复现”改为“同 provider_fingerprint 内可复现”；
- M0 从“固定源码 commit 自构建”改为“provider 检测 + release pin + 能力矩阵”；
- 公开发布资产必须排除 nonfree 构建。

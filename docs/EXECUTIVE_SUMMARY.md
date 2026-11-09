# 结论摘要

> 日期：2026-11-10  
> 状态：架构评估阶段，尚未开始实现  
> 完整报告：[`ARCHITECTURE_REPORT.md`](ARCHITECTURE_REPORT.md)  
> 依赖决策：[`LIBRARY_DECISIONS.md`](LIBRARY_DECISIONS.md)  
> 自构建 FFmpeg 评估：[`FFMPEG_BUILD_REPORT.md`](FFMPEG_BUILD_REPORT.md)

---

## 1. 一句话结论

`LiveArchiveEncoder-VideoDamage` 应采用：

> **Rust 控制面 + C++ AVX2 数据面 + LAE 共享的自构建 FFmpeg 三件套。**

- **Rust** 管：CLI、TOML 1.1、Recipe/Plan、FFmpeg 进程监督、调度、输出事务、Manifest、插件 C ABI。
- **C++** 管：AVX2/SSE4 扫描与改写内核、oneTBB 并行、buffer/批处理。
- **FFmpeg / ffprobe / ffplay** 管：解封装、封装、编解码、滤镜、探测、可选播放。

不是“Rust 重写 + C++ 再包一层 FFmpeg 命令”，也不是“用 libav 替代 FFmpeg CLI”。

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
  FFmpeg 三件套发现、版本校验、进程监督
  Manifest / 输出事务 / 插件 C ABI
  │ cxx
  ▼
C++ 数据面
  AVX2/SSE4 扫描与改写内核
  oneTBB 并行
  buffer / 批处理
  │
  ▼
FFmpeg / ffprobe / ffplay（LAE 自构建三件套，build ID 锁定）
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
| 媒体层 | 自构建 FFmpeg / ffprobe / ffplay 三件套；基于 `nonfree+gpl`、强制 `x86-64-v3`、build ID 锁定；共享 LAE 构建 |
| 配置 | TOML 1.1 only；Rust 单点解析；不向下兼容 |
| CPU | x86-64-v3 强制；AVX2/SSE4；不提供 scalar fallback |
| 确定性 | 码流级 D1；重编码决策确定；packet BSF D3；容器只承诺同 trio 构建内可复现 |

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

### FFmpeg 构建链

- fork `FFmpeg-Builds/FFmpeg-Builds`，固定 FFmpeg tag/commit 与全部依赖 commit；
- 使用 `nonfree+gpl` 私有构建，另备 `gpl` 公开构建；
- Linux 使用 Clang 23 + `x86-64-v3` + ThinLTO/PGO 评估；
- Windows 先用 MinGW-w64 基线，再评估 `clang-cl` + VS2026 SDK；VS2026 用于本项目本体；
- 每次构建生成 build ID、`buildconf.txt`、`configure.txt`、`sources.lock`、`capabilities.toml`、`licenses.toml`、`SHA256SUMS`；
- 不使用 upstream floating `latest`；
- 详见 [`FFMPEG_BUILD_REPORT.md`](FFMPEG_BUILD_REPORT.md)。

---

## 7. 确定性口径

| 操作 | 口径 |
|---|---|
| 码流级 damage / truncate | D1：同输入 + 同 plan + 同工具链 → 同输出 |
| frame-drop / visual-loss | 决策确定；编码字节依赖编码器 |
| visual-glitch | 滤镜图确定；像素/编码字节依赖编码器 |
| packet-loss / audio-loss | D3：FFmpeg BSF 内部随机数，同 seed + 同构建尽力复现 |
| 容器输出 | 同 FFmpeg 自构建 trio / build ID 内可复现；跨构建不承诺 |

manifest 必须记录：

- 输入/输出 hash；
- plan hash；
- FFmpeg 三件套版本、路径与 build ID；
- `buildconf` 关键开关与编译器版本；
- 编码器/muxer；
- CPU 基线；
- 验证结果。

---

## 8. 先做什么

### M0：自构建三件套 + TOML + 全流复制

0. 固定 FFmpeg / FFmpeg-Builds / 依赖 commit，生成 build ID 与能力矩阵。
1. FFmpeg 三件套发现、build ID / 版本一致性校验、能力探测。
2. TOML 1.1 schema + Rust 解析 + `deny_unknown_fields`。
3. CLI `probe` / `copy`。
4. C++ 内核骨架 + doctest + oneTBB。
5. 全流复制：4 流输入 → 4 流输出，逐流 hash 一致。
6. 输出事务 + manifest。
7. Windows/Linux 构建，C++23/C++26 双标准。

**M0 验收：**

- 4 流输入 4 流输出；
- 每流 hash 一致；
- 三件套版本不一致直接报错；
- 非预期 build ID / `buildconf` 直接报错；
- C++23/C++26 都能编译；
- AVX2 CPUID 检查生效；
- Ctrl-C 能清理临时文件。

M0 通过后再写 packet/frame/bitstream damage。

---

## 9. 主要风险

| 风险 | 对策 |
|---|---|
| FFmpeg 进程开销 | 批量合并、接受启动成本、控制 `-threads` |
| FFmpeg 版本变化 | 只用 LAE 自构建最新 trio；build ID + manifest 记录；不跨版本承诺 D1 |
| 自构建 FFmpeg 工具链 | upstream LTO Windows broken、Clang 全依赖切换、VS2026 与 MinGW 差异；固定 commit、分阶段、FATE/语料/性能门禁 |
| packet BSF 随机数 | 如实标注 D3 |
| 全流保留复杂 | `-map 0` + 逐流验证 + 真实语料 |
| HEVC/AV1 parser 复杂 | 先 H.264/HEVC，AV1 后置 |
| TOML 1.1 生态 | `toml 1.2.1+spec-1.1.0`；Rust 单点解析 |
| AVX2 强制 | 启动 CPUID 检查；文档明确 |
| 多调度器超订 | tokio/crossbeam/oneTBB/FFmpeg 线程配额 |
| 插件 ABI | 版本化 C ABI、opaque handle、catch |
| GPL/nonfree | nonfree 私有制品；gpl 公开构建；明确许可证与构建来源 |

---

## 10. 相关文档

- 完整架构报告：[`ARCHITECTURE_REPORT.md`](ARCHITECTURE_REPORT.md)
- 自构建 FFmpeg 评估：[`FFMPEG_BUILD_REPORT.md`](FFMPEG_BUILD_REPORT.md)
- 依赖评估：[`LIBRARY_DECISIONS.md`](LIBRARY_DECISIONS.md)
- 文档索引：[`README.md`](README.md)

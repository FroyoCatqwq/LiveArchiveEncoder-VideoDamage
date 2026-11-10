# 文档索引与权威顺序

> 项目：LiveArchiveEncoder-VideoDamage
> 状态：编码前第二轮核查完成，尚未开始实现
> 文档版本：Round 2 / 2026-11-10
> 说明：`ARCHITECTURE_REPORT.md` 与 `LIBRARY_DECISIONS.md` 保留 2026-11-09 编写日期；本轮审查日期为 2026-11-10。

---

## 0. 权威顺序

当文档之间出现口径差异时，按以下顺序裁决：

| 优先级 | 文档 | 权威范围 |
|---|---|---|
| 1 | [`PRE_CODING_ROUND2_CHECK.md`](PRE_CODING_ROUND2_CHECK.md) | 第二轮核查记录、修正清单、最终开工前检查结论 |
| 2 | [`PRE_CODING_FINAL_REVIEW.md`](PRE_CODING_FINAL_REVIEW.md) | Go/No-Go 基线、C1–C10 契约冻结清单、Phase 0–4 |
| 3 | [`WORK_BREAKDOWN.md`](WORK_BREAKDOWN.md) | 分工、交接物、准备步骤、质量门禁、M0 DoD |
| 4 | [`FFMPEG_PROVIDER_REVIEW.md`](FFMPEG_PROVIDER_REVIEW.md) | **仅 FFmpeg 契约**：BtbN 稳定版、trio.lock、Extension Pack 边界 |
| 5 | [`ARCHITECTURE_REPORT.md`](ARCHITECTURE_REPORT.md) | 架构基线；TOML / CLI / C ABI / 接口片段均为草案 |
| 6 | [`EXECUTIVE_SUMMARY.md`](EXECUTIVE_SUMMARY.md) | 派生结论摘要，不覆盖上位文档 |
| 7 | [`LIBRARY_DECISIONS.md`](LIBRARY_DECISIONS.md) | Rust/C++ 依赖决策 |
| 8 | [`FFMPEG_BUILD_REPORT.md`](FFMPEG_BUILD_REPORT.md) | 已暂缓；Extension Pack 未来方案，不是当前 core 依据 |
| 9 | `README.md` / `docs/README.md` | 项目入口与文档索引，不承担规则裁决 |

冲突规则：

- Go/No-Go、冻结清单、Phase 编号：以 `PRE_CODING_FINAL_REVIEW.md` 与第二轮核查为准；
- 分工、质量门禁、M0 DoD：以 `WORK_BREAKDOWN.md` 为准；
- FFmpeg 契约：以 `FFMPEG_PROVIDER_REVIEW.md` 为准；
- 架构背景：以 `ARCHITECTURE_REPORT.md` 为准，但接口片段均为草案；
- 自构建 / nonfree / 安装器：只在 Extension Pack 未来方案中讨论，不进入 core；
- 示例 schema、CLI、C ABI 都不能被视为已冻结。

---

## 1. 核心文档

| 文档 | 状态 | 内容 |
|---|---|---|
| [`PRE_CODING_ROUND2_CHECK.md`](PRE_CODING_ROUND2_CHECK.md) | **当前第二轮核查记录** | 文档审查、P1/P2 修正、权威顺序、最终开工前判定 |
| [`PRE_CODING_FINAL_REVIEW.md`](PRE_CODING_FINAL_REVIEW.md) | **Go/No-Go 基线（第一轮）** | 编码前最后检查、core/extension 边界、C1–C10、Phase 0–4 |
| [`WORK_BREAKDOWN.md`](WORK_BREAKDOWN.md) | **当前分工** | 6 个工作流、职责、交接物、准备步骤、质量门禁、M0 DoD |
| [`FFMPEG_PROVIDER_REVIEW.md`](FFMPEG_PROVIDER_REVIEW.md) | **当前 FFmpeg 契约（仅 FFmpeg）** | core 只消费 BtbN `n9.0` 稳定版；安装部署/nonfree/自构建转 Extension Pack |
| [`ARCHITECTURE_REPORT.md`](ARCHITECTURE_REPORT.md) | 架构基线 | 旧项目分析、目标架构、TOML、CPU、Rust/C++ 分工、并发、跨平台、测试、风险；接口片段为草案 |
| [`EXECUTIVE_SUMMARY.md`](EXECUTIVE_SUMMARY.md) | 结论摘要 | 派生摘要，不覆盖权威顺序 |
| [`LIBRARY_DECISIONS.md`](LIBRARY_DECISIONS.md) | 依赖决策 | Rust/C++ 库的版本、维护状态、采用/拒绝理由 |
| [`FFMPEG_BUILD_REPORT.md`](FFMPEG_BUILD_REPORT.md) | **已暂缓 / Extension Pack 未来方案** | 自构建 FFmpeg 技术评估，不作为当前 core 实施依据 |

---

## 2. 当前结论

```text
Rust 控制面
  CLI / TOML 1.1 / Recipe / Plan / Scheduler
  FFmpeg trio 发现、锁文件校验、进程监督
  Manifest / 输出事务 / 插件 C ABI
        │ cxx / C ABI
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

- 产品代码只 Rust + C++；
- Rust 1.99.1 / edition 2024；
- C++26 目标，C++23 下限；
- Windows + Linux；
- 仅 CLI；
- 独立 CLI + Rust crate + C ABI 插件；
- TOML 1.1 only；
- 本项目 Rust/C++ 强制 x86-64-v3；FFmpeg 上游预构建不承诺编译期 v3；
- Core 只消费已安装的 BtbN `n9.0` 稳定版 GPL 三件套；
- Core 不下载、不安装、不部署、不更新 FFmpeg；
- `run` / `verify` 默认要求 `ffmpeg-trio.lock`；
- 下载/安装/部署/更新、nonfree、自构建属于 Extension Pack；
- 自构建 FFmpeg 暂缓，暂不进入 core 路线。

---

## 3. 当前 Go/No-Go

允许：

- Phase 0：冻结 C1–C10；
- Phase 1：搭仓库骨架；
- 写契约测试；
- 手写 `ffmpeg-trio.lock` 示例；
- 写 `probe` / `copy`；
- 做全流复制。

不允许：

- 写 damage 算法；
- 把 FFmpeg 安装器写进 core；
- 支持 nonfree / 自构建；
- 无锁直接 `run`；
- 跳过 4 流门禁。

最终判定：

> **可以进入 Phase 0 与 Phase 1；damage 实现必须等 M0 全流复制门禁通过。**

---

## 4. 下一步顺序

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

详细判定见 [`PRE_CODING_ROUND2_CHECK.md`](PRE_CODING_ROUND2_CHECK.md)，分工见 [`WORK_BREAKDOWN.md`](WORK_BREAKDOWN.md)。

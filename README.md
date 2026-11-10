# LiveArchiveEncoder-VideoDamage

LiveArchiveEncoder v8 的实验性视频损伤扩展包。

**状态：编码前第二轮核查完成，尚未开始实现。**

当前 Go/No-Go：

- 允许：Phase 0 冻结契约、Phase 1 搭仓库骨架、`probe` / `copy`、全流复制；
- 禁止：damage 算法、FFmpeg 安装器写进 core、nonfree / 自构建、无锁 `run`；
- 硬门禁：M0 全流复制通过后才允许写 damage。

## 目标

- 可复现的视频故障注入：同输入 + 同计划 + 同工具链 → 同输出；
- 覆盖容器、packet、码流、帧、像素多个损伤层级；
- 默认保留所有音频/字幕/数据/附件/章节；
- 可作为独立 CLI 使用，也可作为 LAE v8 的 Rust crate 或 C ABI 插件；
- 仅 CLI，不做 CUI/GUI/TUI。

## 架构

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

## 硬约束

- 产品代码只使用 **Rust + C++**；
- Rust **1.99.1**，edition 2024；
- C++ 目标 **C++26**，最低 **C++23**；
- Windows x64 + Linux x86_64；
- 仅 CLI；
- 独立 CLI + Rust crate + C ABI 插件；
- TOML **1.1 only**；
- 本项目 Rust/C++ 强制 **x86-64-v3**（AVX2/SSE4），不提供 scalar fallback；FFmpeg 上游预构建不承诺编译期 v3；
- Core 只消费**已安装**的 `BtbN / FFmpeg-Builds` 最新稳定版 GPL 三件套，当前为 `n9.0`；
- Core 不下载、不安装、不部署、不更新 FFmpeg；
- `run` / `verify` 默认要求 `ffmpeg-trio.lock`；
- FFmpeg 下载、安装、部署、更新、nonfree、自构建全部属于 **Extension Pack**，不是 core。

## 文档

| 文档 | 角色 |
|---|---|
| [编码前第二轮检查核查报告](docs/PRE_CODING_ROUND2_CHECK.md) | 当前第二轮核查记录、修正清单、最终开工前判定 |
| [编码前 Go/No-Go 基线](docs/PRE_CODING_FINAL_REVIEW.md) | 第一轮最终审查、C1–C10 冻结清单、Phase 0–4 |
| [分工、准备步骤与质量门禁](docs/WORK_BREAKDOWN.md) | 6 个工作流、交接物、质量门禁、M0 DoD |
| [FFmpeg Core 契约（仅 FFmpeg）](docs/FFMPEG_PROVIDER_REVIEW.md) | BtbN 稳定版、trio.lock、Extension Pack 边界 |
| [架构基线](docs/ARCHITECTURE_REPORT.md) | 架构报告；TOML/CLI/C ABI 片段为草案 |
| [结论摘要](docs/EXECUTIVE_SUMMARY.md) | 派生摘要 |
| [依赖评估](docs/LIBRARY_DECISIONS.md) | Rust/C++ 依赖决策 |
| [自构建 FFmpeg 评估（已暂缓 / 未来方案）](docs/FFMPEG_BUILD_REPORT.md) | Extension Pack 未来方案，不是当前 core 依据 |
| [文档索引与权威顺序](docs/README.md) | 文档角色、冲突裁决、Go/No-Go、里程碑 |

文档权威顺序与冲突裁决规则见 [`docs/README.md`](docs/README.md)。

## 参考

- 旧项目：[LAE-Damager](https://codeberg.org/fr0yo0v0_qwq/LAE-Damager)
- 母项目：[LiveArchiveEncoder](https://github.com/FroyoCatqwq/LiveArchiveEncoder)
- FFmpeg 稳定构建：[FFmpeg-Builds/FFmpeg-Builds](https://github.com/FFmpeg-Builds/FFmpeg-Builds)

## 许可证

GPL-3.0-only，见 [LICENSE](LICENSE)。

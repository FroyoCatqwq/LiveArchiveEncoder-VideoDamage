# 文档索引

> 项目：LiveArchiveEncoder-VideoDamage  
> 状态：架构评估阶段，尚未开始实现  
> 日期：2026-11-10

---

## 核心文档

| 文档 | 内容 |
|---|---|
| [`EXECUTIVE_SUMMARY.md`](EXECUTIVE_SUMMARY.md) | 结论摘要：目标架构、硬约束、编码分层、依赖结论、双来源 FFmpeg、确定性口径、M0 计划 |
| [`ARCHITECTURE_REPORT.md`](ARCHITECTURE_REPORT.md) | 完整技术评估：旧项目分析、目标架构、FFmpeg 三件套、TOML 1.1、CPU 基线、Rust/C++ 分工、并发、跨平台、测试、风险、路线图 |
| [`FFMPEG_PROVIDER_REVIEW.md`](FFMPEG_PROVIDER_REVIEW.md) | **当前实施依据**：BtbN / BnqDzj 双来源预构建 FFmpeg 兼容审查、provider_fingerprint、能力矩阵、许可证与报告修正 |
| [`FFMPEG_BUILD_REPORT.md`](FFMPEG_BUILD_REPORT.md) | 自构建 FFmpeg 评估：**已暂缓 / 未来方案**，不作为当前实施依据 |
| [`LIBRARY_DECISIONS.md`](LIBRARY_DECISIONS.md) | 依赖评估：Rust/C++ 库的版本、维护状态、采用/拒绝理由、最终依赖集 |

---

## 当前结论

```text
Rust 控制面
  CLI / TOML 1.1 / Recipe / Plan / Scheduler
  FFmpeg provider 解析、版本/指纹校验、进程监督
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
（BtbN / BnqDzj 双来源预构建，provider_fingerprint 锁定）
```

- 产品代码只 Rust + C++；
- Rust 1.99.1 / edition 2024；
- C++26 目标，C++23 下限；
- Windows + Linux；
- 仅 CLI；
- 独立 CLI + Rust crate + C ABI 插件；
- TOML 1.1 only；
- 本项目 Rust/C++ 强制 x86-64-v3；FFmpeg 上游预构建不承诺编译期 v3；
- 当前支持 `btbn` 与 `bnqdzj-nonfree` 两个 provider；
- 自构建 FFmpeg 暂缓；
- floating `latest` 只作为更新入口，运行时必须 pin release tag + asset SHA-256；
- nonfree 二进制只进私有制品库，不进 public release。

---

## 下一步

先做 M0：

0. 实现 `FfmpegProvider` / `TrioDescriptor` / `provider_fingerprint`，建立 BtbN 与 BnqDzj 注册表；
1. 固定 provider 的 release tag、asset SHA-256 与能力矩阵；
2. FFmpeg 三件套发现、版本/`buildconf`/SHA-256 校验、能力探测；
3. TOML 1.1 schema + Rust 解析；
4. CLI `probe` / `copy`，增加 provider/release/variant 参数；
5. C++ 内核骨架 + doctest + oneTBB；
6. 全流复制：4 流输入 → 4 流输出，逐流 hash 一致；
7. 输出事务 + manifest；
8. Windows/Linux 构建，C++23/C++26 双标准。

当前双来源策略见 [`FFMPEG_PROVIDER_REVIEW.md`](FFMPEG_PROVIDER_REVIEW.md)；自构建方案见 [`FFMPEG_BUILD_REPORT.md`](FFMPEG_BUILD_REPORT.md)。

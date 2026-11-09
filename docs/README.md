# 文档索引

> 项目：LiveArchiveEncoder-VideoDamage  
> 状态：架构评估阶段，尚未开始实现  
> 日期：2026-11-09

---

## 核心文档

| 文档 | 内容 |
|---|---|
| [`EXECUTIVE_SUMMARY.md`](EXECUTIVE_SUMMARY.md) | 结论摘要：目标架构、硬约束、编码分层、依赖结论、确定性口径、M0 计划 |
| [`ARCHITECTURE_REPORT.md`](ARCHITECTURE_REPORT.md) | 完整技术评估：旧项目分析、目标架构、FFmpeg 三件套、TOML 1.1、CPU 基线、Rust/C++ 分工、并发、跨平台、测试、风险、路线图 |
| [`LIBRARY_DECISIONS.md`](LIBRARY_DECISIONS.md) | 依赖评估：Rust/C++ 库的版本、维护状态、采用/拒绝理由、最终依赖集 |

---

## 当前结论

```text
Rust 控制面
  CLI / TOML 1.1 / Recipe / Plan / Scheduler
  FFmpeg 三件套发现、版本校验、进程监督
  Manifest / 输出事务 / 插件 C ABI
        │ cxx / C ABI
        ▼
C++ 数据面
  AVX2/SSE4 扫描与改写内核
  oneTBB 并行
  buffer / 批处理
        │
        ▼
FFmpeg / ffprobe / ffplay（共享 LAE 三件套）
```

- 产品代码只 Rust + C++；
- Rust 1.99.1 / edition 2024；
- C++26 目标，C++23 下限；
- Windows + Linux；
- 仅 CLI；
- 独立 CLI + Rust crate + C ABI 插件；
- TOML 1.1 only；
- x86-64-v3 强制。

---

## 下一步

先做 M0：

1. FFmpeg 三件套发现、版本一致性校验、能力探测；
2. TOML 1.1 schema + Rust 解析；
3. CLI `probe` / `copy`；
4. C++ 内核骨架 + doctest + oneTBB；
5. 全流复制：4 流输入 → 4 流输出，逐流 hash 一致；
6. 输出事务 + manifest；
7. Windows/Linux 构建，C++23/C++26 双标准。

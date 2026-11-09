# LiveArchiveEncoder-VideoDamage

LiveArchiveEncoder v8 的实验性视频损伤扩展包。

**状态：架构评估阶段，尚未开始实现。**

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

## 硬约束

- 产品代码只使用 **Rust + C++**；
- Rust **1.99.1**，edition 2024；
- C++ 目标 **C++26**，最低 **C++23**；
- Windows x64 + Linux x86_64；
- 仅 CLI；
- 独立 CLI + Rust crate + C ABI 插件；
- TOML **1.1 only**；
- 强制 **x86-64-v3**（AVX2/SSE4），不提供 scalar fallback；
- 仅使用 LAE 共享的最新 FFmpeg 三件套，不兼容旧版本。

## 文档

- [结论摘要](docs/EXECUTIVE_SUMMARY.md)
- [完整架构报告](docs/ARCHITECTURE_REPORT.md)
- [依赖评估](docs/LIBRARY_DECISIONS.md)
- [文档索引](docs/README.md)

## 参考

- 旧项目：[LAE-Damager](https://codeberg.org/fr0yo0v0_qwq/LAE-Damager)
- 母项目：[LiveArchiveEncoder](https://github.com/FroyoCatqwq/LiveArchiveEncoder)

## 许可证

GPL-3.0-only，见 [LICENSE](LICENSE)。

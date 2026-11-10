# 编码前 Go/No-Go 基线（第一轮）

> 日期：2026-11-10
> 轮次：第一轮 Go/No-Go 基线；第二轮核查见 [`PRE_CODING_ROUND2_CHECK.md`](PRE_CODING_ROUND2_CHECK.md)
> 状态：**Go/No-Go 基线**
> 原则：质量优先，不以编码速度为目标
> 仓库：`LiveArchiveEncoder-VideoDamage`
> 基线提交：`2efbb83`（文档阶段；本轮文档修订尚未提交）
> 当前 FFmpeg 契约：[`FFMPEG_PROVIDER_REVIEW.md`](FFMPEG_PROVIDER_REVIEW.md)
> 分工与准备：[`WORK_BREAKDOWN.md`](WORK_BREAKDOWN.md)

---

## 1. 最终判定

当前项目**可以进入编码准备阶段，但不能直接开始写 damage 算法**。

```text
架构方向：通过
FFmpeg 边界：通过
依赖选择：通过
仓库骨架：未建立
接口/格式：草案存在，但 C1–C10 未冻结
测试基线：未建立
CI/构建：未建立

结论：Conditionally Ready for Scaffold
      Not Ready for Damage Implementation
```

允许开始：

- 冻结 C1–C10；
- 仓库骨架；
- 构建系统；
- 空 CLI；
- 空 C++ 内核库；
- `ffmpeg-trio.lock` 解析的契约测试；
- `probe` / `copy` 与全流复制。

不允许开始：

- packet / frame / bitstream damage 算法；
- 全量 codec 矩阵实现；
- 插件 ABI 的“边写边定”；
- FFmpeg 下载/安装/部署器；
- nonfree provider；
- 自构建 FFmpeg。

---

## 2. 当前仓库状态

截至基线提交 `2efbb83` 已跟踪文件：

```text
LICENSE
README.md
docs/ARCHITECTURE_REPORT.md
docs/EXECUTIVE_SUMMARY.md
docs/FFMPEG_BUILD_REPORT.md
docs/FFMPEG_PROVIDER_REVIEW.md
docs/LIBRARY_DECISIONS.md
docs/README.md
```

本轮新增、当前未提交：

```text
docs/PRE_CODING_FINAL_REVIEW.md
docs/WORK_BREAKDOWN.md
docs/PRE_CODING_ROUND2_CHECK.md
```

本轮修改、当前未提交：

```text
README.md
docs/README.md
docs/ARCHITECTURE_REPORT.md
docs/EXECUTIVE_SUMMARY.md
docs/FFMPEG_PROVIDER_REVIEW.md
docs/FFMPEG_BUILD_REPORT.md
docs/LIBRARY_DECISIONS.md
```

当前缺失的工程件：

- `Cargo.toml` / workspace；
- `rust-toolchain.toml`；
- `.cargo/config.toml`；
- C++ 源码与 `xmake.lua`；
- `conanfile.txt` / Conan profile；
- `xtask`；
- CI workflow；
- 测试夹具；
- 已冻结的 `ffmpeg-trio.lock` 样例；
- 已冻结的 TOML / Plan / Manifest / 错误码 / C ABI 规范；
- 基准与性能门禁。

结论：

> 目前仍是“文档完整、工程为零”的状态。  
> 文档质量足够开始搭骨架；但接口仍是草案，不能直接写业务算法。

---

## 3. Core / Extension 最终边界

### 3.1 Core 只做消费

Core 负责：

- 发现已安装的 `ffmpeg` / `ffprobe` / `ffplay`；
- 读取并校验 `ffmpeg-trio.lock`；
- 校验版本、`buildconf`、二进制 SHA-256；
- 探测并缓存能力；
- 编排 FFmpeg 进程；
- 生成 Manifest。

Core 不做：

- 不下载 FFmpeg；
- 不解压；
- 不安装；
- 不部署；
- 不更新；
- 不创建 PATH 软链接；
- 不解析远程 latest；
- 不处理 nonfree；
- 不自构建。

### 3.2 Extension Pack 负责安装部署

FFmpeg Provider Extension Pack 负责：

- 解析 BtbN `n9.0` 最新稳定版；
- 下载 GPL 资产；
- 校验 `checksums.sha256`；
- 解压、安装、部署；
- 生成 `ffmpeg-trio.lock`；
- 更新/回滚；
- 未来支持 nonfree / 自构建。

Extension Pack 与 core 只通过文件系统交互：

```text
trio_dir/ffmpeg
trio_dir/ffprobe
trio_dir/ffplay
trio_dir/ffmpeg-trio.lock
```

不链接进 core，不共享内部类型。

### 3.3 当前 core 支持的 FFmpeg

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

---

## 4. FFmpeg 契约摘要

### 4.1 锁文件

Core 默认读取：

```text
<ffmpeg-dir>/ffmpeg-trio.lock
```

锁文件必须记录：

- provider / repository；
- release_line / release_tag / published_at；
- channel / variant / target；
- asset / asset_sha256 / checksums_sha256；
- version；
- 三件套二进制 SHA-256；
- buildconf SHA-256；
- capabilities SHA-256；
- installed_at / installed_by。

`release_tag` 必须是具体 `autobuild-*`，不允许 `latest`。

### 4.2 校验

`run` / `verify` 默认要求锁文件：

- 三件套同目录、可执行；
- 版本为 `n9.0.x` 稳定版；
- `buildconf` 含 `--enable-gpl --enable-version3`；
- 不含 `--enable-nonfree`；
- 三件套 SHA-256 匹配锁文件；
- buildconf / capabilities 哈希匹配；
- provider / release_line / variant / target 匹配；
- 不匹配返回 `LAE-VD-E1004`。

`probe` / `capabilities` 可以无锁运行，但必须标记：

```text
provenance = "unverified"
```

`--allow-unlocked-ffmpeg` 仅开发诊断，不进入默认路径。

### 4.3 安装部署不在 core

```text
core M0 ≠ FFmpeg installer
core M0 = 消费已安装 trio + lock + 校验 + 进程调用
```

即使没有 Extension Pack，用户手工安装 BtbN `n9.0` GPL 三件套并生成锁文件，core 也应该能运行。

---

## 5. 编码前必须冻结的契约

以下 **C1–C10 是唯一冻结清单**。其他文档不得另列不同枚举，只引用本节。

| 编号 | 契约 | 必须回答的问题 |
|---|---|---|
| C1 | `ffmpeg-trio.lock` v1 | provider、release、asset、hash、target、variant 字段 |
| C2 | TOML config v1 | 必填字段、默认值、未知键、旧配置拒绝 |
| C3 | Plan v5 | 原始 artifact hash、stream map、stage、decision、toolchain fingerprint |
| C4 | Manifest v1 | 输入/输出、plan hash、trio fingerprint、逐流 hash、验证结果 |
| C5 | 错误码注册表 | `LAE-VD-E1xxx`…`E5xxx` 与 `E9xxx` 的稳定编号与退出码 |
| C6 | Plugin C ABI v1 | opaque handle、request/callback、取消、版本协商 |
| C7 | CLI 契约 | 子命令、参数、stdout/stderr、退出码、JSON 输出 |
| C8 | 仓库布局 | crate、C++、tests、xtask、extensions 边界 |
| C9 | 构建矩阵 | Windows MSVC 2026 / Linux Clang 23 / C++23+C++26 / v3 |
| C10 | 质量基线、许可证与第三方声明策略 | 4 流门禁、FATE 风格 smoke、真实语料、性能口径、许可证清单 |

> 本节及以下“最低要求”只是草案说明，不代表 C1–C10 已定稿。  
> 任何 `ARCHITECTURE_REPORT.md` / `FFMPEG_PROVIDER_REVIEW.md` 中的 schema、CLI、C ABI 片段也都不是已冻结接口。

### 5.1 Plan v5 最低要求（草案）

- 记录原始 artifact 的 hash 和长度，而不是只记录抽取后的 ES；
- 完整 stream map：index、type、codec、disposition、language、attached_pic；
- stage 列表与每 stage 的 selector；
- 每个 unit 的 decision / span / expected stats；
- toolchain fingerprint；
- plan hash / recipe hash；
- 尾部校验。

### 5.2 Manifest v1 最低要求（草案）

- 输入 / 输出 hash；
- 每条输出流的 hash；
- plan hash；
- FFmpeg provider_fingerprint；
- release tag / asset SHA-256 / 二进制 SHA-256；
- buildconf / capabilities hash；
- 编码器 / muxer；
- 项目 CPU 基线与 FFmpeg CPU 基线；
- 验证结果与 warnings；
- 输出事务状态。

### 5.3 错误码最低要求（草案）

```text
LAE-VD-E1xxx  FFmpeg / provider / 进程
LAE-VD-E2xxx  输入 / 探测 / Plan
LAE-VD-E3xxx  输出 / 事务 / 验证
LAE-VD-E4xxx  配置 / CLI / TOML
LAE-VD-E5xxx  插件 / ABI / 版本协商
LAE-VD-E9xxx  内部错误
```

必须先建注册表，再写代码；不允许各处手写字符串。

### 5.4 FATE 风格 smoke 定义

“FATE 风格 smoke”不是要求跑完整 FFmpeg FATE，而是由 WS4 定义的最小回归集：

- 对 BtbN 三件套做 decode / encode / mux / demux smoke；
- 覆盖 H.264、HEVC、AAC 与至少一个容器组合；
- 覆盖 4 流全流复制；
- 覆盖损坏容器输入；
- 记录 `ffmpeg -buildconf`、退出码、stderr 摘要与产物 hash；
- 每次更换 provider release 或锁文件后重跑。

---

## 6. 编码前检查清单

### Phase 0：契约冻结（不写业务代码）

- [ ] 按 §5 的 **C1–C10** 逐项冻结；
- [ ] 不允许使用其他冻结清单；
- [ ] 每个 schema 有示例；
- [ ] 每个错误码有说明；
- [ ] 每个接口有 Rust/C++ 两侧签名；
- [ ] 文档评审通过；
- [ ] 不再出现“边写边定”。

### Phase 1：仓库骨架

- [ ] Cargo workspace；
- [ ] `rust-toolchain.toml` = 1.99.1；
- [ ] `.cargo/config.toml` = x86-64-v3；
- [ ] 空 `lae-vd-core` / `lae-vd-cli` / `lae-vd-ffmpeg` / `lae-vd-plugin`；
- [ ] C++ `lae-vd-kernels` + `xmake.lua`；
- [ ] Conan clang-23/libc++ profile；
- [ ] `xtask`；
- [ ] 空 CLI 能在 Windows/Linux 编译运行。

### Phase 2：FFmpeg 基线

- [ ] 手工安装 BtbN `n9.0` GPL 三件套；
- [ ] 生成 `ffmpeg-trio.lock`；
- [ ] 记录 `ffmpeg -buildconf`；
- [ ] 生成 `capabilities.toml`；
- [ ] 校验三件套 SHA-256；
- [ ] `probe` 能输出 provider / version / lock 状态。

### Phase 3：测试与 CI

- [ ] 4 流夹具（H.264 + 2×AAC + SRT）；
- [ ] 长归档夹具；
- [ ] 传统/现代 codec 样本；
- [ ] 损坏容器样本；
- [ ] FATE 风格 smoke；
- [ ] `nextest` / `rstest` / `insta` / `loom`；
- [ ] C++ doctest；
- [ ] Windows/Linux CI；
- [ ] C++23/C++26 双标准；
- [ ] clippy / warnings-as-errors / sanitizer。

### Phase 4：M0

- [ ] CLI `probe` / `copy`；
- [ ] all-stream copy `-map 0`；
- [ ] 4 流进 4 流出，逐流 hash 一致；
- [ ] 输出事务 + manifest；
- [ ] provider / buildconf / SHA-256 错误路径；
- [ ] Ctrl-C 清理临时文件；
- [ ] M0 验收通过后才写 damage。

---

## 7. 最终质量红线

1. **契约先于实现**：C1–C10 不冻结，不写业务代码。
2. **全流保留**：默认 `-map 0`；字幕、多音轨、章节、附件不能静默丢。
3. **不静默降级**：provider、buildconf、能力、pix_fmt、HDR、10-bit 都要显式。
4. **确定性诚实**：D1 只在同 `provider_fingerprint` 内成立；packet BSF 标 D3。
5. **core 不含安装器**：下载/安装/部署/更新只在 Extension Pack。
6. **质量优先**：不以编码速度为目标；M0 门禁不过，不进入 damage。
7. **文档与代码同步**：接口变更必须同时改契约文档和测试。

---

## 8. Go / No-Go

### Go

- 进入 Phase 0：冻结 C1–C10；
- 进入 Phase 1：搭仓库骨架；
- 手工准备 BtbN `n9.0` GPL 三件套；
- 写 `probe` / `copy` / 全流复制。

### No-Go

- 直接写 damage 算法；
- 把 FFmpeg 下载/安装/部署写进 core；
- 支持 BnqDzj nonfree / 自构建；
- 没有锁文件就进入 `run`；
- 没有 4 流门禁就发布；
- 在 Windows/Linux 任一平台未编译通过就合并。

### 基线判定

> **可以开始搭骨架和冻结契约；damage 实现必须等 M0 全流复制门禁通过。**

第二轮核查结论见 [`PRE_CODING_ROUND2_CHECK.md`](PRE_CODING_ROUND2_CHECK.md)；若本文件与第二轮核查记录有表述差异，以第二轮核查记录中的修正说明与 `docs/README.md` 的权威顺序为准。

# 编码前第二轮检查核查报告

> 日期：2026-11-10
> 轮次：**第二轮 / 开工前最终核查**
> 状态：文档审查与修正完成，尚未开始实现，尚未提交
> 基线提交：`2efbb83`
> 范围：`README.md` 与 `docs/*.md` 全部 Markdown 文档
> 原则：质量优先，不以编码速度为目标

---

## 1. 本轮结论

第二轮检查没有发现 P0 级策略冲突，但发现并修正了 8 项 P1、11 项 P2 文档问题。

最终判定仍然是：

```text
Go：可以进入 Phase 0 契约冻结、Phase 1 仓库骨架
No-Go：不可以直接写 packet / frame / bitstream damage
硬门禁：M0 全流复制通过后才允许 damage 实现
```

也就是说：

- 架构方向、FFmpeg 边界、依赖选择、平台/语言/CPU/许可证口径没有互斥错误；
- 但文档之间曾存在“谁是权威”“哪些契约算冻结”“示例 schema 是否算定稿”“旧自构建报告是否用现在时”等歧义；
- 本轮已统一为一套权威顺序、一套冻结清单、一套里程碑语义，并给所有草案片段加了明确标注。

---

## 2. 文档权威顺序

为避免第三轮再次出现“多个文档都自称当前实施依据”的问题，现固定如下顺序。

| 优先级 | 文档 | 权威范围 |
|---|---|---|
| 1 | [`PRE_CODING_ROUND2_CHECK.md`](PRE_CODING_ROUND2_CHECK.md) | 第二轮核查记录、修正清单、最终开工前检查结论 |
| 2 | [`PRE_CODING_FINAL_REVIEW.md`](PRE_CODING_FINAL_REVIEW.md) | Go/No-Go 基线、C1–C10 契约冻结清单、Phase 0–4 |
| 3 | [`WORK_BREAKDOWN.md`](WORK_BREAKDOWN.md) | 分工、交接物、准备步骤、质量门禁、M0 DoD |
| 4 | [`FFMPEG_PROVIDER_REVIEW.md`](FFMPEG_PROVIDER_REVIEW.md) | **仅 FFmpeg 契约**：BtbN 稳定版、trio.lock、Extension Pack 边界 |
| 5 | [`ARCHITECTURE_REPORT.md`](ARCHITECTURE_REPORT.md) | 架构基线；其中 TOML / CLI / C ABI / 接口片段均为草案 |
| 6 | [`EXECUTIVE_SUMMARY.md`](EXECUTIVE_SUMMARY.md) | 派生结论摘要，不覆盖上位文档 |
| 7 | [`LIBRARY_DECISIONS.md`](LIBRARY_DECISIONS.md) | Rust/C++ 依赖决策 |
| 8 | [`FFMPEG_BUILD_REPORT.md`](FFMPEG_BUILD_REPORT.md) | 已暂缓；Extension Pack 未来方案，不是当前 core 依据 |
| 9 | `README.md` / `docs/README.md` | 项目入口与文档索引，不承担规则裁决 |

冲突裁决规则：

- Go/No-Go、冻结清单、Phase 编号：以 `PRE_CODING_FINAL_REVIEW.md` 与本第二轮报告为准；
- 分工、质量门禁、M0 DoD：以 `WORK_BREAKDOWN.md` 为准；
- FFmpeg 契约：以 `FFMPEG_PROVIDER_REVIEW.md` 为准；
- 架构背景：以 `ARCHITECTURE_REPORT.md` 为准，但接口片段均为草案；
- 自构建 / nonfree / 安装器：只在 Extension Pack 未来方案中讨论，不进入 core；
- 任何文档中的示例 schema、CLI、C ABI 都不能被视为已冻结。

---

## 3. 当前 FFmpeg 契约复核

| 检查项 | 结论 |
|---|---|
| core 只消费已安装的 BtbN `n9.0` 最新稳定版 GPL 三件套 | 一致 |
| core 不下载、不解压、不安装、不部署、不更新 FFmpeg | 一致 |
| 下载/安装/部署/更新/锁文件生成归 Extension Pack | 一致 |
| BnqDzj nonfree / 自构建仅 Extension Pack 未来能力 | 一致 |
| `run` / `verify` 默认要求 `ffmpeg-trio.lock` | 一致 |
| `probe` / `capabilities` 无锁可运行但标记 `provenance=unverified` | 一致 |
| core 不接受 `master-latest` / `n9.1-dev` / `n8.1` / `nonfree` / `lgpl` / 非 BtbN 构建 | 一致 |
| 本项目 Rust/C++ 强制 x86-64-v3；FFmpeg 预构建不承诺编译期 v3 | 一致 |

已核对 BtbN 当前 `latest` release：

- 2026-11-09 13:03 UTC；
- 49 个资产；
- `n9.0` 稳定分支 GPL / LGPL / shared 资产存在；
- `nonfree` 资产数量为 0；最近 37 个 release 中 nonfree 资产数量也为 0；
- 当前 FFmpeg 稳定标签包含 `n9.0.2`。

---

## 4. 冻结契约的唯一清单

唯一冻结清单为 `PRE_CODING_FINAL_REVIEW.md` §5 的 **C1–C10**：

| 编号 | 契约 |
|---|---|
| C1 | `ffmpeg-trio.lock` v1 |
| C2 | TOML config v1 |
| C3 | Plan v5 |
| C4 | Manifest v1 |
| C5 | 错误码注册表（含 `E1xxx`–`E5xxx` 与 `E9xxx`） |
| C6 | Plugin C ABI v1 |
| C7 | CLI 契约 |
| C8 | 仓库布局 |
| C9 | 构建矩阵 |
| C10 | 质量基线、许可证与第三方声明策略 |

其他文档不再另列不同枚举，只引用 C1–C10。

本轮修正：

- `PRE_CODING_FINAL_REVIEW.md` 的 C5 补入 `E9xxx`；
- C10 补入许可证与第三方声明策略；
- `WORK_BREAKDOWN.md` Phase 0 改为引用 C1–C10；
- `FFMPEG_PROVIDER_REVIEW.md` F10 改为引用 C1–C10；
- `ARCHITECTURE_REPORT.md` 中所有 TOML / CLI / C ABI 片段标注为草案。

---

## 5. 第二轮发现与修正

### P1：已在编码前修正

| 编号 | 问题 | 修正 |
|---|---|---|
| P1-1 | 多篇文档自称“当前实施依据”，且 1 篇/2 篇口径不一 | 在 `docs/README.md` 新增权威顺序；限定 `FFMPEG_PROVIDER_REVIEW.md` 为“当前 FFmpeg 契约依据” |
| P1-2 | 冻结契约存在 C1–C10 / 9 项 / 6 项三套枚举 | 统一为 C1–C10；其他文档改为引用 |
| P1-3 | “接口/格式未冻结”与文档已有 schema/CLI/C ABI 片段冲突 | 架构报告顶部与相关章节标注“草案，未冻结，不代表 C1–C7 定稿” |
| P1-4 | 架构报告仍写 FFmpeg trio 为 MSVC/Clang 构建 | 改为 BtbN 上游预构建；core 不校验 FFmpeg 本体编译器，只校验 version/buildconf/SHA-256 |
| P1-5 | 自构建暂缓报告 §5.6 / §9.3 / §9.4 / §11 仍以现在时描述当前门禁 | 加“未来方案”标题与局部提示；§12 链接改为当前 FFmpeg 契约、Go/No-Go 与第二轮报告 |
| P1-6 | `PRE_CODING_FINAL_REVIEW.md` 当前文件清单漏新增文档 | 改为“截至 `2efbb83` 已跟踪文件”并列出本轮新增未提交文件 |
| P1-7 | M0 语义三套并存，M0 后 damage 顺序不一致 | 统一 Phase 0–4 + canonical M0；M1→M4 顺序以 `WORK_BREAKDOWN.md` 与架构报告为准；自构建报告 M0–M5 标为 Extension Pack 未来路线 |
| P1-8 | CLI 缺 `copy`、解析顺序/缓存 key/骨架 crate 列表不一致 | 补 `copy`；同步解析顺序；缓存 key 补 `release_line`；骨架统一为 4 个 crate |

### P2：已同步修正

- 顶层 `README.md` 增加当前 Go/No-Go；
- 顶层与索引增加“第二轮检查”入口；
- `FFMPEG_PROVIDER_REVIEW.md` 状态改为“当前 FFmpeg 契约依据（仅 FFmpeg）”；
- `ffmpeg-trio.lock` 样例 `installed_by` 改为 `manual`，并说明 Extension Pack 上线后可改为扩展包标识；
- 错误码范围补 `E9xxx`；
- 质量基线补“FATE 风格 smoke”定义；
- `README.md` 文档链接改为带角色的表；
- `EXECUTIVE_SUMMARY.md` / `LIBRARY_DECISIONS.md` 的“文档索引”链接文字改为 `docs/README.md`；
- 自构建报告末尾改为“仅记录未来自构建方案”；
- `docs/README.md` 补“FFmpeg 预构建不承诺 v3”；
- `docs/README.md` 增加文档版本/审查轮次说明，兼容旧文档日期。

---

## 6. 草案与冻结边界

以下内容在第二轮检查中明确为**草案**：

- `ARCHITECTURE_REPORT.md` §7.2 TOML schema；
- `ARCHITECTURE_REPORT.md` §12 CLI 命令与参数；
- `ARCHITECTURE_REPORT.md` §13 C ABI；
- `FFMPEG_PROVIDER_REVIEW.md` §3.2 `ffmpeg-trio.lock` schema；
- `PRE_CODING_FINAL_REVIEW.md` §5.1/5.2/5.3 的 Plan/Manifest/错误码最低要求。

它们的作用是帮助讨论，**不是已冻结接口**。  
正式编码前必须在 C1–C10 中逐项定稿，并同步到测试与文档。

---

## 7. 里程碑语义

唯一开工序列：

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

说明：

- `FFMPEG_BUILD_REPORT.md` 中的 M0–M5 是 **Extension Pack 自构建未来路线**，不是 core 的 Phase/M0；
- `ARCHITECTURE_REPORT.md` 的 M0–M4 是架构功能里程碑，不覆盖 `WORK_BREAKDOWN.md` 的 Phase 0–4；
- M0 未通过，不写 damage；
- 任何“先跑起来再补契约”的做法都视为质量门禁失败。

---

## 8. 最终核查结果

| 项 | 结果 |
|---|---|
| P0 策略冲突 | 无 |
| P1 文档问题 | 已修正 |
| P2 文档同步 | 已修正 |
| 当前 FFmpeg 策略 | 一致 |
| Core / Extension 边界 | 一致 |
| Go/No-Go | 一致 |
| C1–C10 冻结清单 | 唯一 |
| 草案/冻结状态 | 已标注 |
| 里程碑语义 | 已统一 |
| Markdown 相对链接 | 检查通过 |
| Linter 诊断 | 0 |
| `git diff --check` | 通过 |
| 代码实现 | 未开始 |
| Git 状态 | 文档改动未提交 |

---

## 9. 第二轮最终判定

> **可以进入 Phase 0 与 Phase 1；不可以直接进入 damage 实现。**

允许：

- 写/冻结 C1–C10；
- 搭 Cargo / C++ / xmake / Conan 骨架；
- 写空 CLI、空内核、契约测试；
- 手工准备 BtbN `n9.0` GPL 三件套与 `ffmpeg-trio.lock`；
- 做 `probe` / `copy` / 全流复制。

禁止：

- 写 packet / frame / bitstream damage；
- 把 FFmpeg 安装器写进 core；
- 让 floating `latest` 进入运行时基线；
- 无锁 `run`；
- 支持 nonfree / 自构建；
- 跳过 4 流门禁。

最终硬门禁：

```text
M0 全流复制通过前，不写 damage。
```

---

## 10. 相关文档

- Go/No-Go 基线：[`PRE_CODING_FINAL_REVIEW.md`](PRE_CODING_FINAL_REVIEW.md)
- 分工与准备：[`WORK_BREAKDOWN.md`](WORK_BREAKDOWN.md)
- 当前 FFmpeg 契约：[`FFMPEG_PROVIDER_REVIEW.md`](FFMPEG_PROVIDER_REVIEW.md)
- 架构基线：[`ARCHITECTURE_REPORT.md`](ARCHITECTURE_REPORT.md)
- 结论摘要：[`EXECUTIVE_SUMMARY.md`](EXECUTIVE_SUMMARY.md)
- 文档索引与权威顺序：[`README.md`](README.md)

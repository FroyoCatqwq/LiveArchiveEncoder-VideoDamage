# FFmpeg Core 契约与 Provider 扩展包边界

> 日期：2026-11-10
> 状态：**当前 FFmpeg 契约依据（编码前冻结）**
> 权威范围：仅 FFmpeg 契约；全局 Go/No-Go 见 [`PRE_CODING_FINAL_REVIEW.md`](PRE_CODING_FINAL_REVIEW.md)，第二轮核查见 [`PRE_CODING_ROUND2_CHECK.md`](PRE_CODING_ROUND2_CHECK.md)，权威顺序见 [`README.md`](README.md)。
> 核心策略：VideoDamage core 只消费**已安装**的 `BtbN / FFmpeg-Builds` 最新稳定版三件套，不下载、不安装、不部署。
> 下载、安装、部署、更新、nonfree、自构建全部转入 **FFmpeg Provider Extension Pack**。
> 相关文档：[`WORK_BREAKDOWN.md`](WORK_BREAKDOWN.md)、[`ARCHITECTURE_REPORT.md`](ARCHITECTURE_REPORT.md)、[`EXECUTIVE_SUMMARY.md`](EXECUTIVE_SUMMARY.md)

---

## 1. 当前决策

### 1.1 Core 的边界

VideoDamage core **只做消费侧**：

- 发现已安装的 `ffmpeg` / `ffprobe` / `ffplay`；
- 读取并校验 `ffmpeg-trio.lock`；
- 校验三件套版本、`buildconf`、二进制 SHA-256；
- 探测并缓存能力；
- 编排 FFmpeg 进程；
- 记录 Manifest。

Core **不做**：

- 不下载 FFmpeg；
- 不解压 FFmpeg；
- 不安装 FFmpeg；
- 不创建/修改 PATH 软链接；
- 不检查远程最新版本；
- 不自动更新；
- 不处理 nonfree provider；
- 不自构建 FFmpeg。

### 1.2 Extension Pack 的边界

FFmpeg Provider Extension Pack 负责：

- 解析当前 `BtbN / FFmpeg-Builds` 最新稳定版；
- 下载 `n9.0-latest` GPL 资产（或未来其他受支持渠道）；
- 校验 `checksums.sha256`；
- 解压、安装、部署到 LAE 共享目录；
- 生成 `ffmpeg-trio.lock`；
- 更新/回滚；
- 未来可扩展：
  - `BnqDzj / FFmpeg-Builds-nonfree`；
  - 自构建 FFmpeg；
  - 其他 provider。

Extension Pack **不链接进 core**，不把网络/解压/安装依赖带进 core 二进制。

### 1.3 当前唯一支持的 FFmpeg

```text
provider       = "btbn"
repository     = "FFmpeg-Builds/FFmpeg-Builds"
release_line   = "n9.0"
channel        = "stable"
variant        = "gpl"
target         = "win64" | "linux64"
```

当前只接受：

- `FFmpeg 9.0.x` 稳定版；
- GPL 变体；
- 静态构建优先；
- 三件套完整；
- 非 `-dev`、非 `master`、非 `n9.1-dev`；
- 不含 `--enable-nonfree`。

`n8.1`、`master-latest`、`nonfree` 当前都不进入 core 支持矩阵。

---

## 2. BtbN 最新稳定版事实

调研时间：2026-11-10。

- 仓库：`FFmpeg-Builds/FFmpeg-Builds`；`BtbN/FFmpeg-Builds` 指向同一仓库。
- 最新 `latest` release：2026-11-09 13:03 UTC，`target_commitish=master`。
- 当前 `latest` release 有 49 个资产。
- `n9.0` 稳定分支资产包括：
  - `ffmpeg-n9.0-latest-win64-gpl-9.0.zip`
  - `ffmpeg-n9.0-latest-win64-gpl-shared-9.0.zip`
  - `ffmpeg-n9.0-latest-win64-lgpl-9.0.zip`
  - `ffmpeg-n9.0-latest-win64-lgpl-shared-9.0.zip`
  - `ffmpeg-n9.0-latest-linux64-gpl-9.0.tar.xz`
  - `ffmpeg-n9.0-latest-linux64-gpl-shared-9.0.tar.xz`
  - `ffmpeg-n9.0-latest-linux64-lgpl-9.0.tar.xz`
  - `ffmpeg-n9.0-latest-linux64-lgpl-shared-9.0.tar.xz`
- 当前 FFmpeg 稳定标签包含 `n9.0.2`。
- `latest` release 中 `nonfree` 资产数量为 0；最近 37 个 release 中 nonfree 资产数量也为 0。
- 因此 core 不能从 BtbN 获得 nonfree；nonfree 属于 Extension Pack 的未来能力。

### 2.1 “最新稳定版”的准确定义

Core 不读取 `latest` URL，也不自己解析远程版本。Extension Pack 负责：

1. 在 `n9.0` 稳定分支中解析当前最新稳定资产；
2. 将浮动的 `latest` 解析为具体 `autobuild-YYYY-MM-DD-HH-MM` release；
3. 下载并校验资产；
4. 生成锁文件。

Core 只读取锁文件中的具体版本与哈希：

```text
ffmpeg-n9.0-latest-win64-gpl-9.0.zip
```

不是 `master-latest`，不是 `n9.1-dev`，不是 `n8.1`。

---

## 3. Core 的三件套契约

### 3.1 解析顺序

```text
--ffmpeg-dir
  → LAE_FFMPEG_DIR
  → LAE v8 配置中的 FFmpeg 目录
  → 出错
```

不静默回退 PATH；不自动猜测安装位置；不自动下载。

`--ffmpeg-lock <PATH>` 可显式指定锁文件；默认查找：

```text
<ffmpeg-dir>/ffmpeg-trio.lock
```

### 3.2 `ffmpeg-trio.lock` Schema（草案，未冻结）

> 以下为 C1 草案示例，不代表 `ffmpeg-trio.lock` v1 已定稿。冻结前必须以 [`PRE_CODING_FINAL_REVIEW.md`](PRE_CODING_FINAL_REVIEW.md) §5 的 C1 为准。

TOML 1.1 only：

```toml
schema = "lae.ffmpeg-trio-lock/v1"

[provider]
name = "btbn"
repository = "FFmpeg-Builds/FFmpeg-Builds"
release_line = "n9.0"
release_tag = "autobuild-2026-11-09-12-42"
published_at = "2026-11-09T12:43:45Z"
channel = "stable"
variant = "gpl"
target = "win64"
asset = "ffmpeg-n9.0-latest-win64-gpl-9.0.zip"
asset_sha256 = "..."
checksums_sha256 = "..."
version = "n9.0.2"

[binaries]
ffmpeg_sha256 = "..."
ffprobe_sha256 = "..."
ffplay_sha256 = "..."
buildconf_sha256 = "..."
capabilities_sha256 = "..."

[install]
installed_at = "2026-11-10T00:00:00Z"
installed_by = "manual"
```

当前 core M0 允许手工安装并生成锁文件；Extension Pack 上线后，`installed_by` 可改为扩展包标识。

字段要求：

- `provider.name` 当前只能是 `btbn`；
- `release_line` 当前只能是 `n9.0`；
- `variant` 当前只能是 `gpl`；
- `target` 必须匹配当前主机；
- `release_tag` 必须是具体 autobuild tag，不允许 `latest`；
- `version` 必须是 `n9.0.x`，不含 `-dev`；
- 所有 SHA-256 必须存在且匹配；
- 未识别字段拒绝。

### 3.3 启动校验

`run` / `verify` 默认要求存在有效锁文件：

1. 三个二进制存在且可执行；
2. 同一目录；
3. `ffmpeg -version` / `ffprobe -version` / `ffplay -version` 核心版本一致；
4. 版本匹配 `n9.0.x` 稳定版；
5. `ffmpeg -buildconf` 包含：
   - `--enable-gpl`
   - `--enable-version3`
   - 不含 `--enable-nonfree`
6. `ffmpeg` / `ffprobe` / `ffplay` 本地 SHA-256 匹配锁文件；
7. `buildconf_sha256` 匹配；
8. 锁文件 `provider.name` / `release_line` / `variant` / `target` 匹配；
9. 否则返回 `LAE-VD-E1004`。

`probe` / `capabilities` 可以在缺少锁文件时运行，但必须输出：

```text
provenance = "unverified"
```

并明确警告；不得静默当成可复现基线。

开发期可显式：

```text
--allow-unlocked-ffmpeg
```

只用于诊断，不进入默认路径。

### 3.4 能力探测

```text
ffmpeg -version
ffmpeg -buildconf
ffmpeg -decoders
ffmpeg -encoders
ffmpeg -filters
ffmpeg -bsfs
ffmpeg -hwaccels
ffmpeg -muxers
ffmpeg -demuxers
ffmpeg -protocols
ffprobe -version
ffplay -version
```

能力缓存：

```text
%LOCALAPPDATA%/LAE/VideoDamage/caps/<provider_fingerprint>.toml
```

缓存 key：

```text
provider + release_line + release_tag + asset_sha256 + binary_sha256 + buildconf_hash + target + variant
```

`provider_fingerprint` 由锁文件关键字段和本地二进制哈希共同生成；不是远程版本号，也不是自构建 build ID。

---

## 4. Extension Pack 设计

### 4.1 职责

```text
lae-ffmpeg-provider-extension
  ├─ resolve latest stable (n9.0)
  ├─ download asset
  ├─ verify checksums.sha256
  ├─ extract
  ├─ install to trio_dir
  ├─ write ffmpeg-trio.lock
  ├─ update / rollback
  └─ future: bnqdzj-nonfree / self-build
```

### 4.2 与 Core 的接口

Extension Pack 只通过文件系统与 core 交互：

- `trio_dir/ffmpeg`；
- `trio_dir/ffprobe`；
- `trio_dir/ffplay`；
- `trio_dir/ffmpeg-trio.lock`。

Core 不 import Extension Pack；Extension Pack 不依赖 core 的内部类型。

### 4.3 当前 core 不做 / Extension Pack 后续实现

> 本节指 core 与 core M0 的范围；这些能力由 §4.1 的 Extension Pack 在后续实现，不是“永久不做”。

当前 core 不实现：

- 下载器；
- 解压器；
- 安装器；
- 更新器；
- PATH 管理；
- nonfree provider；
- 自构建。

这些都属于 **编码后、M0 之后的独立扩展包工作**，不是 core M0 的前置条件。

---

## 5. 确定性、CPU 与许可证

### 5.1 确定性

- D1 只在同一 `provider_fingerprint` 内成立；
- 跨 FFmpeg 版本、跨资产、跨 buildconf 不承诺逐字节一致；
- `packet-loss` 依赖 FFmpeg BSF 内部随机数，仍标 D3；
- 容器输出只承诺同 provider + release + asset hash + buildconf 内可复现。

### 5.2 CPU

- 本项目 Rust/C++ 强制 x86-64-v3；
- BtbN 预构建 FFmpeg 不承诺编译期 `x86-64-v3`；
- FFmpeg 的 SIMD 能力以 `runtime-cpudetect`、`buildconf` 和运行时探测为准；
- Manifest 分开记录：
  - `project_cpu_baseline = "x86-64-v3"`；
  - `ffmpeg_cpu_baseline = "btbn-prebuilt/runtime-cpudetect"`。

### 5.3 许可证

- BtbN GPL 构建按 GPL 体系合规使用；
- 当前 core 只接受 GPL 变体；
- nonfree 不属于 core 支持矩阵；
- Extension Pack 未来引入 nonfree 时，二进制不得进入 public release；
- LICENSES / provenance 必须记录 provider、release tag、资产名、哈希。

---

## 6. 对旧报告的修正

| 旧表述 | 新表述 |
|---|---|
| Core 支持 BtbN + BnqDzj 双来源 | Core 只支持 BtbN `n9.0` stable GPL；BnqDzj 属 Extension Pack 未来能力 |
| Core 负责下载/安装/部署 | Core 不下载、不安装、不部署；Extension Pack 负责 |
| floating `latest` 可作更新入口 | Extension Pack 解析 latest 为具体 autobuild tag；core 只读锁文件 |
| `provider_fingerprint` 从远程 release 推导 | Core 从锁文件 + 本地二进制哈希生成 |
| FFmpeg 三件套也强制 v3 | 只有本项目 Rust/C++ 强制 v3；FFmpeg 预构建不承诺 |
| M0 先做 provider registry | M0 只做锁文件消费 + 校验；安装器属扩展包 |

---

## 7. 最终决策

| 编号 | 决策 |
|---|---|
| F1 | Core 只消费已安装的 BtbN `n9.0` 最新稳定版 GPL 三件套 |
| F2 | Core 不下载、不安装、不部署、不更新 FFmpeg |
| F3 | Core 要求 `ffmpeg-trio.lock`；`run`/`verify` 默认强制 |
| F4 | Extension Pack 负责下载/安装/部署/更新，未来支持 nonfree/self-build |
| F5 | Core 只接受 `provider=btbn`、`release_line=n9.0`、`variant=gpl` |
| F6 | `probe` 可无锁诊断，但必须标记 `provenance=unverified` |
| F7 | 能力缓存按 `provider_fingerprint` |
| F8 | 本项目 Rust/C++ 强制 x86-64-v3；FFmpeg 预构建不承诺 |
| F9 | nonfree 不进入 core，也不进入 public release |
| F10 | 编码前先冻结 [`PRE_CODING_FINAL_REVIEW.md`](PRE_CODING_FINAL_REVIEW.md) §5 的 C1–C10；其中 C1 的 FFmpeg 契约以本文为准 |

---

## 8. 结论

当前 core 的 FFmpeg 策略已经收敛为：

> **外部安装、内部校验、锁文件消费、进程调用。**

这样做的收益：

- core 不背网络/解压/安装/平台分发逻辑；
- core 可以完全离线运行；
- 安装器可以独立更新、回滚、扩展；
- core 的确定性边界由 `ffmpeg-trio.lock` 明确；
- 未来 nonfree / 自构建只需扩展包演进，不污染 core。

代价：

- 首次使用必须先由用户或 Extension Pack 准备三件套；
- 没有锁文件时，`run` 默认拒绝；
- core 的 FFmpeg 版本更新节奏由 Extension Pack 控制，而不是 core 自动更新。

这是质量优先、边界清晰的选择。

本文只裁决 FFmpeg 契约；全局 Go/No-Go 以 [`PRE_CODING_FINAL_REVIEW.md`](PRE_CODING_FINAL_REVIEW.md) 为准，第二轮修正记录见 [`PRE_CODING_ROUND2_CHECK.md`](PRE_CODING_ROUND2_CHECK.md)，冲突裁决顺序见 [`README.md`](README.md)。

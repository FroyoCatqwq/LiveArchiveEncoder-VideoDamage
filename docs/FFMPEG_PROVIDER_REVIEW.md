# FFmpeg 双来源兼容与报告审查

> 日期：2026-11-10  
> 状态：**当前实施依据**  
> 变更：自构建 FFmpeg 暂缓；当前统一支持 `BtbN / FFmpeg-Builds` 与 `BnqDzj / FFmpeg-Builds-nonfree` 两家预构建三件套  
> 相关文档：[`ARCHITECTURE_REPORT.md`](ARCHITECTURE_REPORT.md)、[`EXECUTIVE_SUMMARY.md`](EXECUTIVE_SUMMARY.md)、[`FFMPEG_BUILD_REPORT.md`](FFMPEG_BUILD_REPORT.md)

---

## 1. 结论

当前阶段的 FFmpeg 策略调整为：

> **不自构建、不 fork、不固定源码 commit；直接使用两家上游预构建三件套，通过 provider 抽象统一接入。**

当前支持：

| 简称 | 实际仓库 | 当前角色 |
|---|---|---|
| `btbn` | `FFmpeg-Builds/FFmpeg-Builds`（`BtbN/FFmpeg-Builds` 指向同一仓库） | GPL / LGPL 预构建，适合公开/合规场景 |
| `bnqdzj-nonfree` | `BnqDzj/FFmpeg-Builds-nonfree` | nonfree 预构建，适合 LAE/VideoDamage 内部与私有制品 |

用户所说的 `bnqdzj/nonfree` 不是 GitHub 仓库路径；`bnqdzj/nonfree` 返回 404。实际公开仓库是：

```text
https://github.com/BnqDzj/FFmpeg-Builds-nonfree
```

自构建方案 [`FFMPEG_BUILD_REPORT.md`](FFMPEG_BUILD_REPORT.md) 保留为**未来方案**，不再作为当前实施依据。

---

## 2. 上游调研事实

### 2.1 BtbN / FFmpeg-Builds

- 仓库：`FFmpeg-Builds/FFmpeg-Builds`，默认分支 `master`。
- `BtbN/FFmpeg-Builds` 重定向到同一仓库。
- `latest` release：2026-11-09 13:03 UTC，`target_commitish=master`。
- 当前 `latest` release 有 49 个资产。
- 主要资产：
  - `ffmpeg-master-latest-win64-gpl.zip`
  - `ffmpeg-master-latest-win64-gpl-shared.zip`
  - `ffmpeg-master-latest-linux64-gpl.tar.xz`
  - `ffmpeg-master-latest-linux64-gpl-shared.tar.xz`
  - `ffmpeg-n9.0-latest-win64-gpl-9.0.zip`
  - `ffmpeg-n9.0-latest-linux64-gpl-9.0.tar.xz`
  - `lgpl` / `lgpl-shared` 对应资产
  - `checksums.sha256`
- 最近 37 个 release 的资产中，`nonfree` 资产数量为 0。
- 结论：BtbN 当前只提供 `gpl` / `lgpl`，不提供 `nonfree` 资产；README 中仍描述 nonfree 变体，但当前 release 不发布对应二进制。

### 2.2 BnqDzj / FFmpeg-Builds-nonfree

- 仓库：`BnqDzj/FFmpeg-Builds-nonfree`。
- 是 `FFmpeg-Builds/FFmpeg-Builds` 的 fork，脚本许可证 MIT。
- `latest` release：2026-11-08 18:03 UTC，名称 `Latest Auto-Build (2026-11-08 17:59)`。
- 当前 `latest` release 有 9 个资产：
  - `checksums.sha256`
  - `ffmpeg-master-latest-win64-nonfree.zip`
  - `ffmpeg-master-latest-win64-nonfree-shared.zip`
  - `ffmpeg-master-latest-linux64-nonfree.tar.xz`
  - `ffmpeg-master-latest-linux64-nonfree-shared.tar.xz`
  - `winarm64` / `linuxarm64` 对应资产
- README 明确：`nonfree` = GPL 全部依赖 + `fdk-aac`。
- 结论：BtbN 缺失的 `nonfree` 能力由 BnqDzj 提供；其二进制不可再分发。

### 2.3 双来源差异

| 维度 | BtbN | BnqDzj nonfree |
|---|---|---|
| 当前资产 | GPL / LGPL | nonfree / nonfree-shared |
| 发布频率 | 每日 autobuild + floating `latest` | 每日 autobuild + floating `latest` |
| 最新发布 | 2026-11-09 | 2026-11-08 |
| 目标平台 | win64/linux64/winarm64/linuxarm64 | win64/linux64/winarm64/linuxarm64 |
| 本项目目标 | win64 / linux64 | win64 / linux64 |
| 许可证 | GPL/LGPL 体系 | nonfree，二进制不可分发 |
| `checksums.sha256` | 有 | 有 |
| `shared` 变体 | 有 | 有 |
| x86-64-v3 编译基线 | 上游未承诺 | 上游未承诺 |

---

## 3. “统一支持”的设计

### 3.1 Provider 抽象

Rust 控制面引入：

```text
FfmpegProvider
  ├─ Btbn
  └─ BnqDzjNonfree
```

每个 provider 解析为 `TrioDescriptor`：

| 字段 | 含义 |
|---|---|
| `provider` | `btbn` / `bnqdzj-nonfree` |
| `channel` | `master-latest` / `n9.0-latest` / 其他当前发布通道 |
| `release_tag` | 实际 release tag；不能只写 `latest` |
| `asset_name` | 下载资产名 |
| `asset_sha256` | 资产校验和 |
| `binary_sha256` | `ffmpeg` / `ffprobe` / `ffplay` 本地哈希 |
| `variant` | `gpl` / `lgpl` / `nonfree` / `*-shared` |
| `target` | `win64` / `linux64` |
| `version_string` | `ffmpeg -version` 核心版本 |
| `buildconf_hash` | `ffmpeg -buildconf` 哈希 |
| `capabilities_hash` | 能力矩阵哈希 |
| `published_at` | release 发布时间 |
| `provider_fingerprint` | 以上关键字段的组合哈希 |

`provider_fingerprint` 取代此前“自构建 build ID”的位置，但语义不同：它描述**外部预构建产物的来源和指纹**，不是自构建 ID。

### 3.2 TOML 1.1 配置

```toml
schema = "lae.videodamage/v1"

[ffmpeg]
provider = "bnqdzj-nonfree"     # 或 "btbn"
channel = "master-latest"
variant = "nonfree"             # btbn 可为 "gpl" / "lgpl"
release_tag = "autobuild-2026-11-08-17-59"
asset_sha256 = "..."
trio_dir = "C:/Tools/LAE/ffmpeg"
allow_shared = false
```

规则：

- `provider` 必填；
- `channel` 必填；
- `release_tag` 与 `asset_sha256` 必填；
- `latest` 只能作为更新通道名称，不能作为可复现运行时基线；
- 不读取旧 LAE/v7 配置；
- 未知键拒绝；
- TOML 1.1 only，Rust 单点解析。

### 3.3 解析顺序

```text
--ffmpeg-dir
  → --ffmpeg-provider / --ffmpeg-release / --ffmpeg-variant
  → LAE_FFMPEG_DIR
  → LAE v8 配置
  → 出错
```

不静默回退到 PATH；不自动猜测 provider。

### 3.4 启动校验

必须校验：

1. `ffmpeg` / `ffprobe` / `ffplay` 三个文件存在且可执行；
2. 三件套来自同一目录；
3. `ffmpeg -version` / `ffprobe -version` / `ffplay -version` 核心版本一致；
4. `ffmpeg -buildconf` 包含预期许可证与组件开关：
   - BtbN GPL：`--enable-gpl --enable-version3`；
   - BtbN LGPL：不含 `--enable-gpl`；
   - BnqDzj nonfree：`--enable-gpl --enable-version3 --enable-nonfree`；
5. 资产 `checksums.sha256` 与本地二进制哈希一致；
6. `provider_fingerprint` 与配置/LAE 记录一致；
7. 若不匹配，返回 `LAE-VD-E1004`，不降级、不混用。

### 3.5 能力探测

运行时探测：

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

能力缓存 key：

```text
provider + release_tag + asset_sha256 + binary_sha256 + buildconf_hash + target + variant
```

缓存写入 TOML 1.1：

```text
%LOCALAPPDATA%/LAE/VideoDamage/caps/<provider_fingerprint>.toml
```

能力差异不能靠 provider 名称硬编码；一律以 `capabilities.toml` 为准。BtbN 与 BnqDzj 的 `buildconf`、编码器、滤镜、硬件接口可能不同，同一操作在某一 provider 不支持时应明确警告或拒绝，而不是静默降级。

---

## 4. “latest” 与“只支持最新构建”的重新定义

此前报告写“只使用最新构建 FFmpeg，不支持老版本”。双来源下应改为：

> **运行时不自动追踪 floating `latest`；由 LAE/用户在激活时显式 pin 当前 provider 的最新发布 tag 与资产 SHA-256。**

规则：

- `latest` release 是更新入口，不是可复现基线；
- 激活后记录 `release_tag`、`published_at`、`asset_sha256`、`binary_sha256`；
- 不自动下载、不自动切换；
- 不维护历史版本兼容矩阵；
- 配置中的 pinned tag/hash 不匹配当前 LAE 记录时拒绝启动；
- 需要升级时由显式命令/LAE 配置更新，重新跑能力矩阵、FATE 风格 smoke、真实语料与性能基准；
- BtbN 与 BnqDzj 发布时间可能相差一天，不要求两者同一时间戳，但要求各自是当前激活的 provider latest snapshot。

这样同时满足：

- “不支持老版本”；
- “统一支持两家”；
- “可复现”；
- “floating latest 不直接进 Manifest”。

---

## 5. 确定性与 CPU 基线修正

### 5.1 确定性

双来源下不能说“同一 trio 构建”而不说明 provider。应改为：

| 操作 | 口径 |
|---|---|
| 码流级 damage / truncate | D1：同输入 + 同 plan + 同 provider_fingerprint → 同输出 |
| frame/pixel | 决策确定；编码字节依赖 encoder/buildconf |
| packet BSF | D3：FFmpeg 内部随机数 |
| 容器输出 | 同 provider + release + asset hash + buildconf 内可复现；跨 provider、跨 release、跨 buildconf 不承诺 |

Manifest 必须记录：

- provider；
- channel；
- release tag；
- asset SHA-256；
- 三件套二进制 SHA-256；
- `ffmpeg -version` banner；
- `ffmpeg -buildconf` hash；
- capabilities hash；
- CPU 基线；
- 编码器/muxer；
- 验证结果。

### 5.2 x86-64-v3

此前报告把“项目强制 x86-64-v3”和“FFmpeg 三件套也 v3”混在一起，需要修正：

- 本项目 Rust/C++ 代码：强制 `x86-64-v3`，无 scalar fallback；
- FFmpeg / ffprobe / ffplay：当前是 BtbN / BnqDzj 上游预构建二进制，**不承诺编译期 `x86-64-v3`**；
- FFmpeg 的 SIMD 能力以 `runtime-cpudetect`、`ffmpeg -buildconf`、运行时探测为准；
- 启动时仍需检查宿主 CPU 是否满足项目自身的 AVX2/BMI2/FMA/MOVBE/SSE4.2；
- Manifest 分开记录：
  - `project_cpu_baseline = "x86-64-v3"`；
  - `ffmpeg_cpu_baseline = "upstream-prebuilt/runtime-cpudetect"`。

自构建报告中的 `--cpu=x86-64-v3`、ThinLTO、PGO、Clang/VS2026 FFmpeg 本体编译等内容全部归入**未来方案**，当前不实施。

---

## 6. 许可证边界

| Provider | 二进制性质 | 建议用途 |
|---|---|---|
| BtbN GPL | GPLv3 体系，按许可证可合规分发 | 公开 demo、CI、GPL release 验证 |
| BtbN LGPL | LGPL 体系 | 仅需 LGPL 能力的场景 |
| BnqDzj nonfree | FFmpeg `--enable-nonfree`，二进制 unredistributable | LAE/VideoDamage 内部、私有制品库、开发机 |

硬规则：

- 不要把 BnqDzj nonfree 二进制放进 public GitHub Release；
- 不要把它与 GPL-3.0-only 项目一起打包发布；
- 公开交付使用 BtbN GPL 构建；
- 内部若需要 `fdk-aac` 等 nonfree 能力，使用 BnqDzj；
- LICENSES / provenance 文件必须标明具体 provider、release tag、资产名和许可证。

---

## 7. 对现有报告的审查结果

### 7.1 必须修正的冲突

| 位置 | 原表述 | 问题 | 修正 |
|---|---|---|---|
| `README.md` 硬约束 | “仅使用 LAE 共享的自构建最新 FFmpeg 三件套” | 自构建已暂缓 | 改为两家上游预构建 provider |
| `docs/README.md` 当前结论 | “自构建 FFmpeg 三件套，build ID 锁定” | 与当前策略冲突 | 改为 BtbN / BnqDzj 双来源，provider_fingerprint |
| `docs/EXECUTIVE_SUMMARY.md` | “Rust + C++ + 自构建 FFmpeg 三件套” | 当前形态错误 | 改为外部预构建双来源 |
| `docs/ARCHITECTURE_REPORT.md` §5.0–5.2 | fork、固定 commit、自构建 build ID | 已暂缓 | 改为 provider 注册、release pin、能力探测 |
| `docs/ARCHITECTURE_REPORT.md` §8 | 容易误读为 FFmpeg 三件套也强制 v3 | 上游预构建不承诺 v3 | 明确只本项目 Rust/C++ 强制 v3 |
| `docs/ARCHITECTURE_REPORT.md` §14 / §21 | “同 trio 构建内可复现” | 未说明 provider 差异 | 改为同 provider_fingerprint 内可复现 |
| `docs/FFMPEG_BUILD_REPORT.md` | 全篇作为当前方案 | 自构建已暂缓 | 顶部加“未来方案/非当前实施依据” |
| M0 计划 | “固定 FFmpeg/FFmpeg-Builds commit，生成 build ID” | 当前不 fork/不编译 | 改为 provider 检测、release pin、capability matrix |

### 7.2 保持不变的结论

以下结论不因 provider 调整而改变：

- Rust 控制面 + C++ AVX2 数据面 + FFmpeg 三件套媒体层；
- 产品代码只 Rust + C++；
- Rust 1.99.1 / C++23 下限 / C++26 目标；
- Windows + Linux；
- 仅 CLI；
- 独立 CLI + Rust crate + C ABI 插件；
- TOML 1.1 only；
- 默认全流保留 `-map 0`；
- 输出事务、journal、quarantine；
- packet BSF D3 如实标注；
- 不 mock FFmpeg 的 e2e；
- C++ 依赖仍采用 `oneTBB` / `doctest` / `fmt`，拒绝 `folly` / `taskflow` / `GoogleTest` / `Catch2`。

### 7.3 需要新增的结论

- `BtbN/FFmpeg-Builds` 当前 latest 不发布 nonfree；
- `bnqdzj/nonfree` 实际仓库是 `BnqDzj/FFmpeg-Builds-nonfree`；
- 两家 provider 的 `buildconf` 和能力矩阵不同，必须按能力探测结果决策；
- floating `latest` 只能作为更新入口，运行时必须 pin release tag + SHA-256；
- nonfree 二进制不可公开分发；
- 自构建报告降级为未来方案。

---

## 8. 对 M0 的修正

原 M0 以自构建为前提，现改为：

1. 实现 `FfmpegProvider` / `TrioDescriptor` / `provider_fingerprint`；
2. TOML 1.1 中增加 `[ffmpeg] provider / channel / release_tag / asset_sha256`；
3. CLI 增加 `--ffmpeg-provider`、`--ffmpeg-release`、`--ffmpeg-variant`；
4. 实现 BtbN 与 BnqDzj 两种 provider 的目录解析、版本校验、buildconf 校验、SHA-256 校验；
5. 生成 `capabilities.toml`，按 provider_fingerprint 缓存；
6. CLI `probe` / `copy`；
7. C++ 内核骨架 + doctest + oneTBB；
8. 全流复制：4 流输入 → 4 流输出，逐流 hash 一致；
9. 输出事务 + manifest；
10. Windows/Linux 构建，C++23/C++26 双标准。

M0 验收新增：

- BtbN GPL/LGPL 与 BnqDzj nonfree 都能独立通过；
- provider 声明与 `buildconf` 不匹配直接 `LAE-VD-E1004`；
- release tag / asset SHA-256 / binary SHA-256 不匹配直接失败；
- floating `latest` 不进入 manifest；
- 能力矩阵按 provider 分别生成；
- nonfree 构建不会出现在公开发布资产清单中。

---

## 9. 测试矩阵

| 维度 | BtbN | BnqDzj |
|---|---|---|
| Windows x64 static | `gpl` / `lgpl` | `nonfree` |
| Linux x86_64 static | `gpl` / `lgpl` | `nonfree` |
| shared 变体 | 可选验证 | 可选验证 |
| `checksums.sha256` | 必测 | 必测 |
| buildconf 许可证开关 | 必测 | 必测 |
| 全流复制 | 必测 | 必测 |
| 老/新 codec decode | 必测 | 必测 |
| frame/pixel 操作 | 必测 | 必测 |
| packet BSF D3 | 必测 | 必测 |
| 跨 provider 输出一致性 | 不要求逐字节一致 | 不要求逐字节一致 |
| 公开发布许可检查 | GPL 通过 | nonfree 必须拒绝 |

---

## 10. 最终决策

| 编号 | 决策 |
|---|---|
| P1 | 自构建 FFmpeg 暂缓，`FFMPEG_BUILD_REPORT.md` 降级为未来方案 |
| P2 | 当前支持 `btbn` 与 `bnqdzj-nonfree` 两个 provider |
| P3 | `bnqdzj/nonfree` 的正式仓库名是 `BnqDzj/FFmpeg-Builds-nonfree` |
| P4 | BtbN 负责 GPL/LGPL；BnqDzj 负责 nonfree |
| P5 | 运行时不自动追踪 floating `latest`，必须 pin release tag + asset SHA-256 |
| P6 | 用 `provider_fingerprint` 取代自构建 `build ID` |
| P7 | 能力矩阵以实际 `ffmpeg -buildconf` / `-decoders` / `-filters` 等输出为准 |
| P8 | 本项目 Rust/C++ 强制 x86-64-v3；FFmpeg 上游预构建不承诺 v3 |
| P9 | nonfree 二进制只进私有制品库，不进 public release |
| P10 | 架构报告、结论摘要、README、文档索引全部按本审查修正 |

---

## 11. 结论

现有架构报告的**主体架构仍然成立**，但 FFmpeg 来源章节已经过时。需要把“自构建 FFmpeg”从当前策略降级为未来路线，把当前实施改为：

> **Rust 控制面 + C++ AVX2 数据面 + BtbN/BnqDzj 双来源预构建 FFmpeg 三件套。**

最关键的工程红线变成：

1. **provider 显式声明**：BtbN 与 BnqDzj 不混用、不猜测、不回退 PATH；
2. **release pin**：floating `latest` 只用于更新，运行时必须固定 release tag 与 SHA-256；
3. **能力探测**：不同 provider 的 buildconf 和 codec/filter 能力不同，按 capabilities 决策；
4. **确定性诚实**：D1 只在同 provider_fingerprint 内成立，跨 provider / 跨 release 不承诺；
5. **许可证分流**：BtbN GPL 可公开，BnqDzj nonfree 仅私有。

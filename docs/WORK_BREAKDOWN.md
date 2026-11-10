# 分工、准备步骤与质量门禁

> 日期：2026-11-10
> 轮次：第二轮核查同步
> 原则：质量优先，不以编码速度为目标
> 前置 Go/No-Go：[`PRE_CODING_FINAL_REVIEW.md`](PRE_CODING_FINAL_REVIEW.md)
> 第二轮核查：[`PRE_CODING_ROUND2_CHECK.md`](PRE_CODING_ROUND2_CHECK.md)
> FFmpeg 契约：[`FFMPEG_PROVIDER_REVIEW.md`](FFMPEG_PROVIDER_REVIEW.md)

---

## 1. 工作流划分

项目按**职责边界**拆成 6 个工作流，不按“谁有空谁写”拆。

| 工作流 | 负责范围 | 主要产物 |
|---|---|---|
| WS1 Rust Control Plane | CLI、TOML、Recipe、Plan、调度、Manifest、输出事务、错误码 | `lae-vd-core`、`lae-vd-cli` |
| WS2 FFmpeg Integration | trio 发现、锁文件、buildconf、能力探测、进程监督、命令构建、全流映射 | `lae-vd-ffmpeg` |
| WS3 C++ Data Plane | AVX2/SSE4 内核、buffer、oneTBB、doctest、benchmark | `lae-vd-kernels` |
| WS4 Quality & Test | 夹具、e2e、fuzz、CI、FATE 风格 smoke、性能基线、回归门禁 | `tests/`、CI |
| WS5 Build / Release / Plugin | xmake、Conan、Cargo、Windows/Linux、C ABI、打包、发布 | `xtask`、`lae-vd-plugin` |
| WS6 FFmpeg Provider Extension | 下载、安装、部署、更新、锁生成、未来 nonfree/自构建 | 独立扩展包 |

### 1.1 目录归属建议

```text
Cargo.toml
rust-toolchain.toml
.cargo/config.toml
crates/
  lae-vd-core/          WS1
  lae-vd-cli/           WS1
  lae-vd-ffmpeg/        WS2
  lae-vd-plugin/        WS5
cpp/
  kernels/              WS3
  include/              WS3
  tests/                WS3 + WS4
tests/
  fixtures/             WS4
  e2e/                  WS4
  golden/               WS4
docs/                   WS1 + WS2 + WS4
xtask/                  WS5
extensions/
  ffmpeg-provider/      WS6
```

---

## 2. 各工作流职责

### 2.1 WS1：Rust Control Plane

拥有：

- CLI 命令与参数；
- TOML 1.1 解析与 schema；
- Recipe → Plan 编译器；
- Play / Stage / Unit 调度；
- Manifest；
- 输出事务；
- 错误码；
- 插件适配层（与 WS5 共同）。

交付：

- `lae-vd-core` 公共 API；
- `lae-vd-cli` 可执行文件；
- CLI 契约文档；
- 错误码注册表；
- Plan / Manifest 编解码；
- 单元测试与契约测试。

不负责：

- FFmpeg 下载/安装；
- C++ 内核实现；
- FFmpeg 命令细节；
- 插件 C ABI 的 ABI 布局（与 WS5 共同评审）。

### 2.2 WS2：FFmpeg Integration

拥有：

- `ffmpeg-trio.lock` 解析；
- trio 发现与版本校验；
- `ffmpeg -buildconf` 校验；
- 能力探测与缓存；
- 进程监督（stdout/stderr drain、超时、取消、进程树）；
- FFmpeg 命令构建；
- 全流映射 `-map 0`；
- 输出验证。

交付：

- `lae-vd-ffmpeg`；
- `ffmpeg-trio.lock` 样例；
- `capabilities.toml`；
- FFmpeg 进程监督测试；
- 4 流全流复制门禁。

不负责：

- 下载/安装/部署；
- damage 算法；
- C++ 内核；
- CLI 外观。

### 2.3 WS3：C++ Data Plane

拥有：

- AVX2/SSE4 扫描；
- bitflip / destructive 内核；
- buffer / span / 批处理；
- oneTBB 并行；
- doctest 单测；
- Google Benchmark 微基准。

交付：

- `lae-vd-kernels` 静态库；
- `cxx` 桥头文件；
- 内核 API；
- 单测与基准；
- Windows MSVC / Linux Clang 构建。

不负责：

- TOML；
- FFmpeg 进程；
- 输出事务；
- CLI；
- Manifest。

### 2.4 WS4：Quality & Test

拥有：

- 测试策略；
- 夹具与 golden；
- `nextest` / `rstest` / `insta` / `loom` / `proptest`；
- C++ doctest 汇总；
- fuzz / sanitizer；
- FATE 风格 smoke；
- CI 矩阵；
- 性能基线；
- 回归门禁。

交付：

- 4 流夹具；
- 长归档夹具；
- 老/新 codec 样本；
- 损坏容器样本；
- CI workflow；
- FATE 风格 smoke 定义与语料；
- 性能阈值；
- 质量报告。

不负责：

- 产品功能实现；
- FFmpeg 安装；
- 发布打包。

### 2.5 WS5：Build / Release / Plugin

拥有：

- Cargo workspace；
- xmake；
- Conan profile；
- Rust 1.99.1 / C++23/C++26；
- Windows MSVC 2026；
- Linux Clang 23 + libc++；
- x86-64-v3；
- C ABI plugin；
- 打包、签名、发布清单；
- `xtask`。

交付：

- 可重复构建；
- 双平台产物；
- 插件 DLL/SO；
- ABI 版本；
- 发布检查表；
- 许可证/第三方声明。

不负责：

- 业务逻辑；
- 测试夹具内容；
- FFmpeg provider 安装。

### 2.6 WS6：FFmpeg Provider Extension Pack

拥有：

- BtbN `n9.0` 稳定版解析；
- 下载、校验、解压、安装、部署；
- `ffmpeg-trio.lock` 生成；
- 更新/回滚；
- 未来 BnqDzj nonfree；
- 未来自构建。

交付：

- 独立安装器/扩展包；
- 锁文件；
- 安装目录布局；
- 版本清单；
- 回滚策略。

不负责：

- core 内部逻辑；
- damage 算法；
- C++ 内核；
- CLI 业务。

---

## 3. 接口与交接

### 3.1 关键交接物

| 从 | 到 | 交接物 | 验收 |
|---|---|---|---|
| WS6 | WS2 | `trio_dir` + `ffmpeg-trio.lock` | 三件套哈希、buildconf、版本一致 |
| WS1 | WS3 | Plan ranges / unit decisions | span、kind、seed、expected stats |
| WS3 | WS1 | mutation result / stats | 实际字节数、哈希、错误码 |
| WS2 | WS1 | FFmpeg 进程结果 / 能力 | exit code、stderr、capabilities |
| WS1 | WS5 | C ABI 请求/回调 | ABI 版本、opaque handle、取消 |
| WS4 | 全部 | 门禁结果 | 通过/失败、日志、性能数据 |

### 3.2 不允许的耦合

- C++ 不解析 TOML；
- C++ 不启动 FFmpeg；
- core 不包含下载/解压/安装逻辑；
- Extension Pack 不 import core 内部类型；
- 插件不跨边界传 Rust `String` / `Vec` / `Result`；
- 任何模块不手写重复错误码。

---

## 4. 编码前准备步骤

### Phase 0：契约冻结（不写业务代码）

唯一产出：

- 按 [`PRE_CODING_FINAL_REVIEW.md`](PRE_CODING_FINAL_REVIEW.md) §5 的 **C1–C10** 逐项冻结。

不再另列不同枚举；其他文档只引用 C1–C10。C1–C10 包含：

- `ffmpeg-trio.lock` v1；
- TOML config v1；
- Plan v5；
- Manifest v1；
- 错误码注册表（含 `E9xxx`）；
- Plugin C ABI v1；
- CLI 契约；
- 仓库布局；
- 构建矩阵；
- 质量基线、许可证与第三方声明策略。

完成标准：

- 每个 schema 有示例；
- 每个错误码有说明；
- 每个接口有 Rust/C++ 两侧签名；
- 文档评审通过；
- 不再出现“边写边定”；
- 所有草案片段被替换为冻结版本或明确标注为草案。

### Phase 1：仓库骨架

产出：

- Cargo workspace；
- `rust-toolchain.toml`；
- `.cargo/config.toml`；
- 空 `lae-vd-core` / `lae-vd-cli` / `lae-vd-ffmpeg` / `lae-vd-plugin`；
- 空 C++ kernels + `xmake.lua`；
- Conan clang-23/libc++ profile；
- `xtask`。

完成标准：

- Windows / Linux 都能编译空 CLI；
- C++23 / C++26 都能编译空内核；
- `clippy` / warnings-as-errors 通过；
- 不包含任何业务逻辑。

### Phase 2：FFmpeg 基线

产出：

- 手工安装 BtbN `n9.0` GPL 三件套；
- `ffmpeg-trio.lock`；
- `capabilities.toml`；
- `probe` 输出。

完成标准：

- 三件套版本一致；
- buildconf 校验通过；
- SHA-256 匹配；
- `probe` 输出 provider / version / lock 状态；
- 无锁时 `probe` 标记 `unverified`。

### Phase 3：测试与 CI

产出：

- 4 流夹具；
- 长归档夹具；
- codec 样本；
- 损坏容器；
- FATE 风格 smoke；
- Rust/C++ 测试；
- CI workflow；
- 性能基线。

完成标准：

- `nextest` 全绿；
- C++ doctest 全绿；
- Windows/Linux 双平台；
- C++23/C++26 双标准；
- 4 流夹具逐流 hash 一致；
- FATE 风格 smoke 通过。

### Phase 4：M0

产出：

- CLI `probe` / `copy`；
- all-stream copy；
- 输出事务；
- Manifest；
- 错误路径。

完成标准：

- 4 流进 4 流出；
- 每流 hash 一致；
- 锁文件/版本/buildconf/SHA-256 不匹配直接失败；
- Ctrl-C 清理临时文件；
- 通过后才写 damage。

---

## 5. 质量门禁

### 5.1 每个 PR

- `cargo fmt --check`；
- `cargo clippy -D warnings`；
- `cargo nextest`；
- C++ warnings-as-errors；
- C++ doctest；
- Markdown 链接检查；
- 不允许新增未登记错误码；
- 不允许静默降级。

### 5.2 每个 milestone

- Windows + Linux 都构建；
- C++23 + C++26 都构建；
- 4 流门禁；
- 全流保留；
- 输出事务故障注入；
- 锁文件/哈希校验；
- FATE 风格 smoke；
- 性能基线；
- 文档与契约同步。

### 5.3 发布前

- `cargo-deny` / `cargo-audit`；
- 依赖版本锁定；
- 许可证清单；
- nonfree 不进 public release；
- 安装包 SHA-256；
- 回滚说明；
- 发布说明。

---

## 6. M0 Definition of Done

M0 必须同时满足：

1. BtbN `n9.0` GPL 三件套可被 core 发现；
2. `ffmpeg-trio.lock` 解析、校验、错误路径完整；
3. `probe` / `copy` 可用；
4. 4 流输入 → 4 流输出；
5. 每条流 hash 一致；
6. 字幕、第二音轨、章节、附件不丢；
7. 输出事务可恢复；
8. Manifest 完整；
9. Windows / Linux 构建；
10. C++23 / C++26 构建；
11. Ctrl-C 清理；
12. 没有 FFmpeg 下载/安装代码进入 core。

---

## 7. 工作顺序

不按时间排，只按依赖排：

```text
Phase 0  C1–C10 契约冻结
   ↓
Phase 1  骨架
   ↓
Phase 2  FFmpeg 基线
   ↓
Phase 3  测试 / CI / 性能基线
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

- 上述 M1–M4 与 [`ARCHITECTURE_REPORT.md`](ARCHITECTURE_REPORT.md) §19 的功能里程碑一致；
- [`FFMPEG_BUILD_REPORT.md`](FFMPEG_BUILD_REPORT.md) 中的 M0–M5 是 Extension Pack 自构建未来路线，不是 core 的 Phase/M0；
- M0 未通过，不写 damage；
- 任何“先跑起来再补契约”的做法视为门禁失败。

原则：

- 上游未冻结，不开始下游；
- M0 未通过，不写 damage；
- 测试未覆盖，不合并；
- 文档未同步，不发布。

---

## 8. 最终约定

- 时间不着急；
- 不以“先跑起来”为理由跳过契约；
- 不以“以后再补测试”为理由合并；
- 不以“先支持 nonfree”为理由污染 core；
- 不以“先做安装器”为理由拖延 M0；
- 质量优先，边界清晰，逐步推进。

# 自构建 FFmpeg 三件套技术评估报告（暂缓 / 未来方案）

> 日期：2026-11-10  
> 状态：**已暂缓，不是当前实施依据**  
> 当前实施依据：[`FFMPEG_PROVIDER_REVIEW.md`](FFMPEG_PROVIDER_REVIEW.md)  
> 项目：LiveArchiveEncoder-VideoDamage / LiveArchiveEncoder v8  
> 原目标：为母项目与 VideoDamage 扩展包共同提供一套定制、`nonfree+gpl`、覆盖尽可能全、面向 `x86-64-v3` 深度优化的 `ffmpeg` / `ffprobe` / `ffplay` 三件套  
> 原基线：<https://github.com/FFmpeg-Builds/FFmpeg-Builds>  
> 说明：本报告中关于自构建、FFmpeg 本体 `x86-64-v3`、ThinLTO/PGO、Clang/VS2026 编译 FFmpeg 的内容全部保留为未来技术储备；当前项目不执行自构建。

---

## 1. 执行摘要

> 状态提示：本章及以下所有“建议 / 最终建议”均属于**未来方案**；当前实施依据是 [`FFMPEG_PROVIDER_REVIEW.md`](FFMPEG_PROVIDER_REVIEW.md)，即 BtbN / BnqDzj 双来源预构建三件套。

可以基于 `FFmpeg-Builds/FFmpeg-Builds` 做自构建，但不能直接照搬上游 daily build。原因是用户要求同时满足：

1. `nonfree + gpl`、尽可能全的编解码器、硬件加速、字幕、滤镜；
2. 最新工具链 + `x86-64-v3`，深度优化，程序内部也使用 v3 指令；
3. Linux 最新 Clang + OpenMP/LTO，Windows VS2026；
4. 修复潜在 BUG，功能与性能都要可验证；
5. 双平台产物供 LAE 母项目与 VideoDamage 共享。

调研结论：

- 上游 `FFmpeg-Builds` 的 `nonfree` 变体已经包含 `--enable-gpl --enable-version3 --enable-nonfree` 和约 144 个 `scripts.d` 依赖，覆盖绝大多数现代/传统编解码器、字幕、滤镜、硬件接口。
- 但上游默认：
  - Linux/Windows 都用 **crosstool-ng 生成的 GCC/MinGW-w64 工具链**；
  - 默认 `GIT_BRANCH=master`，floating `latest` 不可复现；
  - 默认优化是 GCC/Clang `-O3`、MSVC `-O2`，没有强制 `x86-64-v3`；
  - `lto` addin 是 `-flto=auto`，README 明确标注 **Windows broken、Linux experimental**；
  - 没有 OpenMP 统一开关；FFmpeg configure 本身也没有 `--enable-openmp`。
- 因此必须 fork 并新增自定义 target/variant/addin，做**工具链替换、提交固定、能力验收、LTO/PGO/OpenMP 分层验证、许可证分流**。

最终建议：

> **建立 `lae-ffmpeg-build` 独立构建仓库/子模块，从 FFmpeg-Builds fork，固定 FFmpeg tag/commit 和全部依赖 commit；Linux 用 Clang 23 + LLD + ThinLTO/PGO，Windows 先保留上游 MinGW-w64 基线、再评估 clang-cl + VS2026 SDK；产物仍作为 LAE/VideoDamage 共享三件套。**

---

## 2. 上游基线调研

### 2.1 仓库状态

- `FFmpeg-Builds/FFmpeg-Builds` 当前默认分支：`master`。
- `BtbN/FFmpeg-Builds` 已指向同一仓库；用户给出的 URL 是现在正确的组织名。
- 上游 `latest` release 是 floating tag，2026-11-09 13:03 UTC 发布，`target_commitish=master`。
- 仓库脚本许可证：MIT（Copyright 2020-2026 BtbN）。
- 目录结构：
  - `variants/`：69 个变体；
  - `addins/`：版本分支、`debug`、`lto` 等；
  - `scripts.d/`：144 个依赖脚本/目录；
  - `images/`：Docker 基础镜像与交叉工具链；
  - `util/`、`build.sh`、`generate.sh`、`makeimage.sh`：构建编排。

### 2.2 变体

与需求最相关的是：

| 变体 | 含义 |
|---|---|
| `gpl` | `--enable-gpl --enable-version3 --disable-debug`，包含 GPL-only 依赖，如 x264/x265 |
| `nonfree` | 在 `gpl` 基础上追加 `--enable-nonfree`，主要是 `libfdk-aac` |
| `*-shared` | 输出 libav* 共享库而非纯静态可执行文件 |
| `linux64-nonfree` | Linux x86_64 静态 nonfree |
| `win64-nonfree` | Windows x86_64 静态 nonfree |

`linux64-nonfree.sh` / `win64-nonfree.sh` 都是：

```bash
source linux64-gpl.sh   # 或 win64-gpl.sh
FF_CONFIGURE="--enable-nonfree $FF_CONFIGURE"
LICENSE_FILE=""
```

注意 `LICENSE_FILE=""`：上游 nonfree 构建不附带许可证文件，因为它**不可再分发**。

### 2.3 Addins

| Addin | 作用 |
|---|---|
| `9.0` | `GIT_BRANCH=release/9.0` |
| `8.1` | `GIT_BRANCH=release/8.1` |
| `lto` | `FF_CONFIGURE="$FF_CONFIGURE --enable-lto"`，并把 `-flto=auto` 注入 CFLAGS/CXXFLAGS/LDFLAGS |
| `debug` | 不 strip 调试符号 |

README 对 `lto` 的原话是：

> HIGHLY EXPERIMENTAL, broken for Windows, sometimes works for Linux

所以上游 `lto` addin 不能直接当生产方案。

### 2.4 FFmpeg 上游版本

查询 FFmpeg tags，当前可见：

```text
n9.1-dev
n9.0.2
n9.0.1
n9.0
n8.2-dev
n8.1.3
...
```

建议：

- **生产基线**：固定 `n9.0.2` 或精确 commit；不要使用 floating `latest`。
- **前沿构建**：固定 master commit，用于验证新 codec / 新优化，不与生产 trio 混用。
- 每个产物写入构建 ID：`ffmpeg-<tag>-<commit>-<target>-<toolchain>-<flags-hash>`。

### 2.5 上游工具链现实

`images/base/Dockerfile`：

- Ubuntu 26.04；
- 安装 `build-essential`、`clang`、`llvm`、`lld`、`nasm`、`yasm` 等；
- 但 `images/base-linux64/Dockerfile` 与 `images/base-win64/Dockerfile` 都通过 crosstool-ng 构造交叉 GCC：
  - Linux：`x86_64-ffbuild-linux-gnu-gcc/g++`；
  - Windows：`x86_64-w64-mingw32-gcc/g++`；
  - `CC` / `CXX` 都指向 GCC，不是 Clang/MSVC。

所以：

- “Linux 用最新 Clang 编译”不是上游默认，需要改 toolchain；
- “Windows 用 VS2026 编译”也不是上游默认，上游是 Linux 上交叉 MinGW-w64；
- 但因为本项目通过**外部进程**调用三件套，不链接 libav*，FFmpeg 自身的编译器/CRT 与本项目 MSVC ABI **没有直接 ABI 冲突**。

---

## 3. “开启全部支持的编解码器”的准确含义

### 3.1 FFmpeg configure 默认行为

FFmpeg `configure` 有：

```text
--disable-everything     disable all components listed below
--enable-decoder=NAME
--enable-encoder=NAME
--enable-hwaccel=NAME
--enable-parser=NAME
--enable-bsf=NAME
--enable-filter=NAME
```

默认是**启用全部内建组件**，只有 `--disable-everything` 才会全部关闭。  
因此“全部支持的编解码器”不需要逐个 `--enable-decoder=...`，关键是：

1. 不使用 `--disable-everything`；
2. 不传任何 `--disable-encoder/decoder/parser/bsf/filter/muxer/demuxer`；
3. 打开 `--enable-gpl --enable-version3 --enable-nonfree`；
4. 通过 `scripts.d` 编入尽可能全的外部库；
5. 提供所有硬件加速头/库，让 autodetect 成功。

外部库由 `scripts.d/zz-final.sh` 汇总，上游 nonfree 变体实际覆盖极广，包括：

- 视频编码：`x264`、`x265`、`libaom`、`dav1d`、`svtav1`、`rav1e`、`libvpx`、`openh264`、`xvid`、`vvenc`（VVC）、`davs2/xavs2`（AVS2）、`uavs3d`（AVS3）、`lcevcdec`、`openapv`、`libjxl`、`libwebp` 等；
- 音频编码：`fdk-aac`、`libmp3lame`、`libopus`、`libvorbis`、`opencore-amr`、`twolame`、`soxr`、`rubberband` 等；
- 字幕/字体：`libass`、`freetype`、`fontconfig`、`harfbuzz`、`fribidi`、`libunibreak`、`libaribb24`、`libaribcaption`、`libzvbi`；
- 滤镜/图像：`zimg`、`libplacebo`、`vidstab`、`frei0r`、`lv2`、`lilv`、`chromaprint`、`whisper`、`snappy`、`lcms2`、`libsvg` 等；
- 协议/容器/工具：`openssl`、`srt`、`librist`、`libssh`、`libcurl`、`libbluray`、`libdvd*`、`libxml2`、`zlib`、`xz`、`brotli`、`vapoursynth`、`avisynth` 等；
- 硬件接口：`ffnvcodec` + `cuda-llvm`、`amf`、`onevpl`、`opencl`、`vulkan`/`shaderc`/`spirv-cross`、`vaapi`。

### 3.2 老解码器与现代解码器

应明确写入能力矩阵：

| 类别 | 例子 |
|---|---|
| 传统/归档视频 | H.261、H.263/H.263+、MPEG-1/2/4 Part 2、MSMPEG4、WMV7/8/9、RealVideo、Sorenson、Cinepak、MJPEG、DV、Huffyuv、FFV1、Ut Video、ProRes、DNxHD、Theora、VP8 |
| 现代视频 | H.264/AVC、HEVC、VVC/H.266、AV1、VP9、AVS2/AVS3、LCEVC、JPEG XL、WebP/AVIF 相关 |
| 传统/归档音频 | MP2、MP3、AAC、AC3/E-AC3、DTS、TrueHD/MLP、Vorbis、WMA、RealAudio、AMR、G.7xx、iLBC、Speex、Opus、FLAC、ALAC、APE 等 |
| 字幕 | SRT/ASS/SSA、DVB subtitle、teletext、ARIB caption、CC/EIA-608/708、PGS、DVB/ATSC 相关 |

“全部支持”应定义为：

> 上游 FFmpeg 内建组件 + 当前 `scripts.d` 在该 target/variant 下能成功编译进来的全部外部库能力。

不要承诺“所有历史上存在过的 codec 都有”。验收方式是对比：

```text
ffmpeg -buildconf
ffmpeg -decoders
ffmpeg -encoders
ffmpeg -hwaccels
ffmpeg -filters
ffmpeg -bsfs
ffmpeg -muxers
ffmpeg -demuxers
ffmpeg -protocols
ffprobe -version
ffplay -version
```

生成 TOML 能力矩阵并纳入构建产物。

---

## 4. 硬件加速评估

FFmpeg configure 硬件相关选项包括 autodetect / enable：

```text
--enable-ffnvcodec --enable-cuda-llvm
--enable-nvenc --enable-nvdec --enable-cuvid
--enable-amf
--enable-libvpl
--enable-opencl
--enable-vulkan
--enable-vaapi --enable-vdpau
--enable-d3d11va / --enable-d3d12va / --enable-dxva2 / --enable-mediafoundation
```

上游脚本对应：

| 接口 | 上游脚本 | 备注 |
|---|---|---|
| NVIDIA | `50-ffnvcodec.sh` | 同时 `--enable-ffnvcodec --enable-cuda-llvm`；`cuda-llvm` 依赖 Clang |
| AMD AMF | `50-amf.sh` | `--enable-amf` |
| Intel QSV / oneVPL | `50-onevpl.sh` | `--enable-libvpl`；`libmfx` 已废弃 |
| Vulkan | `47-vulkan/*` | headers/loader/shaderc/spirv-cross |
| OpenCL | `45-opencl.sh` | `--enable-opencl` |
| VAAPI | `50-vaapi/*` | libpciaccess/libdrm/libva；Linux 为主 |

关键判断：

- 这些是**编译期能力**，不等于运行期一定可用；实际还依赖 GPU、驱动、SDK、容器权限、显示器会话。
- Windows 的 D3D11VA/D3D12VA/DXVA2/MediaFoundation 多为 autodetect，不应传 `--disable-*`。
- `cuda-llvm` 要求 Clang；若 Windows 走纯 MSVC `cl.exe`，可能需要关闭 CUDA_LLVM 或改用 `clang-cl`。
- 建议构建产物附带 `capabilities.toml`，记录编译期开关；运行期另用 `ffmpeg -hwaccels` 和真实硬件 smoke test 验证。

---

## 5. 深度优化评估

### 5.1 x86-64-v3

用户要求“强制 x86-64-v3，不向下兼容，程序内部能用就用”。

FFmpeg configure 提供：

```text
--cpu=CPU    select the minimum required CPU (affects instruction selection, may crash on older CPUs)
--optflags=OPTFLAGS
--extra-cflags / --extra-cxxflags / --extra-ldflags
--enable-runtime-cpudetect
--enable-hardcoded-tables
--enable-lto
```

在 GCC/Clang 路径，`--cpu=x86-64-v3` 会映射为 `-march=x86-64-v3`。建议：

```text
--cpu=x86-64-v3
--extra-cflags="-march=x86-64-v3 -mtune=generic -fno-semantic-interposition -fvisibility=hidden"
--extra-cxxflags="-march=x86-64-v3 -mtune=generic -fno-semantic-interposition -fvisibility=hidden"
--extra-ldflags="-march=x86-64-v3"
```

注意：

- 默认 `runtime-cpudetect` 建议**保留**：FFmpeg 有大量手写 x86inc SIMD 依赖运行期分发；编译期 v3 已经让二进制只跑在 v3 CPU 上，禁用 runtime cpudetect 收益很小、风险更高。
- 不要使用 `-march=native`：构建机相关，破坏可复现。
- 不要使用 `-Ofast` / `-ffast-math`：会改变浮点舍入，影响解码一致性和 golden output。
- FMA/AVX2 可能改变浮点结果的最后一位；必须用 FATE 和归档语料验证，必要时对特定文件禁用 FMA contraction。
- MSVC `cl.exe` 不支持 `-march=`，configure 也会过滤 `-march=*`；MSVC 路径要用 `/arch:AVX2` 并验证 FFmpeg 的 `__AVX2__` / x86inc 行为。

### 5.2 优化等级

FFmpeg configure 内部默认：

| 编译器 | `cflags_speed` |
|---|---|
| GCC | `-O3` |
| Clang | `-O3` |
| MSVC | `-O2` |

因此不要盲目重复叠加 `-O3`；深度优化重点应放在：

- `-march=x86-64-v3`；
- LTO / ThinLTO；
- PGO；
- `-fno-semantic-interposition` / `-fvisibility=hidden`；
- `--enable-hardcoded-tables`；
- `--disable-debug`；
- 依赖库自身的 ASM / SIMD 开关；
- 针对工作负载的线程/队列参数。

### 5.3 LTO

上游 `addins/lto.sh`：

```bash
FF_CONFIGURE="$FF_CONFIGURE --enable-lto"
CFLAGS/CXXFLAGS/LDFLAGS += -flto=auto
AR/NM/RANLIB 换 gcc-ar/gcc-nm/gcc-ranlib
```

问题：

- README 明说 Windows broken；
- `-flto=auto` 是 GCC 语义；
- MSVC 需要 `/GL` + `/LTCG`；
- Clang 更适合 `-flto=thin` + `lld` + `llvm-ar`；
- 144 个依赖里有 autotools、CMake、Meson、Rust，LTO 不可能“一条 ENV 全开就稳定”。

建议分平台：

| 平台 | 方案 |
|---|---|
| Linux + Clang | `--enable-lto=thin`，`CC=clang`、`CXX=clang++`、`LD=lld`、`AR=llvm-ar`、`RANLIB=llvm-ranlib`、`NM=llvm-nm`；所有依赖审计 LTO 兼容性 |
| Windows + MinGW GCC | 可用上游 `lto` 做实验，但必须单独验证；不要默认进生产 |
| Windows + MSVC | 需要自定义 `/GL` `/LTCG` 或 clang-cl ThinLTO；上游脚本不能直接复用 |
| Windows + clang-cl | 最可能统一 Clang 风格优化，但要改大量依赖脚本 |

`--enable-lto` 必须伴随 FATE/全 codec smoke/性能回归，不能只看链接成功。

### 5.4 OpenMP

调研结论：

- FFmpeg `configure` **没有** `--enable-openmp`；
- 搜索到的只有 `--enable-libopenmpt`（tracker module 解码），不是 OpenMP；
- FFmpeg 核心线程模型是 pthreads / w32threads，不是 OpenMP；
- OpenMP 应当理解为**依赖库层面的线程并行**：例如 x264/x265/VMAF/vvenc 等各自 CMake/configure 的 OpenMP 选项，或在 Clang 下 `-fopenmp=libomp` 编支持它的依赖；
- 不能给 FFmpeg 传一个不存在的开关，也不能假设“全局加 `-fopenmp`”就会加速。

建议：

1. 先列出每个依赖是否支持 OpenMP；
2. 只在实际受益的依赖上打开；
3. 用 `oneTBB`/Rust 调度器和 FFmpeg `-threads` 做整体线程预算，避免超订；
4. 对启用/不启用 OpenMP 做 A/B 基准；
5. 在 `capabilities.toml` 记录每个依赖的 OpenMP 状态。

### 5.5 PGO / BOLT

用户要求“架构、性能、效率全方位大量提升”，PGO 值得做，但不是第一版默认：

- Clang PGO：
  - 第一遍 `-fprofile-instr-generate`；
  - 用代表性语料跑 `ffmpeg` / `ffprobe`；
  - `llvm-profdata merge`；
  - 第二遍 `-fprofile-instr-use` + LTO。
- BOLT：
  - 需要 `lld` + 可重定位二进制；
  - 对大二进制布局有收益，但构建链复杂度高。
- 风险：
  - 训练语料不代表归档/损伤工作负载会负优化；
  - 必须保留非 PGO 基线；
  - PGO 产物与编译 flag、commit、依赖版本强绑定。

建议放在 M4，而不是 M0/M1。

### 5.6 “程序内部能用 v3 特性就用”

分三层：

| 层 | 做法 |
|---|---|
| 本项目 Rust | `.cargo/config.toml` `target-cpu=x86-64-v3`，`is_x86_feature_detected!` 启动检查；AVX2 内核放 C++ |
| 本项目 C++ | `/arch:AVX2`（MSVC）/ `-march=x86-64-v3`（Clang）；`<immintrin.h>`；doctest + Google Benchmark |
| FFmpeg | `--cpu=x86-64-v3` + 保留 runtime cpudetect + 手写 SIMD/AVX2 分发；不禁用 avx2/fma3 |

三层必须同时满足 v3；启动时的 CPUID 检查由本项目入口统一给出清晰错误。

---

## 6. Windows / Linux 工具链评估

### 6.1 Linux：最新 Clang

目标：

- Clang 23.1.6；
- `lld`、`llvm-ar`、`llvm-ranlib`、`llvm-nm`；
- `libomp`（若依赖启用 OpenMP）；
- C 用 Clang，C++ 依赖用 `clang++` + libc++ 或 libstdc++ 需统一验证；
- glibc 2.36（Debian 12）兼容。

需要改的地方：

- `images/base-linux64/Dockerfile` 不再默认 `CC=x86_64-ffbuild-linux-gnu-gcc`；
- `toolchain.cmake` 的 `CMAKE_C_COMPILER` / `CMAKE_CXX_COMPILER` 改 Clang；
- `cross.meson` 改 Clang；
- `util/vars.sh` 的 `FFBUILD_TARGET_FLAGS` 改 `--cc=clang --cxx=clang++ --ld=lld --ar=llvm-ar --ranlib=llvm-ranlib --nm=llvm-nm`；
- 所有 `scripts.d` 依赖重新验证：autotools 可能依赖 GCC 行为，CMake/Meson 大多可切；
- Rust 依赖（如 `rav1e`、`rustdedup`）单独检查 target 与 linker。

结论：**可行，但必须逐依赖验证；不能只改 FFmpeg 的 `--cc`。**

### 6.2 Windows：VS2026

上游现实：

- `win64` 是 Linux Docker 内用 `x86_64-w64-mingw32` 交叉编译；
- 产物是 MinGW-w64/UCRT 静态 Windows exe；
- 不是 MSVC 构建；
- 上游 LTO addin 对 Windows 明确 broken。

关键判断：

> 本项目调用 FFmpeg 是**进程边界**，不是链接 `libav*`。因此 FFmpeg 用 MinGW-w64 构建，与本项目用 VS2026/MSVC 构建并不冲突。强行要求 FFmpeg 本体也用 VS2026 编译，收益主要是统一工具链/调试体验，成本高且风险大。

三条路径：

| 路径 | 说明 | 评价 |
|---|---|---|
| A. 保留上游 MinGW-w64，VS2026 只编译本项目 | 最稳；先拿功能/性能基线 | **推荐 M0/M1** |
| B. `clang-cl` + VS2026 SDK + `lld-link` | 保留 MSVC ABI/CRT，同时更接近 Clang 优化语义；可尝试 `-Xclang -march=x86-64-v3` / ThinLTO | **推荐作为 M2 评估主线** |
| C. 纯 `cl.exe` / `link.exe` MSVC 原生 | `--toolchain=msvc`、`/arch:AVX2`、`/GL`、`/LTCG`；144 个依赖的 autotools/CMake/Meson 改造量最大 | 仅在合规/调试强需求时做 |

建议路线：

1. M0/M1：Windows 先沿用 `win64-nonfree` MinGW-w64，追加 v3 编译 flag 与能力验收；VS2026 用于本项目。
2. M2：并行做 `clang-cl` + VS2026 SDK 实验分支，跑 FATE/语料/性能对比。
3. 只有当 B 通过全部门禁、且确实有性能/维护收益，才把 Windows 生产 trio 切到 VS2026 工具链。
4. 无论哪条路径，构建产物必须带完整 `buildconf` 和编译器版本；项目 Manifest 记录 build ID。

---

## 7. 授权与分发边界

这是必须写进报告的硬风险。

`nonfree` 变体：

```text
--enable-gpl --enable-version3 --enable-nonfree
```

FFmpeg configure 帮助明确写：

```text
--enable-nonfree ... the resulting libs and binaries will be unredistributable
```

含义：

- 产物**不可再分发**；
- 不能作为 GitHub Release 资产公开；
- 不能随 GPL-3.0-only 的本项目二进制一起发布；
- 上游 `LICENSE_FILE=""` 正是因为这个原因；
- 内部使用、CI、LAE 本机共享可以，但要有内部制品库和访问边界；
- GPL 部分的源码提供义务仍在，但 nonfree 组件使整体二进制无法合法公开发行。

建议双轨产物：

| 产物 | 用途 | 分发 |
|---|---|---|
| `nonfree+gpl` 自构建 trio | LAE/VideoDamage 内部、CI、开发机共享 | 私有制品库；不公开 |
| `gpl` 构建（去掉 `--enable-nonfree` / `fdk-aac`） | 公开 demo / release 验证 | 可随 GPL 项目分发，但需完整许可证与源码信息 |

另外：

- `FFmpeg-Builds` 脚本本身 MIT，fork 时需保留版权声明；
- 每个依赖的许可证写入 `licenses.toml`；
- 构建产物附带 `LICENSES.md`、`buildconf.txt`、`configure.txt`、`sources.lock`。

---

## 8. 修复潜在 BUG 的策略

“修复可能存在的 BUG”不能靠感觉，需要独立验证层：

1. **固定上游**：fork 并记录 FFmpeg commit、FFmpeg-Builds commit、每个 `scripts.d` 的 `SCRIPT_COMMIT`。
2. **补丁目录**：所有 patch 放 `patches/`，写明 issue、复现、测试、上游 PR 链接；不直接改 vendor 源码。
3. **FATE**：每个平台跑 FFmpeg FATE；失败不得发布。
4. **能力矩阵**：`ffmpeg -buildconf/-decoders/-encoders/-filters/-hwaccels/-bsfs` 与预期 TOML 对比。
5. **归档语料**：真实 30 min+ 文件、多音轨/字幕/章节/附件、10-bit/HDR、VFR、老编码、现代编码、损坏容器。
6. **全流程 smoke**：`ffprobe → ffmpeg 处理 → ffprobe/ffmpeg -f null → ffplay 可选`。
7. **Sanitizer 构建**：单独 ASAN/UBSAN build 跑 malformed input 与 damage 输出；不进 release。
8. **Fuzz**：对 demuxer/parser/BSF 输入做 fuzz，尤其是旧项目要处理的损坏码流。
9. **性能回归**：criterion / Google Benchmark / FFmpeg bench，记录吞吐、RSS、启动时间。
10. **上游回馈**：能复现的 FFmpeg/FFmpeg-Builds bug 提 issue/PR；本地 patch 标过期条件。

---

## 9. 构建产物与验收

### 9.1 自定义变体命名

建议新增：

```text
variants/linux64-vd-nonfree.sh
variants/win64-vd-nonfree.sh
addins/v3.sh
addins/lto-clang.sh
addins/openmp-audit.sh
addins/pgo.sh
```

产物名包含：

```text
ffmpeg-<ffmpeg-tag>-<commit>-<target>-<compiler>-<v3>-<lto>-<pgo>-<build-id>
```

### 9.2 每个产物必须包含

```text
bin/ffmpeg
bin/ffprobe
bin/ffplay
buildconf.txt
configure.txt
sources.lock
licenses.toml
capabilities.toml
fate-summary.txt
SHA256SUMS
```

### 9.3 项目侧版本门禁

VideoDamage/LAE 不应只看 `ffmpeg -version` 字符串，应校验：

- 三个二进制同目录；
- `ffmpeg -buildconf` 包含预期 `--cpu=x86-64-v3`、`--enable-gpl`、`--enable-nonfree`、硬件/字幕/滤镜开关；
- `--extra-version=lae-vd-<build-id>`；
- 三件套 banner 一致；
- `SHA256SUMS` 与本地记录一致；
- Manifest 写入 build ID、编译器、configure hash、CPU 基线。

### 9.4 验收矩阵

| 项 | 验收 |
|---|---|
| 功能 | FATE + 能力矩阵 + 全流复制 + 老/新编码 decode/encode smoke |
| 全流保留 | 4 流输入 4 流输出，逐流 hash 一致 |
| 确定性 | 同 trio + 同输入 + 同 plan → 同输出 |
| 性能 | 2 GB H.264 扫描/damage 基准不劣于基线；记录 RSS |
| 稳定性 | 超时/取消/进程树/临时文件清理 |
| 安全 | ASAN/UBSAN、fuzz 无高危问题 |
| 双平台 | Windows x64 + Linux x86_64 均通过 |
| 许可证 | nonfree 不公开；GPL 产物有完整许可证 |

---

## 10. 风险清单

| 风险 | 等级 | 对策 |
|---|---|---|
| `nonfree` 不可分发 | 高 | 私有制品库；另做 GPL 公开构建 |
| 上游 `latest` 不可复现 | 高 | 固定 tag/commit，build ID |
| Windows LTO broken | 高 | 不直接使用上游 lto addin；分平台实现 |
| MSVC/VS2026 与上游脚本不兼容 | 高 | 先 MinGW 基线，再评估 clang-cl；VS2026 用于本项目 |
| Clang 替换影响 144 个依赖 | 中高 | 逐依赖验证、锁 commit、FATE |
| `-march=x86-64-v3` 导致老 CPU 崩溃 | 中 | 明确无兼容；启动 CPUID 检查 |
| FMA/AVX2 改变浮点结果 | 中 | 禁用 fast-math；golden output 验证 |
| OpenMP 超订 | 中 | 每依赖 A/B；全局线程预算 |
| PGO 训练偏差 | 中 | M4 再做，保留非 PGO 基线 |
| 硬件加速运行时不可用 | 中 | 编译期矩阵 + 运行期驱动检测分开记录 |
| 自定义 patch 长期漂移 | 中 | patch 台账 + 上游 PR + 定期 rebase |

---

## 11. 分阶段路线图

### M0：固定基线

- fork `FFmpeg-Builds/FFmpeg-Builds`；
- 固定 FFmpeg `n9.0.2` 或 master commit；
- 固定所有 `scripts.d` commit；
- 建立 `linux64-vd-nonfree` / `win64-vd-nonfree`；
- 产物包含 buildconf / configure / licenses / SHA256。

### M1：功能全覆盖

- 不加 `--disable-everything`；
- 启用全部内建组件；
- 编入 upstream `zz-final.sh` 全部依赖；
- 硬件/字幕/滤镜能力矩阵；
- FATE + 真实语料 + 全流复制。

### M2：Linux Clang + v3

- 全依赖切 Clang 23 + lld + llvm-ar；
- `--cpu=x86-64-v3`；
- ThinLTO 实验；
- OpenMP 逐依赖审计；
- 性能回归。

### M3：Windows 工具链

- 基线：MinGW-w64 + v3；
- 实验：`clang-cl` + VS2026 SDK + lld-link；
- 对比 FATE、性能、`buildconf`、崩溃率；
- 决定生产 Windows trio 工具链。

### M4：深度优化

- PGO；
- BOLT（可选）；
- `-fno-semantic-interposition` / visibility / hardcoded tables；
- 线程预算与 FFmpeg `-threads` 调优；
- 成果写入基准报告。

### M5：BUG 修复与上游回馈

- sanitizer / fuzz；
- patch 台账；
- upstream PR；
- 每月重新 pin 一次；
- 每次 trio 更新必须重新跑能力矩阵和 golden output。

---

## 12. 结论（未来方案）

> 注意：以下结论仅适用于将来自构建重新启动时；当前不实施。当前实施依据是 [`FFMPEG_PROVIDER_REVIEW.md`](FFMPEG_PROVIDER_REVIEW.md)。

用户的目标可以成立，但需要把“编译 FFmpeg”升级为“维护一条 FFmpeg 发行工程线”。

最关键的四条判断：

1. **不能直接用 upstream floating `latest`**：必须固定 FFmpeg/FFmpeg-Builds/依赖 commit，生成 build ID。
2. **不能直接用 upstream `lto` addin 当 Windows 生产方案**：README 已标注 Windows broken；需分平台重做 LTO。
3. **OpenMP 不是 FFmpeg configure 开关**：只能逐依赖启用并做 A/B，FFmpeg 核心仍以 pthreads/w32threads 为主。
4. **VS2026 与 FFmpeg 编译器不需要 ABI 一致**：本项目是进程边界调用三件套；VS2026 用于本项目是硬要求，FFmpeg 本体走 MinGW-w64 或 clang-cl 才是工程现实。

推荐最终形态：

```text
lae-ffmpeg-build（独立构建仓库）
  ├─ 固定 FFmpeg n9.0.2 / master commit
  ├─ 固定 FFmpeg-Builds fork + scripts.d commits
  ├─ linux64-vd-nonfree：Clang 23 + x86-64-v3 + ThinLTO
  ├─ win64-vd-nonfree：MinGW-w64 基线 → clang-cl/VS2026 SDK 评估
  ├─ nonfree 私有制品库
  ├─ gpl 公开构建
  └─ buildconf / capabilities / FATE / SHA256 / licenses 全量归档
        │
        ▼
LAE v8 与 VideoDamage 共享同一 build ID 的 ffmpeg/ffprobe/ffplay
```

本报告即完整构建方案；架构上下文见 [`ARCHITECTURE_REPORT.md`](ARCHITECTURE_REPORT.md)，结论见 [`EXECUTIVE_SUMMARY.md`](EXECUTIVE_SUMMARY.md)。

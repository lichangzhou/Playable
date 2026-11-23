> **本文件是公开版移植规格。** 它描述 PlayNative 内核的平台隔离方式、四个钩子的
> 平台实现要求，以及 Android UI 的完整复刻口径（颜色值、字号、动画参数）。
>
> 想自己搭一个播放器的话，这份文档就是入口。原始工程位于
> `Playable-pn-android/07-porting/PORTING-AND-UI-SPEC.md`。

# PlayNative 内核移植 + Android UI 1:1 复刻文档

> 整理日期：2026-11-17
> **核验基线：2026-11-23 对 Android 端 v56（1.1.64）逐行核验并修正**（行号断言、价格、授权地址、
> UI 双形态路径均以该基线为准；此后 Android 版本推进时，行号断言需重新核验）。
> 适用对象：Windows / iOS（含 tvOS）端移植 PlayNative 内核，并 1:1 复刻 Android 版
> （`com.playable.player`，内核名 PlayNative / pn-crossport）整体 UI。
> 本文档合并两部分：**Part 1 内核移植清单**（平台无关层 / 需替换钩子）+
> **Part 2 安卓 UI 复刻规范**（布局、开机动画、LOGO、全部页面内容与颜色、含全部功能）。
> 所有颜色色值、字号、动画参数均取自 Android 端源码实测（Theme.kt / LogoIntro.kt /
> AndroidManifest.xml / themes.xml / ic_launcher_bg.xml）。

---

# Part 1 内核移植清单（其他端怎么对接核心）

## 0. 总体结论（先读这段）

这套内核**架构上已经做了平台隔离**，不是"Android 专用代码硬塞"，移植成本比重新写低很多。
隔离手段是**四个不透明句柄 / 回调接口**，核心层只认接口、不认平台：

| 接口 | 头文件 | 作用 | Android 后端 | Windows 需替换为 | iOS 需替换为 |
|------|--------|------|--------------|------------------|--------------|
| `pn_surface` | `core/render/surface.h` | 视频上屏（lock/写入/提交） | ANativeWindow | HWND + GPU 纹理 / DirectShow Surface | CVPixelBuffer / CALayer |
| `pn_audio_out` | `core/audio/audio_out.h` | 音频输出（PCM/IEC61937 写入） | AAudio | WASAPI / PortAudio | AudioUnit / AVAudioEngine |
| `HwDecoder` | `hwdec/hwdec.h` | 硬件解码（send/receive/flush） | MediaCodec | D3D11VA / NVDEC | VideoToolbox |
| `pn_platform` | `platform/*/pn_platform.c` | 日志回调 + 调试开关 | logcat + setprop | stderr/文件日志 + 环境变量 | os_log + UserDefaults |

**核心层（player/render/subtitle/demux/decode/audio）不包含任何 Android SDK 头文件**，
编译期靠 `pn_surface` / `pn_audio_out` / `HwDecoder` 三个不透明指针 + `pn_platform.c`
唯一允许包含 SDK 头文件的约定来隔离。因此：

- **平台无关层**：直接拷贝，无需改动（仅确认 FFmpeg / libass / libswscale 依赖在目标平台有等价构建）。
- **需替换钩子**：核心层调用的是接口，移植时只需为 Windows / iOS **各写一份接口实现**，核心层一行不改。

移植工作量集中在"四个钩子的后端实现"+"上层 UI 壳（Kotlin → 各平台 UI 框架）"，
内核调度逻辑（上屏节奏钟、音画同步、背压、PGS 兜底、HDR/DV 路由）**可原样复用**。

---

## 1. 平台无关层（可直接移植，无需改逻辑）

以下文件是本轮所有调优的落点，**逻辑全部平台无关**，移植时整体拷贝即可。

### 1.1 调度核心 — `core/state/player.c`（补丁 `fix_player_v9.c/.h`）

**职责**：播放状态机、上屏节奏钟、音画同步、阻塞式背压、音频节流。

**关键机制（移植时必须原样保留，这些是卡顿/不同步的根因修复）**：

| 机制 | 说明 | 移植注意 |
|------|------|----------|
| **上屏节奏钟** | `s_next_pres_ms += fd`，绝不用帧 PTS；迟到 >0.6 帧则重锚 `s_next_pres_ms = nowm + fd`，不补帧 | 依赖 `CLOCK_MONOTONIC`，Windows 用 `QueryPerformanceCounter`，iOS 用 `mach_absolute_time`，语义等价即可 |
| **取帧提前量** | `max(40ms, 1.5×frame_dur)`，因 4K NV12→RGBA sws_scale 单帧 17.29ms | 若目标平台 sws_scale 耗时不同，需重新实测这个闸门余量，但逻辑不变 |
| **单帧 in-flight** | `s_prep_us` 标记 scale 已完成，到点只 memcpy + unlockAndPost | 与渲染层配合，无需平台相关改动 |
| **阻塞式背压** | `p->v_cond` + `pthread_cond_timedwait`；`v_pending[24]` 深拷贝入队 | 依赖 pthread；Windows 用 `condition_variable`（C++）或 `SRWCONDITION`，iOS 原生支持 pthread，改动极小 |
| **音画同步（主时钟）** | 以 `audio_ctx.fifo_read_pos`（声卡已消费采样）为主时钟；EMA 0.995、死区 120ms、一次性整改 60% 上限 60ms、偏差 >400ms 才 resync | **强依赖音频后端能提供"已消费采样数"**。见 §2.2 钩子要求，这是移植时唯一需要后端配合的点 |
| **禁用容器 pts / 帧数×帧时长** | 因 Emby 分片转码 pts 跳变、开环不收敛 | 逻辑已定，无需平台相关改动 |
| **音频节流** | `win_us = is_audio_pkt ? 400000 : 3000000`，时基 `pos_est` | 逻辑不变 |
| **日志** | `PERF draw` 直方图、`avsync: drift=... ema=...`、`enq: frames=...`，周期 10s | 走 `pn_log`，由平台层路由，无需改动 |

**平台依赖检查**：仅 `<pthread.h>`、`<time.h>`（`clock_gettime`）、FFmpeg 类型。无 Android API。
**移植动作**：直接拷贝。pthread 在 iOS 原生支持；Windows 若用 MSVC，需把 pthread 调用
换成标准线程原语（见 §3 平台原语对照）。

### 1.2 渲染层 — `core/render/render.c`（补丁 `fix_render_v8.c/.h`）

**职责**：sws 缩放、几何降采样、HDR/SDR dataspace、PGS 字幕合成、上屏。

**关键机制**：

| 机制 | 说明 | 移植注意 |
|------|------|----------|
| **FIX29 sws 缓存回写** | `ctx->sws_ctx = (pn_handle)sws;`，否则每帧泄漏+重建（自动退出根因） | 纯逻辑，直接保留 |
| **FIX30 几何降采样** | CPU 只产视频原生尺寸 buffer，由 SurfaceFlinger 硬件放大；在 `render_draw` lock **之前**设 geometry | **平台相关**：依赖"合成器硬件放大"。Windows 用 GPU 纹理缩放（D3D11/DWrite/或 CPU 缩放到目标尺寸），iOS 用 CALayer 的 contentsScale。若目标平台合成器不提供等价硬件放大，需改为 CPU/GPU 显式缩放到窗口尺寸，逻辑不变但实现位置不同 |
| **HDR 懒升级** | SDR 默认 RGBA_8888；HDR 帧懒升级 RGBA_1010102 + `cur_dataspace=0` 强制下发；失败计数最多 5 次 | 依赖 `pn_surface_set_geometry`/`pn_surface_set_dataspace` 钩子返回真实支持情况，核心逻辑不变 |
| **Surface 重建清缓存** | `render_set_surface` 清 `cur_dataspace/view_w/geom_w` | 逻辑不变 |
| **PGS 上屏回读探针** | `render_blend_pgs` 落笔统计 `drawn=%d px` | 纯逻辑 |

**平台依赖检查**：通过 `pn_surface_*` 接口操作上屏，**不直接调 ANativeWindow**。
**移植动作**：核心逻辑直接拷贝；`render_draw` 里的 geometry/dataspace 调用走 §2.1 钩子。

### 1.3 字幕层 — `core/subtitle/subtitle.c`（补丁 `fix_subtitle_v3.c/.h`）

**职责**：libass 渲染 ASS/SSA/SRT、PGS 位图字幕兜底。

**关键机制**：

| 机制 | 说明 | 移植注意 |
|------|------|----------|
| **PGS 兜底窗口** | 解码时刻按单调钟记截止时刻，与 pts 窗口取或：`now <= pgs_end_us || now <= pgs_fallback_end_us` | 依赖单调钟，逻辑不变 |
| **libass 渲染** | `ass_library_init`、`ass_render`、消息回调路由到 `pn_log` | libass 跨平台（Windows/iOS 均有构建），字体查找路径需按平台配置（fontconfig / 手动指定字体目录） |

**平台依赖检查**：libass + 单调钟。无 Android API。
**移植动作**：直接拷贝；字体目录配置按平台调整（见 §3）。

### 1.4 解复用层 — `core/net/demux.c`（补丁 `fix_demux.c`）

**职责**：FFmpeg `avformat_open_input`、seek、音轨/字幕流选择。

**关键机制**：

| 机制 | 说明 | 移植注意 |
|------|------|----------|
| **音频流打分择优** | EAC3=11 / AC3=10 / AAC=9 / DTS=5 / TRUEHD=1 | 纯逻辑 |
| **seek 修正** | `AVSEEK_FLAG_BACKWARD`、time_base 单位换算（TS 1/90000 防偏 90 倍）、`fmt_lock` 串行化 | `fmt_lock` 用 pthread mutex，跨平台等价替换 |
| **字幕流优先文本** | 优先 ASS/SSA/SRT，跳过 PGS 位图 | 纯逻辑 |
| **HTTP/302** | 依赖 FFmpeg 内建 HTTP 协议（`avformat_open_input` 自动跟随 302） | **FFmpeg 的 http 协议跨平台一致**，Windows/iOS 构建的 FFmpeg 含同样 http 处理。见 §4 关于 302 的说明 |

**平台依赖检查**：FFmpeg libavformat + pthread mutex。无 Android API。
**移植动作**：直接拷贝。

### 1.5 解码层 — `core/decode/decode.c`（补丁 `fix_decode_v1.c`）

**职责**：音视频解码，硬解走 `hwdec.h` 抽象，软解走 avcodec。

**关键机制**：

| 机制 | 说明 | 移植注意 |
|------|------|----------|
| **硬解抽象** | 不再内联 MediaCodec，改调 `hwdec_supports/hwdec_create/hwdec_send_packet/hwdec_receive_frame/hwdec_flush/hwdec_destroy` | **核心层只认 `HwDecoder` 接口**，移植时提供 Windows/iOS 的 hwdec 实现即可（§2.3） |
| **音频解码器用 `d->audio_stream`** | 修复"完全无声"（demux 选 AC3，decode 却用 TrueHD 建解码器） | 纯逻辑 |
| **DV 前置 NAL 扫描** | `dvpass_try_open` 前置扫 NAL type 62 | 纯逻辑，DV passthrough 见 §2.4 |
| **HDR SEI 提取** | MediaCodec 不解析 SEI HDR，需手动提取 | 纯逻辑 |

**平台依赖检查**：FFmpeg libavcodec + `hwdec.h` 抽象接口。无 Android API。
**移植动作**：直接拷贝；链接目标平台的 hwdec 实现。

### 1.6 音频层 — `core/audio/audio.c`（补丁 `fix_audio_v1.c/.h`）

**职责**：PCM/IEC61937 FIFO、音频输出、音画同步主时钟的提供方。

**关键机制**：

| 机制 | 说明 | 移植注意 |
|------|------|----------|
| **FIFO 水位** | `fifo_write_pos`/`fifo_read_pos`，`s_a_fifo_peak` 峰值 | `fifo_read_pos`（声卡已消费）是音画同步主时钟，**必须由音频后端提供真实消费位置**（§2.2） |
| **音频输出抽象** | `pn_audio_out_create/open/write/request_stop/buffer_size/close` | 走 `pn_audio_out` 接口，核心层不感知 AAudio |
| **IEC61937 透传** | SPDIF 比特流直通 | 依赖后端支持 IEC61937 格式（AAudio API34+、WASAPI 需特殊处理、AudioUnit 需配置） |

**平台依赖检查**：`pn_audio_out.h` 抽象接口。无 Android API。
**移植动作**：直接拷贝；提供目标平台 `pn_audio_out` 实现（§2.2）。

---

## 2. 需替换钩子（平台适配层，核心层调用、各平台各写一份实现）

以下四个钩子是移植的**主要工作量**。核心层只通过接口调用，移植时**为 Windows / iOS
各实现一份**，核心层无需改动。

### 2.1 钩子一：`pn_surface`（视频上屏）— `core/render/surface.h`

**核心层怎么用**：`render_draw` 里调 `pn_surface_set_geometry` / `pn_surface_lock` /
`pn_surface_unlock_post` / `pn_surface_set_dataspace` / `pn_surface_hdr10_available`，
并通过 `pn_surface_from_window(native_handle)` 把平台原生窗口包装成 `pn_surface`。

**需要实现的函数**：

| 函数 | 语义 | Android 参考 | Windows 实现要点 | iOS 实现要点 |
|------|------|--------------|------------------|--------------|
| `pn_surface_from_window` | 包装原生窗口 | ANativeWindow* | 从 HWND/ID3D11Texture2D 构造 | 从 CVPixelBuffer/CALayer 构造 |
| `pn_surface_set_geometry` | 设宽高+像素格式，返回是否 HDR 就绪 | ANativeWindow_setBuffersGeometry | D3D11 texture 尺寸/格式；返回 HDR 支持情况 | CVPixelBuffer 格式（RGBA_1010102） |
| `pn_surface_lock` | 锁定缓冲，返回 `bits/stride/height/format` | ANativeWindow_lock | 锁定纹理/共享内存，返回 stride | CVPixelBufferLock |
| `pn_surface_unlock_post` | 解锁并提交上屏 | unlockAndPost | 解锁纹理 + Present | CVPixelBufferUnlock + CALayer 提交 |
| `pn_surface_set_dataspace` | 切 HDR/SDR dataspace | ADATASPACE 配置 | 按显示器 HDR 能力设置 | 设置 CALayer/像素的 HDR 属性 |
| `pn_surface_hdr10_available` | 查询 HDR10 支持 | 探测 surface 能力 | 探测显示器/纹理能力 | 探测设备 HDR 能力 |

**格式常量已归一化**：`PN_SURF_FMT_RGBA_8888/RGBX_8888/RGB_565/RGBA_1010102`、
`PN_SURF_DS_BT2020_PQ/SRGB`（数值对齐 Android 实测值，Android 后端直接透传）。
**移植要求**：Windows/iOS 后端需把原生格式**映射到这些常量**，核心层格式判定逻辑不变。

**移植风险点**：
- **几何降采样（FIX30）依赖"合成器硬件放大"**。Android 靠 SurfaceFlinger；Windows 需用
  GPU 纹理缩放（D3D11 SamplerState 或计算着色器），iOS 用 CALayer contentsScale。
  若目标平台合成器不自动放大，需在 `render_draw` 里显式缩放到窗口尺寸（逻辑不变，实现位置不同）。
- **HDR 懒升级**依赖 `set_geometry` 返回真实支持情况，后端必须如实返回 0/1/-1，否则 HDR 判定失效。

### 2.2 钩子二：`pn_audio_out`（音频输出）— `core/audio/audio_out.h`

**核心层怎么用**：`audio.c` 调 `pn_audio_out_create/open/write/request_stop/buffer_size/close`，
并通过 **`fifo_read_pos`（声卡已消费采样数）作为音画同步主时钟**。

**需要实现的函数**：

| 函数 | 语义 | Android 参考 | Windows 实现要点 | iOS 实现要点 |
|------|------|--------------|------------------|--------------|
| `pn_audio_out_create/destroy` | 分配/释放句柄 | AAudioStream 封装 | WASAPI IAudioClient 封装 | AudioUnit 封装 |
| `pn_audio_out_open` | 开流（声道/采样率/格式） | AAudioStreamBuilder | IAudioClient 初始化 + 格式协商 | AudioUnit 初始化 + 格式配置 |
| `pn_audio_out_write` | 写入（阻塞到写完或超时） | AAudioStream_write | IAudioClient 写入缓冲 + 超时控制 | AudioUnit 写入 + 超时 |
| `pn_audio_out_request_stop` | 非阻塞请求停止 | AAudioStream_requestStop | 停止流 + 唤醒线程 | 停止 Unit |
| `pn_audio_out_buffer_size` | 返回内部缓冲帧数 | AAudioStream_getBufferSizeInFrames | WASAPI 缓冲帧数 | AudioUnit 缓冲帧数 |
| **`fifo_read_pos`（音画同步主时钟）** | **声卡已消费的采样数** | AAudio 实际消费位置 | WASAPI 用 `IAudioClient::GetCurrentPadding` + 已写入累计，算出已消费采样 | AudioUnit 用读位置/已消费计数 |

**⚠️ 音画同步强依赖此钩子**：`player.c` 的音画同步以 `fifo_read_pos` 为主时钟。
**移植时后端必须提供真实、稳定的"已消费采样数"**，否则音画同步（FIX25~FIX28、FIX31）
在目标平台会失效。这是移植时最需要后端配合的一点。

**格式常量已归一化**：`PN_AUDIO_FMT_PCM_I16/PCM_FLOAT/IEC61937`（对齐 AAudio 枚举）。
**移植要求**：Windows/iOS 后端映射原生格式到这些常量。

**移植风险点**：
- **IEC61937 透传**（SPDIF）：AAudio API34+ 支持；WASAPI 需特殊处理（IEC61937 直通非标准）；
  AudioUnit 需配置直通格式。若目标平台不支持，需回退为 PCM 解码（逻辑上 IEC61937 是可选路径）。
- **缓冲大小**影响音画同步稳定性：Android 实测 `buffer_size × 2` 作为 FIFO 容量，
  Windows/iOS 需按各自缓冲帧数实测调整，逻辑不变。

### 2.3 钩子三：`HwDecoder`（硬件解码）— `hwdec/hwdec.h`

**核心层怎么用**：`decode.c` 调 `hwdec_supports/create/configure/send_packet/receive_frame/
flush/is_hw_active/destroy`，不感知具体后端。

**需要实现的函数**（接口已定义，移植时填后端）：

| 函数 | 语义 | Android 参考 | Windows 实现要点 | iOS 实现要点 |
|------|------|--------------|------------------|--------------|
| `hwdec_supports` | 查 codec_id 是否支持硬解 | 查 `h264_mediacodec` 等 + `avcodec_get_hw_config` | 查 D3D11VA/NVDEC 编码器可用性 | 查 VideoToolbox 编码器可用性 |
| `hwdec_create` | 建硬解器，挂 hw_device_ctx | MediaCodec + av_hwdevice_ctx | D3D11VA + av_hwdevice_ctx（hwcontext_d3d11va） | VideoToolbox + av_hwdevice_ctx（hwcontext_videotoolbox） |
| `hwdec_configure` | 同步流参数（宽高/像素格式/CSD） | setParameters | avcodec_parameters_to_context | 同 |
| `hwdec_send_packet/receive_frame` | 送包/收帧（处理 EAGAIN） | MediaCodec input/output buffer | avcodec_send/receive_frame | 同 |
| `hwdec_flush` | seek 前刷缓冲 | avcodec_flush_buffers | 同 | 同 |
| `hwdec_is_hw_active` | 确认真在硬解 | 检查实际状态 | 检查 D3D11VA 是否激活 | 检查 VideoToolbox 是否激活 |
| `hwdec_destroy` | 释放 | MediaCodec release | avcodec_free_context | VTDecompressionSession invalidate |

**移植动作**：接口已定义好，**为 Windows 写 `hwdec_d3d11va.c`（或 NVDEC），为 iOS 写
`hwdec_videotoolbox.c`**，实现上述函数即可。FFmpeg 已内置 `hwcontext_d3d11va` /
`hwcontext_videotoolbox`，主要工作是把 MediaCodec 的 buffer 管理逻辑换成对应后端。

**移植风险点**：
- **硬解输出像素格式不同**：MediaCodec 不挂 Surface 时输出 8-bit NV12，会绕过 10-bit DV
  转换与 tone-map。D3D11VA/VideoToolbox 的输出格式需核对，确保 10-bit HDR/DV 链路不被绕过。
- **`pn_debug_swdec_forced`**：Android 用 adb setprop 强制软解做 A/B 验证；Windows/iOS
  用环境变量/配置替代（见 §2.4）。

### 2.4 钩子四：`pn_platform`（日志 + 调试开关）— `platform/*/pn_platform.c`

**职责**：唯一允许包含平台 SDK 头文件的文件；配置 `pn_log` 日志回调 + 实现 4 个调试开关。

**需要实现的函数**：

| 函数 | 语义 | Android 参考 | Windows 实现要点 | iOS 实现要点 |
|------|------|--------------|------------------|--------------|
| `pn_platform_init` | 设日志回调 | logcat（`__android_log_print`） | stderr/文件日志（`fprintf`） | os_log（`OS_LOG`） |
| `pn_debug_dvpass_forced_off` | 强制关 DV 直通（调试） | `adb setprop debug.playable.dvpass 0` | 环境变量 `PN_DEBUG_DVASS_OFF` | UserDefaults / 环境变量 |
| `pn_debug_sdr_forced` | 强制回退 SDR（调试） | `adb setprop debug.playable.sdr 1` | 环境变量 `PN_DEBUG_SDR` | 同上 |
| `pn_debug_swdec_forced` | 强制软解（调试） | `adb setprop debug.playable.swdec 1` | 环境变量 `PN_DEBUG_SWDEC` | 同上 |
| `pn_debug_blacklift_disabled` | 关黑位提升（调试） | `adb setprop debug.playable.blacklift 0` | 环境变量 `PN_DEBUG_BLACKLIFT` | 同上 |

**移植动作**：`bm_plat_pc.c`（Windows/Linux 骨架）和 `bm_plat_apple.c`（Apple 骨架）
**已经写好**，只需把 4 个调试开关从"恒返回 0"改成读取环境变量/配置即可。日志回调已提供
stderr 版本，iOS 可换 os_log。

**移植注意**：这 4 个开关是 A/B 验证的关键（DV 直通、SDR 回退、软解对比、黑位提升），
移植时务必保留可配置入口，否则目标平台出问题时无法用同一套 A/B 方法定位。

---

## 3. 平台原语与构建对照

核心层用到的跨平台原语，在 Windows / iOS 的等价实现：

| 原语 | Android/iOS 原生 | Windows 等价 | 说明 |
|------|------------------|--------------|------|
| `pthread_mutex_t` | 原生支持 | `CRITICAL_SECTION` 或 `std::mutex` | FFmpeg 在 Windows 已抽象，核心层若直接用 pthread 需替换 |
| `pthread_cond_t` + `pthread_cond_timedwait` | 原生支持 | `SRWCONDITION` / `condition_variable` | **背压队列（v_cond）依赖此**，需等价实现 |
| `clock_gettime(CLOCK_MONOTONIC)` | 原生支持 | `QueryPerformanceCounter` | 上屏节奏钟、PGS 兜底依赖单调钟 |
| 线程创建/join | `pthread_create` | `_beginthreadex` / `std::thread` | 3 个 worker 线程（demux/decode/audio） |
| 字体查找 | fontconfig（`ensureFontconfig`） | 手动指定字体目录 / DirectWrite | libass 渲染依赖字体，需配置字体路径 |

**构建对照**：

| 项 | Android 现状 | Windows | iOS |
|----|--------------|---------|-----|
| 构建系统 | CMake + ninja（4 ABI） | CMake + MSVC / ninja | CMake + Xcode / Clang |
| FFmpeg | `D:/ffmpeg-src`（4 套 ABI） | 需构建含 `hwcontext_d3d11va` 的 FFmpeg | 需构建含 `hwcontext_videotoolbox` 的 FFmpeg |
| 第三方 | libass / libplacebo / FFmpeg | 同（跨平台） | 同（跨平台） |
| 产物 | `libplaynative.so`（多 ABI） | `playnative.dll` / `.lib` | `PlayNative.framework` / 静态库 |

---

## 4. 关于 302 直连（用户关注点）

Emby 库影片全部是 **302 直连网盘**、不需要 Emby 硬解。

**结论（基于代码核查）**：302 处理**不在内核补丁范围内**，由 **FFmpeg 内建 HTTP 协议**
（`libavformat/http.c`）在 `avformat_open_input` 时自动跟随重定向。`fix_demux.c` 里
**没有任何自定义 302 逻辑**，说明 302 是 FFmpeg 底层透明处理的。

**移植影响**：
- FFmpeg 的 http 协议在 Windows / iOS 构建中**行为一致**，302 跟随逻辑无需移植改动。
- 若目标平台播放 302 直连出现跳位置/不同步，**先查 FFmpeg http 协议构建是否完整、
  Range 请求（seek）是否正确**，而非内核调度层——内核层收到的是已解复用的包，
  不感知 HTTP 层是否发生重定向。
- 此前记录的"Emby 换流 URL 时 `applyNewUrl → engine.seekTo()` 跳位置"属 **App/Kotlin 层**
  行为（见 §5），与 302 本身无直接关系，但 302 直连场景下换流更频繁，会放大该现象。

---

## 5. 上层 UI 壳（Kotlin/JNI，需按平台重写）

核心层之上的 Android 壳层是平台相关的，移植到 Windows / iOS 需**重写**（或提供等价封装）。
这部分不是内核逻辑，但承担与内核的桥接 + UI 交互。UI 部分的 1:1 复刻规范见 **Part 2**。

| 文件 | 职责 | 移植动作 |
|------|------|----------|
| `player/PlayNativeEngine.kt`（`fix_PlayNativeEngine.kt`） | JNI 桥接、`attachSurface`、`media_opened` 事件、轨道枚举 | Windows：用 C ABI 导出 + 上层调用；iOS：Objective-C++ / Swift 桥接同名 C API |
| `PlayNativePlayerActivity.kt`（`fix_PlayNativePlayerActivity.kt`） | 续播 seek、`applyNewUrl` 换流、PiP、输出切换 | 各平台 UI 框架重写播放器页面，续播 seek 时机（`media_opened` 后）逻辑保留 |
| `ProgressReporter.kt`（`fix_ProgressReporter.kt`） | 进度上报（续播/断点） | 各平台实现进度上报，逻辑保留 |
| `jni_bridge.c` | JNI 层，`pn_player_*` → Kotlin | Windows：改为 C ABI 导出函数；iOS：改为 C 函数供 Obj-C++ 调用 |

**关键事件契约（跨平台保留）**：内核通过 `PVN_CALLBACKS`（`on_info` / `on_state_changed`）
向上层上报事件，`media_opened` 是"已 open 成功、未出第一帧"的窗口，**续播 seek 必须在此
之后执行**。这个契约跨平台不变，各平台壳层按此契约对接即可。

---

## 6. 移植执行清单（按顺序）

### 阶段 A：平台无关层拷贝（无需改逻辑）
1. [ ] 拷贝 `core/state/player.c`（`fix_player_v9.c`）
2. [ ] 拷贝 `core/render/render.c`（`fix_render_v8.c`）
3. [ ] 拷贝 `core/subtitle/subtitle.c`（`fix_subtitle_v3.c`）
4. [ ] 拷贝 `core/net/demux.c`（`fix_demux.c`）
5. [ ] 拷贝 `core/decode/decode.c`（`fix_decode_v1.c`）
6. [ ] 拷贝 `core/audio/audio.c`（`fix_audio_v1.c`）
7. [ ] 拷贝头文件 `fix_player_v9.h` / `fix_render_v8.h` / `fix_subtitle_v3.h` 等
8. [ ] 确认 FFmpeg / libass / libswscale 在目标平台有等价构建

### 阶段 B：平台适配层实现（四个钩子）
9. [ ] **`pn_surface` 实现**（Windows：HWND+D3D11；iOS：CVPixelBuffer+CALayer）— §2.1
10. [ ] **`pn_audio_out` 实现**（Windows：WASAPI；iOS：AudioUnit）— §2.2，**务必提供 `fifo_read_pos`**
11. [ ] **`HwDecoder` 实现**（Windows：D3D11VA；iOS：VideoToolbox）— §2.3
12. [ ] **`pn_platform` 适配**：基于 `bm_plat_pc.c` / `bm_plat_apple.c`，把 4 个调试开关改为读环境变量/配置 — §2.4

### 阶段 C：平台原语替换
13. [ ] pthread mutex/cond → Windows 等价（背压队列 `v_cond` 优先）
14. [ ] `clock_gettime` → `QueryPerformanceCounter`
15. [ ] 线程创建 → Windows/iOS 等价
16. [ ] 字体路径配置（libass）

### 阶段 D：上层壳重写
17. [ ] `jni_bridge.c` → 目标平台 C ABI 导出
18. [ ] `PlayNativeEngine` 等价封装（attachSurface / media_opened / 轨道枚举）
19. [ ] 播放器页面（续播 seek 时机、换流处理、输出切换）— UI 规范见 Part 2 §13.3
20. [ ] 进度上报

### 阶段 E：验证（用同一套 A/B 方法）
21. [ ] 用 `pn_debug_*` 四个开关做 DV/SDR/软解/黑位 A/B 对比
22. [ ] 验证音画同步（`fifo_read_pos` 主时钟在目标平台收敛、不发散）
23. [ ] 验证卡顿（`PERF draw` 直方图、worst 倍率）
24. [ ] 验证 seek 不崩（`fmt_lock` 串行化生效）
25. [ ] 验证 PGS 字幕（兜底窗口）
26. [ ] 验证 HDR/DV 路由（`pn_surface_hdr10_available` / NAL 62 扫描）
27. [ ] 验证长时间播放不自动退出（sws_ctx 缓存、内存收敛）

---

## 7. 移植风险汇总（按优先级）

| 优先级 | 风险 | 说明 | 缓解 |
|--------|------|------|------|
| **高** | **`fifo_read_pos` 主时钟** | 音画同步强依赖音频后端提供真实已消费采样数，后端不提供则同步失效 | §2.2 后端实现务必提供；用 `avsync` 日志验证收敛 |
| **高** | **背压队列 `v_cond`** | 阻塞式背压依赖条件变量，Windows 原语替换若语义不等价会导致丢帧/卡顿 | 优先验证 pthread_cond_timedwait 等价实现 |
| **高** | **几何降采样依赖合成器放大** | FIX30 靠 SurfaceFlinger 硬件放大，Windows/iOS 合成器行为不同 | 核对目标平台是否自动放大，否则显式缩放 |
| **中** | **硬解输出格式绕过 10-bit 链路** | D3D11VA/VideoToolbox 输出格式不同，可能绕过 DV/tone-map | 用 `pn_debug_swdec_forced` 做软解 A/B 对比 |
| **中** | **IEC61937 透传** | WASAPI/AudioUnit 对 SPDIF 直通支持不一 | 不支持则回退 PCM 解码 |
| **中** | **字体查找** | libass 跨平台需配置字体路径 | 手动指定字体目录 |
| **低** | **302 直连** | FFmpeg http 协议跨平台一致，内核不感知 | 出问题先查 FFmpeg http 构建 + Range 请求 |
| **低** | **上层壳重写** | Kotlin/JNI 是平台相关，需重写 | 非内核逻辑，按事件契约对接 |

---
---

# Part 2 Android UI 1:1 复刻规范（含全部功能）

## 7.5 UI 双形态：手机/平板（v48）与电视（v54）— 交接必读

> **这是所有 UI 移植的起点，先读完这一节再动手。**
> 同一份 Compose 代码通过 `isTv` 分流成两套**完全不同**的外观，两端都要各自复刻，
> **不是二选一，也不是「TV 是手机的放大版」**。
>
> | | 基线版本 | APK |
> |---|---|---|
> | 手机 / 平板 | **v48**（1.1.56, versionCode 92, 2026-11-22） | `Playable-v48-1.1.56-20261122.apk` |
> | 电视 | **v54**（1.1.61, versionCode 97, 2026-11-23） | `Playable-v54-1.1.61-20261123.apk` |
> | 当前合并版（双形态共存） | v56（1.1.64, versionCode 100） | `Playable-v56-手机v48-TVv54-1.1.64-20261123.apk` |
>
> 基线来由（历史教训，务必知道）：v49~v54 做 TV 改造时把共享文件一起改了，
> 手机版 UI 被 TV 顶栏顶掉，导致所有平台的移植参考源全部出错。用户据此立红线：
> **手机 UI 以 v48 为准，TV UI 以 v54 为准，互不干扰。** 未来任何改动都按双基线校验。

### 7.5.1 判定链（唯一开关）

```
设置 → 播放设置 → 其它 → 「强制 TV 界面」开关      AppSettings.forceTvUi
                    ↓
        isTvDevice(ctx, forceTvUi)                  ui/tv/（设备判定 + 强制开关）
                    ↓
   ┌────────────────┴─────────────────┐
   isTv = true                        isTv = false
   TV 形态（v54）                      手机/平板形态（v48）
   RootScreen → PsTopBar              RootScreen → GlassBottomBar（6 页签）
   HomeScreen → TvHomeScreen          HomeScreen → PhoneHomeScreen
   LibraryScreen → 8 列 / 2000 条      LibraryScreen → Adaptive(118dp) / 400 条
   DetailScreen → TV 版式              DetailScreen → v48 版式
   PlayerControls → 焦点描边            PlayerControls → 纯触摸
```

**判断源必须全应用统一**：一律用响应式 settings（`vm.settings` 的 Flow /
`SettingsStore.appSettings`），不要用 `PlayableApp.cachedSettings` 这类同步快照。

> ⚠️ 实测坑（2026-11-23，v56 / 1.1.64 修复）：`RootScreen` 曾读 `cachedSettings` 快照、
> 首页读 `vm.settings` Flow。切换「强制 TV 界面」后首页已变成仪表盘、导航却还停在
> 手机玻璃底栏，出现 **「仪表盘 + 手机底栏」混合态**（既没有 PS5 顶栏，底部又挂着 6 个图标）。
> 修复 = RootScreen 改用 `vm.settings.collectAsStateWithLifecycle()`。
> 移植到其他端时，导航与内容必须读**同一个**状态源。

### 7.5.2 逐项对照表（移植时逐行对照）

| 维度 | 手机 / 平板（v48 基线） | 电视（v54 基线） | 代码入口 |
|------|------------------------|------------------|----------|
| **主导航** | 底部悬浮**玻璃页签栏**，6 个：媒体库 / Live / Music / 搜索 / 服务器(列表图标) / 设置 | **PS5 风格顶栏**（高 64dp）：3 页签（媒体库 / Live / Music）+ 时间 + 搜索 + 设置 + 服务器头像 | `RootScreen.kt`：`phoneTabs`(GlassBottomBar) ↔ `mainTabs`(PsTopBar) |
| **首页结构** | 「继续观看」横排 → 「我的媒体」3 个大卡横排 → 各媒体库横排行 | 仪表盘：顶排方形块（P 封面 + 库方块，可用 ‹ › 调序）→ 继续观看 → 统计卡 → 已挂载服务器 → 媒体库方块 | 调度器在 `HomeScreenPhone.kt` 尾部 → `TvHomeScreen` / `PhoneHomeScreen` |
| **统计卡** | **无** | **一排 4 个**：已挂载服务器 / 媒体库 / 电影 / 剧集·综艺；第二排：音乐·演唱会 / 电视台 | `PsDashboard(statColumns = if (isTv) 4 else 2)` |
| **媒体库卡** | 「我的媒体」3 张大卡横排（单列视觉） | **一排 4 个**库卡，带图标 + 库名 + 「20 项」计数 | `PsDashboard(libColumns = if (isTv) 4 else 1)` |
| **媒体库网格** | `GridCells.Adaptive(118.dp)`（竖屏约 5 列）+ `limit 400` | `GridCells.Fixed(8)` 固定 8 列 + `limit 2000`（要能滚到全部影片） | `LibraryScreen.kt`：`itemsLimit` / `columns` |
| **电影详情** | 整行 16:9 头图 → 标题信息 → 播放大按钮 → 音轨/字幕 → 操作图标行 → 简介 → 演职员 → 媒体信息（垫底） | 左侧玻璃信息卡（标题/元信息/播放/收藏/已看/评分）+ 右侧 16:9 剧照，左右并排 | `DetailScreen.kt` 电影 `if (isTv)` 分支 |
| **单集详情** | `EpisodeHeaderBlock`：剧照 + 集名 + 整行播放按钮 + 音轨/字幕 + 列表 | 左信息卡（`SxxExx · 集名` + 剧名 + `★评分·首播·时长` + 播放/收藏/已看/评分）+ 右 16:9 剧照 | `DetailScreen.kt` 单集 `if (isTv)` 分支 |
| **播放器控制条** | 触摸点击；圆按钮 1dp 半透明描边 | 遥控器可聚焦；聚焦时 **2.5dp 白描边 + 1.12 倍放大**；返回键第一下只亮控制条不退出 | `player/PlayerControls.kt`（`onFocusChanged`）/ `PlayNativePlayerActivity.kt`（BackHandler） |
| **Live 页** | 触摸频道列表，无特殊按键 | 遥控器 `CHANNEL_UP/DOWN` 换台 + 根容器可聚焦持焦（否则方向键无反应） | `live/ui/LiveScreen.kt` |
| **Music 页** | v48 简洁版（列表 + 播放条），**没有**顶部随机/循环/歌词/队列按钮 | 与手机一致（v54 的音乐播放页/队列面板改动已按红线回退，不属于 TV 形态） | `music/ui/MusicScreen.kt` |
| **字号/密度** | 系统默认（手机可读距离） | 720dp 压缩 + 更大字号（3 米观看距离可读） | `PsTopBar.kt` / `PsDashboard.kt` / `HomeScreenTv.kt` |
| **返回键** | 内容区返回 → 焦点收底栏；再按 Toast「再按一次退出」 | 内容区返回 → 焦点收顶栏页签；再按 Toast 退出 | `RootScreen.kt` BackHandler |
| **搜索/服务器/设置入口** | 底栏独立页签（第 4/5/6 个） | 顶栏图标（搜索 / 头像 / 齿轮），不占页签位 | `PsTopBar.kt` |

### 7.5.3 基准截图（2026-11-23 实测，同一台平板切开关拍摄）

**手机 / 平板形态（v48）**

| 首页：继续观看 + 我的媒体，底栏 6 图标 | 媒体库：自适应约 5 列 | 播放器：触摸控制条 |
|---|---|---|
| ![手机首页 v48](./shots/ui-phone-home-v48.png) | ![手机媒体库 v48](./shots/ui-phone-library-v48.png) | ![手机播放器 v48](./shots/ui-phone-player-v48.png) |

**电视形态（v54）**

| 首页：PS5 顶栏 + 仪表盘 + 统计卡一排 4 个 | 媒体库：固定 8 列 | 播放器：遥控器焦点白描边 |
|---|---|---|
| ![TV 首页 v54](./shots/ui-tv-home-v54.png) | ![TV 媒体库 v54](./shots/ui-tv-library-v54.png) | ![TV 播放器 v54](./shots/ui-tv-player-v54.png) |

> 截图目录：`07-porting/shots/`。六张图就是**验收基准**，改完 UI 逐张比对；
> 图不对就是改错了，不要凭记忆判断。

### 7.5.4 移植交接铁律（逐条打勾）

1. [ ] **两端都实现**：手机形态与电视形态各做一份，判据只有 `isTv`（或目标平台的等价"10 英尺场景"判定）。
2. [ ] **新增样式只写进 isTv 分支**；`isTv = false` 的默认路径必须与 v48 保持一致，禁止"顺手优化"。
3. [ ] **手机端验收**：底栏 6 个图标（含 Live / Music）、首页只有「继续观看 + 我的媒体」、**没有仪表盘、没有顶栏**。
4. [ ] **电视端验收**：有 PS5 顶栏、统计卡一排 4 个、媒体库一排 4 个、遥控器移动有白描边焦点。
5. [ ] **状态源统一**：导航与内容读同一个 settings 流，切换开关**来回两次**不出现混合态。
6. [ ] **六张基准图逐张比对**（`./shots/`），有任何一张不符先回滚再排查。
7. [ ] 手机端与电视端各自出包验证后再合并交付。

### 7.5.5 后续 Playable 文档的统一标注块（写新 MD 时直接复制）

**规则：之后任何 Playable 播放器的交接 / 移植 / 版本说明 / 审计文档，正文开头都要带下面这段，
保证读文档的人第一眼就知道 UI 是两套形态、各自基线是什么。**

```markdown
> **UI 双形态（必读）**：本项目 Android 端 UI 分两套，由 `isTv` 分流，互不干扰：
> - **手机 / 平板 = v48 基线**（1.1.56）：底部玻璃页签栏 6 个、首页「继续观看 + 我的媒体」、无仪表盘；
> - **电视 = v54 基线**（1.1.61）：PS5 顶栏、仪表盘、统计卡与媒体库均一排 4 个、遥控器焦点体系。
>
> 逐项对照表与六张基准截图见
> `Playable-pn-android/07-porting/PORTING-AND-UI-SPEC.md` §7.5（截图在 `07-porting/shots/`）。
> 改 UI 前先读该节；改完按六张基准图逐张比对，手机路径不得出现任何 TV 元素。
```

一句话版（空间紧张时用）：

```markdown
> UI 双形态：手机/平板 = v48（底栏 6 页签、继续观看+我的媒体），电视 = v54（PS5 顶栏 + 仪表盘 + 一排 4 个）；
> 详见 `07-porting/PORTING-AND-UI-SPEC.md` §7.5 与 `07-porting/shots/` 六张基准图。
```

---

## 8. UI 技术栈与工程结构

**技术栈**：Kotlin + Jetpack Compose（Material 3），无三方播放器内核；
开屏动画用系统框架 `VideoView`（不依赖 Media3 / ExoPlayer）。
目标其他端复刻时：Compose Multiplatform（桌面/iOS）可直接迁移 Composable；
非 Compose 平台按本文档的布局、色值、字号、文案逐页实现。

**UI 包结构**（`app/src/main/java/com/playable/player/ui/`）：

```
ui/
├── RootScreen.kt              根导航（Tab 容器 + 开屏动画挂载）
├── HomeScreenPhone.kt        手机/平板首页（继续观看 + 媒体库入口；原 HomeScreen.kt 已拆分）
├── HomeScreenTv.kt           电视首页（仪表盘形态）
├── LibraryScreen.kt          媒体库（海报墙 + 筛选/排序）
├── DetailScreen.kt           详情页（片名/海报/简介/操作区）
├── EpisodeDetail.kt          分集列表（季/集选择）
├── SearchScreen.kt           搜索
├── ServersScreen.kt          媒体服务器管理（多客户端同步媒体服务器）
├── PersonScreen.kt           人物页（演员/导演作品）
├── CollectionScreen.kt       合集页
├── LogoIntro.kt              开机动画
├── TrackPickSheet.kt         音轨/字幕轨道选择弹层
├── LibraryFilterSheet.kt     媒体库筛选弹层
├── LibrarySortSheet.kt       媒体库排序弹层
├── LibraryPicker.kt          媒体库选择器
├── CastSheetHost.kt          DLNA 投屏弹层宿主
├── UpdatePrompt.kt           OTA 更新提示
├── Components.kt             通用组件（glass 玻璃材质、卡片、行等）
├── DetailActions.kt          详情页操作按钮组（播放/收藏等）
├── DetailExtras.kt           详情页附加信息区
├── HomeLibraryRow.kt         首页媒体库行
├── LibraryCoverCard.kt       媒体库海报卡片
├── LibraryExtras.kt          媒体库附加信息
├── MediaInfoCard.kt          媒体信息卡（分辨率/编码/音轨）
├── EpisodeCardsRow.kt        分集卡片行
├── SeasonCardsRow.kt         分季卡片行
├── ServerTypeUi.kt           服务器类型界面
├── settings/
│   ├── SettingsScreen.kt     设置主页
│   ├── AppearanceSettingsScreen.kt  外观（主题/强调色）
│   ├── PlaybackSettingsScreen.kt    播放设置
│   ├── SubtitleSettingsScreen.kt    字幕设置
│   ├── VideoAudioSettingsScreen.kt  视频/音频设置
│   ├── NetworkSettingsScreen.kt     网络设置
│   ├── LicenseScreen.kt            付费与授权（PRO 会员）
│   ├── AboutScreen.kt              关于
│   └── SubtitleSearchSheet.kt      字幕搜索弹层
└── tv/
    ├── TvSidebar.kt          TV 侧边栏导航
    └── TvFocus.kt            TV 焦点态组件
```

**资源结构**（`app/src/main/res/`）：`drawable`（含 `ic_launcher_bg.xml`）、
`drawable-nodpi` / `drawable-xhdpi`、`layout`、`mipmap-*`（多密度启动图标）、
`raw`（`logo_intro.mp4` 开屏成片）、`values`（`strings.xml` / `themes.xml`）、`xml`。

---

## 9. 主题系统（颜色 / 强调色 / 玻璃材质 / 字号）

### 9.1 PlayableTheme 入口

```kotlin
PlayableTheme(
    theme: String = "dark",   // dark / light / amoled / system
    accent: String = "green",  // blue / purple / orange / pink / green
    forceTvUi: Boolean = false,
    content: @Composable () -> Unit
)
```

- 深色判定 `resolveIsDark`：`"light"` → 浅色；`"system"` → 跟随系统
  `isSystemInDarkTheme()`；其余（`dark`/`amoled`）→ 深色。
- 根容器用 `GlassBackground`（Box）而非 `Surface`，并在主题层全局兜底
  `LocalContentColor provides colors.onSurface`——否则所有没写颜色的 Text
  会被 Surface 重置成黑色，压在深色背景上不可见。**复刻时必须保留这一兜底。**

### 9.2 强调色（accentColors，五档 × 明暗双组 RGB）

格式：`Triple(primary, onPrimary, primaryContainer)`。

| 档位 | 浅色 (light) | 深色 (dark) |
|------|--------------|-------------|
| blue | 2563EB / FFFFFF / BFDBFE | 60A5FA / 07131F / BFDBFE |
| purple（PlayAble logo 紫） | 7C3AED / FFFFFF / DDD6FE | B18CFF / 150B26 / 2A1745 |
| orange | EA580C / FFFFFF / FED7AA | FB923C / 201004 / FED7AA |
| pink | DB2777 / FFFFFF / FBCFE8 | F472B6 / 22091A / FBCFE8 |
| green（默认） | 16A34A / FFFFFF / BBF7D0 | 4ADE80 / 04170C / B9F5CB |

### 9.3 完整色板

**浅色（lightColorScheme）**：

| 角色 | 色值 |
|------|------|
| primary / onPrimary / primaryContainer | 见强调色表 |
| onPrimaryContainer | `#0B2512` |
| secondary（PlayAble logo 橙） | `#F5A623` |
| background / onBackground | `#EEF2F7` / `#16202C` |
| surface / onSurface | `#F8FAFD` / `#16202C` |
| surfaceVariant / onSurfaceVariant | `#E2E8F0` / `#4B5A6B` |
| surfaceContainer | `#EAEFF5` |
| surfaceContainerHigh | `#E2E8F0` |
| surfaceContainerLow | `#F0F4F9` |
| error | `#DC2626` |

**深色（darkColorScheme，含 amoled 分支）**：

| 角色 | 标准 dark | amoled |
|------|-----------|--------|
| primary / onPrimary / primaryContainer | 见强调色表 | 同 |
| onPrimaryContainer | = onPrimary | 同 |
| secondary（PlayAble logo 橙） | `#F5A623` | 同 |
| background | `#05070A` | `#000000` |
| onBackground | `#EDF4F0` | 同 |
| surface | `#0B1014` | `#070707` |
| onSurface | `#F4FAF6` | 同 |
| surfaceVariant | `#18202A` | `#121212` |
| onSurfaceVariant | `#C2D0C8` | 同 |
| surfaceContainer | `#10161C` | `#0A0A0A` |
| surfaceContainerHigh | = surfaceVariant | 同 |
| surfaceContainerLow | = surfaceContainer | 同 |
| error | `#FF6B6B` | 同 |

设计基调：暗色 surface 偏"烟熏玻璃"，叠在光斑背景上透出底色；浅色为雾面白玻璃，
文字用深色，强调色自动切加深版。`secondary` 固定用 PlayAble logo 橙 `#F5A623`。

> **【真机实测更正 · 2026-11-18】**：上面色板是源码默认值（`accent="green"` / `theme="dark"`）。
> 但用户真机截图（`home2.png`）当前实际生效的是 **紫色主题**：强调色走 purple 档深色值
> `primary #B18CFF`（logo 紫 `#7C3AED`），背景为左上深紫 `#3D2560` → 右上蓝紫 `#2A3A5E`
> → 底部深紫黑 `#150C22`，选中胶囊为紫色渐变 `#5B3A8C→#7C4DB5`。
> 1:1 复刻真机当前画面时以这组紫色为准；green/4ADE80 仅代表 App 默认档，并非真机现状。
> 品牌名统一为 **PlayAble**（大写 A）。
> 真实 App Logo 见 `assets/logo.png`（192×192：深蓝底 + 金色 P + 镜头齿轮），
> 非文档早前描述的绿色播放图标。

### 9.4 液态玻璃组件色板（GlassPalette）

`glass()` 是 Modifier 扩展读不到 MaterialTheme，通过 `LocalGlass` 在主题层注入。
组件九个字段：`fill / sheen / rim / iconBg / iconBorder / chipBg / chipBorder / sheetBg / barTint`。

**暗色（darkGlass）**：

| 字段 | 值 |
|------|----|
| fill / sheen / rim | `Color.White`（透明度在用法处 copy） |
| iconBg | `White.copy(0.10f)` |
| iconBorder | `White.copy(0.22f)` |
| chipBg | `White.copy(0.08f)` |
| chipBorder | `White.copy(0.16f)` |
| sheetBg | `0xF20B0F13`（近黑玻璃） |
| barTint | `0x33000000` |

**浅色（lightGlass）**：

| 字段 | 值 |
|------|----|
| fill / sheen | `Color.White` |
| rim | `0xFF334155` |
| iconBg | `White.copy(0.62f)` |
| iconBorder | `0xFF334155.copy(0.16f)` |
| chipBg | `White.copy(0.60f)` |
| chipBorder | `0xFF334155.copy(0.14f)` |
| sheetBg | `0xF7F8FAFD` |
| barTint | `0x33FFFFFF` |

另有 `LocalPlayableDark`（当前是否深色）供 GlassBackground / 系统栏图标使用。

### 9.5 字号规范（Typography，16 级）

设计基调：对标 Yamby / Emby 系客户端，高信息密度、紧凑标题、小而清晰的辅助字。
M3 默认值偏大偏松，中文观感散，故全部显式指定 `lineHeight`，保证中文行距在不同 ROM 一致。

| 级别 | 字号/行高 (sp) | 用途 |
|------|----------------|------|
| displayLarge | 28 / 34 | 大数字/大标题 |
| displayMedium | 26 / 32 | 大标题 |
| displaySmall | 24 / 30 | 标题 |
| headlineLarge | 24 / 30 | 页面主标题 |
| headlineMedium | 22 / 28 | 区块标题 |
| headlineSmall | 20 / 26 | **详情页片名** |
| titleLarge | 18 / 24 | **顶栏标题** |
| titleMedium | 16 / 22 | **列表行主标题** |
| titleSmall | 14 / 20 | **海报下方标题 / 设置行标题** |
| bodyLarge | 16 / 24 | 正文 |
| bodyMedium | 14 / 20 | **标准正文 / 集数行** |
| bodySmall | 12 / 16 | 辅助文字 |
| labelLarge | 14 / 20 | 标签 |
| labelMedium | 12 / 16 | 标签 |
| labelSmall | 10 / 14 | **角标 / 时间戳** |

**TV 端规则**：整体字号 ×1.15（≈ +2sp @14sp）。对 ≥16sp 的字号效果明显；
最小的两级在 TV 上有**绝对下限**，保证 3 米（10 英尺）距离可读：
- `bodySmall` 下限 14sp（TV）
- `labelMedium` 下限 13sp（TV）
- `labelSmall` 下限 12sp（TV）

TV 判定：`UiModeManager.currentModeType == UI_MODE_TYPE_TELEVISION`，
比屏幕尺寸启发式准；`forceTvUi` 参数允许平板预览电视版时同样放大。

---

## 10. 开机动画（LogoIntro）1:1 复刻

**实现方式**：Android 框架自带 `VideoView`（系统 MediaPlayer）播放 `res/raw/logo_intro.mp4`，
**不依赖 Media3 / ExoPlayer**。其他端复刻：用平台原生视频播放控件播同一条成片即可，
动画时序、跳过逻辑、兜底逻辑按下述规格完全一致。

**时长**：`INTRO_MS = 3200ms`（成片 3.33s，取 3.2s 留余量让尾帧站稳再淡出）。
冷启动有缓存时给 2s，让"点图标 → 看到片单"更快。**计时从解码器第一帧上屏开始算**，
不是从 Activity 启动算。

**画面**：黑色背景（`Color.Black`），`logo_intro.mp4` 按宽高比铺满并保持比例（FIT），
**绝不裁切 Logo 上下**。

**三阶段动画**：

| 阶段 | 触发 | 时长 | 效果 |
|------|------|------|------|
| ① 整体淡入 | 组合建立 | 260ms（tween `introAlpha`） | alpha 0→1 |
| ② 静态 LOGO 垫图淡出 | 第一帧上屏（`MEDIA_INFO_VIDEO_RENDERING_START`） | 320ms（tween `introPoster`） | 垫图 alpha 1→0 |
| ③ 关闭序列 | 计时到 / 播放完成 / 点击跳过 | 260ms 淡出 + 280ms 后卸载 | `closing=true` → alpha→0 → `onFinish()` |

**细节规格**：
- **静态垫图**：解码器出第一帧前，用 `R.drawable.logo_icon_reverse` 静态 LOGO 居中垫着，
  尺寸 = 短边 × 0.42f，避免用户对着纯黑等解码。出帧后淡出。
- **提示文字**："轻触跳过"，白色 `alpha=0.42`，13sp，距底部 56dp；
  启动 1500ms 后自行淡掉（`hintAlpha` 1→0）。
- **跳过**：单击屏幕任意位置 / 返回键 / 遥控器确认 = 立刻跳过（260ms 淡出）。
  开屏期间按返回**当跳过处理，不退 App**（BackHandler enabled = !closing）。
- **静音**：`mp.setVolume(0f, 0f)`，开屏不发声。
- **兜底**：解码器起不来（损坏 mp4 / 解码器被占用）时，
  `durationMs + 4000ms` 后强制关闭，绝不把 App 卡在开屏；`OnError` 也当结束处理。
- **每个进程只放一次**；设置里可整体关闭开屏动画。
- **焦点**：开屏层 `focusable`，电视遥控器 OK 键也能跳过。
- **释放**：组合销毁时 `VideoView.stopPlayback()`，避免泄漏。

---

## 11. LOGO、图标与资源格式

### 11.1 应用 LOGO / 启动图标

| 项 | 规格 |
|------|------|
| 应用名 | `PlayAble`（`strings.xml` → `app_name`） |
| 启动图标 | `@mipmap/ic_launcher`，多密度目录：mipmap-mdpi / hdpi / xhdpi / xxhdpi / xxxhdpi + mipmap-anydpi-v26 |
| 图标背景 | `drawable/ic_launcher_bg.xml`：**藏蓝→近黑对角渐变**（angle=315），`#1B3A63` → `#12294A` → `#070C18`，配新 logo 的深蓝底 + 金色描边 |
| TV 横幅 | `@drawable/tv_banner`（应用级 `android:banner`，Leanback 启动器展示） |
| 开屏 LOGO | `res/raw/logo_intro.mp4`（3.33s 成片）+ `R.drawable.logo_icon_reverse`（静态垫图） |

**复刻要求**：图标用矢量/位图按各平台密度等价提供；开屏 LOGO 直接复用 `logo_intro.mp4`
（或同画面素材）；渐变背景色值 `#1B3A63 / #12294A / #070C18`（315°）保持一致。

### 11.2 主题 XML（`res/values/themes.xml`）

```xml
<style name="Theme.Lumen" parent="android:Theme.Material.NoActionBar">
    <item name="android:windowBackground">@android:color/black</item>
    <item name="android:statusBarColor">@android:color/transparent</item>
    <item name="android:navigationBarColor">@android:color/transparent</item>
</style>

<style name="Theme.Lumen.Player" parent="Theme.Lumen">
    <item name="android:windowFullscreen">true</item>
    <item name="android:windowBackground">@android:color/black</item>
</style>
```

要点：无 ActionBar；系统栏透明（由 Compose 内容决定观感）；播放页全屏、纯黑背景。

---

## 12. Manifest 与功能开关配置

### 12.1 权限

| 权限 | 用途 |
|------|------|
| INTERNET | 网络访问（Emby/媒体源/弹幕/计费） |
| ACCESS_NETWORK_STATE | 网络状态 |
| WAKE_LOCK | 播放保持唤醒 |
| REQUEST_INSTALL_PACKAGES | OTA 自更新（App 内下载 APK 调起系统安装器，Android 8+ 需授权未知来源） |
| ACCESS_WIFI_STATE | DLNA 投屏 |
| CHANGE_WIFI_MULTICAST_STATE | DLNA SSDP 组播发现 |

### 12.2 特性声明

- `android.software.leanback` required=false（TV 能力，非 TV 设备不强制）
- `android.hardware.touchscreen` required=false（TV/无触屏设备可装）

### 12.3 应用与组件

| 组件 | 关键配置 |
|------|----------|
| Application `.PlayableApp` | `allowBackup=false`；`icon=@mipmap/ic_launcher`；`banner=@drawable/tv_banner`；`theme=@style/Theme.Lumen`；`networkSecurityConfig=@xml/network_security_config`；`supportsRtl=true` |
| `.MainActivity` | `launchMode=singleTop`；`exported=true`；`configChanges=orientation\|screenSize\|keyboardHidden\|uiMode`；意图过滤器 = LAUNCHER + **LEANBACK_LAUNCHER**（TV 主屏入口）；另有 `playable://plex-auth` BROWSABLE 过滤器承接 Plex 浏览器登录回跳 |
| `.player.PlayerActivity` | `launchMode=singleTask`；`supportsPictureInPicture=true`；`theme=@style/Theme.Lumen.Player` |
| `.player.PlayNativePlayerActivity` | 自建播放内核（PlayNative）入口：设置切引擎到 PlayNative，或 DV 强制软解回落时走此 Activity；`singleTask` + PiP + `Theme.Lumen.Player` |
| `com.android.cast.dlna.dmc.DLNACastService` | DLNA 控制点服务（SSDP 监听 + AVTransport 动作执行）；`exported=false` |
| FileProvider | `authorities=${applicationId}.update.provider`，供 OTA 更新包共享给系统安装器 |

DLNA 库 `com.android.cast.dlna.dmc, com.android.cast.dlna.core` 的 minSdk 高于 21，
工程保持 21，运行时路由已按 API 守卫，低版本自动走降级路径。

---

## 13. 页面清单与功能内容（1:1 复刻依据）

### 13.1 手机端主界面（`ui/`）

| 页面 | 内容与功能 |
|------|------------|
| **RootScreen** | 根导航容器：挂载开机动画 LogoIntro；Tab 结构承载首页/媒体库/设置等；开屏结束才进入主界面 |
| **HomeScreen** | 首页：**继续观看**（断点续播列表）+ 媒体库入口行（HomeLibraryRow）；顶部入口到搜索/服务器/设置 |
| **LibraryScreen** | 媒体库：海报墙网格（LibraryCoverCard）；筛选（LibraryFilterSheet：类型/年份/标签等）与排序（LibrarySortSheet：名称/时间/评分等）；LibraryPicker 选择媒体库；LibraryExtras 附加信息 |
| **DetailScreen** | 详情页：片名（headlineSmall 20sp）、海报、简介、评分/年份等元信息；DetailActions 操作组（播放、收藏、投屏等）；DetailExtras 附加信息区；MediaInfoCard 媒体信息卡（分辨率/编码/音轨/字幕流） |
| **EpisodeDetail** | 分集列表：SeasonCardsRow 选季 + EpisodeCardsRow 选集；显示集数标题、时长、已看状态 |
| **SearchScreen** | 搜索：关键词搜索媒体库，结果列表点击进详情 |
| **ServersScreen** | 媒体服务器管理：添加/切换 Emby 等媒体源；**多客户端同步媒体服务器**（PRO 权益）的服务端配置入口；ServerTypeUi 服务器类型界面 |
| **PersonScreen** | 人物页：演员/导演等人物信息及其作品列表 |
| **CollectionScreen** | 合集页：合集内条目列表与播放 |
| **TrackPickSheet** | 音轨/字幕轨道选择弹层：音轨（含 EAC3/AC3/AAC/DTS/TRUEHD 按内核打分择优）、字幕轨切换；底层 sheetBg 玻璃材质 |
| **LibraryFilterSheet / LibrarySortSheet** | 媒体库筛选/排序底部弹层（玻璃材质 sheetBg） |
| **LibraryPicker** | 媒体库选择器（多媒体库切换） |
| **CastSheetHost** | DLNA 投屏弹层宿主：SSDP 发现局域网渲染设备（电视/盒子），投屏到目标设备 |
| **UpdatePrompt** | OTA 更新提示：新版本弹窗，确认后 App 内下载 APK 调起系统安装器 |
| **MediaInfoCard** | 媒体信息卡：展示当前媒体的分辨率、编码、音轨、字幕等技术信息 |
| **Components** | 通用组件库：glass 玻璃材质修饰符、卡片、行、胶囊 chip、圆形图标按钮等，全页复用 |

### 13.2 设置（`ui/settings/`）

| 页面 | 内容与功能 |
|------|------------|
| **SettingsScreen** | 设置主页：入口分组到外观/播放/字幕/视频音频/网络/付费与授权/关于 |
| **AppearanceSettingsScreen** | 外观：主题选择（dark / light / amoled / system）、强调色选择（blue/purple/orange/pink/green，五档实时预览）；控制开机动画开关 |
| **PlaybackSettingsScreen** | 播放设置：播放引擎选择（PlayNative / 系统）、续播开关、倍速、输出方式等 |
| **SubtitleSettingsScreen** | 字幕设置：默认字幕轨、字幕大小/位置、PGS 位图字幕（PRO 权益）开关、字幕搜索入口 |
| **VideoAudioSettingsScreen** | 视频/音频设置：硬解开关、HDR/杜比视界输出策略、音频输出模式（含**声音直通模式** PRO 权益）、音轨选择策略 |
| **NetworkSettingsScreen** | 网络设置：媒体服务器地址、代理、DNS、缓冲等网络相关参数 |
| **LicenseScreen** | **付费与授权（PRO 会员）**：见 §14 |
| **AboutScreen** | 关于：版本号、版权、开源/内核信息 |
| **SubtitleSearchSheet** | 字幕搜索弹层：在线搜索字幕并挂载（配合在线弹幕/字幕 PRO 权益） |

### 13.3 播放器（播放器页面与内核壳）

| 页面/壳 | 内容与功能 |
|------|------------|
| **PlayerActivity** | 系统播放页：`singleTask` + PiP；承载系统引擎播放 |
| **PlayNativePlayerActivity** | **PlayNative 自建内核播放页**：续播 seek（`media_opened` 事件后执行）、`applyNewUrl` 换流、PiP、输出切换；DV 强制软解回落时也走此 Activity |
| **PlayNativeEngine（壳）** | JNI 桥接内核：`attachSurface`、`media_opened` 事件、轨道枚举、进度上报（ProgressReporter，续播/断点） |
| 播放器内功能 | 音轨/字幕切换（TrackPickSheet）、投屏（CastSheetHost）、进度条 seek、倍速、HDR/DV 路由（内核处理）、PGS 字幕渲染、**画中画 PiP**（PRO 权益）、**声音直通**（PRO 权益） |

### 13.4 安卓 TV 端（`ui/tv/`）

| 页面 | 内容与功能 |
|------|------------|
| **TvSidebar** | TV 侧边栏导航：左侧抽屉式菜单（首页/媒体库/搜索/设置等），遥控器焦点导航 |
| **TvFocus** | TV 焦点态组件：卡片/行的焦点高亮、缩放、阴影等 10 英尺可读性焦点样式 |
| TV 全局规则 | 字号整体 ×1.15（绝对下限见 §9.5）；MainActivity 挂 LEANBACK_LAUNCHER 作为 TV 主屏入口；`leanback` 特性声明；开屏层 focusable 支持遥控器 OK 跳过 |

### 13.5 功能全清单（复刻必须覆盖的功能）

**免费版直接可用**：
- 基础播放（PlayNative 内核：硬解、音画同步、seek、倍速）
- 媒体库浏览、搜索、详情、分集/选季、人物、合集
- 字幕基础能力（ASS/SSA/SRT 文本字幕渲染、字幕轨切换、字幕搜索挂载）
- DLNA 投屏、Plex 登录回跳（playable://plex-auth）
- 外观（主题 dark/light/amoled/system + 五档强调色）、播放/字幕/视频音频/网络设置
- 断点续播（ProgressReporter）、OTA 自更新
- 安卓 TV 端全套（侧边栏、焦点态、遥控器交互）

**PRO 会员专属（9 项，见 §14）**：
1. 不支持杜比视界的设备上启用 HDR/SDR 兼容模式观看杜比视界影片
2. 支持的设备下按设备杜比视界授权播放杜比原色视频
3. 在线弹幕
4. 多客户端同步媒体服务器
5. 声音直通模式
6. 画中画（PiP）悬浮小窗
7. PGS 位图字幕等高级字幕渲染
8. 高级解码选项（ISO 原盘等更多格式/编码档位）
9. 设置与播放进度云同步

---

## 14. PRO 会员（付费与授权页）

**入口**：设置 → 付费与授权（LicenseScreen）。

**套餐（1 档，2026-11-22 v45 起收敛）**：

| 套餐 id | 名称 | 价格 |
|--------|------|------|
| `pro` | 专业版 | ¥68 |

> ⚠️ **【2026-11-23 更正】** 本节早期版本写的是「4 档：pro ¥39 / team ¥99 / enterprise ¥299 /
> 买断 ¥68」，那是**旧价目表**，已作废。v45 起产品只有「免费版 / 专业版」两条线，
> 专业版一次性 ¥68。代码权威来源 = `ui/settings/LicenseScreen.kt:53-55`
> （`PLAN_OPTIONS = listOf(PlanOption("pro", "专业版", "¥68"))`），
> 服务端定价权威来源 = VPS `/opt/license-svc/app_new.py` 的 `APP_PLAN_PRICE`
> （第 192–197 行；`playable/pro` = ¥68）。**移植端只做 1 档，不要复刻 4 档。**

**支付方式**：wechat / alipay。

**权益文案（9 项，逐字复刻）**：

1. 在不支持杜比视界的设备上，启用 HDR/SDR 兼容模式观看杜比视界格式影片
2. 在支持的设备下，根据设备的杜比视界授权播放杜比原色视频
3. 解锁在线弹幕
4. 解锁多客户端同步媒体服务器
5. 支持声音直通模式
6. 画中画（PiP）悬浮小窗播放，边看边操作其他应用
7. PGS 位图字幕等高级字幕渲染能力
8. 高级解码选项，支持 ISO 原盘等更多格式与编码档位
9. 设置与播放进度云同步，跨设备保持一致

**其余功能免费版直接可用。** 另有激活码入口（买断/授权码激活）。
计费后端：License 验证服务（VPS `47.95.29.81:9090`，`POST /api/license/verify`，
HttpOnly Cookie + 8h session、设备上限、续期/解绑/吊销恢复/审计日志）。

> ⚠️ **【2026-11-23 更正】授权服务地址已迁移**：旧 VPS `182.255.84.109` 已废，
> 现行地址 = **`47.95.29.81:9090`**（容器 `license-verify` v2.1，多应用版；
> `app_new.py` 以 `:ro` bind mount 到容器 `/app/app.py`）。
> 客户端地址常量见 `data/LicenseClient.kt:212`（`LICENSE_BASE_URL`），
> 且 `normalizeLicenseBaseUrl()` 会把旧地址自动归一到新地址（`:217`）。
> 端口与协议在迁移前后**未变**（都是 9090），移植端照抄 `:9090` 即可，只需换 IP。

---

## 15. UI 复刻执行清单

1. [ ] 建立主题层：PlayableTheme 入口 + 五档强调色 + 明暗完整色板 + glass 调色板（§9）
2. [ ] 全局兜底 `LocalContentColor = onSurface`（根容器用 GlassBackground 而非 Surface）
3. [ ] 落地 16 级 Typography 字号表 + TV ×1.15 与绝对下限（§9.5）
4. [ ] 复刻开机动画：VideoView/等价控件播 logo_intro.mp4，三阶段时序、跳过、兜底（§10）
5. [ ] 准备资源：ic_launcher 多密度、tv_banner、ic_launcher_bg 渐变（315° #1B3A63→#12294A→#070C18）、logo_icon_reverse（§11）
6. [ ] 主题 XML 等价：无 ActionBar、系统栏透明、播放页全屏纯黑（§11.2）
7. [ ] 功能开关等价：Manifest 权限/组件/DLNA/FileProvider 按平台映射（§12）
8. [ ] 逐页复刻：手机端 16 页 + 设置 9 页 + TV 2 页（§13.1-§13.4）
9. [ ] 播放器壳：续播 seek 时机（media_opened 后）、换流、PiP、轨道选择（§13.3）
10. [ ] PRO 会员页：4 套餐 + 9 权益 + 支付/激活码（§14）
11. [ ] 功能核对：免费功能全量 + PRO 9 项权益在对应页面的开关/入口齐全（§13.5）

---

## 附：文档关系

- 本文档 Part 1 内核移植清单的前身为 `07-porting/PORTING-CHECKLIST.md`，内容已合并于此。
- 核心架构（core/hwdec/platform 分层、红线规则）另见 `md/core-architecture.md`；
  iOS/macOS/tvOS 跨端开发（C API 契约、KMP 工程架构）另见
  `md/Playable-lumen-player-iOS跨端开发-20261116.md`。

---

## ⚠️ 2026-11-23 权威口径补充：手机版样式没有丢，本文档与 UI 骨架已过时

**结论先行**：手机/平板样式完整保留在 APK 源码里。v50–v54 这轮所有改动全部 `isTv` 门控
（`forceTvUi` + `isTvDevice()` 设备检测），**非 TV 分支 = 手机版原样，逐处可查**：
- `ui/PsDashboard.kt`：`statColumns=2 / libColumns=1` 默认值即手机版（TV 传 4）
- `ui/HomeScreenPhone.kt`（原 `ui/HomeScreen.kt` 已拆分）：手机版首页实现在
  `PhoneHomeScreen()`，TV 版在 `ui/HomeScreenTv.kt` 的 `TvHomeScreen()`，
  由 `HomeScreenPhone.kt` 尾部调度器按 `isTv` 二选一（`:997` TV / `:999` 手机）
- `ui/LibraryScreen.kt:358`：手机 `GridCells.Adaptive(118.dp)` 原样，仅 TV 固定 8 列
- `ui/DetailScreen.kt:1176`：单集详情手机分支（EpisodeHeaderBlock/EpisodeCardsRow）在
  `if (d.type=="Episode")` 首分支内原样保留；`else if (isTv)` 才走 TV 左右分栏
- `music/ui/MusicScreen.kt`：**页签顺序手机/TV 共用（队列 | 专辑 | 歌手 | 文件夹），不按 TV 重排**；
  TV 差异只体现为焦点描边（`ui/tv/TvFocus.kt` 的 `tvFocusBorder`）。
  早期文档写的 `musicIsTv` 重排分支**已在 v54 按红线回退**，现源码中已不存在该标识符
- `ui/PsHomeStrip.kt`：手机走 `LegacyStripTile` 原样式，TV 才是 `TvStripTile`

**为什么移植时"找不到最新手机版样式"**：
1. **本文档（PORTING-AND-UI-SPEC.md）停在 2026-11-18**，早于 v50–v54（11-22/23）。
2. **UI 骨架（Playable-UI-Skeleton/）是 1600×900 的 TV/Windows 原型**，本来就不是
   手机版样式的载体，不能当手机版参考。
3. v50–v54 手机版实质变化只有三处（均为跨平台非样式或共用改动）：
   - v52：排序菜单 8 项 → **19 项**（`SortSpec.kt` SortField，`LibrarySortSheet` FlowRow
     自动枚举，手机/TV 同源生效，对齐 Emby 原生菜单）
   - v51：`isScopeCancelled` 守卫——协程取消异常不再当业务错误显示（跨平台通用）
   - v53：引擎 seek 音画同步修复（native + PlayNativeEngine，无 UI）
   其余全部 TV 专属，手机版零改动。

**移植时的正确姿势**：
- 手机/平板样式唯一权威 = `pn-build/android/lumen-player/app/src/main/java/com/playable/player/`
  源码中**不带 isTv 分支的默认路径**（每个参数的默认值即手机版）。
- TV 样式参考 = UI 骨架（Playable-UI-Skeleton/）+ 各屏 `if (isTv)` 分支。
- 排序 19 项映射表见 AUTHORITATIVE-SOURCES.md v52 章节。

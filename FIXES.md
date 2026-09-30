# 修复说明

本仓库是 [lid664951-crypto/Cap](https://github.com/lid664951-crypto/Cap) 的个人维护分支，在中文版 `0.4.3-cn` 基础上修复了 **Windows 平台窗口录制产生空录像** 的问题。

## 问题现象

Windows 上录制窗口后，编辑器报错：

```
Failed to remux recording: Failed to concatenate video fragments:
FFmpeg error: Invalid data found when processing input
```

录像为空、无法播放，日志中反复出现 `Failed to encode frame: Converter: Input changed`。

## 根本原因

采集帧的宽高与编码器声明的 `VideoInfo` 不一致，导致**每一帧都被丢弃**。

`ffmpeg-next` 的 `software::scaling::Context::run` 有前置校验：帧的 `format` / `width` / `height` 必须与创建 sws context 时声明的值完全一致，差 1 像素即返回 `InputChanged`。

而 Windows 上「声明尺寸」与「实际帧尺寸」走了两条独立的取整路径：

| | 来源 | 取整方式 |
|---|---|---|
| 声明尺寸 | `crop_bounds.size()` | `(v / 2).floor() * 2`，向下取偶 |
| 实际帧尺寸 | D3D11 裁剪盒 `crop.right - crop.left` | `left` / `right` 各自独立 `as u32` |

窗口宽高为奇数（DWM 扩展边框常见 1923×1041）或坐标为负（最大化窗口的负偏移，`as u32` 会饱和截断为 0）时，两者必然相差 1~2 像素。

于是编码器收到 0 帧，没有任何 `.m4s` 分段产出，`display.combined_fmp4.mp4` 为 0 字节。最后由 FFmpeg 打开这个空文件，抛出上面的 `Invalid data found`——**这只是下游噪音，不是真正的故障点**。

这也解释了为什么同为屏幕录制的某次 1920×1080 录制能成功：该尺寸恰好是偶数且无偏移。

## 修复内容

### 1. 裁剪盒尺寸与声明尺寸对齐

`crates/recording/src/sources/screen_capture/windows.rs`

新增 `crop_box_for_bounds()`，统一以 `left` / `top` 为基准推导 `right` / `bottom`，保证 `right - left` 恒等于取偶后的宽度。采集侧与声明侧共用同一个函数，两条路径不会再分叉。

### 2. 编码器尺寸不匹配时自愈

`crates/enc-ffmpeg/src/video/h264.rs`

新增 `convert_frame()`：一旦遇到 `InputChanged`，按帧的真实参数重建 sws 转换器并重试一次，同时输出 `actual_*` / `declared_*` 对照日志。即使还存在未知的尺寸不一致路径，录像也不会再整体变空。

### 3. 收尾 `dash_manifest.mpd.tmp`

`crates/enc-ffmpeg/src/mux/segmented_stream.rs`

FFmpeg 的 DASH muxer 写 MPD 同样是「先写 `.tmp` 再改名」，Windows 上目标文件被占用会导致改名失败，内容留在 `.tmp` 里。原逻辑只处理 `segment_*.m4s.tmp`，现在把 `dash_manifest.mpd.tmp` 纳入同一套重命名重试逻辑。

### 4. 移除重复的片段探测实现

`apps/desktop/src-tauri/src/recording.rs`、`crates/recording/src/recovery.rs`

桌面端原有一份 `find_fragments_in_dir`，按扩展名匹配 `.mp4` / `.m4a`。而录制目录里是 `init.mp4` + `segment_001.m4s`：`.m4s` 不匹配、`init.mp4` 匹配，于是 **DASH 初始化段被当成了媒体片段**，拿它去 concat 必然失败，并报出误导性的 remux 错误。

`crates/recording` 中早有正确实现（包含 `.m4s`，且靠「能否解出帧」排除 `init.mp4`）。现已删除重复实现，桌面端改用 `RecoveryManager::probe_fragments_in_dir`。

## 构建相关改动

以下改动仅为了能在**本地无签名密钥**的情况下构建，与上述缺陷无关：

- `createUpdaterArtifacts: false` —— 本地构建跳过更新包签名
- 更新检查失败改为静默记录 —— 自编译版本未配置更新源，弹窗属误报
- pnpm 10 需要显式批准原生依赖的构建脚本

> ⚠️ 由于 `createUpdaterArtifacts: false`，本仓库构建出的包**不具备自动更新能力**。若改用 CI 出包，需另行处理签名配置。

## 验证

已在 Windows 11（NVIDIA GeForce MX250 + 核显，共 4 个 GPU 适配器）实测通过：窗口录制、区域录制、全屏录制均能正常出画面，`dash_manifest.mpd` 正常生成，无 `Input changed` 与 remux 报错。

## 与上游同步

```bash
git fetch upstream
git merge upstream/main
```
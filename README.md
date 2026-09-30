<p align="center">
  <img width="150" height="150" src="./apps/desktop/src-tauri/icons/Square310x310Logo.png" alt="Logo">
</p>

<h1 align="center"><b>Cap 中文版</b></h1>

<p align="center">
  Loom 的开源替代方案，基于 <a href="https://github.com/CapSoftware/Cap">Cap 官方项目</a> 汉化
  <br />
  <a href="https://github.com/wumajiehechuan-lab/Cap-cn"><strong>本仓库 »</strong></a> ·
  <a href="https://github.com/lid664951-crypto/Cap"><strong>上游中文版 »</strong></a>
  <br /><br />
  <b>预编译安装包：</b>Windows x64
</p>

<br/>

## 📖 简介

本仓库是 [Cap 中文版](https://github.com/lid664951-crypto/Cap) 的个人维护分支，在 `0.4.3-cn` 基础上修复了 **Windows 窗口录制产生空录像** 的问题。预编译安装包见 [Releases](https://github.com/wumajiehechuan-lab/Cap-cn/releases)。

## ✨ 特性

- **完全汉化**：界面与功能说明全部汉化
- **移除登录限制**：无需登录即可使用所有功能
- **去除付费板块**：所有功能完全免费
- **编辑器优化**：优化了视频编辑功能

## 🔧 修复说明（v0.4.3-cn.1）

### 问题现象

Windows 上录制窗口后，录像为空、无法播放，编辑器报错：

```text
Failed to remux recording: Failed to concatenate video fragments:
FFmpeg error: Invalid data found when processing input
```

日志中反复出现 `Failed to encode frame: Converter: Input changed`。

### 根本原因

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

### 修复内容

**1. 裁剪盒尺寸与声明尺寸对齐** — `crates/recording/src/sources/screen_capture/windows.rs`

新增 `crop_box_for_bounds()`，统一以 `left` / `top` 为基准推导 `right` / `bottom`，保证 `right - left` 恒等于取偶后的宽度。采集侧与声明侧共用同一个函数，两条路径不会再分叉。

**2. 编码器尺寸不匹配时自愈** — `crates/enc-ffmpeg/src/video/h264.rs`

新增 `convert_frame()`：一旦遇到 `InputChanged`，按帧的真实参数重建 sws 转换器并重试一次，同时输出 `actual_*` / `declared_*` 对照日志。即使还存在未知的尺寸不一致路径，录像也不会再整体变空。

**3. 收尾 `dash_manifest.mpd.tmp`** — `crates/enc-ffmpeg/src/mux/segmented_stream.rs`

FFmpeg 的 DASH muxer 写 MPD 同样是「先写 `.tmp` 再改名」，Windows 上目标文件被占用会导致改名失败，内容留在 `.tmp` 里。原逻辑只处理 `segment_*.m4s.tmp`，现在把 `dash_manifest.mpd.tmp` 纳入同一套重命名重试逻辑。

**4. 移除重复的片段探测实现** — `apps/desktop/src-tauri/src/recording.rs`、`crates/recording/src/recovery.rs`

桌面端原有一份 `find_fragments_in_dir`，按扩展名匹配 `.mp4` / `.m4a`。而录制目录里是 `init.mp4` + `segment_001.m4s`：`.m4s` 不匹配、`init.mp4` 匹配，于是 **DASH 初始化段被当成了媒体片段**，拿它去 concat 必然失败，并报出误导性的 remux 错误。现已删除重复实现，桌面端改用 `crates/recording` 中已有的 `RecoveryManager::probe_fragments_in_dir`（包含 `.m4s`，且靠「能否解出帧」排除 `init.mp4`）。

### 构建相关改动

以下改动仅为了能在**本地无签名密钥**的情况下构建，与上述缺陷无关：

- `createUpdaterArtifacts: false` —— 本地构建跳过更新包签名
- 更新检查失败改为静默记录 —— 自编译版本未配置更新源，弹窗属误报
- pnpm 10 需要显式批准原生依赖的构建脚本

> ⚠️ 由于关闭了 `createUpdaterArtifacts`，本仓库构建出的包**不具备自动更新能力**。

### 验证

已在 Windows 11（NVIDIA GeForce MX250 + 核显，共 4 个 GPU 适配器）实测通过：窗口录制、区域录制、全屏录制均能正常出画面，`dash_manifest.mpd` 正常生成，无 `Input changed` 与 remux 报错。

## 🚀 安装使用

### 方法一：下载安装包

从 [Releases](https://github.com/wumajiehechuan-lab/Cap-cn/releases) 下载 Windows x64 安装包：

| 文件 | 说明 |
|---|---|
| `Cap 中文版_0.4.3-cn_x64-setup.exe` | NSIS 安装包，推荐 |
| `Cap 中文版_0.4.3-cn_x64_zh-CN.msi` | MSI 安装包 |

### 方法二：源码构建

```bash
git clone https://github.com/wumajiehechuan-lab/Cap-cn.git
cd Cap-cn
pnpm install
pnpm build:desktop
```

同步上游更新：

```bash
git remote add upstream https://github.com/lid664951-crypto/Cap.git
git fetch upstream
git merge upstream/main
```

## 📸 界面预览

### 首页
<img src="./UI界面/首页.png" alt="首页" width="800" />

### 编辑器页面
<img src="./UI界面/编辑器页面.png" alt="编辑器页面" width="800" />

### 设置页面
<img src="./UI界面/设置页面.png" alt="设置页面" width="800" />

## 🛠️ 技术栈

- **桌面端**：Tauri · Rust
- **前端**：SolidStart · React (Next.js) · TypeScript · TailwindCSS
- **构建**：Turborepo

## 📄 许可证

本项目基于原项目的许可证进行分发：

- `cap-camera*` 和 `scap-*` 系列代码使用 MIT 许可证
- 第三方组件使用其原始许可证
- 其他内容使用 AGPLv3 许可证

详细信息请查看 [LICENSE](LICENSE) 文件。

## 🔗 相关链接

- **本仓库**：https://github.com/wumajiehechuan-lab/Cap-cn
- **上游中文版**：https://github.com/lid664951-crypto/Cap
- **Cap 官方项目**：https://github.com/CapSoftware/Cap

## 🤝 贡献

欢迎提交 Issue 和 Pull Request 来帮助改进这个项目！
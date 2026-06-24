# ZnScreen

Go 实现的跨平台实时屏幕共享工具，**无 CGO 依赖**，单二进制部署。

浏览器访问 → 输入密码 → 实时观看屏幕，延迟 < 100ms。

[![Go](https://img.shields.io/badge/Go-1.21+-00ADD8?logo=go)](https://go.dev/)
[![CGO](https://img.shields.io/badge/CGO-disabled-red)](https://pkg.go.dev/cmd/cgo)
[![License](https://img.shields.io/badge/license-MIT-blue)](LICENSE)

## 特性

- **零外部依赖** — Linux/Windows 内置 ffmpeg 静态二进制，解压即用；macOS 使用系统 ffmpeg
- **极低延迟** — ffmpeg MJPEG 硬件编码 + WebSocket 二进制帧直推，不经 Canvas 渲染
- **背压保护** — 环形缓冲区仅保留最新一帧，慢客户端自动丢帧，不拖垮服务端
- **fan-out 广播** — 单个 ffmpeg 进程服务所有 WebSocket 客户端，按需启停
- **跨平台** — Linux（X11）、macOS（AVFoundation）、Windows（GDI）均可捕获
- **随机密码** — 每次启动生成随机密码，打印在终端

## 快速开始

```bash
# 下载对应平台的二进制
# 或自行编译：make build

# 启动（默认监听 0.0.0.0:9115）
./znscreen

# 终端输出：
# 密码: a1b2c3d4e5f6g7h8
# ZnScreen 已启动: http://0.0.0.0:9115
```

浏览器打开 `http://服务器IP:9115`，输入密码即可观看屏幕。

## 命令行参数

| 参数 | 默认值    | 说明                          |
| ---- | --------- | ----------------------------- |
| `-h` | `0.0.0.0` | 监听地址                      |
| `-p` | `9115`    | 监听端口                      |
| `-r` | `15`      | 帧率（fps）                   |
| `-q` | `5`       | JPEG 质量（2–31，越小越清晰） |

```bash
# 自定义示例
./znscreen -h 127.0.0.1 -p 8080 -r 30 -q 3
```

## 编译

### 内置 ffmpeg（推荐，~140MB）

```bash
# 1. 下载各平台 ffmpeg（仅需一次）
make ffmpeg

# 2. 全平台编译
make all

# 或编译单个平台
make linux     # Linux amd64 + arm64
make darwin    # macOS amd64 + arm64（系统 ffmpeg）
make windows   # Windows amd64
```

### 不含 ffmpeg（~5MB，依赖系统 ffmpeg）

```bash
make noembed
```

所有产物在 `dist/` 目录。

## 架构

```
┌─────────────┐   WebSocket (binary JPEG)   ┌─────────────────────┐
│   浏览器     │◄══════════════════════════►│   Go Server          │
│  (单页 HTML) │                             │   - 密码验证         │
└─────────────┘                             │   - 帧 fan-out 广播  │
                                            └──────────┬──────────┘
                                                       │
                                            ┌──────────▼──────────┐
                                            │   ffmpeg 子进程      │
                                            │   MJPEG → stdout     │
                                            │   Linux:   x11grab   │
                                            │   macOS:   avfound.. │
                                            │   Windows: gdigrab   │
                                            └─────────────────────┘
```

### 数据链路

```
ffmpeg stdout ──→ bufio.Scanner ──→ splitJPEG 切帧 ──→ channel[1] ──→ WebSocket WriteMessage
                    (零拷贝)         (FFD8/FFD9)      (环形缓冲)       (BinaryMessage)
```

### 目录结构

```
ZnScreen/
├── main.go                              # 入口
├── frontend/index.html                  # 嵌入式 Web 前端
├── internal/
│   ├── auth/auth.go                     # 密码生成 + session token
│   ├── capture/capture.go               # ffmpeg 管理 + 帧分割 + fan-out
│   ├── ffmpegbin/                       # ffmpeg 内置二进制管理
│   │   ├── ffmpeg.go                    # 提取/缓存逻辑
│   │   ├── embed_linux_amd64.go         # linux/amd64 embed
│   │   ├── embed_linux_arm64.go         # linux/arm64 embed
│   │   ├── embed_windows_amd64.go       # windows/amd64 embed
│   │   ├── embed_noembed.go            # -tags noembed 模式
│   │   ├── embed_unsupported.go         # 未支持平台 fallback
│   │   └── ffmpeg/                      # gitignored 的 ffmpeg 二进制
│   └── server/server.go                 # HTTP + WebSocket
├── go.mod / go.sum
└── Makefile
```

### ffmpeg 来源

静态构建来自 [BtbN/FFmpeg-Builds](https://github.com/BtbN/FFmpeg-Builds)，GPL 许可。

- Linux amd64: `ffmpeg-master-latest-linux64-gpl.tar.xz`
- Linux arm64: `ffmpeg-master-latest-linuxarm64-gpl.tar.xz`
- Windows amd64: `ffmpeg-master-latest-win64-gpl.zip`

首次运行时，内置的 ffmpeg 二进制会被提取到 `$TMPDIR/znscreen-ffmpeg/` 并缓存，后续启动直接复用。

## 平台支持

| 平台          | 内置 ffmpeg | 二进制大小 | 说明                     |
| ------------- | :---------: | ---------- | ------------------------ |
| Linux amd64   |      ✅      | ~143MB     | X11 屏幕捕获             |
| Linux arm64   |      ✅      | ~107MB     | X11 屏幕捕获             |
| macOS amd64   |      ❌      | ~5MB       | 需 `brew install ffmpeg` |
| macOS arm64   |      ❌      | ~5MB       | 需 `brew install ffmpeg` |
| Windows amd64 |      ✅      | ~142MB     | GDI 屏幕捕获             |

## 依赖

- [github.com/gorilla/websocket](https://github.com/gorilla/websocket) — WebSocket 实现

## License

MIT

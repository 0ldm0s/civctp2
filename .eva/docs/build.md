# 构建指南

Windows 用 Visual Studio（MSVC），Linux 用 autotools 或 meson。无包管理器清单文件（非 npm/cargo 项目），依赖为系统级或源码集成。

## Windows（Visual Studio）

前提条件：
1. Visual Studio 2017 或 2019（含 Windows SDK）
2. SDL 构建需要 NASM：`NASM/` 目录自带 64 位版本，按 README 说明放入 VS 的 VC 目录
3. 环境变量 `CDKDIR` 指向源码路径下的 `bin` 目录（含 bison、flex 等工具）
4. 源码所在盘根目录需要 `tmp` 目录（如 `C:\tmp`、`D:\tmp`）

打开解决方案：`ctp2_code/ctp/civctp.sln`

配置（平台 Win32/x64/ARM/ARM64，ARM 无法链接）：

| 配置 | 输出可执行文件 | 说明 |
|------|--------------|------|
| Debug | ctp2-dbg-dx.exe | DirectX，含日志/断言/泄漏检测 |
| Final | ctp2-dx.exe | DirectX，原始发行版 |
| Logging | ctp2-dbg-dx.exe | DirectX，Final + 日志（AI 调试常用） |
| Release | ctp2-log-dx.exe | DirectX，无 CD 检查 |
| Debug-SDL | ctp2-dbg.exe | SDL2，含日志/断言/泄漏检测 |
| Final-SDL | ctp2.exe | SDL2，常规发行版 |
| Logging-SDL | ctp2-log.exe | SDL2，Final + 日志 |

输出位置：`ctp2_code/ctp/`（Win32）或 `ctp2_code/ctp/x64/`（x64）。

调试启动前确认启动项目为 ctp2（否则 VS 可能尝试启动 Crater.dll）。

## Linux（autotools，主流程）

依赖：GCC 5+、SDL2、SDL2_mixer、SDL2_image、libtiff、zlib、byacc、flex、unzip：

```bash
sudo apt install libsdl2-dev libsdl2-mixer-dev libsdl2-image-dev libtiff-dev libavcodec-dev libavformat-dev libswscale-dev byacc flex
```

三种构建变体：

```bash
# 常规优化版（Final 等价）
./autogen.sh
CFLAGS="$CFLAGS -O3 -fuse-ld=gold" CXXFLAGS="$CXXFLAGS -O3 -fuse-ld=gold" ./configure --enable-silent-rules
make -j$(nproc)

# 日志 + 优化（长局 AI 测试用）
./configure --enable-silent-rules --enable-logging

# 调试版（断言 + 泄漏检测 + 可调试）
./configure --enable-silent-rules --enable-debug
```

注意事项：
- 首次 `make -j$(nproc)` 可能失败——部分生成文件（flex/byacc）的依赖未声明，重跑即可
- 链接器建议 gold（`-fuse-ld=gold`）
- 视频播放需加 `--enable-ffmpeg4movies`（默认关闭，FFmpeg API 变动频繁）
- 构建产物 `ctp2_code/ctp2`，需位于 `ctp2_code/ctp/` 下运行（`tools/MakeAndRun.sh` 会自动移动并启动）
- mapgen 插件 .so 由 `GNUmakefile.am` 的 all-local 自动复制到 `ctp2_code/ctp/dll/map/`

## Linux（meson，替代方案）

`ctp2_code/meson.build` 提供独立 meson 构建（anet、freetype 作为 meson 子项目）：

```bash
cd ctp2_code
meson setup build
meson compile -C build
```

## configure 选项

| 选项 | 作用 |
|------|------|
| `--enable-logging` | 启用日志（USE_LOGGING） |
| `--enable-debug` | 调试版（_DEBUG、断言、泄漏报告、_PLAYTEST） |
| `--enable-ffmpeg4movies` | FFmpeg 视频播放（默认关） |
| `--enable-precisetraderoutecalc` | 精确但慢的贸易路线计算 |

强制依赖 byacc（configure 检查失败即报错）和 unzip。

## 运行

需要完整游戏数据文件（仓库不含）：`ctp2_data`、`Scenarios` 需从原版 CD 或 GOG 版补齐（README 有 innoextract 提取方法）。可执行文件从 `ctp2_code/ctp/` 目录运行。

运行时用户数据写入 `.civctp2/`（存档 save、日志 logs、`userprofile.txt`、`userkeymap.txt`）。

## CI（事实说明，本地构建不依赖它）

- `.gitlab-ci.yml`：GitLab CI，Docker 多阶段构建（system → builder → install），产出 .deb 包；test 阶段在 Xvfb 中跑 Sikulix GUI 测试，截图由 `GL-CI.md` 展示
- `Dockerfile`：配套的多阶段构建文件（target：system / builder / install）
- `.travis.yml`：旧 Travis 配置（autotools 直构建 + docker 构建）
- 本项目无 GitHub Actions

## 常用工具脚本

- `tools/MakeAndRun.sh`：make 后移动可执行文件并启动
- `tools/Run.sh`：直接启动游戏
- `tools/build-deb.sh`：CI 中打 deb 包
- `bin/`：Windows 用的 byacc/flex/Unix 工具二进制

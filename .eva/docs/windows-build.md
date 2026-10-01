# Windows 命令行构建指南（MSBuild）

2026-10-01 在本机（VS 18 Community）全流程验证通过，产出 Final-SDL|x64 的 `ctp2_code/ctp/x64/ctp2.exe`。VS 图形界面打开 `civctp.sln` 的方式仍然有效，但本机装的 VS 18 与工程声明的工具集不一致，命令行构建需要下述覆盖参数。

## 环境要求

| 项 | 要求 | 说明 |
|------|------|------|
| Visual Studio | VS 18 Community（`D:\Program Files\Microsoft Visual Studio\18\Community`） | MSBuild 18.10 |
| 平台工具集 | **v145**（命令行覆盖） | VS 18 唯一可用；工程声明的 v141 已无编译器；`v150/v160/v170` 目录是空壳，传这些报 MSB8020 |
| Windows SDK | **10.0.26100.0**（四段式，装在 `D:\Windows Kits\10`） | 写 `10.0.26100`（三段）报 MSB8036 |
| CDKDIR | 仓库 `bin\` 目录（含 byacc/flex） | 结尾反斜杠必须有（工程内是 `$(CDKDIR)\flex` 拼接） |
| `D:\tmp` | 必须存在 | MSVC 历史遗留要求 |
| NASM | 见下节 | 仅 FFmpeg 需要 |

## NASM 部署（FFmpeg 汇编必需）

1. 解压 `NASM/nasm-2.16.03-win64.zip`
2. `nasm.exe` 复制到 VS 的 `VC\` 根目录（`nasm.props` 默认 `NasmPath=$(VCInstallDir)`）
3. `nasm.props`、`nasm.targets`、`nasm.xml` 复制到 `MSBuild\Microsoft\VC\v180\BuildCustomizations\`

注意是 **v180** 不是 v160——FFmpeg SMP 工程的 `$(VCTargetsPath)` 解析到 v180 目录。

## 标准命令

bash（MSYS2/Git Bash）环境执行。两个关键点：`MSYS2_ARG_CONV_EXCL="*"` 防止 MSYS2 把 `/t`、`/p` 参数转成路径；PowerShell 内先清掉 `MSYSTEM` 系变量避免与 MSVC 冲突：

```bash
MSYS2_ARG_CONV_EXCL="*" powershell.exe -NoProfile -Command "Remove-Item Env:MSYSTEM,Env:MSYSTEM_PREFIX,Env:MSYSTEM_CARCH,Env:MSYSTEM_CHOST -ErrorAction SilentlyContinue; & 'D:\Program Files\Microsoft Visual Studio\18\Community\MSBuild\Current\Bin\amd64\MSBuild.exe' 'D:\workspaces\civctp2\ctp2_code\ctp\civctp.sln' '/t:ctp2' /p:Configuration=Final-SDL /p:Platform=x64 /p:PlatformToolset=v145 /p:WindowsTargetPlatformVersion=10.0.26100.0 /p:VisualStudioVersion=18.0 '/p:CDKDIR=D:\workspaces\civctp2\bin\' /m /v:m /nologo"
```

参数说明：

- `/t:ctp2` — 目标项目；`/t` 可用分号列多个（如 `'/t:dbgen;zlibvc;tiff;freetype;anet'`）
- `/p:PlatformToolset=v145` — 覆盖所有工程的 v141 声明
- `/p:WindowsTargetPlatformVersion=10.0.26100.0` — 覆盖工程的 10.0.19041.0
- `/p:VisualStudioVersion=18.0` — 必须！否则 FFmpeg 失败（见"构建适配"第 4 条）
- `/p:CDKDIR` — flex/byacc 生成器路径

## 构建顺序（首次全量）

```
libavutil
  → libswresample、libswscale、libavcodec、libavfilter、libavformat、libavdevice
dbgen（生成器 ctpdb.exe，产出 34 组 *Record.cpp）
zlibvc → zlibwapi.lib
tiff   → tiff.lib
freetype → freetype.lib
anet   → anet2.lib（+ anet2.dll）
civctp → ctp2.exe（内部自动跑 .cdb 生成、flex/byacc 解析器生成）
Crater、fault、geometric、Plasma2（mapgen DLL）
wipx、winets、wudplan（网络传输 DLL）
```

增量构建直接 `/t:ctp2`（或对应目标），MSBuild 自动跳过最新部分。

## 构建适配修改（仓库内已做的改动）

这些改动是本机构建链能通过的前提，`git status` 中可见：

1. **`ctp2_code/ctp/civctp.vcxproj`** — Final-SDL/Logging-SDL 的 Win32+x64 四个配置段删除了 `USE_SDL_FFMPEG` 预处理宏；Final-SDL|x64 的链接项删除了 7 个 `..\msvc\lib\$(Platform)\libav*.lib`。效果：视频播放功能禁用（README 认可的做法）。恢复视频需还原这两处并把 FFmpeg DLL 拷到 exe 目录。
2. **`ctp2_code/libs/anet/anet.vcxproj`** — 所有配置的预处理定义加了 `COMM_INST`。原因：anet dp 层几十处 `commXxx(&req, &resp, commPtr)` 三参调用依赖该宏让原型带第三参；Linux 上 `UNIX` 宏自动定义它所以上游没暴露，Windows 必须显式加。
3. **`ctp2_code/libs/FFmpeg-n6.1.2/libavcodec/refstruct.c`** — 补 `#include <stddef.h>` + MSVC 下 `typedef double max_align_t`。原因：VS 18 的 C 工具链（14.51）不提供 `max_align_t`（其 STL 在 C++ 里定义为 double），FFmpeg 代码依赖 glibc 的链式引入。
4. **`/p:VisualStudioVersion=18.0`** — SMP 工程的 `<LanguageStandard_C Condition="'$(VisualStudioVersion)' > '15.0'">stdc11</LanguageStandard_C>`，而 sln 头声明 `# Visual Studio 15` 会把该属性压成 15.0，条件不成立导致 C89 模式报 `_Alignof`/`max_align_t` 错误。

## 运行与数据（2026-10-01 社区版实机跑通验证）

- 主程序：`ctp2_code/ctp/x64/ctp2.exe`（约 5.1 MB）
- 运行时依赖（均在 `ctp/x64/`，构建自动落位）：`anet2.dll`、`tiff.dll`、`zlibwapi.dll`（构建产物）；`SDL2.dll`、`SDL2_mixer.dll`、`SDL2_image.dll`（预编译自动拷贝）；`dll/map/`（4 个 mapgen 插件）；`dll/net/`（3 个传输 DLL）

### 启动方式

必须控制工作目录（civpaths.txt 及其内所有相对路径都相对 cwd 解析）：

```bash
MSYS2_ARG_CONV_EXCL="*" powershell.exe -NoProfile -Command "Start-Process -FilePath 'D:\workspaces\civctp2\ctp2_code\ctp\x64\ctp2.exe' -WorkingDirectory 'D:\workspaces\civctp2\ctp2_code\ctp\x64'"
```

### civpaths.txt 路径机制（`ctp/x64/civpaths.txt`，用上游原版即可）

文件为纯值行列表，`fscanf %s` 顺序读入（空格会截断，不能写带空格的路径）：

| 行 | 内容 | 去向 |
|----|------|------|
| 1 | `..` | m_hdPath |
| 2 | （CD 路径，SDL 版丢弃） | dummy |
| 3 | `default` | m_defaultPath |
| 4 | `english` | m_localizedPath |
| 5 | `..\..\ctp2_data` | m_dataPath（数据根，**两跳**） |
| 6 | `..\..\Scenarios` | m_scenariosPath |
| 7-13 | save 系目录 | 存档树（实际落在 `ctp/save/`） |
| 14-29 | gamedata/uidata/graphics... | assetPaths[C3DIR_*]，**保持树内短名** |

**跳数陷阱**：文件查找（`CivPaths::FindFile`）的拼串格式是 `hdPath\dataPath\default|english\assetPath\filename`，`hdPath` 的 `..` **自占一跳**。cwd=x64 时 dataPath 写两跳（`..` + 两跳 = 三跳到仓库根）才正确；写成三跳会落到 `D:\workspaces\ctp2_data`。

**不要给 assetPaths（行 14-29）加数据根前缀**：语言回退组合是 `dataPath\english\assetPath`，assetPaths 带 `..\..\` 前缀会把 english 组合劫持回 default 树，导致 `Strings.txt`（只在语言树）找不到。

### 数据完整性现状（重要）

上游 git 仓库**有意不携带二进制资源**，仓库 `ctp2_data`（74 MB）与完整安装差距很大：

1. LDL 布局（`default/uidata/layouts/*.ldl`，94 个）引用 998 个 tga，仓库仅存少量；CD 安装（`civmain.ctp`/`civlang.ctp`，即 ZIP 包）与 GOG 安装同样不含大部分社区引用的资源。
2. 原版贴图以 **ZFS 档案**（`graphics/pictures/pic555.zfs`、`pic565.zfs`、`patterns/pat*.zfs`、`sound/sound.zfs`）形式分发，仓库同样没有。游戏有 `.tga → .rim` 的 ZFS 回退（`FindFile` + `g_ImageMapPF`，按像素格式选 555/565 档案），前提是 ZFS 能被 `AddSearchPacks` 按 assetPaths 目录找到。
3. 部分社区 UI 资源（cba 系列、b4_titlebar 系列、ug 系列等 839 个）**任何发行物中都不存在**——上游从未发布，只能在 Apolyton 论坛/CivFanatics 资源区找社区积累包，或用占位图兜底。

### 本机已做的数据补齐（不入库，见 .gitignore）

- 从 GOG 安装（`D:\Games\GOG Galaxy\Games\Call To Power 2\`）robocopy 补缺到仓库 `ctp2_data/default/`：playlist.txt、全量 ZFS、原版贴图（`/XC /XN /XO` 只补缺不覆盖仓库文本改动）
- 从原版 CD 镜像（archive.org 的 [CTP2 2000 版](https://archive.org/details/Call_to_Power_II_Microprose_Activision_2000)，BIN 为 MODE1/2352 raw 格式，需剥 16 字节扇区头转 ISO 后 7z 解包）提取 ZFS
- 为 839 个无源贴图生成 32x32 **24 位**黑色占位 TGA 放 `default/graphics/pictures/`（注意：32 位 TGA 会让 aui 的 Targa 加载器直接崩溃退出，必须 24 位）。拿到真实资源后直接覆盖同名文件即可。

### 运行期错误对照

| 弹窗 | 含义 |
|------|------|
| `Paths Error: 'xxx' not found in asset tree` | 该文件不在数据树：Language.txt/Strings.txt 类=路径跳数错或语言树缺文件；playlist.txt=仓库缺文件，需从 GOG 补 |
| `Targa Load Error: Unable to find 'xxx.tga'` | 数据树已通，缺具体贴图：uptg 系列可由 ZFS 回退解决（确认 pic*.zfs 可达）；cba/ug/b4 系列需占位或社区资源 |
| 启动即无声退出（exit -1） | 32 位占位 TGA 崩溃，换 24 位 |
| 游戏能进但界面黑块 | 占位贴图生效中，属预期；替换真实资源即恢复 |

## 常见错误速查

| 错误 | 原因与解法 |
|------|------|
| MSB8020 找不到 v160/v141 生成工具 | 工具集必须是 v145 |
| MSB8036 找不到 SDK 10.0.26100 | 版本号写四段式 `10.0.26100.0` |
| MSB4019 找不到 nasm.props | NASM 自定义文件放 `v180\BuildCustomizations` |
| C2197 "用于调用的参数太多"（anet） | anet.vcxproj 缺 `COMM_INST` 宏 |
| C2061 标识符 "max_align_t"（FFmpeg） | refstruct.c 补丁缺失，或缺 `/p:VisualStudioVersion=18.0` |
| bash 报 `/t:` 变成路径 | 加 `MSYS2_ARG_CONV_EXCL="*"` |
| 后台任务引号断裂 | 避免 PowerShell 命令串里出现 `\$` 转义和尾反斜杠引号组合 |

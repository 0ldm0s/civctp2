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

## 输出与运行

- 主程序：`ctp2_code/ctp/x64/ctp2.exe`（约 5.1 MB）
- 运行时依赖（均在 `ctp/x64/`，构建自动落位）：`anet2.dll`、`tiff.dll`、`zlibwapi.dll`（构建产物）；`SDL2.dll`、`SDL2_mixer.dll`、`SDL2_image.dll`（预编译自动拷贝）；`dll/map/`（4 个 mapgen 插件）；`dll/net/`（3 个传输 DLL）
- 游戏数据 `ctp2_data` 需在运行目录可见（见 README 补数据方法）

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

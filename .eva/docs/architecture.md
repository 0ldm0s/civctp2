# 架构与模块总览

CTP2 是单主程序架构：主可执行文件 `ctp2`（Windows）或 `ctp2/ctp2`（Linux），加上 4 个地图生成器 DLL 插件。所有模块以静态方式编入主程序。

## 模块依赖关系（自底向上）

```
os（平台抽象）
 └─ libs（anet 网络、freetype、SDL2/SDL_mixer/SDL_image、tiff、zlib、FFmpeg 可选）
     └─ gs（游戏状态核心） ← robot（寻路/序列化） ← net（网络同步）
         └─ gfx（图形渲染）、sound（音频）、ai（人工智能）
             └─ ui（用户界面，抽象层 + DirectX/SDL 双后端）
                 └─ ctp（主程序入口，组装一切）
```

## ctp2_code/ 各子模块

### ctp/ — 主程序
- 入口：`ctp/civ3_main.cpp`（`main`/`WinMain`），应用类 `CivApp`（`civapp.cpp`）
- `ctp/ctp2_utils/`：基础设施（`c3debug.h` 日志开关、`c3files` 文件系统、`c3errors` 错误、`appstrings` 字符串表、`c3cmdline` 命令行）
- `ctp/debugtools/`：断言、调用栈、内存调试、日志（log.cpp）
- `ctp/fingerprint/`：指纹/校验
- `ctp/dll/map/`：mapgen 插件加载目录（Linux 构建后自动复制 .so 过来）
- 也是可执行文件输出目录（`ctp/` 与 `ctp/x64/`）
- 工程文件：`civctp.sln`（VS 2017/2019）

### gs/ — 游戏状态（Game State，核心业务逻辑）
- `gs/gameobj/`：游戏对象，对象池模式。核心类：`Player`、`Army`/`ArmyPool`、`Unit`/`UnitPool`、`CityData`、`Civilisation`、`TradeRoute`/`TradeOffer`/`TradePool`、`TerrImprove`、`WonderTracker`、`Pollution` 等。数据类（`*Data.cpp`）与池类（`*Pool.cpp`）分离
- `gs/newdb/`：新数据库。schema 定义在 34 个 `.cdb` 文件（如 `unit.cdb`、`advance.cdb`、`government.cdb`），由代码生成器处理（见下方"数据管线"）
- `gs/database/`：旧数据库（`StrDB` 字符串库、`profileDB` 玩家配置、`PlayListDB`、`highscoredb` 等），与 newdb 并存
- `gs/dbgen/`：**代码生成器** `ctpdb`（flex/byacc：`ctpdb.l`、`ctpdb.y`），读取 `.cdb` 生成 `*Record.cpp/h`
- `gs/slic/`：**SLIC 脚本引擎**（游戏事件脚本语言）。`slic.l`/`slic.y`（flex/byacc 解析器）、`SlicEngine`、`SlicContext`、`SlicFunc` 等。脚本文件为 `.slc`（在 `ctp2_data/default/gamedata/` 下，如 `diplomacy.slc`）
- `gs/events/`：游戏事件系统（`GameEventManager`、`GameEventHook`），SLIC 通过它挂钩游戏事件
- `gs/world/`：地图世界（`MapPoint`、`Cell`、`wldgen` 世界生成、`WrldCity` 等）
- `gs/fileio/`：存档与路径（`GameFile`、`CivPaths`、`civscenarios`）
- `gs/utility/`：工具（`RandGen` 随机数、`TurnCnt` 回合计数、checksum）
- `gs/outcom/`：随机数（`c3rand`）

### ai/ — 人工智能
- `ai/ctpai.h/.cpp`：`CtpAi` 静态类，AI 总入口
- `ai/CityManagement/governor.cpp`：城市总督（城市建设管理）
- `ai/diplomacy/`：外交 AI（`Diplomat`、`Foreigner`、`ProposalAnalysis`、各种 `*Event`）
- `ai/strategy/`：军事战略（`scheduler/` 调度器、`agents/` 代理、`goals/` 目标、`squads/` 分队）
- `ai/mapanalysis/`：地图分析（`settlemap` 选址）
- `ai/personality/`：AI 个性（由 `personality.cdb` 数据驱动）

### ui/ — 用户界面
- `ui/aui_common/`：**UI 抽象基类**（`aui_Window`、`aui_Button`、`aui_ListBox` 等全套控件 + `aui_Factory` 工厂）
- `ui/aui_directx/`、`ui/aui_sdl/`：两个后端实现，构建时二选一
- `ui/aui_ctp2/`：CTP2 风格控件（`c3_*` 基础控件、`ctp2_*` 高级控件）
- `ui/interface/`：游戏界面窗口（`citymanager` 城市管理、`diplomacywindow` 外交、`greatlibrary` 大图书馆、`spnewgamewindow` 新游戏、`ChatBox.cpp` 聊天框/调试命令）
- `ui/ldl/`：**LDL 布局描述语言**（`ldl.l`/`ldl.y` flex/byacc 解析器），UI 布局由 `.ldl` 文件定义（在 `ctp2_data/default/uidata/`）
- `ui/netshell/`：多人联机大厅界面（`ns_*` 系列、`lobbywindow`）
- `ui/slic_debug/`：SLIC 调试界面

### gfx/ — 图形
- `gfx/tilesys/`：瓦片地图渲染（`tiledmap`、`tileset`、`tiledraw`）
- `gfx/spritesys/`：精灵系统（`Actor`、`Sprite`、`SpriteGroup`、`director` 动画导演）
- `gfx/layers/`：覆盖层（`citylayer`）
- `gfx/gfx_utils/`：像素/颜色/TGA/TIFF 工具

### net/ — 网络同步
- `net/io/`：传输层（`net_io`、`net_anet` 封装 anet 库、`net_thread` 线程）
- `net/general/`：每个游戏对象一个同步类（`net_city`、`net_unit`、`net_army`、`net_player` 等 40 余个），负责对象序列化广播

### robot/ — 机器人服务
- `robot/pathing/`：**A* 寻路**（`astar.cpp` 基础实现，`unitastar`、`CityAstar`、`TradeAstar` 三种用途变体）
- `robot/aibackdoor/`：`CivArchive` 序列化基类（存档/网络共用的对象序列化机制）
- `robot/utility/`：`roboinit` 初始化

### 其他模块
- `sound/`：音频（`soundmanager`、`civsound`、`gamesounds`、`soundevent`）
- `mapgen/`：**地图生成器 DLL 插件**（`Crater`、`FaultGen`、`Geometric`、`PlasmaGen2`，通过 `.def` 文件导出符号，运行时从 `ctp/dll/map/` 加载）
- `os/`：平台抽象（`os/nowin32/windows.h` 为非 Windows 平台提供 Win32 兼容层、`os/include/config_win32.h`；Linux 下 `config.h` 由 configure/meson 生成）
- `libs/`：第三方库。源码集成：`anet`（LGPL 网络库，meson 子项目）、`freetype-1.3.1`、`SDL2-2.30.12`、`SDL2_image`、`SDL2_mixer`、`tiff`、`zlib`、`FFmpeg-n6.1.2`（可选）；`miles` 为旧版音频库残留
- `compiler/`：msvc6/msvc8 旧工程文件残留，仅参考

## 数据管线（重要）

1. **数据库 schema → Record 类**：`gs/newdb/*.cdb`（34 个）→ `ctpdb` 代码生成器 → `*Record.cpp/h`（如 `unit.cdb` 生成 `UnitRecord` 等 6 个类）。构建系统自动执行此步骤（meson `custom_target`；autotools 类似）
2. **游戏数据**：`ctp2_data/default/gamedata/*.txt` 为文本格式的游戏数据（`Units.txt`、`buildings.txt`、`Advance.txt` 等），运行时加载；`.slc` 为 SLIC 脚本
3. **UI 布局**：`ctp2_data/default/uidata/*.ldl`，由 LDL 解析器加载
4. **语言目录**：`ctp2_data/{default,english,german,french,...}`，default 为基础，其余为各语言覆盖
5. **科技树图**：`Advance-Graph/`（graphviz 生成，`Makefile` 可重新生成）

## 序列化机制

`CivArchive`（`robot/aibackdoor/civarchive.cpp`）是所有可持久化对象的序列化基类，存档和网络游戏同步共用。net/general 下每个 `net_*` 类对应一种对象的网络序列化。

## 仓库辅助目录

- `Scenarios/`：4 个官方场景（AE_Mod、AlexanderTheGreat、MagnificentSamurai、NuclearDetente），含脚本与数据
- `doc/`：`doc/dev/ctp2_dev.tex`（LaTeX 开发者文档，可 make 出 PDF）、`doc/user/`（用户手册与 playtest 资料）、`doc/common/images`
- `templates/`：新源文件模板（c、cpp、h、cdb、l、y），与项目文件头惯例配套
- `Advance-Graph/`：科技树依赖图（graphviz `.gv` 源 + 生成的 PNG/SVG/PDF，`Makefile` 可重新生成）
- `ModTools/`：Mod 制作工具压缩包（tileedit、SpriteEdit、BMP2CTP2 等，非源码）
- `bin/`：Windows 用的 byacc/flex 及 Unix 工具链二进制（CDKDIR 环境变量指向此处）
- `NASM/`：Windows SDL 构建所需的 NASM 汇编器
- `debian/`、`etc/`（svn-config 历史残留）、`.civctp2/`（运行时生成的用户数据：存档/日志/配置）
- `.civctp2/logs`：游戏运行日志输出目录之一


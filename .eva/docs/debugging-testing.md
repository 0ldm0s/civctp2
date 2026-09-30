# 调试与测试

## 聊天窗口调试命令

游戏中按 `'`（撇号键）打开聊天窗口，可用命令（完整列表见 `ChatBox::CheckForEasterEggs`，位于 `ctp2_code/ui/interface/ChatBox.cpp`）：

- `/attach N` — 观察者模式附加到 N 号玩家
- `/rnd 200` — AI 自动快进 200 回合（AI 测试核心命令）
- `/reloadslic` — 重载 SLIC 脚本（旧存档脚本不兼容时使用）
- `/debugcells` — 显示寻路调试格（配合 `PRINT_COSTS` 编译标志）

## AI 测试流程（CONTRIBUTING.md 记载）

1. 新开游戏，不动任何单位直接存档
2. 聊天窗口输入 `/attach 1`（屏幕玩家为 1 号时）
3. 输入 `/rnd 200` 让 AI 玩 200 回合
4. 检查 AI 行为无异常跳变；旧存档先执行 `/reloadslic`

## 日志系统

- Logging 配置（VS）或 `--enable-logging`（Linux）启用
- 日志输出：`ctp2_code/ctp/logs`（Win32/Linux）或 `ctp2_code/ctp/x64/logs`（x64）
- 日志开关在 `civ3_main.cpp` 的 `main_InitializeLogs` 设置，标志定义在 `ctp/ctp2_utils/c3debug.h`
- 日志可还原 AI 决策和外交状态，AI 调试必备

## 内存泄漏报告

Debug 构建退出时生成 `CTP_LEAKS_99999.TXT`（栈自顶向下）和 `CTP_LEAKS_ALT_99999.TXT`（栈自底向上，便于对齐共同栈帧找到分叉点）在可执行文件目录。无泄漏时文件内容为 `None`。

## GUI 自动化测试（Sikulix）

- `tests/*.sikuli/`：Sikulix 图像识别测试脚本（Jython），依赖 `tests/*.png` 截图基准
- 场景：start-game、new-game、load-game、name-game、play-game_build-city、load-sprite、loop-sprite
- 本地复现参考 `.gitlab-ci.yml` test 阶段：Xvfb 虚拟显示 + `tools/run-DI.sh` 启动游戏 + jython 运行 `.sikuli` 脚本
- 无单元测试框架，GUI 测试是唯一的自动化测试

## 调试基础设施（代码内）

- `ctp/debugtools/`：断言（debugassert）、调用栈（debugcallstack）、异常捕获（debugexception）、内存跟踪（debugmemory）、计时器
- `ctp/ctp2_utils/c3errors.h`：错误报告宏
- `ctp/ctp2_utils/cheatkey.cpp`：作弊键处理
- `ui/interface/debugwindow.cpp`：游戏内调试窗口

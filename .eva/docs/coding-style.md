# 代码风格与贡献规范

来源：`CONTRIBUTING.md`（此文件为权威来源，本文件是摘要）。

## 命名约定（Activision 遗产，全项目强制）

- 函数、结构体、类：驼峰命名（camelCase），变量首字母小写
- 成员变量前缀 `m_`（如 `m_name`）
- 静态成员变量前缀 `s_`
- 全局变量前缀 `g_`（如 `g_theProfileDB`）
- 局部变量无前缀
- 枚举值全大写

## 缩进与空白

- **Tab 缩进**，Tab 宽度按 4 空格显示
- 对齐续行用空格
- 大规模空白整理必须单独成提交（white space only commit），严禁与功能修改混合
- 重新格式化时不得破坏已有对齐

## 提交与 PR

- 提交信息遵循 [How to Write a Git Commit Message](https://chris.beams.io/posts/git-commit/)
- PR 可在工作未完成时创建，标题以 `WIP` 开头，完成后移除
- 非 WIP PR 至少保留 2 周合并窗口（临近发布可缩短）

## 源文件头格式

每个源文件有固定注释头（项目惯例）：

```
//----------------------------------------------------------------------------
//
// Project      : Call To Power 2
// File type    : C++ source
// Description  : <一句话描述>
//
//----------------------------------------------------------------------------
//
// Disclaimer
//
// THIS FILE IS NOT GENERATED OR SUPPORTED BY ACTIVISION.
//
// This material has been developed at apolyton.net by the Apolyton CtP2
// Source Code Project.
//
//----------------------------------------------------------------------------
//
// Modifications from the original Activision code:
//
// - <修改记录，含日期和作者>
//
//----------------------------------------------------------------------------
```

新增 Apolyton 代码的修改应在 "Modifications" 段落追加记录（日期 + 作者），这是本项目追踪修改的惯例。

## 语言标准

- C++11 强制（configure.ac 中 `AX_CXX_COMPILE_STDCXX_11(noext, mandatory)`）
- 遗留代码为 MSVC 6 时代风格，含大量宏和原始指针——修改时保持局部风格一致，不做顺手重构

## 平台兼容注意事项

- 代码需同时兼容 MSVC 和 GCC（`_MSC_VER` / `__GNUC__` 条件编译常见）
- 文件路径大小写敏感（Linux 构建），include 必须与实际文件名大小写完全一致
- 非 Windows 平台经 `os/nowin32/windows.h` 兼容层提供 Win32 API 替代实现（`USE_COM_REPLACEMENT`）

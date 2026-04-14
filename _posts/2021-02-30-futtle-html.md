---
title: Flutter fix —— 自动修复废弃 API 的利器
author: morric
date: 2021-02-28 21:25:00 +0800
categories: [开发记录, Flutter]
tags: [Hybird, Flutter]
---

随着 Flutter 持续迭代，API 的废弃和更新越来越频繁。手动逐一排查和修改废弃 API 既耗时又容易遗漏。Flutter 官方为此提供了 `dart fix` 工具，可以自动检测并修复代码库中已弃用的 API，大幅降低升级成本。

该工具支持两种使用方式：通过命令行批量处理整个项目，或在 IDE 中逐个应用修复建议。

> 在 IntelliJ、Android Studio 中，这类自动修复功能称为 **quick-fixes**；在 VS Code 中称为 **code actions**。
{: .prompt-tip }

---

## 在 IDE 中逐个应用修复

### IntelliJ / Android Studio

当静态分析器（analyzer）检测到废弃 API 时，对应代码行左侧会出现一个**灯泡图标**。

点击灯泡图标，会弹出将代码更新为新 API 的修复建议列表。点击对应的建议，即可自动完成 API 的替换。

### VS Code

当分析器检测到废弃 API 时，代码下方会出现波浪线提示。可以通过以下任意一种方式触发修复：

- **悬停触发**：将鼠标悬停在报错位置，点击弹出菜单中的 `Quick Fix`，只展示代码修复选项；
- **灯泡图标**：将光标移至报错代码处，点击出现的灯泡图标，会展示包含重构操作在内的完整操作列表；
- **快捷键**：将光标移至报错代码处，按下快捷键（macOS 为 `Command + .`，其他平台为 `Control + .`），同样展示完整操作列表。

---

## 对整个项目批量应用修复

对于有大量废弃 API 需要更新的项目，逐个手动修复效率太低。`dart fix` 命令行工具可以扫描整个项目并批量处理。

**第一步：预览所有可修复的问题**

在执行实际修改前，建议先用 `--dry-run` 参数查看完整的变更列表，确认无误后再应用：

```bash
$ dart fix --dry-run
```

**第二步：批量应用所有修复**

确认变更内容后，执行以下命令一次性完成所有修复：

```bash
$ dart fix --apply
```

> 建议在执行 `dart fix --apply` 之前先提交当前代码到版本控制，方便对比修改内容或在出现问题时回滚。
{: .prompt-tip }

---

更多关于 Flutter 废弃 API 生命周期的详细说明，可参考 Flutter 官方在 Medium 上发布的[《Flutter 废弃 API 的周期》](https://medium.com/flutter)一文。
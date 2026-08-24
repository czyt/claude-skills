# Quickshell Reference（参考资料）

本文件只提炼编写 Omarchy QML 插件时常用的 Quickshell 线索，不是完整 API 镜像。抓取时间：2026-08-24；主要参考 [Quickshell v0.3.1 Usage Guide](https://quickshell.org/docs/v0.3.1/guide/)、[QML Language](https://quickshell.org/docs/v0.3.1/guide/qml-language/) 和 [类型索引](https://quickshell.org/docs/v0.3.1/types)。

## QML 基础

- 用 `import QtQuick` 引入 QML 基础类型；用 `import Quickshell` 引入 Quickshell 类型；按需引入 `Quickshell.Wayland`、`Quickshell.Io` 等模块。
- 用 `id` 引用对象；用 property binding 表达派生状态，不要在多个 signal handler 中复制同一份状态。
- 信号处理器写成 `onSignalName`，可用函数封装可复用行为；需要跨文件逻辑时使用相对路径的 JS 模块或 QML 组件。
- QML 布局不是 CSS 布局。先确认 anchors、implicit size、父子关系，再设置视觉细节。

## 对 Omarchy 插件有用的类型

| 类型/模块 | 用途 |
|---|---|
| `SystemClock` | 通过 `date` 提供时钟状态；设置 `precision` 控制更新粒度 |
| `Timer` | 周期性任务；明确 `interval`、`running`、`repeat` 和 `onTriggered` |
| `PanelWindow` | 独立 Wayland layer-shell 窗口；只在插件 contract 或确实需要的窗口类型中使用 |
| `Quickshell.Wayland` | `WlrLayershell`、layer、exclusive zone 等 Wayland 能力 |
| `Variants` | 为多屏或多个 model 实例化组件 |
| `Quickshell.Io` | 进程、文件和 IPC 相关能力；每个外部命令都要审查参数和权限 |

Omarchy 自己的 `BarWidget`、`Panel`、`KeyboardPanel`、`PanelKeyCatcher`、`WidgetButton`、`qs.Ui` 和 `qs.Commons` 是 Omarchy shell 提供的应用层类型，不是通用 Quickshell 类型。应以当前 Omarchy 源码为准。

## 版本边界

Quickshell API 和 Omarchy 的 `qs.*` 类型会随版本变化。遇到 `module/type not found`：

1. 读取当前 Omarchy shell 的 import 路径与 pinned Quickshell 版本。
2. 对照对应版本的官方类型文档。
3. 用 `qmllint` 验证，而不是把 standalone Quickshell 示例硬塞进 Omarchy plugin。

官方文档的示例和其他用户配置只作为思路参考；Omarchy 插件的 manifest、生命周期、bar 注入和 IPC 约束由 Omarchy 官方仓库决定。


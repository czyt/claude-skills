# Omarchy Plugin Reference（参考资料）

本文件是从 Omarchy 插件开发/发布页面和官方仓库提炼的工作参考，不替代当前安装版本的 shell 源码。抓取时间：2026-08-24。

## 优先级

1. 当前 Omarchy 安装中的 `shell/`、`services/PluginRegistry.qml` 和实际命令行为。
2. [Omarchy shell README](https://github.com/basecamp/omarchy/blob/quattro/shell/README.md)。
3. [官方插件 README](https://github.com/basecamp/omarchy/blob/quattro/shell/plugins/README.md)。
4. [Omarchy Plugins 开发指南](https://omarchyplugins.com/develop.html) 与 [发布指南](https://omarchyplugins.com/publish.html)。

## 运行模型

- `omarchy-shell` 是每个图形会话中的单个长驻 Quickshell 进程；bar、panel、overlay、menu 和 service 都作为插件运行在其中。
- 第三方插件位于 `~/.config/omarchy/plugins/<plugin-id>/`，安装的是根目录带 `manifest.json` 的 git 仓库。
- 插件无沙箱，以当前用户权限运行。安装器只负责 clone、校验 manifest 和切换启用状态，不应假设会执行插件安装脚本或 sudo。

## Manifest 速查

必需字段：`schemaVersion`、`id`、`name`、`version`、`author`、`description`、`kinds`、`entryPoints`。

| kind | entry point | 说明 |
|---|---|---|
| `bar-widget` | `barWidget` | 当前 bar 中的组件 |
| `panel` | `panel` | 可召唤或常驻的浮层 |
| `overlay` | `overlay` | 全屏覆盖层 |
| `menu` | `menu` | 召唤式菜单 |
| `service` | `service` | 无 UI 的单例 |
| `bar` | `bar` | 完整 bar 替换，同一时间只有一个生效 |

`entryPoints` 必须使用相对路径并与磁盘大小写完全一致。第三方 ID 不能使用 `omarchy.*`；插件目录不能包含 symlink。

## 官方实例

- [官方 Clock 插件目录](https://github.com/basecamp/omarchy/tree/quattro/shell/plugins/panels/clock)：`manifest.json`、`BarWidget.qml`、`Panel.qml`、`Model.js` 的完整 bar-widget + popup 实例。
- [官方 Clock BarWidget.qml](https://github.com/basecamp/omarchy/blob/quattro/shell/plugins/panels/clock/BarWidget.qml)：bar 注入、`Loader`、面板生命周期和 IPC 暴露方式。
- [官方 Clock manifest.json](https://github.com/basecamp/omarchy/blob/quattro/shell/plugins/panels/clock/manifest.json)：`barWidget` 配置、默认 section 和 schema 形式。
- [Marketplace development guide 的完成示例](https://omarchyplugins.com/develop.html#finished)：适合复制结构，不要复制其 ID、作者、仓库 URL 或描述。

## 常用命令

```bash
omarchy plugin clone omarchy.clock --edit
omarchy plugin validate "$HOME/.config/omarchy/plugins/<id>"
omarchy plugin list --json
omarchy-shell shell rescanPlugins
omarchy plugin enable <id>
omarchy plugin disable <id>
omarchy plugin add <git-url> --enable --yes
omarchy plugin update <id> --yes
omarchy plugin remove <id>
```

bar widget 的默认位置来自 `barWidget.defaultSection`，缺省时通常落在 center；可用 `omarchy bar move <id> --section <section>` 调整。


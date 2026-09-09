# 物理显示器 VT

物理显示器 VT 适用于需要把 Linux 控制台、Xorg、LightDM 等本地图形界面直接输出到微服物理显示器的应用。仅提供浏览器、VNC 或远程桌面的应用不需要启用。

## 前置条件

1. lzcos v1.6.1 或更高版本。
2. `package.yml` 声明 `vt.display`。
3. `lzc-manifest.yml` 设置 `application.vt: true`。
4. 所有 service 使用默认 `runc`；VT 不支持 `sysbox-runc`。
5. 如果图形程序还需要 GPU，另行配置 GPU 加速。

```yaml
# package.yml
package: cloud.lazycat.app.display-demo
version: 0.0.1
name: Display Demo
min_os_version: 1.6.1

permissions:
  required:
    - vt.display
```

```yaml
# lzc-manifest.yml
application:
  subdomain: display-demo
  vt: true
```

## 应用内接口

`app` 与所有 services 共享同一应用显示界面。应用只应使用以下兼容路径，不要依赖宿主动态分配的 `/dev/ttyN` 编号：

- `/dev/tty0`
- `/dev/tty7`

查询当前应用界面是否激活：

```bash
/lzcinit/vt.active status
```

| 结果 | stdout | 退出码 |
|------|--------|--------|
| 当前应用已显示 | `active` | `0` |
| 当前应用未显示 | `inactive` | `1` |
| 查询失败 | 错误写入 stderr | `2` |

退出码 `1` 是正常的“未激活”状态，不是接口故障。

激活当前应用界面：

```bash
/lzcinit/vt.active activate
```

该命令只能激活调用方所属的 lzcapp，不接受 VT 编号或其他应用标识。只有界面实际激活后才成功返回；不能把提交激活请求当成已显示成功。

## 系统分配与调试

- 宿主会从动态池分配 VT；应用重启后编号可能变化，不能保存或硬编码。
- 物理键盘按 `Ctrl+Alt+F1` 返回系统 VT 选择界面。
- SSH 调试时先运行 `hc vt list`，再使用当前输出给出的 `hc vt switch <number>`；列表变化后必须重新查询。

## 验证

1. 构建并安装 LPK。
2. 在任一 service 执行 `/lzcinit/vt.active status`。
3. 执行 `/lzcinit/vt.active activate`。
4. 再次查询，确认 stdout 为 `active` 且退出码为 `0`。
5. 确认物理显示器显示应用界面，并可通过 `Ctrl+Alt+F1` 返回系统选择界面。

## 常见错误

| 错误 | 修复 |
|------|------|
| 只设置 `application.vt: true` | 补充 `permissions.required: [vt.display]` |
| 找不到 `/lzcinit/vt.active` | 同时检查权限和 `application.vt` |
| `sysbox-runc` 不支持 VT | 所有 service 改用默认 `runc` |
| 硬编码宿主 `/dev/ttyN` | 应用内只使用 `/dev/tty0` 或 `/dev/tty7` |

官方来源：`docs/advanced-vt.md`、`docs/spec/manifest.md`、`docs/spec/package.md`（lzc-developer-doc，2026-08-05 同步）。

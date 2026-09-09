# 动态应用图标（lzcos v1.6.2+）

运行中的应用通过 PNG 文件更新启动器图标，无需修改静态安装包或新增 manifest 字段。`package.yml.min_os_version` 在依赖此功能时设置为 `1.6.2`。

| 项目 | 规则 |
|------|------|
| 目录 | `/lzcapp/run/launcher-icon/` |
| 部署级图标 | `icon.png`，对当前部署可见用户生效 |
| 用户专属 | `<uid>.png`，使用真实平台 UID；主要用于单实例，多实例通常只需 `icon.png` |
| 优先级 | `<uid>.png` → `icon.png` → 安装包 `pkg/icon.png` |
| 格式与大小 | PNG，每个文件不超过 1 MiB；其他文件名不识别 |
| 更新频率 | 系统最多处理每秒 4 次，超过约 4 FPS 的中间帧不显示 |
| 内容数量 | 尽量保持 24 个不同内容以内以利用缓存；这是建议，不是硬限制 |
| 生命周期 | 运行时目录不持久；应用 / LPK 重启后必须重新生成 |
| 生效范围 | 启动器；其他应用列表、系统设置和拖拽仍使用静态图标 |

在应用运行环境内，将已生成的有效 PNG 原子替换到目标文件：

```bash
set -eu
icon_dir=/lzcapp/run/launcher-icon
mkdir -p "$icon_dir"
# /tmp/new-icon.png 由应用生成，须为不超过 1 MiB 的 PNG。
tmp_file=$(mktemp "$icon_dir/.icon.XXXXXX")
trap 'rm -f "$tmp_file"' EXIT
cp /tmp/new-icon.png "$tmp_file"
mv -f "$tmp_file" "$icon_dir/icon.png"
```

按用户更新时将最终文件名换为真实 `<uid>.png`。先写临时文件再重命名，避免读到半写内容；只在内容变化时更新。删除当前动态图标后回退到下一层，恢复静态图标需要删除全部动态覆盖文件。

验证：用 `file` 和 `ls -l /lzcapp/run/launcher-icon/` 核对格式、大小和名称；在启动器确认变化，再测试删除回退和重启再生成。若不更新，检查系统版本、是否写在目标应用运行环境、是否存在优先级更高的用户图标。

来源：`lzc-developer-doc/docs/advanced-dynamic-icon.md`、`docs/changelog.md`（v1.6.2，2026-09-09）。

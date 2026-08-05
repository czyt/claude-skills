# LPK、LightOS 与 Docker 的边界

官方当前不再指导用户直接维护 lzcos 上的 Playground Docker、Dockge 或宿主 Docker socket。需要 Docker、Docker Compose 或完整 Linux 环境时，使用 LightOS。

## 选择模型

| 目标 | 选择 | 原因 |
|------|------|------|
| 向普通用户交付独立应用 | LPK | 可分发、可复现、一键安装，应用边界清晰 |
| 长期使用包管理器、工具链、shell、系统服务 | LightOS | 完整 Linux 环境和系统级状态按实例持久保存 |
| 运行 Docker / Docker Compose、自用 NAS 容器 | LightOS 内安装 Docker | 避免依赖 lzcos 宿主内部实现和高危 socket |
| 临时查看 lzcos | SSH | 系统为只读；系统级修改重启后会丢失 |

## LPK 原则

- 把前端、后端、路由和应用级数据封装为可复现包。
- 应用内部持久数据写入 `/lzcapp/var`，可清理缓存写入 `/lzcapp/cache`。
- 用户可理解和管理的文件才写入 `/lzcapp/documents/<uid>`。
- 不挂载宿主 Docker socket，不依赖 `pg-docker`、Playground daemon 或 lzcos 内部路径。
- 高级权限只按官方 `package.yml.permissions` 声明，避免用 `compose_override` 绕过权限边界。

## LightOS 原则

- 用户从应用商店安装 LightOS 入口应用，并在其中创建、管理实例。
- 在实例内按普通 Linux 流程安装 Docker、软件包、脚本和服务。
- 软件、系统配置与 Docker 数据跟随 LightOS 实例持久化。
- LightOS 权限较高，只开放给可信用户或可信管理应用。

## 迁移旧方案

遇到旧项目使用以下任一模式时，不要照抄旧教程：

- `/data/playground/docker.sock`
- `pg-docker`
- Dockge LPK 激活独立 Docker daemon
- 通过 `compose_override` 暴露宿主 Docker socket
- 依赖 lzcos SSH 修改持久保存

先判断项目是否仍应作为独立 LPK 分发；如果核心需求是维护容器环境或完整 Linux 状态，迁移到 LightOS。

官方来源：`docs/dockerd-support.md`、`docs/advanced-lightos.md`、`docs/framework.md`（lzc-developer-doc，2026-08-05 同步）。

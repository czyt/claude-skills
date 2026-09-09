# LPK、LightOS 与 Docker 的选择

LPK 用于分发独立、可复现的应用；LightOS 适合长期维护完整 Linux 环境。官方推荐 Docker 环境使用 LightOS，不等于社区或用户自用 LPK 禁止使用 `docker.sock`。

## 按交付目标选择

| 场景 | 处理 |
|------|------|
| 分发独立应用 | 制作 LPK，保留应用必需功能 |
| 长期维护包管理器、工具链、系统服务或 Docker 环境 | 推荐 LightOS；用户已选择 LPK 时按其需求继续 |
| 社区或自用应用明确需要宿主 Docker socket | 保留挂载，核实目标 daemon、实际路径和服务身份权限后生成配置 |
| 上架商店 | 单独核对当期审核规则；未查到明确条文时不声称 socket 被商店禁止，也不保证审核通过 |

## 社区 / 自用 socket 挂载

已有授权与路径信息直接复用；仅缺少实际 socket 路径或目标 daemon 时补充确认。不能仅因检测到 `docker.sock` 拒绝转换、强制迁移 LightOS，或删除依赖后交付功能残缺的 Portainer/Jenkins。

以下假定目标微服的 `/var/run/docker.sock` 已确认是要管理的 daemon socket，且 manifest 的服务名是 `portainer`。在 `lzc-build.yml` 通过 Compose 扩展保留挂载：

```yaml
manifest: ./lzc-manifest.yml
pkgout: ./
icon: ./icon.png
compose_override:
  services:
    portainer:
      volumes:
        - /var/run/docker.sock:/var/run/docker.sock
```

- 该宿主路径只是示例，不是所有 lzcos 版本的稳定接口。使用已核实的真实路径；旧 `/data/playground/docker.sock`、`pg-docker` 或 Playground daemon 不能假定仍存在。
- 在目标主机执行 `test -S /var/run/docker.sock`，检查 socket owner/group 与容器实际运行身份；部署后从应用内验证 daemon API 连通性及所需操作。
- socket 通常授予目标 Docker daemon 的管理能力；挂载 `:ro` 不会把 Docker API 变成只读。不要因权限失败直接加 `privileged: true`，先排查路径、UID/GID 与 daemon 状态。
- `compose_override` 的兼容性须在目标系统验证，不虚构 `docker.sock` permission id。构建通过不代表运行时挂载成功。

## 数据与环境

LPK 内部数据写 `/lzcapp/var`，缓存写 `/lzcapp/cache`；用户可管理的文件写 `/lzcapp/documents/<uid>`。LightOS 实例可持久保存软件、系统配置和 Docker 数据，只开放给可信用户。lzcos SSH 系统级修改重启后会丢失。

来源：`lzc-developer-doc/docs/dockerd-support.md`、`docs/advanced-lightos.md`、`docs/framework.md`。社区 / 自用 socket 处理是本 skill 的维护策略（2026-09-09 用户明确要求），不是上游新增承诺或商店审核结论。

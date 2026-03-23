# 懒猫微服底层架构与实现

本文档详细说明懒猫微服的底层 Docker 架构和实现机制，帮助开发者深入理解系统运作原理。

---

## 📋 目录

1. [三套 Docker 环境](#三套-docker-环境)
2. [DOCKER_HOST 封装机制](#docker_host-封装机制)
3. [daemon.json 配置详解](#daemonjson-配置详解)
4. [自定义 Docker 环境](#自定义-docker-环境)

---

## 三套 Docker 环境

懒猫微服内部运行着**三套独立的 Docker 环境**，每套环境有不同的用途和隔离级别。

### 环境列表

| Docker 命令 | 环境名称 | Socket 路径 | daemon.json 路径 | 用途 |
|------------|---------|-------------|------------------|------|
| `docker` | 系统 Docker | `/var/run/docker.sock` | `/etc/docker/daemon.json` | 系统组件专用 |
| `lzc-docker` | 商店 Docker | `/lzcsys/run/lzc-docker/docker.sock` | `/lzcsys/etc/docker/daemon.json` | 懒猫应用商店应用 |
| `pg-docker` | Playground Docker | `/data/playground/docker.sock` | `/lzcsys/var/playground/daemon.json` | 用户自定义容器 |

### 架构特点

**关键发现：** 懒猫并不是运行了三套完全独立的 Docker 服务，而是通过 **shell 脚本封装**，复用同一个 `docker` 客户端，切换不同的 socket 实现环境隔离。

```bash
# 查看命令类型
which docker      # /usr/bin/docker (ELF 可执行文件)
which lzc-docker  # /lzcsys/bin/lzc-docker (Shell 脚本)
which pg-docker   # /lzcsys/bin/pg-docker (Shell 脚本)

# 查看文件类型
file $(which docker)      # ELF 64-bit LSB pie executable
file $(which lzc-docker)  # Bourne-Again shell script
file $(which pg-docker)   # Bourne-Again shell script
```

---

## DOCKER_HOST 封装机制

### pg-docker 脚本实现

```bash
#!/bin/bash
set -e

export DOCKER_HOST=unix:///data/playground/docker.sock
exec docker "$@"
```

**工作原理：**

1. **设置环境变量**: `DOCKER_HOST=unix:///data/playground/docker.sock`
   - 使得后续 `docker` 命令连接到 `/data/playground/docker.sock`
   - 而不是默认的 `/var/run/docker.sock`

2. **exec 替换进程**: `exec docker "$@"`
   - `exec` 会用新进程替换当前脚本进程
   - `"$@"` 原样传递所有参数
   - 例如：`pg-docker ps -a` → `docker ps -a`（但使用不同的 socket）

### lzc-docker 脚本实现

```bash
#!/bin/bash
set -e

export DOCKER_HOST=unix:///lzcsys/run/lzc-docker/docker.sock
exec docker "$@"
```

**效果：**
- `lzc-docker ps` 只能看到懒猫应用商店的容器
- `pg-docker ps` 只能看到用户 Playground 的容器
- `docker ps` 只能看到系统组件容器

---

## daemon.json 配置详解

每套 Docker 环境都有独立的 `daemon.json` 配置文件，确保完全隔离。

### 配置文件位置

```bash
# 查找所有 daemon.json
sudo find / -type f -name daemon.json 2>/dev/null

# 结果：
/etc/docker/daemon.json                           # 系统 Docker
/lzcsys/etc/docker/daemon.json                    # 商店 Docker
/lzcsys/var/playground/daemon.json                # Playground Docker
```

### 1. 系统 Docker 配置

**路径**: `/etc/docker/daemon.json`

```json
{
  "registry-mirrors": [],
  "insecure-registries": [
    "registry.lazycat.cloud"
  ],
  "log-driver": "journald",
  "cgroup-parent": "sys_docker.slice"
}
```

**特点：**
- 最简洁的配置
- 使用系统默认路径
- 专用于系统组件

### 2. 商店 Docker 配置

**路径**: `/lzcsys/etc/docker/daemon.json`

```json
{
  "bridge": "none",
  "insecure-registries": [
    "registry.lazycat.cloud"
  ],
  "default-address-pools": [],
  "ipv6": true,
  "hosts": [
    "unix:///lzcsys/run/lzc-docker/docker.sock"
  ],
  "containerd-namespace": "lzc-docker",
  "containerd-plugins-namespace": "lzc-docker-plugins",
  "exec-root": "/lzcsys/run/lzc-docker/docker",
  "pidfile": "/lzcsys/run/lzc-docker/docker.pid",
  "data-root": "/lzcsys/run/data/system/docker",
  "cgroup-parent": "lzc_docker.slice"
}
```

**特点：**
- 独立的 `containerd-namespace`: `lzc-docker`
- 自定义 socket: `/lzcsys/run/lzc-docker/docker.sock`
- 数据目录: `/lzcsys/run/data/system/docker`
- 支持 IPv6

### 3. Playground Docker 配置

**路径**: `/lzcsys/var/playground/daemon.json`

```json
{
  "bridge": "",
  "containerd-namespace": "playground-docker",
  "containerd-plugins-namespace": "playground-docker",
  "data-root": "/data/playground/data/docker",
  "default-address-pools": [],
  "exec-root": "/data/playground/docker",
  "hosts": [
    "unix:///data/playground/docker.sock"
  ],
  "insecure-registries": [
    "registry.lazycat.cloud"
  ],
  "pidfile": "/data/playground/docker.pid"
}
```

**特点：**
- 独立的 `containerd-namespace`: `playground-docker`
- 用户可访问的 socket: `/data/playground/docker.sock`
- 数据目录: `/data/playground/data/docker`
- 用户可以自由管理容器

---

## 关键配置字段说明

### daemon.json 核心字段

| 字段 | 作用 | 示例 |
|------|------|------|
| `data-root` | Docker 数据存储目录（镜像、容器、卷） | `/data/playground/data/docker` |
| `exec-root` | 运行时数据目录 | `/data/playground/docker` |
| `pidfile` | Docker daemon 进程 PID 文件 | `/data/playground/docker.pid` |
| `hosts` | Docker daemon 监听的 socket | `unix:///data/playground/docker.sock` |
| `containerd-namespace` | Containerd 命名空间（容器隔离） | `playground-docker` |
| `containerd-plugins-namespace` | Containerd 插件命名空间 | `playground-docker` |
| `cgroup-parent` | Cgroup 父组（资源隔离） | `lzc_docker.slice` |
| `insecure-registries` | 允许不安全的镜像仓库 | `registry.lazycat.cloud` |
| `bridge` | Docker 网桥配置 | `none` 或留空 |

### 隔离机制总结

通过不同的配置字段实现多层隔离：

1. **文件系统隔离**: `data-root`, `exec-root` 不同
2. **进程隔离**: `pidfile`, `hosts` 不同
3. **命名空间隔离**: `containerd-namespace` 不同
4. **资源隔离**: `cgroup-parent` 不同

---

## 自定义 Docker 环境

### 使用场景

1. **开发测试**: 不影响生产环境的独立 Docker
2. **多项目隔离**: 不同项目使用不同 Docker 环境
3. **权限控制**: 普通用户可以管理自己的 Docker

### 快速部署脚本

创建 `init-docker-dev.sh`:

```bash
#!/usr/bin/env bash
# init-docker-dev.sh
set -e

BASE=$(realpath "./dev")
mkdir -p "$BASE"/{data,exec}

# 生成 daemon.json
cat > "$BASE/daemon.json" <<EOF
{
  "data-root": "$BASE/data",
  "exec-root": "$BASE/exec",
  "pidfile": "$BASE/docker.pid",
  "hosts": ["unix://$BASE/docker.sock"],
  "containerd-namespace": "dev-docker",
  "containerd-plugins-namespace": "dev-docker"
}
EOF

# 启动 dockerd
dockerd --config-file="$BASE/daemon.json" --log-level=info &

# 创建 dev-docker 命令
cat > "./dev-docker" <<EOF
#!/usr/bin/env bash
export DOCKER_HOST=unix://$BASE/docker.sock
exec docker "\$@"
EOF
chmod +x ./dev-docker

echo "🎉 Dev Docker 已就绪，使用 ./dev-docker 访问！"
```

### 使用步骤

```bash
# 1. 赋予执行权限
chmod +x init-docker-dev.sh

# 2. 运行脚本
./init-docker-dev.sh

# 3. 使用自定义 Docker
./dev-docker ps
./dev-docker pull ubuntu
./dev-docker run -it ubuntu bash
```

### 验证环境

```bash
# 查看 Docker 信息
sudo ./dev-docker info

# 输出示例：
# Docker Root Dir: /home/ubuntu/dev/data
# Containers: 0
# Images: 0
# Server Version: 26.1.3
```

---

## 应用开发实践

### 1. 容器可视化工具配置

在懒猫应用中挂载 Playground Docker socket：

```yaml
# lzc-manifest.yml
services:
  containly:
    image: registry.lazycat.cloud/u04123229/cloudsmithy/containly:30e4e3279afe9a52
    ports:
      - 5003:5000
    restart: unless-stopped

# lzc-build.yml
compose_override:
  services:
    containly:
      volumes:
        - /data/playground/docker.sock:/var/run/docker.sock
```

**重要：**
- ✅ 挂载 `/data/playground/docker.sock` 可以管理用户容器
- ⚠️ 挂载 `/var/run/docker.sock` 会管理系统容器（慎用！）
- ✅ 挂载 `/lzcsys/run/lzc-docker/docker.sock` 管理应用商店容器

### 2. Dockerd 支持应用

需要运行 Docker-in-Docker 的应用（如 Dockge）：

```yaml
services:
  dockge:
    image: louislam/dockge:latest
    runtime: sysbox-runc  # 必需
    binds:
      - /lzcapp/var/stacks:/opt/stacks

# lzc-build.yml
compose_override:
  services:
    dockge:
      volumes:
        - /data/playground/docker.sock:/var/run/docker.sock
```

---

## 管理与维护

### 查看 Docker 进程

```bash
ps aux | grep dockerd

# 输出示例：
# root   470  0.8  0.3  2653088  100248  /usr/bin/dockerd -H fd://
# root  2226  6.6  0.6  7246472  227108  /usr/bin/dockerd --config-file /lzcsys/etc/docker/daemon.json
# root 27520  0.0  0.2  2874220   90788  /usr/bin/dockerd --config-file /lzcsys/var/playground/daemon.json
```

### 停止自定义 Docker

**方法 1: 使用 PID**

```bash
# 查找进程
ps aux | grep dockerd

# 优雅停止
kill -15 <PID>
```

**方法 2: 使用配置文件匹配**

```bash
# 停止特定 Docker
pkill -f './dev/daemon.json'

# 验证
ps aux | grep dockerd
```

### 清除数据

```bash
# 删除数据目录
rm -rf ./dev

# 删除命令（如果安装到 PATH）
sudo rm -f /usr/local/bin/dev-docker
```

---

## 安全与最佳实践

### 1. Socket 权限

```bash
# Playground socket 权限较宽松，用户可访问
ls -l /data/playground/docker.sock
# srw-rw---- 1 root docker /data/playground/docker.sock

# 系统 socket 权限严格，仅 root 可访问
ls -l /var/run/docker.sock
# srw-rw---- 1 root root /var/run/docker.sock
```

### 2. 应用开发建议

- ✅ **推荐**: 挂载 `/data/playground/docker.sock` 用于用户容器管理
- ⚠️ **慎用**: 挂载 `/var/run/docker.sock` 可能影响系统稳定性
- ✅ **安全**: 使用 `runtime: sysbox-runc` 提供更好的隔离

### 3. 资源隔离

通过 `cgroup-parent` 实现资源隔离：

```json
{
  "cgroup-parent": "user_docker.slice"
}
```

这样可以通过 systemd cgroup 限制整个 Docker 环境的资源使用。

---

## 故障排查

### 问题 1: Socket 无法连接

```bash
# 检查 socket 是否存在
ls -l /data/playground/docker.sock

# 检查 dockerd 是否运行
ps aux | grep dockerd

# 检查 daemon.json 配置
cat /lzcsys/var/playground/daemon.json
```

### 问题 2: 容器无法启动

```bash
# 查看 dockerd 日志
journalctl -u docker -f

# 检查 data-root 磁盘空间
df -h /data/playground/data/docker
```

### 问题 3: 命名空间冲突

确保每个 Docker 环境使用不同的 `containerd-namespace`：

```json
{
  "containerd-namespace": "unique-name-here",
  "containerd-plugins-namespace": "unique-name-here"
}
```

---

## 总结

懒猫微服的多 Docker 环境设计：

1. **复用 Docker 客户端**: 通过 shell 脚本封装切换环境
2. **独立配置隔离**: 每个环境有独立的 daemon.json
3. **命名空间隔离**: Containerd namespace 确保容器互不干扰
4. **灵活可扩展**: 用户可以创建自己的 Docker 环境
5. **安全可控**: 通过不同 socket 权限控制访问

这种设计既保证了系统稳定性，又提供了用户灵活性，是一个优秀的架构实践。

---

**参考文档：**
- OFFICIAL_SPEC_REFERENCE.md - 官方规范参考
- ADVANCED_FEATURES.md - 高级功能文档
- dockerd-support.md - Dockerd 支持文档（官方）

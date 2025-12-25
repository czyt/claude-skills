# LazyCat 高级功能完整参考

## 📋 概述

本文档涵盖 LazyCat v1.4.1+ 的所有高级功能，包括 compose_override、资源限制、网络配置、文件处理等。

---

## 1. compose_override - 覆盖不支持的参数

### 作用
当 LazyCat 原生不支持某些 Docker Compose 参数时，通过 `compose_override` 在构建时注入这些参数。

### 使用场景
- 挂载 Docker socket（`/var/run/docker.sock`）
- 添加特殊设备映射
- 设置不支持的网络模式
- 任何需要在构建时覆盖的服务配置

### 语法格式

```yaml
# lzc-build.yml
compose_override:
  services:
    <service_name>:
      # 任何需要覆盖的 Docker Compose 参数
      volumes:
        - /host/path:/container/path
      devices:
        - /dev/ttyUSB0:/dev/ttyUSB0
      network_mode: host
```

### 实际项目示例

**来自 lucky-lzcapp:**
```yaml
# lzc-build.yml
compose_override:
  services:
    lucky:
      volumes:
        - /data/playground/docker.sock:/var/run/docker.sock
```

**完整项目结构:**
```yaml
# lzc-manifest.yml
name: Lucky
package: cloud.lazycat.app.lucky
version: 1.0.0
min_os_version: 1.3.8

services:
  lucky:
    image: lucky:latest
    environment:
      - DOCKER_HOST=unix:///var/run/docker.sock
    # 注意：这里没有 volumes，因为 volumes 会在 compose_override 中添加

# lzc-build.yml
compose_override:
  services:
    lucky:
      volumes:
        - /data/playground/docker.sock:/var/run/docker.sock
```

### 官方文档说明

**来源:** `/home/czyt/Desktop/lzc-developer-doc-master/docs/advanced-compose-override.md`

**关键点:**
1. `compose_override` 在 `lzc-build.yml` 中定义
2. 仅在构建时生效，不影响 manifest
3. 用于补充 LazyCat 不原生支持的 Docker Compose 功能
4. 优先级高于 manifest 中的配置

### 常见使用场景

#### 1. Docker Socket 挂载
```yaml
# lzc-build.yml
compose_override:
  services:
    portainer:
      volumes:
        - /var/run/docker.sock:/var/run/docker.sock
```

#### 2. 设备直通
```yaml
# lzc-build.yml
compose_override:
  services:
    zigbee2mqtt:
      devices:
        - /dev/ttyACM0:/dev/ttyACM0
```

#### 3. 网络模式
```yaml
# lzc-build.yml
compose_override:
  services:
    plex:
      network_mode: host
```

#### 4. 特权模式
```yaml
# lzc-build.yml
compose_override:
  services:
    privileged-app:
      privileged: true
```

### 注意事项

⚠️ **重要提醒:**
- `compose_override` 不支持所有 Docker Compose 参数
- 仅用于补充 LazyCat 原生不支持的功能
- 过度使用可能导致应用审核被拒
- 优先使用 LazyCat 原生支持的参数

---

## 2. 高级路由配置

### 2.1 upstreams（推荐）

**优势:**
- 支持路径路由
- 支持负载均衡
- 更灵活的配置

```yaml
application:
  subdomain: myapp
  upstreams:
    - location: /
      backend: http://myapp:8080/
    - location: /api
      backend: http://backend:3000/
    - location: /static
      backend: http://static:80/
```

**高级用法 - 负载均衡:**
```yaml
application:
  subdomain: myapp
  upstreams:
    - location: /
      backend:
        - http://app-1:8080/
        - http://app-2:8080/
        - http://app-3:8080/
      algorithm: round_robin  # 或 least_conn, ip_hash
```

### 2.2 routes（传统方式）

```yaml
application:
  subdomain: myapp
  routes:
    - /=http://myapp:8080/
    - /api/=http://backend:3000/
```

### 2.3 ingress（TCP/UDP）

```yaml
application:
  subdomain: myapp
  ingress:
    - protocol: tcp
      port: 22
      service: gitlab
      description: "SSH for Git operations"
    - protocol: tcp
      port: 5432
      service: postgres
      description: "PostgreSQL"
    - protocol: udp
      port: 51820
      service: wireguard
      description: "WireGuard VPN"
```

### 2.4 混合配置

```yaml
application:
  subdomain: myapp
  upstreams:
    - location: /
      backend: http://web:8080/
    - location: /api
      backend: http://api:3000/
  ingress:
    - protocol: tcp
      port: 22
      service: gitlab
```

---

## 3. 资源限制

### 3.1 CPU 配置

```yaml
services:
  app:
    image: myapp:latest
    # 方式 1: cpu_shares (相对权重，1024=100%)
    cpu_shares: 512  # 50% 权重

    # 方式 2: cpus (CPU 核数，v1.4.1+)
    cpus: 1.5  # 1.5 个 CPU 核心

    # 方式 3: cpu_quota (微秒，高级)
    cpu_quota: 50000  # 50% of 100000us
```

**推荐配置:**
```yaml
# 轻量应用
cpu_shares: 256  # 25%

# 标准应用
cpu_shares: 512  # 50%

# 重型应用
cpu_shares: 1024  # 100%
cpus: 2.0  # 2 核心
```

### 3.2 内存限制

```yaml
services:
  app:
    image: myapp:latest
    mem_limit: 1024M  # 最大 1GB
    mem_reservation: 512M  # 保留 512MB
```

**单位支持:**
- `b` - 字节
- `k` - KB
- `m` - MB
- `g` - GB

**示例:**
```yaml
mem_limit: 512M   # 512 MB
mem_limit: 2g     # 2 GB
mem_limit: 1024m  # 1024 MB
```

### 3.3 其他资源限制

```yaml
services:
  app:
    image: myapp:latest
    # 共享内存大小
    shm_size: 256m

    # 重启策略
    restart: unless-stopped  # 或: always, on-failure, no

    # 优先级
    cpu_priority: 10  # -20 到 19，默认 0

    # I/O 权重 (v1.4.1+)
    io_weight: 100  # 10-1000，默认 100
```

### 3.4 完整资源配置示例

```yaml
services:
  postgres:
    image: postgres:15
    cpu_shares: 512
    cpus: 1.0
    mem_limit: 2048M
    mem_reservation: 1024M
    shm_size: 256m
    restart: unless-stopped

  app:
    image: myapp:latest
    cpu_shares: 1024
    cpus: 2.0
    mem_limit: 4096M
    mem_reservation: 2048M
    restart: on-failure:5
```

---

## 4. 网络配置

### 4.1 网络模式

```yaml
services:
  app:
    image: myapp:latest
    network_mode: host  # 使用主机网络
    # 注意: 使用 host 模式时，ports 配置无效
```

**支持的模式:**
- `bridge` - 默认桥接模式
- `host` - 主机网络模式
- `none` - 无网络

### 4.2 特殊域名（内部服务发现）

LazyCat 提供特殊域名用于服务间通信：

| 域名 | 说明 | 示例 |
|------|------|------|
| `<service>.lzcapp` | 服务内部域名 | `postgres.lzcapp` |
| `host.lzcapp` | 宿主机域名 | 访问宿主机服务 |
| `_outbound` | 外部网络 | 访问互联网 |
| `_gateway` | 网关 | 网络网关 |

**示例:**
```yaml
services:
  app:
    image: myapp:latest
    environment:
      # 数据库连接
      - DATABASE_URL=postgresql://postgres:password@postgres.lzcapp:5432/db

      # 访问宿主机服务
      - HOST_API=http://host.lzcapp:8080/api

      # 外部 API
      - EXTERNAL_API=https://api.example.com
```

### 4.3 DNS 配置

```yaml
services:
  app:
    image: myapp:latest
    dns:
      - 8.8.8.8
      - 8.8.4.4
    dns_search:
      - example.com
    extra_hosts:
      - "host.docker.internal:host-gateway"
```

### 4.4 网络别名

```yaml
services:
  app:
    image: myapp:latest
    networks:
      - default
        aliases:
          - myapp-alias
          - web
```

---

## 5. 文件处理（File Handlers）

### 5.1 概述
文件处理允许应用处理特定文件类型，支持 MIME 类型和自定义处理逻辑。

### 5.2 语法格式

```yaml
application:
  subdomain: myapp
  file_handlers:
    - extension: .pdf
      mime_type: application/pdf
      handler: download  # 或: open, preview
    - extension: .mp4
      mime_type: video/mp4
      handler: stream
    - extension: .txt
      mime_type: text/plain
      handler: preview
```

### 5.3 支持的处理类型

| Handler | 说明 | 使用场景 |
|---------|------|----------|
| `download` | 强制下载 | 文档、压缩包 |
| `preview` | 浏览器预览 | 图片、文本 |
| `stream` | 流式播放 | 视频、音频 |
| `open` | 打开/执行 | 可执行文件 |

### 5.4 完整示例

```yaml
name: FileServer
package: cloud.lazycat.app.fileserver
version: 1.0.0
min_os_version: 1.3.8

application:
  subdomain: files
  upstreams:
    - location: /
      backend: http://fileserver:8080/

  file_handlers:
    # 文档类
    - extension: .pdf
      mime_type: application/pdf
      handler: download

    - extension: .doc
      mime_type: application/msword
      handler: download

    # 媒体类
    - extension: .mp4
      mime_type: video/mp4
      handler: stream

    - extension: .mp3
      mime_type: audio/mpeg
      handler: stream

    - extension: .jpg
      mime_type: image/jpeg
      handler: preview

    # 代码类
    - extension: .json
      mime_type: application/json
      handler: preview

    - extension: .md
      mime_type: text/markdown
      handler: preview

services:
  fileserver:
    image: fileserver:latest
    environment:
      - FILE_DIR=/data/files
    binds:
      - /lzcapp/var/files:/data/files
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:8080/health"]
      start_period: 30s
```

---

## 6. Setup Scripts & Initialization

### 6.1 setup_script

在应用启动前执行的初始化脚本。

```yaml
services:
  app:
    image: myapp:latest
    setup_script: |
      #!/bin/bash
      echo "初始化应用..."

      # 创建必要目录
      mkdir -p /data/config
      mkdir -p /data/logs

      # 检查并生成配置
      if [ ! -f /data/config/app.conf ]; then
        cat > /data/config/app.conf << EOF
      database_url=${DATABASE_URL}
      secret_key=${SECRET_KEY}
      EOF
      fi

      # 设置权限
      chown -R app:app /data

      echo "初始化完成"
```

### 6.2 init_script

容器初始化时执行（仅一次）。

```yaml
services:
  postgres:
    image: postgres:15
    init_script: |
      #!/bin/bash
      echo "首次初始化数据库..."

      # 创建初始数据库
      psql -U postgres -c "CREATE DATABASE myapp;"

      # 导入初始数据
      if [ -f /docker-entrypoint-initdb.d/init.sql ]; then
        psql -U postgres -d myapp -f /docker-entrypoint-initdb.d/init.sql
      fi
```

### 6.3 healthcheck_script

自定义健康检查脚本。

```yaml
services:
  app:
    image: myapp:latest
    healthcheck_script: |
      #!/bin/bash
      # 检查主服务
      curl -f http://localhost:8080/health || exit 1

      # 检查依赖服务
      curl -f http://postgres.lzcapp:5432 || exit 1

      # 检查资源使用
      MEMORY=$(free -m | awk 'NR==2{print $3/$2*100}')
      if (( $(echo "$MEMORY > 90" | bc -l) )); then
        echo "Memory usage too high: $MEMORY%"
        exit 1
      fi
```

---

## 7. 环境变量与模板渲染

### 7.1 模板变量语法

```yaml
# 内部服务变量（自动生成）
{{.INTERNAL.db_password}}
{{.INTERNAL.redis_password}}

# 用户配置变量
{{.U.jwt_secret}}
{{.U.admin_password}}

# 系统变量
{{.SYSTEM.domain}}
{{.SYSTEM.version}}
```

### 7.2 运行时环境变量

LazyCat 在运行时注入的环境变量：

| 变量名 | 说明 | 示例 |
|--------|------|------|
| `LAZYCAT_APP_ID` | 应用唯一ID | `app_123456` |
| `LAZYCAT_APP_NAME` | 应用名称 | `MyApp` |
| `LAZYCAT_APP_VERSION` | 应用版本 | `1.0.0` |
| `LAZYCAT_BOX_DOMAIN` | 盒子域名 | `mybox.lazycat.cloud` |
| `LAZYCAT_SUBDOMAIN` | 子域名 | `myapp` |
| `LAZYCAT_NETWORK_MODE` | 网络模式 | `bridge` |
| `LAZYCAT_INSTALL_PATH` | 安装路径 | `/lzcapp/var/myapp` |

**使用示例:**
```yaml
services:
  app:
    image: myapp:latest
    environment:
      - APP_ID=${LAZYCAT_APP_ID}
      - APP_NAME=${LAZYCAT_APP_NAME}
      - PUBLIC_URL=https://${LAZYCAT_SUBDOMAIN}.${LAZYCAT_BOX_DOMAIN}
```

### 7.3 环境变量引用

```yaml
services:
  app:
    image: myapp:latest
    environment:
      # 从文件读取
      - DB_PASSWORD_FILE=/run/secrets/db_password

      # 引用其他变量
      - DATABASE_URL=postgresql://postgres:${DB_PASSWORD}@postgres:5432/db

      # 默认值
      - LOG_LEVEL=${LOG_LEVEL:-INFO}

      # 数组
      - ALLOWED_HOSTS=localhost,127.0.0.1,${LAZYCAT_SUBDOMAIN}
```

---

## 8. 高级配置选项

### 8.1 ext_config（扩展配置）

```yaml
services:
  app:
    image: myapp:latest
    ext_config:
      # 自定义标签
      labels:
        - "com.example.version=1.0"
        - "com.example.description=My Application"

      # 额外的挂载
      tmpfs:
        - /tmp
        - /var/run

      # 安全选项
      security_opt:
        - no-new-privileges:true

      # 内核参数
      sysctls:
        - net.core.somaxconn=1024
        - vm.swappiness=0
```

### 8.2 handlers（事件处理）

```yaml
services:
  app:
    image: myapp:latest
    handlers:
      - event: on_start
        action: script
        script: |
          echo "服务启动中..."

      - event: on_stop
        action: script
        script: |
          echo "服务停止中..."
          curl -X POST http://localhost:8080/shutdown

      - event: on_health_failure
        action: restart
        max_retries: 3
```

### 8.3 platform（平台支持）

控制应用运行的平台架构：

```yaml
services:
  app:
    image: myapp:latest
    platform: linux/amd64  # 或: linux/arm64, linux/arm/v7
```

### 8.4 单实例 vs 多实例

```yaml
# 单实例（默认）
services:
  app:
    image: myapp:latest
    instances: 1  # 可省略

# 多实例（负载均衡）
services:
  app:
    image: myapp:latest
    instances: 3  # 启动 3 个实例
    environment:
      - INSTANCE_ID=${INSTANCE_ID}  # 每个实例唯一
```

---

## 9. USB/GPU/KVM 硬件加速

### 9.1 USB 设备直通

```yaml
services:
  zigbee2mqtt:
    image: koenkk/zigbee2mqtt
    devices:
      - /dev/ttyACM0:/dev/ttyACM0
    environment:
      - TZ=Asia/Shanghai
```

**高级配置（通过 compose_override）:**
```yaml
# lzc-build.yml
compose_override:
  services:
    zigbee2mqtt:
      devices:
        - /dev/serial/by-id/usb-Texas_Instruments_CC2531_USB_CDC-if00-port0:/dev/ttyACM0
```

### 9.2 GPU 加速

```yaml
# lzc-build.yml
compose_override:
  services:
    ai-app:
      deploy:
        resources:
          reservations:
            devices:
              - driver: nvidia
                count: 1
                capabilities: [gpu]
```

**支持的 GPU 厂商:**
- NVIDIA (`nvidia`)
- AMD (`amd`)
- Intel (`intel`)

### 9.3 KVM 虚拟化

```yaml
# lzc-build.yml
compose_override:
  services:
    vm-manager:
      volumes:
        - /dev/kvm:/dev/kvm
      devices:
        - /dev/kvm:/dev/kvm
      privileged: true
```

---

## 10. 完整高级应用示例

### 场景：带 GPU 加速的 AI 应用 + 数据库 + 文件处理

```yaml
# lzc-manifest.yml
name: AIImageGen
package: cloud.lazycat.app.aiimagegen
version: 1.0.0
min_os_version: 1.3.8
description: "AI Image Generation Service"
license: MIT

application:
  subdomain: aiimagegen
  upstreams:
    - location: /
      backend: http://web:8080/
    - location: /api
      backend: http://api:3000/
  file_handlers:
    - extension: .png
      mime_type: image/png
      handler: preview
    - extension: .jpg
      mime_type: image/jpeg
      handler: preview
  health_check:
    test_url: http://localhost:8080/health
    timeout: 10s
    start_period: 60s

services:
  postgres:
    image: postgres:15
    environment:
      - POSTGRES_DB={{.INTERNAL.db_name}}
      - POSTGRES_USER={{.INTERNAL.db_user}}
      - POSTGRES_PASSWORD={{.INTERNAL.db_password}}
    cpu_shares: 512
    mem_limit: 2048M
    binds:
      - /lzcapp/var/postgres:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 30s

  redis:
    image: redis:7-alpine
    command: redis-server --requirepass {{.INTERNAL.redis_password}} --appendonly yes
    cpu_shares: 256
    mem_limit: 512M
    binds:
      - /lzcapp/cache/redis:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 30s
      start_period: 10s

  api:
    image: aiimagegen-api:latest
    environment:
      - DATABASE_URL=postgresql://{{.INTERNAL.db_user}}:{{.INTERNAL.db_password}}@postgres.lzcapp:5432/{{.INTERNAL.db_name}}
      - REDIS_URL=redis://:{{.INTERNAL.redis_password}}@redis.lzcapp:6379/0
      - JWT_SECRET={{.U.jwt_secret}}
      - ADMIN_PASSWORD={{.U.admin_password}}
      - GPU_ENABLED={{.U.gpu_enabled}}
    depends_on:
      - postgres
      - redis
    cpu_shares: 1024
    cpus: 2.0
    mem_limit: 4096M
    mem_reservation: 2048M
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:3000/health"]
      start_period: 30s
      timeout: 5s

  web:
    image: aiimagegen-web:latest
    environment:
      - API_URL=http://api:3000
      - PUBLIC_URL=https://{{.SYSTEM.subdomain}}.{{.SYSTEM.domain}}
    depends_on:
      - api
    cpu_shares: 256
    mem_limit: 512M
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:8080/health"]
      start_period: 30s

locales:
  zh:
    name: "AI 图像生成器"
    description: "基于 GPU 加速的 AI 图像生成服务"
```

```yaml
# lzc-deploy-params.yml
params:
  - id: jwt_secret
    type: string
    name: "JWT secret key"
    description: "Secret key for authentication (min 32 chars)"
    optional: false
    regex: "^.{32,}$"
    regex_message: "Minimum 32 characters required"

  - id: admin_password
    type: string
    name: "admin password"
    description: "Administrator password"
    optional: false
    regex: "^.{8,}$"
    regex_message: "Minimum 8 characters required"

  - id: gpu_enabled
    type: bool
    name: "enable GPU"
    description: "Enable GPU acceleration (requires NVIDIA GPU)"
    default_value: false
    optional: true

  - id: max_concurrent
    type: number
    name: "max concurrent jobs"
    description: "Maximum number of concurrent image generation jobs"
    default_value: 2
    optional: true
    min: 1
    max: 10

locales:
  zh:
    jwt_secret:
      name: "JWT 密钥"
      description: "用于身份验证的密钥（至少 32 字符）"
    admin_password:
      name: "管理员密码"
      description: "管理员账户密码"
    gpu_enabled:
      name: "启用 GPU"
      description: "启用 GPU 加速（需要 NVIDIA GPU）"
    max_concurrent:
      name: "最大并发数"
      description: "最大并发图像生成任务数"
```

```yaml
# lzc-build.yml
manifest: ./lzc-manifest.yml
pkgout: ./
icon: ./icon.png

# GPU 支持和 Docker socket 挂载
compose_override:
  services:
    api:
      deploy:
        resources:
          reservations:
            devices:
              - driver: nvidia
                count: 1
                capabilities: [gpu]
      volumes:
        - /var/run/docker.sock:/var/run/docker.sock  # 用于容器管理
```

---

## 11. 最佳实践检查清单

### 发布前检查

- [ ] **Manifest 格式**
  - [ ] 使用 `healthcheck`（无下划线）用于服务
  - [ ] 使用 `health_check.test_url`（带下划线）用于应用
  - [ ] 包含 `min_os_version: 1.3.8`
  - [ ] 不包含 `lzc-sdk-version`
  - [ ] 推荐使用 `upstreams` 代替 `routes`

- [ ] **资源限制**
  - [ ] 设置合理的 `cpu_shares` 或 `cpus`
  - [ ] 设置 `mem_limit` 防止内存溢出
  - [ ] 为数据库设置 `mem_reservation`

- [ ] **存储路径**
  - [ ] 永久数据使用 `/lzcapp/var`
  - [ ] 缓存数据使用 `/lzcapp/cache`
  - [ ] 不使用相对路径或绝对路径

- [ ] **安全配置**
  - [ ] 使用 `{{.INTERNAL.xxx}}` 生成内部密码
  - [ ] 使用 `{{.U.xxx}}` 处理用户配置
  - [ ] 不硬编码敏感信息
  - [ ] 避免使用 `privileged: true`

- [ ] **健康检查**
  - [ ] 所有服务都有健康检查
  - [ ] 设置合理的 `start_period`
  - [ ] 使用正确的检测命令

- [ ] **高级功能**
  - [ ] 如需特殊参数，使用 `compose_override`
  - [ ] 如需文件处理，配置 `file_handlers`
  - [ ] 如需 TCP/UDP，使用 `ingress`

- [ ] **Setup Wizard**
  - [ ] `params` 使用英文 ID
  - [ ] `params.name/description` 使用英文
  - [ ] `locales` 提供中文翻译
  - [ ] 必填参数设置 `optional: false`
  - [ ] 敏感参数设置 `regex` 验证

---

## 12. 故障排除

### 常见问题

**1. 健康检查失败**
```yaml
# 检查 start_period 是否足够
healthcheck:
  start_period: 60s  # 增加等待时间
  interval: 30s
  retries: 5
```

**2. 内存不足**
```yaml
# 增加内存限制
mem_limit: 8192M  # 8GB
mem_reservation: 4096M  # 4GB 保留
```

**3. compose_override 不生效**
- 确认在 `lzc-build.yml` 中定义
- 检查参数名称是否正确
- 查看构建日志

**4. GPU 不可用**
- 确认硬件支持
- 检查驱动安装
- 验证 `compose_override` 配置

---

## 13. 参考文档

### 官方文档
- **Manifest 规范**: `/home/czyt/Desktop/lzc-developer-doc-master/docs/spec/manifest.md`
- **Changelog v1.4.1**: `/home/czyt/Desktop/lzc-developer-doc-master/docs/changelogs/v1.4.1.md`
- **Compose Override**: `/home/czyt/Desktop/lzc-developer-doc-master/docs/advanced-compose-override.md`
- **Setup Wizard**: `/home/czyt/Desktop/lzc-developer-doc-master/docs/spec/deploy-params.md`

### 实际项目
- **Blinko**: `~/code/lazycat/blinko-lzcapp/`
- **Yuque Sync**: `~/code/lazycat/yuque-sync-lzcapp/`
- **Lucky**: `~/code/lazycat/lucky-lzcapp/`
- **New API**: `~/code/lazycat/new-api-lzcapp/`

### 你的博客
- `/home/czyt/Documents/blog/content/post/simple-guide-for-developing-for-lazycat-nas.md`

---

**最后更新**: 2025-12-25
**版本**: LazyCat v1.4.1+
**状态**: ✅ 完整参考文档

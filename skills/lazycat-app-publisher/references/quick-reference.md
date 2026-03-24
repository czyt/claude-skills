# LazyCat 应用发布 - 快速参考指南

> 注：本文中 `package.yml` / `pkg_id` / `pkg_name` 相关内容以 `lzcos v1.5.0+`、`LPK v2`、`lzc-cli v2.0.0+` 为前提。

## 🎯 核心要点（v1.4.1+）

### 1. 完整 lzc-build.yml 示例

```yaml
# manifest: 指定 lpk 包的 manifest.yml 文件路径
manifest: ./lzc-manifest.yml

# pkgout: lpk 包的输出路径
pkgout: ./

# icon: 指定 lpk 包 icon 的路径（必须是 PNG 格式）
icon: ./icon.png

# pkg_id/pkg_name: 可选，覆盖 package.yml 中的 package/name
# pkg_id: cloud.lazycat.app.myapp.dev
# pkg_name: MyApp Dev

# contentdir: 可选，额外内容目录
# contentdir: ./dist

# compose_override: 覆盖不支持的参数（可选）
compose_override:
  services: {}
```

**关键字段：**
- ✅ **manifest** (必需): 指向 lzc-manifest.yml
- ✅ **pkgout** (必需): 输出目录
- ✅ **icon** (必需): 512x512 PNG 图标
- ✅ **pkg_id / pkg_name** (可选): 构建时覆盖 `package.yml`
- ✅ **compose_override** (可选): 覆盖不支持参数

### 2. Package + Manifest 结构关键变更

| 项目 | 旧格式 | 新格式 | 状态 |
|------|--------|--------|------|
| `lzc-sdk-version` | ✅ 存在 | ❌ 移除 | 必须删除 |
| 静态元数据位置 | `lzc-manifest.yml` 顶层 | `package.yml` | ✅ `LPK v2` 推荐 |
| `min_os_version` | manifest 顶层 | `package.yml` | ✅ `LPK v2` 应迁移 |
| `services.healthcheck` | `health_check` | `healthcheck` | ✅ 新标准 |
| `services.healthcheck.timeout` | ❌ 不支持 | ✅ 支持 | v1.4.1 新增 |
| `application.upstreams` | `routes` | `upstreams` | ⭐ 推荐使用 |

### 2. 健康检查字段对照

```yaml
# ❌ 旧格式 (pre-v1.4.1)
services:
  postgres:
    health_check:  # 已废弃
      test: ["CMD", "pg_isready"]
      start_period: 30  # 无单位

# ✅ 新格式 (v1.4.1+) - 推荐多行格式
services:
  postgres:
    healthcheck:  # 与 Docker Compose 100% 兼容
      test:
        - CMD
        - pg_isready
      start_period: 30s  # 必须带单位
      timeout: 10s  # v1.4.1 新增

# ✅ 备选格式 (JSON 数组，适合简单命令)
services:
  postgres:
    healthcheck:
      test: ["CMD", "pg_isready"]
      start_period: 30s
      timeout: 10s
```

### 3. 完整 Package + Manifest 示例（`LPK v2`）

```yaml
# package.yml
package: cloud.lazycat.app.myapp
version: 1.0.0
name: MyApp
description: "My application"
license: MIT
min_os_version: 1.3.8
```

```yaml
# lzc-manifest.yml

application:
  subdomain: myapp
  upstreams:  # ⭐ 推荐
    - location: /
      backend: http://myapp:8080/

services:
  postgres:
    image: postgres:15
    environment:
      - POSTGRES_PASSWORD={{.INTERNAL.db_password}}
    healthcheck:  # ✅ 无下划线
      test:
        - CMD-SHELL
        - pg_isready -U postgres
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 30s
    binds:
      - /lzcapp/var/db:/var/lib/postgresql/data

  app:
    image: myapp:latest
    environment:
      - DATABASE_URL=postgresql://postgres:{{.INTERNAL.db_password}}@postgres:5432/app
      - SECRET_KEY={{.U.secret_key}}
    depends_on:
      - postgres
    healthcheck:
      test:
        - CMD-SHELL
        - curl -f http://localhost:8080/health
      start_period: 30s
```

---

## 🔧 高级功能快速参考

### compose_override（覆盖不支持参数）

**使用场景：** Docker socket、设备直通、特权模式等

```yaml
# lzc-build.yml
compose_override:
  services:
    app:
      volumes:
        - /var/run/docker.sock:/var/run/docker.sock
      devices:
        - /dev/ttyUSB0:/dev/ttyUSB0
      privileged: true
```

**完整项目示例：**
```yaml
# lzc-manifest.yml
services:
  lucky:
    image: lucky:latest
    environment:
      - DOCKER_HOST=unix:///var/run/docker.sock

# lzc-build.yml
compose_override:
  services:
    lucky:
      volumes:
        - /data/playground/docker.sock:/var/run/docker.sock
```

### 资源限制

```yaml
services:
  app:
    cpu_shares: 512  # 50% 权重 (1024=100%)
    cpus: 1.5        # 1.5 个 CPU 核心
    mem_limit: 2048M # 最大 2GB
    mem_reservation: 1024M  # 保留 1GB
    shm_size: 256m   # 共享内存
    restart: unless-stopped  # 重启策略
```

### 高级路由

```yaml
application:
  subdomain: myapp
  upstreams:  # ⭐ 推荐
    - location: /
      backend: http://web:8080/
    - location: /api
      backend: http://api:3000/
  ingress:  # TCP/UDP
    - protocol: tcp
      port: 22
      service: gitlab
```

### 特殊域名（服务发现）

| 域名 | 用途 | 示例 |
|------|------|------|
| `<service>.lzcapp` | 服务间通信 | `postgres.lzcapp:5432` |
| `host.lzcapp` | 访问宿主机 | `host.lzcapp:8080` |
| `_outbound` | 外部网络 | 访问互联网 |
| `_gateway` | 网关 | 网络网关 |

```yaml
services:
  app:
    environment:
      - DB_URL=postgresql://postgres.lzcapp:5432/db
      - HOST_API=http://host.lzcapp:8080/api
```

### 文件处理

```yaml
application:
  file_handlers:
    - extension: .pdf
      mime_type: application/pdf
      handler: download  # download, preview, stream, open
    - extension: .mp4
      mime_type: video/mp4
      handler: stream
```

### 环境变量模板

```yaml
services:
  app:
    environment:
      # 内部服务（自动生成）
      - DB_PASSWORD={{.INTERNAL.db_password}}

      # 用户配置
      - JWT_SECRET={{.U.jwt_secret}}

      # 运行时变量
      - APP_ID=${LAZYCAT_APP_ID}
      - PUBLIC_URL=https://${LAZYCAT_SUBDOMAIN}.${LAZYCAT_BOX_DOMAIN}
```

### 多实例负载均衡

```yaml
services:
  app:
    instances: 3  # 启动 3 个实例
    environment:
      - INSTANCE_ID=${INSTANCE_ID}  # 每个实例唯一
```

### Setup Scripts

```yaml
services:
  app:
    setup_script: |
      #!/bin/bash
      mkdir -p /data/config
      chown -R app:app /data
```

---

## 📋 发布流程

### Stage 1: 初始构建
```bash
lzc-cli project build -o app-1.0.0.lpk
```

### Stage 2: 镜像复制
```bash
lzc-cli appstore copy-image myapp:latest
# 输出: registry.lazycat.cloud/czyt/myapp:HASH
```

### Stage 3: 更新 Manifest
```bash
# 自动更新 lzc-manifest.yml 中的 image
# 旧: image: myapp:latest
# 新: image: registry.lazycat.cloud/czyt/myapp:HASH
```

### Stage 4: 重新构建 + 发布
```bash
lzc-cli project build -o app-1.0.1.lpk
lzc-cli appstore publish app-1.0.1.lpk
```

### 使用 build.sh 自动化脚本
```bash
# 查看应用信息
./build.sh info

# 构建应用
./build.sh build

# 复制镜像（需要登录）
./build.sh copy

# 发布应用（需要登录）
./build.sh publish

# 一键全流程（推荐）
./build.sh one-click
```

**⚠️ 登录状态检查：**
- build.sh 使用 `lzc-cli appstore my-images` 检查登录状态
- 如果未登录，会提示执行 `lzc-cli appstore login`
- 登录后即可使用复制镜像和发布功能

### ⚠️ 服务名称保留字

**重要：服务名称不能使用 `app`**

```yaml
# ❌ 错误 - 会报错
services:
  app:  # 保留名称！
    image: myapp:latest

# ✅ 正确 - 重命名
services:
  web:  # 或 api, backend, myapp-app
    image: myapp:latest
```

**保留名称：**
- ❌ `app` - LazyCat 保留
- ✅ 推荐：`web`, `api`, `backend`, `<name>-app`

**自动重命名规则：**
- Docker Compose 的 `app` → `app-service`
- 或使用更有意义的名称如 `web`, `api`, `backend`

---

## ⚠️ 常见错误

### ❌ 错误 1: 健康检查字段名
```yaml
# ❌ 错误
services:
  postgres:
    health_check:  # 应该用 healthcheck
      test: ["CMD", "pg_isready"]

# ✅ 正确
services:
  postgres:
    healthcheck:  # 无下划线
      test: ["CMD", "pg_isready"]
```

### ❌ 错误 2: 缺少 min_os_version
```yaml
# ❌ 错误
name: MyApp
# 缺少 min_os_version

# ✅ 正确
name: MyApp
min_os_version: 1.3.8
```

### ❌ 错误 3: 使用 lzc-sdk-version
```yaml
# ❌ 错误
lzc-sdk-version: "0.1"  # 已移除
name: MyApp

# ✅ 正确
name: MyApp
min_os_version: 1.3.8
```

### ❌ 错误 4: 体积路径错误
```yaml
# ❌ 错误
binds:
  - ./data:/data  # 相对路径
  - /home/user:/data  # 绝对路径

# ✅ 正确
binds:
  - /lzcapp/var/data:/data  # 永久数据
  - /lzcapp/cache/temp:/tmp  # 缓存
```

### ❌ 错误 5: Setup Wizard 参数格式
```yaml
# ❌ 错误
params:
  - id: YUQUE_TOKEN
    name: "语雀 Token"  # 应该用英文
    description: "API说明"  # 应该用英文

# ✅ 正确
params:
  - id: yuque_token  # 推荐小写
    name: "yuque token"  # 英文
    description: "API Token for Yuque"  # 英文
locales:
  zh:
    yuque_token:
      name: "语雀 Token"
      description: "语雀 API Token说明"
```

---

## ✅ 发布前检查清单

### Manifest 格式
- [ ] 没有 `lzc-sdk-version`
- [ ] 有 `min_os_version: 1.3.8`
- [ ] 服务使用 `healthcheck`（无下划线）
- [ ] 应用使用 `health_check.test_url`（带下划线）
- [ ] 推荐使用 `upstreams` 代替 `routes`

### 资源配置
- [ ] 设置 `cpu_shares` 或 `cpus`
- [ ] 设置 `mem_limit`
- [ ] 为数据库设置 `mem_reservation`

### 存储路径
- [ ] 永久数据使用 `/lzcapp/var`
- [ ] 缓存数据使用 `/lzcapp/cache`
- [ ] 不使用相对路径或绝对路径

### 安全配置
- [ ] 内部服务使用 `{{.INTERNAL.xxx}}`
- [ ] 用户配置使用 `{{.U.xxx}}`
- [ ] 不硬编码敏感信息

### 健康检查
- [ ] 所有服务都有健康检查
- [ ] 设置合理的 `start_period`
- [ ] 时间字段带单位（如 `30s`）

### Setup Wizard
- [ ] `params` 使用英文 ID
- [ ] `params.name/description` 使用英文
- [ ] `locales` 提供中文翻译
- [ ] 必填参数设置 `optional: false`

### 高级功能
- [ ] 如需特殊参数，使用 `compose_override`
- [ ] 如需文件处理，配置 `file_handlers`
- [ ] 如需 TCP/UDP，使用 `ingress`

---

## 📚 完整文档索引

| 文档 | 内容 |
|------|------|
| **SKILL.md** | 主技能文档，完整使用指南 |
| **SKILL-INTELLIGENT.md** | 智能分析逻辑和算法 |
| **HEALTHCHECK_REFERENCE.md** | 健康检查完整参考 |
| **MANIFEST_REFERENCE.md** | Manifest 格式完整参考 |
| **ADVANCED_FEATURES.md** | 高级功能（compose_override、资源、网络、文件处理等） |
| **DEVELOPER_GUIDE.md** | 你的博客文章 |
| **QUICK_REFERENCE.md** | 本文档，快速查找 |

---

## 🎯 一句话总结

**记住 3 个核心：**
1. **健康检查**：用 `healthcheck`（无下划线），与 Docker Compose 一致
2. **Manifest**：无 `lzc-sdk-version`，必须有 `min_os_version: 1.3.8`
3. **高级参数**：不支持的用 `compose_override` 在 `lzc-build.yml` 中覆盖

---

**最后更新**: 2025-12-25
**版本**: LazyCat v1.4.1+
**状态**: ✅ 所有高级功能已整合

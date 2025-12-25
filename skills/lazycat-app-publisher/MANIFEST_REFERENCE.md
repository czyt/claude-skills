# LazyCat Manifest 配置参考 (v1.4.1+)

## 📋 最新格式标准

### ✅ 完整 Manifest 示例（最新推荐）

```yaml
# ✅ CORRECT - LazyCat v1.4.1+ 最新格式
name: MyApp
package: cloud.lazycat.app.myapp
version: 1.0.0
min_os_version: 1.3.8  # ✅ 必须添加
description: "My application description"
license: MIT
homepage: https://github.com/your/app
author: Your Name

application:
  subdomain: myapp
  background_task: true  # 对于非 HTTP 应用
  upstreams:  # ✅ 推荐使用 upstreams
    - location: /
      backend: http://myapp:8080/
  # 对于 TCP/UDP 服务：
  # ingress:
  #   - protocol: tcp
  #     port: 22
  #     service: gitlab

services:
  postgres:
    image: postgres:15
    environment:
      - POSTGRES_PASSWORD={{.INTERNAL.db_password}}
    healthcheck:  # ✅ v1.4.1: 无下划线，与 Docker Compose 兼容
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 30s
    binds:
      - /lzcapp/var/db:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    command: redis-server --requirepass {{.INTERNAL.redis_password}}
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 30s
      start_period: 10s
    binds:
      - /lzcapp/cache/redis:/data

  app:
    image: myapp:latest
    environment:
      - DATABASE_URL=postgresql://postgres:{{.INTERNAL.db_password}}@postgres:5432/app
      - REDIS_URL=redis://:{{.INTERNAL.redis_password}}@redis:6379/0
      - SECRET_KEY={{.U.secret_key}}
    depends_on:
      - postgres
      - redis
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:8080/health || exit 1"]
      start_period: 30s

locales:
  zh:
    name: "我的应用"
    description: "我的应用程序描述"
```

---

## 🎯 关键变更总结

### 1. **command 字段格式**
| 类型要求 | 说明 |
|----------|------|
| ✅ **必须是字符串** | LazyCat manifest 要求 command 必须是 string 类型 |
| ❌ **不能是数组** | 即使 Docker Compose 支持数组格式，LazyCat 也不支持 |

**错误示例（数组格式）：**
```yaml
# ❌ 错误 - LazyCat 不支持数组格式的 command
services:
  app:
    command:
      - sh
      - -c
      - sleep 15 && bun run server.js

# 错误信息：
# 'services[app].command' expected type 'string', got unconvertible type '[]interface {}'
```

**正确示例（字符串格式）：**
```yaml
# ✅ 正确 - 使用字符串格式
services:
  app:
    command: sh -c 'sleep 15 && bun run server.js'

# ✅ 正确 - 简单命令
services:
  redis:
    command: redis-server --requirepass {{.INTERNAL.redis_password}}
```

**注意事项：**
1. **引号使用**：
   - 如果命令中包含 `&&`、`||`、管道等 shell 操作符，建议用单引号包裹
   - 例如：`sh -c 'sleep 15 && node server.js'`

2. **sleep 命令**：
   - 某些镜像（如基于 BusyBox）不支持 `sleep 15s` 格式
   - 应使用 `sleep 15`（数字默认单位为秒）

3. **Docker Compose 兼容性**：
   - Docker Compose 同时支持字符串和数组格式
   - LazyCat 仅支持字符串格式
   - 转换时需要将数组格式改为字符串

**转换规则：**
```yaml
# Docker Compose（数组格式）→ LazyCat（字符串格式）

# 示例 1：简单转换
# Docker Compose:
command:
  - redis-server
  - --requirepass
  - mypassword
# LazyCat:
command: redis-server --requirepass mypassword

# 示例 2：shell 命令转换
# Docker Compose:
command:
  - sh
  - -c
  - sleep 15 && npm start
# LazyCat:
command: sh -c 'sleep 15 && npm start'

# 示例 3：复杂命令转换
# Docker Compose:
command:
  - /bin/bash
  - -c
  - |
    sleep 10
    npm run migrate
    npm start
# LazyCat:
command: /bin/bash -c 'sleep 10 && npm run migrate && npm start'
```

### 2. **lzc-sdk-version 字段**
| 状态 | 说明 |
|------|------|
| ❌ **已移除** | 最新项目不再使用 |
| ✅ **替代** | 无需任何替代字段 |

**对比：**
```yaml
# ❌ 旧格式（已废弃）
lzc-sdk-version: "0.1"
name: MyApp
package: cloud.lazycat.app.myapp
version: 1.0.0

# ✅ 新格式（推荐）
name: MyApp
package: cloud.lazycat.app.myapp
version: 1.0.0
min_os_version: 1.3.8  # 必须添加
```

### 2. **min_os_version 字段**
| 状态 | 说明 |
|------|------|
| ✅ **必须添加** | 新版应用必需 |
| ✅ **推荐值** | `1.3.8` |

**来源：**
- 您的博客文章第 221 行
- 所有最新项目（Blinko、New API、Vaultwarden 等）

### 3. **健康检查字段**
| 版本 | 字段名 | 兼容性 |
|------|--------|--------|
| **v1.4.1+** | `healthcheck` | ✅ 与 Docker Compose 100% 兼容 |
| **pre-v1.4.1** | `health_check` | ❌ 已废弃 |

**v1.4.1 更新内容（2025-11-19）：**
- ✅ 新增 `services.[].healthcheck` 字段
- ❌ 废弃 `services.[].health_check` 字段
- ✅ `application.health_check` 新增 `timeout` 字段

### 4. **路由配置**
| 方式 | 状态 | 推荐度 |
|------|------|--------|
| `upstreams` | ✅ 推荐 | ⭐⭐⭐⭐⭐ |
| `routes` | ✅ 支持 | ⭐⭐⭐ |

**upstreams 示例：**
```yaml
application:
  upstreams:
    - location: /
      backend: http://myapp:8080/
    - location: /api
      backend: http://backend:3000/
```

---

## 📊 实际项目对比

### 旧格式项目示例
```yaml
# Yuque Sync (旧格式)
lzc-sdk-version: "0.1"  # 旧
name: Yuque Sync
package: cloud.lazycat.app.yuque-sync
version: 1.0.0
min_os_version: 1.3.8  # ✅ 已有
services:
  yuque-sync:
    healthcheck:  # ✅ v1.4.1 格式
      test: ["CMD", "pgrep", "yuque-sync"]
```

### 新格式项目示例
```yaml
# Blinko (最新格式)
name: Blinko  # ✅ 无 lzc-sdk-version
package: lazycat.community.app.blinko
min_os_version: 1.3.8  # ✅ 必须
version: 1.7.1
application:
  upstreams:  # ✅ 推荐
    - location: /
      backend: http://blinko-website:1111/
services:
  blinko-website:
    health_check:  # ⚠️ 旧格式，需迁移
      test: ["CMD-SHELL", "curl -f http://blinko-website:1111/"]
```

---

## 🚀 迁移指南

### 从旧格式迁移到新格式

```yaml
# ❌ 旧格式 (pre-v1.4.1)
lzc-sdk-version: "0.1"
name: MyApp
package: cloud.lazycat.app.myapp
version: 1.0.0
# 缺少 min_os_version

application:
  routes:  # 旧路由方式
    - /=http://myapp:8080/

services:
  postgres:
    health_check:  # 旧健康检查
      test: ["CMD", "pg_isready"]
      start_period: 30  # 无单位
```

```yaml
# ✅ 新格式 (v1.4.1+)
name: MyApp
package: cloud.lazycat.app.myapp
version: 1.0.0
min_os_version: 1.3.8  # ✅ 添加

application:
  upstreams:  # ✅ 推荐
    - location: /
      backend: http://myapp:8080/

services:
  postgres:
    healthcheck:  # ✅ 改为 healthcheck
      test: ["CMD", "pg_isready"]
      start_period: 30s  # ✅ 添加单位
```

**迁移步骤：**
1. 删除 `lzc-sdk-version` 行
2. 添加 `min_os_version: 1.3.8`
3. 将 `routes` 改为 `upstreams`（可选，但推荐）
4. 将 `health_check` 改为 `healthcheck`
5. 为所有时间字段添加单位（如 `30` → `30s`）

---

## 📚 参考来源

1. **您的博客文章**：`/home/czyt/Documents/blog/content/post/simple-guide-for-developing-for-lazycat-nas.md`
   - 第 221 行：`min_os_version: 1.3.8`
   - 第 435 行：`health_check` 示例

2. **官方 Changelog**：`/home/czyt/Desktop/lzc-developer-doc-master/docs/changelogs/v1.4.1.md`
   - v1.4.1 于 2025-11-19 发布
   - 新增 `healthcheck` 字段
   - 废弃 `health_check` 字段

3. **实际项目**：`~/code/lazycat/` 目录
   - Blinko、New API、Vaultwarden 等最新项目

---

## ✅ 快速检查清单

创建新应用时，确保：
- [ ] 没有 `lzc-sdk-version`
- [ ] 有 `min_os_version: 1.3.8`
- [ ] 服务使用 `healthcheck`（无下划线）
- [ ] 应用使用 `health_check.test_url`（带下划线）
- [ ] 推荐使用 `upstreams` 代替 `routes`
- [ ] 内部服务参数使用 `{{.INTERNAL.xxx}}`
- [ ] 用户配置参数使用 `{{.U.xxx}}`
- [ ] ⚠️ **`command` 字段必须是字符串**（不能是数组）
- [ ] `sleep` 命令使用纯数字（如 `sleep 15`，不要 `sleep 15s`）

---

## 🎓 模板函数参考

LazyCat 支持多种模板函数，用于生成动态值和安全配置。

### 1. 内部服务模板 ({{.INTERNAL.xxx}})

**用途：** 自动生成内部服务的敏感配置（密码、密钥等）

```yaml
services:
  postgres:
    environment:
      - POSTGRES_PASSWORD={{.INTERNAL.db_password}}  # 自动生成
  redis:
    command: redis-server --requirepass {{.INTERNAL.redis_password}}  # 自动生成
```

**特性：**
- ✅ 自动生成随机密码
- ✅ 同一应用内多次调用结果相同
- ✅ 不同应用结果不同
- ✅ 应用卸载后重新安装会变化

### 2. 用户配置模板 ({{.U.xxx}})

**用途：** 引用用户在安装时配置的参数

```yaml
services:
  app:
    environment:
      - JWT_SECRET_KEY={{.U.jwt_secret_key}}  # 用户填写
      - ENCRYPTION_KEY={{.U.encryption_key}}  # 用户填写
```

**特性：**
- ✅ 用户在安装向导中填写
- ✅ 支持验证规则（regex, min/max）
- ✅ 可设置默认值

### 3. 运行时环境变量 (${LAZYCAT_*})

**用途：** LazyCat 系统注入的运行时变量

```yaml
services:
  app:
    environment:
      - APP_ID=${LAZYCAT_APP_ID}           # 应用唯一ID
      - APP_NAME=${LAZYCAT_APP_NAME}       # 应用名称
      - PUBLIC_URL=${LAZYCAT_PUBLIC_URL}   # 公共访问URL
      - SUBDOMAIN=${LAZYCAT_SUBDOMAIN}     # 子域名
      - BOX_DOMAIN=${LAZYCAT_BOX_DOMAIN}   # 盒子域名
```

**可用变量：**
- `${LAZYCAT_APP_ID}` - 应用唯一标识
- `${LAZYCAT_APP_NAME}` - 应用名称
- `${LAZYCAT_SUBDOMAIN}` - 子域名
- `${LAZYCAT_BOX_DOMAIN}` - 盒子域名
- `${LAZYCAT_PUBLIC_URL}` - 完整访问URL

### 4. 稳定秘密生成 (stable_secret)

**⚠️ 重要：懒猫微服内置模板函数**

```yaml
services:
  app:
    environment:
      # 使用 stable_secret 生成稳定的密钥
      - API_KEY={{ stable_secret "api_key_seed"}}  # 基于种子生成
      - ENCRYPTION_KEY={{ stable_secret "encryption_seed"}}
```

**stable_secret 特性：**
- **参数：** 任意字符串作为种子（seed）
- **稳定性保证：**
  - ✅ 同样种子 → 不同应用 → 结果不同
  - ✅ 同样种子 → 同一应用 → 不同微服 → 结果不同
  - ✅ 同样种子 → 同一应用 → 同一微服 → 结果相同（多次调用）
  - ✅ 微服恢复出厂设置 → 结果改变

**使用场景：**
- 生成稳定的 API 密钥
- 生成加密密钥
- 生成需要在多实例间保持一致的密钥

**示例：**
```yaml
# 生成稳定的 JWT 密钥
- JWT_SECRET={{ stable_secret "jwt_secret_v1"}}

# 生成稳定的加密密钥
- ENCRYPTION_KEY={{ stable_secret "encryption_v1"}}

# 生成稳定的数据库密码（同一应用内一致）
- DB_PASSWORD={{ stable_secret "db_password_v1"}}
```

**优势：**
- ✅ **无需用户输入** - 使用模板函数自动生成
- ✅ **稳定性** - 同一应用多次部署结果一致
- ✅ **安全性** - 不同应用结果不同
- ✅ **可预测** - 基于种子，便于调试

### 5. 模板函数对比

| 函数 | 用途 | 稳定性 | 适用场景 |
|------|------|--------|----------|
| `{{.INTERNAL.xxx}}` | 内部服务密码 | 自动管理 | 数据库、Redis |
| `{{.U.xxx}}` | 用户配置 | 用户输入 | JWT密钥、管理员密码 |
| `${LAZYCAT_*}` | 运行时变量 | 系统注入 | 应用ID、URL |
| `{{ stable_secret "seed"}}` | 稳定密钥 | 种子决定 | API密钥、加密密钥 |

**💡 最佳实践：**
- 内部服务密码 → 使用 `{{.INTERNAL.xxx}}`
- 需要稳定的密钥 → 使用 `{{ stable_secret "seed"}}`
- 用户必须配置的 → 使用 `{{.U.xxx}}`
- 运行时信息 → 使用 `${LAZYCAT_*}`


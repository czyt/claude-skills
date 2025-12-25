# 健康检查配置参考 (LazyCat v1.4.1+)

## 📋 核心规则

### 1. 字段名称（v1.4.1 重要更新！）

| 使用场景 | Docker Compose | LazyCat v1.4.1+ | 兼容性 |
|---------|----------------|-----------------|--------|
| **服务容器** | `healthcheck` | `healthcheck` | ✅ 100% 兼容 |
| **应用容器** | N/A | `health_check` | ✅ 仅用于 test_url |

**⚠️ v1.4.1 关键变更（2025-11-19）：**
- ✅ **新增**：`services.*.healthcheck` - 与 Docker Compose 100% 兼容
- ❌ **废弃**：`services.*.health_check` - 旧格式，需迁移到 `healthcheck`
- ✅ **新增**：`application.health_check.timeout` - 现在支持

**来源：**
- 您的博客文章 `/home/czyt/Documents/blog/content/post/simple-guide-for-developing-for-lazycat-nas.md`
- v1.4.1 更新日志
- 实际项目（部分已升级，部分仍用旧格式）

---

## 2. 服务容器健康检查 (`services.*.healthcheck`)

v1.4.1+ 使用 `healthcheck`（无下划线），与 Docker Compose 完全一致

### 支持的字段

| 字段 | 类型 | 必需 | 说明 | 示例 |
|------|------|------|------|------|
| `test` | `[]string` | ✅ | 检测命令 | `["CMD", "curl", "-f", "http://localhost"]` |
| `timeout` | `string` | ❌ | 单次检测超时 | `"10s"` |
| `interval` | `string` | ❌ | 检测间隔 | `"30s"` |
| `retries` | `int` | ❌ | 失败重试次数 | `3` |
| `start_period` | `string` | ❌ | 启动等待时间 | `"30s"` 或 `"90s"` |
| `start_interval` | `string` | ❌ | 启动期间检测间隔 | `"5s"` |
| `disable` | `bool` | ❌ | 禁用检测 | `false` |

### 常用数据库检测命令（v1.4.1+ 正确格式）

**💡 推荐写法：多行数组格式（更清晰，更易读）**

```yaml
# PostgreSQL (推荐格式)
postgres:
  healthcheck:  # ✅ 无下划线，与 Docker Compose 一致
    test:
      - CMD-SHELL
      - pg_isready -U postgres
    interval: 30s
    timeout: 10s
    retries: 3
    start_period: 30s

# Redis (推荐格式)
redis:
  healthcheck:  # ✅ 无下划线
    test:
      - CMD
      - redis-cli
      - ping
    interval: 30s
    start_period: 10s

# MySQL (推荐格式)
mysql:
  healthcheck:
    test:
      - CMD
      - mysqladmin
      - ping
      - -h
      - localhost
      - -u
      - root
      - -p${MYSQL_ROOT_PASSWORD}
    interval: 30s
    timeout: 10s
    start_period: 60s

# MariaDB (推荐格式)
mariadb:
  healthcheck:
    test:
      - CMD-SHELL
      - healthcheck.sh --connect --innodb_initialized
    start_period: 30s

# MongoDB (推荐格式)
mongo:
  healthcheck:
    test:
      - CMD
      - mongosh
      - --eval
      - db.adminCommand('ping')
    interval: 30s
    start_period: 10s

# Custom HTTP check (推荐格式)
app:
  healthcheck:
    test:
      - CMD-SHELL
      - curl -f http://localhost:8080/health || exit 1
    interval: 30s
    timeout: 5s
    start_period: 30s
```

**备选写法：JSON 数组格式（更紧凑）**

```yaml
# PostgreSQL (紧凑格式)
postgres:
  healthcheck:
    test: ["CMD-SHELL", "pg_isready -U postgres"]
    interval: 30s
    timeout: 10s
    retries: 3
    start_period: 30s

# Redis (紧凑格式)
redis:
  healthcheck:
    test: ["CMD", "redis-cli", "ping"]
    interval: 30s
    start_period: 10s

# App (紧凑格式)
app:
  healthcheck:
    test: ["CMD-SHELL", "curl -f http://localhost:8080/health || exit 1"]
    interval: 30s
    timeout: 5s
    start_period: 30s
```

**⚠️ 重要说明：**
- 您的博客文章写于 2025-02-07，当时使用的是 `health_check`（旧格式）
- v1.4.1（2025-11-19）后应使用 `healthcheck`（新格式）
- 旧格式仍可工作，但建议迁移到新格式

---

## 3. 两种写法对比与推荐

### 推荐：多行数组格式（⭐⭐⭐⭐⭐）

```yaml
services:
  postgres:
    healthcheck:
      test:
        - CMD-SHELL
        - pg_isready -U postgres
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 30s
```

**优势：**
- ✅ **清晰易读** - 每行一个参数，一目了然
- ✅ **易于编辑** - 添加/删除参数更方便
- ✅ **适合复杂命令** - 长命令不会超出屏幕
- ✅ **团队协作** - 代码审查更容易
- ✅ **实际项目使用** - Blinko、RSS Translator 等项目都用这种格式

**适用场景：**
- 所有情况（强烈推荐）

---

### 备选：JSON 数组格式（⭐⭐⭐）

```yaml
services:
  postgres:
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 30s
```

**优势：**
- ✅ **紧凑** - 占用行数少
- ✅ **适合简单命令** - 一行搞定

**劣势：**
- ❌ **不易读** - 命令长时难以阅读
- ❌ **不易编辑** - 修改参数需要拆分数组
- ❌ **不适合复杂命令** - 参数多时会很长

**适用场景：**
- 非常简单的命令（如 `["CMD", "redis-cli", "ping"]`）
- 配置文件需要极度紧凑

---

### 实际项目对比

**Blinko 项目（多行格式）：**
```yaml
health_check:
  test:
    - CMD-SHELL
    - curl -f http://blinko-website:1111/
  start_period: 60s
```

**Yuque Sync 项目（JSON 格式）：**
```yaml
healthcheck:
  test: ["CMD", "pgrep", "yuque-sync"]
  start_period: 30s
```

**RSS Translator 项目（多行格式）：**
```yaml
healthcheck:
  test:
    - CMD-SHELL
    - redis-cli ping
  interval: 30s
```

---

## 4. 最终推荐

### ✅ 技能默认使用：多行数组格式

**原因：**
1. **可读性优先** - 配置文件应该易于理解和维护
2. **实际项目验证** - 多个真实项目都使用这种格式
3. **适合所有场景** - 无论是简单还是复杂命令都清晰
4. **团队友好** - 代码审查和协作更高效

### 使用建议

**对于新项目：**
```yaml
# ✅ 推荐
healthcheck:
  test:
    - CMD-SHELL
    - curl -f http://localhost:8080/health || exit 1
  interval: 30s
  timeout: 10s
  retries: 3
  start_period: 30s
```

**对于简单命令（可选）：**
```yaml
# ✅ 可接受（但多行更好）
healthcheck:
  test: ["CMD", "redis-cli", "ping"]
  interval: 30s
```

**技能转换时：**
- Docker Compose 的多行格式 → 直接复制（无需转换）
- Docker Compose 的 JSON 格式 → 转换为多行格式

## 3. 应用容器健康检查 (`application.health_check`)

### 支持的字段

| 字段 | 类型 | 必需 | 说明 | 示例 |
|------|------|------|------|------|
| `test_url` | `string` | ❌ | HTTP检测URL | `"http://localhost:8080/health"` |
| `timeout` | `string` | ❌ | 检测超时 | `"10s"` |
| `start_period` | `string` | ❌ | 启动等待时间 | `"30s"` |
| `disable` | `bool` | ❌ | 禁用检测 | `false` |

### 示例

```yaml
application:
  subdomain: myapp
  upstreams:  # ✅ 推荐使用 upstreams
    - location: /
      backend: http://myapp:8080/
  health_check:
    test_url: http://localhost:8080/health
    timeout: 10s
    start_period: 30s
```

---

## 4. Docker Compose vs LazyCat v1.4.1+ 对比

### Docker Compose 格式
```yaml
services:
  postgres:
    image: postgres:15
    healthcheck:  # 无下划线
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 30s
```

### LazyCat v1.4.1+ 格式（完全相同！）
```yaml
services:
  postgres:
    image: postgres:15
    healthcheck:  # ✅ 无下划线，完全一致
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 30s
```

**✅ v1.4.1 优势：** Docker Compose 和 LazyCat 的健康检查格式**完全相同**，无需转换！

---

## 5. 智能转换规则

### 内部服务识别
```python
# 内部服务 = 有 healthcheck + 无外部端口
if 'healthcheck' in service_config and 'ports' not in service_config:
    return 'INTERNAL'  # 自动生成配置
```

### 健康检查保持规则
```python
# Docker Compose 的 healthcheck 直接复制到 LazyCat
if 'healthcheck' in docker_compose_service:
    lazy_cat_service['healthcheck'] = docker_compose_service['healthcheck']
    # 字段完全保持不变！
```

---

## 6. 实际项目示例

### Yuque Sync (真实项目)
```yaml
services:
  yuque-sync:
    image: heizicao/yuque-sync:latest
    healthcheck:
      test: ["CMD", "pgrep", "yuque-sync"]
      start_period: 30s
      timeout: 10s
      interval: 60s
      retries: 3
```

### RSS Translator (真实项目)
```yaml
services:
  rssbox_redis:
    image: registry.lazycat.cloud/edward/library/redis:4c6b9d7b7a334a2e
    healthcheck:
      test:
        - CMD-SHELL
        - redis-cli ping
      interval: 30s
```

### Odoo (真实项目)
```yaml
services:
  odoo-db:
    image: registry.lazycat.cloud/czyt/library/postgres:4c94230970ec8303
    health_check:  # ⚠️ 注意：这里用了旧格式
      test:
        - CMD-SHELL
        - pg_isready -U postgres
      start_period: 30s
```

**⚠️ 注意：** Odoo 项目使用了旧格式 `health_check`，但官方文档已说明这是**已废弃**的格式。

---

## 7. 常见错误

### ❌ 错误 1: 使用 `health_check` 用于服务
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

### ❌ 错误 2: 缺少 healthcheck 导致无法识别为内部服务
```yaml
# ❌ 错误 - 会被识别为 EXTERNAL
services:
  postgres:
    image: postgres:15
    # 没有 healthcheck

# ✅ 正确 - 会被识别为 INTERNAL
services:
  postgres:
    image: postgres:15
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
```

### ❌ 错误 3: 混淆应用和服务的字段名
```yaml
# ❌ 错误
application:
  healthcheck:  # 应该用 health_check
    test_url: http://localhost:8080

# ✅ 正确
application:
  health_check:  # 带下划线
    test_url: http://localhost:8080
```

---

## 8. 快速参考表

| 场景 | 字段名 | 关键字段 | 示例 |
|------|--------|----------|------|
| PostgreSQL | `healthcheck` | `test: ["CMD-SHELL", "pg_isready -U postgres"]` | ✅ |
| Redis | `healthcheck` | `test: ["CMD", "redis-cli", "ping"]` | ✅ |
| MySQL | `healthcheck` | `test: ["CMD", "mysqladmin", "ping"]` | ✅ |
| HTTP App | `healthcheck` | `test: ["CMD-SHELL", "curl ..."]` | ✅ |
| App Container | `health_check` | `test_url: http://localhost:8080` | ✅ |

---

## 9. 官方文档参考

**来源：** `/home/czyt/Desktop/lzc-developer-doc-master/docs/spec/manifest.md`

**关键引用：**
> `healthcheck` | `*HealthCheckConfig` | 容器的健康检测策略, 老版本`health_check`已被废弃

**HealthCheckConfig 结构：**
```go
type HealthCheckConfig struct {
    Test        []string      `yaml:"test"`
    Timeout     string        `yaml:"timeout"`
    Interval    string        `yaml:"interval"`
    Retries     int           `yaml:"retries"`
    StartPeriod string        `yaml:"start_period"`
    StartInterval string      `yaml:"start_interval"`
    Disable     bool          `yaml:"disable"`
}
```

---

## 10. 总结

### ✅ 记住这 3 点

1. **服务用 `healthcheck`**（无下划线）
2. **应用用 `health_check`**（带下划线，仅用于 `test_url`）
3. **转换时保持字段不变**，只需确保字段名正确

### 转换流程
```bash
Docker Compose → LazyCat
├─ 服务健康检查：healthcheck → healthcheck (保持不变)
├─ 应用健康检查：不支持 → health_check.test_url
└─ 字段内容：完全复制，无需修改
```

---

**最后更新：** 2025-12-25
**参考文档：** LazyCat 官方文档 v1.4.1

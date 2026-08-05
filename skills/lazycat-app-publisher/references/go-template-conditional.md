# Go Template 渲染参考

LazyCat 在安装或重新配置实例时，使用 Go `text/template` 渲染 `/lzcapp/pkg/manifest.yml`，结果写入 `/lzcapp/run/manifest.yml`。模板只处理部署阶段才能确定的值；dev/release 结构差异应放到 `lzc-build.dev.yml` 或 build 预处理，不要塞进部署模板。

## 目录

- [渲染上下文](#渲染上下文)
- [取值、管道与变量](#取值管道与变量)
- [条件语句](#条件语句)
- [`with` 与 `range`](#with-与-range)
- [内置函数](#内置函数)
- [空值与类型陷阱](#空值与类型陷阱)
- [YAML 安全规则](#yaml-安全规则)
- [调试与验证](#调试与验证)

## 渲染上下文

当前官方只保证两类模板参数：

| 上下文 | 来源 | 示例 |
|--------|------|------|
| `.U` / `.UserParams` | `lzc-deploy-params.yml` | `{{ .U.target }}` |
| `.S` / `.SysParams` | 系统部署上下文 | `{{ .S.AppDomain }}` |

`.S` 的稳定字段：

| 字段 | 语义 |
|------|------|
| `.S.BoxName` | 微服名称 |
| `.S.BoxDomain` | 微服域名 |
| `.S.OSVersion` | 系统版本；测试版可能为 `v99.99.99-xxx` |
| `.S.AppDomain` | 当前应用实例域名 |
| `.S.IsMultiInstance` | 是否为多实例应用 |
| `.S.DeployUID` | 部署用户 UID；单实例下无实际意义 |
| `.S.DeployID` | 当前应用实例唯一 ID |

不要使用 `.E` / `.PkgEnvs` 或 `.INTERNAL`。前两者已不再提供，后者也不是当前官方稳定上下文。内部服务密码应优先使用 `stable_secret`，或显式声明部署参数。

## 取值、管道与变量

模板动作写在 `{{` 与 `}}` 之间。模板注释使用 `{{/* ... */}}`，不会进入渲染结果：

```gotemplate
{{/* 只在源码中可见 */}}
{{ .U.target }}
```

### 字段访问

简单字段名优先使用点语法，`index` 同样合法；包含 `.`、`-` 等特殊字符时必须使用 `index`：

```yaml
# lzc-deploy-params.yml
params:
  - id: target
    type: string
    name: Target
    description: Target host
  - id: listen.port
    type: string
    name: Listen port
    description: Listen port
    optional: true
    default_value: "8080"
```

```yaml
# lzc-manifest.yml
application:
  upstreams:
    - location: /
      backend_launch_command: /app/server --target={{ .U.target }} --port={{ index .U "listen.port" }}
```

不要把 key 中的 `.` 当作层级分隔符。`{{ .U.listen.port }}` 表示访问 `listen` 的子字段 `port`，不能读取名为 `listen.port` 的部署参数。

### 管道

管道 `|` 会把左侧结果作为右侧函数的最后一个参数。以下两种写法等价：

```gotemplate
{{ quote (default "guest" .U.username) }}
{{ .U.username | default "guest" | quote }}
```

Manifest 中的用户字符串通常应经过 `quote`，避免 `:`、`#`、布尔样式文本或前后空格改变 YAML 语义：

```yaml
services:
  app:
    environment:
      - {{ printf "TARGET=%s" .U.target | quote }}
```

### 变量与根上下文

使用 `:=` 声明变量，使用 `$` 保留根上下文。进入 `with` 或 `range` 后，点 `.` 会改变，此时用 `$.U` 回到根上下文：

```gotemplate
{{ $target := .U.target }}
{{ $target | trim | quote }}
{{ $.S.AppDomain }}
```

变量作用域持续到其所在控制结构的 `end`；不要假设变量能跨越独立模板文件。

## 条件语句

Go Template 条件块可用于在参数有值时渲染字段：

```yaml
services:
  app:
    environment:
      - MODE={{ if .U.debug }}debug{{ else }}production{{ end }}
```

需要整段 YAML 条件渲染时，控制行必须独立放置，并让两个分支都生成完整、合法且缩进一致的 YAML：

```yaml
application:
  subdomain: demo
{{ if .U.public_access }}
  public_path:
    - /
{{ else }}
  public_path: []
{{ end }}
```

常见条件写法：

```gotemplate
{{ if .U.enabled }}...{{ end }}
{{ if not .U.enabled }}...{{ end }}
{{ if eq .U.mode "advanced" }}...{{ end }}
{{ if and .U.enabled (eq .U.mode "advanced") }}...{{ end }}
{{ with .U.optional_value }}{{ . }}{{ else }}fallback{{ end }}
```

比较函数：

| 函数 | 含义 |
|------|------|
| `eq a b` / `ne a b` | 等于 / 不等于 |
| `lt a b` / `le a b` | 小于 / 小于等于 |
| `gt a b` / `ge a b` | 大于 / 大于等于 |
| `and a b` / `or a b` / `not a` | 与 / 或 / 非 |

比较值应具有兼容类型。部署参数只使用官方支持的 `bool`、`lzc_uid`、`string`、`secret`；不要虚构 `number`、`enum` 或校验字段。字符串形式的端口不能直接与数字比较，应先把约束写入参数描述，或使用 Sprig 的显式类型转换函数后再比较。

## `with` 与 `range`

### `with`

`with` 在值非空时执行并把 `.` 改为该值，适合处理可选参数。需要原根上下文时使用 `$`：

```yaml
services:
  app:
    environment:
{{ with .U.proxy_url }}
      - {{ printf "PROXY_URL=%s" . | quote }}
      - {{ printf "APP_DOMAIN=%s" $.S.AppDomain | quote }}
{{ else }}
      - PROXY_DISABLED=true
{{ end }}
```

### `range`

`range` 可遍历数组、切片、map 或 channel。部署参数本身没有数组类型；若用户输入逗号分隔字符串，可先用 Sprig 的 `splitList` 生成列表：

```yaml
services:
  app:
    command:
      - /app/server
{{ range $host := splitList "," .U.allowed_hosts }}
      - {{ printf "--allow=%s" ($host | trim) | quote }}
{{ else }}
      - --deny-all
{{ end }}
```

常用形式：

```gotemplate
{{ range $index, $value := $items }}...{{ end }}
{{ range $key, $value := $map }}...{{ end }}
{{ range $items }}...{{ else }}列表为空时的内容{{ end }}
```

## 内置函数

官方提供 Sprig 函数（排除 `env` / `expandenv`）和 `stable_secret`：

```yaml
services:
  mysql:
    environment:
      - MYSQL_ROOT_PASSWORD={{ stable_secret "root_password" }}
      - MYSQL_PASSWORD={{ stable_secret "admin_password" | substr 0 16 }}
```

同一个 seed 在同一台微服、同一个应用内保持稳定；换应用或换微服结果不同。不要把 secret 明文写进 Manifest，也不要用 `env`/`expandenv` 读取构建机环境。

常用 Sprig 函数包括：

| 类别 | 函数示例 | 用途 |
|------|----------|------|
| 默认值 | `default`、`coalesce`、`empty`、`ternary` | 处理空值或选择结果 |
| 字符串 | `trim`、`lower`、`upper`、`replace`、`contains` | 清理和判断文本 |
| 引号 | `quote`、`squote` | 生成带引号字符串 |
| 列表 | `list`、`splitList`、`join`、`first`、`last` | 构造或处理列表 |
| 转换 | `int`、`toString`、`toJson`、`b64enc` | 显式转换或编码 |
| 密码学 | `sha256sum` | 计算摘要；生成密码仍用 `stable_secret` |

函数细节以 Sprig 文档为准。平台明确排除了 `env` 和 `expandenv`，不要依赖进程环境。也不要假设 Helm 专有函数一定存在；LazyCat 提供的是 Go `text/template` + Sprig，不是 Helm 模板引擎。

## 空值与类型陷阱

Go Template 的 `if`、`with` 和 Sprig 的多种默认值函数会把以下值视为空：

- `false`
- 数值 `0`
- 空字符串
- 长度为 0 的数组、切片、map
- `nil` 或无效值

这会带来两个常见陷阱：

```gotemplate
{{/* false 会被 default 当作空值，结果变成 true；通常不是想要的行为 */}}
{{ .U.enabled | default true }}

{{/* bool 参数应直接分支 */}}
{{ if .U.enabled }}enabled{{ else }}disabled{{ end }}
```

只访问 `lzc-deploy-params.yml` 中已经声明的参数，不要依赖缺失 key 的具体渲染文本；平台或 Go 版本改变缺失 key 选项时，结果可能不同。需要默认值时在 `default_value` 中声明，或在模板内显式使用 `default`。

## 空白控制与字面量

`{{-` 会裁掉动作左侧空白，`-}}` 会裁掉右侧空白：

```gotemplate
value={{- "compact" -}}
```

在 YAML 中慎用空白裁剪，因为它可能把相邻行粘在一起并破坏缩进。控制整段 YAML 时，优先让 `if` / `range` / `end` 各占一行，接受渲染后的空行。

如需输出字面量 `{{`，让模板自己输出字符串：

```gotemplate
{{ "{{" }} untouched {{ "}}" }}
```

## YAML 安全规则

1. 模板展开后的文本必须仍是合法 YAML；字符串值存在 `:`、`#`、换行等风险时要加引号。
2. 不要让 `if` 只包住半个键值对、列表项前缀或不完整缩进。
3. 不要用普通 YAML 解析器直接 round-trip 含独立 `{{ if }}` / `{{ else }}` / `{{ end }}` 的原始 Manifest；编辑工具必须先保护控制行并原样恢复。
4. 可以用本地 Go `text/template` + Sprig 夹具覆盖不同输入，但它只能做语法和结构预检，不能代替平台真实渲染。
5. 每个条件分支、空列表分支和可选参数缺省分支都必须单独验证为合法 YAML。
6. 不要用部署期模板裁剪 dev/release 结构；这属于 build 阶段。

## 调试与验证

部署后查看最终渲染结果：

```bash
cat /lzcapp/run/manifest.yml
```

排查时可临时输出完整上下文：

```yaml
xx-debug: {{ . }}
```

调试字段不得进入正式发布包。推荐按以下顺序验证：

1. 用本地模板夹具分别渲染必填、可选、空值和各条件分支。
2. 对每份渲染结果执行 YAML 解析。
3. 使用 `lzc-cli project build` 或 `lzc-cli project release` 检查打包阶段。
4. 在真实微服安装或重新配置实例，再检查 `/lzcapp/run/manifest.yml`。

仅执行构建不足以证明部署期参数渲染正确；最终至少执行一次真实部署，并覆盖所有关键条件分支。

官方来源：`docs/advanced-manifest-render.md`（lzc-developer-doc，2026-08-05 同步）。控制语法遵循 Go `text/template`，通用函数遵循 Sprig；平台能力和排除项以 LazyCat 官方文档为准。

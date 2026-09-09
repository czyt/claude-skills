# 客户端平台支持

`unsupported_platforms` 声明应用不支持的**客户端平台**，不是 Docker 镜像 CPU 架构（amd64/arm64）筛选。

| LPK 格式 | 字段位置 |
|----------|----------|
| v1 | `lzc-manifest.yml` 顶层，与 `application` 同级 |
| v2 | `package.yml` 顶层，与 `package`、`version` 同级 |

```yaml
unsupported_platforms:
  - ios
```

此例会在 iOS / iPad 客户端点击图标时提示“您的应用不支持当前平台”。已列出的参数为 `ios`（含 iPad）、`android`、`linux`、`windows`、`macos`、`tvos`（懒猫智慧屏，系统 1.0.18+）。客户端覆盖范围不等于该字段可用参数；上游虽提到鸿蒙客户端，但未列出鸿蒙专用枚举，不能自行编造。

来源：`lzc-developer-doc/docs/advanced-platform.md`，2026-09-08。

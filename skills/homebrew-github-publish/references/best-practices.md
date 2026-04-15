# 最佳实践与常见问题

## ✅ 推荐做法

### 1. 使用 livecheck

为所有 Cask 和 Formula 配置 livecheck:

```ruby
livecheck do
  url :url
  strategy :github_latest
end
```

### 2. 多架构支持

同时支持 arm64 和 x64:

```ruby
on_arm do
  sha256 "PLACEHOLDER"
  url "https://example.com/app-arm64.dmg"
end

on_intel do
  sha256 "PLACEHOLDER"
  url "https://example.com/app-x64.dmg"
end
```

### 3. 定时自动更新

设置 schedule 自动检查:

```yaml
on:
  schedule:
    - cron: "0 */12 * * *"  # 每12小时
```

### 4. 手动触发选项

提供 workflow_dispatch:

```yaml
on:
  workflow_dispatch:
    inputs:
      version:
        description: "Version (optional)"
        required: false
```

### 5. sha256 占位符

使用 PLACEHOLDER 或在 workflow 中自动计算:

```ruby
sha256 "PLACEHOLDER"  # workflow 自动更新
```

### 6. 文件验证

检查下载文件大小:

```bash
if [ $(stat -c%s /tmp/file) -lt 100000 ]; then
  echo "File too small"
  exit 1
fi
```

### 7. 本地验证

提交前运行 audit:

```bash
brew audit --cask {name}
brew style --cask {name}
```

## ❌ 避免的做法

### 1. 硬编码版本

❌ 不使用 livecheck:
```ruby
cask "myapp" do
  version "1.0.0"
  # 没有 livecheck
end
```

✅ 配置 livecheck:
```ruby
cask "myapp" do
  version "1.0.0"

  livecheck do
    url :url
    strategy :github_latest
  end
end
```

### 2. 单架构支持

❌ 只支持 Intel:
```ruby
cask "myapp" do
  sha256 "..."
  url "https://example.com/app.dmg"  # 只有 x64
end
```

✅ 多架构:
```ruby
cask "myapp" do
  on_arm do
    sha256 "PLACEHOLDER"
    url "https://example.com/app-arm64.dmg"
  end
  on_intel do
    sha256 "PLACEHOLDER"
    url "https://example.com/app-x64.dmg"
  end
end
```

### 3. 跳过验证

❌ 不运行 audit:
```bash
# 直接提交，没有验证
git push
```

✅ 先验证:
```bash
brew audit --cask {name}
brew style --cask {name}
git push
```

### 4. 忽略下载错误

❌ 不检查 curl 结果:
```bash
curl -o /tmp/file.dmg "${URL}"
sha256sum /tmp/file.dmg
```

✅ 使用 -f 并验证:
```bash
curl -f -L -o /tmp/file.dmg "${URL}" || exit 1
if [ ! -s /tmp/file.dmg ]; then
  echo "Download failed"
  exit 1
fi
sha256sum /tmp/file.dmg | awk '{print $1}'
```

### 5. 手动计算 sha256

❌ 手动计算并硬编码:
```ruby
sha256 "abc123..."  # 手动计算
```

✅ workflow 自动计算:
```ruby
sha256 "PLACEHOLDER"  # workflow 更新
```

## 常见问题

### 1. curl 失败

**原因**: URL 格式错误或文件不存在
**解决**: 检查 URL 格式和版本号

```bash
# 检查 URL
curl -I "https://example.com/app-v1.0.0.dmg"
```

### 2. sha256 不匹配

**原因**: 文件更新但 sha256 未更新
**解决**: 使用 workflow 自动计算

### 3. livecheck 失败

**原因**: URL strategy 不匹配
**解决**: 检查 livecheck 配置

```bash
brew livecheck --debug {name}
```

### 4. 版本号格式问题

**原因**: tag_name 包含前缀
**解决**: 使用 sed 处理

```bash
VERSION=$(echo "$TAG_NAME" | sed 's/^v//' | sed 's/^release-//')
```

### 5. API 限流

**原因**: 无认证请求过多
**解决**: 使用 GITHUB_TOKEN

```yaml
env:
  GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

## 参考链接

- [Homebrew Cask Cookbook](https://docs.brew.sh/Cask-Cookbook)
- [Homebrew Formula Cookbook](https://docs.brew.sh/Formula-Cookbook)
- [GitHub Actions 最佳实践](https://docs.github.com/en/actions/learn-github-actions/best-practices-for-github-actions)
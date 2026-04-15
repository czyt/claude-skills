# 版本检测策略

## GitHub Releases API

### 获取最新版本

```bash
curl -s https://api.github.com/repos/{owner}/{repo}/releases/latest | jq -r '.tag_name' | sed 's/^v//'
```

### 处理版本号格式

常见版本格式:
- `v1.2.3` → 移除 `v` 前缀
- `1.2.3` → 直接使用
- `release-1.2.3` → 移除 `release-` 前缀

```bash
# 移除 v 前缀
sed 's/^v//'

# 移除 release- 前缀
sed 's/^release-//'

# 只提取数字版本
grep -oE '[0-9]+\.[0-9]+\.[0-9]+'
```

## livecheck 配置

### GitHub Latest

```ruby
livecheck do
  url :url
  strategy :github_latest
end
```

### GitHub Releases List

```ruby
livecheck do
  url "https://github.com/{owner}/{repo}/releases"
  regex(/v?(\d+(?:\.\d+)+)/i)
end
```

### Sparkle Appcast

```ruby
livecheck do
  url "https://example.com/appcast.xml"
  strategy :sparkle
end
```

### 自定义 Regex

```ruby
livecheck do
  url "https://example.com/downloads/"
  regex(/href=".*?v?(\d+(?:\.\d+)+)[._-]/i)
end
```

## 本地测试 livecheck

```bash
# 测试 livecheck
brew livecheck {cask_or_formula}

# 查看检测详情
brew livecheck --debug {cask_or_formula}
```

## 版本比较逻辑

### Workflow 中的比较

```bash
if [ "$CURRENT_VERSION" = "$NEW_VERSION" ]; then
  echo "No update needed"
  echo "should_update=false" >> $GITHUB_OUTPUT
else
  echo "Update available: $CURRENT_VERSION -> $NEW_VERSION"
  echo "should_update=true" >> $GITHUB_OUTPUT
fi
```

### 版本号排序

```bash
# 使用 sort -V 进行版本排序
NEW_VERSION=$(curl -s https://api.github.com/repos/{owner}/{repo}/releases | jq -r '.[].tag_name' | sed 's/^v//' | sort -V | tail -n 1)
```

## 处理预发布版本

### 跳过预发布

```bash
# 只获取正式版本（draft: false, prerelease: false）
curl -s https://api.github.com/repos/{owner}/{repo}/releases | \
  jq -r '[.[] | select(.draft == false and .prerelease == false)] | .[0].tag_name'
```

### 包含预发布

```bash
curl -s https://api.github.com/repos/{owner}/{repo}/releases/latest | jq -r '.tag_name'
```

## API 限流处理

### 使用认证

```yaml
- name: Get latest version
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
  run: |
    curl -s -H "Authorization: token $GITHUB_TOKEN" \
      https://api.github.com/repos/{owner}/{repo}/releases/latest | \
      jq -r '.tag_name'
```

### 处理 API 错误

```bash
RELEASE_INFO=$(curl -f -s "https://api.github.com/repos/{owner}/{repo}/releases/latest")
if [ $? -ne 0 ]; then
  echo "Failed to fetch release info"
  exit 1
fi
```

## 参考链接

- [GitHub Releases API](https://docs.github.com/en/rest/releases/releases)
- [Homebrew livecheck 文档](https://docs.brew.sh/Brew-Livecheck)
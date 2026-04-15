# Homebrew Cask 模板语法与结构

## 基本结构

```ruby
cask "{name}" do
  version "{version}"
  sha256 "{checksum}"  # 使用 PLACEHOLDER，workflow 自动更新

  url "{download_url}"
  name "{app_name}"
  desc "{description}"
  homepage "{homepage_url}"

  app "{app_name}.app"
end
```

## 多架构支持

### 使用 on_arm/on_intel

```ruby
cask "myapp" do
  version "1.0.0"

  on_arm do
    sha256 "PLACEHOLDER"  # workflow 自动更新
    url "https://github.com/user/repo/releases/download/v#{version}/myapp-#{version}-mac-arm64.dmg"
  end

  on_intel do
    sha256 "PLACEHOLDER"  # workflow 自动更新
    url "https://github.com/user/repo/releases/download/v#{version}/myapp-#{version}-mac-x64.dmg"
  end

  name "MyApp"
  desc "A great application"
  homepage "https://github.com/user/repo"

  app "MyApp.app"
end
```

### 使用 on_macos/on_linux

```ruby
cask "myapp" do
  version "1.0.0"

  on_macos do
    on_arm do
      sha256 "PLACEHOLDER"
      url "https://example.com/myapp-arm64.dmg"
    end
    on_intel do
      sha256 "PLACEHOLDER"
      url "https://example.com/myapp-x64.dmg"
    end
  end

  on_linux do
    # Linux 特定配置
  end
end
```

## livecheck 配置

### GitHub 最新版本检测

```ruby
livecheck do
  url :url
  strategy :github_latest
end
```

### 自定义版本检测

```ruby
livecheck do
  url "https://github.com/user/repo/releases"
  regex(/v?(\d+(?:\.\d+)+)/i)
end
```

### Atom/Appcast 检测

```ruby
livecheck do
  url "https://example.com/appcast.xml"
  strategy :sparkle
end
```

## 安装类型

### app (应用程序)

```ruby
app "MyApp.app"
```

### pkg (安装包)

```ruby
pkg "MyApp.pkg"
```

### binary (二进制)

```ruby
binary "myapp"
```

### 多个安装目标

```ruby
app "MyApp.app"
binary "myapp-cli"
pkg "Helper.pkg"
```

## 软链接/硬链接

```ruby
# 软链接到 /usr/local/bin
binary "myapp", target: "myapp"

# 硬链接
binary "myapp", target: "myapp-hard"
```

## uninstall 配置

```ruby
uninstall do
  # 删除应用
  delete: "~/Library/Application Support/MyApp"
  delete: "~/Library/Caches/MyApp"
  delete: "~/Library/Preferences/com.myapp.plist"

  # 使用 uninstaller
  script: {
    executable: "MyApp.app/Contents/Resources/uninstall.sh",
    sudo: true
  }
end
```

## zap 配置 (完全删除)

```ruby
zap do
  delete: "~/.myapp"
  delete: "~/Library/Application Support/MyApp"
  delete: "~/Library/Caches/MyApp"
  delete: "~/Library/Preferences/com.myapp.plist"
  delete: "~/Library/HTTPStorages/com.myapp"
  delete: "~/Library/Webkit/com.myapp"
end
```

## depends_on 配置

### macOS 版本依赖

```ruby
depends_on macos: ">= :big_sur"  # macOS 11+
depends_on macos: ":catalina"    # macOS 10.15
depends_on macos: ">= :mojave"   # macOS 10.14+
```

### 架构依赖

```ruby
depends_on arch: :arm64  # 仅 Apple Silicon
depends_on arch: :intel  # 仅 Intel
```

### 其他 Cask 依赖

```ruby
depends_on cask: "another-cask"
```

### Formula 依赖

```ruby
depends_on formula: "python"
```

## caveats (提示信息)

```ruby
def caveats
  <<~EOS
    MyApp has been installed!

    To complete the setup:
      1. Open MyApp from Applications
      2. Configure your settings

    For shell integration:
      echo 'export PATH="/Applications/MyApp.app/Contents/Resources:$PATH"' >> ~/.zshrc
  EOS
end
```

## preflight/postflight (安装前/后脚本)

```ruby
preflight do
  # 安装前执行
  system_command "/usr/bin/xattr", args: ["-cr", staged_path.join("MyApp.app")]
end

postflight do
  # 安装后执行
  system_command "/usr/bin/open", args: [staged_path.join("MyApp.app")]
end
```

## URL 变量

```ruby
# 使用 #{version} 变量
url "https://github.com/user/repo/releases/download/v#{version}/myapp-#{version}.dmg"

# 使用 #{arch} 变量
url "https://example.com/myapp-#{arch}.dmg"
```

## 常见问题

### 1. sha256 使用 SKIP

❌ 不要使用 `'SKIP'` 作为真实的 checksum
✅ 使用 `PLACEHOLDER` 占位符，workflow 自动计算

### 2. URL 格式

❌ `url "https://github.com/.../v${version}/..."`
✅ `url "https://github.com/.../v#{version}/..."` 使用 Ruby 插值

### 3. livecheck 不工作

确保 URL 正确且 strategy 匹配:
- GitHub Releases: `strategy :github_latest`
- Sparkle Appcast: `strategy :sparkle`

## Workflow 中的 sed 命令

更新 sha256:

```bash
# 更新 arm64 sha256
sed -i '/on_arm do/,/end/{s/sha256 ".*"/sha256 "${{ steps.checksums.outputs.arm64_sha256 }}"/}' Casks/{name}.rb

# 更新 intel sha256
sed -i '/on_intel do/,/end/{s/sha256 ".*"/sha256 "${{ steps.checksums.outputs.x64_sha256 }}"/}' Casks/{name}.rb

# 更新版本
sed -i "s/version \".*\"/version \"${{ steps.version_check.outputs.new_version }}\"/" Casks/{name}.rb
```

## 参考链接

- [Homebrew Cask 官方文档](https://docs.brew.sh/Cask-Cookbook)
- [Homebrew Cask DSL 参考](https://github.com/Homebrew/homebrew-cask/blob/master/DOCUMENTATION.md)
- [livecheck 文档](https://docs.brew.sh/Brew-Livecheck)
# Homebrew Formula 模板语法与结构

## 基本结构

```ruby
class Myapp < Formula
  desc "Description of the formula"
  homepage "https://example.com"
  version "1.0.0"
  license "MIT"

  url "https://example.com/myapp-1.0.0.tar.gz"
  sha256 "PLACEHOLDER"  # workflow 自动更新

  def install
    bin.install "myapp"
  end

  test do
    system "#{bin}/myapp", "--version"
  end
end
```

## 多架构支持

### macOS 多架构

```ruby
class Myapp < Formula
  desc "A great tool"
  homepage "https://github.com/user/repo"
  version "1.0.0"
  license "MIT"

  on_macos do
    if Hardware::CPU.arm?
      url "https://github.com/user/repo/releases/download/v#{version}/myapp-#{version}-macos-arm64"
      sha256 "PLACEHOLDER"  # workflow 自动更新

      def install
        bin.install "myapp-#{version}-macos-arm64" => "myapp"
      end
    end

    if Hardware::CPU.intel?
      url "https://github.com/user/repo/releases/download/v#{version}/myapp-#{version}-macos-x64"
      sha256 "PLACEHOLDER"  # workflow 自动更新

      def install
        bin.install "myapp-#{version}-macos-x64" => "myapp"
      end
    end
  end

  test do
    system "#{bin}/myapp", "--version"
  end
end
```

### 使用 on_arm/on_intel 方法

```ruby
class Myapp < Formula
  desc "A great tool"
  homepage "https://example.com"
  version "1.0.0"

  on_arm do
    url "https://example.com/myapp-arm64.tar.gz"
    sha256 "PLACEHOLDER"
  end

  on_intel do
    url "https://example.com/myapp-x64.tar.gz"
    sha256 "PLACEHOLDER"
  end

  def install
    bin.install "myapp"
  end
end
```

## URL 和版本变量

### 使用 #{version}

```ruby
url "https://github.com/user/repo/releases/download/v#{version}/myapp-#{version}.tar.gz"
```

### URL 变量

```ruby
# 版本号插值
url "https://example.com/myapp-#{version}.tar.gz"

# 自定义变量
url "https://example.com/#{name}.tar.gz"
```

## 依赖声明

### depends_on

```ruby
# 外部命令依赖
depends_on "go" => :build  # 构建依赖
depends_on "python"        # 运行时依赖

# macOS 版本依赖
depends_on macos: ">= :big_sur"

# 架构依赖
depends_on arch: :arm64
```

## 安装方法

### bin.install (推荐)

```ruby
def install
  bin.install "myapp"
  # 等同于 bin.install "myapp" => "myapp"
end

# 重命名
def install
  bin.install "myapp-cli" => "myapp"
end
```

### prefix.install

```ruby
def install
  prefix.install "config"
  bin.install "myapp"
end
```

### man 安装

```ruby
def install
  bin.install "myapp"
  man1.install "docs/myapp.1"
end
```

### lib/pkg-config

```ruby
def install
  lib.install "libmyapp.so"
  (lib/"pkgconfig").install "myapp.pc"
end
```

## 构建配置

### Go 项目

```ruby
class Myapp < Formula
  desc "Go tool"
  homepage "https://example.com"
  version "1.0.0"

  url "https://github.com/user/repo/archive/refs/tags/v#{version}.tar.gz"
  sha256 "PLACEHOLDER"

  depends_on "go" => :build

  def install
    system "go", "build", *std_go_args(ldflags: "-s -w"), "./cmd/myapp"
    bin.install "myapp"
  end
end
```

### Rust 项目

```ruby
class Myapp < Formula
  desc "Rust tool"
  depends_on "rust" => :build

  def install
    system "cargo", "install", *std_cargo_args, "--path", "."
  end
end
```

### Python 项目

```ruby
class Myapp < Formula
  desc "Python tool"
  depends_on "python@3.11"

  def install
    virtualenv_install_with_resources
  end
end
```

## test do 配置

### 版本检查测试

```ruby
test do
  system "#{bin}/myapp", "--version"
end
```

### 功能测试

```ruby
test do
  output = shell_output("#{bin}/myapp --help")
  assert_match "Usage", output
end
```

### 文件测试

```ruby
test do
  (testpath/"test.txt").write "hello"
  output = shell_output("#{bin}/myapp #{testpath}/test.txt")
  assert_match "hello", output
end
```

## caveats 配置

```ruby
def caveats
  <<~EOS
    myapp has been installed!

    To get started, run:
      myapp --version

    For shell integration:
      # Bash
      echo 'eval "$(myapp init bash)"' >> ~/.bashrc

      # Zsh
      echo 'eval "$(myapp init zsh)"' >> ~/.zshrc
  EOS
end
```

## service 配置 (后台服务)

```ruby
service do
  run [bin/"myapp", "--serve"]
  keep_alive true
  log_path var/"log/myapp.log"
  error_log_path var/"log/myapp-error.log"
  working_dir var
end
```

## bottle 配置 (预编译)

```ruby
bottle do
  root_url "https://example.com/bottles"
  sha256 cellar: :any_skip_relocation, arm64_sonoma: "PLACEHOLDER"
  sha256 cellar: :any_skip_relocation, ventura: "PLACEHOLDER"
  sha256 cellar: :any_skip_relocation, big_sur: "PLACEHOLDER"
  rebuild 1
end
```

## livecheck 配置

```ruby
livecheck do
  url :stable
  strategy :github_latest
end
```

## 常见问题

### 1. class 名称命名

❌ `class myapp < Formula` (小写)
✅ `class Myapp < Formula` (首字母大写)

### 2. install 方法中使用变量

❌ `bin.install "myapp-#{version}" => "myapp"` (在 install 中)
✅ 在 install 外使用变量，或使用 `version` 方法

### 3. test 方法返回值

确保 test 方法正确使用 assert:
```ruby
test do
  assert_match "expected", shell_output("#{bin}/myapp")
end
```

## Workflow sed 命令

```bash
# 更新版本
sed -i "s/version \".*\"/version \"${{ steps.version_check.outputs.new_version }}\"/" Formula/{name}.rb

# 更新 URL 中的版本
sed -i "s|/v[0-9.]*\/|/v${{ steps.version_check.outputs.new_version }}/|g" Formula/{name}.rb

# 更新 arm sha256
sed -i '/if Hardware::CPU.arm?/,/if Hardware::CPU.intel?/{s/sha256 ".*"/sha256 "${{ steps.checksums.outputs.arm64_sha256 }}"/}' Formula/{name}.rb

# 更新 intel sha256
sed -i '/if Hardware::CPU.intel?/,/end/{s/sha256 ".*"/sha256 "${{ steps.checksums.outputs.x64_sha256 }}"/}' Formula/{name}.rb
```

## 参考链接

- [Homebrew Formula Cookbook](https://docs.brew.sh/Formula-Cookbook)
- [Homebrew Formula DSL](https://rubydoc.brew.sh/Formula)
- [Homebrew New Formula Checklist](https://docs.brew.sh/How-To-Open-A-Homebrew-Pull-Request)
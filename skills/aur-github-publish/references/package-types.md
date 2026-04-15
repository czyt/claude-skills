# 不同包类型处理

## 二进制包 (bin)

直接下载预编译二进制文件。

### PKGBUILD 模板

```bash
pkgname=myapp-bin
pkgver=1.0.0
pkgrel=1
arch=('x86_64')

source=("https://github.com/user/repo/releases/download/v${pkgver}/myapp")
sha256sums=('SKIP')

package() {
    install -Dm755 myapp "${pkgdir}/usr/bin/myapp"
}
```

### 多架构二进制

```bash
arch=('x86_64' 'aarch64')

source_x86_64=("https://github.com/user/repo/releases/download/v${pkgver}/myapp-x86_64")
source_aarch64=("https://github.com/user/repo/releases/download/v${pkgver}/myapp-aarch64")

sha256sums_x86_64=('SKIP')
sha256sums_aarch64=('SKIP')

package() {
    install -Dm755 myapp "${pkgdir}/usr/bin/myapp"
}
```

---

## deb 包转换

从 deb 包提取文件并安装。

### PKGBUILD 模板

```bash
pkgname=myapp-bin
pkgver=1.0.0
pkgrel=1
arch=('x86_64')

source_x86_64=("myapp.deb::https://github.com/user/repo/releases/download/v${pkgver}/myapp-${pkgver}-amd64.deb")
sha256sums_x86_64=('SKIP')

package() {
    # 解压 deb 包
    ar p "${srcdir}/myapp.deb" data.tar.gz | tar xz -C "${pkgdir}"

    # 修复权限
    chmod -R u=rwX,go=rX "${pkgdir}"
}
```

### 多架构 deb

```bash
arch=('x86_64' 'aarch64')

source_x86_64=("CC-Switch.deb::https://github.com/user/repo/releases/download/v${pkgver}/CC-Switch-v${pkgver}-Linux-x86_64.deb")
source_aarch64=("CC-Switch.deb::https://github.com/user/repo/releases/download/v${pkgver}/CC-Switch-v${pkgver}-Linux-arm64.deb")

sha256sums_x86_64=('SKIP')
sha256sums_aarch64=('SKIP')

package() {
    local _debfile
    if [[ "$CARCH" == "x86_64" ]]; then
        _debfile="CC-Switch.deb"
    else
        _debfile="CC-Switch-arm64.deb"
    fi

    ar p "${srcdir}/${_debfile}" data.tar.gz | tar xz -C "${pkgdir}"
    chmod -R u=rwX,go=rX "${pkgdir}"
}
```

### deb 包处理要点

- deb 包使用 `ar` 命令解压
- 解压后包含: `control.tar.gz`, `data.tar.gz`, `debian-binary`
- `data.tar.gz` 包含实际文件
- 使用 `chmod -R u=rwX,go=rX` 修复权限

---

## tar.gz 包

从压缩包解压并安装。

### PKGBUILD 模板

```bash
pkgname=myapp-bin
pkgver=1.0.0
pkgrel=1
arch=('x86_64')

source=("https://github.com/user/repo/releases/download/v${pkgver}/myapp-${pkgver}-linux-amd64.tar.gz")
sha256sums=('SKIP')

package() {
    # tar.gz 自动解压到 ${srcdir}
    install -Dm755 myapp "${pkgdir}/usr/bin/myapp"
}
```

### 包含多个文件的 tar.gz

```bash
package() {
    cd "${srcdir}"

    # 安装二进制
    install -Dm755 myapp "${pkgdir}/usr/bin/myapp"

    # 安装配置文件
    install -Dm644 config.yaml "${pkgdir}/etc/myapp/config.yaml"

    # 安装桌面文件
    install -Dm644 myapp.desktop "${pkgdir}/usr/share/applications/myapp.desktop"
}
```

---

## AppImage 包

AppImage 是自包含的可执行文件。

### PKGBUILD 模板

```bash
pkgname=myapp-bin
pkgver=1.0.0
pkgrel=1
arch=('x86_64')

source=("https://github.com/user/repo/releases/download/v${pkgver}/MyApp-${pkgver}-x86_64.AppImage")
sha256sums=('SKIP')

package() {
    install -Dm755 MyApp-${pkgver}-x86_64.AppImage "${pkgdir}/opt/myapp/MyApp.AppImage"

    # 创建启动脚本
    install -dm755 "${pkgdir}/usr/bin"
    ln -s /opt/myapp/MyApp.AppImage "${pkgdir}/usr/bin/myapp"

    # 创建桌面文件
    install -Dm644 /dev/stdin "${pkgdir}/usr/share/applications/myapp.desktop" <<EOF
[Desktop Entry]
Name=MyApp
Exec=myapp
Icon=myapp
Type=Application
Categories=Utility;
EOF
}
```

---

## rpm 包转换

从 rpm 包提取文件。

### PKGBUILD 模板

```bash
pkgname=myapp-bin
pkgver=1.0.0
pkgrel=1
arch=('x86_64')

source=("https://github.com/user/repo/releases/download/v${pkgver}/myapp-${pkgver}-1.x86_64.rpm")
sha256sums=('SKIP')

depends=('rpm-tools')

package() {
    # 使用 rpm2cpio 解压
    rpm2cpio "${srcdir}/myapp-${pkgver}-1.x86_64.rpm" | cpio -idmv -D "${pkgdir}"
}
```

---

## macOS dmg (不适用于 AUR)

dmg 是 macOS 特有格式，AUR 不处理 dmg 包。

---

## 源码包

需要编译的源码包。

### Go 项目

```bash
pkgname=myapp
pkgver=1.0.0
pkgrel=1
arch=('x86_64' 'aarch64')

source=("https://github.com/user/repo/archive/v${pkgver}.tar.gz")
sha256sums=('SKIP')

makedepends=('go')

build() {
    cd "${srcdir}/myapp-${pkgver}"
    go build -ldflags="-s -w" -o myapp .
}

package() {
    cd "${srcdir}/myapp-${pkgver}"
    install -Dm755 myapp "${pkgdir}/usr/bin/myapp"
}
```

### Rust 项目

```bash
makedepends=('rust')

build() {
    cd "${srcdir}/myapp-${pkgver}"
    cargo build --release --locked
}

package() {
    install -Dm755 target/release/myapp "${pkgdir}/usr/bin/myapp"
}
```

---

## 特殊处理

### udev 规则

```bash
package() {
    # 安装 udev 规则
    install -Dm644 /dev/stdin "${pkgdir}/usr/lib/udev/rules.d/51-myapp.rules" << 'EOF'
SUBSYSTEM=="usb", ATTRS{idVendor}=="1234", ATTRS{idProduct}=="5678", MODE="0666"
EOF
}

post_install() {
    udevadm control --reload-rules 2>/dev/null || true
}

post_remove() {
    udevadm control --reload-rules 2>/dev/null || true
}
```

### systemd 服务

```bash
package() {
    install -Dm644 myapp.service "${pkgdir}/usr/lib/systemd/system/myapp.service"
}
```

### 桌面文件

```bash
package() {
    install -Dm644 myapp.desktop "${pkgdir}/usr/share/applications/myapp.desktop"

    # 图标
    for size in 16 32 48 64 128 256 512; do
        install -Dm644 "icons/${size}x${size}/myapp.png" \
            "${pkgdir}/usr/share/icons/hicolor/${size}x${size}/apps/myapp.png"
    done
}
```

---

## 参考链接

- [Arch 包指南](https://wiki.archlinux.org/title/Arch_package_guidelines)
- [PKGBUILD 参考](https://wiki.archlinux.org/title/PKGBUILD)
---
name: aur-github-publish
description: AUR GitHub 发布助手 - 自动更新 PKGBUILD 版本、发布到 GitHub 仓库并同步到 AUR。支持手动版本输入、自动版本检测、pkgrel bump、多架构支持（x86_64/aarch64）、deb/rpm/tar.gz 包处理。触发词：更新 AUR 包、发布到 AUR、创建 PKGBUILD、更新 Arch Linux 包、AUR 自动发布。
---

# AUR GitHub 发布助手

协助将软件版本更新到 AUR (Arch User Repository)，自动化版本检测、PKGBUILD 更新、GitHub 发布和 AUR 同步流程。

## Reference Documents

| Document | Content |
|----------|---------|
| [references/pkgbuild-template.md](references/pkgbuild-template.md) | PKGBUILD 模板语法与结构 |
| [references/workflow-template.md](references/workflow-template.md) | GitHub Actions workflow 模板（含真实实例） |
| [references/aur-deploy-action.md](references/aur-deploy-action.md) | AUR 部署 action 配置 + 首次发布指南 |
| [references/package-types.md](references/package-types.md) | 不同包类型处理 (deb/rpm/tar.gz/AppImage) |
| [references/best-practices.md](references/best-practices.md) | 最佳实践与常见问题 |
| [references/real-world-examples.md](references/real-world-examples.md) | 真实项目示例（含 GitHub Action 实例） |

---

## 仓库结构

### AUR GitHub 标准布局

```
aur-repo/
├── {pkgname}/
│   ├── PKGBUILD           # 包构建文件
│   ├── {pkgname}.install  # 安装钩子（可选）
│   └── .SRCINFO           # 源信息（自动生成）
├── .github/
│   ├── workflows/
│   │   ├── update-{pkgname}.yml
│   │   └── ...
│   └── scripts/
│       └── prune-aur-workdir.sh
└── README.md
```

---

## 核心功能

### 1. 版本检测

**手动版本输入**: 用户指定版本号
**自动版本检测**: 从 GitHub Releases API 获取最新版本

```bash
# GitHub API 获取最新版本
curl -s https://api.github.com/repos/{owner}/{repo}/releases/latest | jq -r '.tag_name' | sed 's/^v//'
```

### 2. checksum 处理

**⚠️ 重要: checksum 在 GitHub Actions workflow 运行时通过 `updpkgsums` 自动生成，PKGBUILD 文件中使用 'SKIP' 占位符**

`updpkgsums` action 参数会:
1. 下载 source 文件
2. 计算并更新 sha256sums/md5sums
3. 更新 PKGBUILD 中的 checksum 字段

```yaml
- name: Publish to AUR
  uses: KSXGitHub/github-actions-deploy-aur@v4.1.2
  with:
    updpkgsums: true  # ✅ 自动计算 checksum
```

### 3. pkgrel 管理

- **新版本**: pkgver 更新，pkgrel 重置为 1
- **同版本修复**: 只 bump pkgrel

```bash
# 新版本
sed -i "s/^pkgver=.*/pkgver=${NEW_VERSION}/" PKGBUILD
sed -i "s/^pkgrel=.*/pkgrel=1/" PKGBUILD

# 同版本修复
CURRENT_REL=$(grep '^pkgrel=' PKGBUILD | cut -d'=' -f2)
NEW_REL=$((CURRENT_REL + 1))
sed -i "s/^pkgrel=.*/pkgrel=$NEW_REL/" PKGBUILD
```

### 3. AUR 同步

使用 `KSXGitHub/github-actions-deploy-aur` action 自动发布到 AUR:

```yaml
- name: Publish to AUR
  uses: KSXGitHub/github-actions-deploy-aur@v4.1.2
  with:
    pkgname: {pkgname}
    pkgbuild: ./{pkgname}/PKGBUILD
    updpkgsums: true
    commit_username: ${{ secrets.AUR_USERNAME }}
    commit_email: ${{ secrets.AUR_EMAIL }}
    ssh_private_key: ${{ secrets.AUR_SSH_PRIVATE_KEY }}
    commit_message: "Update to version {version}"
    ssh_keyscan_types: rsa,ecdsa,ed25519
```

---

## PKGBUILD 模板

### 基本 PKGBUILD 结构

**⚠️ sha256sums 使用 'SKIP' 占位符，workflow 运行时通过 `updpkgsums: true` 自动更新**

```bash
# Maintainer: {maintainer} <{email}>
pkgname={pkgname}
pkgver={version}  # workflow 会更新
pkgrel=1
pkgdesc="{description}"
arch=('x86_64')
url="{homepage}"
license=('{license}')
depends=({dependencies})
provides=('{provides}')
conflicts=('{conflicts}')
source_x86_64=("{url}/releases/download/v${pkgver}/{file}")
sha256sums_x86_64=('SKIP')  # ✅ workflow 通过 updpkgsums 自动计算

package() {
    # 包安装逻辑
}
```

### 多架构 PKGBUILD

**⚠️ checksum 使用 'SKIP'，workflow 自动计算**

```bash
pkgname={pkgname}
pkgver={version}
pkgrel=1
arch=('x86_64' 'aarch64')

source_x86_64=("{url}/releases/download/v${pkgver}/{file}-x86_64.tar.gz")
source_aarch64=("{url}/releases/download/v${pkgver}/{file}-aarch64.tar.gz")

sha256sums_x86_64=('SKIP')  # ✅ workflow 自动计算
sha256sums_aarch64=('SKIP')  # ✅ workflow 自动计算

package() {
    install -Dm755 {binary} "${pkgdir}/usr/bin/{name}"
}
```

### deb 包处理

**⚠️ checksum 使用 'SKIP'**

```bash
source_x86_64=("{file}.deb::${url}/releases/download/v${pkgver}/{deb_file}")
sha256sums_x86_64=('SKIP')  # workflow 自动计算

package() {
    # Extract deb package
    ar p "${srcdir}/${source_x86_64[0]}" data.tar.gz | tar xz -C "${pkgdir}"
    chmod -R u=rwX,go=rX "${pkgdir}"
}
```

---

## GitHub Actions Workflow

### 标准 Workflow 模板

```yaml
name: Update {pkgname} Version

on:
  workflow_dispatch:
    inputs:
      version:
        description: "Version number. Leave empty to auto-detect."
        required: false
        type: string
      force:
        description: "Force update (bump pkgrel)"
        required: false
        default: false
        type: boolean
  schedule:
    - cron: "0 */12 * * *"

jobs:
  update-pkgbuild:
    runs-on: ubuntu-latest
    permissions:
      contents: write

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Get latest version
        id: get_version
        run: |
          if [ -n "${{ github.event.inputs.version }}" ]; then
            echo "version=${{ github.event.inputs.version }}" >> $GITHUB_OUTPUT
          else
            VERSION=$(curl -s https://api.github.com/repos/{owner}/{repo}/releases/latest | jq -r '.tag_name' | sed 's/^v//')
            echo "version=$VERSION" >> $GITHUB_OUTPUT
          fi

      - name: Get current version
        id: current_version
        run: |
          CURRENT=$(grep '^pkgver=' {pkgname}/PKGBUILD | cut -d'=' -f2)
          echo "current=$CURRENT" >> $GITHUB_OUTPUT

      - name: Compare versions
        id: compare
        run: |
          if [ "${{ steps.get_version.outputs.version }}" = "${{ steps.current_version.outputs.current }}" ]; then
            if [ "${{ github.event.inputs.force }}" = "true" ]; then
              echo "needs_update=true" >> $GITHUB_OUTPUT
              echo "bump_rel=true" >> $GITHUB_OUTPUT
            else
              echo "needs_update=false" >> $GITHUB_OUTPUT
            fi
          else
            echo "needs_update=true" >> $GITHUB_OUTPUT
            echo "bump_rel=false" >> $GITHUB_OUTPUT
          fi

      - name: Update PKGBUILD
        if: steps.compare.outputs.needs_update == 'true'
        run: |
          cd {pkgname}
          if [ "${{ steps.compare.outputs.bump_rel }}" = "true" ]; then
            CURRENT_REL=$(grep '^pkgrel=' PKGBUILD | cut -d'=' -f2)
            NEW_REL=$((CURRENT_REL + 1))
            sed -i "s/^pkgrel=.*/pkgrel=$NEW_REL/" PKGBUILD
          else
            sed -i "s/^pkgver=.*/pkgver=${{ steps.get_version.outputs.version }}/" PKGBUILD
            sed -i "s/^pkgrel=.*/pkgrel=1/" PKGBUILD
          fi

      - name: Commit changes
        if: steps.compare.outputs.needs_update == 'true'
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add {pkgname}/PKGBUILD
          git commit -m "Update {pkgname} to version ${{ steps.get_version.outputs.version }}"
          git push

      - name: Publish to AUR
        if: steps.compare.outputs.needs_update == 'true'
        uses: KSXGitHub/github-actions-deploy-aur@v4.1.2
        with:
          pkgname: {pkgname}
          pkgbuild: ./{pkgname}/PKGBUILD
          updpkgsums: true
          post_process: bash /github/workspace/.github/scripts/prune-aur-workdir.sh .
          commit_username: ${{ secrets.AUR_USERNAME }}
          commit_email: ${{ secrets.AUR_EMAIL }}
          ssh_private_key: ${{ secrets.AUR_SSH_PRIVATE_KEY }}
          commit_message: "Update to version ${{ steps.get_version.outputs.version }}"
          ssh_keyscan_types: rsa,ecdsa,ed25519
```

---

## Secrets 配置

需要在 GitHub 仓库配置以下 Secrets:

| Secret | 说明 |
|--------|------|
| `AUR_USERNAME` | AUR 用户名 |
| `AUR_EMAIL` | AUR 邮箱 |
| `AUR_SSH_PRIVATE_KEY` | AUR SSH 私钥 |

### SSH 密钥配置

```bash
# 生成 SSH 密钥
ssh-keygen -f aur_key -t ed25519 -C "your@email.com"

# 上传公钥到 AUR
# https://aur.archlinux.org/account/

# 将私钥添加到 GitHub Secrets
cat aur_key  # 复制完整内容，包括 BEGIN/END 行
```

### ⚠️ 首次发布前必须创建 AUR 仓库

**首次发布新包时，必须先克隆并初始化 AUR 仓库**:

```bash
# 1. 克隆空的 AUR 仓库（创建新包）
git clone ssh://aur.archlinux.org/{pkgname}.git /tmp/{pkgname}

# 2. 进入目录，添加基本文件
cd /tmp/{pkgname}

# 3. 创建 PKGBUILD（可以从模板复制）
cp ~/your-project/PKGBUILD .

# 4. 生成 .SRCINFO
makepkg --printsrcinfo > .SRCINFO

# 5. 配置 git 用户信息
git config user.name "your-aur-username"
git config user.email "your@email.com"

# 6. 提交并推送
git add PKGBUILD .SRCINFO
git commit -m "Initial commit: {pkgname}"
git push origin master
```

**注意**: GitHub Actions workflow 只能推送修改，不能创建新仓库。首次发布必须手动完成以上步骤。

---

## 工作流程

### Phase 1: 信息收集与前置检查

**输入**: 用户请求（创建/更新 PKGBUILD）
**输出**: 上游仓库信息 + AUR 仓库状态确认

#### Step 1.1: 确认上游仓库信息

**⚠️ 检查点**: 在创建之前，必须确认以下信息：

| 必需信息 | 说明 | 用户确认方式 |
|---------|------|-------------|
| GitHub owner/repo | 上游仓库地址 | 用户提供或从 URL 提取 |
| 发布文件格式 | deb、tar.gz、二进制等 | 分析 GitHub Releases |
| 架构支持 | x86_64、aarch64 或两者 | 分析发布文件命名 |
| 包名称 | 用于 pkgname（推荐 `-bin` 后缀） | 用户确认 |
| 运行时依赖 | depends 列表 | 分析应用文档或询问 |

```
询问用户：
「请确认以下信息：
- GitHub 仓库: {owner}/{repo}
- 发布文件格式: {format}
- 架构支持: {archs}
- 包名称: {pkgname}-bin
- 运行时依赖: {depends}

是否正确？」
```

#### Step 1.2: 检查 AUR 仓库状态

**⚠️ 关键检查点**: 确认 AUR 仓库是否存在

```bash
# 检查包是否已存在
curl -s "https://aur.archlinux.org/packages/{pkgname}" | grep -q "Package Details"
```

| 状态 | 处理方式 |
|------|---------|
| **已存在** | 直接进入更新流程 |
| **不存在** | 提示用户需要首次手动创建 |

```
如果包不存在：
「⚠️ 该包在 AUR 中不存在，首次发布需要手动操作：
1. 克隆空仓库: git clone ssh://aur.archlinux.org/{pkgname}.git
2. 添加 PKGBUILD 和 .SRCINFO
3. 推送创建仓库

是否需要我提供详细步骤？」
```

---

### Phase 2: 文件生成

**输入**: Phase 1 确认的信息
**输出**: PKGBUILD + GitHub Workflow + 相关文件

#### Step 2.1: 选择包类型模板

| 包类型 | 模板选择 | source 格式 |
|-------|---------|-------------|
| 二进制 | 直接安装 | `{url}/{binary}` |
| deb 包 | ar 解压 | `{file}.deb::${url}/{deb_file}` |
| tar.gz | 解压安装 | `{url}/{file}.tar.gz` |
| AppImage | 单文件 | `{url}/{file}.AppImage` |

**⚠️ 检查点**: checksum 使用策略确认：

```
询问用户：
「PKGBUILD 中的 checksum 使用 SKIP 占位符，
GitHub Actions workflow 通过 updpkgsums: true 自动计算。
是否继续？」
```

#### Step 2.2: 生成 PKGBUILD

根据包类型生成对应的 PKGBUILD，确保：
- `sha256sums=('SKIP')` 或 `sha256sums_x86_64=('SKIP')`
- 正确的 `arch` 声明
- `provides` 和 `conflicts` 声明

#### Step 2.3: 生成 GitHub Workflow

生成自动更新 workflow，包含：
- `workflow_dispatch`: 手动触发 + 版本输入 + force 参数
- `schedule`: 定时检查（推荐每12小时）
- `updpkgsums: true`: 自动 checksum
- AUR 发布步骤

---

### Phase 3: 验证与发布

**输入**: Phase 2 生成的文件
**输出**: 验证结果 + 发布

#### Step 3.1: 本地验证（推荐）

```bash
# 验证 PKGBUILD
namcap PKGBUILD

# 生成 .SRCINFO
makepkg --printsrcinfo > .SRCINFO
```

**⚠️ 检查点**: 验证失败时询问用户：

```
如果 namcap 报错：
「验证发现问题：{error}
是否继续提交，或先修复问题？」
```

#### Step 3.2: 提交到 GitHub 仓库

```bash
git add {pkgname}/PKGBUILD {pkgname}/.SRCINFO .github/workflows/update-{pkgname}.yml
git commit -m "feat: add {pkgname} AUR package with auto-update workflow"
git push
```

#### Step 3.3: AUR 发布（首次需手动）

| 场景 | 操作 |
|------|------|
| **首次发布** | 参考 `references/aur-deploy-action.md` 手动创建 |
| **更新版本** | workflow 自动触发 AUR 发布 |

---

## 边界条件与错误处理

| 异常情况 | 处理方式 | Fallback |
|---------|---------|---------|
| AUR SSH 认证失败 | 检查私钥格式和公钥上传 | 提示用户检查 Secrets |
| GitHub API 不可达 | 提示用户手动输入版本 | 使用 workflow_dispatch 输入 |
| 发布文件不存在 | 检查 URL 格式和版本号 | 询问用户确认文件命名 |
| pkgrel bump 需求 | 同版本修复 | workflow force 参数 |
| deb 包权限问题 | chmod 修复 | `chmod -R u=rwX,go=rX` |
| 多架构文件缺失 | 只有单架构 | 询问用户是否降级 |
| AUR 包名冲突 | 搜索 AUR 确认 | 建议使用 `-bin` 后缀 |

---

## 决策速查表

| 场景 | 决策 |
|------|------|
| 用户未提供包名后缀 | **推荐**: 使用 `-bin` 后缀 |
| AUR 包是否已存在 | **必须检查**: curl AUR API 或询问用户 |
| checksum 是否预填 | **使用 SKIP**: workflow 通过 updpkgsums 计算 |
| 是否需要 .install 文件 | **可选**: 有 post_install 钩子时添加 |
| 定时更新频率 | **推荐**: 每12小时 (`0 */12 * * *`) |
| Secrets 是否配置 | **首次提示**: 需配置 AUR_USERNAME/AUR_EMAIL/AUR_SSH_PRIVATE_KEY |

---

## 常用命令

### 本地验证

```bash
# 生成 .SRCINFO
makepkg --printsrcinfo > .SRCINFO

# 验证 PKGBUILD
namcap PKGBUILD

# 本地构建测试
makepkg -sf

# 安装测试
sudo pacman -U {pkgname}-{version}-1-x86_64.pkg.tar.zst
```

### 版本检测

```bash
# 获取最新版本
curl -s https://api.github.com/repos/{owner}/{repo}/releases/latest | jq -r '.tag_name' | sed 's/^v//'

# 检查当前版本
grep '^pkgver=' PKGBUILD | cut -d'=' -f2
```

---

## 包类型处理

### 二进制包 (bin)

**⚠️ checksum 使用 'SKIP'，workflow 通过 `updpkgsums: true` 自动计算**

```bash
source=("{url}/releases/download/v${pkgver}/{binary}")
sha256sums=('SKIP')  # ✅ workflow 自动计算

package() {
    install -Dm755 {binary} "${pkgdir}/usr/bin/{name}"
}
```

### deb 包转换

从 deb 包提取文件:

```bash
source=("{file}.deb::${url}/releases/download/v${pkgver}/{deb_file}")
sha256sums=('SKIP')  # workflow 自动计算

package() {
    ar p "${srcdir}/${source}" data.tar.gz | tar xz -C "${pkgdir}"
    chmod -R u=rwX,go=rX "${pkgdir}"
}
```

### tar.gz 包

解压并安装:

```bash
source=("{url}/releases/download/v${pkgver}/{file}.tar.gz")
sha256sums=('SKIP')  # workflow 自动计算

package() {
    install -Dm755 ${srcdir}/{binary} "${pkgdir}/usr/bin/{name}"
}
```

---

## 最佳实践

### ✅ 推荐做法

1. **使用 `-bin` 后缀**: 预编译包使用 `{name}-bin`
2. **提供 provides/conflicts**: 与官方包冲突声明
3. **定时更新**: 设置 schedule 自动检查版本
4. **手动触发**: 提供 workflow_dispatch 支持手动指定版本
5. **pkgrel 管理**: 同版本修复只 bump pkgrel

### ❌ 避免的做法

1. **硬编码版本**: 不更新 pkgver
2. **缺少依赖声明**: 导致安装失败
3. **跳过验证**: 不运行 namcap
4. **忽略错误**: curl 失败不检查
5. **不更新 .SRCINFO**: AUR 无法正确索引

---

## 触发场景

使用此技能的场景:

- 「帮我更新 AUR 中的 xxx 包」
- 「创建一个新的 PKGBUILD」
- 「发布 xxx 到 AUR」
- 「更新 Arch Linux 包版本」
- 「配置 AUR 自动发布」
- 「创建 xxx 的 AUR workflow」

---

## 输入要求

创建/更新 PKGBUILD 时，请提供:

1. **上游仓库信息**: GitHub owner/repo
2. **包名称**: 用于 pkgname
3. **发布文件格式**: deb、rpm、tar.gz、二进制等
4. **架构支持**: x86_64、aarch64 或两者
5. **版本号**: 手动指定或自动检测
6. **依赖信息**: 运行时依赖

---

## 输出格式

技能将生成:

1. **PKGBUILD 文件**: 包构建定义
2. **.install 文件**: 安装钩子（可选）
3. **GitHub Workflow**: `.github/workflows/update-xxx.yml`
4. **更新说明**: commit message 格式
5. **验证结果**: namcap 输出

详见各 reference 文档获取完整模板和示例。
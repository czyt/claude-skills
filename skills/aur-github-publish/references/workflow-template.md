# GitHub Actions Workflow 模板

## Mutable `latest` Debian 包：URL 无版本号

当上游始终覆盖 `latest/app_amd64.deb`、`latest/app_arm64.deb` 时，不能从 URL 推导版本，也不能只比较 `pkgver`。使用以下状态机：

1. 对每个架构执行 `HEAD`，读取 ETag。ETag 仅用于减少大文件下载，不是版本号或完整性证明。
2. 将上一次成功发布的 ETag 存入 GitHub-only 状态文件，例如 `{pkgname}/upstream.env`。
3. ETag 缺失、变化或用户显式指定版本时，下载所有架构的 deb。
4. 从每个 deb 的 control 元数据读取 `Version` 和 `Architecture`，拒绝架构错误或版本不同步。
5. 新版本：更新 `pkgver`、重置 `pkgrel=1`；ETag 变化但版本相同：增加 `pkgrel`。
6. PKGBUILD checksum 始终为 `SKIP`；AUR deploy action 使用 `updpkgsums: true` 写入真实 checksum。
7. 精确列出 AUR assets，并在 `post_process` 后断言 `upstream.env` 不存在。
8. **先发布到 AUR，成功后再提交 GitHub 的 PKGBUILD 与 ETag 状态。**发布失败时旧状态不变，下次调度会自动重试。

### GitHub-only 状态文件

```dotenv
# {pkgname}/upstream.env — 不得发布到 AUR
ETAG_X86_64=0x0123456789ABCDEF
ETAG_AARCH64=0xFEDCBA9876543210
```

不要把 ETag 放进 PKGBUILD。PKGBUILD 会被 deploy action 复制到 AUR；独立状态文件能清楚分隔“GitHub 调度状态”和“AUR 构建输入”。

### Debian 元数据提取

Ubuntu runner 可直接使用 `dpkg-deb`：

```bash
version=$(dpkg-deb --field app_amd64.deb Version)
arch=$(dpkg-deb --field app_amd64.deb Architecture)
```

如果 runner 没有 `dpkg-deb`，不要硬编码 `control.tar.xz`。动态找到 control member：

```bash
control_member=$(ar t app_amd64.deb | awk '/^control\.tar\.(gz|xz|zst|bz2|lzma)$/ { print; exit }')
[[ -n "$control_member" ]] || { echo "deb has no supported control archive" >&2; exit 1; }
ar p app_amd64.deb "$control_member" | bsdtar -xOf - ./control
```

### Workflow 骨架

以下骨架省略项目特定 URL、依赖和提交信息。替换 `{pkgname}` 与两个 URL 后再使用：

```yaml
name: Update {pkgname}

on:
  workflow_dispatch:
    inputs:
      version:
        description: "Expected version; empty means read it from the deb"
        required: false
        type: string
      force:
        description: "Force a packaging-only pkgrel bump"
        required: false
        default: false
        type: boolean
  schedule:
    - cron: "0 */12 * * *"

concurrency:
  group: aur-{pkgname}
  cancel-in-progress: false

jobs:
  update-pkgbuild:
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - uses: actions/checkout@<full-commit-sha> # pin verified release

      - name: Read upstream validators
        id: validators
        env:
          INPUT_VERSION: ${{ inputs.version }}
        run: |
          set -euo pipefail
          header_value() {
            local url=$1 header=$2
            curl --fail --silent --show-error --location --head \
              --retry 3 --retry-all-errors --connect-timeout 15 --max-time 120 \
              "$url" | awk -v wanted="$header" '
                tolower($1) == tolower(wanted) ":" {
                  sub(/^[^:]+:[[:space:]]*/, ""); sub(/\r$/, ""); print
                }
              ' | tail -n1
          }

          AMD64_URL='<latest-amd64-url>'
          ARM64_URL='<latest-arm64-url>'
          NEW_X86=$(header_value "$AMD64_URL" etag)
          NEW_ARM=$(header_value "$ARM64_URL" etag)
          OLD_X86=$(sed -n 's/^ETAG_X86_64=//p' {pkgname}/upstream.env)
          OLD_ARM=$(sed -n 's/^ETAG_AARCH64=//p' {pkgname}/upstream.env)

          if [[ -n "$INPUT_VERSION" || -z "$NEW_X86" || -z "$NEW_ARM" || \
                "$NEW_X86" != "$OLD_X86" || "$NEW_ARM" != "$OLD_ARM" ]]; then
            echo "download=true" >> "$GITHUB_OUTPUT"
          else
            echo "download=false" >> "$GITHUB_OUTPUT"
          fi
          echo "x86_etag=$NEW_X86" >> "$GITHUB_OUTPUT"
          echo "arm_etag=$NEW_ARM" >> "$GITHUB_OUTPUT"

      - name: Download and verify all architectures
        if: steps.validators.outputs.download == 'true'
        id: release
        env:
          INPUT_VERSION: ${{ inputs.version }}
        run: |
          set -euo pipefail
          curl -fSL --retry 3 --retry-all-errors --max-time 900 \
            '<latest-amd64-url>' -o app_amd64.deb
          curl -fSL --retry 3 --retry-all-errors --max-time 900 \
            '<latest-arm64-url>' -o app_arm64.deb

          X86_VERSION=$(dpkg-deb --field app_amd64.deb Version)
          ARM_VERSION=$(dpkg-deb --field app_arm64.deb Version)
          [[ $(dpkg-deb --field app_amd64.deb Architecture) == amd64 ]]
          [[ $(dpkg-deb --field app_arm64.deb Architecture) == arm64 ]]
          [[ "$X86_VERSION" == "$ARM_VERSION" ]] || {
            echo "architectures are not released atomically" >&2; exit 1;
          }
          [[ "$X86_VERSION" =~ ^[0-9]+([.][0-9]+)+$ ]] || {
            echo "unsupported pkgver: $X86_VERSION" >&2; exit 1;
          }
          [[ -z "$INPUT_VERSION" || "${INPUT_VERSION#v}" == "$X86_VERSION" ]] || {
            echo "requested version does not match latest deb" >&2; exit 1;
          }
          echo "version=$X86_VERSION" >> "$GITHUB_OUTPUT"

      - name: Decide update
        id: compare
        env:
          DOWNLOADED: ${{ steps.validators.outputs.download }}
          FORCE: ${{ inputs.force }}
          LATEST: ${{ steps.release.outputs.version }}
        run: |
          set -euo pipefail
          CURRENT=$(sed -n 's/^pkgver=//p' {pkgname}/PKGBUILD)
          UPDATE=false; BUMP=false
          if [[ "$DOWNLOADED" == true ]]; then
            if [[ "$LATEST" != "$CURRENT" ]]; then
              [[ $(printf '%s\n%s\n' "$CURRENT" "$LATEST" | sort -V | head -n1) == "$CURRENT" ]] || {
                echo "refusing automatic downgrade" >&2; exit 1;
              }
              UPDATE=true
            else
              UPDATE=true; BUMP=true
            fi
          elif [[ "$FORCE" == true ]]; then
            UPDATE=true; BUMP=true
          fi
          echo "update=$UPDATE" >> "$GITHUB_OUTPUT"
          echo "bump=$BUMP" >> "$GITHUB_OUTPUT"

      - name: Stage package metadata
        if: steps.compare.outputs.update == 'true'
        env:
          BUMP: ${{ steps.compare.outputs.bump }}
          LATEST: ${{ steps.release.outputs.version }}
          X86_ETAG: ${{ steps.validators.outputs.x86_etag }}
          ARM_ETAG: ${{ steps.validators.outputs.arm_etag }}
        run: |
          set -euo pipefail
          cd {pkgname}
          if [[ "$BUMP" == true ]]; then
            rel=$(sed -n 's/^pkgrel=//p' PKGBUILD)
            sed -i "s/^pkgrel=.*/pkgrel=$((rel + 1))/" PKGBUILD
          else
            sed -i "s/^pkgver=.*/pkgver=$LATEST/" PKGBUILD
            sed -i 's/^pkgrel=.*/pkgrel=1/' PKGBUILD
          fi
          [[ -z "$X86_ETAG" ]] || sed -i "s/^ETAG_X86_64=.*/ETAG_X86_64=$X86_ETAG/" upstream.env
          [[ -z "$ARM_ETAG" ]] || sed -i "s/^ETAG_AARCH64=.*/ETAG_AARCH64=$ARM_ETAG/" upstream.env

      - name: Publish to AUR
        if: steps.compare.outputs.update == 'true'
        uses: KSXGitHub/github-actions-deploy-aur@<full-commit-sha>
        with:
          pkgname: {pkgname}
          pkgbuild: ./{pkgname}/PKGBUILD
          assets: ./{pkgname}/launcher.sh
          updpkgsums: true
          post_process: >-
            bash /github/workspace/.github/scripts/prune-aur-workdir.sh . launcher.sh
            && test -f launcher.sh
            && test ! -e upstream.env
          commit_username: ${{ secrets.AUR_USERNAME }}
          commit_email: ${{ secrets.AUR_EMAIL }}
          ssh_private_key: ${{ secrets.AUR_SSH_PRIVATE_KEY }}
          commit_message: "Publish ${{ steps.release.outputs.version }}"

      - name: Commit successful state to GitHub
        if: steps.compare.outputs.update == 'true'
        run: |
          set -euo pipefail
          git config user.name 'github-actions[bot]'
          git config user.email 'github-actions[bot]@users.noreply.github.com'
          git add {pkgname}/PKGBUILD {pkgname}/upstream.env
          git commit -m 'Record successfully published AUR state'
          git push
```

### 包含与排除规则

- `assets` 是包含列表：只列 `source=()` 需要但不在远端 URL 的 launcher、patch、install、service 等文件。
- GitHub-only 状态不得出现在 `assets`。
- `post_process` 是最后防线：清理下载产物，并断言必需 asset 存在、状态文件不存在。
- 如果共享 prune 脚本支持额外白名单参数，传入精确 basename；不要用 `*.sh` 这种过宽规则。

### 首次空 AUR 仓库验证

首次只需让 `ssh://aur.archlinux.org/{pkgname}.git` 可被克隆；deploy action 能向空仓库创建 root commit。触发完整发布后验证三层状态：

```bash
gh run watch <run-id> --exit-status
git clone https://aur.archlinux.org/{pkgname}.git /tmp/{pkgname}-verify
find /tmp/{pkgname}-verify -maxdepth 1 -type f -printf '%f\n' | sort
curl -fsSL 'https://aur.archlinux.org/rpc/?v=5&type=info&arg={pkgname}'
```

AUR Git 已出现 commit 而 RPC 暂时为 `resultcount: 0` 时，先按索引延迟处理；轮询页面/RPC，不能据此重复发布。检查 AUR Git 中只包含 `.SRCINFO`、PKGBUILD 与白名单 assets，并确认 checksum 已由 action 生成。

### 失败分支

| 触发条件 | 一线处理 | 仍失败兜底 |
|---|---|---|
| ETag 缺失 | 下载并检查所有架构 | 使用 Last-Modified + Content-Length 仅作下载提示；仍以包内版本为准 |
| 两架构 Version 不同 | 立即失败，不更新状态 | 等上游完成原子发布后由下一次 schedule 重试 |
| control archive 无法读取 | 优先 `dpkg-deb --field` | 动态查找 `control.tar.*`；仍失败则停止发布 |
| AUR deploy 失败 | 不提交 PKGBUILD/ETag 状态 | 修复 SSH、source 或 asset 后重跑 |
| GitHub 状态提交失败但 AUR 已成功 | 读取 AUR Git 的 pkgver/pkgrel 后对齐 GitHub | 不要盲目 force 再发布 |
| AUR RPC 暂无结果 | 克隆 AUR Git 验证 commit | 等待索引同步，不重复 bump pkgrel |

### Mutable URL 反例黑名单

- 不要从 `latest` URL、ETag 或 Last-Modified 猜 `pkgver`。
- 不要只下载一个架构后假设另一个架构同步。
- 不要在 AUR 发布成功前保存新 ETag。
- 不要把 `upstream.env`、API 响应或下载缓存放进 `assets`。
- 不要把 action 生成的真实 checksum 回写到要求保留 `SKIP` 的 GitHub PKGBUILD。
- 不要把 ETag 当完整性校验；完整性由 AUR 中固定 checksum 承担。
- 不要让 schedule 与 workflow_dispatch 并发修改同一 PKGBUILD；必须设置 `concurrency`。

## AUR 自动更新 + 发布 Workflow

### 基本模板

本节是常规 GitHub Release 的最小骨架。套用前必须同时满足：网络请求 fail-fast、解析结果非空、资产可下载、自动更新不降级、同包 workflow 设置 `concurrency`。新建 workflow 时 pin 已核验的 action commit SHA；示例中的 tag 只表示最低功能版本。

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
        uses: actions/checkout@v6

      - name: Get latest version
        id: get_version
        run: |
          if [ -n "${{ inputs.version }}" ]; then
            echo "version=${{ inputs.version }}" >> $GITHUB_OUTPUT
          else
            VERSION=$(curl -fsSL --retry 3 --retry-all-errors \
              https://api.github.com/repos/{owner}/{repo}/releases/latest |
              jq -er '.tag_name | sub("^v"; "")')
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
            if [ "${{ inputs.force }}" = "true" ]; then
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
            # Bump pkgrel
            CURRENT_REL=$(grep '^pkgrel=' PKGBUILD | cut -d'=' -f2)
            NEW_REL=$((CURRENT_REL + 1))
            sed -i "s/^pkgrel=.*/pkgrel=$NEW_REL/" PKGBUILD
          else
            # Update version
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
        uses: KSXGitHub/github-actions-deploy-aur@v4.1.3
        with:
          pkgname: {pkgname}
          pkgbuild: ./{pkgname}/PKGBUILD
          updpkgsums: true  # ✅ 自动计算 checksum
          commit_username: ${{ secrets.AUR_USERNAME }}
          commit_email: ${{ secrets.AUR_EMAIL }}
          ssh_private_key: ${{ secrets.AUR_SSH_PRIVATE_KEY }}
          commit_message: "Update to version ${{ steps.get_version.outputs.version }}"
          ssh_keyscan_types: rsa,ecdsa,ed25519
```

## KSXGitHub/github-actions-deploy-aur 参数

| 参数 | 说明 | 必需 |
|------|------|------|
| `pkgname` | AUR 包名 | ✅ |
| `pkgbuild` | PKGBUILD 文件路径 | ✅ |
| `updpkgsums` | 自动计算 checksum | 推荐 |
| `assets` | 附加文件 (.install, .patch) | 可选 |
| `commit_username` | AUR 用户名 | ✅ |
| `commit_email` | AUR 邮箱 | ✅ |
| `ssh_private_key` | AUR SSH 私钥 | ✅ |
| `commit_message` | 提交消息 | ✅ |
| `post_process` | 后处理脚本 | 可选 |
| `ssh_keyscan_types` | SSH 密钥类型 | 推荐 |

## 真实实例

### 实例 1: 二进制包自动更新

来源: aur/.github/workflows/update-autocli-bin.yml

```yaml
name: Update autocli-bin Version

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
        uses: actions/checkout@v6

      - name: Get latest version
        id: get_version
        run: |
          if [ -n "${{ github.event.inputs.version }}" ]; then
            echo "version=${{ github.event.inputs.version }}" >> $GITHUB_OUTPUT
          else
            VERSION=$(curl -s https://api.github.com/repos/nashsu/AutoCLI/releases/latest | jq -r '.tag_name' | sed 's/^v//')
            echo "version=$VERSION" >> $GITHUB_OUTPUT
          fi

      - name: Get current version
        id: current_version
        run: |
          CURRENT=$(grep '^pkgver=' autocli-bin/PKGBUILD | cut -d'=' -f2)
          echo "current=$CURRENT" >> $GITHUB_OUTPUT

      - name: Compare versions
        id: compare
        run: |
          if [ "${{ steps.get_version.outputs.version }}" = "${{ steps.current_version.outputs.current }}" ]; then
            if [ "${{ github.event.inputs.force }}" = "true" ]; then
              echo "needs_update=true" >> $GITHUB_OUTPUT
              echo "bump_rel=true" >> $GITHUB_OUTPUT
              echo "Forcing pkgrel bump for version ${{ steps.get_version.outputs.version }}"
            else
              echo "needs_update=false" >> $GITHUB_OUTPUT
              echo "Version is already up to date"
            fi
          else
            echo "needs_update=true" >> $GITHUB_OUTPUT
            echo "bump_rel=false" >> $GITHUB_OUTPUT
            echo "Updating from ${{ steps.current_version.outputs.current }} to ${{ steps.get_version.outputs.version }}"
          fi

      - name: Update PKGBUILD version
        if: steps.compare.outputs.needs_update == 'true'
        run: |
          cd autocli-bin
          if [ "${{ steps.compare.outputs.bump_rel }}" = "true" ]; then
            CURRENT_REL=$(grep '^pkgrel=' PKGBUILD | cut -d'=' -f2)
            NEW_REL=$((CURRENT_REL + 1))
            sed -i "s/^pkgrel=.*/pkgrel=$NEW_REL/" PKGBUILD
            echo "Bumped pkgrel from $CURRENT_REL to $NEW_REL"
          else
            sed -i "s/^pkgver=.*/pkgver=${{ steps.get_version.outputs.version }}/" PKGBUILD
            sed -i "s/^pkgrel=.*/pkgrel=1/" PKGBUILD
          fi

      - name: Commit changes
        if: steps.compare.outputs.needs_update == 'true'
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add autocli-bin/PKGBUILD
          if [ "${{ steps.compare.outputs.bump_rel }}" = "true" ]; then
            git commit -m "Bump pkgrel for autocli-bin ${{ steps.get_version.outputs.version }}"
          else
            git commit -m "Update autocli-bin to version ${{ steps.get_version.outputs.version }}"
          fi
          git push

      - name: Publish to AUR
        if: steps.compare.outputs.needs_update == 'true'
        uses: KSXGitHub/github-actions-deploy-aur@v4.1.3
        with:
          pkgname: autocli-bin
          pkgbuild: ./autocli-bin/PKGBUILD
          assets: ./autocli-bin/autocli-bin.install  # 包含 install 文件
          updpkgsums: true  # ✅ 自动计算 checksum
          post_process: bash /github/workspace/.github/scripts/prune-aur-workdir.sh .
          commit_username: ${{ secrets.AUR_USERNAME }}
          commit_email: ${{ secrets.AUR_EMAIL }}
          ssh_private_key: ${{ secrets.AUR_SSH_PRIVATE_KEY }}
          commit_message: ${{ steps.compare.outputs.bump_rel == 'true' && format('Bump pkgrel for version {0}', steps.get_version.outputs.version) || format('Update to version {0}', steps.get_version.outputs.version) }}
          ssh_keyscan_types: rsa,ecdsa,ed25519
```

**关键点**:
- `force` 参数支持同版本 bump pkgrel
- `assets` 包含 .install 文件
- `updpkgsums: true` 自动计算 checksum
- `post_process` 清理工作目录

### 实例 2: deb 包自动更新

来源: aur/.github/workflows/update-cc-switch-bin.yml

```yaml
- name: Update PKGBUILD version
  if: steps.compare.outputs.needs_update == 'true'
  run: |
    cd cc-switch-bin
    if [ "${{ steps.compare.outputs.bump_rel }}" = "true" ]; then
      CURRENT_REL=$(grep '^pkgrel=' PKGBUILD | cut -d'=' -f2)
      NEW_REL=$((CURRENT_REL + 1))
      sed -i "s/^pkgrel=.*/pkgrel=$NEW_REL/" PKGBUILD
    else
      sed -i "s/^pkgver=.*/pkgver=${{ steps.get_version.outputs.version }}/" PKGBUILD
      sed -i "s/^pkgrel=.*/pkgrel=1/" PKGBUILD
    fi
```

### 实例 3: 多架构包

```yaml
- name: Update PKGBUILD
  run: |
    cd {pkgname}
    # 更新版本
    sed -i "s/^pkgver=.*/pkgver=${{ steps.get_version.outputs.version }}/" PKGBUILD
    sed -i "s/^pkgrel=.*/pkgrel=1/" PKGBUILD

    # 更新 source URL（如果有版本号在 URL 中）
    sed -i "s|v[0-9.]*-Linux|v${{ steps.get_version.outputs.version }}-Linux|g" PKGBUILD
```

## prune-aur-workdir.sh 脚本

来源: aur/.github/scripts/prune-aur-workdir.sh

```bash
#!/usr/bin/env bash

set -euo pipefail

workdir=${1:-.}

if [[ ! -d "${workdir}" ]]; then
  echo "workdir does not exist: ${workdir}" >&2
  exit 1
fi

shopt -s dotglob nullglob

for path in "${workdir}"/*; do
  name=$(basename -- "${path}")

  case "${name}" in
    .|..|.git|PKGBUILD|.SRCINFO|*.install|*.patch|*.conf|*.service|*.desktop)
      continue
      ;;
  esac

  rm -rf -- "${path}"
done
```

**作用**: 清理 AUR 发布目录，只保留必要文件:
- PKGBUILD
- .SRCINFO
- *.install
- *.patch
- *.conf
- *.service
- *.desktop

## Secrets 配置

### GitHub Secrets 设置

| Secret | 获取方式 |
|--------|----------|
| `AUR_USERNAME` | AUR 注册用户名 |
| `AUR_EMAIL` | AUR 注册邮箱 |
| `AUR_SSH_PRIVATE_KEY` | AUR SSH 私钥 |

### SSH 密钥生成

```bash
# 生成 SSH 密钥
ssh-keygen -f aur_key -t ed25519 -C "your@email.com"

# 上传公钥到 AUR
# 登录 https://aur.archlinux.org/account/
# 在 "SSH Public Key" 字段粘贴 aur_key.pub 内容

# 将私钥添加到 GitHub Secrets
# 复制 aur_key 完整内容（包括 BEGIN/END 行）
```

## 常用 sed 命令

```bash
# 更新 pkgver
sed -i "s/^pkgver=.*/pkgver=${NEW_VERSION}/" PKGBUILD

# 更新 pkgrel
sed -i "s/^pkgrel=.*/pkgrel=1/" PKGBUILD

# bump pkgrel
CURRENT_REL=$(grep '^pkgrel=' PKGBUILD | cut -d'=' -f2)
NEW_REL=$((CURRENT_REL + 1))
sed -i "s/^pkgrel=.*/pkgrel=$NEW_REL/" PKGBUILD

# 更新 source URL 中的版本
sed -i "s|v[0-9.]*-|v${NEW_VERSION}-|g" PKGBUILD
```

## 参考链接

- [KSXGitHub/github-actions-deploy-aur](https://github.com/KSXGitHub/github-actions-deploy-aur)
- [GitHub Actions 文档](https://docs.github.com/en/actions)
- [AUR 提交指南](https://wiki.archlinux.org/title/AUR_submission_guidelines)

---

## ⚠️ GitHub Actions 版本检查

**在使用本模板之前，请检查以下 GitHub Actions 的最新版本**：

```bash
# 检查 actions/checkout 最新版本
curl -s https://api.github.com/repos/actions/checkout/releases/latest | jq -r '.tag_name'

# 检查 KSXGitHub/github-actions-deploy-aur 最新版本
curl -s https://api.github.com/repos/KSXGitHub/github-actions-deploy-aur/releases/latest | jq -r '.tag_name'

# 检查 actions/setup-go 最新版本
curl -s https://api.github.com/repos/actions/setup-go/releases/latest | jq -r '.tag_name'

# 检查 goreleaser/goreleaser-action 最新版本
curl -s https://api.github.com/repos/goreleaser/goreleaser-action/releases/latest | jq -r '.tag_name'
```

| Action | 当前模板版本 | 检查最新 |
|--------|-------------|---------|
| `actions/checkout` | v6 | [releases](https://github.com/actions/checkout/releases) |
| `KSXGitHub/github-actions-deploy-aur` | v4.1.3 | [releases](https://github.com/KSXGitHub/github-actions-deploy-aur/releases) |
| `actions/setup-go` | v6 | [releases](https://github.com/actions/setup-go/releases) |
| `goreleaser/goreleaser-action` | v7 | [releases](https://github.com/goreleaser/goreleaser-action/releases) |
| `softprops/action-gh-release` | v3 | [releases](https://github.com/softprops/action-gh-release/releases) |
| `actions/upload-artifact` | v7 | [releases](https://github.com/actions/upload-artifact/releases) |
| `actions/download-artifact` | v8 | [releases](https://github.com/actions/download-artifact/releases) |

**最佳实践**: 每次创建新 workflow 时，先检查上述 Actions 是否有新版本发布，使用最新版本可以获得更好的性能和安全性。

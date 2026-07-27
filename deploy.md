# 部署与 CI 指南

本文说明如何配置 Secrets、更新加密源码、触发 GitHub Actions 构建，以及发布 Release。

源码本身不落明文；解密口令通过 GitHub Actions Secret 注入。

## Secrets

| Name | 说明 |
|------|------|
| `SOURCE_GPG_PASSPHRASE` | 与打包脚本加密时相同的 GPG 对称口令 |
| `TAURI_SIGNING_PRIVATE_KEY` | Tauri updater 私钥内容（或 base64）；无则跳过 updater 签名产物 |
| `TAURI_SIGNING_PRIVATE_KEY_PASSWORD` | 私钥密码（可选） |

## 更新加密源码

在 `fast-sign` 仓库执行：

```bash
export SOURCE_GPG_PASSPHRASE='your-passphrase'
./desktop/scripts/pack_encrypted_source.sh
# 默认写出到 ../apple-sign-desktop-action/source.zip.gpg
```

然后在本仓库提交并 push `source.zip.gpg`。

## 触发构建

### 手动触发

Actions → **Build Desktop** → Run workflow。

- 填写 **Release tag**（如 `v1.0.0`）：构建完成后自动创建 GitHub Release 并上传各平台安装包。
- 留空：仅上传 workflow artifacts，不创建 Release。

### 推送 tag 自动发布

推送符合 `v*` 的 tag（如 `v1.0.0`）也会触发构建，并在成功后发布到 GitHub Release。

### 推送加密源码

提交并 push `source.zip.gpg` 会触发构建；未带 Release tag 时只保留 workflow artifacts（14 天）。

构建矩阵：

- `macos-latest`（Apple Silicon）
- `macos-13`（Intel x86_64，若 runner 可用）
- `ubuntu-22.04`
- `windows-latest`

产物同时上传为 workflow artifacts；若指定了 Release tag，还会出现在仓库的 **Releases** 页面。

## Author / 元数据

桌面应用 Author 在加密源码中配置（`desktop/src-tauri/Cargo.toml`、`tauri.conf.json`），不含个人邮箱。设置页展示为「猪猪」。

Release 说明使用固定文案，不会从 commit 自动生成（避免暴露 git 作者邮箱）。若希望 git 历史也不带个人邮箱，可配置 GitHub 提供的 `noreply` 地址：

```bash
git config user.email "ID+username@users.noreply.github.com"
```

（`ID` 与 `username` 见 GitHub → Settings → Emails）

## Workflow 权限

工作流使用默认 `GITHUB_TOKEN`，无需额外 Secret。权限在 workflow 中显式声明：

| 范围 | 用途 |
|------|------|
| `contents: read` | 检出代码（build job） |
| `actions: write` | 上传构建产物 artifact |
| `contents: write` | 创建 Release、上传安装包（release job） |
| `actions: read` | 下载各平台 artifact（release job） |

若仓库 **Settings → Actions → General → Workflow permissions** 设为「Read repository contents and packages permissions only」，以上 job 级权限声明仍会生效；无需改为全局 Read and write。

## 本地解密试跑

```bash
gpg --batch --yes --pinentry-mode loopback \
  --passphrase "$SOURCE_GPG_PASSPHRASE" \
  --decrypt -o source.zip source.zip.gpg
unzip -q source.zip -d src
cd src/desktop && npm ci && npm run tauri build
```

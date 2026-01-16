# GitHub Actions 自助构建指南

本文档说明如何使用 GitHub Actions 自动构建 Simple Live 应用。

## 工作流概览

项目包含 4 个 GitHub Actions 工作流：

| 工作流文件 | 用途 | 触发条件 | 分支 |
|-----------|------|---------|------|
| `publish_app_dev.yaml` | 开发版应用构建 | `dev_v*` 标签或手动触发 | `dev` |
| `publish_app_release.yml` | 正式版应用发布 | `v*` 标签 | `master` |
| `publish_tv_app_dev.yaml` | 开发版 TV 应用构建 | `dev_tv_v*` 标签 | `dev` |
| `publish_tv_app_release.yaml` | 正式版 TV 应用发布 | `tv_*` 标签 | `master` |

## 构建平台

### 普通应用 (Simple Live App)
- **Android**: APK (armeabi-v7a, arm64-v8a, x86_64)
- **iOS**: IPA (未签名)
- **macOS**: DMG, ZIP
- **Linux**: DEB, ZIP
- **Windows**: MSIX, ZIP

### TV 应用 (Simple Live TV App)
- **Android TV**: APK (armeabi-v7a, arm64-v8a, x86_64)

## 配置步骤

### 1. 生成 Android 签名密钥

使用 Java 自带的 `keytool` 命令生成签名密钥：

```bash
keytool -genkey -v -keystore keystore.jks -keyalg RSA -keysize 2048 -validity 10000 -alias your-key-alias
```

**参数说明：**
- `-keystore keystore.jks`: 生成的密钥库文件名
- `-keyalg RSA`: 密钥算法
- `-keysize 2048`: 密钥长度
- `-validity 10000`: 有效期（10000 天，约 27 年）
- `-alias your-key-alias`: 密钥别名（自定义）

**执行后会提示输入：**
1. 密钥库口令（存储密码）- 输入并记住
2. 再次输入新口令 - 确认存储密码
3. 密钥口令（密钥密码）- 输入并记住
4. 再次输入新口令 - 确认密钥密码
5. 姓名等信息 - 可随意填写或直接回车跳过

### 2. 生成 keystore Base64 编码

将 keystore.jks 文件转换为 Base64 编码：

```bash
# macOS
base64 -i keystore.jks | pbcopy

# Linux
base64 -w 0 keystore.jks
```

### 3. 配置 GitHub Secrets

进入 GitHub 仓库页面：**Settings** → **Secrets and variables** → **Actions** → **New repository secret**

#### 开发版构建所需的 Secrets

| Secret 名称 | 值 | 说明 |
|------------|-----|------|
| `KEYSTORE_BASE64` | keystore.jks 的 Base64 编码 | 普通应用签名密钥 |
| `STORE_PASSWORD` | 密钥库口令 | 存储密码 |
| `KEY_PASSWORD` | 密钥口令 | 密钥密码 |
| `KEY_ALIAS` | 密钥别名 | 命令中的 `-alias` 参数 |
| `TV_KEYSTORE_BASE64` | 同上（与 KEYSTORE_BASE64 相同） | TV 应用签名密钥 |
| `TV_STORE_PASSWORD` | 同上（与 STORE_PASSWORD 相同） | TV 存储密码 |
| `TV_KEY_PASSWORD` | 同上（与 KEY_PASSWORD 相同） | TV 密钥密码 |
| `TV_KEY_ALIAS` | 同上（与 KEY_ALIAS 相同） | TV 密钥别名 |

> **提示**：如果普通应用和 TV 应用使用同一个签名密钥，只需准备一套密钥信息，然后在 Secrets 中配置两份（key 名称不同，值相同）。

#### 正式版发布额外需要的 Secrets

TODO: ...

<!-- | Secret 名称 | 值 | 说明 |
|------------|-----|------|
| `TOKEN` | GitHub Personal Access Token | 用于创建 Release |

**获取 GitHub Token：**
1. 进入 GitHub Settings → Developer settings → Personal access tokens → Tokens (classic)
2. 点击 "Generate new token (classic)"
3. 勾选 `repo` 权限
4. 生成并复制 token -->

## 触发构建

### 开发版构建

#### 普通应用

```bash
# 推送标签触发构建
git tag dev_v1.0.0
git push origin dev_v1.0.0
```

或在 GitHub Actions 页面手动触发 `app-build-action-dev` 工作流。

#### TV 应用

```bash
# 推送标签触发构建
git tag dev_tv_v1.0.0
git push origin dev_tv_v1.0.0
```

### 正式版发布

#### 普通应用

```bash
# 推送标签触发构建并发布
git tag v1.0.0
git push origin v1.0.0
```

#### TV 应用

```bash
# 推送标签触发构建并发布
git tag tv_v1.0.0
git push origin tv_v1.0.0
```

## 下载构建产物

### 开发版

1. 进入 GitHub 仓库的 **Actions** 页面
2. 点击对应的工作流运行记录
3. 滚动到底部找到 **Artifacts** 区域
4. 下载对应的构建产物

### 正式版

1. 进入 GitHub 仓库的 **Releases** 页面
2. 找到对应的版本
3. 下载所需的安装包

## 重要提醒

### 签名密钥安全

- **妥善保管 keystore.jks 文件**，丢失后无法更新应用
- 记住所有密码，否则无法签名
- 将 keystore.jks 文件备份到安全位置（如加密的云存储）
- 不要将 keystore.jks 文件提交到 Git 仓库

### 版本信息管理

- 正式版发布前，需要更新 `assets/app_version.json`（普通应用）或 `assets/tv_app_version.json`（TV 应用）
- 版本信息格式示例：

```json
{
  "version": "v1.0.0",
  "version_desc": "发布说明内容",
  "prerelease": false
}
```

### iOS 签名说明

- 当前工作流生成的 iOS IPA 为**未签名版本**
- 需要使用 Xcode 进行签名后才能安装到真机
- 或使用第三方签名工具（如 AltStore、Sideloadly）

## 常见问题

### Q: 构建失败怎么办？

A: 检查以下几点：
1. GitHub Secrets 是否正确配置
2. keystore.jks 的 Base64 编码是否正确
3. 分支是否正确（dev 分支用 dev 版本，master 分支用正式版）
4. 标签格式是否正确

### Q: 如何只构建特定平台？

A: 当前工作流会构建所有支持的平台。如需修改，可以编辑对应的工作流文件，注释掉不需要的构建步骤。

### Q: 可以自定义 Flutter 版本吗？

A: 可以。编辑工作流文件中的 `flutter-version` 参数（当前为 3.38.x）。

### Q: 如何修改应用图标和名称？

A: 修改对应平台的配置文件：
- Android: `simple_live_app/android/app/src/main/AndroidManifest.xml`
- iOS: `simple_live_app/ios/Runner/Info.plist`
- macOS: `simple_live_app/macos/Runner/Configs/AppInfo.xcconfig`
- Windows: `simple_live_app/windows/runner/Runner.rc`
- Linux: `simple_live_app/linux/my_application.cc`

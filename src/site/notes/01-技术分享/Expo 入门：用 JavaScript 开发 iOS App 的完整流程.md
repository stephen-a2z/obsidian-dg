---
{"dg-publish":true,"permalink":"/01-技术分享/Expo 入门：用 JavaScript 开发 iOS App 的完整流程/","tags":["javascript","expo","ios","app","development"],"noteIcon":"","created":"2026-04-21T11:05:12.144+08:00","updated":"2026-04-21T11:58:22.507+08:00"}
---


# Expo 入门：用 JavaScript 开发 iOS App 的完整流程

> 你想做一个 iOS App，但不想学 Swift，不想买 Mac（其实还是要买），不想跟 Xcode 搏斗？Expo 可能是目前最省心的方案。

## 一、Expo 是什么

Expo 是一个基于 React Native 的应用开发平台。React Native 让你用 JavaScript/TypeScript 写原生移动应用，而 Expo 在这个基础上做了大量封装，把移动开发中最痛苦的部分——原生配置、构建、签名、发布——变成了几条命令。

Meta（React Native 的维护者）现在官方推荐 Expo 作为创建 React Native 项目的默认方式。

目前最新的稳定版是 SDK 53，基于 React Native 0.79 和 React 19，默认启用了 New Architecture。

## 二、为什么选 Expo

### 跟纯 React Native 对比

| 维度 | 纯 React Native | Expo |
|------|-----------------|------|
| 项目初始化 | 需要配置 Xcode、CocoaPods、Gradle | 一条命令 |
| 原生模块 | 手动链接，写 Objective-C/Swift 桥接 | 大部分已封装好，直接 `npx expo install` |
| iOS 构建 | 必须有 Mac + Xcode | EAS Build 云端构建（不需要 Mac） |
| 签名管理 | 手动管理证书和 Provisioning Profile | EAS 自动处理 |
| OTA 更新 | 需要自建方案（CodePush 等） | EAS Update 内置支持 |
| 发布到 App Store | 手动用 Xcode 或 Transporter 上传 | `eas submit` 一条命令 |

### 跟 Flutter、Swift 对比

| 维度 | Expo (React Native) | Flutter | Swift (原生) |
|------|---------------------|---------|-------------|
| 语言 | JavaScript/TypeScript | Dart | Swift |
| 跨平台 | iOS + Android + Web | iOS + Android + Web | 仅 Apple 平台 |
| 生态 | npm 生态，极其丰富 | pub.dev，在增长 | Apple 原生 API |
| 热更新 | ✅ OTA 更新不需要重新提审 | ❌ 需要重新提审 | ❌ 需要重新提审 |
| 性能 | 接近原生 | 接近原生 | 原生 |
| 学习曲线 | 低（会 React 就行） | 中（需要学 Dart） | 高（需要学 Swift + Apple 生态） |

Expo 的核心优势：**如果你已经会 React，上手成本几乎为零**。而且 OTA 热更新是杀手级功能——修个 bug 不用等 App Store 审核，直接推送到用户手机上。

## 三、核心概念

在动手之前，先理解几个关键概念：

### Expo Go vs Development Build

- **Expo Go**：一个预装了常用原生模块的沙盒 App。你在手机上装好 Expo Go，扫个码就能预览你的项目。适合学习和快速原型验证，但不能用于生产。
- **Development Build**：包含你项目实际需要的原生代码的调试版本。相当于你自己的"Expo Go"，但只包含你用到的模块。**正式开发应该用这个**。

Expo 官方已经明确表示 Expo Go 是学习工具，不适合构建要上架的 App。

### EAS（Expo Application Services）

Expo 的云服务套件，包含三个核心服务：

- **EAS Build**：云端构建 iOS/Android 应用。你不需要本地配置 Xcode 或 Android Studio，代码推上去，云端帮你编译出 `.ipa`（iOS）或 `.apk`/`.aab`（Android）。
- **EAS Submit**：把构建好的包直接提交到 App Store 或 Google Play，不需要手动上传。
- **EAS Update**：OTA 热更新。修改了 JavaScript 代码后，直接推送到已安装的用户设备上，不需要重新提审。

### app.json / app.config.js

项目的核心配置文件，定义了 App 的名称、图标、启动画面、权限、版本号等所有元信息。

## 四、从零开始：开发一个 iOS App

### 前置条件

- Node.js 18+
- 一个 Apple Developer 账号（$99/年，上架 App Store 必须）
- 一个 Expo 账号（免费注册：https://expo.dev）
- 如果要在本地 iOS 模拟器调试：需要 Mac + Xcode

### 第一步：创建项目

```bash
npx create-expo-app@latest my-app
cd my-app
```

这会创建一个基于最新 SDK 的项目，默认使用 TypeScript 和文件系统路由（Expo Router）。

项目结构：

```
my-app/
├── app/                  # 页面路由（文件系统路由）
│   ├── _layout.tsx       # 根布局
│   ├── index.tsx         # 首页
│   └── +not-found.tsx    # 404 页面
├── assets/               # 静态资源（图片、字体）
├── components/           # 可复用组件
├── constants/            # 常量定义
├── app.json              # Expo 配置
├── package.json
└── tsconfig.json
```

### 第二步：启动开发服务器

```bash
npx expo start
```

终端会显示一个二维码。你可以：

- 用 iPhone 上的 Expo Go 扫码预览（快速体验）
- 按 `i` 在 iOS 模拟器中打开（需要 Mac + Xcode）
- 按 `w` 在浏览器中打开（Web 版）

### 第三步：写你的第一个页面

编辑 `app/index.tsx`：

```tsx
import { Text, View, StyleSheet, Pressable, Alert } from 'react-native';

export default function HomeScreen() {
  return (
    <View style={styles.container}>
      <Text style={styles.title}>我的第一个 App</Text>
      <Pressable style={styles.button} onPress={() => Alert.alert('Hello!')}>
        <Text style={styles.buttonText}>点我</Text>
      </Pressable>
    </View>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1, justifyContent: 'center', alignItems: 'center' },
  title: { fontSize: 24, fontWeight: 'bold', marginBottom: 20 },
  button: { backgroundColor: '#007AFF', paddingHorizontal: 24, paddingVertical: 12, borderRadius: 8 },
  buttonText: { color: '#fff', fontSize: 16 },
});
```

保存后，手机上会自动热更新。这就是 Expo 的开发体验——改代码，立刻看到效果。

### 第四步：添加原生功能

比如要用摄像头：

```bash
npx expo install expo-camera
```

```tsx
import { CameraView, useCameraPermissions } from 'expo-camera';
import { View, Text, Pressable } from 'react-native';

export default function CameraScreen() {
  const [permission, requestPermission] = useCameraPermissions();

  if (!permission?.granted) {
    return (
      <View style={{ flex: 1, justifyContent: 'center', alignItems: 'center' }}>
        <Text>需要相机权限</Text>
        <Pressable onPress={requestPermission}>
          <Text>授权</Text>
        </Pressable>
      </View>
    );
  }

  return <CameraView style={{ flex: 1 }} facing="back" />;
}
```

注意：用了 `expo-camera` 这类原生模块后，Expo Go 可能不支持（取决于模块），你需要切换到 Development Build。

### 第五步：创建 Development Build

```bash
# 安装 EAS CLI
npm install -g eas-cli

# 登录 Expo 账号
eas login

# 初始化 EAS 配置
eas build:configure
```

这会在项目根目录生成 `eas.json`：

```json
{
  "build": {
    "development": {
      "developmentClient": true,
      "distribution": "internal"
    },
    "preview": {
      "distribution": "internal"
    },
    "production": {}
  }
}
```

构建开发版本：

```bash
# 云端构建（不需要 Mac）
eas build --platform ios --profile development

# 或者本地构建（需要 Mac + Xcode）
eas build --platform ios --profile development --local
```

首次构建时，EAS 会自动帮你创建 Apple 开发证书和 Provisioning Profile。你只需要输入 Apple ID 和密码（或 App-Specific Password）。

构建完成后，你会得到一个可以安装到真机上的开发版 App。

### 第六步：配置 App 信息

编辑 `app.json`，填写上架需要的信息：

```json
{
  "expo": {
    "name": "我的 App",
    "slug": "my-app",
    "version": "1.0.0",
    "icon": "./assets/icon.png",
    "splash": {
      "image": "./assets/splash-icon.png",
      "resizeMode": "contain",
      "backgroundColor": "#ffffff"
    },
    "ios": {
      "bundleIdentifier": "com.yourname.myapp",
      "buildNumber": "1",
      "supportsTablet": true,
      "infoPlist": {
        "NSCameraUsageDescription": "需要使用相机来拍照"
      }
    }
  }
}
```

关键字段：
- `bundleIdentifier`：App 的唯一标识，格式为反向域名，一旦上架不能改
- `buildNumber`：每次提交到 App Store 都要递增
- `infoPlist`：iOS 权限说明文字，用到什么权限就要写对应的描述

### 第七步：构建生产版本

```bash
eas build --platform ios --profile production
```

这会在 EAS 云端编译出一个可以提交到 App Store 的 `.ipa` 文件。构建过程通常需要 10-20 分钟。

你可以在 https://expo.dev 的 Dashboard 上查看构建进度和日志。

### 第八步：提交到 App Store

在提交之前，你需要先在 [App Store Connect](https://appstoreconnect.apple.com/) 上创建你的 App 记录，填写：

- App 名称和描述
- 截图（至少需要 6.7 英寸和 5.5 英寸两种尺寸）
- 隐私政策 URL
- 年龄分级
- 联系信息

然后一条命令提交：

```bash
eas submit --platform ios
```

EAS 会把构建好的 `.ipa` 上传到 App Store Connect。上传完成后，你可以在 App Store Connect 中：

1. 先通过 **TestFlight** 分发给测试人员
2. 测试没问题后，提交审核
3. Apple 审核通过后，发布到 App Store

整个审核通常需要 1-3 天。

## 五、上架后的更新策略

这是 Expo 最强大的地方之一。

### JavaScript 更新（不需要重新提审）

如果你只改了 JavaScript/TypeScript 代码（UI、业务逻辑、样式等）：

```bash
eas update --branch production --message "修复了首页加载问题"
```

用户下次打开 App 时会自动下载更新。不需要经过 App Store 审核。

这就是 OTA（Over-The-Air）更新。适用于：
- Bug 修复
- UI 调整
- 业务逻辑变更
- 文案修改

### 原生更新（需要重新提审）

如果你添加了新的原生模块、升级了 SDK 版本、或修改了 `app.json` 中的原生配置：

```bash
# 重新构建
eas build --platform ios --profile production
# 重新提交
eas submit --platform ios
```

需要经过完整的 App Store 审核流程。

### 版本管理建议

```json
// app.json
{
  "expo": {
    "version": "1.1.0",        // 用户可见的版本号，语义化版本
    "ios": {
      "buildNumber": "5"        // 每次提交到 App Store 递增
    }
  }
}
```

也可以用 EAS 自动递增：

```json
// eas.json
{
  "build": {
    "production": {
      "autoIncrement": true
    }
  }
}
```

## 六、Expo Router：文件系统路由

Expo Router 是 Expo 的路由方案，灵感来自 Next.js 的文件系统路由。`app/` 目录下的文件结构直接对应 App 的页面结构：

```
app/
├── _layout.tsx          # 根布局（Tab 导航、Stack 导航等）
├── index.tsx            # 首页 → /
├── about.tsx            # 关于页 → /about
├── settings/
│   ├── _layout.tsx      # settings 的子布局
│   ├── index.tsx        # → /settings
│   └── profile.tsx      # → /settings/profile
└── post/
    └── [id].tsx         # 动态路由 → /post/123
```

根布局示例（Tab 导航）：

```tsx
import { Tabs } from 'expo-router';
import { Ionicons } from '@expo/vector-icons';

export default function Layout() {
  return (
    <Tabs>
      <Tabs.Screen
        name="index"
        options={{ title: '首页', tabBarIcon: ({ color }) => <Ionicons name="home" size={24} color={color} /> }}
      />
      <Tabs.Screen
        name="settings"
        options={{ title: '设置', tabBarIcon: ({ color }) => <Ionicons name="settings" size={24} color={color} /> }}
      />
    </Tabs>
  );
}
```

页面间导航：

```tsx
import { Link } from 'expo-router';

// 声明式
<Link href="/post/42">查看文章</Link>

// 命令式
import { router } from 'expo-router';
router.push('/post/42');
```

如果你用过 Next.js，这套路由系统会非常熟悉。

## 七、常用的 Expo 模块

| 模块 | 功能 | 安装 |
|------|------|------|
| `expo-camera` | 相机 | `npx expo install expo-camera` |
| `expo-location` | 地理位置 | `npx expo install expo-location` |
| `expo-notifications` | 推送通知 | `npx expo install expo-notifications` |
| `expo-image-picker` | 图片选择 | `npx expo install expo-image-picker` |
| `expo-secure-store` | 安全存储（类似 Keychain） | `npx expo install expo-secure-store` |
| `expo-file-system` | 文件系统操作 | `npx expo install expo-file-system` |
| `expo-haptics` | 触觉反馈 | `npx expo install expo-haptics` |
| `expo-local-authentication` | 生物识别（Face ID / Touch ID） | `npx expo install expo-local-authentication` |
| `expo-sqlite` | 本地 SQLite 数据库 | `npx expo install expo-sqlite` |

始终用 `npx expo install` 而不是 `npm install` 来安装 Expo 模块，它会自动匹配当前 SDK 版本兼容的包版本。

## 八、常见问题

### 不用 Mac 能开发 iOS App 吗？

**开发阶段**：可以。用 EAS Build 云端构建 Development Build，安装到 iPhone 上调试。但你没法用 iOS 模拟器（模拟器只能在 Mac 上跑）。

**上架阶段**：可以。EAS Build + EAS Submit 全程云端完成。

**实际建议**：如果你认真做 iOS 开发，还是建议有一台 Mac。本地模拟器调试的效率远高于每次都等云端构建。

### EAS Build 收费吗？

免费账号每月有 30 次 iOS 构建和 30 次 Android 构建。对个人开发者够用。超出后需要付费计划（$99/月起）。

本地构建（`--local`）不消耗配额，但需要 Mac + Xcode。

### 能用第三方原生库吗？

可以。Expo 现在支持 Config Plugins 机制，大部分 React Native 社区的原生库都可以通过 Config Plugin 集成，不需要手动修改原生代码。

如果遇到没有 Config Plugin 的库，你可以用 `npx expo prebuild` 生成原生项目目录（ios/ 和 android/），然后手动配置。这叫 "bare workflow"，但现在 Expo 更推荐用 Config Plugins 来避免这种情况。

### Expo 的性能够用吗？

SDK 52 开始默认启用 New Architecture（JSI + Fabric + TurboModules），JavaScript 和原生层之间的通信不再经过 JSON 序列化的 Bridge，而是直接通过 C++ 接口调用。性能已经非常接近纯原生。

对于绝大多数 App（社交、电商、工具、内容类），Expo 的性能完全够用。只有极少数场景（高性能游戏、复杂视频编辑）才需要考虑纯原生。

## 九、完整流程总结

```
1. npx create-expo-app@latest my-app     # 创建项目
2. npx expo start                         # 启动开发，Expo Go 快速预览
3. 写代码，添加页面和功能
4. eas build:configure                    # 初始化 EAS
5. eas build --platform ios --profile development  # 构建开发版
6. 在真机上安装开发版，继续开发调试
7. 完善 app.json 配置（图标、权限、bundleIdentifier）
8. eas build --platform ios --profile production   # 构建生产版
9. App Store Connect 上创建 App 记录，填写元信息
10. eas submit --platform ios              # 提交到 App Store
11. TestFlight 测试 → 提交审核 → 发布
12. 后续 JS 更新用 eas update，原生更新重新走 8-11
```

从写第一行代码到 App 上架，Expo 把这个流程压缩到了最短。你不需要理解 Xcode 的签名机制，不需要手动管理证书，不需要跟 CocoaPods 斗智斗勇。专注于产品本身就好。

---

*最后更新：2026-04-21*

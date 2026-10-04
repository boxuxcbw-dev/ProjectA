# Project A

SwiftUI 多平台 App（iPhone / iPad / visionOS / macOS）。Xcode 27.0。

| 项 | 值 |
|---|---|
| 项目文件 | `Project A.xcodeproj` |
| Target / Scheme | `Project A`（**含空格，命令里必须加引号**） |
| Bundle ID | `DuoDuoYS.Project-A` |
| iOS 部署目标 | 27.0 |
| 版本 | 1.0（build 1） |
| Swift | 语言模式 5，编译器 6.4 |

## 构建 / 测试 / 运行

**构建（iOS 模拟器）**

```bash
xcodebuild -project "Project A.xcodeproj" -scheme "Project A" \
  -destination 'platform=iOS Simulator,name=iPhone 17' build
```

**跑测试**

```bash
xcodebuild test -project "Project A.xcodeproj" -scheme "Project A" \
  -destination 'platform=iOS Simulator,name=iPhone 17'
```

**macOS**（该 target 也支持，调试 UI 很快）

```bash
xcodebuild -project "Project A.xcodeproj" -scheme "Project A" \
  -destination 'platform=macOS' build
```

**装到模拟器并启动**

```bash
xcrun simctl boot "iPhone 17"      # 若未启动
open -a Simulator

# 查出构建产物路径（BUILT_PRODUCTS_DIR + FULL_PRODUCT_NAME）
xcodebuild -project "Project A.xcodeproj" -scheme "Project A" \
  -destination 'platform=iOS Simulator,name=iPhone 17' -showBuildSettings 2>/dev/null \
  | grep -E ' (BUILT_PRODUCTS_DIR|FULL_PRODUCT_NAME) ='

xcrun simctl install booted "<BUILT_PRODUCTS_DIR>/<FULL_PRODUCT_NAME>"
xcrun simctl launch booted DuoDuoYS.Project-A
```

可用模拟器用 `xcrun simctl list devices available` 查（本机有 iPhone 18 Pro / 18 Pro Max / 17 / 17e / Air，以及各型号 iPad）。

## 约定

- **不要手改 `Project A.xcodeproj/project.pbxproj`**。它是易冲突的文本格式，改坏会导致 Xcode 打不开项目。需要新增 / 删除 / 移动文件时，先说明意图，由我在 Xcode 界面里操作。
- `xcuserdata/`、`DerivedData/`、`.DS_Store` 已在 `.gitignore` 中忽略 —— 不要提交它们，也不要删掉这些忽略规则。
- 提交信息用英文、动词开头（例如 `Add dark mode toggle`）。
- 新代码优先使用 Swift Concurrency（async/await），不要引入 Combine。
- 用中文跟我交流。

## 仓库

- 默认分支：`main`
- 远端：待创建（`boxuxcbw-dev/ProjectA`），创建后补上 URL

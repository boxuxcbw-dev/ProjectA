# 开发环境搭建记录

> 本文档**不含任何密钥**，可安全提交到仓库、跨机器查看。
> 目的：把「Claude Code + DeepSeek + VS Code + Xcode + Git/GitHub」这套环境固化下来，便于在第二台 Mac 上复现，并留下关键决策与踩坑记录。

来源机器：Mac mini（macOS 26.6.2，Apple Silicon）
记录时间：2026-10-04

---

## 一、这套环境由什么组成

| 层 | 组件 | 位置 / 版本 |
|---|---|---|
| 运行时 | Node.js v24.21.0 LTS（官方 .pkg，**通用二进制**） | `/usr/local/bin/node` |
| AI CLI | Claude Code 2.1.288（**官方原生安装**，非 npm） | `~/.local/bin/claude` |
| 模式切换 | 自定义 zsh 函数库 | `~/.config/claude-code/claude.zsh` |
| 密钥与接口 | DeepSeek API key + 变量（600 权限） | `~/.config/claude-code/deepseek.env` |
| 当前模式 | `deepseek` / `claude` | `~/.config/claude-code/mode` |
| IDE | VS Code 1.140.0 + Claude Code 扩展 2.1.288 | `/Applications` |
| IDE 双模式 | VS Code **Profiles**：Default = DeepSeek，Claude 官方 = 订阅 | `.../Code/User/settings.json` + `profiles/-2e177217/` |
| 移动端 | Xcode 27.0 + iOS 27 SDK + 模拟器运行时（11 个设备） | `/Applications` |
| 版本控制 | git 2.54.0 + SSH ed25519 + GitHub | `~/.ssh`、`~/.gitconfig` |

关键路径速查：

```
~/.config/claude-code/deepseek.env    DeepSeek key 与接口变量
~/.config/claude-code/claude.zsh      模式切换 + 项目识别 + 包装函数
~/.config/claude-code/mode            当前模式（内容就是 deepseek 或 claude）
~/.local/bin/claude                   Claude Code 启动器（符号链接）
~/.local/bin/code                     VS Code CLI（符号链接）
~/.ssh/{id_ed25519,id_ed25519.pub,config}
~/.claude.json                        Claude Code 配置 + 目录信任 + OAuth 凭据
```

---

## 二、关键设计决策与理由

1. **Claude Code 用官方原生安装，不用 npm。**
   全局 npm 目录 `/usr/local/lib/node_modules` 属 `root:wheel`，装全局包需要 root，而本环境无法非交互使用 `sudo`。原生安装免 root、原生 arm64、自带更新。
2. **DeepSeek 变量不常驻 shell，改由包装函数按「模式」在启动瞬间注入。**
   好处：同一终端可随时切换、不污染其他工具的 `ANTHROPIC_*` 环境（实测 `env | grep -c '^ANTHROPIC_'` = 0）。
3. **模式用文件持久化，而非环境变量。** 跨终端一致，`claude-mode toggle` 即可。
4. **项目目录自动识别**：从 `$PWD` 向上查找
   `.git` / `.claude` / `.mcp.json` / `*.xcodeproj` / `*.xcworkspace` / `Package.swift`；
   都不满足则落到 `$CLAUDE_DEFAULT_PROJECT`。
   加 Apple 标记是为了让 Xcode 项目**不依赖 git 也能被识别**。
5. **VS Code 内切换用 Profiles，而非 `claudeProcessWrapper`。**
   Profile 的 `claudeCode.environmentVariables` 是 `machine` scope（可 per-profile），而 `application` scope 的设置才是跨 profile 共享的。
   注意：profile 注册存在 `storage.json` 里，**不能跨机器拷贝**，新机器必须重建。
6. **备份用加密 `.dmg`，而不是加密卷。**
   与文件系统无关；外部卷（ExFAT / HFS+）通常 `Owners: Disabled`、不保留 POSIX 权限，而**磁盘映像内部能保留**。

---

## 三、踩过的坑（均已在当前配置里修掉）

| 症状 | 根因 | 处理 |
|---|---|---|
| 家目录启动 `claude` 仍弹目录信任 | 项目识别把 `~/.claude` 当成项目标记，误判家目录为项目 | 排除 `$HOME` 的 `.claude` 检查 |
| CLI 建的 VS Code profile 在 UI 里看不到 | profile 注册写在 `storage.json`，而运行中的实例用的是启动时的内存副本 | **完全退出** VS Code 再打开 |
| `/status` 显示 Claude 而不是 `deepseek-flash` | `/status` 展示的是**账号**，而 claude.ai 订阅登录仍在；且 `deepseek-flash` 不在内置模型目录 | 判断实际后端要查会话记录的 `model` 字段，**不要信 `/status`** |
| `connectors are disabled` 提示 | 设置了 `ANTHROPIC_AUTH_TOKEN` 时，claude.ai connectors 按设计不加载 | 属预期行为；切到订阅 profile 即恢复 |
| 备份从 191 KB 膨胀到 283 MB | ExFAT 128 KB 簇 + 数千个技能小文件，放大 ~36 倍 | 精简备份（排除 `plugins`/`skills`） |
| 备份脚本报「找不到 AI 文件夹」 | 脚本要求用户手动建的 `AI` 目录预先存在 | 改为 `mkdir -p` 自动创建 |
| 密钥被提交进仓库 | 项目没有 `.gitignore` | 补 Xcode 专用 `.gitignore` + `git rm --cached xcuserdata` |
| 后台任务 `sudo` 一律失败 | 沙箱禁止执行 `sudo` | 需要 root 的步骤（改分区、装 .pkg）交给用户本人操作 |

---

## 四、可复用的验证方法

1. **判断 Claude Code 实际走哪个后端**：读会话记录
   `~/.claude/projects/<路径编码>/<uuid>.jsonl` 里的 `"model"` 字段。
2. **区分命令行还是 IDE 面板**：同一记录里的 `entrypoint` 字段（`cli` / `sdk-cli`）。
3. **测试目录识别逻辑**：用一个「打印 `$PWD` 的假 claude」放在 `PATH` 最前当桩程序，避免真的启动 Claude Code 并写会话。
4. **验证备份**：`shasum -a 256 -c manifest.sha256` → 对**私钥**单独做逐字节比对 → **实跑 `restore.sh` 到临时目录**并检查权限。
   没实跑过的恢复脚本等于没有。
5. **校验 GitHub 主机密钥**：把 `ssh-keyscan` 结果与 `https://api.github.com/meta` 公布的指纹比对（RSA + ED25519 都要对）。
6. **验证克隆/推送归属**：`https://api.github.com/repos/<owner>/<repo>/commits` 返回的 `author.login` 非 null，说明提交邮箱已在 GitHub 验证。

---

## 五、日常操作速查

```bash
claude                     # 按当前模式启动（项目内原地，否则默认项目）
claude-mode                # 查看当前模式
claude-mode toggle         # DeepSeek ↔ Claude 官方
claude-deepseek            # 本次用 DeepSeek（不改默认模式）
claude-official            # 本次用 Claude 订阅
claude-pick                # 列出 ~/AI 下的项目，选一个启动
cdp <项目名>               # 切到某个项目
```

构建 / 运行（项目名含空格，注意引号）：

```bash
cd ~/AI/Project_A
xcodebuild -project "Project A.xcodeproj" -scheme "Project A" \
  -destination 'platform=iOS Simulator,name=iPhone 17' build
xcrun simctl boot "iPhone 17" && open -a Simulator
```

---

## 六、已知遗留（都可选）

- VS Code 存储里有一条指向**已删除目录** `profile-scratch` 的关联记录（无害）
- `make-backup.sh` 控制台输出有一句过时的 “ExFAT” 文案（实际是 HFS+）
- **FileVault 未开启**（Mac mini 仅在家使用，风险较低；新机器建议开）
- DeepSeek key 存在于两处：`deepseek.env` 与 VS Code **Default profile** 的 settings
  → 轮换时要改两处，漏一处会出现「一边能用一边 401」
- SSH 私钥**无口令**（`ssh-keygen -p -f ~/.ssh/id_ed25519` 可补）

---

## 七、本次对话的记录在哪

- 本机路径：`~/.dsh/sessions/--Users-boxu-Documents-deepseek-harness-default-workspace--/session-7db2d526-…/session.v4.jsonl.zstd`（约 1.9 MB，zstd 压缩）
- **只存在本机**，不会自动同步到新 Mac
- ⚠️ 其中包含聊天里粘贴过的 API key，**不要同步到云盘、不要提交到仓库**
- 真正需要移交给新机器的内容在 U 盘上：`START-HERE.md`（步骤清单）+ 加密 `.dmg`（密钥与配置）

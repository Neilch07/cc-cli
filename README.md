# cc-cli

**Claude Code 多账号/API Key 快速切换工具**

在终端一条命令切换 Claude Code 的 API Key、中转 API 或 OAuth 账号，不用手动改配置文件。

## 功能亮点

- **多 Profile 管理** — 同时存储多个 API Key / OAuth 账号，随时切换
- **OAuth 凭证缓存** — OAuth 账号登录一次后自动缓存，切换时无需重新登录
- **多 OAuth 账号共存** — 支持缓存多个 OAuth 账号，与 API Key 无缝交替使用
- **第三方中转支持** — 支持 `--url` 指定自定义 Base URL（如 runapi、openrouter 等）
- **一键连通测试** — `cc test` 验证 API Key 是否有效
- **临时切换执行** — `cc exec` 在不改全局状态下临时使用某个 profile 执行命令
- **标签分组** — `--tag` 给 profile 打标签，`cc list --tag` 按标签过滤
- **fzf 交互选择** — `cc use` 无参数时弹出交互式选择器（需安装 fzf）
- **Tab 补全** — 子命令和 profile 名称自动补全
- **导入导出** — `cc export` / `cc import-file` 跨机器迁移配置
- **并发安全** — 文件锁防止多终端同时写配置
- **Key 脱敏显示** — list / show 只显示 key 头尾，不泄露完整密钥

## 平台支持

| 平台 | Shell | 状态 |
|------|-------|------|
| Linux | bash | 完整支持 |
| Linux | zsh | 完整支持 |
| macOS | zsh（默认） | 完整支持 |
| macOS | bash | 完整支持（需 brew install bash） |

## 依赖

| 依赖 | 必选 | 安装方式 |
|------|------|----------|
| bash >= 4.0 | 是 | 系统自带 / `brew install bash`（macOS 自带版本过低） |
| jq | 是 | `sudo apt install jq` / `brew install jq` |
| curl | 是 | `sudo apt install curl` / `brew install curl` |
| fzf | 否 | `sudo apt install fzf` / `brew install fzf`（用于交互选择） |

> **macOS 用户注意**：macOS 自带的 bash 版本为 3.2（因 GPLv3 许可证原因），cc-cli 需要 bash >= 4.0。请通过 Homebrew 安装新版 bash：`brew install bash`。安装后 `/usr/local/bin/bash`（Intel）或 `/opt/homebrew/bin/bash`（Apple Silicon）会被自动使用。

## 安装

### 方式一：一键安装（推荐）

```bash
git clone https://github.com/Evan-Huang-yf/cc-cli.git
cd cc-cli
bash install.sh
```

如果你的登录 shell 是 zsh 但想安装到 bash（或反过来），可以用 `--shell` 指定：

```bash
bash install.sh --shell bash    # 强制安装到 bash
bash install.sh --shell zsh     # 强制安装到 zsh
```

安装脚本会自动检测你的默认 shell（bash/zsh），执行：
1. 检测依赖是否满足
2. 复制 `cc` 到 `~/.local/bin/`
3. 复制对应的补全脚本（bash → `~/.bash_completion.d/`，zsh → `~/.zsh_completion.d/`）
4. 在 `~/.bashrc` 或 `~/.zshrc` 中添加 wrapper 函数
5. 初始化 `~/.cc-profiles/` 数据目录

安装完成后：
```bash
# bash 用户
source ~/.bashrc

# zsh 用户（macOS 默认）
source ~/.zshrc
```

### 方式二：手动安装（bash）

```bash
# 1. 复制主脚本
cp cc ~/.local/bin/cc
chmod +x ~/.local/bin/cc

# 2. 复制补全脚本
mkdir -p ~/.bash_completion.d
cp completions/cc.bash ~/.bash_completion.d/cc

# 3. 确保 ~/.local/bin 在 PATH 中
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc

# 4. 在 ~/.bashrc 末尾添加以下内容
cat >> ~/.bashrc << 'EOF'

# cc-cli: Claude Code 账号切换工具
[ -f "$HOME/.cc-profiles/env.sh" ] && source "$HOME/.cc-profiles/env.sh"
[ -f "$HOME/.bash_completion.d/cc" ] && source "$HOME/.bash_completion.d/cc"
cc() {
    command cc "$@"
    local ret=$?
    [ -f "$HOME/.cc-profiles/env.sh" ] && source "$HOME/.cc-profiles/env.sh"
    if [ "${1:-}" = "use" ] || [ "${1:-}" = "switch" ]; then
        claude
    fi
    return $ret
}
EOF

# 5. 生效
source ~/.bashrc
```

### 方式三：手动安装（zsh / macOS）

```bash
# 1. 复制主脚本
cp cc ~/.local/bin/cc
chmod +x ~/.local/bin/cc

# 2. 复制 zsh 补全脚本
mkdir -p ~/.zsh_completion.d
cp completions/cc.zsh ~/.zsh_completion.d/_cc

# 3. 确保 ~/.local/bin 在 PATH 中
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zshrc

# 4. 在 ~/.zshrc 末尾添加以下内容
cat >> ~/.zshrc << 'EOF'

# cc-cli: Claude Code 账号切换工具
[ -f "$HOME/.cc-profiles/env.sh" ] && source "$HOME/.cc-profiles/env.sh"
fpath=($HOME/.zsh_completion.d $fpath)
autoload -Uz compinit && compinit -C
cc() {
    command cc "$@"
    local ret=$?
    [ -f "$HOME/.cc-profiles/env.sh" ] && source "$HOME/.cc-profiles/env.sh"
    if [ "${1:-}" = "use" ] || [ "${1:-}" = "switch" ]; then
        claude
    fi
    return $ret
}
EOF

# 5. 生效
source ~/.zshrc
```

## 快速上手

```bash
# 添加一个官方 API Key
cc add work --key sk-ant-api03-xxxxxxxxxxxx

# 添加一个第三方中转 API
cc add runapi --key sk-xxx --url https://runapi.co --tag dev

# 添加 OAuth 官方账号
cc add personal --oauth

# 添加第二个 OAuth 账号（如团队账号）
cc add team --oauth --tag work

# 查看所有 profile
cc list

# 切换到 work
cc use work

# 测试连通性
cc test work

# OAuth 和 API Key 无缝交替（首次 OAuth 需登录，之后自动缓存）
cc use personal    # OAuth，首次需浏览器登录
cc use runapi      # 切到 API Key，OAuth 凭证自动保存
cc use personal    # 再切回 OAuth，从缓存恢复，无需重新登录
cc use team        # 切到另一个 OAuth 账号，同样免登录

# 强制刷新 OAuth 登录（token 过期时使用）
cc login personal
```

## 完整命令参考

### `cc list [--tag <TAG>]`

列出所有 profile。当前激活的用 `▶` 标记。

```bash
cc list             # 列出全部
cc list --tag dev   # 只显示标签为 dev 的 profile
```

输出示例：
```
Profile 列表:

       名称               类型     标签     标识
  ────────────────────────────────────────────────────────────────────────
    official             oauth      -          OAuth 登录
  ▶ runapi               api_key    dev        sk-ant***m3Kx @ https://api.example.com
    work                 api_key    prod       sk-ant***-xxx
```

### `cc current`

查看当前激活的 profile 详细信息。

### `cc add <名称> [选项]`

添加新 profile。

| 选项 | 说明 |
|------|------|
| `--key <KEY>` | API Key（与 --oauth 二选一） |
| `--oauth` | OAuth 认证（与 --key 二选一） |
| `--url <URL>` | 自定义 Base URL（第三方中转 API） |
| `--tag <TAG>` | 标签（可选） |

```bash
cc add my-key --key sk-ant-api03-xxx
cc add proxy --key sk-xxx --url https://example.com --tag test
cc add personal --oauth --tag personal
```

### `cc use [名称]`

切换到指定 profile。

- **有参数**：直接切换到指定 profile
- **无参数**：如果安装了 fzf，弹出交互式选择器；否则提示手动指定

```bash
cc use work     # 直接切换
cc use          # fzf 交互选择
```

切换 API Key 类型 profile 时，工具会自动：
1. 更新 `~/.claude/config.json`（API Key）
2. 更新 `~/.claude/settings.json` 的 `env` 字段（Base URL，让 IDE 侧边栏也能读到）
3. 更新终端环境变量 `ANTHROPIC_API_KEY` / `ANTHROPIC_BASE_URL`
4. 自动启动 `claude`

切换 OAuth 类型时：
- **有缓存凭证** → 直接恢复，无需重新登录（秒切）
- **无缓存** → 执行 `claude /login` 流程，登录后自动缓存凭证

### `cc login [名称]`

强制刷新 OAuth 登录并更新凭证缓存。当 token 过期或需要切换到不同的 claude.ai 账号时使用。

```bash
cc login personal   # 强制重新登录指定 OAuth profile
cc login            # 强制重新登录当前激活的 OAuth profile
```

### `cc edit <名称> [选项]`

就地修改 profile 属性，不用先删再加。

```bash
cc edit work --key sk-new-key-xxx      # 更新 key
cc edit work --url https://new-api.com # 更新 URL
cc edit work --url ""                  # 清除 URL
cc edit work --tag production          # 更新标签
```

如果修改的是当前激活的 profile，会自动重新生效。

### `cc test [名称]`

测试 profile 的 API 连通性。无参数时测试当前激活的 profile。

```bash
cc test work    # 测试指定 profile
cc test         # 测试当前激活的
```

输出示例：
```
测试 profile: work
  Endpoint: https://api.anthropic.com/v1/messages
  ✓ 连接成功 (HTTP 200)
  模型响应: claude-haiku-4-5-20251001
```

### `cc exec <名称> -- <命令>`

临时使用某个 profile 执行命令，**不改变全局状态**。适合一次性操作或脚本中使用。

```bash
# 用 work profile 的环境跑一个命令
cc exec work -- env | grep ANTHROPIC

# 用 work profile 临时启动 claude
cc exec work -- claude "你好"
```

### `cc rm <名称>`

删除 profile。如果删除的是当前激活的，会提示切换到其他 profile。

### `cc rename <旧名> <新名>`

重命名 profile。

### `cc show <名称>`

查看 profile 的完整详情（类型、创建时间、标签、状态等）。

### `cc export [--file <路径>]`

导出所有 profiles。

```bash
cc export                        # 输出到 stdout（JSON 格式）
cc export --file backup.json     # 输出到文件
```

### `cc import-file <路径>`

从文件导入 profiles。合并模式，同名 profile 不会被覆盖。

```bash
cc import-file backup.json
```

### `cc backup`

备份当前配置到 `~/.cc-profiles/backups/`，自动保留最近 10 份。

### `cc update`

从 GitHub 拉取最新版本并自动更新本地安装的 `cc` 脚本和补全脚本。不影响已有的 profiles 数据。

```bash
cc update
```

### `cc help`

显示帮助信息。

## 更新

```bash
cc update
# bash 用户
source ~/.bashrc
# zsh 用户
source ~/.zshrc
```

一条命令即可更新到最新版本。你的 profile 数据（`~/.cc-profiles/`）不会受影响。

## 配置文件说明

cc-cli 使用以下文件：

```
~/.cc-profiles/
├── profiles.json          # 所有 profile 数据（JSON 格式）
├── env.sh                 # 当前激活 profile 的环境变量（自动生成）
├── oauth-credentials/     # OAuth 凭证缓存目录（每个 OAuth profile 一个文件）
│   ├── personal.credentials.json
│   └── team.credentials.json
├── .lock_dir/             # 目录锁（并发控制，自动管理）
├── .config_backup.json    # 上次切换前的 config.json 备份
└── backups/               # cc backup 的备份目录

~/.claude/
├── config.json            # cc 写入 primaryApiKey（Claude Code 读取）
└── settings.json          # cc 写入 env.ANTHROPIC_BASE_URL（IDE 侧边栏读取）

# bash 用户
~/.bash_completion.d/cc    # bash 补全脚本

# zsh 用户
~/.zsh_completion.d/_cc    # zsh 补全脚本
```

### profiles.json 结构

```json
{
  "profiles": {
    "work": {
      "type": "api_key",
      "key": "sk-ant-api03-xxxxxxxxxxxx",
      "tag": "production",
      "created": "2026-03-29T07:00:00Z"
    },
    "proxy": {
      "type": "api_key",
      "key": "sk-xxxxxxxxxxxx",
      "url": "https://example.com",
      "tag": "dev",
      "created": "2026-03-29T08:00:00Z"
    },
    "personal": {
      "type": "oauth",
      "created": "2026-03-29T09:00:00Z"
    }
  },
  "active": "work"
}
```

### 环境变量

切换 profile 后，以下环境变量会自动设置：

| 环境变量 | 说明 |
|----------|------|
| `ANTHROPIC_API_KEY` | API Key（API Key 类型 profile） |
| `ANTHROPIC_BASE_URL` | 自定义 Base URL（仅当 profile 配置了 --url 时） |

## Wrapper 函数说明

`~/.bashrc` 或 `~/.zshrc` 中的 wrapper 函数是必要的，因为：

1. **环境变量生效**：脚本是子进程，无法修改父 shell 的环境变量。wrapper 在当前 shell 中 `source env.sh`，让环境变量生效。
2. **自动启动 claude**：切换 profile 后自动启动 Claude Code CLI。

如果你不需要自动启动 claude，可以删掉 wrapper 中的 `claude` 那一行。

## 命令别名

大部分命令支持简写：

| 简写 | 完整命令 |
|------|----------|
| `cc ls` | `cc list` |
| `cc cur` | `cc current` |
| `cc del` | `cc rm` |
| `cc mv` | `cc rename` |
| `cc info` | `cc show` |

## FAQ

### Q: 支持 zsh 吗？

完整支持。安装脚本会自动检测你的默认 shell。macOS 默认使用 zsh，安装时会自动：
- 写入 `~/.zshrc`（而非 `~/.bashrc`）
- 安装 zsh 原生补全脚本（compdef）
- 配置 fpath 和 compinit

### Q: API Key 安全吗？

当前版本 API Key 以明文存储在 `~/.cc-profiles/profiles.json` 中。建议：
- 确保该文件权限为 `600`（`chmod 600 ~/.cc-profiles/profiles.json`）
- 不要将 `~/.cc-profiles/` 目录上传到任何公开仓库
- 后续版本计划接入系统 keyring 加密存储

### Q: 两个终端同时切换会冲突吗？

不会。cc-cli 使用目录锁（`mkdir` 原子操作），写操作（add、use、edit、rm、rename）互斥执行。该机制在 Linux 和 macOS 上均可用。

### Q: cc 和系统的 C 编译器 cc 冲突怎么办？

安装后 `cc` 会覆盖系统的 C 编译器命令。如果你需要使用 C 编译器，可以：
- 直接使用 `gcc` 或 `clang`
- 使用完整路径 `/usr/bin/cc`
- 或者将工具重命名为其他名称（如 `ccs`），修改 `~/.local/bin/` 中的文件名和 `.bashrc` 中的 wrapper 函数名即可

### Q: OAuth 账号切换需要每次重新登录吗？

不需要。cc-cli 会自动缓存 OAuth 凭证（refresh token）。首次添加 OAuth profile 并登录后，凭证会保存到 `~/.cc-profiles/oauth-credentials/` 目录。之后在 OAuth 和 API Key 之间来回切换时，OAuth 凭证会自动恢复，无需重新打开浏览器登录。

如果 token 过期或需要切换到不同的 claude.ai 账号，使用 `cc login <名称>` 强制刷新。

### Q: 可以同时缓存多个 OAuth 账号吗？

可以。每个 OAuth profile 的凭证独立缓存。例如你可以有 `personal`（个人 Pro 账号）和 `team`（团队账号），两者互不干扰，切换时都无需重新登录。

## 卸载

```bash
cd cc-cli
bash uninstall.sh
```

或手动卸载：

```bash
rm ~/.local/bin/cc
rm ~/.bash_completion.d/cc 2>/dev/null    # bash 用户
rm ~/.zsh_completion.d/_cc 2>/dev/null    # zsh 用户
# 编辑 ~/.bashrc 或 ~/.zshrc，删除 cc-cli 相关区块（搜索 "cc-cli" 关键字）
# 可选：rm -rf ~/.cc-profiles
```

## 协议

MIT License

## 姊妹工具：cx — Codex 多账号轮换

与 `cc` 同一思路的 Codex CLI 版：**一个账号 = 一个独立 `CODEX_HOME` 目录**（`~/.codex-profiles/<名>/`），多个订阅账号共存互不覆盖，并支持**触限自动换号**。

```bash
ln -s "$(pwd)/cx" ~/.local/bin/cx   # 安装（与 cc 并列）

cx add work        # 新建 profile 并交互登录（可反复 add 多个账号）
cx list            # 列出各账号登录态与上次可用账号
cx exec work -- exec --sandbox read-only "..."   # 指定账号执行 codex
cx auto -- exec "..."   # 自动轮换：从上次可用账号开始，烧到 usage limit 自动换下一个
```

`auto` 模式的行为：命中 usage limit / 429 / quota 类错误 → 换下一个已登录账号重试（stdin 已缓存，heredoc/管道输入不会丢）；非触限的真实错误 → 如实透出并停止，不空烧其它账号；成功的账号记入 `~/.codex-profiles/.last_good`，下次优先。

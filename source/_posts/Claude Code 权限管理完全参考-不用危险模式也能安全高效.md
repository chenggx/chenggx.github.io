---
title: Claude Code 权限管理完全参考：不用危险模式也能安全高效
date: 2026-05-19
tags:
  - Claude Code
  - 权限管理
  - 安全
  - 沙箱
  - Hooks
categories:
  - AI工具
description: Claude Code的权限管理是分层防御体系，不使用危险模式也能安全高效工作。本文介绍三道防线（权限规则、权限模式、沙箱隔离）的配置方法和实战模板。
---

# Claude Code 权限管理完全参考：不用危险模式也能安全高效

Claude Code 能读取代码、执行命令、修改文件，甚至代你提交代码。这种能力带来一个现实问题：要么每次操作都手动确认，效率低下；要么使用 `--dangerously-skip-permissions` 完全放开权限，一旦 AI 误判就可能导致核心文件被删、敏感配置泄露，甚至执行危险命令。

这篇文章的目标是：**在不触碰危险模式的前提下，建立一套分层、精细、可落地的 Claude Code 权限管理体系**。读完之后，你应该能够根据自己的场景配置出既不影响效率、又能有效防止核心内容被误删的安全边界。

![Claude Code 权限体系全景图：三道防线 + 四层配置](https://static.xiangdangnian.net.cn/blog/01-framework-permission-system-overview.png?v=20260519)

---

## 权限体系全景图：三道防线 + 四层配置

Claude Code 的权限系统不是单一开关，而是一个分层的防御体系。

### 三道防线

| 防线 | 机制 | 作用 |
|------|------|------|
| **第一层** | Permission Rules（权限规则） | 决定 Claude 能调用什么工具、访问什么文件、执行什么命令 |
| **第二层** | Permission Modes（权限模式） | 决定 Claude 在运行时以什么策略请求或自动批准操作 |
| **第三层** | Sandboxing（沙箱隔离） | 在操作系统层面限制 Bash 命令的文件系统和网络访问范围 |

权限规则控制 Claude "想做什么"，权限模式控制"要不要问人"，沙箱控制"实际能做到什么"。三层互补，不是替代。

### 四层配置作用域（Scopes）

Claude Code 的配置按优先级从高到低分为五层：

1. **Managed（托管策略）**：由 IT/管理员通过 MDM、注册表或服务器下发，**不可被任何层级覆盖**。适合企业强制安全基线。
2. **命令行参数**：`--settings`、`--permission-mode` 等，仅对当前会话生效。
3. **Local（本地设置）**：`.claude/settings.local.json`，仅影响当前仓库的当前用户，**不进入 Git**。
4. **Project（项目设置）**：`.claude/settings.json`，团队共享，**进入 Git**。
5. **User（用户设置）**：`~/.claude/settings.json`，跨所有项目生效。

关键规则：
- **标量值**（如字符串、布尔值）：高优先级覆盖低优先级。
- **数组值**（如权限规则列表）：**合并并去重**，而不是覆盖。这意味着 managed settings 可以添加 `deny` 规则，用户和项目也可以继续追加自己的规则，所有人的规则共同生效。
- **deny 规则一旦命中，任何层级的 allow 都无法推翻**。

---

## 第一道防线：精细化权限规则

权限规则是最直接、最细粒度的控制手段。规则写在 **任意 `settings.json` 的 `permissions` 对象中**——包括用户级 `~/.claude/settings.json`、项目级 `.claude/settings.json`、本地覆盖 `.claude/settings.local.json`，或管理员下发的 `managed-settings.json`。规则在不同层级中按 deny-first 原则合并生效，分为三类：

- `allow`：匹配到的操作**无需确认**，直接执行。
- `ask`：匹配到的操作**每次都弹窗确认**。
- `deny`：匹配到的操作**直接被拒绝**。

评估顺序是固定的：**deny → ask → allow**，第一个匹配的规则生效。这意味着 `deny` 始终拥有最高优先级。

### 规则语法基础

规则格式为 `Tool` 或 `Tool(specifier)`：

```json
{
  "permissions": {
    "allow": [
      "Bash(npm run *)",
      "Bash(git status)",
      "Read(~/.zshrc)"
    ],
    "ask": [
      "Bash(git push *)"
    ],
    "deny": [
      "Bash(curl *)",
      "Bash(wget *)",
      "Read(./.env)",
      "Read(./.env.*)",
      "Read(./secrets/**)",
      "Read(//etc/**)"
    ]
  }
}
```

### 保护核心文件：Read/Edit 规则

文件权限规则遵循 **gitignore 规范**，但有四个锚点前缀需要特别注意：

| 前缀 | 含义 | 示例 |
|------|------|------|
| `//path` | 绝对路径（从文件系统根开始） | `Read(//Users/alice/secrets/**)` |
| `~/path` | 相对于用户主目录 | `Read(~/.aws/credentials)` |
| `/path` | 相对于**项目根目录** | `Edit(/src/**/*.ts)` |
| `path` 或 `./path` | 相对于当前工作目录 | `Read(./.env)` |

**常见陷阱**：`Read(/Users/alice/file)` 不是绝对路径，而是相对于项目根目录的 `Users/alice/file`。绝对路径必须用双斜杠 `//`。

**最佳实践：保护敏感文件**

```json
{
  "permissions": {
    "deny": [
      "Read(./.env)",
      "Read(./.env.*)",
      "Read(./secrets/**)",
      "Read(./config/credentials.json)",
      "Read(~/.ssh/**)",
      "Read(~/.aws/**)",
      "Read(//etc/**)",
      "Edit(./.git/**)"
    ]
  }
}
```

这些规则会阻止 Claude 读取环境变量、SSH 密钥、AWS 凭证、系统配置文件，以及直接修改 Git 内部目录。

### 控制命令执行：Bash 规则

Bash 规则支持 glob 通配符 `*`，且 `*` 可以匹配空格分隔的多个参数：

```json
{
  "permissions": {
    "allow": [
      "Bash(npm run *)",
      "Bash(git commit *)",
      "Bash(git * main)",
      "Bash(* --version)"
    ],
    "deny": [
      "Bash(curl *)",
      "Bash(wget *)",
      "Bash(rm -rf *)"
    ]
  }
}
```

**重要细节**：
- `Bash(ls *)`（带空格）匹配 `ls -la`，但不匹配 `lsof`。
- `Bash(ls*)`（不带空格）匹配 `ls -la` 和 `lsof`。
- 复合命令（`&&`、`||`、`;`、`|`）会被拆分为子命令分别检查规则。
- 预设的进程包装器（`timeout`、`time`、`nice`、`nohup`、`stdbuf`、`xargs`）会被自动剥离后再匹配。

**Bash 规则的根本缺陷**：

Bash 规则匹配的是**命令行字符串的表面形式**，而不是解析后的实际参数。这意味着你写的规则很容易被 Shell 语法绕过。

举个例子，你写下 `Bash(curl http://github.com/*)`，想限制 curl 只能访问 GitHub。实际上这个规则基本没用：

| 绕过方式 | 命令示例 | 为什么规则拦不住 |
|---------|---------|----------------|
| 选项插在 URL 前面 | `curl -X GET http://github.com/...` | 规则匹配以 `curl http://github.com/` 开头，`-X` 插在前面就不匹配了 |
| 用变量替代硬编码 URL | `URL=http://github.com && curl $URL` | 规则看到的是 `curl $URL`，不是 `curl http://github.com`，直接放行 |
| 用短链接重定向 | `curl -L http://bit.ly/xyz` | 规则匹配 `http://bit.ly`，但实际跳转到了任意目标 |
| 管道接力 | `echo "http://github.com" \| xargs curl` | xargs 把 URL 传给 curl，命令行里根本看不到完整 URL |
| 命令替换 | `curl $(cat url.txt)` | 规则看到的是 `curl $(cat ...)`，具体 URL 在运行时才能确定 |

**一句话总结**：Bash 规则只能控制"命令行以什么字符串开头"，控制不了"命令执行时实际访问了什么"。

**正确做法**：
- 如果目的是限制网络访问，**直接 deny 掉 `curl`、`wget`、`nc` 等工具**，改用 `WebFetch(domain:github.com)` 进行受控的网络请求。
- 如果必须允许 Bash 网络工具，用 **PreToolUse Hook** 在运行时解析命令内容并拦截，或者启用**沙箱网络隔离**在 OS 层面限制域名。

### Symlink（符号链接）的特殊处理

**Symlink 是什么？**

Symlink（符号链接，也叫软链接）类似于 Windows 的快捷方式。它是一个特殊的文件，本身不包含数据，只保存了指向另一个文件或目录的路径。比如：

```bash
ln -s ~/.ssh/id_rsa ./my-link
```

这会在当前目录创建一个名为 `my-link` 的文件，打开它实际上打开的是 `~/.ssh/id_rsa`。

**为什么权限管理要特别处理它？**

因为 symlink 可以被用来**绕过访问限制**。假设你的配置是这样的：

```json
{
  "permissions": {
    "allow": ["Read(./project/**)"],
    "deny": ["Read(~/.ssh/**)"]
  }
}
```

这个配置允许 Claude 读取 `./project/` 目录下的所有文件，但禁止读取 `~/.ssh/` 目录（里面存放着你的 SSH 私钥）。

如果有人（或 Claude 自己）在项目目录里创建了一个 symlink：

```bash
cd ./project
ln -s ~/.ssh/id_rsa ./stolen-key
```

那么 `./project/stolen-key` 这个路径是在允许范围内的，但它指向的却是被禁止的 `~/.ssh/id_rsa`。如果不做特殊处理，Claude 读取 `./project/stolen-key` 就等于绕过了 `deny` 规则，泄露了你的私钥。

**Claude Code 的处理方式**

权限规则在评估 symlink 时会**同时检查两条路径**：symlink 本身所在的路径，以及它最终指向的真实文件路径。

| 规则类型 | 评估逻辑 | 效果 |
|---------|---------|------|
| **Allow 规则** | symlink 路径 **和** 真实目标路径 **都** 匹配 allow 规则，才放行 | `./project/stolen-key` 在 allow 范围内，但 `~/.ssh/id_rsa` 不在 → **仍然弹窗确认，不会自动放行** |
| **Deny 规则** | symlink 路径 **或** 真实目标路径 **任一** 匹配 deny 规则，就拒绝 | `./project/stolen-key` 没命中 deny，但 `~/.ssh/id_rsa` 命中了 → **直接拒绝** |

**具体例子**

沿用上面的配置：

```json
{
  "permissions": {
    "allow": ["Read(./project/**)"],
    "deny": ["Read(~/.ssh/**)"]
  }
}
```

场景一：symlink 指向允许区域
```bash
ln -s ./project/README.md ./project/link-to-readme
```
- Claude 尝试读取 `./project/link-to-readme`
- symlink 路径 `./project/link-to-readme` → 匹配 allow
- 真实目标 `./project/README.md` → 也匹配 allow
- **结果：自动放行（无需弹窗）**

场景二：symlink 指向受限区域（Allow 规则）
```bash
ln -s ~/.ssh/id_rsa ./project/stolen-key
```
- Claude 尝试读取 `./project/stolen-key`
- symlink 路径 `./project/stolen-key` → 匹配 allow
- 真实目标 `~/.ssh/id_rsa` → **不匹配 allow**（`~/.ssh` 不在 `./project/**` 范围内）
- **结果：弹窗向用户确认，不会自动放行**

场景三：symlink 指向受限区域（Deny 规则）
```bash
ln -s ~/.ssh/id_rsa ./project/stolen-key
```
- Claude 尝试读取 `./project/stolen-key`
- symlink 路径 `./project/stolen-key` → 没命中 deny
- 真实目标 `~/.ssh/id_rsa` → **命中 deny**
- **结果：直接拒绝，无法读取**

这个机制防止了通过创建 symlink 绕过文件访问限制的简单攻击。攻击者（或一个被注入恶意指令的 Claude）无法在项目目录里放一个指向敏感文件的快捷方式，然后让 Claude 自动读取它。

---

## 第二道防线：权限模式选型

权限模式（Permission Mode）决定了 Claude Code 在运行时如何处理工具调用请求。你可以在 `settings.json` 中设置 `permissions.defaultMode`。

| 模式 | 行为 | 适用场景 |
|------|------|----------|
| `default` | 首次使用每个工具时弹窗确认，之后记住选择 | 初学者，逐步建立信任 |
| `acceptEdits` | **自动批准文件编辑**和常见文件系统命令（`mkdir`、`touch`、`rm`、`mv`、`cp`、`sed` 等），其他 Bash 命令仍弹窗 | **最推荐的日常开发模式** |
| `plan` | 只读模式：允许读文件和只读命令，**绝不修改源文件** | 探索新仓库、审查代码 |
| `auto` | 自动批准所有工具调用，带后台安全检查（研究预览） | 对效率要求极高且信任度高的场景 |
| `dontAsk` | 自动拒绝所有未预先允许的工具调用 | 高度受限环境 |
| `bypassPermissions` | **跳过所有权限提示**。仅对 `rm -rf /` 或 `rm -rf ~` 保留熔断提示 | **仅用于隔离环境（容器/VM）** |

### 为什么 `acceptEdits` 是最佳平衡点

`acceptEdits` 模式解决了核心痛点：

- **文件编辑自动批准**：修改代码、重构、写测试等高频操作不再被打断。
- **危险命令仍需确认**：执行 `npm install`、`docker build`、`git push` 等仍需要你的许可。
- **文件系统命令受限**：自动批准的 `rm`、`mv` 等命令仅限于**工作目录及其子目录**内，不会误删系统文件或上级目录内容。

配置方式：

```json
{
  "permissions": {
    "defaultMode": "acceptEdits"
  }
}
```

### 为什么避免使用 `bypassPermissions`

`bypassPermissions`（通过 `--dangerously-skip-permissions` 或 `defaultMode: "bypassPermissions"` 开启）虽然能彻底消除确认弹窗，但代价是：

- Claude 可以**无提示地写入 `.git`、`.claude`、`.vscode` 等关键目录**。
- 可以**无提示地执行任何 Bash 命令**，包括 `curl` 下载并执行远程脚本。
- 仅存的 "熔断器" 只是对根目录和用户主目录的 `rm -rf`，远远不够。

如果你确实在某些特殊场景（如在一次性容器里运行）需要这个模式，务必在配置中**锁定自己无法轻易开启它**：

```json
{
  "permissions": {
    "disableBypassPermissionsMode": "disable"
  }
}
```

这会禁用 `--dangerously-skip-permissions` 命令行标志，防止误操作进入完全无保护状态。

---

## 第三道防线：OS 级沙箱隔离

权限规则控制的是 Claude "尝试做什么"，而沙箱控制的是 Bash 命令在操作系统层面"实际能访问什么"。两者结合才能实现真正的深度防御。

### 沙箱的工作原理

**沙箱是 Claude Code 客户端本地的机制**，与后端接入的模型无关（无论用 Anthropic API、Bedrock、Vertex 还是 Foundry，本地 Bash 命令都会被隔离）。不同平台的实现方式不同：

| 平台 | 实现机制 | 支持状态 |
|------|---------|----------|
| macOS | Seatbelt 框架 | 支持 |
| Linux | bubblewrap（bwrap）+ socat | 支持 |
| WSL2 | bubblewrap（与 Linux 相同） | 支持 |
| WSL1 | — | **不支持**（缺少内核命名空间特性） |
| Windows 原生 | — | **尚未支持**（文档标注 "planned"） |

沙箱为 Bash 命令创建一个受限的执行环境：

- **文件系统**：默认只允许写入**当前工作目录及其子目录**。可以读取整个文件系统（除非被显式拒绝）。
- **网络**：通过本地代理服务器拦截所有出站连接，只有允许的域名才能访问。

![沙箱工作原理：OS 级隔离机制](https://static.xiangdangnian.net.cn/blog/02-flowchart-sandbox-isolation-principle.png?v=20260519)

### 启用沙箱

在交互式会话中运行：

```bash
/sandbox
```

或在 `settings.json` 中启用：

```json
{
  "sandbox": {
    "enabled": true,
    "autoAllowBashIfSandboxed": true,
    "failIfUnavailable": true
  }
}
```

- `autoAllowBashIfSandboxed: true`（默认）：沙箱内的命令自动批准，无需弹窗。沙箱边界替代了逐条命令的人工确认。
- `failIfUnavailable: true`：如果沙箱因缺少依赖无法启动，Claude Code 直接退出而不是降级到无沙箱模式。适合企业强制部署。

### 配置沙箱边界

```json
{
  "sandbox": {
    "enabled": true,
    "filesystem": {
      "allowWrite": ["~/.kube", "/tmp/build"],
      "denyWrite": ["/etc", "/usr/local/bin"],
      "denyRead": ["~/.aws/credentials"],
      "allowRead": ["."]
    },
    "network": {
      "allowedDomains": ["github.com", "*.npmjs.org", "registry.yarnpkg.com"],
      "deniedDomains": ["uploads.github.com"]
    }
  }
}
```

**路径前缀规则**：

| 前缀 | 含义 |
|------|------|
| `/path` | 绝对路径 |
| `~/path` | 相对于用户主目录 |
| `./path` 或 `path` | 相对于项目根目录（project settings）或 `~/.claude`（user settings） |

注意：沙箱路径语法与 Read/Edit 权限规则不同。沙箱使用标准路径约定，`/tmp` 就是绝对路径 `/tmp`；而 Read/Edit 规则中 `/tmp` 表示相对于项目根目录。

### 禁用沙箱逃逸舱口

默认情况下，如果沙箱内的命令因权限不足失败，Claude 可能会被提示使用 `dangerouslyDisableSandbox` 参数重试。这虽然方便处理边缘情况，但也削弱了沙箱的意义。

**推荐做法**：在需要严格安全的场景中禁用这个逃逸舱口。

```json
{
  "sandbox": {
    "enabled": true,
    "allowUnsandboxedCommands": false
  }
}
```

设置后，`dangerouslyDisableSandbox` 参数会被完全忽略，所有命令必须在沙箱内运行，或显式列入 `excludedCommands`。

### 沙箱与权限规则的关系

这是最容易混淆的一点，务必理清：

- **权限规则**（`permissions.allow/deny`）作用于**所有工具**（Bash、Read、Edit、WebFetch、MCP、Agent）。
- **沙箱**仅作用于 **Bash 命令及其子进程**的 OS 级文件系统和网络访问。
- Read/Edit 的 `deny` 规则会自动合并到沙箱的 `denyRead`/`denyWrite` 中。
- WebFetch 的 allow/deny 规则与沙箱的 `allowedDomains`/`deniedDomains` 是互补的。

**最佳实践**：同时启用两者。
- 用 `permissions.deny` 阻止 Claude 尝试读取敏感文件。
- 用 `sandbox.filesystem.denyRead` 在 OS 层面阻止所有子进程（包括 Node 脚本、Python 脚本）间接读取敏感文件。
- 用 `sandbox.network.allowedDomains` 限制所有子进程只能访问白名单域名。

---

## 第四道防线：Hooks 运行时动态控制

Hooks 是 Claude Code 的扩展机制，允许你在特定生命周期事件发生时运行自定义脚本。对于权限管理，最有价值的是 **`PreToolUse` Hook**。

![PreToolUse Hook 决策流程](https://static.xiangdangnian.net.cn/blog/03-flowchart-pretooluse-hook-flow.png?v=20260519)

### PreToolUse Hook 能做什么

在 Claude 调用任何工具之前，`PreToolUse` Hook 会收到工具调用的完整信息，包括工具名称和参数。Hook 可以：

- **拒绝工具调用**（exit code 2）：工具不会执行，Claude 会收到拒绝通知。
- **强制要求确认**（exit code 1）：绕过 allow 规则，强制弹窗让用户确认。
- **允许执行**（exit code 0）：正常进入权限规则评估流程。

**重要**：Hook 的决策与权限规则是叠加关系，不是替代关系：

- Hook 返回 "拒绝"（exit 2）→ 直接阻断，即使存在匹配的 `allow` 规则。
- Hook 返回 "允许"（exit 0）→ 仍会继续检查 `deny` 和 `ask` 规则。`deny` 规则依然可以阻断。

这意味着你可以用 Hook 实现非常灵活的**动态策略**，例如：

- 拦截所有包含 `rm -rf` 的命令，要求二次确认。
- 检查 Bash 命令中的 URL，阻止访问非白名单域名。
- 在特定时间段（如周末）禁止 `git push` 到生产分支。

### Hook 配置示例

```json
{
  "hooks": {
    "preToolUse": {
      "type": "command",
      "command": "/usr/local/bin/claude-permission-gate.sh"
    }
  }
}
```

Hook 脚本 `claude-permission-gate.sh` 会收到 JSON 输入，包含 `toolName` 和 `input` 字段。下面是一个**拦截 Bash 网络工具并检查域名白名单**的完整示例：

```bash
#!/bin/bash
# claude-permission-gate.sh
# PreToolUse Hook：在 Claude 调用工具前执行

read -r payload
tool=$(echo "$payload" | jq -r '.toolName')

# 仅处理 Bash 工具调用
if [ "$tool" != "Bash" ]; then
    exit 0
fi

command=$(echo "$payload" | jq -r '.input.command // empty')

# 白名单域名（正则格式）
ALLOWED_DOMAINS="github\.com|npmjs\.org|registry\.yarnpkg\.com"

# 如果命令包含 curl/wget，提取 URL 并检查白名单
if echo "$command" | grep -Eq '\b(curl|wget)\b'; then
    # 尝试提取命令中的 URL（简单正则，覆盖常见写法）
    urls=$(echo "$command" | grep -oE 'https?://[^[:space:]"]+' || true)

    if [ -n "$urls" ]; then
        while IFS= read -r url; do
            domain=$(echo "$url" | sed -E 's|https?://||' | cut -d'/' -f1)
            if ! echo "$domain" | grep -Eq "$ALLOWED_DOMAINS"; then
                echo "Hook blocked: network tool accessing non-allowed domain '$domain'" >&2
                exit 2  # 拒绝执行
            fi
        done <<< "$urls"
    fi
fi

# 拦截 rm -rf 对重要目录的操作
if echo "$command" | grep -Eq 'rm\s+-rf\s+(/|~|/etc|/usr|/bin)'; then
    echo "Hook blocked: dangerous rm -rf detected" >&2
    exit 2
fi

exit 0
```

这个脚本演示了 Hook 的核心用法：
- 通过 `jq` 从 JSON 中取出 `toolName` 和 `input.command`。
- 用 `exit 2` 直接拒绝不符合策略的工具调用。
- 对非 Bash 调用直接 `exit 0` 放行，不干扰其他工具。

**注意**：这只是一个示例级别的正则匹配，面对复杂的 Shell 语法（如变量拼接、命令替换）仍有局限。生产环境建议结合沙箱网络隔离，形成双重保障。

### 其他相关 Hook 事件

- **`PermissionRequest`**：在权限请求弹窗之前触发，可用于记录审计日志或实现自定义审批流程。
- **`PostToolUse`**：工具执行后触发，可用于记录实际执行了什么操作。
- **`ConfigChange`**：配置变更时触发，适合在团队环境中审计 settings.json 的修改。

---

## 组织级安全策略（Managed Settings）

对于团队或企业，Claude Code 提供了 **Managed Settings** 机制，允许管理员通过以下方式下发不可覆盖的安全策略：

- **Server-managed**：通过 Claude.ai 管理控制台下发。
- **MDM/OS 策略**：macOS 配置描述文件、Windows 注册表（`HKLM\SOFTWARE\Policies\ClaudeCode`）、Intune/Group Policy。
- **文件部署**：`managed-settings.json` 放在系统目录（如 `/Library/Application Support/ClaudeCode/` 或 `C:\Program Files\ClaudeCode\`）。

### 关键的组织级强制设置

```json
{
  "permissions": {
    "deny": [
      "Read(//etc/**)",
      "Read(~/.ssh/**)",
      "Read(~/.aws/**)"
    ],
    "disableBypassPermissionsMode": "disable",
    "disableAutoMode": "disable"
  },
  "sandbox": {
    "enabled": true,
    "allowUnsandboxedCommands": false,
    "failIfUnavailable": true
  },
  "allowManagedPermissionRulesOnly": true
}
```

**各设置含义**：

| 设置 | 作用 |
|------|------|
| `allowManagedPermissionRulesOnly` | 禁止用户和项目设置中定义 `allow`/`ask`/`deny` 规则，仅 managed settings 中的规则生效 |
| `disableBypassPermissionsMode` | 禁用 `--dangerously-skip-permissions` 标志 |
| `disableAutoMode` | 禁用 auto 权限模式 |
| `sandbox.allowUnsandboxedCommands: false` | 禁用沙箱逃逸舱口 |
| `sandbox.failIfUnavailable: true` | 沙箱无法启动时直接退出 CLI |

**注意**：Managed settings 的数组值（如 `deny`、`allowWrite`）会与用户/项目设置的对应数组合并，而不是替换。这意味着管理员可以设置基线 deny 规则，团队仍可在项目中追加额外的 deny 规则。

---

## 实战配置模板

以下是三套可直接复制使用的 `settings.json` 模板，分别对应个人开发者、团队协作和企业安全场景。

### 模板一：个人开发者（安全与效率兼顾）

保存到 `~/.claude/settings.json`：

```json
{
  "$schema": "https://json.schemastore.org/claude-code-settings.json",
  "permissions": {
    "defaultMode": "acceptEdits",
    "disableBypassPermissionsMode": "disable",
    "allow": [
      "Bash(npm run *)",
      "Bash(git status)",
      "Bash(git log *)",
      "Bash(git diff *)"
    ],
    "ask": [
      "Bash(git push *)",
      "Bash(git pull *)",
      "Bash(docker *)"
    ],
    "deny": [
      "Bash(curl *)",
      "Bash(wget *)",
      "Read(~/.ssh/**)",
      "Read(~/.aws/**)",
      "Read(//etc/**)",
      "Read(~/.bashrc)",
      "Read(~/.zshrc)"
    ]
  },
  "sandbox": {
    "enabled": true,
    "autoAllowBashIfSandboxed": true,
    "filesystem": {
      "denyRead": ["~/.ssh", "~/.aws", "//etc"]
    }
  }
}
```

### 模板二：团队项目（共享安全基线）

保存到 `.claude/settings.json`（进入 Git）：

```json
{
  "$schema": "https://json.schemastore.org/claude-code-settings.json",
  "permissions": {
    "defaultMode": "acceptEdits",
    "deny": [
      "Read(./.env)",
      "Read(./.env.*)",
      "Read(./secrets/**)",
      "Read(./config/credentials.json)",
      "Edit(./.git/**)",
      "Bash(curl *)",
      "Bash(wget *)",
      "Bash(rm -rf *)"
    ]
  },
  "sandbox": {
    "enabled": true,
    "filesystem": {
      "denyRead": ["./secrets", "./.env", "./.env.*"],
      "denyWrite": ["./.git"]
    },
    "network": {
      "allowedDomains": ["github.com", "*.npmjs.org", "registry.yarnpkg.com"]
    }
  }
}
```

### 模板三：企业强制安全（Managed Settings）

部署到 `C:\Program Files\ClaudeCode\managed-settings.json`（Windows）或 `/Library/Application Support/ClaudeCode/managed-settings.json`（macOS）：

```json
{
  "permissions": {
    "deny": [
      "Read(//etc/**)",
      "Read(~/.ssh/**)",
      "Read(~/.aws/**)",
      "Read(~/.kube/**)",
      "Bash(curl *)",
      "Bash(wget *)",
      "Bash(nc *)",
      "Bash(netcat *)"
    ],
    "disableBypassPermissionsMode": "disable",
    "disableAutoMode": "disable"
  },
  "sandbox": {
    "enabled": true,
    "allowUnsandboxedCommands": false,
    "failIfUnavailable": true,
    "filesystem": {
      "denyWrite": ["/usr/local/bin", "/bin", "/sbin", "~/.bashrc", "~/.zshrc"],
      "denyRead": ["~/.ssh", "~/.aws", "//etc"]
    }
  },
  "allowManagedPermissionRulesOnly": true,
  "disableRemoteControl": true,
  "disableSkillShellExecution": true
}
```

---

## 常见错误与排障指南

### 1. Bash 规则匹配过于宽泛

**错误**：
```json
"allow": ["Bash(git *)"]
```
这会匹配 `git status`，但也匹配 `git push --force`。

**修复**：尽可能写明确的规则：
```json
"allow": ["Bash(git status)", "Bash(git log *)", "Bash(git diff *)"],
"ask": ["Bash(git push *)"]
```

### 2. 忘记 Read/Edit 规则的路径语义

**错误**：
```json
"deny": ["Read(/Users/alice/secrets)"]
```
这实际上匹配的是 `<项目根>/Users/alice/secrets`，而不是绝对路径。

**修复**：
```json
"deny": ["Read(//Users/alice/secrets)"]
```

### 3. 依赖 WebFetch 规则阻止网络访问

**错误**：认为设置 `deny: ["WebFetch"]` 就能阻止所有网络访问。

**事实**：WebFetch 规则只控制 Claude 内置的 `WebFetch` 工具。如果 Bash 工具被允许，Claude 仍可以通过 `curl`、`wget`、Python 脚本等方式访问网络。

**修复**：同时 deny Bash 网络工具，并启用沙箱网络隔离。

### 4. 沙箱路径与权限规则路径混淆

**错误**：在沙箱配置中使用 `//path` 语法。

**事实**：沙箱路径使用标准路径约定，`/tmp` 就是绝对路径。而 Read/Edit 权限规则中 `/tmp` 表示项目相对路径。

### 5. 通过 `additionalDirectories` 意外扩大攻击面

使用 `--add-dir` 或 `permissions.additionalDirectories` 可以扩展文件访问范围，但**不会**让该目录成为配置根目录。大部分 `.claude/` 配置不会从附加目录加载。如果只是为了让 Claude 读取项目外的文档，请确保不在附加目录中放置敏感文件。

### 6. 沙箱内工具不兼容

某些工具无法在沙箱内运行：
- `docker`（需要访问 Docker socket）
- `watchman`（与沙箱不兼容，影响 Jest watch 模式）
- Windows 二进制（在 WSL2 沙箱中无法启动 `.exe`）

**修复**：将这些工具加入 `sandbox.excludedCommands`：
```json
{
  "sandbox": {
    "excludedCommands": ["docker *", "watchman *"]
  }
}
```
这些命令将在沙箱外运行，但仍受权限规则约束。

---

## 总结

![五大安全策略总览](https://static.xiangdangnian.net.cn/blog/04-infographic-five-security-strategies.png?v=20260519)

Claude Code 的权限管理是一个**分层防御**体系，而不是一个开关。要在不使用危险模式的前提下安全高效地工作，核心策略是：

1. **用 `acceptEdits` 模式替代 `bypassPermissions`**：自动批准文件编辑，但保留对危险命令的确认。
2. **用 `deny` 规则建立文件保护基线**：优先保护 `.env`、密钥、系统配置和 Git 目录。
3. **用沙箱建立 OS 级硬边界**：即使 Claude 被注入恶意指令，子进程也无法突破文件系统和网络限制。
4. **用 Hooks 实现动态策略**：对特别敏感的操作（如删除、推送到生产分支）实现二次确认或审计日志。
5. **在团队/企业中使用 Managed Settings**：将安全基线下发到设备，确保不可覆盖。

最有效的安全策略不是让 AI 什么都问，而是**让 AI 在清晰的边界内自由工作，同时确保边界外的一切被物理隔离**。

---

## 延伸阅读

- [Claude Code Settings 官方文档](https://code.claude.com/docs/en/settings) — 最权威的配置参考
- [Claude Code Permissions 官方文档](https://code.claude.com/docs/en/permissions) — 权限规则语法和模式详解
- [Claude Code Sandboxing 官方文档](https://code.claude.com/docs/en/sandboxing) — 沙箱机制的技术细节
- [Claude Code Security 官方文档](https://code.claude.com/docs/en/security) — 安全最佳实践和威胁模型
- [Claude Code Hooks 官方文档](https://code.claude.com/docs/en/hooks) — 扩展权限控制的运行时钩子

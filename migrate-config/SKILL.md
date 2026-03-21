---
name: migrate-config
description: 导出/导入 Claude Code 完整配置（settings、marketplaces、plugins、skills），支持文件导出和 Git 仓库同步
---

# Migrate Config

一键导出/导入 Claude Code 完整运行环境。支持导出为本地脚本文件，或推送到 Git 仓库实现跨机器同步。

## 用法

| 命令 | 说明 |
|------|------|
| `/migrate-config export` | 交互选择导出方式（文件 / Git 仓库） |
| `/migrate-config export --output <path>` | 导出配置到指定文件路径 |
| `/migrate-config export --git <repo-url>` | 导出配置到 Git 仓库（跳过方式选择） |
| `/migrate-config import <script-path>` | 从本地脚本文件导入 |
| `/migrate-config import --git <repo-url>` | 从 Git 仓库 clone 并导入 |

## Export 流程

### Step 0: 选择导出方式

> 如果命令行已指定 `--output` 或 `--git`，跳过此步骤，直接进入 Step 1。

用 AskUserQuestion 询问导出方式：

```
请选择配置导出方式：
```

| 选项 | 说明 |
|------|------|
| **导出为脚本文件（推荐）** | 生成自包含的 `claude-setup.sh` 安装脚本，可离线使用，适合一次性迁移 |
| **推送到 Git 仓库** | 将配置文件推送到 Git 仓库，适合多台机器持续同步。需要提供仓库 URL |

**如果用户选择「推送到 Git 仓库」**，用 AskUserQuestion 追问仓库 URL：

```
请输入 Git 仓库 URL（支持 HTTPS 和 SSH）：
```

选项：
- **手动输入** — 用户在 Other 中填写仓库 URL（如 `https://github.com/user/claude-config.git` 或 `git@github.com:user/claude-config.git`）

> 无预设选项，用户必须通过 Other 输入。拿到 URL 后立即执行 **Git 仓库权限检测**（见下文），通过后再继续 Step 1。

#### Git 仓库权限检测

在确定仓库 URL 后、扫描配置之前，必须验证仓库可访问且有 push 权限。

**检测流程**：

```bash
# 1. 检查 git 是否可用
command -v git &>/dev/null || { echo "[FAIL] git 未安装"; exit 1; }

# 2. 尝试 ls-remote 验证仓库可达性和读权限
if ! git ls-remote "$REPO_URL" HEAD 2>/tmp/claude-git-check.log; then
    ERROR=$(cat /tmp/claude-git-check.log)
    # 根据错误信息分类提示
    # - "Permission denied (publickey)" → SSH Key 未配置
    # - "Repository not found" → 仓库不存在或无权限
    # - "Could not resolve host" → 网络问题
    echo "[FAIL] 无法访问仓库: $REPO_URL"
    echo "$ERROR"
    exit 1
fi

# 3. 验证 push 权限（通过 clone + 尝试 push 空提交检测）
#    对于新仓库（空仓库），ls-remote 成功即认为有权限
#    对于已有仓库，clone 后检查是否能 push：
TEMP_DIR=$(mktemp -d)
if git clone --depth 1 --quiet "$REPO_URL" "$TEMP_DIR/test-repo" 2>/dev/null; then
    cd "$TEMP_DIR/test-repo"
    # 尝试 dry-run push 检测写权限
    if ! git push --dry-run 2>/tmp/claude-git-push-check.log; then
        echo "[FAIL] 有读权限但无 push 权限"
        echo "$(cat /tmp/claude-git-push-check.log)"
        rm -rf "$TEMP_DIR"
        exit 1
    fi
    cd -
    rm -rf "$TEMP_DIR"
else
    # clone 失败但 ls-remote 成功 → 可能是空仓库，继续
    rm -rf "$TEMP_DIR"
fi

echo "[OK] 仓库可访问且有 push 权限"
```

**错误处理映射**：

| 错误类型 | 错误特征 | 提示信息 |
|---------|---------|---------|
| SSH Key 缺失 | `Permission denied (publickey)` | `SSH Key 未配置或未添加到 ssh-agent。运行 ssh-add ~/.ssh/id_rsa 或检查 Git 平台的 SSH Key 设置` |
| 仓库不存在 | `Repository not found` | `仓库不存在。请先在 Git 平台创建仓库，或检查 URL 是否正确` |
| 网络不可达 | `Could not resolve host` | `无法解析主机名，请检查网络连接` |
| 无 push 权限 | `push --dry-run` 失败 | `有读权限但无 push 权限。请检查仓库权限设置（需要 write/push 权限）` |
| Token 过期 | `401` / `403` | `认证失败或 Token 过期，请更新 Git 凭据` |

> 检测失败时终止流程，显示错误信息和修复建议，不继续扫描。
> 检测通过后输出 `[OK] 仓库可访问且有 push 权限`，继续 Step 1。

### Step 1: 扫描当前配置

读取以下文件，收集完整配置信息：

**必须导出（默认）**：
```
~/.claude/settings.json              → 核心设置（model、env、permissions、enabledPlugins）
~/.claude/CLAUDE.md                  → 全局用户指令（如果存在）
~/.claude/agents/                    → 自定义 agent 定义（如果存在）
~/.claude/plugins/known_marketplaces.json → marketplace 来源列表
~/.claude/skills/                    → 自定义 skill 目录
```

**可选导出（交互询问）**：
```
~/.claude/settings.local.json        → 本地设置覆盖（如果存在）
~/.claude/projects/*/memory/         → 项目记忆（每个项目独立选择）
```

**不导出（会话临时数据）**：
```
~/.claude/sessions/                  → 会话历史
~/.claude/history.jsonl              → 命令历史
~/.claude/cache/                     → 缓存
~/.claude/file-history/              → 文件变更历史
~/.claude/tasks/                     → 临时任务
~/.claude/teams/                     → 临时团队
~/.claude/session-env/               → 会话环境快照
```

#### 可选导出的交互

扫描完成后，如果发现可选组件存在，用 AskUserQuestion 询问：

```
发现以下可选配置，是否一并导出？
```

选项（multiSelect: true，可多选）：

| 选项 | 说明 |
|------|------|
| **settings.local.json** | 本地设置覆盖（同样会脱敏） |
| **项目记忆 (dirichlet-analytics)** | 该项目的 AI 记忆上下文 (MEMORY.md + 记忆文件) |
| **项目记忆 (PhonePilot)** | 该项目的 AI 记忆上下文 |
| **项目记忆 (adn-track)** | 该项目的 AI 记忆上下文 |

> 只展示实际存在的选项。如果没有可选组件，跳过此交互。
> 项目记忆的目录名从路径中提取最后一段作为显示名（如 `-Users-wszf-...-dirichlet-analytics` → `dirichlet-analytics`）。

### Step 2: 分类 Skills 并交互确认

遍历 `~/.claude/skills/` 下的每个子目录，检测类型和状态：

```bash
for d in ~/.claude/skills/*/; do
  name=$(basename "$d")
  if [ -d "$d/.git" ]; then
    url=$(cd "$d" && git remote get-url origin 2>/dev/null || echo "")
    dirty=$(cd "$d" && git status --porcelain 2>/dev/null)
    size=$(du -sh "$d" | cut -f1)
    # 判断是否为 SSH URL（可能是私有仓库）
    is_ssh=$( [[ "$url" == git@* ]] && echo "yes" || echo "no" )
    echo "GIT  $name  url=$url  dirty=$dirty  size=$size  ssh=$is_ssh"
  else
    size=$(du -sh "$d" | cut -f1)
    echo "LOCAL  $name  ($size)"
  fi
done
```

#### 2a. 边界情况处理

在展示扫描结果之前，先处理以下特殊情况：

**情况 1：Git 仓库无 remote URL**
- `git remote get-url origin` 返回空或失败
- 说明是本地 `git init` 的仓库，无法网络拉取
- **处理**：自动归为 Local Skill，不出现在 Git Skill 列表中

**情况 2：Git 仓库有未提交修改**
- `git status --porcelain` 返回非空
- 说明用户对 clone 的 skill 做了本地定制
- **处理**：在扫描结果中标注 `[有本地修改]`，并警告：
  ```
  ⚠ gstack 有未提交的本地修改，选择「网络拉取」会丢失这些改动，建议选择「本地打包」
  ```

**情况 3：没有发现任何 Git Skill**
- 所有 skill 都是 Local
- **处理**：跳过交互询问，直接进入 Step 3 全部打包

**情况 4：没有发现任何 Local Skill**
- 所有 skill 都是 Git 仓库
- **处理**：仍然询问打包方式，但如果全部选「网络拉取」，则不生成 `__SKILLS_ARCHIVE__` 段

**情况 5：SSH URL 或私有仓库**
- URL 以 `git@` 开头（SSH 协议），通常用于私有仓库
- **处理**：在扫描结果中标注 `[SSH/私有仓库]`，并警告：
  ```
  ⚠ get-shit-done 使用 SSH URL (git@github.com:...)，目标机器需要配置对应的 SSH Key 才能 clone。
    如果目标机器没有 SSH 权限，建议选择「本地打包」。
  ```

> **判断私有仓库的启发式规则**：
> - `git@` 开头 → 标注 SSH，提醒需要 SSH Key
> - `https://` 开头但非 github.com/gitlab.com 等公共平台 → 标注可能需要认证
> - 无法 100% 确定是否私有，所以用「提醒」而非「强制」

#### 2b. 展示扫描结果并询问

**发现 Git Skill 后，展示扫描结果并用 AskUserQuestion 让用户选择**：

```
扫描到以下 Git Skills：

| Skill | Git URL | 大小 | 状态 |
|-------|---------|------|------|
| gstack | https://github.com/garrytan/gstack.git | 27MB | - |
| get-shit-done | git@github.com:gsd-build/get-shit-done.git | 7.3MB | ⚠ SSH/私有仓库 |
| my-custom | https://github.com/user/my-custom.git | 2MB | ⚠ 有本地修改 |

本地 Skills（自动打包）：
| Skill | 大小 |
|-------|------|
| migrate-config | 8KB |
| permission-manager | 12KB |

⚠ 提醒：
- get-shit-done 使用 SSH URL，目标机器需要对应 SSH Key
- my-custom 有未提交的本地修改，网络拉取会丢失这些改动
```

AskUserQuestion 选项：

| 选项 | 说明 |
|------|------|
| **网络拉取（推荐）** | Git Skills 在新机器上通过 `git clone` 安装，脚本体积小，且自动获取最新版本。⚠ SSH URL 的 skill 需要目标机器有对应权限 |
| **全部本地打包** | 所有 Skills（含 Git 仓库）都 tar.gz 压缩后嵌入脚本，离线可用，不依赖网络和权限 |
| **逐个选择** | 对每个 Git Skill 分别选择网络拉取或本地打包 |

**如果用户选择「逐个选择」**，对每个 Git Skill 分别用 AskUserQuestion 询问：

```
Skill: gstack (27MB)
Git URL: https://github.com/garrytan/gstack.git

选择打包方式？
```

选项：
- **网络拉取** — 脚本中记录 git clone URL，目标机器联网安装
- **本地打包** — 压缩嵌入脚本（排除 .git 目录以减小体积）

**本地打包时的优化**：如果用户选择本地打包 Git Skill，打包时排除 `.git` 目录以减小体积：

```bash
tar -czf /tmp/skill.tar.gz -C ~/.claude --exclude='.git' skills/<name>
```

### Step 3: 打包 Skills

根据 Step 2 的用户选择，收集需要本地打包的 skill：

```bash
# LOCAL_SKILLS 包含：所有非 git skill + 用户选择本地打包的 git skill
LOCAL_SKILLS=()
for d in ~/.claude/skills/*/; do
  name=$(basename "$d")
  if <该 skill 需要本地打包>; then
    LOCAL_SKILLS+=("skills/$name")
  fi
done

# 打包（排除 .git 目录）
tar -czf /tmp/claude-local-skills.tar.gz -C ~/.claude --exclude='.git' "${LOCAL_SKILLS[@]}"
```

### Step 4: 根据导出方式分流

> 从 Step 0 选择的导出方式决定接下来的路径：
> - **文件模式** → Step 4a（生成安装脚本）
> - **Git 仓库模式** → Step 4b（推送到 Git 仓库）

### Step 4a: 生成安装脚本（文件模式）

生成 `claude-setup.sh`，结构如下：

```bash
#!/bin/bash
set -e
CLAUDE_DIR="$HOME/.claude"

# base64 解码兼容 macOS(-D) 和 Linux(-d)
if base64 --help 2>&1 | grep -q -- '-D'; then
    B64_DECODE="base64 -D"
else
    B64_DECODE="base64 -d"
fi

# ===== 1. 前置检查 =====
command -v claude &>/dev/null || { echo "请先安装 Claude Code: npm i -g @anthropic-ai/claude-code"; exit 1; }
command -v git &>/dev/null || { echo "请先安装 git"; exit 1; }

# ===== 2. settings.json（脱敏后） =====
setup_settings() {
    mkdir -p "$CLAUDE_DIR"
    [ -f "$CLAUDE_DIR/settings.json" ] && cp "$CLAUDE_DIR/settings.json" "$CLAUDE_DIR/settings.json.bak"
    cat > "$CLAUDE_DIR/settings.json" << 'SETTINGS_EOF'
<settings.json 内容，敏感字段已替换为占位符>
SETTINGS_EOF
}

# ===== 3. Marketplaces =====
install_marketplaces() {
    mkdir -p "$CLAUDE_DIR/plugins/marketplaces"
    # 对 known_marketplaces.json 中每个 marketplace：
    clone_or_pull() {
        local name="$1" repo="$2"
        if [ -d "$CLAUDE_DIR/plugins/marketplaces/$name" ]; then
            (cd "$CLAUDE_DIR/plugins/marketplaces/$name" && git pull --quiet 2>/dev/null || true)
        else
            git clone --quiet "https://github.com/$repo.git" "$CLAUDE_DIR/plugins/marketplaces/$name"
        fi
    }
    <对每个 marketplace 调用 clone_or_pull>
    <写入 known_marketplaces.json，installLocation 用 $CLAUDE_DIR 拼接，不用源机器绝对路径>
}

# ===== 4. Git Skills（从 GitHub clone） =====
install_git_skills() {
    mkdir -p "$CLAUDE_DIR/skills"
    clone_skill() {
        local name="$1" repo="$2"
        if [ -d "$CLAUDE_DIR/skills/$name" ]; then
            echo "[SKIP] $name (pull 更新)"
            (cd "$CLAUDE_DIR/skills/$name" && git pull --quiet 2>/dev/null || true)
        else
            echo "[CLONE] $name ..."
            if ! git clone --quiet "$repo" "$CLAUDE_DIR/skills/$name" 2>/dev/null; then
                echo "[FAIL] $name clone 失败"
                echo "       URL: $repo"
                echo "       可能原因: 私有仓库需要 SSH Key 或 Access Token"
                echo "       解决方法:"
                echo "         - SSH: 确保 ~/.ssh/ 下有对应私钥且已 ssh-add"
                echo "         - HTTPS 私有仓库: git clone https://<token>@github.com/..."
                echo "         - 或联系仓库管理员获取权限"
                GIT_FAILURES=$((GIT_FAILURES + 1))
            fi
        fi
    }
    GIT_FAILURES=0
    <对每个 git skill 调用 clone_skill>
    if [ "$GIT_FAILURES" -gt 0 ]; then
        echo ""
        echo "[WARN] $GIT_FAILURES 个 Git Skill clone 失败，请检查上方提示手动处理"
    fi
}

# ===== 5. 全局 CLAUDE.md =====
restore_claude_md() {
    <如果导出时存在 ~/.claude/CLAUDE.md，用 heredoc 写入>
    <如果不存在则跳过此步骤>
}

# ===== 6. 自定义 Agents =====
restore_agents() {
    <如果导出时存在 ~/.claude/agents/*.md，用 heredoc 逐个写入>
    <agents 文件通常很小（几KB），直接 heredoc 嵌入即可>
    <如果不存在则跳过此步骤>
}

# ===== 7. Local Skills（从脚本末尾 base64 解压） =====
restore_local_skills() {
    ARCHIVE_LINE=$(awk '/^__SKILLS_ARCHIVE__$/{print NR + 1; exit}' "$0")
    if [ -n "$ARCHIVE_LINE" ]; then
        tail -n +"$ARCHIVE_LINE" "$0" | $B64_DECODE | tar -xzf - -C "$CLAUDE_DIR"
    fi
}

# ===== 6. 验证 =====
verify() { <检查各组件是否存在并输出 OK/FAIL> }

setup_settings
install_marketplaces
install_git_skills
restore_local_skills
verify
exit 0

__SKILLS_ARCHIVE__
<仅 local skills 的 base64 编码 tar.gz 数据>
```

### Step 5a: 组装最终文件（文件模式）

**⚠ 重要：必须使用 Bash 工具写入脚本文件，禁止使用 Write 工具。** Write 工具会在用户界面展示完整文件 diff（几百行脚本内容），体验极差。用 Bash 的 `cat << 'EOF' > file` 方式写入可避免这个问题。

```bash
# Claude 执行步骤：
# 1. 用 Bash 工具的 cat heredoc 写脚本逻辑部分（到 __SKILLS_ARCHIVE__ 标记结尾）
cat << 'SCRIPT_EOF' > ~/claude-setup.sh
<脚本内容>
SCRIPT_EOF

# 2. 用 Bash 追加需要本地打包的 skills 的 base64 数据（排除 .git）：
tar -czf /tmp/claude-local-skills.tar.gz -C ~/.claude --exclude='.git' skills/migrate-config skills/permission-manager
base64 < /tmp/claude-local-skills.tar.gz >> ~/claude-setup.sh
rm /tmp/claude-local-skills.tar.gz
chmod +x ~/claude-setup.sh
```

> 注意：如果用户选择了某些 Git Skill 也本地打包，将它们一起加入 tar 命令（带 `--exclude='.git'`）。

### Step 6a: 输出（文件模式）

- 显示文件路径和大小
- 显示包含的组件清单，分类标注安装方式：
  - settings.json (脱敏)
  - CLAUDE.md (全局指令)
  - Agents: N 个 (heredoc 内嵌)
  - Marketplace: N 个 (git clone)
  - Git Skill: M 个 (git clone / 本地打包)
  - Local Skill: K 个 (base64 内嵌)
  - 可选: settings.local.json、项目记忆等
- 提示使用方法

### Step 4b: 推送到 Git 仓库（Git 仓库模式）

将配置组织为目录结构，推送到 Git 仓库。

#### 目录结构

在临时目录中组织以下结构：

```
claude-config/
├── README.md                          # 自动生成的说明文件
├── settings.json                      # 核心设置（脱敏后）
├── settings.local.json                # 本地设置覆盖（可选，脱敏后）
├── CLAUDE.md                          # 全局用户指令（如果存在）
├── agents/                            # 自定义 agent 定义
│   └── *.md
├── plugins/
│   └── known_marketplaces.json        # marketplace 来源列表（路径已替换为占位符）
├── skills/                            # 仅 Local Skills（tar.gz 打包，排除 .git）
│   └── local-skills.tar.gz            # 所有本地 skill 的压缩包
├── projects/                          # 可选：项目记忆
│   └── <project-name>/
│       └── memory/
│           ├── MEMORY.md
│           └── *.md
├── git-skills.json                    # Git Skills 的 URL 清单（网络拉取用）
└── install.sh                         # 自动安装脚本（从仓库恢复配置）
```

#### git-skills.json 格式

```json
{
  "skills": [
    {
      "name": "gstack",
      "url": "https://github.com/garrytan/gstack.git",
      "type": "git_clone"
    },
    {
      "name": "get-shit-done",
      "url": "git@github.com:gsd-build/get-shit-done.git",
      "type": "git_clone",
      "note": "SSH/私有仓库，需要对应 SSH Key"
    }
  ],
  "marketplaces": [
    {
      "name": "marketplace-name",
      "url": "https://github.com/org/marketplace.git"
    }
  ]
}
```

#### install.sh（仓库内嵌安装脚本）

生成一个轻量安装脚本，放在仓库根目录，用户 clone 后执行即可恢复：

```bash
#!/bin/bash
set -e
CLAUDE_DIR="$HOME/.claude"
SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"

echo "===== Claude Code 配置恢复 ====="

# 1. settings.json
mkdir -p "$CLAUDE_DIR"
[ -f "$CLAUDE_DIR/settings.json" ] && cp "$CLAUDE_DIR/settings.json" "$CLAUDE_DIR/settings.json.bak"
cp "$SCRIPT_DIR/settings.json" "$CLAUDE_DIR/settings.json"
echo "[OK] settings.json"

# 2. settings.local.json（如果存在）
[ -f "$SCRIPT_DIR/settings.local.json" ] && cp "$SCRIPT_DIR/settings.local.json" "$CLAUDE_DIR/settings.local.json" && echo "[OK] settings.local.json"

# 3. CLAUDE.md
[ -f "$SCRIPT_DIR/CLAUDE.md" ] && cp "$SCRIPT_DIR/CLAUDE.md" "$CLAUDE_DIR/CLAUDE.md" && echo "[OK] CLAUDE.md"

# 4. Agents
if [ -d "$SCRIPT_DIR/agents" ]; then
    mkdir -p "$CLAUDE_DIR/agents"
    cp "$SCRIPT_DIR/agents/"*.md "$CLAUDE_DIR/agents/" 2>/dev/null && echo "[OK] agents"
fi

# 5. Marketplaces（从 git-skills.json 读取 URL 并 clone）
if command -v jq &>/dev/null && [ -f "$SCRIPT_DIR/git-skills.json" ]; then
    mkdir -p "$CLAUDE_DIR/plugins/marketplaces"
    jq -r '.marketplaces[]? | "\(.name) \(.url)"' "$SCRIPT_DIR/git-skills.json" | while read name url; do
        if [ -d "$CLAUDE_DIR/plugins/marketplaces/$name" ]; then
            (cd "$CLAUDE_DIR/plugins/marketplaces/$name" && git pull --quiet 2>/dev/null || true)
            echo "[SKIP] marketplace: $name (pull 更新)"
        else
            git clone --quiet "$url" "$CLAUDE_DIR/plugins/marketplaces/$name" 2>/dev/null && echo "[CLONE] marketplace: $name" || echo "[FAIL] marketplace: $name"
        fi
    done
    # 恢复 known_marketplaces.json（路径替换为本机路径）
    cp "$SCRIPT_DIR/plugins/known_marketplaces.json" "$CLAUDE_DIR/plugins/known_marketplaces.json"
    # sed 替换 installLocation 中的占位符为实际路径
    sed -i.bak "s|\\\$CLAUDE_DIR|$CLAUDE_DIR|g" "$CLAUDE_DIR/plugins/known_marketplaces.json"
    rm -f "$CLAUDE_DIR/plugins/known_marketplaces.json.bak"
fi

# 6. Git Skills（从 git-skills.json 读取 URL 并 clone）
if command -v jq &>/dev/null && [ -f "$SCRIPT_DIR/git-skills.json" ]; then
    mkdir -p "$CLAUDE_DIR/skills"
    GIT_FAILURES=0
    jq -r '.skills[]? | "\(.name) \(.url)"' "$SCRIPT_DIR/git-skills.json" | while read name url; do
        if [ -d "$CLAUDE_DIR/skills/$name" ]; then
            (cd "$CLAUDE_DIR/skills/$name" && git pull --quiet 2>/dev/null || true)
            echo "[SKIP] skill: $name (pull 更新)"
        else
            if git clone --quiet "$url" "$CLAUDE_DIR/skills/$name" 2>/dev/null; then
                echo "[CLONE] skill: $name"
            else
                echo "[FAIL] skill: $name ($url)"
                GIT_FAILURES=$((GIT_FAILURES + 1))
            fi
        fi
    done
fi

# 7. Local Skills（解压 tar.gz）
if [ -f "$SCRIPT_DIR/skills/local-skills.tar.gz" ]; then
    tar -xzf "$SCRIPT_DIR/skills/local-skills.tar.gz" -C "$CLAUDE_DIR"
    echo "[OK] local skills"
fi

# 8. 项目记忆（如果有）
if [ -d "$SCRIPT_DIR/projects" ]; then
    for proj_dir in "$SCRIPT_DIR/projects"/*/; do
        proj_name=$(basename "$proj_dir")
        # 提醒用户项目路径可能需要调整
        echo "[INFO] 项目记忆: $proj_name — 已复制，如用户名不同请手动重命名目录"
        mkdir -p "$CLAUDE_DIR/projects/$proj_name/memory"
        cp -r "$proj_dir/memory/"* "$CLAUDE_DIR/projects/$proj_name/memory/" 2>/dev/null || true
    done
fi

echo ""
echo "===== 恢复完成 ====="
echo "⚠ 请检查 settings.json 中的脱敏占位符（如 <YOUR_TOKEN_HERE>），填入实际值"
```

#### 推送流程

```bash
# Claude 执行步骤：

# 1. 创建临时工作目录
WORK_DIR=$(mktemp -d)

# 2. 尝试 clone 已有仓库，或初始化新仓库
if git clone --quiet "$REPO_URL" "$WORK_DIR/claude-config" 2>/dev/null; then
    echo "[OK] clone 已有仓库"
    cd "$WORK_DIR/claude-config"
    # 清空旧内容（保留 .git）
    find . -maxdepth 1 ! -name '.git' ! -name '.' -exec rm -rf {} +
else
    echo "[INFO] 仓库为空或 clone 失败，初始化新仓库"
    mkdir -p "$WORK_DIR/claude-config"
    cd "$WORK_DIR/claude-config"
    git init --quiet
    git remote add origin "$REPO_URL"
fi

# 3. 复制配置文件到工作目录（按上述目录结构）
cp <脱敏后的 settings.json> .
cp <CLAUDE.md, agents/, plugins/, etc.> .
# Local Skills 打包为 tar.gz
mkdir -p skills
tar -czf skills/local-skills.tar.gz -C ~/.claude --exclude='.git' <local skill 列表>
# 生成 git-skills.json
# 生成 install.sh
# 生成 README.md

# 4. 生成 README.md
cat > README.md << 'EOF'
# Claude Code Configuration

此仓库由 `/migrate-config export --git` 自动生成，包含 Claude Code 运行环境配置。

## 使用方法

在新机器上执行：

```bash
git clone <此仓库 URL> ~/claude-config
cd ~/claude-config
bash install.sh
```

## 注意事项

- `settings.json` 中的敏感信息已替换为占位符，请手动填入
- Git Skills 需要目标机器有对应的访问权限
- 项目记忆路径包含源机器用户名，如不同请手动重命名
EOF

# 5. Commit & Push
git add -A
git commit -m "chore: export Claude Code configuration ($(date +%Y-%m-%d))"

# 对于新仓库（无 upstream）
git push -u origin main 2>/dev/null || git push -u origin master 2>/dev/null

# 对于已有仓库
git push
```

#### 输出（Git 仓库模式）

- 显示推送结果（成功/失败）
- 显示仓库 URL
- 显示包含的组件清单（同 Step 6a）
- 提示导入方法：
  ```
  在新机器上执行：
    git clone <repo-url> ~/claude-config
    cd ~/claude-config
    bash install.sh
  或使用：
    /migrate-config import --git <repo-url>
  ```
- 清理临时目录

## 关键实现细节

1. **Git Skill 判断**：检查 `~/.claude/skills/<name>/.git` 目录是否存在
2. **Git URL 提取**：`git remote get-url origin` 获取远程仓库地址；**无 remote 时自动归为 Local Skill**
3. **本地修改检测**：`git status --porcelain` 非空时标注警告，建议本地打包
4. **SSH/私有仓库识别**：URL 以 `git@` 开头时标注 `⚠ SSH/私有仓库`，提醒目标机器需要权限
5. **clone 失败容错**：生成的脚本中 `git clone` 失败时不退出（不被 `set -e` 中断），输出详细的权限排查指引
6. **base64 跨平台兼容**：macOS 用 `base64 -D`，Linux 用 `base64 -d`，脚本中需自动检测
7. **脱敏规则**：settings.json 中以下 key 的值替换为占位符：
   - `ANTHROPIC_AUTH_TOKEN` → `"<YOUR_TOKEN_HERE>"`
   - `ANTHROPIC_BASE_URL` → `"<YOUR_BASE_URL_HERE>"`
   - 其他包含 `TOKEN`、`SECRET`、`PASSWORD`、`KEY` 的 env key
8. **marketplace 路径**：`known_marketplaces.json` 中 `installLocation` 用 `$CLAUDE_DIR` 拼接，不用源机器绝对路径
9. **幂等性**：已存在的 marketplace/skill 只 pull 更新不重复 clone，settings.json 备份后覆盖
10. **settings.local.json**：如果存在，也一并导出（同样脱敏）
11. **全局 CLAUDE.md**：如果 `~/.claude/CLAUDE.md` 存在，用 heredoc 嵌入脚本一并恢复
12. **自定义 Agents**：`~/.claude/agents/*.md` 文件通常很小，直接 heredoc 嵌入脚本恢复到 `$CLAUDE_DIR/agents/`
13. **项目记忆（可选）**：用户选择导出的项目记忆打包进 `__SKILLS_ARCHIVE__` 段（复用同一个 tar.gz），恢复时解压到 `$CLAUDE_DIR/projects/` 对应路径。注意项目路径含源机器用户名（如 `-Users-wszf-...`），**目标机器用户名不同时需手动重命名**，脚本中输出提醒
14. **macOS 专属权限**：permissions 中的 `pbcopy`、`xcodebuild` 等命令在 Linux 上不可用，导出时提示但不自动移除（用户可能多平台共用配置）
15. **禁止用 Write 工具生成脚本**：Write 工具会在 UI 展示完整 diff（几百行），体验极差。必须用 Bash 工具的 `cat << 'EOF' > file` 写入
16. **进度追踪**：导出过程必须用 TaskCreate/TaskUpdate 创建任务列表，让用户看到执行进度。任务粒度参考：
    - 扫描配置
    - 可选组件交互
    - 分类 Skills 并交互
    - 生成安装脚本 / 推送到 Git 仓库
    - 打包并输出结果
17. **Git 仓库权限检测**：必须在扫描配置之前完成，失败则终止流程并给出修复建议
18. **Git 仓库导出幂等性**：每次 export 都覆盖仓库全部内容（保留 .git 历史），commit message 带日期便于追溯
19. **Git 仓库 install.sh 可独立运行**：不依赖 Claude Code，纯 bash + git + jq（jq 可选，无 jq 时跳过 Git Skills 的自动 clone，提示手动安装）
20. **Git 仓库脱敏**：与文件模式相同的脱敏规则，settings.json 推送到仓库前必须脱敏
21. **Git 仓库分支**：默认使用 `main` 分支，如果远程默认分支是 `master` 则跟随

## Import 流程

### 文件导入

当用户执行 `/migrate-config import <path>` 时：

1. 检查脚本文件是否存在
2. 展示脚本包含的组件摘要（grep 关键标记）
3. 确认后执行：`bash <path>`
4. 验证结果

### Git 仓库导入

当用户执行 `/migrate-config import --git <repo-url>` 时：

1. **权限检测**：执行与 Export 相同的 Git 仓库权限检测（见 Step 0），验证仓库可达性和读权限
2. **Clone 仓库**：
   ```bash
   TEMP_DIR=$(mktemp -d)
   git clone --depth 1 "$REPO_URL" "$TEMP_DIR/claude-config"
   ```
3. **检查仓库结构**：验证 `install.sh` 和 `settings.json` 是否存在
   - 如果不存在 → 报错：`该仓库不是有效的 Claude Code 配置仓库（缺少 install.sh）`
4. **展示组件摘要**：读取仓库内容，列出将要恢复的组件
5. **确认后执行**：`bash "$TEMP_DIR/claude-config/install.sh"`
6. **验证结果**：检查各组件是否正确恢复
7. **清理临时目录**：`rm -rf "$TEMP_DIR"`

## 体积优化效果

| 场景 | 全部打包（含 .git） | 网络拉取 Git Skills | 本地打包（排除 .git） |
|------|---------------------|--------------------|-----------------------|
| 2 git + 2 local skill | ~45MB | **~16KB** | ~2MB |
| 5 git + 5 local skill | ~100MB+ | **~30KB** | ~5MB |
| 纯 local skill (10个) | ~15KB | ~15KB | ~15KB |

- **网络拉取**：Git Skill 只记录 URL，脚本最小，目标机器需联网
- **本地打包（排除 .git）**：用 `--exclude='.git'` 打包，体积比含 .git 小 80-90%，离线可用
- **全部打包（含 .git）**：不推荐，体积过大

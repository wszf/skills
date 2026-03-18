---
name: migrate-config
description: 导出/导入 Claude Code 完整配置（settings、marketplaces、plugins、skills），一键在新机器上复现环境
---

# Migrate Config

一键导出/导入 Claude Code 完整运行环境。

## 用法

| 命令 | 说明 |
|------|------|
| `/migrate-config export` | 导出配置到 `~/claude-setup.sh` |
| `/migrate-config export --output <path>` | 指定输出路径 |
| `/migrate-config import <script-path>` | 在新机器上执行导入 |

## Export 流程

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

### Step 4: 生成安装脚本

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

### Step 5: 组装最终文件

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

### Step 6: 输出

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
    - 生成安装脚本
    - 打包并输出结果

## Import 流程

当用户执行 `/migrate-config import <path>` 时：

1. 检查脚本文件是否存在
2. 展示脚本包含的组件摘要（grep 关键标记）
3. 确认后执行：`bash <path>`
4. 验证结果

## 体积优化效果

| 场景 | 全部打包（含 .git） | 网络拉取 Git Skills | 本地打包（排除 .git） |
|------|---------------------|--------------------|-----------------------|
| 2 git + 2 local skill | ~45MB | **~16KB** | ~2MB |
| 5 git + 5 local skill | ~100MB+ | **~30KB** | ~5MB |
| 纯 local skill (10个) | ~15KB | ~15KB | ~15KB |

- **网络拉取**：Git Skill 只记录 URL，脚本最小，目标机器需联网
- **本地打包（排除 .git）**：用 `--exclude='.git'` 打包，体积比含 .git 小 80-90%，离线可用
- **全部打包（含 .git）**：不推荐，体积过大

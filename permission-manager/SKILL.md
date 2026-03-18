---
name: permission-manager
description: 分析 skill 权限需求、扫描项目构建工具、一键授权 — Claude Code 权限管理器
---

# Permission Manager

管理 Claude Code 的 Bash 命令权限：分析 skill 所需权限、扫描项目构建工具链、写入配置。

### 权限配置层级

| 层级 | 路径 | 作用域 |
|------|------|--------|
| 全局 | `~/.claude/settings.json` | 所有项目生效 |
| 项目 | `<project>/.claude/settings.json` | 仅当前项目生效 |

## 命令

| 命令 | 说明 |
|------|------|
| `/permission-manager` | 分析所有 skill 的权限需求汇总 |
| `/permission-manager <skill-name>` | 分析指定 skill 的权限需求 |
| `/permission-manager allow <skill-name>` | 授权指定 skill 所需权限（交互选范围） |
| `/permission-manager allow-all` | 授权所有 skill 所需权限（交互选范围） |
| `/permission-manager scan` | 扫描当前项目，检测构建工具并建议权限（交互选范围） |
| `/permission-manager status` | 显示当前已有权限统计（全局 + 项目） |
| `/permission-manager self` | 授权本 skill 自身所需的权限（交互选范围） |

> **消歧：** `allow` 后必须跟 skill 名称。授权本 skill 自身用 `self`。

---

## 首次使用检查

每次调用 `/permission-manager`（无参数）时，先检查本 skill 自身所需权限是否已授权：

1. 读取全局 + 项目两层 settings.json 的 `permissions.allow`
2. 检查是否已包含 `Bash(find ~/.claude/skills *)`、`Bash(find ~/.claude/plugins *)` 和 `Bash(cat ~/.claude/*)`
3. 若缺失，用 AskUserQuestion 询问：「检测到 permission-manager 自身权限未授权，是否先授权？」
   - **授权** — 进入 `self` 流程（选择写入范围后写入）
   - **跳过** — 继续执行原命令（可能遇到权限弹窗）
4. 若已授权，直接执行原命令

---

## 写入范围选择（通用流程）

所有涉及写入的命令（`allow`、`allow-all`、`scan`、`self`），在完成分析并展示结果后，**必须用 AskUserQuestion 让用户选择**：

选项：
1. **写入全局** — 写入 `~/.claude/settings.json`，所有项目生效
2. **写入项目** — 写入 `<当前项目>/.claude/settings.json`，仅当前项目生效
3. **跳过** — 不写入，仅查看结果

写入时：读取目标文件已有 `permissions.allow`，合并新规则（去重），写回。若目标文件不存在则创建（含 `{"permissions":{"allow":[]}}` 结构）。

---

## Part 1：Skill 权限分析

### Skill 来源

同时扫描两类 skill：

| 类型 | 路径 | 说明 |
|------|------|------|
| User Skills | `~/.claude/skills/` | 用户自建 |
| Plugin Skills | `~/.claude/plugins/cache/<org>/<plugin>/<version>/skills/` | 插件提供 |

扫描命令：
```bash
# User skills
find ~/.claude/skills -name "SKILL.md" -type f 2>/dev/null
# Plugin skills（只取每个插件最新版本）
find ~/.claude/plugins/cache -path "*/skills/*/SKILL.md" -type f 2>/dev/null
```

> 同一插件有多个版本缓存时，取路径中版本号最大的（或最后修改的）。

### 分析流程

1. 扫描上述两个路径下所有 SKILL.md
2. 从 bash 代码块和内联 `` `command` `` 中提取命令（跳过注释行和纯文档说明）
3. 转为 `Bash(command *)` 格式的权限规则
4. 与全局 + 项目两层已有规则对比，标记新增/已有

### 命令识别规则

只提取 **skill 执行步骤中明确要运行的命令**，忽略：
- 纯说明性文字中引用的命令示例
- 注释中的命令
- 管道右侧的**纯辅助命令**（如 `| head -1`、`| tail -1`、`| sort`、`| wc -l`）

> **注意**：`grep`、`awk`、`sed`、`jq`、`cat` 等命令既可能出现在管道中，也经常作为**独立主命令**使用。
> 判断规则：如果命令出现在行首或 `&&`/`;` 之后，视为主命令，需要提取权限。
> 仅当它**只出现在 `|` 右侧**且无独立使用时，才忽略。

### `/permission-manager allow <skill-name>` 执行步骤

1. 定位 skill（先查 user skills，再查 plugin skills）：
   ```bash
   find ~/.claude/skills -type d -name "<skill-name>" 2>/dev/null | head -1
   find ~/.claude/plugins/cache -path "*/<skill-name>/SKILL.md" -type f 2>/dev/null | sort | tail -1
   ```
2. 读取 SKILL.md，按上述规则提取权限
3. **黑名单校验**：移除命中「禁止授权」列表的规则，输出 `⚠ Blocked: ...`
4. 展示分析结果（新增规则 / 已有规则）
5. **AskUserQuestion 选择写入范围**（全局 / 项目 / 跳过）
6. 写入目标 settings.json，输出结果

### `/permission-manager allow-all` 执行步骤

对所有扫描到的 skill 执行上述流程，合并后统一展示，再选择写入范围。

---

## Part 2：项目构建扫描

### `/permission-manager scan` 执行步骤

#### Step 1：检测构建文件

```bash
# 分组检测，避免 zsh glob 报错
ls build.gradle build.gradle.kts gradlew pom.xml 2>/dev/null
ls package.json yarn.lock pnpm-lock.yaml bun.lockb 2>/dev/null
ls Cargo.toml go.mod go.sum 2>/dev/null
ls pyproject.toml setup.py setup.cfg requirements.txt Pipfile 2>/dev/null
ls Makefile CMakeLists.txt Dockerfile docker-compose.yml docker-compose.yaml 2>/dev/null
ls Podfile Gemfile Package.swift 2>/dev/null
find . -maxdepth 1 \( -name "*.xcodeproj" -o -name "*.xcworkspace" -o -name "*.sln" \) 2>/dev/null
```

#### Step 2：文件 → 权限映射

检测到哪个文件就加哪些规则，**未检测到的不加**：

| 检测到 | 权限规则 |
|--------|----------|
| `gradlew` | `./gradlew *` |
| `build.gradle(.kts)` | `./gradlew *`, `gradle *` |
| `pom.xml` | `mvn *` |
| `package.json` | `npm *`, `npx *` |
| `yarn.lock` | `yarn *` |
| `pnpm-lock.yaml` | `pnpm *` |
| `bun.lockb` | `bun *` |
| `Cargo.toml` | `cargo *` |
| `go.mod` | `go *` |
| `Makefile` | `make *` |
| `CMakeLists.txt` | `cmake *`, `make *` |
| `Dockerfile` | `docker *` |
| `docker-compose.y(a)ml` | `docker-compose *`, `docker compose *` |
| `pyproject.toml` | `poetry *`, `pdm *`, `python *`, `pip *` |
| `setup.py`/`setup.cfg` | `python *`, `pip *` |
| `requirements.txt` | `pip *`, `python *` |
| `Pipfile` | `pipenv *`, `python *` |
| `Podfile` | `pod *` |
| `*.xcodeproj/workspace` | `xcodebuild *`, `xcrun *`, `xcode-select *` |
| `Package.swift` | `swift *` |
| `Gemfile` | `bundle *`, `gem *`, `ruby *` |
| `*.sln` | `dotnet *` |

> 所有规则写入时自动加 `Bash(...)` 包裹。

#### Step 3：检测源码类型（补充语言工具链）

```bash
find . -maxdepth 4 -type f \( -name "*.java" -o -name "*.kt" -o -name "*.swift" -o -name "*.go" -o -name "*.rs" -o -name "*.py" -o -name "*.rb" -o -name "*.cs" -o -name "*.ts" -o -name "*.tsx" \) 2>/dev/null | head -20
```

若检测到源码但 Step 2 未覆盖对应工具链，补充：`*.java` → `java *`, `javac *`, `jar *`, `jps *` | `*.kt` → `kotlin *`, `kotlinc *` | `*.ts/*.tsx` → `npx *`, `tsc *` | 其他同理。

#### Step 4：检测脚本和 Android

```bash
# 脚本
find . -maxdepth 1 -name "*.sh" 2>/dev/null | head -5
ls -d scripts/ bin/ 2>/dev/null
# Android
find . -maxdepth 3 -name "AndroidManifest.xml" 2>/dev/null | head -1
```

- 有 `.sh` 脚本 → `bash *`, `sh *`
- 有 AndroidManifest.xml → `adb *`, `adb shell *`, `emulator *`, `sdkmanager *`, `avdmanager *`

#### Step 5：Claude Code 常用命令（始终建议）

以下是 Claude Code 日常工作中高频使用的 Bash 命令，按类别分组，**始终建议添加**（不依赖项目类型）：

**搜索和查看**：
| 命令 | 说明 |
|------|------|
| `grep *` | 文件内容搜索（Claude 经常通过 Bash 而非 Grep 工具调用） |
| `cat *` | 查看文件内容（补充 Read 工具覆盖不到的场景） |
| `head *` | 查看文件头部 |
| `tail *` | 查看文件尾部 |
| `jq *` | JSON 解析处理 |
| `wc *` | 统计行数/字数 |

**文件操作**：
| 命令 | 说明 |
|------|------|
| `mkdir -p *` | 创建目录（递归） |
| `cp *` | 复制文件 |
| `mv *` | 移动/重命名文件 |
| `chmod +x *` | 添加可执行权限 |
| `ln -s *` | 创建符号链接 |
| `touch *` | 创建空文件 |

**路径和系统工具**：
| 命令 | 说明 |
|------|------|
| `basename *` | 提取文件名 |
| `dirname *` | 提取目录路径 |
| `readlink *` | 解析符号链接 |
| `which *` | 查找命令路径 |
| `du *` | 查看文件/目录大小 |
| `uname *` | 系统信息 |

**归档和编码**：
| 命令 | 说明 |
|------|------|
| `tar *` | 打包/解包（skill 导出常用） |
| `base64 *` | Base64 编解码 |
| `unzip *` | 解压 ZIP |

**网络和端口**：
| 命令 | 说明 |
|------|------|
| `curl *` | HTTP 请求 |
| `lsof -i:*` | 端口占用检查 |

**文本处理**：
| 命令 | 说明 |
|------|------|
| `sort *` | 排序 |
| `sed *` | 流式文本替换（脚本生成常用） |

**Git 补充**（scan 时检测到 `.git` 目录则建议）：
| 命令 | 说明 |
|------|------|
| `git status*` | 查看工作区状态 |
| `git diff *` | 查看差异 |
| `git remote *` | 远程仓库管理 |
| `git stash *` | 暂存修改 |
| `git show *` | 查看提交详情 |
| `git fetch *` | 拉取远程更新 |
| `git clone *` | 克隆仓库 |
| `git branch *` | 分支管理 |
| `git tag *` | 标签管理 |

**npm 补充**（检测到 `package.json` 时额外建议）：
| 命令 | 说明 |
|------|------|
| `npm run *` | 执行 npm scripts |
| `npx *` | 执行 npx 命令 |

> 上述命令在 scan 时统一展示，用户可在交互中取消不需要的。
> Git 补充命令仅在项目有 `.git` 目录时建议。
> npm 补充命令仅在检测到 `package.json` 时建议。

#### Step 6：去重 & 展示结果

1. 读取全局 `~/.claude/settings.json` 和项目 `<project>/.claude/settings.json` 的已有权限
2. **黑名单校验**
3. 输出扫描结果：检测到的工具、新增规则、已有规则

#### Step 7：交互选择写入范围

用 AskUserQuestion 让用户选择：**写入全局** / **写入项目** / **跳过**。

选择后执行写入，输出最终结果。

---

## Part 3：`status` 和 `self`

### `/permission-manager status`

同时读取两个层级，分别输出：

```
全局权限 (~/.claude/settings.json): N 条
  [按类别分组统计]

项目权限 (<project>/.claude/settings.json): M 条
  [按类别分组统计]
  （若文件不存在则显示：未配置）
```

### `/permission-manager self`

将本 skill 自身所需权限写入（交互选范围）：

```json
["Bash(find ~/.claude/skills *)", "Bash(find ~/.claude/plugins *)", "Bash(cat ~/.claude/*)"]
```

---

## 禁止授权（黑名单）

写入前必须校验，命中以下模式的规则**不写入**，并输出 `⚠ Blocked`：

| 模式 | 原因 |
|------|------|
| `rm *` / `rm -rf *` | 文件删除 |
| `sudo *` | 提权 |
| `chmod 777 *` | 权限全开 |
| `kill -9 *` / `pkill *` | 强杀进程 |
| `eval *` | 动态执行 |
| `git push --force *` | 覆盖远程 |
| `git reset --hard *` | 丢失修改 |

---

## 输出格式

所有命令输出遵循统一格式：

```
[标题行]

检测到:
  ✓ [item]

新增规则 (N 条):
  + Bash(xxx *)

已有规则 (全局/项目):
  ✓ Bash(xxx *)

[⚠ Blocked: Bash(rm *) — 文件删除]

→ 请选择写入范围（AskUserQuestion）
```

## 注意事项

- 权限更新后需新会话或 `/clear` 生效
- 只增不删，不会移除用户已有权限
- 使用 `*` 通配符匹配可变参数部分

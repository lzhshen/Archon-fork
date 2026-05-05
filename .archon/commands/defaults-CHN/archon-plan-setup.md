---
description: 计划执行前的准备 -- 读取计划、确保分支就绪、编写上下文产物
argument-hint: <path/to/plan.md>
---

# 计划准备

**计划**: $ARGUMENTS
**工作流 ID**: $WORKFLOW_ID

---

## 任务目标

为计划实现准备一切所需条件：
1. 读取并解析计划（包括范围限制）
2. 确保处于正确的分支
3. 为后续步骤编写全面的上下文产物

**本步骤不执行任何实现** -- 仅设置环境。
**本步骤不创建 PR** -- PR 将在实现完成后由 `archon-finalize-pr` 创建。

---

## 阶段 1: 加载 - 读取计划

### 1.1 定位计划文件

**按以下顺序检查：**

1. **若提供了 `$ARGUMENTS`**: 使用该路径
2. **若计划已存在于工作流产物中**: 使用 `$ARTIFACTS_DIR/plan.md`

```bash
# 检查计划是否由本工作流中的 archon-create-plan 创建
if [ -f "$ARTIFACTS_DIR/plan.md" ]; then
  PLAN_PATH="$ARTIFACTS_DIR/plan.md"
  echo "使用工作流中的计划: $PLAN_PATH"
elif [ -n "$ARGUMENTS" ] && [ -f "$ARGUMENTS" ]; then
  PLAN_PATH="$ARGUMENTS"
  echo "使用参数指定的计划: $PLAN_PATH"
else
  echo "错误: 未找到计划"
  exit 1
fi
```

### 1.2 加载计划文件

读取计划文件：

```bash
cat $PLAN_PATH
```

若 `$ARGUMENTS` 是 GitHub issue URL 或编号（如 `#123`），则改为获取 issue 正文。

### 1.3 提取关键信息

从计划中识别并提取：

| 字段 | 查找位置 | 示例 |
|------|----------|------|
| **标题** | 第一个 `#` 标题或"概要"部分 | "Discord 平台适配器" |
| **概要** | "概要"或"功能描述"部分 | 1-2 句话概述 |
| **需修改的文件** | "需修改的文件"或"任务"部分 | CREATE/UPDATE 文件列表 |
| **验证命令** | "验证命令"或"验证策略"部分 | `bun run type-check` 等 |
| **验收标准** | "验收标准"部分 | 清单项 |
| **不构建（范围限制）** | "不构建"、"范围限制"或"范围外"部分 | 明确排除项 |

**关键**: "不构建"部分定义了**有意排除**的范围。必须捕获此内容并传递给审查代理，以免他们将有意排除项标记为缺陷。

### 1.4 推导分支名称

根据计划标题创建分支名称：

```
feature/{slug}
```

其中 `{slug}` 为标题小写、空格替换为连字符，最多 50 个字符。

示例：
- "Discord Platform Adapter" -> `feature/discord-platform-adapter`
- "ESLint/Prettier Integration" -> `feature/eslint-prettier-integration`

**阶段 1 检查点:**

- [ ] 计划文件已加载且可读
- [ ] 关键信息已提取
- [ ] 分支名称已推导

---

## 阶段 2: 准备 - Git 状态

### 2.1 检查当前状态

```bash
git branch --show-current
git status --porcelain
git remote get-url origin
```

### 2.2 确定仓库信息

从远程 URL 提取 owner/repo，用于后续创建 PR：

```bash
gh repo view --json nameWithOwner -q .nameWithOwner
```

### 2.3 分支决策

按以下顺序评估（第一个匹配的条件生效）：

```text
是否在 Worktree 中？
  -- 是 -> 保持当前分支不变。不要切换分支。不要创建
           新分支。隔离系统已设置正确的
           分支；任何偏离都会操作错误的代码。
           记录日志: "使用 worktree 分支: {name}"
  |
是否在 $BASE_BRANCH 上？（main、master 或配置的基础分支）
  |
  -- 工作目录是否干净？
     -- 是 -> 创建并切换: `git checkout -b {branch-name}`
     |       （仅在 worktree 外适用 -- 如手动使用 CLI）
     -- 否 -> 停止: "在 $BASE_BRANCH 上有未提交的变更。请先暂存或提交。"
  |
是否在其他分支上？
  |
  -- 是否匹配此计划的预期分支？
     -- 是 -> 使用它，记录 "使用现有分支: {name}"
     -- 否 -> 停止: "当前在分支 {X}，预期 {Y}。请切换分支或调整计划。"
```

### 2.4 与远程同步

```bash
git fetch origin
git rebase origin/$BASE_BRANCH || git merge origin/$BASE_BRANCH
```

若发生冲突，停止并报错："与 $BASE_BRANCH 存在合并冲突。请手动解决。"

### 2.5 推送分支（若已有提交）

若分支上已有提交：
```bash
git push -u origin HEAD
```

若尚无提交（全新分支），跳过推送 -- 将在实现完成后进行。

**阶段 2 检查点:**

- [ ] 已处于正确分支
- [ ] 无未提交的变更
- [ ] 已与基础分支同步

---

## 阶段 3: 产物 - 编写上下文文件

### 3.1 创建产物目录

```bash
```

### 3.2 编写上下文产物

写入 `$ARTIFACTS_DIR/plan-context.md`：

```markdown
# 计划上下文

**生成时间**: {YYYY-MM-DD HH:MM}
**工作流 ID**: $WORKFLOW_ID
**计划来源**: $ARGUMENTS

---

## 分支

| 字段 | 值 |
|------|-----|
| **分支** | {branch-name} |
| **基础分支** | {base-branch} |

---

## 计划概要

**标题**: {extracted-title}

**概述**: {从计划中提取的 1-2 句话概要}

---

## 需修改的文件

{复制计划中的"需修改的文件"表格，或列出提取的文件}

| 文件 | 操作 |
|------|------|
| `src/example.ts` | 创建 |
| `src/other.ts` | 更新 |

---

## 不构建（范围限制）

**审查人员注意**: 这些项目已**有意排除**在范围之外。请勿将其标记为缺陷或缺失功能。

{从计划的"不构建"、"范围限制"或"范围外"部分复制}

- {明确排除项 1 及理由}
- {明确排除项 2 及理由}

{若计划中无明确排除项: "计划中未定义明确的范围限制。"}

---

## 验证命令

{从计划的"验证命令"部分复制}

```bash
bun run type-check
bun run lint
bun test
bun run build
```

---

## 验收标准

{从计划的"验收标准"部分复制}

- [ ] 标准 1
- [ ] 标准 2
- [ ] ...

---

## 参考模式

{从计划的"参考模式"部分复制关键文件引用}

| 模式 | 源文件 | 行号 |
|------|--------|------|
| {模式名称} | `src/example.ts` | 10-50 |

---

## 后续步骤

1. `archon-confirm-plan` - 验证模式是否仍然存在
2. `archon-implement-tasks` - 执行计划
3. `archon-validate` - 运行完整验证
4. `archon-finalize-pr` - 创建 PR 并标记为就绪
```

**阶段 3 检查点:**

- [ ] 产物目录已创建
- [ ] `plan-context.md` 已写入所有部分
- [ ] "不构建"部分已捕获（即使为空）

---

## 阶段 4: 输出 - 向用户报告

```markdown
## 计划准备完成

**计划**: `$ARGUMENTS`
**工作流 ID**: `$WORKFLOW_ID`

### 分支

| 字段 | 值 |
|------|-----|
| 分支 | `{branch-name}` |
| 基础分支 | `{base-branch}` |

### 计划概要

**{plan-title}**

{1-2 句话概述}

### 范围

- {N} 个文件需创建
- {M} 个文件需更新
- {K} 个明确排除项已捕获

### 产物

上下文已写入: `$ARTIFACTS_DIR/plan-context.md`

### 下一步

继续执行 `archon-confirm-plan` 以验证计划的调研仍然有效。
```

---

## 错误处理

### 计划文件未找到

```
错误: 未找到计划: $ARGUMENTS

请验证路径是否存在后重试。
```

### 基础分支上有未提交的变更

```
错误: 基础分支上有未提交的变更

选项:
1. 暂存变更: `git stash`
2. 提交变更: `git add . && git commit -m "WIP"`
3. 丢弃变更: `git checkout .`

然后重试。
```

### 合并冲突

```
错误: 与 $BASE_BRANCH 存在合并冲突

请手动解决冲突:
1. `git status` 查看冲突
2. 编辑冲突文件
3. `git add <已解决的文件>`
4. `git rebase --continue`

然后重试。
```

---

## 成功标准

- **PLAN_LOADED**: 计划文件已读取和解析
- **SCOPE_LIMITS_CAPTURED**: "不构建"部分已提取（即使为空）
- **BRANCH_READY**: 处于正确分支，已与基础分支同步
- **ARTIFACT_WRITTEN**: `plan-context.md` 包含所有必需部分（包括范围限制）

---
description: 基于当前分支和实施上下文创建 Pull Request
argument-hint: [base-branch] (default: auto-detected from config or repo)
---

# 创建 Pull Request

**目标分支覆盖**: $ARGUMENTS
**默认目标分支**: $BASE_BRANCH

> 如果上方参数提供了目标分支，请将其用于 `--base`。否则使用默认目标分支。

---

## 预检: 检查已有 PR

从当前分支名称或上下文中提取 issue 编号（例如 `fix/issue-580` → `580`）。

```bash
BRANCH=$(git branch --show-current)
ISSUE_NUM=$(echo "$BRANCH" | grep -oE '[0-9]+' | tail -1)
```

如果找到 issue 编号，搜索已引用该编号的开放 PR：

```bash
gh pr list \
  --search "Fixes #${ISSUE_NUM} OR Closes #${ISSUE_NUM}" \
  --state open \
  --json number,url,headRefName
```

**如果返回了匹配的 PR**: 到此为止，报告已有 PR 的 URL，**不要**继续到阶段 2 或阶段 3。

```
已找到 issue #${ISSUE_NUM} 的 PR: [url]
跳过 PR 创建。
```

**如果未找到匹配**（或无法提取 issue 编号）: 继续到阶段 1。

---

## 阶段 1: 收集上下文

### 1.1 检查 Git 状态

```bash
git branch --show-current
git status --short
git log origin/$BASE_BRANCH..HEAD --oneline
```

### 1.2 检查实施报告

查找最近的实施报告：

```bash
ls -t $ARTIFACTS_DIR/../reports/*-report.md 2>/dev/null | head -1
```

如果找到，读取并提取：
- 实施内容概要
- 变更文件
- 验证结果
- 与计划的任何偏差

### 1.3 获取提交摘要

```bash
git log origin/$BASE_BRANCH..HEAD --pretty=format:"- %s"
```

---

## 阶段 2: 准备分支

### 2.1 确保所有变更已提交

如果有未提交的变更：

```bash
git status --porcelain
```

**如果有未提交内容**:

1. 仅暂存属于本次变更的源文件 -- 绝不使用 `git add -A`、`git add .` 或 `git add -u`。逐一列出文件名：
   ```bash
   git add path/to/file1 path/to/file2 ...
   git status --porcelain  # 验证没有多余内容被暂存
   ```
2. **绝不暂存** 临时/审查/PR正文制品，即使它们出现在 `git status` 中：
   - `.pr-body.md`, `pr-body.md`, `*.scratch.md`, `*.tmp.md`
   - `review/`, 根目录下的 `*-report.md`
   - `$ARTIFACTS_DIR` 下的任何内容
3. 提交: `git commit -m "Final changes before PR"`

### 2.2 推送分支

```bash
git push -u origin HEAD
```

---

## 阶段 3: 创建 PR

### 3.1 检查 PR 模板

查找项目的 PR 模板，路径为 `.github/pull_request_template.md`、`.github/PULL_REQUEST_TEMPLATE.md` 或 `docs/PULL_REQUEST_TEMPLATE.md`。读取找到的文件。

**如果找到模板**: 使用其结构，用实施报告和提交中的详情填充**每个章节**。不要跳过章节或留下占位符。

**如果没有模板**，使用以下格式：

```markdown
## 概要

[来自实施报告或提交的简要描述]

## 变更内容

[来自实施报告"变更文件"章节，或来自提交]
- file1.ts - 描述
- file2.ts - 描述

## 验证

[来自实施报告"验证结果"章节]
- [x] 类型检查通过
- [x] 代码检查通过
- [x] 测试通过
- [x] 构建成功

## 测试说明

[已进行的手动测试或集成测试结果]

---

[如果来自 GitHub issue，添加: Closes #XXX]
```

### 3.2 确定 PR 标题

**标题**: 简洁、祈使语气
- 来自实施报告概要，或
- 来自提交消息

### 3.3 创建 PR

```bash
# 将正文写入文件以避免 shell 转义问题
cat > $ARTIFACTS_DIR/pr-body.md <<'EOF'
[上方的正文内容]
EOF

gh pr create \
  --title "[标题]" \
  --body-file $ARTIFACTS_DIR/pr-body.md \
  --base $BASE_BRANCH
```

或者如果内容较简单：

```bash
gh pr create --fill --base $BASE_BRANCH
```

创建 PR 后，捕获其标识信息供后续步骤使用。仅在 PR 创建成功后才写入制品 -- 绝不保存来自已有 PR 的过期数据：

```bash
# 创建 PR 后，捕获并保存 PR 编号供后续步骤使用
# 重要: 仅在确认 PR 创建成功后写入制品
if gh pr view --json number,url -q '.number,.url' > /dev/null 2>&1; then
  PR_NUMBER=$(gh pr view --json number -q '.number')
  PR_URL=$(gh pr view --json url -q '.url')
  echo "$PR_NUMBER" > "$ARTIFACTS_DIR/.pr-number"
  echo "$PR_URL" > "$ARTIFACTS_DIR/.pr-url"
else
  echo "WARNING: 无法确认 PR 创建; 跳过 .pr-number/.pr-url 制品"
fi
```

---

## 阶段 4: 输出

报告结果：

```markdown
## PR 已创建

**URL**: [PR URL]
**分支**: [branch-name] → [base-branch]
**标题**: [PR 标题]

### 概要
[PR 内容的简要概要]

### 后续步骤
1. 如有需要，请求代码审查
2. 处理 CI 失败（如有）
3. 获得批准后合并
```

---

## 错误处理

### 没有可推送的提交

```
origin/$BASE_BRANCH 和 HEAD 之间没有提交。
没有可创建 PR 的内容。
```

### 分支已有 PR

```bash
gh pr view --web
```

打开已有 PR 而非创建重复的。

### 推送失败

1. 检查分支是否存在于远程: `git ls-remote --heads origin [branch]`
2. 如果有冲突: `git pull --rebase origin $BASE_BRANCH` 然后重试推送
3. 如果是权限问题: 检查 GitHub 访问权限

---
description: 提交变更、使用模板创建 PR、标记为准备就绪待审查
argument-hint: (no arguments - reads from workflow artifacts)
---

# 完成 Pull Request

**工作流 ID**: $WORKFLOW_ID

---

## 你的任务

完成实现并创建 PR:
1. 提交所有变更
2. 推送到远程仓库
3. 使用项目模板（如存在）创建 PR
4. 将 PR 标记为准备就绪待审查

---

## 阶段 1: 加载 - 收集上下文

### 1.1 加载工作流产物

```bash
cat $ARTIFACTS_DIR/plan-context.md
cat $ARTIFACTS_DIR/implementation.md
cat $ARTIFACTS_DIR/validation.md
```

提取以下信息:
- 计划标题和摘要
- 分支名称
- 变更的文件
- 编写的测试
- 验证结果
- 与计划的偏差（如有）

### 1.2 检查 PR 模板

**重要提示**: 始终先检查项目的 PR 模板。在 `.github/pull_request_template.md`、`.github/PULL_REQUEST_TEMPLATE.md` 或 `docs/PULL_REQUEST_TEMPLATE.md` 中查找。读取存在的那个文件。

**如果找到模板**: 使用其作为结构，用实现细节填充**每个章节**。
**如果没有模板**: 使用阶段 3 中定义的默认格式。

### 1.3 检查已有 PR

```bash
gh pr list --head $(git branch --show-current) --json number,url,state
```

**如果 PR 已存在**: 将更新现有 PR 而非创建新的。
**如果没有 PR**: 将创建新的 PR。

**阶段 1 检查点:**

- [ ] 产物已加载
- [ ] 模板已确认（或使用默认格式）
- [ ] 已知现有 PR 状态

---

## 阶段 2: 提交 - 暂存并提交变更

### 2.1 检查 Git 状态

```bash
git status --porcelain
```

### 2.2 暂存变更

仅暂存你实际编辑的实现文件——切勿使用 `git add -A`、`git add .` 或 `git add -u`。逐个列出文件名:

```bash
git add path/to/file1 path/to/file2 ...
git status --porcelain  # 验证没有其他文件被暂存
```

**切勿暂存**临时/审查/PR 正文产物，即使它们出现在 `git status` 中:

- `.pr-body.md`、`pr-body.md`、`*.scratch.md`、`*.tmp.md`
- 仓库根目录下的 `review/`、`*-report.md`
- `$ARTIFACTS_DIR` 下的任何内容

**检查已暂存的文件**——确保不包含敏感文件（`.env`、凭据）和临时产物:

```bash
git diff --cached --name-only
```

### 2.3 创建提交

创建描述性的提交信息:

```bash
git commit -m "{实现摘要}

- {关键变更 1}
- {关键变更 2}
- {关键变更 3}

{如果来自计划/issue: Implements #{number}}
"
```

### 2.4 推送到远程仓库

```bash
git push origin HEAD
```

**阶段 2 检查点:**

- [ ] 所有变更已暂存
- [ ] 未包含敏感文件
- [ ] 提交已创建
- [ ] 已推送到远程仓库

---

## 阶段 3: 创建/更新 - Pull Request

### 3.1 准备 PR 正文

**如果项目有 PR 模板**，用实现细节填充每个章节:
- 用实际内容替换占位文本
- 根据已完成的工作填写复选框
- 保持模板的原始结构不变

**如果没有模板**，使用以下默认格式:

```markdown
## 摘要

{来自计划摘要的简要描述}

## 变更内容

{来自 implementation.md 的"变更文件"章节}

| 文件 | 操作 | 描述 |
|------|------|------|
| `src/x.ts` | CREATE | {功能说明} |
| `src/y.ts` | UPDATE | {变更说明} |

## 测试

{来自 implementation.md 的"编写的测试"章节}

- `src/x.test.ts` - {测试描述}
- `src/y.test.ts` - {测试描述}

## 验证

{来自 validation.md}

- [x] 类型检查通过
- [x] 代码检查通过
- [x] 格式检查通过
- [x] 所有测试通过 ({N} 个测试)
- [x] 构建成功

## 实现备注

{如有与计划的偏差:}
### 与计划的偏差

{列出偏差及其原因}

{如遇到的问题:}
### 已解决的问题

{列出问题及其解决方案}

---

**计划**: `{plan-source-path}`
**工作流 ID**: `$WORKFLOW_ID`
```

### 3.2 创建或更新 PR

**如果没有现有 PR**，创建一个:

```bash
# 将准备好的正文写入文件以避免 shell 转义问题
cat > $ARTIFACTS_DIR/pr-body.md <<'EOF'
{准备好的正文}
EOF

gh pr create \
  --title "{plan-title}" \
  --body-file $ARTIFACTS_DIR/pr-body.md \
  --base $BASE_BRANCH
```

**如果 PR 已存在**，更新它:

```bash
gh pr edit {pr-number} --body-file $ARTIFACTS_DIR/pr-body.md
```

### 3.3 确保准备就绪待审查

如果 PR 是以草稿状态创建的，标记为准备就绪:

```bash
gh pr ready {pr-number} 2>/dev/null || true
```

### 3.4 获取 PR 信息

```bash
gh pr view --json number,url,headRefName,baseRefName
```

### 3.5 写入 PR 编号注册

将 PR 编号写入文件，供下游审查步骤使用:

```bash
PR_NUMBER=$(gh pr view --json number -q '.number')
PR_URL=$(gh pr view --json url -q '.url')
echo "$PR_NUMBER" > $ARTIFACTS_DIR/.pr-number
echo "$PR_URL" > $ARTIFACTS_DIR/.pr-url
```

**阶段 3 检查点:**

- [ ] PR 已创建或更新
- [ ] PR 正文使用了模板（如有）
- [ ] PR 已准备就绪待审查
- [ ] PR URL 已获取
- [ ] PR 编号注册已写入

---

## 阶段 4: 产物 - 写入 PR 就绪状态

### 4.1 写入最终产物

写入 `$ARTIFACTS_DIR/pr-ready.md`:

```markdown
# PR 准备就绪待审查

**生成时间**: {YYYY-MM-DD HH:MM}
**工作流 ID**: $WORKFLOW_ID

---

## Pull Request

| 字段 | 值 |
|------|-----|
| **编号** | #{number} |
| **URL** | {url} |
| **分支** | `{head}` → `{base}` |
| **状态** | 准备就绪待审查 |

---

## 提交

**哈希值**: {commit-sha}
**提交信息**: {commit-message-first-line}

---

## PR 中的文件

{来自 git diff --name-only origin/$BASE_BRANCH}

| 文件 | 状态 |
|------|------|
| `src/x.ts` | Added |
| `src/y.ts` | Modified |

---

## PR 描述

{是否使用了模板或默认格式}

- 使用了模板: {yes/no}
- 模板路径: {如使用则填写路径}

---

## 下一步

继续 PR 审查工作流:
1. `archon-pr-review-scope`
2. `archon-sync-pr-with-main`
3. 审查代理（并行）
4. `archon-synthesize-review`
5. `archon-implement-review-fixes`
```

**阶段 4 检查点:**

- [ ] PR 就绪产物已写入

---

## 阶段 5: 输出 - 报告状态

```markdown
## PR 准备就绪待审查 ✅

**工作流 ID**: `$WORKFLOW_ID`

### Pull Request

| 字段 | 值 |
|------|-----|
| PR | #{number} |
| URL | {url} |
| 分支 | `{branch}` → `{base}` |
| 状态 | 🟢 准备就绪待审查 |

### 提交

```
{commit-sha-short} {commit-message-first-line}
```

### 变更文件

- {N} 个文件添加
- {M} 个文件修改
- {K} 个文件删除

### 验证摘要

| 检查项 | 状态 |
|--------|------|
| 类型检查 | ✅ |
| 代码检查 | ✅ |
| 测试 | ✅ ({N} 个通过) |
| 构建 | ✅ |

### 产物

状态已写入: `$ARTIFACTS_DIR/pr-ready.md`

### 下一步

正在进入全面的 PR 审查。
```

---

## 错误处理

### 没有内容需要提交

如果没有变更需要提交:

```markdown
ℹ️ 没有变更需要提交

所有变更已经提交。继续更新 PR 描述。
```

### 推送失败

```bash
# 如果分支经过了 rebase，尝试强制推送
git push --force-with-lease origin HEAD
```

如果仍然失败:
```
❌ 推送失败

请检查:
1. 分支保护规则
2. 仓库的推送权限
3. 远程分支状态: `git fetch origin && git status`
```

### PR 未找到

```
❌ 未找到 PR: #{number}

草稿 PR 可能已被关闭或删除。请创建新的 PR:
`gh pr create --title "..." --body "..."`
```

### 模板解析

如果模板结构复杂，难以完全填充:
- 尽可能多地使用模板内容
- 在相关章节中添加实现细节
- 在底部注明: "某些模板章节可能需要手动完善"

---

## 成功标准

- **CHANGES_COMMITTED**: 所有变更已包含在一个提交中
- **PUSHED**: 分支已推送到远程仓库
- **PR_UPDATED**: PR 描述反映了实现内容
- **PR_READY**: 草稿状态已移除
- **ARTIFACT_WRITTEN**: PR 就绪产物已创建

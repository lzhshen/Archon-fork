---
description: 实施评审中的 CRITICAL 和 HIGH 级别修复，添加测试，报告剩余问题
argument-hint: (none - reads from consolidated review artifact)
---

# 实施评审修复

---

## 重要提示：输出行为

**你的输出将作为 GitHub 评论发布。** 请将工作输出保持精简：
- 不要逐步叙述操作（"现在我来读取文件..."、"让我检查一下..."）
- 不要输出冗长的进度更新
- 仅在最后输出最终的结构化报告
- 使用 TodoWrite 工具静默跟踪进度

---

## 你的任务

阅读合并评审产物并实施所有 CRITICAL 和 HIGH 优先级的修复。如果缺少测试，为修复的代码添加测试。提交并推送更改。报告哪些已修复、哪些未修复（及原因），并为剩余项目建议后续 Issue。

**输出产物**：`$ARTIFACTS_DIR/review/fix-report.md`
**Git 操作**：提交并推送修复到 PR 分支
**GitHub 操作**：发布修复报告评论

---

## 阶段 1：加载 - 获取修复清单

### 1.1 从注册文件获取 PR 编号

```bash
PR_NUMBER=$(cat $ARTIFACTS_DIR/.pr-number)

# 获取 PR 的 head 分支名称
HEAD_BRANCH=$(gh pr view $PR_NUMBER --json headRefName --jq '.headRefName')
echo "PR: $PR_NUMBER, Branch: $HEAD_BRANCH"
```

### 1.2 切换到 PR 分支

**关键：在 PR 的实际分支上工作，而不是新分支。**

```bash
# 拉取并切换到 PR 分支
git fetch origin $HEAD_BRANCH
git checkout $HEAD_BRANCH
git pull origin $HEAD_BRANCH
```

### 1.3 阅读合并评审报告

```bash
cat $ARTIFACTS_DIR/review/consolidated-review.md
```

提取：
- 所有 CRITICAL 问题及其修复方案
- 所有 HIGH 问题及其修复方案
- MEDIUM 问题（用于报告）
- LOW 问题（用于报告）

### 1.4 阅读各独立产物获取详情

如果合并报告中没有完整的修复代码，阅读原始产物：

```bash
cat $ARTIFACTS_DIR/review/code-review-findings.md
cat $ARTIFACTS_DIR/review/error-handling-findings.md
cat $ARTIFACTS_DIR/review/test-coverage-findings.md
cat $ARTIFACTS_DIR/review/docs-impact-findings.md
```

### 1.5 检查当前 Git 状态

```bash
git status --porcelain
git branch --show-current
```

确认你在正确的 PR 分支上（应为 `$HEAD_BRANCH`）。

**阶段 1 检查点：**
- [ ] PR 编号已确认
- [ ] 在正确的 PR 分支上（不是 base 分支，不是新分支）
- [ ] 合并评审报告已加载
- [ ] CRITICAL/HIGH 问题已提取

---

## 阶段 2：实施 - 应用修复

### 2.1 处理每个 CRITICAL 问题

1. **读取文件**
2. **应用推荐的修复**
3. **验证修复可编译**：`bun run type-check`
4. **跟踪**：记录所做的更改

### 2.2 处理每个 HIGH 问题

与 CRITICAL 相同的流程。

### 2.3 处理测试覆盖缺口

如果 test-coverage-agent 发现修复的代码缺少测试：

1. **创建/更新测试文件**
2. **为修复添加测试**
3. **验证测试通过**：`bun test {file}`

### 2.4 处理无法修复的问题

如果修复无法应用：
- **冲突**：代码自评审以来已更改
- **复杂**：需要架构层面的更改
- **不明确**：建议含糊不清
- **风险**：修复可能破坏其他功能

清楚地记录原因。

**阶段 2 检查点：**
- [ ] 所有 CRITICAL 修复已尝试
- [ ] 所有 HIGH 修复已尝试
- [ ] 已为修复添加测试
- [ ] 无法修复的问题已记录

---

## 阶段 3：验证 - 确认修复

### 3.1 类型检查

```bash
bun run type-check
```

必须通过。如果未通过，修复类型错误。

### 3.2 代码检查

```bash
bun run lint
```

修复引入的任何 lint 错误。

### 3.3 运行测试

```bash
bun test
```

所有测试必须通过。如果新测试失败，修复它们。

### 3.4 构建检查

```bash
bun run build
```

必须成功。

**阶段 3 检查点：**
- [ ] 类型检查通过
- [ ] 代码检查通过
- [ ] 所有测试通过
- [ ] 构建成功

---

## 阶段 4：提交和推送 - 保存并推送更改

### 4.1 暂存更改

**仅**暂存你在应用评审修复时实际编辑的文件——绝不使用 `git add -A`、`git add .` 或 `git add -u`。按名称逐个列出：

```bash
git add path/to/file1 path/to/file2 ...
git status --porcelain  # 确认没有暂存 scratch/review/PR-body 文件
```

**绝不暂存**：

- `.pr-body.md`、`pr-body.md`、`*.scratch.md`、`*.tmp.md`
- `review/`、`*-report.md`（仓库根目录下的）
- `$ARTIFACTS_DIR` 下的任何内容（评审产物存放在那里，不在工作树中）

### 4.2 提交

```bash
git commit -m "fix: Address review findings (CRITICAL/HIGH)

Fixes applied:
- {修复简要列表}

Tests added:
- {新测试列表（如有）}

Skipped (see review artifacts):
- {无法修复的简要列表（如有）}

Review artifacts: $ARTIFACTS_DIR/review/"
```

### 4.3 推送到 PR 分支

**将修复推送到 PR 分支，使其出现在 PR 中。**

```bash
git push origin $HEAD_BRANCH
```

如果推送因分歧而失败：
```bash
git pull --rebase origin $HEAD_BRANCH
git push origin $HEAD_BRANCH
```

**阶段 4 检查点：**
- [ ] 更改已提交
- [ ] 更改已推送到 PR 分支
- [ ] PR 现在显示修复内容

---

## 阶段 5：生成 - 创建修复报告

写入 `$ARTIFACTS_DIR/review/fix-report.md`：

```markdown
# Fix Report: PR #{number}

**Date**: {ISO timestamp}
**Status**: {COMPLETE | PARTIAL}
**Branch**: {HEAD_BRANCH}

---

## Summary

{2-3 句话概述已应用的修复}

---

## Fixes Applied

### CRITICAL Fixes ({n}/{total})

| Issue | Location | Status | Details |
|-------|----------|--------|---------|
| {title} | `file:line` | ✅ FIXED | {what was done} |
| {title} | `file:line` | ❌ SKIPPED | {why} |

---

### HIGH Fixes ({n}/{total})

| Issue | Location | Status | Details |
|-------|----------|--------|---------|
| {title} | `file:line` | ✅ FIXED | {what was done} |

---

## Tests Added

| Test File | Test Cases | For Issue |
|-----------|------------|-----------|
| `src/x.test.ts` | `it('should...')` | {issue title} |

---

## Not Fixed (Requires Manual Action)

### {Issue Title}

**Severity**: {CRITICAL/HIGH}
**Location**: `{file}:{line}`
**Reason Not Fixed**: {reason}

**Suggested Action**:
{What the user should do}

---

## MEDIUM Issues (User Decision Required)

| Issue | Location | Options |
|-------|----------|---------|
| {title} | `file:line` | Fix now / Create issue / Skip |

---

## LOW Issues (For Consideration)

| Issue | Location | Suggestion |
|-------|----------|------------|
| {title} | `file:line` | {brief suggestion} |

---

## Suggested Follow-up Issues

| Issue Title | Priority | Related Finding |
|-------------|----------|-----------------|
| "{title}" | P{1/2/3} | {which finding} |

---

## Validation Results

| Check | Status |
|-------|--------|
| Type check | ✅ |
| Lint | ✅ |
| Tests | ✅ ({n} passed) |
| Build | ✅ |

---

## Git Status

- **Branch**: {HEAD_BRANCH}
- **Commit**: {commit-hash}
- **Pushed**: ✅ Yes
```

**阶段 5 检查点：**
- [ ] 修复报告已创建
- [ ] 所有修复已记录

---

## 阶段 6：发布 - GitHub 评论

### 6.1 发布修复报告

```bash
gh pr comment {number} --body "$(cat <<'EOF'
# Auto-Fix Report

**Status**: {COMPLETE | PARTIAL}
**Pushed**: Changes pushed to PR

---

## Fixes Applied

| Severity | Fixed | Skipped |
|----------|-------|---------|
| CRITICAL | {n} | {n} |
| HIGH | {n} | {n} |

### What Was Fixed

{For each fix:}
- **{title}** (`{file}:{line}`) - {brief description}

### Tests Added

{If any:}
- `{test-file}`: {n} new test cases

---

## Not Fixed (Manual Action Required)

{If any:}
- **{title}** (`{file}`) - {reason}

---

## MEDIUM Issues (Your Decision)

{If any:}
| Issue | Options |
|-------|---------|
| {title} | Fix now / Create issue / Skip |

---

## Suggested Follow-up Issues

{If any items should become issues:}
1. **{Issue Title}** (P{1/2/3}) - {brief description}

---

## Validation

Type check | Lint | Tests | Build

---

*Auto-fixed by Archon comprehensive-pr-review workflow*
*Fixes pushed to branch `{HEAD_BRANCH}`*
EOF
)"
```

**阶段 6 检查点：**
- [ ] GitHub 评论已发布

---

## 阶段 7：输出 - 最终报告

仅输出此摘要（保持简洁）：

```markdown
## Fix Implementation Complete

**PR**: #{number}
**Branch**: {HEAD_BRANCH}
**Status**: {COMPLETE | PARTIAL}

| Severity | Fixed |
|----------|-------|
| CRITICAL | {n}/{total} |
| HIGH | {n}/{total} |

**Validation**: All checks pass
**Pushed**: Changes pushed to PR

See fix report: `$ARTIFACTS_DIR/review/fix-report.md`
```

---

## 错误处理

### 修复后类型检查失败

1. 查看错误
2. 调整修复
3. 重新运行类型检查
4. 如果仍然失败，标记为 "Not Fixed" 并注明原因

### 测试失败

1. 检查是否由修复导致失败
2. 选择：修复实现，或修复测试
3. 如果不确定，标记为 "Not Fixed" 等待人工审查

### 推送失败

1. 使用 rebase 拉取：`git pull --rebase origin $HEAD_BRANCH`
2. 解决冲突
3. 重新推送

---

## 成功标准

- **ON_CORRECT_BRANCH**：在 PR 的 head 分支上工作，而非 base 分支或新分支
- **CRITICAL_ADDRESSED**：所有 CRITICAL 问题已尝试修复
- **HIGH_ADDRESSED**：所有 HIGH 问题已尝试修复
- **VALIDATION_PASSED**：类型检查、代码检查、测试、构建全部通过
- **COMMITTED_AND_PUSHED**：更改已提交并推送到 PR 分支
- **REPORTED**：修复报告产物和 GitHub 评论已创建

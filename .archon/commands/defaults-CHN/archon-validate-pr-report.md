---
description: 综合所有验证发现，生成最终的 PR 审定报告
argument-hint: (none - reads from artifacts)
---

# PR 验证报告

将所有代码审查和端到端测试的发现综合为一份全面的审定报告。

---

## 阶段 1：收集所有产物

读取早期工作流节点生成的每一份产物：

```bash
echo "=== Available artifacts ==="
ls -la $ARTIFACTS_DIR/
echo ""
echo "=== Code review (main) ==="
cat $ARTIFACTS_DIR/code-review-main.md 2>/dev/null || echo "NOT AVAILABLE"
echo ""
echo "=== Code review (feature) ==="
cat $ARTIFACTS_DIR/code-review-feature.md 2>/dev/null || echo "NOT AVAILABLE"
echo ""
echo "=== E2E test (main) ==="
cat $ARTIFACTS_DIR/e2e-main.md 2>/dev/null || echo "NOT AVAILABLE (code-review-only PR)"
echo ""
echo "=== E2E test (feature) ==="
cat $ARTIFACTS_DIR/e2e-feature.md 2>/dev/null || echo "NOT AVAILABLE (code-review-only PR)"
```

同时读取 PR 详情：

```bash
PR_NUMBER=$(cat $ARTIFACTS_DIR/.pr-number | tr -d '\n')
gh pr view "$PR_NUMBER" --json title,body,url,headRefName,baseRefName,additions,deletions,changedFiles
```

列出所有截图：

```bash
ls $ARTIFACTS_DIR/e2e-*.png 2>/dev/null || echo "No screenshots"
```

如果存在截图，阅读几张关键截图以在报告中提供视觉上下文。

---

## 阶段 2：综合分析

### 2.1 交叉对比代码审查与 E2E 测试结果

对于每个已识别的缺陷/问题：
- **代码审查（主分支）**：代码分析是否发现了该缺陷？
- **E2E 测试（主分支）**：该缺陷在 UI 中是否可见？
- **代码审查（功能分支）**：修复代码看起来是否正确？
- **E2E 测试（功能分支）**：该缺陷在 UI 中是否确实已修复？

### 2.2 识别不一致之处

查找以下情况：
- 代码审查认为已修复，但 E2E 测试显示未修复
- E2E 测试显示已修复，但代码修复不够稳健或不完整
- E2E 测试发现了代码审查遗漏的新问题
- 代码审查发现了 E2E 测试无法覆盖的问题

### 2.3 确定最终审定结论

| 条件 | 批准（APPROVE）所需 |
|----------|---------------------|
| 主分支上缺陷已确认 | 是（或说明为何不需要） |
| 修复针对根本原因 | 是 |
| E2E 确认修复有效 | 是（若可通过 E2E 测试） |
| 无回归问题 | 是 |
| 代码质量合格 | 是 |
| 符合 CLAUDE.md 规范 | 是 |

---

## 阶段 3：撰写最终报告

写入 `$ARTIFACTS_DIR/validation-report.md`：

```markdown
# PR 验证报告：#{number}

**标题**：{PR 标题}
**URL**：{PR URL}
**分支**：{head} → {base}
**文件**：{count} 个文件变更（+{additions} -{deletions}）
**验证日期**：{ISO 时间戳}

---

## 审定结论：{APPROVE / REQUEST_CHANGES / NEEDS_DISCUSSION}

{2-3 句话的执行摘要。直接回答：这个 PR 是否可以合并？}

---

## 缺陷确认

| 声明 | 主分支已确认？ | 功能分支已修复？ | 证据 |
|-------|--------------------|--------------------|----------|
| {声明 1} | YES/NO | YES/NO | {截图引用或代码引用} |
| {声明 2} | YES/NO | YES/NO | {截图引用或代码引用} |

---

## 代码审查摘要

### 主分支（修复前）
{来自 code-review-main.md 的简要摘要 — 缺陷在代码中是否明显？}

### 功能分支（修复后）
{来自 code-review-feature.md 的简要摘要 — 修复是否正确且最优？}

**修复质量评分**：{n}/5

---

## E2E 测试摘要

{若已执行 E2E 测试：}

### 主分支（缺陷复现）
{来自 e2e-main.md 的简要摘要 — 缺陷在 UI 中是否可见？}

### 功能分支（修复验证）
{来自 e2e-feature.md 的简要摘要 — 修复是否在 UI 中得到验证？}

**修复置信度**：HIGH / MEDIUM / LOW

{若仅做代码审查：}

_已跳过 E2E 测试 — 本 PR 的变更不涉及 UI 可见部分。验证仅基于代码审查。_

---

## 截图

{列出关键截图并附说明：}

| 截图 | 说明 |
|------------|-------------|
| `e2e-main-01-initial.png` | {展示内容} |
| `e2e-feature-01-initial.png` | {展示内容 — 与主分支对比} |

---

## 发现的问题

### 合并前必须修复
{来自任何审查阶段的 CRITICAL 或 HIGH 级别问题。若无，写"无"。}

### 建议修复（非阻塞）
{MEDIUM 级别问题 — 建议修复但不阻塞合并。若无，写"无"。}

### 次要 / 建议
{LOW 级别问题 — 锦上添花。若无，写"无"。}

---

## 回归问题
{修复引入的任何新问题，或"未发现"。}

---

## 值得肯定之处
{正面评价 — 良好的模式、整洁的代码、完善的修复}

---

## 建议

**{APPROVE / REQUEST_CHANGES / NEEDS_DISCUSSION}**

{最终段落：明确的建议及理由。若为 REQUEST_CHANGES，列出需要修改的具体内容。若为 NEEDS_DISCUSSION，描述需要讨论的事项。}
```

### 3.1 在 PR 上发布摘要（可选）

如果审定结论明确，在 PR 上发布精简摘要作为评论：

```bash
PR_NUMBER=$(cat $ARTIFACTS_DIR/.pr-number | tr -d '\n')

# 创建简洁的 PR 评论
gh pr comment "$PR_NUMBER" --body "$(cat <<'COMMENT'
## Archon PR 验证报告

**审定结论**：{APPROVE / REQUEST_CHANGES}

### 摘要
{2-3 句话概述}

### 缺陷确认
| 声明 | 主分支 | 功能分支 |
|-------|------|---------|
| {声明} | {状态} | {状态} |

### 问题
{列出所有必须修复的问题，或"未发现阻塞问题。"}

---
_由 archon-validate-pr 工作流验证_
COMMENT
)"
```

---

## 成功标准

- **ALL_ARTIFACTS_READ**：所有可用产物已加载和分析
- **CROSS_REFERENCED**：代码审查与 E2E 测试结果已完成交叉对比
- **VERDICT_DETERMINED**：审定结论明确：APPROVE / REQUEST_CHANGES / NEEDS_DISCUSSION
- **REPORT_WRITTEN**：已创建 `$ARTIFACTS_DIR/validation-report.md`
- **PR_COMMENTED**：摘要已发布到 PR

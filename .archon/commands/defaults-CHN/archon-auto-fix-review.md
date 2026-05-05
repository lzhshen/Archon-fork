---
description: 自动修复所有评审发现（明确的 YAGNI 违规除外），发布修复报告
argument-hint: (none - reads all review artifacts from $ARTIFACTS_DIR/review/)
---

# 自动修复评审发现

---

## 重要：输出行为

**你的输出将作为 GitHub 评论发布。** 请将工作输出控制在最少：
- 不要逐步叙述每个操作
- 不要输出冗长的进度更新
- 仅在最后输出最终的结构化报告
- 使用 TodoWrite 工具静默跟踪进度

---

## 任务目标

阅读本次工作流运行中产生的所有评审产物，并修复其中提出的所有问题 -- 除非某个发现是明确的 YAGNI 违规，或是超出原始修复范围之外的推测性过度工程。进行验证、提交、推送、编写产物，并发布 GitHub 评论说明修复了什么以及跳过了什么及其原因。

**输出产物**: `$ARTIFACTS_DIR/review/fix-report.md`
**Git 操作**: 提交并推送修复到 PR 分支
**GitHub 操作**: 在 PR 上发布修复报告作为评论

---

## 阶段 1：加载 -- 获取上下文

### 1.1 获取 PR 编号和分支

```bash
PR_NUMBER=$(cat $ARTIFACTS_DIR/.pr-number)
HEAD_BRANCH=$(gh pr view $PR_NUMBER --json headRefName --jq '.headRefName')
echo "PR: $PR_NUMBER, Branch: $HEAD_BRANCH"
```

### 1.2 切换到 PR 分支

**务必重新切换，确保你在正确的分支上。**

```bash
git fetch origin $HEAD_BRANCH
git checkout $HEAD_BRANCH
git pull origin $HEAD_BRANCH
```

验证：

```bash
git branch --show-current
git status --porcelain
```

### 1.3 阅读所有评审产物

发现所有已存在的评审产物 -- 可能有一个或多个，取决于运行了哪些评审智能体：

```bash
ls $ARTIFACTS_DIR/review/
```

读取每个看起来像发现产物的 `.md` 文件（例如 `code-review-findings.md`、`error-handling-findings.md`、`test-coverage-findings.md`、`docs-impact-findings.md`、`consolidated-review.md`）。跳过非发现类文件，如 `scope.md` 和 `fix-report.md`。

```bash
for f in $ARTIFACTS_DIR/review/*.md; do
  echo "=== $f ==="; cat "$f"; echo
done
```

### 1.4 提取发现项

从所有已加载的产物中，汇总所有发现的统一列表，包含其严重程度、位置和建议修复方案。

**阶段 1 检查点:**
- [ ] PR 编号和分支已确认
- [ ] 已切换到正确的 PR 分支
- [ ] 所有评审产物已阅读
- [ ] 所有发现项已提取

---

## 阶段 2：分诊 -- 决定修复内容

对每个发现项，决定：**修复** 还是 **跳过**。

### 修复条件：
- 它是一个真实的缺陷、类型错误、静默失败，或明确的代码质量问题
- 修复方案具体且风险较低

### 跳过条件（YAGNI / 超出范围）-- 当发现建议以下内容时：
- 添加修复原始问题所不需要的东西（新配置选项、新抽象、推测性回退、"万一"的边缘情况）
- 重构或重组并未损坏的代码
- 为在此上下文中不可能无效的输入添加验证
- 为当前只有一个调用方的代码提取工具函数或辅助函数
- 涉及 PR 范围之外代码的架构变更

请运用判断力 -- 不要过于严格。如果是评审者发现的合理缺陷，即使它紧邻 PR 范围，也应修复。如果明显是推测性的（"将来可能有用"），则跳过。

对于每个跳过的发现项，写下**具体原因** -- 这将写入报告。

**阶段 2 检查点:**
- [ ] 每个发现项已标记为修复或跳过
- [ ] 跳过原因已记录

---

## 阶段 3：实施 -- 应用修复

### 3.1 对每个标记为修复的发现项

1. 阅读相关文件
2. 按照评审产物中建议的方案应用修复
3. 每次修复后运行类型检查：`bun run type-check`
4. 精确记录所做的更改

### 3.2 处理无法修复的发现项

如果修复无法应用（评审后代码已变更、修复方案不明确、修复会破坏其他功能），将其标记为 **受阻（BLOCKED）** 并记录原因。不要强行应用有问题的修复。

### 3.3 为已修复的代码添加测试

如果评审标记了刚修复内容的测试覆盖缺失，添加针对性的测试。运行测试：

```bash
bun test {file}
```

**阶段 3 检查点:**
- [ ] 所有标记为修复的发现项已尝试
- [ ] 已在标记处添加测试
- [ ] 受阻的发现项已记录

---

## 阶段 4：验证 -- 完整检查

```bash
bun run type-check
bun run lint
bun test
```

全部必须通过。如果修复后某项失败：
1. 查看错误
2. 调整修复或回退并标记为受阻
3. 重新运行直到全部通过

**阶段 4 检查点:**
- [ ] 类型检查通过
- [ ] 代码规范检查通过
- [ ] 测试通过

---

## 阶段 5：提交和推送

### 5.1 暂存和提交

仅暂存你实际修改的文件：

```bash
git add {specific files}
git status
git commit -m "fix: address review findings

$(echo "Fixed:"; echo "- {brief list}")
$(echo ""; echo "Skipped (YAGNI/out-of-scope):"; echo "- {brief list if any}")"
```

### 5.2 推送

```bash
git push origin $HEAD_BRANCH
```

如果推送因分叉而失败：

```bash
git pull --rebase origin $HEAD_BRANCH
git push origin $HEAD_BRANCH
```

**阶段 5 检查点:**
- [ ] 更改已提交
- [ ] 已推送到 PR 分支

---

## 阶段 6：生成 -- 编写修复报告

写入 `$ARTIFACTS_DIR/review/fix-report.md`：

```markdown
# 修复报告：PR #{number}

**日期**: {ISO timestamp}
**状态**: COMPLETE | PARTIAL
**分支**: {HEAD_BRANCH}
**提交**: {commit hash}

---

## 概要

{2-3 句话概述发现内容、修复内容、跳过内容及其原因}

---

## 已应用的修复

| 严重程度 | 发现项 | 位置 | 操作内容 |
|----------|--------|------|----------|
| CRITICAL | {标题} | `file:line` | {描述} |
| HIGH     | {标题} | `file:line` | {描述} |

---

## 跳过的发现项

| 严重程度 | 发现项 | 位置 | 跳过原因 |
|----------|--------|------|----------|
| HIGH     | {标题} | `file:line` | YAGNI：{具体原因} |
| MEDIUM   | {标题} | `file:line` | 超出范围：{原因} |

---

## 新增测试

| 文件 | 测试用例 |
|------|----------|
| `{file}.test.ts` | `{测试描述}` |

如未添加测试，填写 *(无)*

---

## 受阻（无法修复）

| 严重程度 | 发现项 | 原因 |
|----------|--------|------|
| {sev}    | {标题} | {无法应用的原因} |

如无受阻项，填写 *(无)*

---

## 验证结果

| 检查项 | 状态 |
|--------|------|
| 类型检查 | ✅ / ❌ |
| 代码规范检查 | ✅ / ❌ |
| 测试 | ✅ {n} 通过 / ❌ |
```

**阶段 6 检查点:**
- [ ] 修复报告已写入

---

## 阶段 7：发布 -- GitHub 评论

将修复报告作为 PR 评论发布：

```bash
gh pr comment $PR_NUMBER --body "$(cat <<'EOF'
## Auto-Fix Report

**状态**: {COMPLETE | PARTIAL}
**推送**: ✅ 更改已推送到 `{HEAD_BRANCH}`

---

### 已应用的修复

| 严重程度 | 发现项 | 位置 |
|----------|--------|------|
| CRITICAL | {标题} | `file:line` |
| HIGH | {标题} | `file:line` |

如无修复项，填写 *(无)*

---

### 跳过项

| 严重程度 | 发现项 | 原因 |
|----------|--------|------|
| HIGH | {标题} | {原因 -- YAGNI、超出范围、受阻} |

如无跳过项，填写 *(无)*

---

### 新增测试

{列表或 "(无)"}

---

### 验证结果

✅ 类型检查 | ✅ 代码规范检查 | ✅ 测试 ({n} 通过)

---

*由 Archon 自动修复 · 修复已推送到 `{HEAD_BRANCH}`*
EOF
)"
```

**阶段 7 检查点:**
- [ ] GitHub 评论已发布

---

## 阶段 8：输出 -- 最终摘要

仅输出以下内容：

```
## Auto-Fix Complete

**PR**: #{number}
**分支**: {HEAD_BRANCH}
**状态**: COMPLETE | PARTIAL

已修复: {n}
已跳过: {n} (YAGNI/超出范围)
受阻: {n}

验证: ✅ 所有检查通过
推送: ✅

修复报告: $ARTIFACTS_DIR/review/fix-report.md
```

---

## 错误处理

### 修复后类型检查失败
1. 查看错误
2. 调整或回退修复
3. 如果合理尝试后仍然失败，标记为受阻

### 测试失败
1. 检查是修复导致的还是原有问题
2. 如果修复是正确的，修复测试
3. 如果不确定，标记为受阻 -- 不要交付有问题的测试

### 推送失败
1. `git pull --rebase origin $HEAD_BRANCH`
2. 如有冲突则解决
3. 再次推送

### 未找到评审产物
```
❌ 在 $ARTIFACTS_DIR/review/ 中未找到评审产物
没有发现项，无法继续。
```

---

## 成功标准

- **在正确分支上**: 在 PR 的 head 分支上工作
- **所有发现已处理**: 每个发现项要么已修复、已跳过（附原因）、要么已标记受阻（附原因）
- **验证已通过**: 类型检查、代码规范检查和测试全部通过
- **已提交和推送**: 更改已提交并推送到 PR 分支
- **已报告**: 修复报告产物已写入且 GitHub 评论已发布

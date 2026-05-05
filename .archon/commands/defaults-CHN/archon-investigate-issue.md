---
description: 调查 GitHub Issue 或问题 - 分析代码库、创建计划、发布到 GitHub
argument-hint: <issue-number|url|"description">
---

# 调查问题

**输入**：$ARGUMENTS

---

## 你的任务

调查问题/故障，并产出一份全面的实施计划，要求：

1. 可由 `/implement-issue` 执行
2. 作为 GitHub 评论发布（如果提供了 GH Issue）
3. 包含一次性实施所需的全部上下文

**黄金法则**：你产出的产物就是规格说明。执行实施的 Agent 应能据此工作而无需提问。

---

## 阶段 1：解析 - 理解输入

### 1.1 判断输入类型

**检查输入格式：**

- 看起来像数字（`123`、`#123`）→ GitHub Issue 编号
- 以 `http` 开头 → GitHub URL（提取 Issue 编号）
- 其他 → 自由格式描述

```bash
# 如果是 GitHub Issue，获取内容：
gh issue view {number} --json title,body,labels,comments,state,url,author
```

### 1.2 提取上下文

**如果是 GitHub Issue：**
- 标题：报告的问题是什么？
- 正文：详情、复现步骤、预期行为与实际行为
- 标签：bug？enhancement？documentation？
- 评论：讨论中的额外上下文
- 状态：是否仍然开放？

**如果是自由格式：**
- 解析为问题描述
- 注意：不发布到 GitHub（仅生成产物）

### 1.3 分类问题类型

| 类型 | 指标 |
|------|------|
| BUG | "broken"、"error"、"crash"、"doesn't work"、堆栈跟踪 |
| ENHANCEMENT | "add"、"support"、"feature"、"would be nice" |
| REFACTOR | "clean up"、"improve"、"simplify"、"reorganize" |
| CHORE | "update"、"upgrade"、"maintenance"、"dependency" |
| DOCUMENTATION | "docs"、"readme"、"clarify"、"example" |

### 1.4 评估严重程度/优先级、复杂度和置信度

每项评估都需要一句**理由说明**，解释为什么选择该值。理由必须基于调查中的具体发现（代码库探索、Git 历史、集成分析）。

**BUG 类型 - 严重程度：**

| 严重程度 | 标准 |
|----------|------|
| CRITICAL | 系统宕机、数据丢失、安全漏洞、无替代方案 |
| HIGH | 主要功能损坏、用户影响重大、替代方案困难 |
| MEDIUM | 功能部分损坏、影响中等、存在替代方案 |
| LOW | 次要问题、外观问题、边界情况、替代方案简单 |

**ENHANCEMENT/REFACTOR/CHORE/DOCUMENTATION 类型 - 优先级：**

| 优先级 | 标准 |
|--------|------|
| HIGH | 阻塞其他工作、频繁请求、用户价值高 |
| MEDIUM | 重要但不紧急、用户价值中等 |
| LOW | 锦上添花、紧急程度低、用户影响最小 |

**复杂度**（基于代码库发现）：

| 复杂度 | 标准 |
|--------|------|
| HIGH | 5 个以上文件、多个集成点、架构级更改、高风险 |
| MEDIUM | 2-4 个文件、部分集成点、中等风险 |
| LOW | 1-2 个文件、独立更改、低风险 |

**置信度**（基于证据质量）：

| 置信度 | 标准 |
|--------|------|
| HIGH | 根因明确、证据充分、代码路径理解透彻 |
| MEDIUM | 可能的根因、部分假设、部分理解 |
| LOW | 根因不确定、证据有限、未知因素多 |

**阶段 1 检查点：**
- [ ] 输入类型已确认（GH Issue 或自由格式）
- [ ] Issue 内容已提取
- [ ] 类型已分类
- [ ] 严重程度（bug）或优先级（其他）已评估并附理由
- [ ] 复杂度已评估并附理由（阶段 2 之后）
- [ ] 置信度已评估并附理由（阶段 3 之后）
- [ ] 如果是 GH Issue：确认其仍然开放且尚无关联 PR

---

## 阶段 2：探索 - 代码库智能分析

### 2.1 搜索相关代码

使用 Task 工具，设置 subagent_type="Explore"：

```
Explore the codebase to understand the issue:

ISSUE: {title/description}

DISCOVER:
1. Files directly related to this functionality
2. How the current implementation works
3. Integration points - what calls this, what it calls
4. Similar patterns elsewhere to mirror
5. Existing test patterns for this area
6. Error handling patterns used

Return:
- File paths with specific line numbers
- Actual code snippets (not summaries)
- Dependencies and data flow
```

### 2.2 记录发现

| 区域 | 文件:行号 | 备注 |
|------|-----------|------|
| 核心逻辑 | `src/x.ts:10-50` | 受影响的主函数 |
| 调用方 | `src/y.ts:20-30` | 使用该核心函数 |
| 类型 | `src/types/x.ts:5-15` | 相关接口 |
| 测试 | `src/x.test.ts:1-100` | 现有测试模式 |
| 相似实现 | `src/z.ts:40-60` | 可参考的模式 |

**阶段 2 检查点：**
- [ ] 探索 Agent 已成功完成
- [ ] 核心文件已识别并标注行号
- [ ] 集成点已映射
- [ ] 已找到可参考的相似模式
- [ ] 测试模式已记录

---

## 阶段 3：分析 - 形成方案

### 3.0 第一性原理分析

在深入 bug 分析或增强范围界定之前，先识别原语：

1. **涉及的原语是什么？** 此 bug/功能触及的核心抽象是什么？
   （例如，条件评估器、审批系统、隔离提供者）
2. **原语是否健全？** 现有设计是否处理了这种情况，还是原语本身不完整或缺少某种情况？
3. **根因 vs 症状** —— 我们要修复的是错误表现的地方，还是错误产生的地方？沿着数据流追溯到源头。
4. **最小更改是什么？** 修复根因的最小编辑是什么？
   当扩展现有方案可行时，避免添加新的抽象。
5. **这能解锁什么？** 如果我们添加/更改一个原语，还有哪些改进成为可能？

| 原语 | 文件:行号 | 是否健全 | 备注 |
|------|-----------|----------|------|
| {抽象名称} | `src/x.ts:10-30` | 是/否/部分 | {如果不完整：缺少什么} |

### 3.1 BUG 类型 - 根因分析

应用 5 个为什么方法：

```
WHY 1: 为什么会出现 [症状]？
→ 因为 [原因 A]
→ 证据：`file.ts:123` - {代码片段}

WHY 2: 为什么会发生 [原因 A]？
→ 因为 [原因 B]
→ 证据：{证明}

... 继续直到找到可修复的代码 ...

ROOT CAUSE: [需要更改的具体代码/逻辑]
Evidence: `source.ts:456` - {有问题的代码}
```

**检查 Git 历史：**
```bash
git log --oneline -10 -- {affected-file}
git blame -L {start},{end} {affected-file}
```

### 3.2 ENHANCEMENT/REFACTOR 类型

**确定：**
- 需要添加/更改什么？
- 它在哪里集成？
- 范围边界是什么？
- 不应该更改什么？

### 3.3 所有类型通用

**确定：**
- 需要创建的文件（新文件）
- 需要更新的文件（现有文件）
- 需要删除的文件（如有）
- 依赖关系和更改顺序
- 边界情况和风险
- 验证策略

**阶段 3 检查点：**
- [ ] 根因已确定（对于 bug）或变更理由已明确（对于增强）
- [ ] 所有受影响的文件已列出并注明具体更改
- [ ] 范围边界已定义（不更改什么）
- [ ] 风险和边界情况已识别
- [ ] 验证方法已确定

---

## 阶段 4：生成 - 创建产物

### 4.1 产物路径

```bash
```

**路径：** `$ARTIFACTS_DIR/investigation.md`

此统一路径允许评审 Agent 无论工作流类型如何都能找到产物。

### 4.2 产物模板

将此结构写入产物文件。

**关于严重程度与优先级的说明：**
- BUG 类型使用**严重程度**（CRITICAL、HIGH、MEDIUM、LOW）
- 其他类型使用**优先级**（HIGH、MEDIUM、LOW）

**重要：** 每项评估必须包含基于调查发现的一句话理由。

```markdown
# Investigation: {Title}

**Issue**: #{number} ({url})
**Type**: {BUG|ENHANCEMENT|REFACTOR|CHORE|DOCUMENTATION}
**Investigated**: {ISO timestamp}

### Assessment

| Metric | Value | Reasoning |
|--------|-------|-----------|
| Severity | {CRITICAL\|HIGH\|MEDIUM\|LOW} | {Why this severity? Based on user impact, workarounds, scope of failure} |
| Complexity | {LOW\|MEDIUM\|HIGH} | {Why this complexity? Based on files affected, integration points, risk} |
| Confidence | {HIGH\|MEDIUM\|LOW} | {Why this confidence? Based on evidence quality, unknowns, assumptions} |

<!-- For non-BUG types, replace Severity row with Priority:
| Priority | {HIGH\|MEDIUM\|LOW} | {Why this priority? Based on user value, blocking status, frequency} |
-->

---

## Problem Statement

{Clear 2-3 sentence description of what's wrong or what's needed}

---

## Analysis

### Root Cause / Change Rationale

{For BUG: The 5 Whys chain with evidence}
{For ENHANCEMENT: Why this change and what it enables}

### Evidence Chain

WHY: {symptom}
↓ BECAUSE: {cause 1}
  Evidence: `file.ts:123` - `{code snippet}`

↓ BECAUSE: {cause 2}
  Evidence: `file.ts:456` - `{code snippet}`

↓ ROOT CAUSE: {the fixable thing}
  Evidence: `file.ts:789` - `{problematic code}`

### Affected Files

| File | Lines | Action | Description |
|------|-------|--------|-------------|
| `src/x.ts` | 45-60 | UPDATE | {what changes} |
| `src/x.test.ts` | NEW | CREATE | {test to add} |

### Integration Points

- `src/y.ts:20` calls this function
- `src/z.ts:30` depends on this behavior
- {other dependencies}

### Git History

- **Introduced**: {commit} - {date} - "{message}"
- **Last modified**: {commit} - {date}
- **Implication**: {regression? original bug? long-standing?}

---

## Implementation Plan

### Step 1: {First change description}

**File**: `src/x.ts`
**Lines**: 45-60
**Action**: UPDATE

**Current code:**
```typescript
// Line 45-50
{actual current code}
```

**Required change:**
```typescript
// What it should become
{the fix/change}
```

**Why**: {brief rationale}

---

### Step 2: {Second change description}

{Same structure...}

---

### Step N: Add/Update Tests

**File**: `src/x.test.ts`
**Action**: {CREATE|UPDATE}

**Test cases to add:**
```typescript
describe('{feature}', () => {
  it('should {expected behavior}', () => {
    // Test the fix
  });

  it('should handle {edge case}', () => {
    // Test edge case
  });
});
```

---

## Patterns to Follow

**From codebase - mirror these exactly:**

```typescript
// SOURCE: src/similar.ts:20-30
// Pattern for {what this demonstrates}
{actual code snippet from codebase}
```

---

## Edge Cases & Risks

| Risk/Edge Case | Mitigation |
|----------------|------------|
| {risk 1} | {how to handle} |
| {edge case} | {how to handle} |

---

## Validation

### Automated Checks

```bash
bun run type-check
bun test {relevant-pattern}
bun run lint
```

### Manual Verification

1. {Step to verify the fix/feature works}
2. {Step to verify no regression}

---

## Scope Boundaries

**IN SCOPE:**
- {what we're changing}

**OUT OF SCOPE (do not touch):**
- {what to leave alone}
- {future improvements to defer}

---

## Metadata

- **Investigated by**: Claude
- **Timestamp**: {ISO timestamp}
- **Artifact**: `$ARTIFACTS_DIR/investigation.md`
```

**阶段 4 检查点：**
- [ ] 产物文件已创建
- [ ] 所有部分已填写具体内容
- [ ] 代码片段是真实的（非编造）
- [ ] 步骤可操作，无需额外澄清

---

## 阶段 5：发布 - GitHub 评论

**仅当输入为 GitHub Issue 时（非自由格式）：**

格式化产物内容并发布到 GitHub：

```bash
gh issue comment {number} --body "$(cat <<'EOF'
## Investigation: {Title}

**Type**: `{TYPE}`

### Assessment

| Metric | Value | Reasoning |
|--------|-------|-----------|
| {Severity or Priority} | `{VALUE}` | {one-sentence why} |
| Complexity | `{COMPLEXITY}` | {one-sentence why} |
| Confidence | `{CONFIDENCE}` | {one-sentence why} |

---

### Problem Statement

{problem statement from artifact}

---

### Root Cause Analysis

{evidence chain, formatted for GitHub}

---

### Implementation Plan

| Step | File | Change |
|------|------|--------|
| 1 | `src/x.ts:45` | {description} |
| 2 | `src/x.test.ts` | Add test for {case} |

<details>
<summary>Detailed Implementation Steps</summary>

{detailed steps from artifact}

</details>

---

### Validation

```bash
bun run type-check && bun test {pattern} && bun run lint
```

---

### Next Step

To implement: `/implement-issue {number}`

---
*Investigated by Claude - {timestamp}*
EOF
)"
```

**阶段 5 检查点：**
- [ ] 评论已发布到 GitHub（如果是 GH Issue）
- [ ] 格式渲染正确

---

## 阶段 6：报告 - 输出给用户

```markdown
## Investigation Complete

**Issue**: #{number} - {title}
**Type**: {BUG|ENHANCEMENT|REFACTOR|...}

### Assessment

| Metric | Value | Reasoning |
|--------|-------|-----------|
| {Severity or Priority} | {value} | {why - based on investigation} |
| Complexity | {LOW\|MEDIUM\|HIGH} | {why - based on files/integration/risk} |
| Confidence | {HIGH\|MEDIUM\|LOW} | {why - based on evidence/unknowns} |

### Key Findings

- **Root Cause**: {one-line summary}
- **Files Affected**: {count} files
- **Estimated Changes**: {brief scope}

### Files to Modify

| File | Action |
|------|--------|
| `src/x.ts` | UPDATE |
| `src/x.test.ts` | CREATE |

### Artifact

`$ARTIFACTS_DIR/investigation.md`

### GitHub

{Posted to issue | Skipped (free-form input)}

### Next Step

Run `/implement-issue {number}` to execute the plan.
```

---

## 处理边界情况

### Issue 已关闭
- 报告："Issue #{number} 已关闭"
- 如果用户需要分析，仍然创建产物

### Issue 已有关联 PR
- 警告："PR #{pr} 已在处理此 Issue"
- 询问用户是否仍要继续

### 无法确定根因
- 记录你的发现
- 将置信度设为 LOW
- 在产物中注明不确定性
- 基于最佳假设继续

### 范围过大
- 建议拆分为较小的 Issue
- 首先聚焦核心问题
- 在 "Out of Scope" 中记录推迟的项目

---

## 成功标准

- **ARTIFACT_COMPLETE**：所有部分已填写具体、可操作的内容
- **EVIDENCE_BASED**：每项声明都有 file:line 引用或证据
- **IMPLEMENTABLE**：其他 Agent 可无障碍执行
- **GITHUB_POSTED**：评论在 Issue 上可见（如果是 GH Issue）
- **COMMITTED**：产物已保存到 git

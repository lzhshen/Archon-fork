---
description: 将完成报告发布到 GitHub issue，包含执行结果、未处理项和后续建议
argument-hint: (none - reads from workflow artifacts)
---

# Issue 完成报告

**输入**: $ARGUMENTS
**工作流 ID**: $WORKFLOW_ID

---

## 任务目标

汇总所有工作流产物，编写最终报告并发布到原始 GitHub issue。总结已完成的工作、未处理的事项（及原因），并在需要时建议后续 issue。

**GitHub 操作**: 将完成报告作为评论发布到原始 issue
**输出产物**: `$ARTIFACTS_DIR/completion-report.md`

---

## 阶段 1: 加载 -- 收集所有产物

### 1.1 获取 Issue 编号

从 `$ARGUMENTS` 中提取 issue 编号：

```bash
# $ARGUMENTS 应为 issue 编号或 URL
echo "$ARGUMENTS"
```

### 1.2 获取 PR 信息

```bash
PR_NUMBER=$(cat $ARTIFACTS_DIR/.pr-number 2>/dev/null || echo "unknown")
PR_URL=$(cat $ARTIFACTS_DIR/.pr-url 2>/dev/null || echo "unknown")
echo "PR: $PR_NUMBER ($PR_URL)"
```

### 1.3 读取所有可用产物

检查并读取可能存在的每项产物：

```bash
# 调查/计划
cat $ARTIFACTS_DIR/investigation.md 2>/dev/null
cat $ARTIFACTS_DIR/plan.md 2>/dev/null

# 实现
cat $ARTIFACTS_DIR/implementation.md 2>/dev/null

# 网络调研
cat $ARTIFACTS_DIR/web-research.md 2>/dev/null

# 验证
cat $ARTIFACTS_DIR/validation.md 2>/dev/null

# 审查产物
ls $ARTIFACTS_DIR/review/ 2>/dev/null
cat $ARTIFACTS_DIR/review/consolidated-review.md 2>/dev/null
cat $ARTIFACTS_DIR/review/fix-report.md 2>/dev/null
```

### 1.4 获取 Git 信息

```bash
git branch --show-current
git log --oneline -5
```

**阶段 1 检查点:**

- [ ] Issue 编号已确认
- [ ] PR 信息已加载
- [ ] 所有可用产物已读取
- [ ] Git 状态已记录

---

## 阶段 2: 汇编 -- 编写报告

### 2.1 总结已完成的工作

根据产物，汇总以下内容：

- **分类**: Issue 类型（缺陷/功能/等）
- **调查/计划**: 关键发现和方法
- **实现**: 修改了什么，涉及哪些文件
- **验证**: 测试结果、代码检查、类型检查
- **审查**: 审查了什么，发现问题数量
- **自我修复**: 哪些审查发现已修复

### 2.2 识别未处理项

从修复报告和综合审查中识别：

- 被跳过的发现项（附原因）
- 被阻塞的发现项（附原因）
- 未自动修复的中/低优先级发现
- 任何持续存在的验证问题

### 2.3 建议后续 Issue

对每个未处理项，判断是否需要创建后续 issue：

| 项目 | 是否需要创建 Issue? | 原因 |
|------|---------------------|------|
| {跳过的发现} | 是/否 | {原因} |

**阶段 2 检查点:**

- [ ] 总结已汇编
- [ ] 未处理项已识别
- [ ] 后续建议已准备

---

## 阶段 3: 生成 -- 编写产物

写入 `$ARTIFACTS_DIR/completion-report.md`：

```markdown
# 完成报告: Issue $ARGUMENTS

**日期**: {ISO 时间戳}
**工作流 ID**: $WORKFLOW_ID
**PR**: #{pr-number} ({pr-url})

---

## 概要

{3-5 句话概述整个工作流的执行过程}

---

## 分类

| 字段 | 值 |
|------|-----|
| 类型 | {缺陷/功能/增强/...} |
| 复杂度 | {低/中/高} |
| 置信度 | {高/中/低} |

---

## 已完成的工作

### 调查/规划

{简要总结根因分析或计划}

### 实现

| 文件 | 操作 | 说明 |
|------|------|------|
| `{file}` | {创建/更新} | {修改内容} |

### 验证

| 检查项 | 结果 |
|--------|------|
| 类型检查 | 通过 / 失败 |
| 代码检查 | 通过 / 失败 |
| 测试 | 通过 ({n} 项通过) / 失败 |

### 审查与自我修复

- **发现**: 审查代理共发现 {n} 项
- **已修复**: {n} 项（包括测试、文档、简化）
- **已跳过**: {n} 项
- **已阻塞**: {n} 项

---

## 未处理项

{若无: "所有发现项已处理。"}

### 已跳过

| 发现项 | 严重程度 | 原因 |
|--------|----------|------|
| {标题} | {严重程度} | {原因} |

### 已阻塞

| 发现项 | 严重程度 | 原因 |
|--------|----------|------|
| {标题} | {严重程度} | {原因} |

---

## 建议的后续 Issue

| 标题 | 优先级 | 说明 |
|------|--------|------|
| "{标题}" | {P1/P2/P3} | {简要描述} |

{若所有问题已处理则写 "无"}

---

## 产物

| 产物 | 路径 |
|------|------|
| 调查/计划 | `$ARTIFACTS_DIR/{investigation 或 plan}.md` |
| 网络调研 | `$ARTIFACTS_DIR/web-research.md` |
| 实现 | `$ARTIFACTS_DIR/implementation.md` |
| 综合审查 | `$ARTIFACTS_DIR/review/consolidated-review.md` |
| 修复报告 | `$ARTIFACTS_DIR/review/fix-report.md` |
```

**阶段 3 检查点:**

- [ ] 完成报告已编写

---

## 阶段 4: 发布 -- GitHub Issue 评论

发布到原始 GitHub issue：

```bash
ISSUE_NUMBER=$(echo "$ARGUMENTS" | grep -oE '[0-9]+')

gh issue comment $ISSUE_NUMBER --body "$(cat <<'EOF'
## Issue 解决报告

**PR**: #{pr-number} ({pr-url})
**状态**: 已完成

---

### 概要

{简要概述为解决此 issue 所做的工作}

---

### 变更内容

| 文件 | 变更 |
|------|------|
| `{file}` | {描述} |

---

### 验证

类型检查 通过 | 代码检查 通过 | 测试 通过 ({n} 项通过)

---

### 审查与自我修复

- **{n}** 项审查发现已处理
- **{n}** 项测试已添加
- **{n}** 项文档已更新
- **{n}** 项代码简化已应用

---

### 未处理项

{若无: "所有审查发现已在 PR 中处理。"}

{若有:}

| 发现项 | 严重程度 | 原因 |
|--------|----------|------|
| {标题} | {严重程度} | {未处理原因} |

---

### 建议的后续 Issue

{若有:}

1. **{Issue 标题}** ({优先级}) -- {简要描述}

{若无: "无需后续 issue。"}

---

*由 Archon 工作流 `$WORKFLOW_ID` 解决*
EOF
)"
```

**阶段 4 检查点:**

- [ ] GitHub 评论已发布到 issue

---

## 阶段 5: 输出 -- 最终总结

```markdown
## Issue 解决完成

**Issue**: $ARGUMENTS
**PR**: #{pr-number}
**工作流**: $WORKFLOW_ID

### 结果

- 实现: 已完成
- 验证: 已完成
- 审查: 已完成
- 自我修复: 已完成

### 未处理项: {n} 项
### 建议的后续 issue: {n} 项

### 产物

- 完成报告: `$ARTIFACTS_DIR/completion-report.md`
- GitHub 评论: 已发布到 issue

### 后续步骤

1. 审查 PR: #{pr-number}
2. 如同意，创建建议的后续 issue
3. 准备就绪后合并
```

---

## 成功标准

- **ALL_ARTIFACTS_READ**: 所有工作流产物已加载和解析
- **REPORT_COMPILED**: 综合完成报告已编写
- **GITHUB_POSTED**: 评论已发布到原始 issue
- **UNADDRESSED_DOCUMENTED**: 任何未修复项均有明确原因说明
- **FOLLOWUPS_SUGGESTED**: 在适当情况下推荐了可执行的后续 issue

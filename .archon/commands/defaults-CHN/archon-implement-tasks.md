---
description: 执行计划任务，每次更改后进行类型检查
argument-hint: (no arguments - reads from workflow artifacts)
---

# 实施任务

**工作流 ID**：$WORKFLOW_ID

---

## 你的任务

执行计划中的每个任务，每次更改后进行验证。

**核心理念**：
- 每次文件更改后都进行类型检查
- 立即修复问题，不拖延
- 记录任何偏离计划的情况

**此步骤假设设置已完成** - 分支已存在、PR 已创建、计划已确认。

---

## 阶段 1：加载 - 读取上下文

### 1.1 加载计划上下文

```bash
cat $ARTIFACTS_DIR/plan-context.md
```

提取：
- 需要更改的文件（CREATE/UPDATE 列表）
- 验证命令（尤其是 type-check）
- 需要参照的模式

### 1.2 加载计划确认

```bash
cat $ARTIFACTS_DIR/plan-confirmation.md
```

检查：
- 状态为 CONFIRMED 或 PROCEED WITH CAUTION
- 记录实施过程中需要注意的警告

### 1.3 加载原始计划

计划源路径在 `plan-context.md` 中。阅读完整计划以获取详细的任务说明：

```bash
cat {plan-source-path}
```

### 1.4 识别包管理器

```bash
test -f bun.lockb && echo "bun" || \
test -f pnpm-lock.yaml && echo "pnpm" || \
test -f yarn.lock && echo "yarn" || \
test -f package-lock.json && echo "npm" || \
echo "unknown"
```

存储运行器以用于后续的验证命令。

**阶段 1 检查点：**

- [ ] 计划上下文已加载
- [ ] 确认状态已验证
- [ ] 原始计划已加载
- [ ] 包管理器已识别

---

## 阶段 2：执行 - 实施每个任务

**针对计划中 "Tasks" 或 "Step-by-Step Tasks" 部分的每个任务：**

### 2.1 阅读任务上下文

在实施每个任务之前：

1. **阅读**任务中引用的 **MIRROR 文件**
2. **理解**要遵循的**模式**
3. **注意**任何 **GOTCHA 警告**
4. **检查**所需的 **IMPORTS**

### 2.2 实施任务

按照说明进行更改：

- **CREATE**：按照模式编写新文件
- **UPDATE**：按照描述修改现有文件
- **严格遵循模式** - 匹配风格、命名、结构

### 2.3 立即类型检查

**每次文件更改后：**

```bash
{runner} run type-check
```

**如果类型检查失败：**

1. 仔细阅读错误消息
2. 修复类型问题
3. 重新运行类型检查
4. 只有通过后才继续

**不要积累错误** - 在进入下一个任务前修复每一个。

### 2.4 跟踪进度

记录每个已完成的任务：

```
Task 1: CREATE src/features/x/models.ts ✅
Task 2: CREATE src/features/x/service.ts ✅
Task 3: UPDATE src/routes/index.ts ✅
```

### 2.5 处理偏差

如果必须偏离计划：

1. **记录改变了什么**
2. **记录为什么改变**
3. **继续执行**，并注明偏差

常见的偏差原因：
- 自计划创建以来模式文件已更改
- 发现缺少导入
- 类型不兼容需要不同的方法
- 实施过程中发现了更好的解决方案

**阶段 2 检查点（每个任务）：**

- [ ] 任务已实施
- [ ] 类型检查通过
- [ ] 进度已记录
- [ ] 偏差已记录（如有）

---

## 阶段 3：测试 - 编写必要的测试

### 3.1 测试要求

每个新函数/功能至少需要一个测试：

- **创建了新文件** → 创建对应的测试文件
- **添加了新函数** → 为该函数添加测试
- **行为已更改** → 更新现有测试

### 3.2 遵循测试模式

查找现有测试文件作为参考：

```bash
find . -name "*.test.ts" -type f | head -5
```

阅读相关测试文件以了解项目的测试模式。

### 3.3 编写测试

对于每个新建/修改的文件，编写涵盖以下情况的测试：

1. **正常路径** - 正常的预期行为
2. **边界情况** - 计划中指出的边界条件
3. **异常情况** - 输入异常时的行为

### 3.4 运行测试

```bash
{runner} test
```

**如果测试失败：**

1. 判断：实现中的 bug 还是测试中的 bug？
2. 修复实际问题（通常是实现）
3. 重新运行测试
4. 重复直到全部通过

**阶段 3 检查点：**

- [ ] 已为新代码编写测试
- [ ] 所有测试通过

---

## 阶段 4：产物 - 编写实施进度

### 4.1 编写进度产物

写入 `$ARTIFACTS_DIR/implementation.md`：

```markdown
# Implementation Progress

**Generated**: {YYYY-MM-DD HH:MM}
**Workflow ID**: $WORKFLOW_ID
**Status**: {COMPLETE | IN_PROGRESS | BLOCKED}

---

## Tasks Completed

| # | Task | File | Status | Notes |
|---|------|------|--------|-------|
| 1 | {description} | `src/x.ts` | ✅ | |
| 2 | {description} | `src/y.ts` | ✅ | |
| 3 | {description} | `src/z.ts` | ✅ | Minor deviation - see below |

**Progress**: {X} of {Y} tasks completed

---

## Files Changed

| File | Action | Lines |
|------|--------|-------|
| `src/new-file.ts` | CREATE | +{N} |
| `src/existing.ts` | UPDATE | +{N}/-{M} |

---

## Tests Written

| Test File | Test Cases |
|-----------|------------|
| `src/x.test.ts` | `should do X`, `should handle Y` |
| `src/y.test.ts` | `creates correctly`, `validates input` |

---

## Deviations from Plan

{If none:}
No deviations. Implementation matched the plan exactly.

{If any:}
### Deviation 1: {brief title}

**Task**: {which task}
**Expected**: {what plan said}
**Actual**: {what was done}
**Reason**: {why the change was necessary}

---

## Type-Check Status

- [x] Passes after all changes

---

## Test Status

- [x] All tests pass
- Tests added: {N}
- Tests modified: {M}

---

## Issues Encountered

{If none:}
No issues encountered.

{If any:}
### Issue 1: {title}

**Problem**: {description}
**Resolution**: {how it was fixed}

---

## Next Step

Continue to `archon-validate` for full validation suite.
```

**阶段 4 检查点：**

- [ ] 实施产物已写入
- [ ] 所有任务已记录
- [ ] 偏差已注明
- [ ] 测试状态已记录

---

## 阶段 5：输出 - 报告进度

```markdown
## Implementation Complete

**Workflow ID**: `$WORKFLOW_ID`
**Status**: All tasks executed

### Progress Summary

| Metric | Count |
|--------|-------|
| Tasks completed | {X}/{Y} |
| Files created | {N} |
| Files updated | {M} |
| Tests written | {K} |

### Type-Check

Passes

### Tests

All pass ({N} tests)

{If deviations:}
### Deviations

{count} deviation(s) from plan documented in artifact.

### Artifact

Progress written to: `$ARTIFACTS_DIR/implementation.md`

### Next Step

Proceed to `archon-validate` for full validation (lint, build, integration tests).
```

---

## 错误处理

### 类型检查失败

不要继续下一个任务。修复问题：

1. 仔细阅读错误
2. 定位文件和行号
3. 修复类型问题
4. 重新运行类型检查
5. 只有通过后才继续

### 测试失败

1. 阅读失败输出
2. 判断：实现 bug 还是测试 bug？
3. 修复根本原因
4. 重新运行测试

### 模式文件已更改

如果模式文件自计划创建以来已更改：

1. 阅读当前版本
2. 调整实现以匹配当前模式
3. 记录为偏差
4. 继续执行

### 任务说明不明确

如果任务描述含糊不清：

1. 查看计划的上下文部分以获取清晰说明
2. 查看 MIRROR 文件获取指导
3. 做出合理的决定
4. 将理解记录为偏差

---

## 成功标准

- **TASKS_COMPLETE**：计划中的所有任务已执行
- **TYPES_PASS**：所有更改后类型检查通过
- **TESTS_WRITTEN**：新代码有测试
- **TESTS_PASS**：所有测试通过
- **DEVIATIONS_DOCUMENTED**：任何计划偏差已记录
- **ARTIFACT_WRITTEN**：实施进度产物已创建

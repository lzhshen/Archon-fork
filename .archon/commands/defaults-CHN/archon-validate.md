---
description: 运行完整验证套件 - 类型检查、代码检查、测试、构建
argument-hint: (no arguments - reads from workflow artifacts)
---

# 验证实现

**工作流 ID**：$WORKFLOW_ID

---

## 任务目标

运行完整的验证套件并修复所有失败项。

这是一个聚焦步骤：运行检查、修复问题、重复直到全部通过。

---

## 阶段 1：加载 - 获取验证命令

### 1.1 加载计划上下文

```bash
cat $ARTIFACTS_DIR/plan-context.md
```

提取"验证命令"部分。

### 1.2 识别包管理器

```bash
test -f bun.lockb && echo "bun" || \
test -f pnpm-lock.yaml && echo "pnpm" || \
test -f yarn.lock && echo "yarn" || \
test -f package-lock.json && echo "npm" || \
echo "unknown"
```

### 1.3 确认可用命令

检查 `package.json` 中的可用脚本：

```bash
cat package.json | grep -A 20 '"scripts"'
```

**阶段 1 检查点：**

- [ ] 验证命令已识别
- [ ] 包管理器已确认

---

## 阶段 2：验证 - 运行所有检查

按顺序运行每项检查。在进入下一步之前修复所有失败项。

### 2.1 类型检查

```bash
{runner} run type-check
```

**如果失败：**
1. 阅读错误输出
2. 修复类型问题
3. 重新运行直到通过

**记录结果**：✅ 通过 / ❌ 失败（已修复）

### 2.2 代码检查

```bash
{runner} run lint
```

**如果失败：**

1. 首先尝试自动修复：
   ```bash
   {runner} run lint:fix
   ```

2. 重新运行代码检查

3. 如果仍然失败，手动修复剩余问题

**记录结果**：✅ 通过 / ❌ 失败（已修复）

### 2.3 格式检查

```bash
{runner} run format:check
```

**如果失败：**

1. 自动修复：
   ```bash
   {runner} run format
   ```

2. 验证修复结果：
   ```bash
   {runner} run format:check
   ```

**记录结果**：✅ 通过 / ❌ 失败（已修复）

### 2.4 测试套件

```bash
{runner} test
```

**如果失败：**

1. 确定哪些测试失败
2. 判断：是实现缺陷还是测试缺陷？
3. 修复根本原因
4. 重新运行测试

**记录结果**：✅ 通过（{N} 个测试） / ❌ 失败（已修复）

### 2.5 构建检查

```bash
{runner} run build
```

**如果失败：**

1. 通常是类型或导入问题
2. 修复后重新运行

**记录结果**：✅ 通过 / ❌ 失败（已修复）

**阶段 2 检查点：**

- [ ] 类型检查通过
- [ ] 代码检查通过
- [ ] 格式检查通过
- [ ] 测试通过
- [ ] 构建通过

---

## 阶段 3：产物 - 写入验证结果

### 3.1 写入验证产物

写入 `$ARTIFACTS_DIR/validation.md`：

```markdown
# 验证结果

**生成时间**：{YYYY-MM-DD HH:MM}
**工作流 ID**：$WORKFLOW_ID
**状态**：{ALL_PASS | FIXED | BLOCKED}

---

## 概要

| 检查项 | 结果 | 详情 |
|-------|--------|---------|
| 类型检查 | ✅ | 无错误 |
| 代码检查 | ✅ | 0 个错误，{N} 个警告 |
| 格式检查 | ✅ | 所有文件已格式化 |
| 测试 | ✅ | {N} 个通过，0 个失败 |
| 构建 | ✅ | 编译成功 |

---

## 类型检查

**命令**：`{runner} run type-check`
**结果**：✅ 通过

{如果修复了问题：}
### 已修复的问题

- `src/file.ts:42` - 添加了缺失的返回类型
- `src/other.ts:15` - 修复了泛型约束

---

## 代码检查

**命令**：`{runner} run lint`
**结果**：✅ 通过

{如果修复了问题：}
### 已修复的问题

- {N} 个由 `lint:fix` 自动修复
- {M} 个手动修复

### 剩余警告

{列出所有未修复的警告，并附理由}

---

## 格式检查

**命令**：`{runner} run format:check`
**结果**：✅ 通过

{如果格式化了文件：}
### 已格式化的文件

- `src/file.ts`
- `src/other.ts`

---

## 测试

**命令**：`{runner} test`
**结果**：✅ 通过

| 指标 | 数量 |
|--------|-------|
| 测试总数 | {N} |
| 通过 | {N} |
| 失败 | 0 |
| 跳过 | {M} |

{如果修复了测试：}
### 已修复的测试

- `src/x.test.ts` - 修复断言以匹配新行为

---

## 构建

**命令**：`{runner} run build`
**结果**：✅ 通过

构建输出：`dist/`（或按配置）

---

## 验证过程中修改的文件

{如果有文件被修改以修复问题：}

| 文件 | 变更内容 |
|------|---------|
| `src/file.ts` | 修复类型错误 |
| `src/other.ts` | 代码检查自动修复 |

---

## 下一步

继续执行 `archon-finalize-pr` 以更新 PR 并标记为可审查状态。
```

**阶段 3 检查点：**

- [ ] 验证产物已写入
- [ ] 所有结果已记录

---

## 阶段 4：输出 - 报告结果

### 如果全部通过：

```markdown
## 验证完成 ✅

**工作流 ID**：`$WORKFLOW_ID`

### 结果

| 检查项 | 状态 |
|-------|--------|
| 类型检查 | ✅ |
| 代码检查 | ✅ |
| 格式检查 | ✅ |
| 测试 | ✅（{N} 个通过） |
| 构建 | ✅ |

{如果修复了问题：}
### 已修复的问题

- {N} 个类型错误已修复
- {M} 个代码检查问题已修复
- {K} 个格式问题已修复

### 产物

结果已写入：`$ARTIFACTS_DIR/validation.md`

### 下一步

继续执行 `archon-finalize-pr` 以更新 PR 并标记为可审查状态。
```

### 如果被阻塞（无法修复的问题）：

```markdown
## 验证被阻塞 ❌

**工作流 ID**：`$WORKFLOW_ID`

### 失败的检查项

**{检查项名称}**：{错误描述}

### 修复尝试

1. {尝试了什么}
2. {尝试了什么}

### 需要的操作

此问题需要人工干预：

{需要执行的操作描述}

### 产物

部分结果已写入：`$ARTIFACTS_DIR/validation.md`
```

---

## 成功标准

- **TYPE_CHECK_PASS**：`{runner} run type-check` 退出码为 0
- **LINT_PASS**：`{runner} run lint` 退出码为 0
- **FORMAT_PASS**：`{runner} run format:check` 退出码为 0
- **TESTS_PASS**：`{runner} test` 全部通过
- **BUILD_PASS**：`{runner} run build` 退出码为 0
- **ARTIFACT_WRITTEN**：验证结果已记录

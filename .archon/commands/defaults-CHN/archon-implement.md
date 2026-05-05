---
description: 执行实施计划，配合严格的验证循环
argument-hint: <path/to/plan.md or GitHub issue URL>
---

# 实施计划

**计划**：$ARGUMENTS

---

## 你的任务

端到端执行计划，配合严格的自我验证。你是自主运行的。

**核心理念**：验证循环能尽早发现错误。每次更改后运行检查。立即修复问题。目标是可工作的实现，而不仅仅是写出来的代码。

**黄金法则**：如果验证失败，先修复再继续。绝不积累破损状态。

---

## 阶段 0：检测 - 项目环境

### 0.1 识别包管理器

检查这些文件以确定项目的工具链：

| 找到的文件 | 包管理器 | 运行器 |
|------------|----------|--------|
| `bun.lockb` | bun | `bun` / `bun run` |
| `pnpm-lock.yaml` | pnpm | `pnpm` / `pnpm run` |
| `yarn.lock` | yarn | `yarn` / `yarn run` |
| `package-lock.json` | npm | `npm run` |
| `pyproject.toml` | uv/pip | `uv run` / `python` |
| `Cargo.toml` | cargo | `cargo` |
| `go.mod` | go | `go` |

**存储检测到的运行器** - 用于所有后续命令。

### 0.2 识别验证脚本

检查 `package.json`（或等效文件）中的可用脚本：
- 类型检查：`type-check`、`typecheck`、`tsc`
- 代码检查：`lint`、`lint:fix`
- 测试：`test`、`test:unit`、`test:integration`
- 构建：`build`、`compile`

**使用计划的 "Validation Commands" 部分** - 它应指定此项目的确切命令。

---

## 阶段 1：加载 - 阅读计划

### 1.1 加载计划文件

```bash
cat $ARGUMENTS
```

如果 `$ARGUMENTS` 是 GitHub Issue URL 或编号（例如 `#123`），获取 Issue 正文内容，其中包含计划。

### 1.2 提取关键部分

定位并理解：

- **Summary** - 我们要构建什么
- **Patterns to Mirror** - 要参考的代码
- **Files to Change** - CREATE/UPDATE 列表
- **Step-by-Step Tasks** - 实施顺序
- **Validation Commands** - 如何验证（使用这些，而非硬编码命令）
- **Acceptance Criteria** - 完成定义

### 1.3 验证计划存在

**如果未找到计划：**

```
Error: Plan not found at $ARGUMENTS

Provide a valid plan path or GitHub issue containing the plan.
```

**阶段 1 检查点：**

- [ ] 计划文件已加载
- [ ] 关键部分已识别
- [ ] 任务列表已提取

---

## 阶段 2：准备 - Git 状态

### 2.1 检查当前状态

```bash
# 当前在哪个分支？
git branch --show-current

# 是否在工作树中？
git rev-parse --show-toplevel
git worktree list

# 工作目录是否干净？
git status --porcelain
```

### 2.2 分支决策

```text
┌─ 在工作树中？
│  └─ 是 → 按原样使用当前分支。不要切换分支。不要创建
│           新分支。隔离系统已设置好正确的
│           分支；任何偏离都会操作错误的代码。
│           日志："Using worktree at {path} on branch {branch}"
│
├─ 在 $BASE_BRANCH 上？（main、master 或配置的 base 分支）
│  └─ 问：工作目录是否干净？
│     ├─ 是 → 创建分支：git checkout -b feature/{plan-slug}
│     │        （仅适用于工作树外——例如手动 CLI 使用）
│     └─ 否 → 停止："请先暂存或提交更改"
│
├─ 在其他分支上？
│  └─ 按原样使用。不要切换到其他分支（例如 `git branch`
│     显示但未检出的分支）。
│     日志："Using existing branch {name}"
│
└─ 有未提交的更改？
   └─ 停止："请先暂存或提交更改"
```

### 2.3 与远程同步

```bash
git fetch origin
git pull --rebase origin $BASE_BRANCH 2>/dev/null || true
```

**阶段 2 检查点：**

- [ ] 在正确的分支上（不是有未提交更改的 $BASE_BRANCH）
- [ ] 工作目录就绪
- [ ] 已与远程同步

---

## 阶段 3：执行 - 实施任务

**针对计划的 "Step-by-Step Tasks" 部分中的每个任务：**

### 3.1 阅读上下文

1. 阅读任务中的 **MIRROR** 文件引用
2. 理解要遵循的模式
3. 阅读指定的 **IMPORTS**

### 3.2 实施

1. 按照说明精确地进行更改
2. 遵循 MIRROR 引用的模式
3. 处理任何 **GOTCHA** 警告

### 3.3 立即验证

**每次文件更改后，运行计划 "Validation Commands" 部分中的类型检查命令。**

常见模式：
- `{runner} run type-check`（JS/TS 项目）
- `mypy .`（Python）
- `cargo check`（Rust）
- `go build ./...`（Go）

**如果类型检查失败：**

1. 阅读错误
2. 修复问题
3. 重新运行类型检查
4. 只有通过后才继续

### 3.4 跟踪进度

完成每个任务时进行记录：

```
Task 1: CREATE src/features/x/models.ts ✅
Task 2: CREATE src/features/x/service.ts ✅
Task 3: UPDATE src/routes/index.ts ✅
```

**偏差处理：**
如果必须偏离计划：

- 记录改变了什么
- 记录为什么改变
- 继续执行，并记录偏差

**阶段 3 检查点：**

- [ ] 所有任务按顺序执行
- [ ] 每个任务通过类型检查
- [ ] 偏差已记录

---

## 阶段 4：验证 - 全面检查

### 4.1 静态分析

**运行计划 "Validation Commands" 部分中的类型检查和代码检查命令。**

常见模式：
- JS/TS：`{runner} run type-check && {runner} run lint`
- Python：`ruff check . && mypy .`
- Rust：`cargo check && cargo clippy`
- Go：`go vet ./...`

**必须零错误通过。**

如果有 lint 错误：

1. 运行 lint 修复命令（例如 `{runner} run lint:fix`、`ruff check --fix .`）
2. 重新检查
3. 手动修复剩余问题

### 4.2 单元测试

**你必须为新代码编写或更新测试。** 这不是可选的。

**测试要求：**

1. 每个新函数/功能至少需要一个测试
2. 计划中识别的边界情况需要测试
3. 如果行为更改，更新现有测试

**编写测试**，然后运行计划中的测试命令。

常见模式：
- JS/TS：`{runner} test` 或 `{runner} run test`
- Python：`pytest` 或 `uv run pytest`
- Rust：`cargo test`
- Go：`go test ./...`

**如果测试失败：**

1. 阅读失败输出
2. 判断：实现 bug 还是测试 bug？
3. 修复实际问题
4. 重新运行测试
5. 重复直到全部通过

### 4.3 构建检查

**运行计划 "Validation Commands" 部分中的构建命令。**

常见模式：
- JS/TS：`{runner} run build`
- Python：不适用（解释型语言）或 `uv build`
- Rust：`cargo build --release`
- Go：`go build ./...`

**必须无错误完成。**

### 4.4 集成测试（如适用）

**如果计划涉及 API/服务器更改，使用计划中的集成测试命令。**

示例模式：
```bash
# 在后台启动服务器（命令因项目而异）
{runner} run dev &
SERVER_PID=$!
sleep 3

# 测试端点（根据项目配置调整 URL/端口）
curl -s http://localhost:{port}/health | jq

# 停止服务器
kill $SERVER_PID
```

### 4.5 边界情况测试

运行计划中指定的任何边界情况测试。

**阶段 4 检查点：**

- [ ] 类型检查通过（使用计划中的命令）
- [ ] 代码检查通过（0 错误）
- [ ] 测试通过（全部通过）
- [ ] 构建成功
- [ ] 集成测试通过（如适用）

---

## 阶段 5：报告 - 创建实施报告

### 5.1 创建报告目录

```bash
mkdir -p $ARTIFACTS_DIR/../reports
```

### 5.2 生成报告

**路径**：`$ARTIFACTS_DIR/../reports/{plan-name}-report.md`

```markdown
# Implementation Report

**Plan**: `$ARGUMENTS`
**Source Issue**: #{number} (if applicable)
**Branch**: `{branch-name}`
**Date**: {YYYY-MM-DD}
**Status**: {COMPLETE | PARTIAL}

---

## Summary

{Brief description of what was implemented}

---

## Assessment vs Reality

Compare the original plan's assessment with what actually happened:

| Metric     | Predicted   | Actual   | Reasoning                                                                      |
| ---------- | ----------- | -------- | ------------------------------------------------------------------------------ |
| Complexity | {from plan} | {actual} | {Why it matched or differed - e.g., "discovered additional integration point"} |
| Confidence | {from plan} | {actual} | {e.g., "root cause was correct" or "had to pivot because X"}                   |

**If implementation deviated from the plan, explain why:**

- {What changed and why - based on what you discovered during implementation}

---

## Tasks Completed

| #   | Task               | File       | Status |
| --- | ------------------ | ---------- | ------ |
| 1   | {task description} | `src/x.ts` | ✅     |
| 2   | {task description} | `src/y.ts` | ✅     |

---

## Validation Results

| Check       | Result | Details               |
| ----------- | ------ | --------------------- |
| Type check  | ✅     | No errors             |
| Lint        | ✅     | 0 errors, N warnings  |
| Unit tests  | ✅     | X passed, 0 failed    |
| Build       | ✅     | Compiled successfully |
| Integration | ✅/⏭️  | {result or "N/A"}     |

---

## Files Changed

| File       | Action | Lines     |
| ---------- | ------ | --------- |
| `src/x.ts` | CREATE | +{N}      |
| `src/y.ts` | UPDATE | +{N}/-{M} |

---

## Deviations from Plan

{List any deviations with rationale, or "None"}

---

## Issues Encountered

{List any issues and how they were resolved, or "None"}

---

## Tests Written

| Test File       | Test Cases               |
| --------------- | ------------------------ |
| `src/x.test.ts` | {list of test functions} |

---

## Next Steps

- [ ] Review implementation
- [ ] Create PR (next step in workflow)
- [ ] Merge when approved
```

### 5.3 归档计划

```bash
mkdir -p $ARTIFACTS_DIR/../plans/completed
cp $ARGUMENTS $ARTIFACTS_DIR/../plans/completed/ 2>/dev/null || true
```

**阶段 5 检查点：**

- [ ] 报告已创建于 `$ARTIFACTS_DIR/../reports/`
- [ ] 计划已复制到 completed 文件夹（如果是本地文件）

---

## 阶段 6：输出 - 向用户报告

```markdown
## Implementation Complete

**Plan**: `$ARGUMENTS`
**Source Issue**: #{number} (if applicable)
**Branch**: `{branch-name}`
**Status**: Complete

### Validation Summary

| Check      | Result          |
| ---------- | --------------- |
| Type check | ✅              |
| Lint       | ✅              |
| Tests      | ✅ ({N} passed) |
| Build      | ✅              |

### Files Changed

- {N} files created
- {M} files updated
- {K} tests written

### Deviations

{If none: "Implementation matched the plan."}
{If any: Brief summary of what changed and why}

### Artifacts

- Report: `$ARTIFACTS_DIR/../reports/{name}-report.md`

### Next Steps

1. Review the report (especially if deviations noted)
2. Create PR (next workflow step)
3. Merge when approved
```

---

## 处理失败

### 类型检查失败

1. 仔细阅读错误消息
2. 修复类型问题
3. 重新运行类型检查命令
4. 通过前不要继续

### 测试失败

1. 识别哪个测试失败
2. 判断：实现 bug 还是测试 bug？
3. 修复根本原因（通常是实现）
4. 重新运行测试
5. 重复直到全部通过

### 代码检查失败

1. 运行 lint 修复命令处理可自动修复的问题
2. 手动修复剩余问题
3. 重新运行 lint
4. 干净后继续

### 构建失败

1. 通常是类型或导入问题
2. 检查错误输出
3. 修复并重新运行

### 集成测试失败

1. 检查服务器是否正确启动
2. 验证端点是否存在
3. 检查请求格式
4. 修复实现并重试

---

## 成功标准

- **TASKS_COMPLETE**：所有计划任务已执行
- **TYPES_PASS**：类型检查命令退出码为 0
- **LINT_PASS**：代码检查命令退出码为 0（允许警告）
- **TESTS_PASS**：测试命令全部通过
- **BUILD_PASS**：构建命令成功
- **REPORT_CREATED**：实施报告已存在

---
description: 从主分支启动 Archon，使用 agent-browser 通过 E2E 测试复现缺陷
argument-hint: (none - reads from artifacts)
---

# E2E 测试：主分支（复现缺陷）

从**主分支**代码启动 Archon，使用浏览器自动化复现 PR 中描述的缺陷或漏洞。截图作为证据。

**关键**：所有浏览器交互必须使用 `agent-browser` CLI。加载 `/agent-browser` 技能以获取完整命令参考。

**关键**：完成前必须清理所有生成的进程。记录 PID 并在阶段 4 中终止。

**关键——会话隔离**：此工作流与其他 validate-pr 实例并行运行。
每个 `agent-browser` 命令都必须使用 `--session $WORKFLOW_ID` 以隔离浏览器会话。
示例：`agent-browser --session $WORKFLOW_ID open "http://..."`、`agent-browser --session $WORKFLOW_ID snapshot -i` 等。
会话 ID 已写入 `$ARTIFACTS_DIR/.browser-session` 以便清理。

**绝对禁止——切勿执行以下操作**：
- `taskkill //F //IM chrome.exe` 或任何按映像名称终止 chrome 的变体——这会终止用户的浏览器
- `taskkill //F //IM node.exe` 或 `taskkill //F //IM bun.exe`——这会终止 Claude Code、Archon 服务器和所有其他工作流
- `pkill chrome`、`pkill node`、`pkill bun` 或任何广泛的进程名终止操作
- 不带 `--session $WORKFLOW_ID` 的 `agent-browser close`——这会终止其他工作流的浏览器会话
- 任何"全部终止"或"全部清除"的升级模式——如果 agent-browser 不工作，跳过 E2E 测试并在报告中注明
- 如果 agent-browser 连续 2 次连接失败，停止尝试，仅基于代码审查撰写分析结果

---

## 阶段 1：加载上下文

### 1.1 读取产物

```bash
PR_NUMBER=$(cat $ARTIFACTS_DIR/.pr-number | tr -d '\n')
BACKEND_PORT=$(cat $ARTIFACTS_DIR/.backend-port | tr -d '\n')
FRONTEND_PORT=$(cat $ARTIFACTS_DIR/.frontend-port | tr -d '\n')
CANONICAL_REPO=$(cat $ARTIFACTS_DIR/.canonical-repo | tr -d '\n')
echo "PR: #$PR_NUMBER"
echo "Backend port: $BACKEND_PORT"
echo "Frontend port: $FRONTEND_PORT"
echo "Main repo: $CANONICAL_REPO"
```

### 1.2 读取 PR 和测试计划

```bash
PR_NUMBER=$(cat $ARTIFACTS_DIR/.pr-number | tr -d '\n')
gh pr view "$PR_NUMBER" --json title,body
```

```bash
# 读取主分支代码审查以了解测试范围
cat $ARTIFACTS_DIR/code-review-main.md 2>/dev/null || echo "No main branch review available yet"
```

### 1.3 可测试性分类

可测试性分类器已确定：
- **结论**：$classify-testability.output.testable
- **理由**：$classify-testability.output.reasoning
- **测试计划**：$classify-testability.output.test_plan

结合上述测试计划与 PR 描述和代码审查，构建执行计划：
- 哪些用户旅程可以复现缺陷？
- 异常行为应该是什么样？
- 哪些截图可以证明缺陷存在？

---

## 阶段 2：在主分支上启动 Archon

### 2.1 创建独立的主分支工作树

**重要**：使用专用工作树而非修改规范仓库。这对并发验证运行是安全的——每次运行都有自己的独立检出。

```bash
CANONICAL_REPO=$(cat $ARTIFACTS_DIR/.canonical-repo | tr -d '\n')
PR_BASE=$(cat $ARTIFACTS_DIR/.pr-base | tr -d '\n')
PR_NUMBER=$(cat $ARTIFACTS_DIR/.pr-number | tr -d '\n')

# 为主分支 E2E 测试创建独立工作树
MAIN_E2E_PATH="$ARTIFACTS_DIR/main-checkout"
git -C "$CANONICAL_REPO" fetch origin "$PR_BASE" --quiet
git -C "$CANONICAL_REPO" worktree add "$MAIN_E2E_PATH" "origin/$PR_BASE" --detach --quiet
echo "$MAIN_E2E_PATH" > "$ARTIFACTS_DIR/.e2e-main-worktree"
echo "Main E2E worktree at: $MAIN_E2E_PATH"
echo "Base branch: $PR_BASE @ $(git -C "$MAIN_E2E_PATH" log --oneline -1)"
```

### 2.2 安装依赖

```bash
MAIN_E2E_PATH=$(cat $ARTIFACTS_DIR/.e2e-main-worktree | tr -d '\n')
cd "$MAIN_E2E_PATH" && bun install --frozen-lockfile 2>/dev/null || bun install
```

### 2.3 在自定义端口上启动后端

**重要**：记录 PID 以便后续终止。服务器输出已记录用于调试。

```bash
MAIN_E2E_PATH=$(cat $ARTIFACTS_DIR/.e2e-main-worktree | tr -d '\n')
BACKEND_PORT=$(cat $ARTIFACTS_DIR/.backend-port | tr -d '\n')

cd "$MAIN_E2E_PATH" && PORT=$BACKEND_PORT bun run --filter @archon/server dev > "$ARTIFACTS_DIR/.e2e-main-backend.log" 2>&1 &
BACKEND_PID=$!
echo "$BACKEND_PID" > "$ARTIFACTS_DIR/.e2e-main-backend-pid"
echo "Backend started with PID: $BACKEND_PID"

# 轮询直到健康（最多 60 秒）
MAX_WAIT=60
WAITED=0
until curl -sf "http://localhost:$BACKEND_PORT/api/health" > /dev/null 2>&1; do
  if [ $WAITED -ge $MAX_WAIT ]; then
    echo "ERROR: Backend did not become healthy within ${MAX_WAIT}s"
    echo "Last log lines:"
    tail -20 "$ARTIFACTS_DIR/.e2e-main-backend.log" 2>/dev/null || true
    exit 1
  fi
  sleep 2
  WAITED=$((WAITED + 2))
done
echo "Backend healthy after ${WAITED}s"
curl -s "http://localhost:$BACKEND_PORT/api/health" | head -c 200
echo ""
```

### 2.4 在自定义端口上启动前端

```bash
MAIN_E2E_PATH=$(cat $ARTIFACTS_DIR/.e2e-main-worktree | tr -d '\n')
BACKEND_PORT=$(cat $ARTIFACTS_DIR/.backend-port | tr -d '\n')
FRONTEND_PORT=$(cat $ARTIFACTS_DIR/.frontend-port | tr -d '\n')

cd "$MAIN_E2E_PATH/packages/web" && PORT=$BACKEND_PORT npx vite --port $FRONTEND_PORT --host > "$ARTIFACTS_DIR/.e2e-main-frontend.log" 2>&1 &
FRONTEND_PID=$!
echo "$FRONTEND_PID" > "$ARTIFACTS_DIR/.e2e-main-frontend-pid"
echo "Frontend started with PID: $FRONTEND_PID"

# 轮询直到就绪（最多 60 秒）
MAX_WAIT=60
WAITED=0
until curl -sf "http://localhost:$FRONTEND_PORT" > /dev/null 2>&1; do
  if [ $WAITED -ge $MAX_WAIT ]; then
    echo "ERROR: Frontend did not become ready within ${MAX_WAIT}s"
    echo "Last log lines:"
    tail -20 "$ARTIFACTS_DIR/.e2e-main-frontend.log" 2>/dev/null || true
    exit 1
  fi
  sleep 2
  WAITED=$((WAITED + 2))
done
echo "Frontend ready after ${WAITED}s"
curl -s "http://localhost:$FRONTEND_PORT" | head -c 100
echo ""
```

### 2.5 初始化测试数据（如需要）

```bash
BACKEND_PORT=$(cat $ARTIFACTS_DIR/.backend-port | tr -d '\n')

# 检查代码库是否存在
CODEBASE_COUNT=$(curl -s "http://localhost:$BACKEND_PORT/api/codebases" | grep -c '"id"' || echo 0)

if [ "$CODEBASE_COUNT" -eq 0 ]; then
  MAIN_E2E_PATH=$(cat $ARTIFACTS_DIR/.e2e-main-worktree | tr -d '\n')
  curl -s -X POST "http://localhost:$BACKEND_PORT/api/codebases" \
    -H "Content-Type: application/json" \
    -d "{\"path\": \"$MAIN_E2E_PATH\"}"
fi
```

---

## 阶段 3：浏览器测试（复现缺陷）

### 3.1 加载 Agent-Browser 技能

**现在必须加载 AGENT-BROWSER 技能。** 使用 `/agent-browser` 或调用技能。这将提供浏览器自动化的完整命令参考。

### 3.2 核心浏览器工作流

每次交互都遵循以下模式：

```bash
# 0. 存储会话 ID 以便清理
echo "$WORKFLOW_ID" > "$ARTIFACTS_DIR/.browser-session"

# 1. 打开 Archon UI（始终使用 --session）
FRONTEND_PORT=$(cat $ARTIFACTS_DIR/.frontend-port | tr -d '\n')
agent-browser --session $WORKFLOW_ID open "http://localhost:$FRONTEND_PORT"

# 2. 等待应用加载
agent-browser --session $WORKFLOW_ID wait --load networkidle

# 3. 获取可交互元素
agent-browser --session $WORKFLOW_ID snapshot -i

# 4. 截取初始状态截图
agent-browser --session $WORKFLOW_ID screenshot "$ARTIFACTS_DIR/e2e-main-01-initial.png"

# 5. 使用快照中的引用进行交互
# agent-browser --session $WORKFLOW_ID click @e1
# agent-browser --session $WORKFLOW_ID fill @e2 "text"

# 6. DOM 变更后重新获取快照
# agent-browser --session $WORKFLOW_ID snapshot -i

# 7. 在每个关键节点截图
# agent-browser --session $WORKFLOW_ID screenshot "$ARTIFACTS_DIR/e2e-main-02-{step}.png"
```

### 3.3 执行测试计划

遵循从 PR 描述和代码审查中得出的测试计划。对**每个**测试用例：

1. **设置前置条件** —— 导航到正确页面，根据需要创建会话/工作流
2. **执行复现步骤** —— 严格按照 Issue/PR 中的描述
3. **收集证据** —— 在操作**前**、操作**中**和操作**后**截图
4. **验证异常行为** —— 确认所见与报告的缺陷一致
5. **阅读每张截图** —— 使用 Read 工具对截图进行视觉检查
6. **记录观察结果** —— 注明准确的错误信息、视觉异常、缺失元素

### 3.4 API 交叉验证

对于涉及数据完整性或 SSE 的缺陷，通过直接 API 调用与 UI 进行交叉验证：

```bash
BACKEND_PORT=$(cat $ARTIFACTS_DIR/.backend-port | tr -d '\n')

# 检查会话列表
curl -s "http://localhost:$BACKEND_PORT/api/conversations" | head -c 500

# 检查特定会话的消息
# curl -s "http://localhost:$BACKEND_PORT/api/conversations/{id}/messages"

# 检查工作流运行
# curl -s "http://localhost:$BACKEND_PORT/api/workflows/runs"
```

---

## 阶段 4：清理和报告

**关键：必须在撰写分析结果前完成清理。孤立进程会累积并导致系统崩溃。**

### 4.1 关闭浏览器

```bash
# 始终使用 --session，仅关闭你的浏览器，不影响其他工作流
agent-browser --session $WORKFLOW_ID close 2>/dev/null || true
```

### 4.2 停止主分支的 Archon（跨平台）

通过 PID（阶段 2 中记录）和端口（备用方式）终止进程。适用于 Windows 和 Unix。

```bash
BACKEND_PORT=$(cat $ARTIFACTS_DIR/.backend-port | tr -d '\n')
FRONTEND_PORT=$(cat $ARTIFACTS_DIR/.frontend-port | tr -d '\n')

# 通过记录的 PID 终止（主要方法）
for pidfile in "$ARTIFACTS_DIR/.e2e-main-backend-pid" "$ARTIFACTS_DIR/.e2e-main-frontend-pid"; do
  if [ -f "$pidfile" ]; then
    PID=$(cat "$pidfile" | tr -d '\n')
    echo "Killing PID $PID from $pidfile"
    # 先尝试 Unix kill，然后尝试 Windows taskkill
    kill "$PID" 2>/dev/null || taskkill //F //T //PID "$PID" 2>/dev/null || true
  fi
done

# 备用：按端口终止（处理 PID 终止可能遗漏的子进程）
# Unix：fuser/lsof，Windows：netstat + taskkill
for PORT in $BACKEND_PORT $FRONTEND_PORT; do
  echo "Cleaning up port $PORT..."
  # 尝试 fuser（Linux）
  fuser -k "$PORT/tcp" 2>/dev/null || true
  # 尝试 lsof（macOS/Linux）
  lsof -ti:"$PORT" 2>/dev/null | xargs kill -9 2>/dev/null || true
  # 尝试 netstat（Windows - Git Bash）
  netstat -ano 2>/dev/null | grep ":$PORT " | grep LISTENING | awk '{print $5}' | sort -u | while read pid; do
    taskkill //F //T //PID "$pid" 2>/dev/null || true
  done
done

sleep 2
echo "Process cleanup complete"
```

### 4.3 移除主分支工作树

```bash
CANONICAL_REPO=$(cat $ARTIFACTS_DIR/.canonical-repo | tr -d '\n')
MAIN_E2E_PATH=$(cat "$ARTIFACTS_DIR/.e2e-main-worktree" 2>/dev/null | tr -d '\n')
if [ -n "$MAIN_E2E_PATH" ] && [ -d "$MAIN_E2E_PATH" ]; then
  echo "Removing main E2E worktree: $MAIN_E2E_PATH"
  git -C "$CANONICAL_REPO" worktree remove "$MAIN_E2E_PATH" --force 2>/dev/null || rm -rf "$MAIN_E2E_PATH"
fi
echo "Worktree cleanup complete"
```

### 4.4 撰写分析结果

写入 `$ARTIFACTS_DIR/e2e-main.md`：

```markdown
# E2E 测试结果：主分支

**PR**：#{number}
**分支**：main @ {commit}
**后端端口**：{port}
**前端端口**：{port}
**截图**：$ARTIFACTS_DIR/e2e-main-*.png

## 测试摘要

| 测试用例 | 结果 | 证据 |
|-----------|--------|----------|
| {测试 1} | 缺陷已复现 / 未复现 | e2e-main-{N}.png |
| {测试 2} | 缺陷已复现 / 未复现 | e2e-main-{N}.png |

## 详细结果

### 测试 1：{描述}
**步骤**：{执行的操作}
**预期**：{修复版本应有的行为}
**实际**：{主分支上发生的情况——即缺陷}
**截图**：`$ARTIFACTS_DIR/e2e-main-{N}.png`

### 测试 2：{描述}
{相同结构...}

## 额外发现的问题
{测试过程中发现的其他缺陷或 UX 问题}

## 复现置信度
**高 / 中 / 低 / 无法复现**

{说明置信度水平。如无法复现，说明已尝试的方式。}
```

---

## 成功标准

- **ARCHON_STARTED**：后端和前端已在分配的端口上运行
- **BROWSER_TESTED**：所有测试用例已通过 agent-browser 执行
- **SCREENSHOTS_TAKEN**：每个测试用例已收集证据截图
- **BUG_ASSESSED**：每项 PR 声明已在主分支上测试
- **ARCHON_STOPPED**：进程已终止，端口已释放——**完成前验证端口已释放**
- **ARTIFACT_WRITTEN**：`$ARTIFACTS_DIR/e2e-main.md` 已创建

---
description: 从功能分支启动 Archon，使用 agent-browser 验证修复是否正确
argument-hint: (none - reads from artifacts)
---

# E2E 测试：功能分支（验证修复）

从**功能分支**（当前工作树）启动 Archon，使用浏览器自动化验证缺陷已修复且 UI/UX 正确。截图作为证据。

**关键**：所有浏览器交互必须使用 `agent-browser` CLI。加载 `/agent-browser` 技能以获取完整命令参考。

**关键**：完成前必须清理所有生成的进程。记录 PID 并在阶段 4 中终止它们。上一次 E2E 运行的孤立进程可能仍在运行——先检查并终止。

**关键——会话隔离**：此工作流与其他 validate-pr 实例并行运行。
每个 `agent-browser` 命令都必须使用 `--session $WORKFLOW_ID` 以隔离浏览器会话。
示例：`agent-browser --session $WORKFLOW_ID open "http://..."`、`agent-browser --session $WORKFLOW_ID snapshot -i` 等。

**绝对禁止——切勿执行以下操作**：
- `taskkill //F //IM chrome.exe` 或任何按映像名称终止 chrome 的变体——这会终止用户的浏览器
- `taskkill //F //IM node.exe` 或 `taskkill //F //IM bun.exe`——这会终止 Claude Code、Archon 服务器和所有其他工作流
- `pkill chrome`、`pkill node`、`pkill bun` 或任何广泛的进程名终止操作
- 不带 `--session $WORKFLOW_ID` 的 `agent-browser close`——这会终止其他工作流的浏览器会话
- 任何"全部终止"或"全部清除"的升级模式——如果 agent-browser 不工作，跳过 E2E 测试并在报告中注明
- 如果 agent-browser 连续 2 次连接失败，停止尝试，仅基于代码审查撰写分析结果

---

## 阶段 0：终止上一次 E2E 运行的孤立进程

开始前，清理主分支 E2E 测试遗留的进程：

```bash
BACKEND_PORT=$(cat $ARTIFACTS_DIR/.backend-port | tr -d '\n')
FRONTEND_PORT=$(cat $ARTIFACTS_DIR/.frontend-port | tr -d '\n')

# 通过主 E2E 运行的 PID 文件终止
for pidfile in "$ARTIFACTS_DIR/.e2e-main-backend-pid" "$ARTIFACTS_DIR/.e2e-main-frontend-pid"; do
  if [ -f "$pidfile" ]; then
    PID=$(cat "$pidfile" | tr -d '\n')
    echo "Killing leftover main E2E PID $PID"
    kill "$PID" 2>/dev/null || taskkill //F //T //PID "$PID" 2>/dev/null || true
  fi
done

# 终止占用端口的进程
for PORT in $BACKEND_PORT $FRONTEND_PORT; do
  fuser -k "$PORT/tcp" 2>/dev/null || true
  lsof -ti:"$PORT" 2>/dev/null | xargs kill -9 2>/dev/null || true
  netstat -ano 2>/dev/null | grep ":$PORT " | grep LISTENING | awk '{print $5}' | sort -u | while read pid; do
    taskkill //F //T //PID "$pid" 2>/dev/null || true
  done
done
sleep 2
echo "Orphan cleanup complete"
```

---

## 阶段 1：加载上下文

### 1.1 读取产物

```bash
PR_NUMBER=$(cat $ARTIFACTS_DIR/.pr-number | tr -d '\n')
BACKEND_PORT=$(cat $ARTIFACTS_DIR/.backend-port | tr -d '\n')
FRONTEND_PORT=$(cat $ARTIFACTS_DIR/.frontend-port | tr -d '\n')
WORKTREE_PATH=$(cat $ARTIFACTS_DIR/.worktree-path | tr -d '\n')
echo "PR: #$PR_NUMBER"
echo "Backend port: $BACKEND_PORT"
echo "Frontend port: $FRONTEND_PORT"
echo "Feature branch path: $WORKTREE_PATH"
```

### 1.2 读取主分支测试结果

```bash
cat $ARTIFACTS_DIR/e2e-main.md 2>/dev/null || echo "No main branch E2E results available"
```

这会告诉你：
- 哪些缺陷在主分支上被复现（需要在此验证它们已**修复**）
- 需要重新运行哪些测试用例
- 需要对比哪些截图

### 1.3 读取代码审查

```bash
cat $ARTIFACTS_DIR/code-review-main.md 2>/dev/null || echo ""
cat $ARTIFACTS_DIR/code-review-feature.md 2>/dev/null || echo ""
```

---

## 阶段 2：在功能分支上启动 Archon

### 2.1 安装依赖（如需要）

```bash
WORKTREE_PATH=$(cat $ARTIFACTS_DIR/.worktree-path | tr -d '\n')
cd "$WORKTREE_PATH" && bun install --frozen-lockfile 2>/dev/null || bun install
```

### 2.2 在自定义端口上启动后端

**重要**：记录 PID 以便后续终止。将输出重定向到 /dev/null 以防止终端信息干扰。

```bash
WORKTREE_PATH=$(cat $ARTIFACTS_DIR/.worktree-path | tr -d '\n')
BACKEND_PORT=$(cat $ARTIFACTS_DIR/.backend-port | tr -d '\n')

cd "$WORKTREE_PATH" && PORT=$BACKEND_PORT bun run --filter @archon/server dev > "$ARTIFACTS_DIR/.e2e-feature-backend.log" 2>&1 &
BACKEND_PID=$!
echo "$BACKEND_PID" > "$ARTIFACTS_DIR/.e2e-feature-backend-pid"
echo "Backend started with PID: $BACKEND_PID"

# 轮询直到健康（最多 60 秒）
MAX_WAIT=60
WAITED=0
until curl -sf "http://localhost:$BACKEND_PORT/api/health" > /dev/null 2>&1; do
  if [ $WAITED -ge $MAX_WAIT ]; then
    echo "ERROR: Backend did not become healthy within ${MAX_WAIT}s"
    echo "Last log lines:"
    tail -20 "$ARTIFACTS_DIR/.e2e-feature-backend.log" 2>/dev/null || true
    exit 1
  fi
  sleep 2
  WAITED=$((WAITED + 2))
done
echo "Backend healthy after ${WAITED}s"
curl -s "http://localhost:$BACKEND_PORT/api/health" | head -c 200
echo ""
```

### 2.3 在自定义端口上启动前端

```bash
WORKTREE_PATH=$(cat $ARTIFACTS_DIR/.worktree-path | tr -d '\n')
BACKEND_PORT=$(cat $ARTIFACTS_DIR/.backend-port | tr -d '\n')
FRONTEND_PORT=$(cat $ARTIFACTS_DIR/.frontend-port | tr -d '\n')

cd "$WORKTREE_PATH/packages/web" && PORT=$BACKEND_PORT npx vite --port $FRONTEND_PORT --host > "$ARTIFACTS_DIR/.e2e-feature-frontend.log" 2>&1 &
FRONTEND_PID=$!
echo "$FRONTEND_PID" > "$ARTIFACTS_DIR/.e2e-feature-frontend-pid"
echo "Frontend started with PID: $FRONTEND_PID"

# 轮询直到就绪（最多 60 秒）
MAX_WAIT=60
WAITED=0
until curl -sf "http://localhost:$FRONTEND_PORT" > /dev/null 2>&1; do
  if [ $WAITED -ge $MAX_WAIT ]; then
    echo "ERROR: Frontend did not become ready within ${MAX_WAIT}s"
    echo "Last log lines:"
    tail -20 "$ARTIFACTS_DIR/.e2e-feature-frontend.log" 2>/dev/null || true
    exit 1
  fi
  sleep 2
  WAITED=$((WAITED + 2))
done
echo "Frontend ready after ${WAITED}s"
curl -s "http://localhost:$FRONTEND_PORT" | head -c 100
echo ""
```

### 2.4 初始化测试数据（如需要）

```bash
BACKEND_PORT=$(cat $ARTIFACTS_DIR/.backend-port | tr -d '\n')

# 检查代码库是否存在
CODEBASE_COUNT=$(curl -s "http://localhost:$BACKEND_PORT/api/codebases" | grep -c '"id"' || echo 0)

if [ "$CODEBASE_COUNT" -eq 0 ]; then
  WORKTREE_PATH=$(cat $ARTIFACTS_DIR/.worktree-path | tr -d '\n')
  curl -s -X POST "http://localhost:$BACKEND_PORT/api/codebases" \
    -H "Content-Type: application/json" \
    -d "{\"path\": \"$WORKTREE_PATH\"}"
fi
```

---

## 阶段 3：浏览器测试（验证修复）

### 3.1 加载 Agent-Browser 技能

**现在必须加载 AGENT-BROWSER 技能。** 使用 `/agent-browser` 或调用技能。这将提供浏览器自动化的完整命令参考。

### 3.2 核心浏览器工作流

```bash
# 1. 打开 Archon UI（始终使用 --session）
FRONTEND_PORT=$(cat $ARTIFACTS_DIR/.frontend-port | tr -d '\n')
agent-browser --session $WORKFLOW_ID open "http://localhost:$FRONTEND_PORT"

# 2. 等待应用加载
agent-browser --session $WORKFLOW_ID wait --load networkidle

# 3. 获取可交互元素
agent-browser --session $WORKFLOW_ID snapshot -i

# 4. 截取初始状态截图
agent-browser --session $WORKFLOW_ID screenshot "$ARTIFACTS_DIR/e2e-feature-01-initial.png"
```

### 3.3 重新运行主分支的所有测试用例

对主分支上运行的**每个**测试用例，在功能分支上重新运行：

1. **相同前置条件** —— 建立完全相同的起始状态
2. **相同复现步骤** —— 执行完全相同的操作
3. **验证修复** —— 缺陷现在应该**不存在**
4. **收集证据** —— 在与主分支相同的节点截图，用于并排对比
5. **阅读每张截图** —— 使用 Read 工具进行视觉检查
6. **与主分支对比** —— 明确记录差异

### 3.4 额外的 UX 验证

除了验证缺陷已修复外，还需评估整体体验：

1. **正常路径可用** —— 标准用户流程顺畅
2. **边界情况** —— 尝试异常输入、快速点击、页面刷新
3. **视觉质量** —— 无布局问题、颜色正确、文字清晰可读
4. **响应式** —— 调整视口大小，检查不同尺寸：
   ```bash
   agent-browser --session $WORKFLOW_ID set viewport 1920 1080
   agent-browser --session $WORKFLOW_ID screenshot "$ARTIFACTS_DIR/e2e-feature-desktop.png"
   agent-browser --session $WORKFLOW_ID set viewport 768 1024
   agent-browser --session $WORKFLOW_ID screenshot "$ARTIFACTS_DIR/e2e-feature-tablet.png"
   ```
5. **无回归** —— 修复附近的其他功能仍然正常工作

### 3.5 API 交叉验证

```bash
BACKEND_PORT=$(cat $ARTIFACTS_DIR/.backend-port | tr -d '\n')

# 验证数据完整性是否与 UI 一致
curl -s "http://localhost:$BACKEND_PORT/api/conversations" | head -c 500
```

---

## 阶段 4：清理和报告

**关键：必须在撰写分析结果前完成清理。孤立进程会累积并导致系统崩溃。**

### 4.1 关闭浏览器

```bash
# 始终使用 --session，仅关闭你的浏览器，不影响其他工作流
agent-browser --session $WORKFLOW_ID close 2>/dev/null || true
```

### 4.2 停止功能分支的 Archon（跨平台）

通过 PID（阶段 2 中记录）和端口（备用方式）终止进程。适用于 Windows 和 Unix。

```bash
BACKEND_PORT=$(cat $ARTIFACTS_DIR/.backend-port | tr -d '\n')
FRONTEND_PORT=$(cat $ARTIFACTS_DIR/.frontend-port | tr -d '\n')

# 通过记录的 PID 终止（主要方法——包括主分支和功能分支的 PID）
for pidfile in "$ARTIFACTS_DIR/.e2e-feature-backend-pid" "$ARTIFACTS_DIR/.e2e-feature-frontend-pid" "$ARTIFACTS_DIR/.e2e-main-backend-pid" "$ARTIFACTS_DIR/.e2e-main-frontend-pid"; do
  if [ -f "$pidfile" ]; then
    PID=$(cat "$pidfile" | tr -d '\n')
    echo "Killing PID $PID from $pidfile"
    kill "$PID" 2>/dev/null || taskkill //F //T //PID "$PID" 2>/dev/null || true
  fi
done

# 备用：按端口终止（处理 PID 终止可能遗漏的子进程）
for PORT in $BACKEND_PORT $FRONTEND_PORT; do
  echo "Cleaning up port $PORT..."
  fuser -k "$PORT/tcp" 2>/dev/null || true
  lsof -ti:"$PORT" 2>/dev/null | xargs kill -9 2>/dev/null || true
  netstat -ano 2>/dev/null | grep ":$PORT " | grep LISTENING | awk '{print $5}' | sort -u | while read pid; do
    taskkill //F //T //PID "$pid" 2>/dev/null || true
  done
done

sleep 2
echo "Cleanup complete — verify ports are free:"
netstat -ano 2>/dev/null | grep -E ":($BACKEND_PORT|$FRONTEND_PORT) " | grep LISTENING || echo "All ports free"
```

### 4.3 撰写分析结果

写入 `$ARTIFACTS_DIR/e2e-feature.md`：

```markdown
# E2E 测试结果：功能分支

**PR**：#{number}
**分支**：{feature-branch} @ {commit}
**后端端口**：{port}
**前端端口**：{port}
**截图**：$ARTIFACTS_DIR/e2e-feature-*.png

## 测试摘要

| 测试用例 | 主分支结果 | 功能分支结果 | 修复已验证？ |
|-----------|-------------|----------------|---------------|
| {测试 1} | 缺陷已复现 | 已修复 | 是 / 否 |
| {测试 2} | 缺陷已复现 | 已修复 | 是 / 否 |

## 详细结果

### 测试 1：{描述}
**主分支**：{缺陷行为——参考 e2e-main 截图}
**功能分支**：{修复后行为——参考 e2e-feature 截图}
**修复已验证**：是 / 否 / 部分
**截图对比**：`e2e-main-{N}.png` vs `e2e-feature-{N}.png`

### 测试 2：{描述}
{相同结构...}

## UX 质量评估

| 方面 | 评分 (1-5) | 说明 |
|--------|-------------|-------|
| 视觉正确性 | {n} | {详情} |
| 响应式表现 | {n} | {详情} |
| 边界情况处理 | {n} | {详情} |
| 错误状态 | {n} | {详情} |
| 性能感受 | {n} | {详情} |

## 发现的回归问题
{修复引入的任何新问题，或"无"}

## 其他观察
{注意到的其他 UX 改进或问题}

## 修复置信度
**高 / 中 / 低**

{对修复正确性和完整性的整体信心评估}
```

---

## 成功标准

- **ARCHON_STARTED**：后端和前端已在功能分支代码上运行
- **ALL_TESTS_RERUN**：主分支 E2E 的每个测试用例已重新执行
- **FIX_VERIFIED**：每个缺陷已确认修复（或记录为仍然存在）
- **UX_VALIDATED**：视觉质量、响应式、边界情况已检查
- **NO_REGRESSIONS**：未引入新问题
- **ARCHON_STOPPED**：进程已终止，端口已释放——**完成前验证端口已释放**
- **ARTIFACT_WRITTEN**：`$ARTIFACTS_DIR/e2e-feature.md` 已创建

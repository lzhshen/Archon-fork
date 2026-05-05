---
description: 收集 PR 上下文、验证可审查性，并为全面审查准备产物目录
argument-hint: <pr-number|url>
---

# PR 审查范围

**输入**: $ARGUMENTS

---

## 任务目标

验证 PR 是否处于可审查状态，收集并行审查代理所需的全部上下文，并准备产物目录结构。

---

## 阶段 1: 识别 - 确定 PR

### 1.1 获取 PR 编号

```bash
if [ -f "$ARTIFACTS_DIR/.pr-number" ]; then
  PR_NUMBER=$(cat $ARTIFACTS_DIR/.pr-number | tr -d '\n' | tr -d ' ')
  if ! echo "$PR_NUMBER" | grep -qE '^[0-9]+$'; then
    PR_NUMBER=""
  fi
fi

# 从参数获取（独立审查）
if [ -z "$PR_NUMBER" ] && [ -n "$ARGUMENTS" ]; then
  PR_NUMBER=$(echo "$ARGUMENTS" | grep -oE '[0-9]+' | head -1)
fi

# 从当前分支获取
if [ -z "$PR_NUMBER" ]; then
  PR_NUMBER=$(gh pr view --json number -q '.number' 2>/dev/null)
fi

if [ -z "$PR_NUMBER" ]; then
  echo "错误: 未找到 PR 编号"
  exit 1
fi

# 写入注册表供下游步骤使用（若尚不存在）
echo "$PR_NUMBER" > $ARTIFACTS_DIR/.pr-number
```

### 1.2 获取 PR 详情

```bash
gh pr view {number} --json number,title,body,url,headRefName,baseRefName,files,additions,deletions,changedFiles,state,author,isDraft,mergeable,mergeStateStatus
```

**提取：**
- PR 编号和标题
- 分支名称（head -> base）
- 变更文件列表
- 新增/删除行数
- 草稿状态
- 可合并状态

**阶段 1 检查点:**
- [ ] PR 编号已确认
- [ ] PR 处于开放状态（未合并/未关闭）
- [ ] 基本元数据已提取

---

## 阶段 2: 验证 - 审查前检查

**在启动审查代理之前，验证 PR 是否处于可审查状态。**

### 2.1 检查合并冲突

```bash
gh pr view {number} --json mergeable,mergeStateStatus --jq '.mergeable, .mergeStateStatus'
```

| 状态 | 操作 |
|------|------|
| `MERGEABLE` | 继续 |
| `CONFLICTING` | **停止** - 告知用户先解决冲突 |
| `UNKNOWN` | 警告，继续（GitHub 仍在计算中） |

**若存在冲突：**
```markdown
错误: **无法审查: PR 存在合并冲突**

请先解决冲突后再请求审查：
```bash
git fetch origin {base}
git rebase origin/{base}
# 解决冲突
git push --force-with-lease
```

然后重新请求审查。
```
**若检测到冲突则退出工作流。**

### 2.2 检查 CI 状态

```bash
gh pr checks {number} --json name,state,conclusion --jq '.[] | "\(.name): \(.state) (\(.conclusion // "pending"))"'
```

| 状态 | 操作 |
|------|------|
| 全部通过 | 继续 |
| 部分失败 | 警告，继续（在范围中注明） |
| 全部失败 | 强烈警告，继续（在范围中注明） |
| 进行中 | 注明，继续 |

**将 CI 状态标记到审查报告中。**

### 2.3 检查是否落后于基础分支

```bash
# 获取分支名称
PR_BASE=$(gh pr view {number} --json baseRefName --jq '.baseRefName')
PR_HEAD=$(gh pr view {number} --json headRefName --jq '.headRefName')

# 拉取并计数
git fetch origin $PR_BASE --quiet
git fetch origin $PR_HEAD --quiet

# 落后于基础分支的提交数
BEHIND=$(git rev-list --count origin/$PR_HEAD..origin/$PR_BASE 2>/dev/null || echo "0")
```

| 落后提交数 | 操作 |
|------------|------|
| 0-5 | 继续 |
| 6-15 | 警告，建议变基，继续 |
| 16+ | 强烈警告，建议在审查前变基 |

**若落后较多：**
```markdown
警告: **分支落后于 {base} {N} 个提交**

建议在审查前进行变基，以确保审查基于最新代码：
```bash
git fetch origin {base}
git rebase origin/{base}
git push --force-with-lease
```
```

### 2.4 检查草稿状态

```bash
gh pr view {number} --json isDraft --jq '.isDraft'
```

| 状态 | 操作 |
|------|------|
| `false` | 正常继续 |
| `true` | 在范围中注明，继续（用户需要早期反馈） |

### 2.5 检查 PR 大小

| 指标 | 警告阈值 | 操作 |
|------|----------|------|
| 变更文件数 | 20+ | 警告审查彻底性可能受影响 |
| 变更行数 | 1000+ | 警告审查彻底性可能受影响 |

**若 PR 非常大：**
```markdown
警告: **大型 PR: {files} 个文件, +{additions} -{deletions} 行**

大型 PR 难以彻底审查。建议拆分为较小的 PR 以提高审查质量。
```

### 2.6 汇编可审查性总结

```markdown
## 审查前状态

| 检查项 | 状态 | 备注 |
|--------|------|------|
| 合并冲突 | 无 / 存在冲突 | {详情} |
| CI 状态 | 通过 / 失败 / 进行中 | {详情} |
| 落后基础分支 | 已更新 / 落后 {N} 个提交 | {详情} |
| 草稿状态 | 就绪 / 草稿 | {详情} |
| 大小 | 正常 / 较大 ({N} 个文件) | {详情} |
```

**阶段 2 检查点:**
- [ ] 无合并冲突（或工作流已停止）
- [ ] CI 状态已记录
- [ ] 落后基础分支状态已检查
- [ ] 草稿状态已记录
- [ ] 大小警告（若需要）已发出

---

## 阶段 3: 上下文 - 收集审查上下文

### 3.1 获取完整差异

```bash
gh pr diff {number}
```

保存以供参考 -- 并行代理将根据需要重新获取。

### 3.2 按类型列出变更文件

```bash
gh pr view {number} --json files --jq '.files[].path'
```

**将文件分类：**
- 源代码 (`.ts`, `.js`, `.py` 等)
- 测试文件 (`*.test.ts`, `*.spec.ts`, `test_*.py`)
- 文档 (`*.md`, `docs/`)
- 配置文件 (`.json`, `.yaml`, `.toml`)
- 类型/接口

### 3.3 检查 CLAUDE.md

```bash
cat CLAUDE.md 2>/dev/null | head -100
```

记录审查者应对照的关键规则。

### 3.4 识别新抽象

扫描差异，查找此 PR 引入的新抽象：

- 新接口、类型或抽象类（在差异中搜索 `interface `、`type `、`abstract class`）
- 新工具模块或辅助文件（非功能文件或测试的新 `.ts` 文件）
- 新配置键或 Schema 字段

对于发现的每个新抽象，在范围清单的"审查关注领域"中注明，以便代码审查代理验证其不会重复已有原语。

```bash
# 快速扫描差异中的新抽象
gh pr diff {number} | grep "^+" | sed 's/^+//' | grep -E "(^interface |^export interface |^type |^abstract class |^export class )" | head -20
```

**阶段 3 检查点:**
- [ ] 差异已获取
- [ ] 文件已按类型分类
- [ ] CLAUDE.md 规则已记录
- [ ] 新抽象已扫描

---

## 阶段 3.5: 计划/Issue 上下文 - 检查工作流产物

**关键**: 若此 PR 由工作流创建，则会有包含审查者重要上下文的产物。

### 3.5.1 查找工作流产物

检查两种工作流类型的产物：

```bash
# 选项 1: 基于计划的工作流 (archon-plan-to-merge)
ls -t $ARTIFACTS_DIR/../runs/*/plan-context.md 2>/dev/null | head -1

# 选项 2: 基于 Issue 的工作流 (archon-fix-github-issue)
ls -t $ARTIFACTS_DIR/../runs/*/investigation.md 2>/dev/null | head -1
```

### 3.5.2 提取范围限制

**若 plan-context.md 存在**（来自计划工作流）：

```bash
# 提取"不构建"部分
sed -n '/## NOT Building/,/^## /p' $ARTIFACTS_DIR/../runs/*/plan-context.md | head -30
```

**若 investigation.md 存在**（来自 Issue 工作流）：

```bash
# 提取"范围边界 / 范围外"部分
sed -n '/## Scope Boundaries/,/^## /p' $ARTIFACTS_DIR/../runs/*/investigation.md | head -30
```

**这些是有意排除项** -- 请勿将其标记为缺陷或缺失功能！

### 3.5.3 检查实现报告

```bash
# 查找实现报告（任一工作流）
ls -t $ARTIFACTS_DIR/../runs/*/implementation.md 2>/dev/null | head -1
```

**若 implementation.md 存在**，记录任何偏差：

```bash
# 提取偏差部分
sed -n '/## Deviations/,/^## /p' $ARTIFACTS_DIR/../runs/*/implementation.md | head -20
```

**阶段 3.5 检查点:**
- [ ] 工作流产物已检查（plan-context.md 或 investigation.md）
- [ ] 范围限制已提取（"不构建"或"范围外"）
- [ ] 实现偏差已记录（若有）

---

## 阶段 4: 准备 - 创建产物目录

### 4.1 创建目录结构

```bash
mkdir -p $ARTIFACTS_DIR/review
```

### 4.2 清理过期产物

```bash
# 删除超过 7 天的审查目录
find $ARTIFACTS_DIR/../reviews/pr-* -maxdepth 0 -mtime +7 -exec rm -rf {} \; 2>/dev/null || true
```

### 4.3 创建范围清单

写入 `$ARTIFACTS_DIR/review/scope.md`：

```markdown
# PR 审查范围: #{number}

**标题**: {PR 标题}
**URL**: {PR URL}
**分支**: {head} -> {base}
**作者**: {作者}
**日期**: {ISO 时间戳}

---

## 审查前状态

| 检查项 | 状态 | 备注 |
|--------|------|------|
| 合并冲突 | {状态} | {详情} |
| CI 状态 | {状态} | {通过}/{总数} 项检查 |
| 落后基础分支 | {状态} | 落后 {N} 个提交 |
| 草稿 | {状态} | {就绪/草稿} |
| 大小 | {状态} | {files} 个文件, +{add}/-{del} |

---

## 变更文件

| 文件 | 类型 | 新增行 | 删除行 |
|------|------|--------|--------|
| `src/file.ts` | 源码 | +10 | -5 |
| `src/file.test.ts` | 测试 | +20 | -0 |
| ... | ... | ... | ... |

**总计**: {changedFiles} 个文件, +{additions} -{deletions}

---

## 文件分类

### 源文件 ({count})
- `src/...`

### 测试文件 ({count})
- `src/...test.ts`

### 文档 ({count})
- `$DOCS_DIR/...`
- `README.md`

### 配置文件 ({count})
- `package.json`

---

## 审查关注领域

根据变更，审查者应关注：

1. **代码质量**: {列出关键源文件}
2. **错误处理**: {包含 try/catch 和错误处理的文件}
3. **测试覆盖**: {需要测试的新功能}
4. **注释/文档**: {有文档变更的文件}
5. **文档影响**: {检查 CLAUDE.md 或 $DOCS_DIR 是否需要更新}
6. **原语对齐**: {若发现新抽象: 列出} -- 验证不重复已有原语

---

## 需检查的 CLAUDE.md 规则

{从 CLAUDE.md 中提取适用于此 PR 的关键规则}

---

## 工作流上下文（若来自自动化工作流）

{若找到了 plan-context.md 或 investigation.md:}

### 范围限制（不构建 / 范围外）

**审查人员注意**: 这些项目已**有意排除**在范围之外。请勿将其标记为缺陷或缺失功能。

{来自 plan-context.md 的"不构建"部分或 investigation.md 的"范围边界/范围外"部分}

**在范围内：**
- {我们要变更的内容}

**范围外（不要触碰）：**
- {明确排除项 1 及理由}
- {明确排除项 2 及理由}

### 实现偏差

{若找到了 implementation.md 且包含偏差:}

{复制 implementation.md 中的"偏差"部分}

{若未找到工作流产物:}

_未找到工作流产物 -- 这似乎是一个手动创建的 PR。_

---

## CI 详情

{若 CI 失败，列出哪些检查失败}

---

## 元数据

- **范围创建时间**: {ISO 时间戳}
- **产物路径**: `$ARTIFACTS_DIR/review/`
```

**阶段 4 检查点:**
- [ ] 目录已创建
- [ ] 过期产物已清理
- [ ] 范围清单已写入，包含审查前状态

---

## 阶段 5: 输出 - 向用户报告

### 若被阻塞（存在冲突）

```markdown
## 审查被阻塞: 存在合并冲突

**PR**: #{number} - {title}

此 PR 存在合并冲突，必须在审查前解决。

### 解决方法

```bash
git fetch origin {base}
git checkout {head}
git rebase origin/{base}
# 在编辑器中解决冲突
git add .
git rebase --continue
git push --force-with-lease
```

然后重新请求审查: `@archon review this PR`
```

### 若继续进行

```markdown
## PR 审查范围准备完成

**PR**: #{number} - {title}
**文件**: {count} 个变更 (+{additions} -{deletions})

### 审查前状态
| 检查项 | 状态 |
|--------|------|
| 冲突 | 无 |
| CI | {通过 / {N} 项失败} |
| 落后基础分支 | {已更新 / 落后 {N}} |
| 草稿 | {就绪 / 草稿} |
| 大小 | {正常 / 较大} |

### 文件分类
- 源码: {count} 个文件
- 测试: {count} 个文件
- 文档: {count} 个文件
- 配置: {count} 个文件

### 产物目录
`$ARTIFACTS_DIR/review/`

### 下一步
正在启动 5 个并行审查代理...
```

---

## 成功标准

- **PR_IDENTIFIED**: 找到有效的开放 PR
- **NO_CONFLICTS**: 合并冲突会阻塞工作流
- **CONTEXT_GATHERED**: 差异和文件列表已获取
- **ARTIFACTS_DIR_CREATED**: 目录结构已存在
- **SCOPE_MANIFEST_WRITTEN**: `scope.md` 文件已创建，包含审查前状态

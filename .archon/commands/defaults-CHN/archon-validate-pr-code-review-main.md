---
description: 分析主/基础分支上的代码，确认 PR 修改前缺陷或漏洞确实存在
argument-hint: (none - reads from artifacts)
---

# 代码审查：主分支（PR 前状态）

在**主分支**上分析代码库，确认 PR 中描述的缺陷、漏洞或缺失功能确实存在。

---

## 阶段 1：加载上下文

### 1.1 读取 PR 详情

```bash
cat $ARTIFACTS_DIR/.pr-number
```

```bash
PR_NUMBER=$(cat $ARTIFACTS_DIR/.pr-number | tr -d '\n')
gh pr view "$PR_NUMBER" --json title,body,headRefName,baseRefName,labels
```

### 1.2 读取路径信息

```bash
cat $ARTIFACTS_DIR/.canonical-repo
cat $ARTIFACTS_DIR/.worktree-path
cat $ARTIFACTS_DIR/.pr-base
```

### 1.3 理解 PR 声称修复的内容

从 PR 标题、正文和关联的 Issue 中：
- PR 声称存在什么缺陷或漏洞？
- 预期行为与实际行为分别是什么？
- 涉及哪些文件/组件？

如果 PR 正文引用了 GitHub Issue，请获取它：

```bash
# 从 PR 正文中提取 Issue 编号（查找 "Fixes #N"、"Closes #N" 等）
PR_NUMBER=$(cat $ARTIFACTS_DIR/.pr-number | tr -d '\n')
ISSUE_NUMBER=$(gh pr view "$PR_NUMBER" --json body -q '.body' | grep -oE '(Fixes|Closes|Resolves)\s*#[0-9]+' | grep -oE '[0-9]+' | head -1)
if [ -n "$ISSUE_NUMBER" ]; then
  gh issue view "$ISSUE_NUMBER" --json title,body,labels,comments
fi
```

---

## 阶段 2：分析主分支代码

### 2.1 读取 PR 变更的文件

从 PR 差异中获取变更文件列表，然后在**主分支**上读取这些**相同文件**（规范仓库路径）。

```bash
PR_NUMBER=$(cat $ARTIFACTS_DIR/.pr-number | tr -d '\n')
gh pr view "$PR_NUMBER" --json files -q '.files[].path'
```

**关键**：从**规范仓库**（主分支）读取文件，而非当前工作树（功能分支）。规范仓库路径存储在 `$ARTIFACTS_DIR/.canonical-repo` 中。

对每个变更文件，从主分支读取：

```bash
CANONICAL_REPO=$(cat $ARTIFACTS_DIR/.canonical-repo | tr -d '\n')
# 从规范仓库（主分支）读取每个文件
cat "$CANONICAL_REPO/<file-path>"
```

### 2.2 追踪缺陷或漏洞

对 PR 中的每项声明：
1. **定位相关代码** —— 在主分支上读取具体的函数、组件、钩子
2. **追踪数据流** —— 数据从何处来？如何变换？
3. **确定根因** —— 能否在代码中看到缺陷？
4. **检查相关代码** —— 是否存在 PR 可能遗漏的邻近问题？

### 2.3 评估严重程度

- 此缺陷/漏洞对主分支的影响有多大？
- 是用户可见的还是内部的？
- 是否影响核心功能还是仅影响边界情况？
- 用户遇到此问题的可能性有多大？

---

## 阶段 3：撰写分析结果

将分析结果写入 `$ARTIFACTS_DIR/code-review-main.md`：

```markdown
# 主分支代码审查：PR #{number}

**PR 标题**：{title}
**基础分支**：{base}
**分析提交**：{主分支 HEAD}

## 缺陷/漏洞评估

### 声称的问题
{PR 声称修复的内容}

### 主分支上已确认？
**是 / 否 / 部分**

### 证据

{对每项声明，提供具体代码证据：}

#### 声明 1：{描述}
**状态**：已确认 / 未发现 / 部分确认

**代码位置**：`{file}:{lines}`
```{language}
{主分支上显示缺陷/漏洞的实际代码}
```

**分析**：{此代码为何存在缺陷/不完整}

#### 声明 2：{描述}
{相同结构...}

### 发现的相关问题
{在同一代码区域发现的任何其他问题}

### 严重程度评估
| 因素 | 评级 |
|--------|--------|
| 用户影响 | 高 / 中 / 低 |
| 出现频率 | 常见 / 不常见 / 罕见 |
| 核心功能 | 是 / 否 |
| 数据丢失风险 | 是 / 否 |

## 总结
{2-3 句总结：缺陷是否真实存在？严重程度如何？PR 的范围是否恰当？}
```

---

## 成功标准

- **PR_CONTEXT_LOADED**：PR 详情和关联 Issue 已读取
- **MAIN_CODE_ANALYZED**：已从主分支读取变更文件
- **BUG_ASSESSED**：每项 PR 声明已对照主分支代码验证
- **ARTIFACT_WRITTEN**：`$ARTIFACTS_DIR/code-review-main.md` 已创建

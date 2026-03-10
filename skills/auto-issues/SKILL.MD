# skills/auto-issues/SKILL.MD

目标：根据本文件，使用本地 `gh` CLI 将 `issues/` 目录下的兼容性分析报告提交到 torch_npu 验证镜像仓库。

## 1. 前置条件

- 已安装 GitHub CLI (`gh`)
- 已登录 GitHub 账号：`gh auth login`
- 有目标仓库的 issue 写权限
- 目标仓库：`https://github.com/aflyingto/torch_npu`

## 2. 执行命令

在项目根目录执行：

```bash
# 遍历 issues 目录并提交
for issue_file in issues/*.md; do
  [ -f "$issue_file" ] || continue
  
  # 提取编号和标题
  FIRST_LINE=$(head -n 1 "$issue_file")
  ISSUE_ID=$(echo "$FIRST_LINE" | sed 's/^# \[\(.*\)\] .*/\1/')
  TITLE=$(echo "$FIRST_LINE" | sed 's/^# \[.*\] //')
  
  # 提取内容（去除第一行）
  BODY=$(tail -n +2 "$issue_file")
  
  # 检查是否已存在
  EXISTING=$(gh issue list -R aflyingto/torch_npu \
    --search "in:title [${ISSUE_ID}]" \
    --state all \
    --json number \
    -q '.[0].number' 2>/dev/null || echo "")
  
  if [ -n "$EXISTING" ]; then
    echo "⏭️  跳过 [${ISSUE_ID}] ${TITLE} - 已存在: https://github.com/aflyingto/torch_npu/issues/${EXISTING}"
    continue
  fi
  
  # 创建 issue
  echo "📝 创建 [${ISSUE_ID}] ${TITLE}..."
  ISSUE_URL=$(gh issue create -R aflyingto/torch_npu \
    --title "[${ISSUE_ID}] ${TITLE}" \
    --body-file - \
    --label "bug,pytorch-nightly" <<< "$BODY")
  
  echo "✅ 已创建: ${ISSUE_URL}"
done
```

## 3. 单文件提交命令

提交单个 issue 文件：

```bash
# 设置文件路径
ISSUE_FILE="issues/2026-03-07-001-CachingHostAllocator-HostBlockPool-api-break.md"

# 提取信息
FIRST_LINE=$(head -n 1 "$ISSUE_FILE")
ISSUE_ID=$(echo "$FIRST_LINE" | sed 's/^# \[\(.*\)\] .*/\1/')
TITLE=$(echo "$FIRST_LINE" | sed 's/^# \[.*\] //')
BODY=$(tail -n +2 "$ISSUE_FILE")

# 创建 issue
gh issue create -R aflyingto/torch_npu \
  --title "[${ISSUE_ID}] ${TITLE}" \
  --body-file - \
  --label "bug,pytorch-nightly" <<< "$BODY"
```

## 4. Issue 内容提取规则

### 标题格式

文件第一行格式：`# [YYYY-MM-DD-NNN] 标题内容`

```bash
# 提取编号
ISSUE_ID=$(head -n 1 "$file" | sed 's/^# \[\(.*\)\] .*/\1/')
# 结果：2026-03-07-001

# 提取标题
TITLE=$(head -n 1 "$file" | sed 's/^# \[.*\] //')
# 结果：CachingHostAllocator 编译失败：PyTorch nightly 重构 HostBlockPool API
```

### 内容提取

去除第一行后的全部内容作为 issue body：

```bash
BODY=$(tail -n +2 "$file")
```

### Label 设置

根据内容自动判断 labels：

| 关键词 | Label |
|--------|-------|
| 编译失败 | `bug` |
| API | `api-change` |
| PyTorch nightly | `pytorch-nightly` |

默认 labels：`bug,pytorch-nightly`

## 5. 检查已存在 Issue

搜索目标仓库是否已存在相同编号的 issue：

```bash
gh issue list -R aflyingto/torch_npu \
  --search "in:title [2026-03-07-001]" \
  --state all
```

参数说明：
- `-R aflyingto/torch_npu` - 目标仓库
- `--search "in:title ..."` - 搜索标题
- `--state all` - 包含已关闭的 issue

## 6. 完整脚本

创建脚本 `scripts/submit-issues.sh`：

```bash
#!/bin/bash
set -e

ISSUES_DIR="issues"
TARGET_REPO="aflyingto/torch_npu"
CREATED=0
SKIPPED=0

echo "开始提交 issues 到 ${TARGET_REPO}..."
echo ""

for issue_file in "${ISSUES_DIR}"/*.md; do
  [ -f "$issue_file" ] || continue
  
  # 提取标题和编号
  FIRST_LINE=$(head -n 1 "$issue_file")
  ISSUE_ID=$(echo "$FIRST_LINE" | sed 's/^# \[\(.*\)\] .*/\1/')
  TITLE=$(echo "$FIRST_LINE" | sed 's/^# \[.*\] //')
  
  # 提取内容
  BODY=$(tail -n +2 "$issue_file")
  
  # 设置 labels
  LABELS="bug,pytorch-nightly"
  
  # 检查是否已存在
  EXISTING=$(gh issue list -R "$TARGET_REPO" \
    --search "in:title [${ISSUE_ID}]" \
    --state all \
    --json number \
    -q '.[0].number' 2>/dev/null || echo "")
  
  if [ -n "$EXISTING" ]; then
    echo "⏭️  跳过 [${ISSUE_ID}] ${TITLE}"
    echo "   已存在: https://github.com/${TARGET_REPO}/issues/${EXISTING}"
    SKIPPED=$((SKIPPED + 1))
    continue
  fi
  
  # 创建 issue
  ISSUE_URL=$(gh issue create -R "$TARGET_REPO" \
    --title "[${ISSUE_ID}] ${TITLE}" \
    --body-file - \
    --label "$LABELS" <<< "$BODY")
  
  echo "✅ 已创建 [${ISSUE_ID}] ${TITLE}"
  echo "   URL: ${ISSUE_URL}"
  CREATED=$((CREATED + 1))
done

echo ""
echo "完成！创建 ${CREATED} 个，跳过 ${SKIPPED} 个"
```

执行：

```bash
chmod +x scripts/submit-issues.sh
./scripts/submit-issues.sh
```

## 7. 常见问题

### Q: 如何检查 gh 是否已登录？

```bash
gh auth status
```

### Q: 如何登录 GitHub？

```bash
gh auth login
# 选择 GitHub.com
# 选择 HTTPS
# 选择 Login with a web browser
```

### Q: 如何查看目标仓库的权限？

```bash
gh repo view aflyingto/torch_npu --json viewerPermission
```

### Q: Issue 创建失败怎么办？

检查：
1. 是否已登录：`gh auth status`
2. 是否有写权限
3. 标题是否过长（最大 256 字符）
4. Body 是否过大（最大 64KB）

## 8. 推荐执行提示词

"请按 `SKILL.MD` 的说明，使用 `gh` CLI 将 `issues/` 目录下的兼容性分析报告提交到 `aflyingto/torch_npu` 仓库。先检查是否已存在相同编号的 issue，若不存在则创建，设置 labels 为 `bug,pytorch-nightly`。"

---

## 相关文件

- `issues/` - 兼容性分析报告目录
- `scripts/submit-issues.sh` - 提交脚本（可选）

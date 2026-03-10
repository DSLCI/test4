# AGENTS.md

This file provides guidance to AI agents when working with code in this repository.

## Project Overview

This is an **infrastructure repository** for tracking and fixing PyTorch nightly compatibility issues in Ascend/pytorch (torch_npu). It is NOT a source code repository - the actual Ascend/pytorch code is cloned during CI from `Ascend/pytorch`.

**Purpose:**
- Track API compatibility issues between PyTorch nightly and Ascend/pytorch
- Store patches for fixing compatibility issues
- Provide skill documents for generating CI workflows
- Document common API break patterns and fixes

**Architecture:**
```
issues/     - Compatibility analysis reports (markdown)
patches/    - Git patches to fix issues (numbered .patch files)
skills/     - Skill documents for CI workflow generation
```

**Target Repositories:**
- **This repo**: `aflyingto/torch_npu-adapt-infra-dev` - Infrastructure/tracking
- **Issues target**: `aflyingto/torch_npu` - Where issues are submitted
- **Source code**: `Ascend/pytorch` - Cloned during CI builds

## Build & Commands

### CI Workflows

This repository generates GitHub Actions workflows via skill documents:

| Skill | Output | Trigger |
|-------|--------|---------|
| `torch_npu-adapt-pytorch-nightly` | `.github/workflows/nightly-build.yml` | Daily schedule, manual, push to patches/ |
| `torch_npu-build` | `.github/workflows/torch_npu-build.yml` | Manual dispatch |
| `auto-issues` | N/A | Manual execution via `gh` CLI |

### Patch Application

Patches are applied to cloned `Ascend/pytorch` source:

```bash
# Correct way - from repo root
git apply --directory=ascend_pytorch patches/0001-xxx.patch

# Wrong - do not cd into the directory
cd ascend_pytorch && git apply ../patches/xxx.patch  # DON'T DO THIS
```

### Issue Submission

Submit issues to target repository using `gh` CLI:

```bash
# Check authentication
gh auth status

# Submit single issue
ISSUE_FILE="issues/2026-03-07-001-CachingHostAllocator-HostBlockPool-api-break.md"
FIRST_LINE=$(head -n 1 "$ISSUE_FILE")
ISSUE_ID=$(echo "$FIRST_LINE" | sed 's/^# \[\(.*\)\] .*/\1/')
TITLE=$(echo "$FIRST_LINE" | sed 's/^# \[.*\] //')
BODY=$(tail -n +2 "$ISSUE_FILE")

gh issue create -R aflyingto/torch_npu \
  --title "[${ISSUE_ID}] ${TITLE}" \
  --body-file - \
  --label "bug,pytorch-nightly" <<< "$BODY"
```

## Code Style

### Issue and Patch Naming Convention

Files use date-based numbering: `YYYY-MM-DD-NNN`

- `issues/YYYY-MM-DD-NNN-description.md` - Problem analysis
- `patches/NNNN-description.patch` - Corresponding fix (padded to 4 digits)

### Issue Document Format

```markdown
# [YYYY-MM-DD-NNN] Title

- **发现日期**: YYYY-MM-DD
- **编号**: YYYY-MM-DD-NNN
- **严重级别**: 编译失败/运行时错误
- **受影响文件**: file paths
- **触发版本**: PyTorch nightly version
- **对应 patch**: patches/NNNN-xxx.patch

## 问题描述
## 根本原因分析
## 修复方案
```

### Patch Format

- Standard unified diff format
- Target path: `torch_npu/csrc/...` (relative to Ascend/pytorch root)
- Include clear comments explaining the fix

## Testing

### CI Workflow Validation

Workflows are generated from skill documents. To validate:

1. **Nightly Build Workflow**: Tests PyTorch nightly compatibility
   - Clones `Ascend/pytorch` master branch
   - Applies all patches from `patches/`
   - Builds wheel package
   - Reports compatibility issues from `issues/`

2. **Stable Build Workflow**: Builds with stable PyTorch
   - Clones specific tag (e.g., `v7.2.0-pytorch2.1.0`)
   - No patches applied
   - Builds CANN stub libraries

### Local Testing

```bash
# Test patch application locally
git clone --depth=1 --recurse-submodules https://github.com/Ascend/pytorch ascend_pytorch
git apply --directory=ascend_pytorch patches/0001-xxx.patch

# Verify patch applied correctly
cd ascend_pytorch && git status
```

## Security

### Credentials Management

- **GitHub CLI**: Requires `gh auth login` before issue submission
- **No secrets in patches**: Patches should never contain credentials
- **Workflow permissions**: Uses `$GITHUB_ENV` for environment variables, not `export`

### Data Protection

- Issue reports may contain file paths and error messages - no sensitive data
- Patches are public - ensure no proprietary code is included
- Build logs are stored as artifacts (retained per GitHub policy)

## Configuration

### PyTorch Nightly Version Format

**Critical**: PyTorch nightly CPU version requires specific format:

```
torch==${MAJOR}.dev${DATE}+cpu
```

- `${MAJOR}`: Must try `2.12.0` → `2.11.0` → `2.10.0` in order
- `${DATE}`: Format `YYYYMMDD` (e.g., `20260309`)
- `+cpu`: **Required suffix**

Example: `torch==2.11.0.dev20260309+cpu`

### ccache Configuration

```bash
ccache --max-size=2G
ccache --zero-stats
echo "CC=ccache gcc" >> $GITHUB_ENV
echo "CXX=ccache g++" >> $GITHUB_ENV
echo "CCACHE_DIR=$HOME/.ccache" >> $GITHUB_ENV
```

### Build Environment Variables

| Variable | Nightly Build | Stable Build |
|----------|--------------|--------------|
| `DISABLE_INSTALL_TORCHAIR` | `FALSE` | `TRUE` |
| `DISABLE_RPC_FRAMEWORK` | (not set) | `TRUE` |
| `BUILD_WITHOUT_SHA` | `1` | `1` |

## Common API Break Patterns

When PyTorch upstream changes APIs, use these fix patterns:

| Break Type | Fix Pattern | Example |
|------------|-------------|---------|
| Struct member removed | Add module-level `static map + mutex` cache | `rank_to_global_rank_cache` |
| Member renamed | Rename conflicting local type to avoid collision | `BlockPool` → `ExpandableBlockPool` |
| Base class adds pure virtual | Implement override (return 0 if no semantic meaning) | `get_offset() override { return 0; }` |
| Virtual signature changed | Update override, extract old logic to private helper | `process_events(BlockPool&)` + `do_process_npu_events()` |

## Key Workflow Patterns

### GitHub Actions Best Practices

1. **Artifact uploads**: Use `if: always()` for logs, `if: success()` for wheels
2. **Environment variables**: Use `$GITHUB_ENV` not `export`
3. **Build status**: Use `${PIPESTATUS[0]}` when piping to `tee`
4. **Patch application**: Always use `--directory` flag

### ccache Cache Key Strategy

Priority order for `restore-keys`:
1. Same OS + Python + PyTorch version + branch
2. Same OS + Python + PyTorch version
3. Same OS + Python + branch
4. Same OS + Python
5. Same OS (fallback)

## Skill Document Usage

Each skill in `skills/*/SKILL.MD` contains complete instructions for generating CI workflows:

1. Read the skill document completely
2. Generate the workflow YAML exactly as specified
3. Follow the "Key防坑" (pitfall avoidance) section strictly
4. Validate against the "最小验收标准" (minimum acceptance criteria)

**Recommended prompt for other agents:**

> "请严格按 `skills/torch_npu-adapt-pytorch-nightly/SKILL.MD` 生成 `.github/workflows/nightly-build.yml`。要求：按文档顺序组织 step，使用单一 `actions/cache@v4` 设置 ccache，使用 `$GITHUB_ENV` 设置环境变量，克隆 Ascend/pytorch master 分支，应用 `patches/` 目录下的兼容性 patch，PyTorch nightly 版本格式必须为 `${MAJOR}.dev${DATE}+cpu`，主版本号需依次尝试 `2.12.0` → `2.11.0` → `2.10.0`，ccache 统计作为独立 step 输出到 step summary（代码块格式），使用表格格式输出构建信息，必须在 Summary 中展示 `issues/` 目录下的兼容性问题分析报告。"

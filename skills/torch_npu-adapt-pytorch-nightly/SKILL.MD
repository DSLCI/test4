# skills/torch_npu-adapt-pytorch-nightly/SKILL.MD

目标：仅根据本文件，自动生成可运行的 `.github/workflows/nightly-build.yml`，用于验证 `Ascend/pytorch` 在 PyTorch nightly 版本下的编译兼容性。

## 1. 输出文件与触发

- 输出文件路径：`.github/workflows/nightly-build.yml`
- workflow 名称：`Ascend/pytorch Nightly Build Validation`
- 触发方式：
  - 定时：`schedule` - 每天 UTC 02:00（北京时间 10:00）
  - 手动：`workflow_dispatch` - 可选参数 `torch_nightly_date`（格式：YYYYMMDD）
  - 推送：`push` - 路径包括 `.github/workflows/nightly-build.yml`、`patches/**`、`AGENTS.md`
- 全局环境变量：`PYTHON_VERSION: '3.11'`

## 2. Job 基础信息

- 单一 job：`build`
- job 名称：`Build torch_npu (x86, PyTorch nightly)`
- runner：`ubuntu-22.04`

## 3. 必须包含的步骤（顺序固定）

1. `actions/checkout@v4`
2. `actions/setup-python@v5`，版本使用 `${{ env.PYTHON_VERSION }}`
3. 安装系统依赖：`cmake ninja-build gcc g++ git patchelf ccache`
4. 安装 PyTorch nightly：
   - 升级 pip
   - **版本格式（重要）**：`torch==${MAJOR}.dev${DATE}+cpu`
     - `${MAJOR}`：主版本号，**不能固定**，需依次尝试：`2.12.0` → `2.11.0` → `2.10.0`
     - `${DATE}`：日期，格式 `YYYYMMDD`，如 `20260309`
     - `+cpu`：**必须包含**，表示 CPU 版本
     - 完整示例：`torch==2.11.0.dev20260309+cpu`
   - **安装逻辑**：
     - 若指定日期：依次尝试 `torch==2.12.0.dev${DATE}+cpu` → `torch==2.11.0.dev${DATE}+cpu` → `torch==2.10.0.dev${DATE}+cpu`，直到成功
     - 否则：安装最新 `torch --pre`（从 nightly/cpu 索引）
   - 输出版本到 `$GITHUB_OUTPUT`
5. 设置 ccache：`actions/cache@v4`（key 包含 torch 版本号）
6. 配置 ccache：设置 max-size、zero-stats、环境变量
7. 克隆 `Ascend/pytorch`：
   - 参数：`--depth=1 --recurse-submodules`
   - 目标目录：`ascend_pytorch`
   - 记录 commit 信息（完整 hash、短 hash、日期）
8. 安装 Python 构建依赖：`pyyaml setuptools auditwheel`
9. 应用兼容性 patches：
   - 遍历 `patches/` 目录下所有 `.patch` 文件
   - 使用 `git apply --directory=ascend_pytorch` 应用
   - 记录成功/失败列表到 `$GITHUB_OUTPUT`
10. 构建 wheel：`python setup.py build bdist_wheel`
11. 显示 ccache 统计（`if: always()`）
12. 上传构建日志 artifact（`if: always()`）
13. 上传 wheel artifact（仅成功时）
14. 输出构建 summary：
    - 构建信息表格
    - **兼容性问题分析报告**（从 `issues/` 目录提取）
    - 构建结果

## 4. ccache 规则

### 步骤顺序要求

**重要**：ccache setup 步骤必须在安装 PyTorch nightly 之后执行，因为 cache key 需要包含 torch 版本号。

正确顺序：
1. 安装系统依赖
2. 安装 PyTorch nightly（输出版本到 `$GITHUB_OUTPUT`）
3. Setup ccache（使用 torch 版本号构建 key）

### 缓存 Action 选择

**方案一（推荐）：单一 `actions/cache@v4`**
- 适用于：缓存 key 固定、不需要 always 保存的场景
- 优点：配置简单，自动处理保存

**方案二：分离 `cache/restore` + `cache/save`**
- 适用于：需要 `if: always()` 保存缓存的场景
- 优点：即使构建失败也能保存缓存

### 缓存 Key 策略

- 缓存路径：`~/.ccache`
- Key 格式：`ccache-{os}-py{python_version}-torch{torch_version}-{ref_name}-{run_id}`
- `restore-keys`（按优先级排序，从高到低）：
  1. `ccache-{os}-py{python_version}-torch{torch_version}-{ref_name}-` - 同分支、同 PyTorch 版本
  2. `ccache-{os}-py{python_version}-torch{torch_version}-` - 同 PyTorch 版本
  3. `ccache-{os}-py{python_version}-{ref_name}-` - 同分支、同 Python 版本
  4. `ccache-{os}-py{python_version}-` - 同 Python 版本
  5. `ccache-{os}-` - 同操作系统（兜底）

**设计原理**：
- 第一级：同 PyTorch 版本 + 同分支，编译产物最相近，命中率最高
- 第二级：同 PyTorch 版本，ABI 完全兼容
- 第三级：同分支 + 同 Python 版本，代码差异小
- 第四级：同 Python 版本，ABI 兼容
- 第五级：操作系统兜底

**示例配置**：
```yaml
# 注意：此步骤必须在 install_torch 之后
- name: Setup ccache
  uses: actions/cache@v4
  with:
    path: ~/.ccache
    key: ccache-${{ runner.os }}-py${{ env.PYTHON_VERSION }}-torch${{ steps.install_torch.outputs.version }}-${{ github.ref_name }}-${{ github.run_id }}
    restore-keys: |
      ccache-${{ runner.os }}-py${{ env.PYTHON_VERSION }}-torch${{ steps.install_torch.outputs.version }}-${{ github.ref_name }}-
      ccache-${{ runner.os }}-py${{ env.PYTHON_VERSION }}-torch${{ steps.install_torch.outputs.version }}-
      ccache-${{ runner.os }}-py${{ env.PYTHON_VERSION }}-${{ github.ref_name }}-
      ccache-${{ runner.os }}-py${{ env.PYTHON_VERSION }}-
      ccache-${{ runner.os }}-
```

### 配置步骤

```bash
ccache --max-size=2G
ccache --zero-stats
echo "CC=ccache gcc" >> $GITHUB_ENV
echo "CXX=ccache g++" >> $GITHUB_ENV
echo "CCACHE_DIR=$HOME/.ccache" >> $GITHUB_ENV
```

**注意**：使用 `$GITHUB_ENV` 而非 `export`，确保环境变量在后续 step 中生效。

### 构建后统计输出

**必须**在 `if: always()` 条件下输出 ccache 统计：

```bash
echo "## CCache Statistics" >> $GITHUB_STEP_SUMMARY
echo "\`\`\`" >> $GITHUB_STEP_SUMMARY
ccache --show-stats >> $GITHUB_STEP_SUMMARY
echo "\`\`\`" >> $GITHUB_STEP_SUMMARY
```

**要求**：
- 使用代码块格式（\`\`\`）包裹统计输出
- 作为独立的 step，`if: always()` 确保构建失败时也能显示

## 5. 构建策略

- 启用 torchair：`export DISABLE_INSTALL_TORCHAIR=FALSE`
- 不禁用 RPC：不设置 `DISABLE_RPC_FRAMEWORK`
- 保留：`export BUILD_WITHOUT_SHA=1`
- 构建命令：
  - 在 `ascend_pytorch` 目录下执行
  - 日志输出：`python setup.py build bdist_wheel 2>&1 | tee /tmp/build.log`
  - 使用 `${PIPESTATUS[0]}` 获取构建状态
- **必须**在构建命令后执行 `exit ${BUILD_STATUS}`，确保构建失败时步骤正确失败
- 构建成功时：
  - 列出生成的 wheel 文件
  - 输出到 `$GITHUB_OUTPUT`

**构建步骤示例**：
```bash
cd ascend_pytorch
export DISABLE_INSTALL_TORCHAIR=FALSE
export BUILD_WITHOUT_SHA=1
START_TIME=$(date +%s)  # 记录开始时间
python setup.py build bdist_wheel 2>&1 | tee /tmp/build.log
BUILD_STATUS=${PIPESTATUS[0]}
END_TIME=$(date +%s)  # 记录结束时间
DURATION=$((END_TIME - START_TIME))
echo "status=${BUILD_STATUS}" >> $GITHUB_OUTPUT
echo "duration=${DURATION}" >> $GITHUB_OUTPUT  # 输出构建时长（秒）
if [ ${BUILD_STATUS} -eq 0 ]; then
  WHL=$(ls dist/*.whl 2>/dev/null | head -1)
  echo "wheel=${WHL}" >> $GITHUB_OUTPUT
fi
exit ${BUILD_STATUS}  # 必须：确保构建失败时步骤失败
```

## 6. Patch 应用规则

- 遍历方式：`for p in "${PATCH_DIR}"/*.patch; do [ -f "$p" ] || continue; ... done`
- 应用命令：`git apply --directory=ascend_pytorch "$p"`
- **禁止**：`cd ascend_pytorch && git apply`（会导致路径解析错误）
- 记录结果：
  - 成功：追加到 `PATCH_APPLIED`
  - 失败：追加到 `PATCH_FAILED`，不中断流程

## 7. Summary 规则

### 构建信息表格

- 使用表格格式输出构建信息：
  - 表头：`| Item | Details |`
  - 包含：构建时间、构建时长、PyTorch Nightly 版本、Ascend/pytorch Commit（带链接）、Commit 时间、已应用 Patch、构建结果
- 构建时长格式化：将秒数转换为 `Xm Ys` 格式（如 `54m 13s`）
- 构建结果使用文本：
  - 成功：`SUCCESS`
  - 失败：`FAILED`
- 成功时列出 wheel 文件
- 失败时提示查看 build-log artifact

**注意**：不要尝试解析 ccache 输出计算命中率，ccache 统计已在上一步骤以代码块格式输出。

### 兼容性问题分析报告

**必须**在 Summary 中展示 `issues/` 目录下的兼容性问题分析报告。

#### 实现方式

```bash
# 读取 issues 目录下的所有 .md 文件并追加到 summary
if [ -d "issues" ]; then
  echo "## 兼容性问题分析报告" >> $GITHUB_STEP_SUMMARY
  echo "" >> $GITHUB_STEP_SUMMARY
  for issue in issues/*.md; do
    [ -f "$issue" ] || continue
    # 提取文件内容（跳过 YAML front matter 如果有）
    echo "### $(basename "$issue" .md)" >> $GITHUB_STEP_SUMMARY
    echo "" >> $GITHUB_STEP_SUMMARY
    # 提取关键信息：问题描述、根本原因、修复方案
    sed -n '/^## 问题描述/,/^## /p' "$issue" | head -n -1 >> $GITHUB_STEP_SUMMARY
    sed -n '/^## 根本原因分析/,/^## /p' "$issue" | head -n -1 >> $GITHUB_STEP_SUMMARY
    sed -n '/^## 修复方案/,/^## /p' "$issue" | head -n -1 >> $GITHUB_STEP_SUMMARY
    echo "" >> $GITHUB_STEP_SUMMARY
    echo "---" >> $GITHUB_STEP_SUMMARY
    echo "" >> $GITHUB_STEP_SUMMARY
  done
fi
```

#### 报告内容要求

每个 issue 报告应包含以下章节（按顺序）：
1. **问题描述**：简述编译失败现象
2. **根本原因分析**：PyTorch API 变更详情
3. **修复方案**：对应 patch 的核心改动

#### 输出位置

兼容性问题报告位于构建信息表格之后、构建结果之前。

## 8. 关键防坑（必须遵守）

1. 日志 artifact 必须 `if: always()`
   - 保证失败时也能下载日志

2. wheel 上传必须检查构建状态
   - 条件：`if: steps.build.outputs.status == '0'`
   - 不要使用 `if: success()`，因为即使构建失败，前面的步骤可能都成功了

3. ccache 统计必须 `if: always()`
   - 即使构建失败也要显示缓存统计

4. Patch 应用必须使用 `--directory` 参数
   - 错误：`cd ascend_pytorch && git apply "$patch"`
   - 正确：`git apply --directory=ascend_pytorch "$patch"`

5. `$GITHUB_OUTPUT` 必须是单行值
   - 禁止输出多行内容

6. 构建状态使用 `${PIPESTATUS[0]}` 而非 `$?`
   - 因为使用了 `tee` 管道

7. ccache 环境变量必须写入 `$GITHUB_ENV`
   - 错误：`export CC="ccache gcc"`（仅在当前 shell 生效）
   - 正确：`echo "CC=ccache gcc" >> $GITHUB_ENV`（后续 step 可用）

8. ccache 统计输出使用代码块格式
   - 使用 `\`\`\`` 包裹，确保可读性

9. **PyTorch nightly 版本格式必须包含 `+cpu` 后缀**
   - 错误：`torch==2.11.0.dev20260309`（缺少 `+cpu`）
   - 正确：`torch==2.11.0.dev20260309+cpu`
   - 原因：PyTorch nightly CPU 版本的包名包含 `+cpu` 后缀，缺少会导致找不到匹配的包

10. **主版本号不能固定，必须依次尝试**
    - 尝试顺序：`2.12.0` → `2.11.0` → `2.10.0`
    - 原因：PyTorch nightly 不同日期可能对应不同主版本
    - 实现：使用循环尝试，成功则跳出

## 9. 最小验收标准

- workflow 可被 GitHub Actions 识别并触发
- 克隆 Ascend/pytorch（master 分支）
- 正确应用 patches 目录下的 patch 文件
- 构建失败时可下载 `build-log-{run_number}`
- 构建成功时可下载 `dist/*.whl`
- ccache 在第二次运行时命中率显著提升

## 10. 推荐生成提示词（给其他模型）

"请严格按 `SKILL.MD` 生成 `.github/workflows/nightly-build.yml`。要求：按文档顺序组织 step，使用单一 `actions/cache@v4` 设置 ccache，使用 `$GITHUB_ENV` 设置环境变量，克隆 Ascend/pytorch master 分支，应用 `patches/` 目录下的兼容性 patch，设置 `DISABLE_INSTALL_TORCHAIR=FALSE`，**PyTorch nightly 版本格式必须为 `${MAJOR}.dev${DATE}+cpu`（如 `2.11.0.dev20260309+cpu`），主版本号不能固定，需依次尝试 `2.12.0` → `2.11.0` → `2.10.0` 直到成功**，ccache 统计作为独立 step 输出到 step summary（代码块格式），使用表格格式输出构建信息，**必须**在 Summary 中展示 `issues/` 目录下的兼容性问题分析报告（提取问题描述、根本原因分析、修复方案三个章节）。生成后输出完整 YAML。"

---

## 附录：常见问题修复模式

| 错误类型 | 修复方式 |
|----------|----------|
| 结构体成员被删除 | `.cpp` 模块级引入 `static map + mutex` 替代 |
| 成员重命名 | 重命名 Ascend 侧冲突的同名类型 |
| 基类新增纯虚函数 | 子类补充 override（无语义则返回 0） |
| 虚函数签名变更 | 更新 override 签名，原逻辑提取为私有方法 |

---

## 相关文件

- `AGENTS.md` - 项目级别的 Agent 指导文档
- `.github/workflows/nightly-build.yml` - CI workflow 定义
- `patches/` - 兼容性 patch 存放目录
- `issues/` - 问题分析记录目录

# torch_npu-build

目标：仅根据本文件，自动生成可运行的 `.github/workflows/torch_npu-build.yml`，用于构建 `Ascend/pytorch` 的 wheel 包。使用稳定版本的pytorch；

## 1. 输出文件与触发

- 输出文件路径：`.github/workflows/torch_npu-build.yml`
- workflow 名称：`Build pytorch-npu (PyTorch 2.1.0+cpu)`
- 触发方式：
  - 手动：`workflow_dispatch`
- 全局环境变量：无

## 2. Job 基础信息

- 单一 job：`build`
- job 名称：无（使用默认 job ID `build`）
- runner：`ubuntu-22.04`

## 3. 必须包含的步骤（顺序固定）

1. `actions/checkout@v4`
2. `actions/setup-python@v5`，版本固定为 `"3.11"`
3. 安装系统依赖：`cmake ninja-build gcc g++ git patchelf ccache`
4. 设置 ccache：`actions/cache@v4`
5. 配置 ccache 环境变量
6. 安装 Python 依赖：
   - 升级 pip
   - 安装 `torch==2.1.0+cpu`（从 PyTorch CPU 索引）
   - 安装 `pyyaml setuptools auditwheel`
7. 克隆 `Ascend/pytorch`：
   - 参数：`--depth=1 --branch v7.2.0-pytorch2.1.0 --recurse-submodules`
   - 目标目录：`ascend_pytorch`
8. 记录 Ascend/pytorch commit 信息
9. 构建 CANN stub 库：`bash ascend_pytorch/third_party/acl/libs/build_stub.sh`
10. 构建 wheel：`python setup.py build bdist_wheel`
11. 输出 ccache 统计信息
12. 上传 wheel artifact（仅成功时）
13. 上传构建日志 artifact（always）

## 4. ccache 规则

- 缓存 action：`actions/cache@v4`（单一 action，非 restore/save 分开）
- 缓存路径：`~/.ccache`
- 缓存 key：`ccache-ubuntu22-v7.2.0-pytorch2.1.0-${{ github.run_id }}`
- `restore-keys`：
  - `ccache-ubuntu22-v7.2.0-pytorch2.1.0-`
- 配置步骤：
  - `ccache --max-size=2G`
  - `ccache --zero-stats`
  - 设置环境变量：
    - `CC=ccache gcc`
    - `CXX=ccache g++`
    - `CCACHE_DIR=$HOME/.ccache`
- 构建后：
  - 输出 `ccache --show-stats` 到 step summary

## 5. 构建策略

- 禁用 torchair：`export DISABLE_INSTALL_TORCHAIR=TRUE`
- 禁用 RPC：`export DISABLE_RPC_FRAMEWORK=TRUE`
- 保留：`export BUILD_WITHOUT_SHA=1`
- 构建命令：
  - 使用 `set -o pipefail`
  - 在 `ascend_pytorch` 目录下执行
  - 日志输出：`python setup.py build bdist_wheel 2>&1 | tee ../build.log`
- 构建成功时：
  - 在 step summary 中输出成功信息
  - 列出生成的 wheel 文件

## 6. Summary 规则

- 使用表格格式输出构建信息：
  - 表头：`| Key | Value |`
  - 包含：PyTorch 版本、Ascend/pytorch commit
- 构建结果单独输出
- 成功时列出 wheel 文件（使用 while 循环遍历）
- ccache 统计使用代码块格式

## 7. 关键防坑（必须遵守）

1. 日志 artifact 必须 `if: always()`
   - 保证失败时也能下载日志

2. wheel 上传必须仅在成功时执行
   - `if: success()`

3. ccache 统计报告必须 `if: always()`
   - 即使构建失败也要显示缓存统计

## 8. 最小验收标准

- workflow 可被 GitHub Actions 识别并手动触发
- 克隆指定分支的 Ascend/pytorch
- 构建 CANN stub 库
- 构建失败时可下载 `build.log`
- 构建成功时可下载 `dist/*.whl`

## 9. 推荐生成提示词（给其他模型）

"请严格按 `build_skills.md` 生成 `.github/workflows/torch_npu-build.yml`。  
要求：按文档顺序组织 step，使用 `actions/cache@v4` 单一缓存 action，克隆 `v7.2.0-pytorch2.1.0` 分支，构建 CANN stub 库，设置 `DISABLE_INSTALL_TORCHAIR=TRUE` 和 `DISABLE_RPC_FRAMEWORK=TRUE`，使用表格格式输出 step summary。生成后输出完整 YAML。"

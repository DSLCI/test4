# 跨仓库 Workflow 结果通知方案

## 问题背景

`workflow_run` 事件**不能直接跨仓库监听**，只能在同一个仓库内触发。GitHub Actions 的安全模型不允许一个仓库监听另一个仓库的 workflow 事件。

本文档提供多种跨仓库通知解决方案。

---

## 方案一：使用 repository_dispatch（推荐）

### 原理

通过 GitHub API 的 `createDispatchEvent` 接口，从一个仓库向另一个仓库发送自定义事件。

### 实现步骤

#### 1. 在源仓库触发事件

**文件位置**：源仓库 `.github/workflows/nightly-build.yml`

```yaml
jobs:
  build:
    # ... 构建步骤 ...
    outputs:
      build_status: ${{ steps.build.outputs.status }}
      build_duration: ${{ steps.build.outputs.duration }}
      torch_version: ${{ steps.install_torch.outputs.version }}
      commit_sha: ${{ steps.clone.outputs.commit_short }}

  notify-target-repo:
    runs-on: ubuntu-latest
    needs: build
    if: always()
    
    steps:
      - name: Dispatch to target repository
        uses: actions/github-script@v7
        with:
          github-token: ${{ secrets.DISPATCH_TOKEN }}  # 需要 target repo 的写权限
          script: |
            await github.rest.repos.createDispatchEvent({
              owner: 'aflyingto',
              repo: 'torch_npu',  // 目标仓库
              event_type: 'nightly_build_completed',
              client_payload: {
                build_status: '${{ needs.build.outputs.build_status }}',
                build_duration: '${{ needs.build.outputs.build_duration }}',
                torch_version: '${{ needs.build.outputs.torch_version }}',
                commit_sha: '${{ needs.build.outputs.commit_sha }}',
                run_id: '${{ github.run_id }}',
                run_url: '${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}',
                source_repo: '${{ github.repository }}'
              }
            });
```

#### 2. 在目标仓库创建监听 workflow

**文件位置**：目标仓库 `aflyingto/torch_npu/.github/workflows/handle-nightly-build.yml`

```yaml
name: Handle Nightly Build Result

on:
  repository_dispatch:
    types: [nightly_build_completed]

jobs:
  handle:
    runs-on: ubuntu-latest
    
    steps:
      - name: Process build result
        run: |
          echo "## Nightly Build Result" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "| Item | Value |" >> $GITHUB_STEP_SUMMARY
          echo "|------|-------|" >> $GITHUB_STEP_SUMMARY
          echo "| Source Repo | ${{ github.event.client_payload.source_repo }} |" >> $GITHUB_STEP_SUMMARY
          echo "| Build Status | ${{ github.event.client_payload.build_status }} |" >> $GITHUB_STEP_SUMMARY
          echo "| Duration | ${{ github.event.client_payload.build_duration }}s |" >> $GITHUB_STEP_SUMMARY
          echo "| PyTorch Version | ${{ github.event.client_payload.torch_version }} |" >> $GITHUB_STEP_SUMMARY
          echo "| Commit | ${{ github.event.client_payload.commit_sha }} |" >> $GITHUB_STEP_SUMMARY
          echo "| Run URL | [View Details](${{ github.event.client_payload.run_url }}) |" >> $GITHUB_STEP_SUMMARY

      - name: Create issue if build failed
        if: github.event.client_payload.build_status != '0'
        uses: actions/github-script@v7
        with:
          script: |
            await github.rest.issues.create({
              owner: context.repo.owner,
              repo: context.repo.repo,
              title: `Nightly Build Failed - ${new Date().toISOString().split('T')[0]}`,
              body: `
## Build Failed

- **Source Repository**: ${{ github.event.client_payload.source_repo }}
- **PyTorch Version**: ${{ github.event.client_payload.torch_version }}
- **Commit**: ${{ github.event.client_payload.commit_sha }}
- **Duration**: ${{ github.event.client_payload.build_duration }}s

[View Build Details](${{ github.event.client_payload.run_url }})
              `,
              labels: ['bug', 'pytorch-nightly']
            });
```

### 优点
- ✅ 简单易实现
- ✅ 安全性好（使用 GitHub App 或 PAT）
- ✅ 可靠性高
- ✅ 支持双向通信

### 缺点
- 需要配置 PAT 或 GitHub App
- 需要在目标仓库创建监听 workflow

---

## 方案二：使用 GitHub API 直接调用

### 原理

通过 GitHub REST API 直接触发目标仓库的 workflow。

### 实现步骤

#### 1. 在源仓库调用目标仓库 API

**文件位置**：源仓库 `.github/workflows/nightly-build.yml`

```yaml
jobs:
  build:
    # ... 构建步骤 ...

  notify-target-repo:
    runs-on: ubuntu-latest
    needs: build
    if: always()
    
    steps:
      - name: Trigger target repo workflow
        env:
          TARGET_TOKEN: ${{ secrets.TARGET_REPO_TOKEN }}
        run: |
          # 方法1：触发目标仓库的 workflow
          curl -X POST \
            -H "Authorization: token $TARGET_TOKEN" \
            -H "Accept: application/vnd.github.v3+json" \
            https://api.github.com/repos/aflyingto/torch_npu/actions/workflows/handle-result.yml/dispatches \
            -d '{
              "ref": "main",
              "inputs": {
                "build_status": "${{ needs.build.outputs.build_status }}",
                "build_duration": "${{ needs.build.outputs.build_duration }}",
                "torch_version": "${{ needs.build.outputs.torch_version }}",
                "commit_sha": "${{ needs.build.outputs.commit_sha }}",
                "run_id": "${{ github.run_id }}",
                "run_url": "${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}"
              }
            }'
```

#### 2. 在目标仓库创建接收 workflow

**文件位置**：目标仓库 `aflyingto/torch_npu/.github/workflows/handle-result.yml`

```yaml
name: Handle Build Result

on:
  workflow_dispatch:
    inputs:
      build_status:
        description: 'Build status'
        required: true
      build_duration:
        description: 'Build duration'
        required: true
      torch_version:
        description: 'PyTorch version'
        required: true
      commit_sha:
        description: 'Commit SHA'
        required: true
      run_id:
        description: 'Run ID'
        required: true
      run_url:
        description: 'Run URL'
        required: true

jobs:
  handle:
    runs-on: ubuntu-latest
    
    steps:
      - name: Process result
        run: |
          echo "Build Status: ${{ inputs.build_status }}"
          echo "Duration: ${{ inputs.build_duration }}"
          echo "PyTorch Version: ${{ inputs.torch_version }}"
```

### 优点
- ✅ 灵活性高
- ✅ 可以传递任意参数

### 缺点
- 需要管理 token
- 需要手动构造 API 请求

---

## 方案三：使用 GitHub App（最安全）

### 原理

创建一个 GitHub App，授予对两个仓库的权限，使用 App 身份进行跨仓库操作。

### 实现步骤

#### 1. 创建 GitHub App

1. 在 GitHub 创建 GitHub App：`Settings → Developer settings → GitHub Apps → New GitHub App`
2. 授予 App 权限：
   - `contents:write` - 读写仓库内容
   - `actions:write` - 触发 workflow
   - `issues:write` - 创建 issue（可选）
3. 安装 App 到两个仓库
4. 生成 Private Key

#### 2. 在源仓库使用 App

**文件位置**：源仓库 `.github/workflows/nightly-build.yml`

```yaml
jobs:
  build:
    # ... 构建步骤 ...

  notify-target-repo:
    runs-on: ubuntu-latest
    needs: build
    if: always()
    
    steps:
      - name: Generate App Token
        id: app_token
        uses: actions/create-github-app-token@v1
        with:
          app-id: ${{ secrets.APP_ID }}
          private-key: ${{ secrets.APP_PRIVATE_KEY }}
          owner: aflyingto
          repositories: torch_npu

      - name: Dispatch to target repo
        uses: actions/github-script@v7
        with:
          github-token: ${{ steps.app_token.outputs.token }}
          script: |
            await github.rest.repos.createDispatchEvent({
              owner: 'aflyingto',
              repo: 'torch_npu',
              event_type: 'nightly_build_completed',
              client_payload: {
                build_status: '${{ needs.build.outputs.build_status }}',
                build_duration: '${{ needs.build.outputs.build_duration }}',
                torch_version: '${{ needs.build.outputs.torch_version }}',
                commit_sha: '${{ needs.build.outputs.commit_sha }}',
                run_id: '${{ github.run_id }}',
                run_url: '${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}'
              }
            });
```

### 优点
- ✅ 最安全的方案
- ✅ 不需要 PAT
- ✅ 细粒度权限控制
- ✅ 支持 token 过期和轮换

### 缺点
- 配置相对复杂
- 需要创建和管理 GitHub App

---

## 方案四：使用 Webhook（最灵活）

### 原理

在源仓库配置 Webhook，将 workflow 事件发送到外部服务器，再由服务器转发到目标仓库。

### 实现步骤

#### 1. 创建 Webhook 服务器

**文件位置**：你的服务器 `webhook_server.py`

```python
from flask import Flask, request, jsonify
import hmac
import hashlib
import requests

app = Flask(__name__)
GITHUB_WEBHOOK_SECRET = 'your-secret'
GITHUB_TOKEN = 'your-github-token'

def verify_signature(payload, signature):
    expected = 'sha256=' + hmac.new(
        GITHUB_WEBHOOK_SECRET.encode(),
        payload,
        hashlib.sha256
    ).hexdigest()
    return hmac.compare_digest(expected, signature)

@app.route('/webhook', methods=['POST'])
def handle_webhook():
    signature = request.headers.get('X-Hub-Signature-256')
    if not verify_signature(request.data, signature):
        return jsonify({'error': 'Invalid signature'}), 403
    
    data = request.json
    
    if data.get('action') == 'completed':
        workflow_run = data.get('workflow_run', {})
        
        # 转发到目标仓库
        requests.post(
            'https://api.github.com/repos/aflyingto/torch_npu/dispatches',
            headers={
                'Authorization': f'token {GITHUB_TOKEN}',
                'Accept': 'application/vnd.github.v3+json'
            },
            json={
                'event_type': 'nightly_build_completed',
                'client_payload': {
                    'workflow': workflow_run.get('name'),
                    'conclusion': workflow_run.get('conclusion'),
                    'run_url': workflow_run.get('html_url'),
                    'run_id': str(workflow_run.get('id')),
                    'head_branch': workflow_run.get('head_branch'),
                    'head_sha': workflow_run.get('head_sha')
                }
            }
        )
    
    return jsonify({'status': 'success'}), 200

if __name__ == '__main__':
    app.run(port=5000)
```

#### 2. 在源仓库配置 Webhook

在源仓库设置中添加 webhook：
- `Settings → Webhooks → Add webhook`
- Payload URL: `https://your-server.com/webhook`
- Secret: 你的密钥
- Events: 选择 `Workflow runs`

### 优点
- ✅ 最灵活
- ✅ 可以添加自定义逻辑
- ✅ 可以集成多个系统

### 缺点
- 需要维护服务器
- 需要处理安全性
- 增加了系统复杂度

---

## 方案五：使用 gh CLI 监控（本地脚本）

### 原理

在本地使用 `gh` CLI 监控 workflow 运行状态，完成后获取结果。

### 实现步骤

**文件位置**：本地脚本 `monitor-workflow.sh`

```bash
#!/bin/bash
# monitor-workflow.sh

REPO="DSLCI/test4"
WORKFLOW_NAME="Ascend/pytorch Nightly Build Validation"

# 提交代码
git push origin main

# 获取最新的 workflow run
echo "Waiting for workflow to start..."
sleep 10

RUN_ID=$(gh run list --workflow="nightly-build.yml" --limit 1 --json databaseId --jq '.[0].databaseId')

echo "Monitoring workflow run: $RUN_ID"

# 等待完成
gh run watch $RUN_ID --exit-status

# 获取结果
RESULT=$(gh run view $RUN_ID --json conclusion,status,name,headBranch,headSha --jq '.')

echo "Workflow completed!"
echo "$RESULT"

# 发送通知（可选）
# curl -X POST -H 'Content-type: application/json' \
#   --data "{\"text\":\"Build completed: $RESULT\"}" \
#   YOUR_SLACK_WEBHOOK_URL
```

**使用方式**：

```bash
chmod +x monitor-workflow.sh
./monitor-workflow.sh
```

### 优点
- ✅ 简单易用
- ✅ 不需要额外配置
- ✅ 适合开发调试

### 缺点
- 需要手动执行
- 需要保持终端连接

---

## 方案对比

| 方案 | 优点 | 缺点 | 适用场景 |
|------|------|------|----------|
| repository_dispatch | 简单、安全、可靠 | 需要配置权限 | **推荐方案**，生产环境 |
| GitHub API 直接调用 | 灵活性高 | 需要管理 token | 需要精细控制的场景 |
| GitHub App | 最安全、细粒度权限 | 配置复杂 | 企业级应用 |
| Webhook | 最灵活、可自定义 | 需要维护服务器 | 需要复杂集成 |
| gh CLI 监控 | 简单易用 | 需要手动执行 | 开发调试阶段 |

---

## 权限配置

### Personal Access Token (PAT) 方式

1. **创建 PAT**
   - `Settings → Developer settings → Personal access tokens → Tokens (classic)`
   - 权限：`repo`, `workflow`

2. **添加到源仓库的 Secrets**
   - `Settings → Secrets → Actions → New repository secret`
   - Name: `TARGET_REPO_TOKEN`
   - Value: `ghp_xxxxxxxxxxxx`

### GitHub App 方式（推荐）

1. **创建 GitHub App**
   - `Settings → Developer settings → GitHub Apps → New GitHub App`

2. **权限设置**
   - Repository permissions:
     - Contents: Read and write
     - Actions: Read and write
     - Issues: Read and write (可选)

3. **安装 App 到两个仓库**

4. **生成 Private Key 并添加到 Secrets**

---

## 推荐方案

### 对于本项目，推荐使用方案一（repository_dispatch）

**原因**：
1. ✅ 简单易实现
2. ✅ 安全性好
3. ✅ 可靠性高
4. ✅ 支持双向通信

**实现步骤**：
1. 在源仓库添加通知 job
2. 在目标仓库创建监听 workflow
3. 配置权限（PAT 或 GitHub App）

---

## 相关文件

- `AGENTS.md` - 项目架构文档
- `.github/workflows/nightly-build.yml` - CI workflow 定义
- `skills/torch_npu-adapt-pytorch-nightly/SKILL.MD` - Workflow 生成指南

# GitHub Actions 工作流变量说明

本文档列出了所有工作流文件所需的变量和配置。

## 必需的 Secrets

### GITHUB_TOKEN
- **类型**: Secret
- **用途**: GitHub API 认证令牌
- **使用位置**:
  - `issue_comment.yml` - 用于处理 PR 评论验证
  - `excavator.yml` - 用于自动更新 manifest 文件
  - `issues.yml` - 用于处理 issue 事件
  - `pull_request.yml` - 用于处理 PR 事件
- **说明**: 这是 GitHub Actions 自动提供的默认 secret，通常不需要手动配置。如果工作流需要更高的权限，可能需要使用 Personal Access Token (PAT) 替代。

## 环境变量

### SKIP_UPDATED
- **类型**: 环境变量
- **当前值**: `1`
- **使用位置**: `excavator.yml`
- **说明**: 控制 excavator 是否跳过已更新的包。设置为 `1` 表示跳过已更新的包。

## 标准 GitHub 变量

以下变量由 GitHub Actions 自动提供，无需配置：

- `github.event` - 触发工作流的事件数据
- `github.repository` - 仓库信息
- `github.ref` - 分支或标签引用

## 工作流特定说明

### ci.yml
- **无需额外变量**: 此工作流仅使用标准的 GitHub Actions 功能和公开的仓库内容

### excavator.yml
- **触发方式**: 
  - 手动触发 (`workflow_dispatch`)
  - 定时触发（每 4 小时，在每小时的第 20 分钟）
- **权限要求**: 
  - `contents: write` - 需要写入权限以推送更新到仓库
- **环境变量**: 
  - `GITHUB_TOKEN`: 必需
  - `SKIP_UPDATED`: 可选，当前设置为 `1`

### issue_comment.yml
- **触发条件**: 当 PR 评论以 `/verify` 开头时
- **权限要求**: 
  - `contents: write` - 需要写入权限以处理 PR
  - `pull-requests: write` - 需要写入权限以更新 PR 状态
- **环境变量**: 
  - `GITHUB_TOKEN`: 必需

### issues.yml
- **触发条件**: 
  - Issue 被打开
  - Issue 被标记为 `verify` 标签
- **权限要求**: 
  - `contents: write` - 需要写入权限以创建或更新文件
  - `issues: write` - 需要写入权限以更新 issue 状态
  - `pull-requests: write` - 需要写入权限以创建 PR
- **环境变量**: 
  - `GITHUB_TOKEN`: 必需

### pull_request.yml
- **触发条件**: PR 被打开
- **权限要求**: 
  - `contents: write` - 需要写入权限以处理 PR
  - `pull-requests: write` - 需要写入权限以更新 PR 状态
- **环境变量**: 
  - `GITHUB_TOKEN`: 必需

## 权限配置

### 工作流权限
所有需要写入仓库的工作流都已显式配置权限：

- **excavator.yml**: `contents: write` - 推送更新到仓库
- **issue_comment.yml**: `contents: write`, `pull-requests: write` - 处理 PR 评论验证
- **issues.yml**: `contents: write`, `issues: write`, `pull-requests: write` - 处理 issue 和创建 PR
- **pull_request.yml**: `contents: write`, `pull-requests: write` - 处理 PR 事件

这些权限配置确保工作流能够正常执行，无需依赖仓库级别的默认权限设置。

### 权限调用时机详解

#### `contents: write` - 仓库内容写入权限

**调用时机**：
- **excavator.yml**: 
  - 当检测到包有新版本时，更新 manifest 文件（如 `bucket/ccswitch.json`）
  - 提交更改并推送到 `main` 分支
  - 触发条件：定时任务（每 4 小时）或手动触发

- **issue_comment.yml**: 
  - 当有人在 PR 中评论 `/verify` 时
  - 验证 PR 中的 manifest 文件格式和内容
  - 可能需要读取和验证文件内容（虽然主要是读取，但需要 write 权限以访问 PR 分支）

- **issues.yml**: 
  - 当 issue 被打开或标记为 `verify` 标签时
  - 根据 issue 内容创建新的 manifest 文件
  - 创建包含新 manifest 的 PR 分支和文件

- **pull_request.yml**: 
  - 当 PR 被打开时
  - 验证 PR 中的 manifest 文件
  - 可能需要访问 PR 分支的文件内容

#### `pull-requests: write` - PR 写入权限

**调用时机**：
- **issue_comment.yml**: 
  - 在 PR 评论中回复验证结果（添加评论）
  - 更新 PR 状态标签（如添加 `verified` 标签）
  - 如果验证失败，添加错误信息到 PR 评论中
  - 触发条件：PR 评论以 `/verify` 开头

- **issues.yml**: 
  - 根据 issue 内容自动创建新的 PR
  - 在创建的 PR 中添加说明和标签
  - 触发条件：issue 被打开，或 issue 被标记为 `verify` 标签

- **pull_request.yml**: 
  - 在 PR 中添加验证结果评论
  - 更新 PR 状态（通过/失败）
  - 添加或更新 PR 标签
  - 触发条件：PR 被打开

#### `issues: write` - Issue 写入权限

**调用时机**：
- **issues.yml**: 
  - 当 issue 被打开时，添加确认评论
  - 当 issue 被标记为 `verify` 标签时，更新 issue 状态
  - 在 issue 中回复处理进度和结果
  - 关闭已处理的 issue（当 PR 创建成功后）
  - 触发条件：
    - Issue 被打开 (`opened`)
    - Issue 被标记为 `verify` 标签 (`labeled`)

### 权限使用场景示例

#### 场景 1: 用户提交 PR 添加新包
1. **pull_request.yml** 触发（PR 被打开）
2. 使用 `contents: write` 读取 PR 中的 manifest 文件
3. 使用 `pull-requests: write` 添加验证评论到 PR
4. 如果验证通过，添加 `verified` 标签

#### 场景 2: 用户在 PR 中评论 `/verify`
1. **issue_comment.yml** 触发（评论创建）
2. 使用 `contents: write` 访问 PR 分支的文件
3. 使用 `pull-requests: write` 在 PR 中回复验证结果

#### 场景 3: 用户创建 issue 请求新包
1. **issues.yml** 触发（issue 被打开）
2. 使用 `issues: write` 在 issue 中添加确认评论
3. 使用 `contents: write` 创建新的 manifest 文件
4. 使用 `pull-requests: write` 创建包含新 manifest 的 PR
5. 使用 `issues: write` 在 issue 中回复 PR 链接

#### 场景 4: 定时更新包版本
1. **excavator.yml** 触发（定时任务）
2. 使用 `contents: write` 更新 manifest 文件中的版本号和哈希值
3. 使用 `contents: write` 提交并推送到 `main` 分支

### 仓库级别权限设置
如果遇到 `Permission denied` 或 `403` 错误，请检查以下设置：

1. **Actions 权限**:
   - 导航到 `Settings` > `Actions` > `General` > `Actions permissions`
   - 选择 `Allow all actions and reusable workflows`
   - 点击 `Save`

2. **工作流权限**:
   - 导航到 `Settings` > `Actions` > `General` > `Workflow permissions`
   - 选择 `Read and write permissions`
   - 点击 `Save`

## 配置建议

1. **GITHUB_TOKEN**: 
   - 默认情况下，GitHub Actions 会自动提供 `GITHUB_TOKEN`
   - 如果遇到权限问题，可以在仓库 Settings > Secrets and variables > Actions 中检查权限设置
   - 如需更高权限，可创建 Personal Access Token (PAT) 并添加为 secret
   - **注意**: 即使设置了仓库级别的写入权限，工作流文件中显式声明权限也是最佳实践

2. **SKIP_UPDATED**:
   - 如需修改 excavator 的行为，可以更改 `excavator.yml` 中的 `SKIP_UPDATED` 值
   - 建议将此值提取为可配置项（如使用 workflow input 或 repository variable）

## 常见问题

### 错误: `Permission to <repo> denied to github-actions[bot]`
**原因**: 工作流没有足够的权限写入仓库

**解决方案**:
1. 在工作流文件中添加 `permissions: contents: write`（已为 excavator.yml 配置）
2. 检查仓库 Settings > Actions > General > Workflow permissions，确保设置为 `Read and write permissions`
3. 如果问题仍然存在，考虑使用 Personal Access Token (PAT) 替代默认的 `GITHUB_TOKEN`

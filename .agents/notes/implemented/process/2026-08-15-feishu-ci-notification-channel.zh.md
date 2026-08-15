# Agent Note: 飞书 CI 通知通道

Status: implemented

## 问题

`deepseek-ai/deepseek-harness` 仓库的 CI 失败分布在多个工作流（CI、E2E、release、landlock、docs、issue lifecycle），仅能通过 GitHub Actions 网页或邮件感知。主要在飞书协作的开发者无法在飞书内及时收到工作流失败或 master 推送成功的通知，延迟了对构建中断的响应。

## 决策

新增 `workflow_run` 触发的工作流（`.github/workflows/feishu-notify.yml`），通过飞书开放平台机器人 API 向已配置的飞书用户发送交互式卡片消息。该工作流：

- 在任意列出工作流完成后触发（`workflow_run` 事件，`completed` 类型），无需修改现有工作流。
- 每次失败和取消都发送通知；成功时仅在 master 推送运行发送（PR 运行不发），降低噪声。
- 支持 `workflow_dispatch` 手动测试，可模拟 conclusion、名称和 URL。
- 使用应用凭据（`app_id` + `app_secret`）获取 `tenant_access_token`，再向目标用户的 `open_id` 发送交互式卡片，包含仓库、分支、事件、提交和结论等元数据，以及链接到运行页面的"查看运行"按钮。
- 运行器中不使用第三方 Action 或 CLI，仅用 `curl` 和 `jq` 调用飞书 REST API，保持零依赖。

凭据存储为 GitHub 仓库配置：

| Secret 或 variable | 类型 | 用途 |
|---|---|---|
| `DSH_FEISHU_APP_ID` | variable | 飞书自建应用 App ID（`cli_…`） |
| `DSH_FEISHU_APP_SECRET` | secret | 飞书自建应用 App Secret |
| `DSH_FEISHU_OPEN_ID` | variable | 目标用户 `open_id`（`ou_…`） |

使用机器人身份（`tenant_access_token`）而非用户 OAuth token，因为它不会交互式过期，且在 CI 中无需浏览器登录。

## 考虑过但未采用的方案

### 为什么不在运行器中使用 `lark-cli`？

`lark-cli` npm 包需要在运行器中安装和配置，增加 Node 依赖和启动延迟。飞书机器人 REST API 只需两次 `curl` 调用（获取 token + 发送消息），无需安装，在纯 bash 中用 `jq` 即可完成。直接调用 API 使运行器依赖最小。

### 为什么不修改每个现有工作流来添加通知步骤？

每个工作流都需要重复的通知步骤或可复用的 composite action。`workflow_run` 触发器将通知逻辑集中在一个文件中，在任何触发工作流完成后触发，不受其内部结构影响。避免了跨工作流重复，且无需修改现有工作流。

### 为什么不用 GitHub 邮件通知？

GitHub 邮件通知是按用户的，无法按工作流配置，也无法路由到飞书群组或用户。飞书通道直接投递到团队主要即时通讯平台。

### 为什么用 P2P 而非群聊？

初始配置面向单个用户（`open_id`）。将 `DSH_FEISHU_OPEN_ID` 改为群聊 `chat_id` 并切换 `receive_id_type` 即可支持群聊通知——设计未锁定 P2P 投递。

## 后果

CI 失败和 master 推送成功现在以飞书交互卡片形式投递，内含运行页面直链。PR 成功噪声通过 `if` 条件抑制。每次触发工作流完成新增一个 GitHub Actions 作业；该作业很快（两次 HTTP 调用，无 checkout，无构建），运行在 `ubuntu-latest`。凭据为仓库级别，更换时只需更新 GitHub variable/secret 而无需修改工作流文件。
# Code Guardian

在拉取请求（PR）上自动调用 Claude Managed Agents 智能体 **Code Guardian**，做代码审查、安全审查和依赖项检查，并把中文报告发布为 PR 评论。

决定、授权边界和验收状态见 [`docs/CONTEXT.md`](docs/CONTEXT.md)。

## 接入一个仓库

1. 在该仓库添加 Actions 密钥 `ANTHROPIC_API_KEY`（与智能体同一工作区的 Claude API 密钥）。
2. 把 [`examples/code-guardian.yml`](examples/code-guardian.yml) 复制为该仓库的 `.github/workflows/code-guardian.yml`。

调用方用完整提交 SHA 引用 `review.yml`（D10），`code-guardian` 的 main 变化不会自动影响接入的仓库。升级步骤：先审阅 `review.yml` 的改动，确认测试 CI 通过，再在各仓库把 `uses:` 里的 SHA 改成新提交。

## 运行过程

`review.yml` 是可复用工作流（reusable workflow），分两个作业：

| 作业 | 令牌权限 | 做什么 |
|---|---|---|
| `review` | `contents: read`、`pull-requests: read` | 预检智能体与环境配置 → 创建会话（挂载本仓库并检出 PR head 提交，用 `initial_events` 下达审查指令）→ 轮询到结束 → 取最后一条智能体消息 |
| `comment` | `pull-requests: write` | 新建或更新带 `<!-- code-guardian:report -->` 标记的那一条 PR 评论 |

- 克隆代码用 `review` 作业的 `GITHUB_TOKEN`，它只有读权限；发评论的写权限只在 `comment` 作业里出现。
- 会话超过 `timeout_minutes`（默认 60）、作业被取消或出错时，`cleanup` 步骤会中断（`user.interrupt`）并归档会话，停止计费。
- 创建会话的请求有副作用：只在 429 时重试；5xx 或网络错误时报告“结果不明”，不盲目重试。
- 会话不设预算上限。报告页眉显示本次会话的标价费用。

## 输入

| 名称 | 默认值 |
|---|---|
| `agent_id` | `agent_01G5WJUpsPuBTzcL8wuruamd` |
| `environment_id` | `env_01Qwdgk4r6FJ99gGrUkjcfVe` |
| `vault_ids` | `vlt_011Cgx6o8i1rkwtiY1eK2FAD`（逗号分隔，留空表示不挂保管库） |
| `timeout_minutes` | `60` |
| `job_timeout_minutes` | `75`（必须大于 `timeout_minutes`） |
| `anthropic_version` | `2026-10-21` |

## 测试

```bash
pip install pyyaml
python3 -m unittest discover -s tests -v
```

测试从 `review.yml` 中提取实际运行的 Python 步骤，对本地模拟的 Anthropic 与 GitHub 接口执行。

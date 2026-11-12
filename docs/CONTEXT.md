# Code Guardian 共同上下文

本文件暂代 Issue #1，作为项目的唯一共同上下文（见 D8）。

## 目标

在 PR 上自动调用 Claude Managed Agents 智能体 Code Guardian（`agent_01G5WJUpsPuBTzcL8wuruamd`，环境 `env_01Qwdgk4r6FJ99gGrUkjcfVe`），完成代码审查、安全审查和依赖项检查，并把报告发布到 PR。

## 起点（2026-11-12 核实）

`ts19980529-dot/niuniu-channel-sync-telegram-bot#95`（head `8610ea6`）中的工作流无法运行：

- 文件路径嵌套成了 `.github/workflows/.github/workflows/code-guardian.yml`，GitHub 不会读取；
- YAML 没有缩进，且 `run` 被翻译成了 `运行`；
- 只创建会话，没有 `initial_events`，会话不会开始工作；
- 没有挂载仓库，也没有给出 PR 信息。

## 决定

以下引号内为唐硕 2026-11-12 选择的原话。

| 编号 | 事项 | 唐硕的选择 |
|---|---|---|
| D1 | 结果回传 | 「工作流等待并发评论」 |
| D2 | 克隆代码的凭据 | 「用 GITHUB_TOKEN」 |
| D3 | 覆盖范围 | 「做成可复用工作流」 |
| D4 | 单次会话预算 | 「不设上限」 |
| D5 | 托管位置 | 「新建公开仓库 code-guardian」 |
| D6 | 单次审查最长时间 | 「60 分钟」 |
| D7 | 推广节奏 | 「niuniu 验收后再开 6 个 PR」 |
| D8 | 共同上下文位置 | 「仓库文件 docs/CONTEXT.md」（GitHub 连接器的建 Issue 表单在手机端不显示，两次尝试均未创建） |

## 工程选择（可由唐硕改定）

- 调用方引用 `review.yml@main`，修改本仓库后所有接入仓库立即生效。改为锁定提交 SHA 更稳，但每次更新要改 7 个仓库。
- 审查结论不阻止合并：Code Guardian 检查只反映审查是否跑完，没有按严重程度设质量门禁。
- 工具权限策略要求确认（`requires_action`）时，工作流不代为批准，直接结束并归档会话。

## 授权边界

- 已授权：创建本仓库并写入工作流；修复 niuniu PR #95 的分支；首轮验收所需的运行。
- 需要逐项确认：合并任何 PR；设置任何密钥（`ANTHROPIC_API_KEY` 由唐硕在各仓库配置）；向其余 6 个仓库开调用方 PR（D7：在 niuniu 验收之后）。

## 验收条件

1. PR #95 上 Code Guardian 检查运行，预检日志显示智能体和环境的当前配置。
2. 会话用 GITHUB_TOKEN 克隆仓库，并检出 PR head SHA（报告页眉“检出核验：通过”）。
3. PR 上出现中文审查报告评论；推送新提交后更新同一条评论，不重复新建。
4. 超时或取消时，会话被中断并归档。

## 进展记录

- 2026-11-12：建立本仓库。本地 29 个场景测试通过；11 项变异核验全部被测试拦下；actionlint v1.7.12 检查通过。
- 2026-11-12：阻塞。GitHub 连接器写普通文件成功（`examples/code-guardian.yml`，提交 `297779e`），写 `.github/workflows/` 下的文件两次被拒（“connector requires additional permissions”）。`review.yml`、`test.yml` 未上传；niuniu PR #95 的修复也要写该目录，同样受阻。需唐硕在 claude.ai 连接器设置中重新连接 GitHub，授予工作流写权限。
- 2026-11-12 16:01（UTC+7）：唐硕告知已重新授权连接器，并已添加密钥。
- 2026-11-12：推送 `test.yml`（`9aece79`）和 `review.yml`（`1531956`），远端内容哈希与本地一致；`1531956` 上的 test CI（run `43187988789`）29 个测试通过。`9aece79` 的 test CI 失败属预期（当时 `review.yml` 尚未推送）。
- 2026-11-12：PR #95 修复：删除嵌套文件（`e8bdea5`），新建 `.github/workflows/code-guardian.yml`（`c77ba83`，内容与 `examples/` 模板相同）。
- 2026-11-12：首轮运行 run `43188153804`，第 1、2 次尝试结论均为 `missing_api_key`：作业日志中 `ANTHROPIC_API_KEY` 为空值，没有创建会话，没有费用。验收第 3 条部分通过：两次运行只产生一条报告评论（`6819379900`），第 2 次原地更新。待办：确认 niuniu 仓库 Actions 能读到 `ANTHROPIC_API_KEY` 后重跑。
- 2026-11-12 16:11（UTC+7）：唐硕告知密钥已改好。
- 2026-11-12：第 3 次尝试，密钥生效，会话 `sesn_01TouQNX1qj7JeYQK42kasg5` 跑完，但工作流读取会话事件时报 400：`Unknown query parameter 'types' … types[]`。根因是代码用了 `types`，官方要求 `types[]`（文档 curl 示例本已写明，实现时未照用；测试模拟接口也未校验）。清理步骤已归档该会话。
- 2026-11-12：修复 `e6cb87f`：改用 `types[]`；模拟接口改为与真实接口一样拒绝 `types`，旧代码因此有 6 个测试失败，修复后 29 个通过；两处变异核验均被拦下；CI 29/29。
- 2026-11-12：第 4 次尝试（重跑）仍失败。日志显示重跑沿用首跑解析的 `review.yml@1531956`（旧代码），又产生一次会话 `sesn_01LeW9CgTfkiRwnZyKmwhhYx`（已归档）。教训：可复用工作流修复后不能靠重跑，要靠新事件；运行对象的 `referenced_workflows` 可在会话开始前确认版本。
- 2026-11-12：在 PR #95 分支推空提交 `b1ff6fc` 触发新运行 `43193139485`，引用 `review.yml@e6cb87f`。结果 success：会话 `sesn_01RazBfZnK3zfLX8a5otjFJZ`，stop_reason `end_turn`，检出核验通过（HEAD `b1ff6fc`），标价费用 $1.04，运行 8.3 分钟；评论 `6819379900` 原地更新为完整中文报告。仓库原有 CI（run `43193139137`）通过。
- 验收状态：第 1、2、3 条通过；第 4 条部分验证（已归档空闲会话，运行中被中断的路径线上未触发，仅单元测试覆盖）。
- 预检看到的实际配置（与最初建议不同，供唐硕决定是否调整）：模型 `claude-opus-5-5`、effort `xhigh`；工具默认权限策略 `auto`；MCP 服务器 github、context7、parallel-task；环境网络 `unrestricted`（建议为 `limited`）；环境的 go 包列表为空（未预装 `osv-scanner`、`gitleaks`）。首份报告显示会话内实际运行了 gitleaks 8.28.0、zizmor 1.32.0、semgrep 1.185.0、pip-audit 2.10.1、actionlint 1.7.12。
- 首份报告的待决事项：M1 `@main` 未锁定 SHA，且 code-guardian 的 main 无分支保护；L1 默认保管库中的 GitHub 凭据为所有者本人、对 code-guardian 有 admin 权限（会话内 `get_me`/`search_repositories` 实测），建议换成只读、限定仓库的 fine-grained token。

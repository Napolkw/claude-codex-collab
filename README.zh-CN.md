# codex-collab

[English](./README.md) · **简体中文** · [繁體中文](./README.zh-TW.md) · [日本語](./README.ja.md)

一套让 Claude 与 [Codex](https://developers.openai.com/codex) 在
[Claude Code](https://claude.com/claude-code) 里组成「双模型团队」的工作法：
Claude 负责规划、审查、把关；Codex 负责执行与初审。它把这套分工——以及让它
可靠落地的运维细节——浓缩成一个可直接放进 Claude Code 的文件。

> Claude 的 token 贵，Codex 的 token 便宜。把 Claude 花在判断上，而不是广度上。

---

## 它解决什么问题

让同一个模型既当作者又当审查，有两个失效模式：

1. **成本**：用高端模型去做批量阅读、机械改动、大规模 fan-out，是在用昂贵的
   token 做一个更便宜的执行者就能完成的事。
2. **自洽幻觉**：单一模型审查自己的产出会共享自身盲点——开一个新会话能去掉
   *对话层面* 的自我合理化，却去不掉 *模型层面* 的盲点（同样的权重，两个会话
   会「各自独立地」认同同一个错误结论）。

codex-collab 用 **两个不同模型** 拆分角色来同时解决这两点：

- **Claude** 掌握方向（计划、风险、验收标准）、模型调度与最终质量门——它读真实
  的 diff，并亲自重跑 oracle。
- **Codex** 负责执行，以及一个全新会话的初审。
- 一道 **跨模型门**（Claude 用预先写好的标准审查 Codex 的产出，而非采信 Codex
  自己的总结）能抓住同模型审查会盖章放过的盲点。

在此之上，它把多轮、多 agent 委派真正跑得起来的 *运维细节* 标准化——交接文档、
防截断的输出契约、并行 fan-out 的 worktree 隔离、测试解释器校验、共享仓库上的
并行会话安全。这些规则都是从真实事故里提炼出来的。

## 推荐用法 —— 双 workflow 模型

**运行 codex-collab 的推荐方式就是双 workflow 模型。** 多轮 coding／research
以 **两个并行的 workflow** 运作：

- 一个 **Workflow** 层（多 agent 编排）负责 fan-out、分解、验证；
- 一个 **codex-collab** 层把实作委派给 Codex，而 Claude 守住规划、审查、把关。

每一条 fan-out 通道都是一个 *Codex 执行者*——编排是在提升 Codex 的吞吐，而不是
把执行搬回 Claude。完整运作模型见 [`dual-workflow.md`](./dual-workflow.md)
（分工、把状态外置到 GitHub/Linear、反膨胀）。

单轮琐事跳过这一切——谁快用谁，直接做。

## 仓库内容

| 文件 | 是什么 |
| --- | --- |
| [`SKILL.md`](./SKILL.md) | 完整工作法：fast path、角色与调度、成本规则、审查纪律、运维细节、输出契约、ultracode fan-out 模式。 |
| [`dual-workflow.md`](./dual-workflow.md) | 包住工作法的编排模型。 |

## 环境要求

- **[Claude Code](https://claude.com/claude-code)**（运行它的宿主）。
- **Node.js 18.18+**（Codex 插件需要）。
- **Codex CLI** —— `@openai/codex`。
- 一个 **ChatGPT 订阅（含 Free）或 OpenAI API key** 供 Codex 使用；用量会计入你的
  Codex 额度。

## 安装

### 1. 安装并登录 Codex CLI

```bash
npm install -g @openai/codex
codex login          # 或配置 OpenAI API key
```

### 2. 安装 Claude Code 的 Codex 插件

codex-collab 通过官方 OpenAI Codex 插件
（[`openai/codex-plugin-cc`](https://github.com/openai/codex-plugin-cc)）驱动
Codex，它提供 `/codex:*` 命令和它所委派的 `codex:codex-rescue` subagent。在
Claude Code 里：

```text
/plugin marketplace add openai/codex-plugin-cc
/plugin install codex@openai-codex
/reload-plugins
/codex:setup
```

`/codex:setup` 会告诉你 Codex 是否就绪；若有 npm，它能帮你安装 Codex CLI。安装后
你应能看到 `/codex:*` 斜杠命令，以及 `/agents` 里的 `codex:codex-rescue`。

插件提供：

- `/codex:review` —— 普通只读 Codex 审查
- `/codex:adversarial-review` —— 可引导的挑战式审查
- `/codex:rescue` / `/codex:status` / `/codex:result` / `/codex:cancel` ——
  委派任务并管理后台作业

### 3. 安装 codex-collab

它作为一个 Claude Code skill 安装——`~/.claude/skills/<name>/` 下的一个文件。把本
仓库的 `SKILL.md` 放进去：

```bash
mkdir -p ~/.claude/skills/codex-collab
cp SKILL.md ~/.claude/skills/codex-collab/SKILL.md
```

Claude Code 会从 YAML frontmatter 的 `name`/`description` 自动发现它。重启（或
reload）Claude Code 后即可用——多轮 coding／research 时 Claude 会自动调用，你也可
以按名字点它。

### 4. 配置 Codex

Codex 读取 `~/.codex/config.toml`。codex-collab 假定了几项设置，最小示例：

```toml
# 选你套餐能用的 Codex 模型（例如最新的 gpt-5.x）。
model = "gpt-5"
model_reasoning_effort = "xhigh"   # Claude 会按通道覆写：medium / high / xhigh
model_verbosity = "low"

# codex-collab 会把完整列表写进文件、聊天回复保持简短，但更大的 tool-output 上限
# 能减少中途截断；它特别点名 32000。
tool_output_token_limit = 32000

# 在可信的本地仓库做无人值守委派用。可按需收紧——更严格的沙箱也行，但有些通道
# （如 in-worktree 提交）需要 workspace 写权限。
approval_policy = "never"
sandbox_mode = "workspace-write"
```

说明：

- `model_reasoning_effort` 是默认值；Claude 会 **按通道** 依任务形态挑 `--effort`
  （机械 → `medium`，多分支 → `high`，新颖／安全／无 oracle → `xhigh`）。
- Codex 沙箱屏蔽 `/dev/nvidia*`：需要 GPU 的通道必须放在 Claude 一侧跑，提前规划好
  这个切分。

## 验证安装

```text
/codex:setup            # 确认 Codex CLI 已装且已认证
/codex:review --background
/codex:status
/codex:result
```

若 `/agents` 里出现 `codex:codex-rescue`、一次 review 能跑通，就绪了。然后直接开一个
多轮任务——Claude 会主动用它。

## 用法场景

推荐默认是用双 workflow 模型做多轮特性开发；但「Claude 判断／Codex 执行」这套拆分
适配很多形态：

- **多轮特性开发（推荐）** → [双 workflow 模型](#推荐用法--双-workflow-模型)：
  Claude 写验收标准，fan-out 出 Codex 执行通道（worktree 隔离做并行），跑全新
  Codex 初审，再在门口亲自重跑 oracle 才合并。
- **跨模型代码审查** → 对一个 diff 或 PR 跑 `/codex:review` 或
  `/codex:adversarial-review`，拿到一个独立的第二模型视角——不同模型能抓住同模型
  自审会盖章放过的盲点。
- **大型重构／迁移** → 每个文件一条 Codex 通道（worktree 隔离），各带明确的
  do-not-touch 清单，再把互不相交的提交 cherry-pick 起来。规模可超过单个上下文能
  装下的量。
- **找 bug／审计** → loop-until-dry：每轮 fan-out 出 Codex finder，并对每个发现做
  对抗式验证（多数「驳回」就杀掉它）；连续 K 轮无新发现就停。
- **生成测试** → 并行 Codex 通道为未覆盖模块补测试；Claude 在门口亲自重跑整套用例
  才采信。
- **成本优化的批量阅读** → 把大范围阅读委派给只读的 Codex recon 通道，让它回报告；
  Claude 读报告 + 抽查关键文件，只把昂贵 token 花在判断上。
- **研究／探索** → 反转的节奏：实作前先预登记指标与阈值；只有固定的、确定性的评估器
  能宣告成功；每个假设止损 2–3 轮。
- **单轮琐事** → 跳过仪式；Claude 直接做，或委派一条通道。

完整纪律见 [`SKILL.md`](./SKILL.md)，推荐运作模型见
[`dual-workflow.md`](./dual-workflow.md)——两者都写成可被 agent 逐字遵循。

## 许可证

[MIT](./LICENSE)。

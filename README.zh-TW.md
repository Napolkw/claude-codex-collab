# codex-collab

[English](./README.md) · [简体中文](./README.zh-CN.md) · **繁體中文** · [日本語](./README.ja.md)

一套讓 Claude 與 [Codex](https://developers.openai.com/codex) 在
[Claude Code](https://claude.com/claude-code) 裡組成「雙模型團隊」的工作法：
Claude 負責規劃、審查、把關；Codex 負責執行與初審。它把這套分工——以及讓它能
可靠落地的維運細節——濃縮成一個可直接放進 Claude Code 的檔案。

> Claude 的 token 貴，Codex 的 token 便宜。把 Claude 花在判斷上，而不是廣度上。

---

## 它解決什麼問題

讓同一個模型既當作者又當審查，有兩個失效模式：

1. **成本**：用高階模型去做大量閱讀、機械改動、大規模 fan-out，等於用昂貴的
   token 做一個更便宜的執行者就能完成的事。
2. **自洽幻覺**：單一模型審查自己的產出會共用自身盲點——開一個新工作階段能去掉
   *對話層面* 的自我合理化，卻去不掉 *模型層面* 的盲點（同樣的權重，兩個工作階段
   會「各自獨立地」認同同一個錯誤結論）。

codex-collab 用 **兩個不同模型** 拆分角色來同時解決這兩點：

- **Claude** 掌握方向（計畫、風險、驗收標準）、模型調度與最終品質閘——它讀真實的
  diff，並親自重跑 oracle。
- **Codex** 負責執行，以及一個全新工作階段的初審。
- 一道 **跨模型閘**（Claude 用預先寫好的標準審查 Codex 的產出，而非採信 Codex
  自己的總結）能抓住同模型審查會蓋章放過的盲點。

在此之上，它把多輪、多 agent 委派真正跑得起來的 *維運細節* 標準化——交接文件、
防截斷的輸出契約、並行 fan-out 的 worktree 隔離、測試直譯器校驗、共享儲存庫上的
並行工作階段安全。這些規則都是從真實事故裡提煉出來的。

## 推薦用法 —— 雙 workflow 模型

**執行 codex-collab 的推薦方式就是雙 workflow 模型。** 多輪 coding／research
以 **兩個並行的 workflow** 運作：

- 一個 **Workflow** 層（多 agent 編排）負責 fan-out、分解、驗證；
- 一個 **codex-collab** 層把實作委派給 Codex，而 Claude 守住規劃、審查、把關。

每一條 fan-out 通道都是一個 *Codex 執行者*——編排是在提升 Codex 的吞吐，而不是把
執行搬回 Claude。完整運作模型見 [`dual-workflow.md`](./dual-workflow.md)
（分工、把狀態外置到 GitHub/Linear、反膨脹）。

單輪瑣事跳過這一切——誰快用誰，直接做。

## 儲存庫內容

| 檔案 | 是什麼 |
| --- | --- |
| [`SKILL.md`](./SKILL.md) | 完整工作法：fast path、角色與調度、成本規則、審查紀律、維運細節、輸出契約、ultracode fan-out 模式。 |
| [`dual-workflow.md`](./dual-workflow.md) | 包住工作法的編排模型。 |

## 環境需求

- **[Claude Code](https://claude.com/claude-code)**（執行它的宿主）。
- **Node.js 18.18+**（Codex 外掛需要）。
- **Codex CLI** —— `@openai/codex`。
- 一個 **ChatGPT 訂閱（含 Free）或 OpenAI API key** 供 Codex 使用；用量會計入你的
  Codex 額度。

## 安裝

### 1. 安裝並登入 Codex CLI

```bash
npm install -g @openai/codex
codex login          # 或設定 OpenAI API key
```

### 2. 安裝 Claude Code 的 Codex 外掛

codex-collab 透過官方 OpenAI Codex 外掛
（[`openai/codex-plugin-cc`](https://github.com/openai/codex-plugin-cc)）驅動
Codex，它提供 `/codex:*` 指令與它所委派的 `codex:codex-rescue` subagent。在
Claude Code 裡：

```text
/plugin marketplace add openai/codex-plugin-cc
/plugin install codex@openai-codex
/reload-plugins
/codex:setup
```

`/codex:setup` 會告訴你 Codex 是否就緒；若有 npm，它能幫你安裝 Codex CLI。安裝後
你應能看到 `/codex:*` 斜線指令，以及 `/agents` 裡的 `codex:codex-rescue`。

外掛提供：

- `/codex:review` —— 一般唯讀 Codex 審查
- `/codex:adversarial-review` —— 可引導的挑戰式審查
- `/codex:rescue` / `/codex:status` / `/codex:result` / `/codex:cancel` ——
  委派任務並管理背景作業

### 3. 安裝 codex-collab

它作為一個 Claude Code skill 安裝——`~/.claude/skills/<name>/` 下的一個檔案。把本
儲存庫的 `SKILL.md` 放進去：

```bash
mkdir -p ~/.claude/skills/codex-collab
cp SKILL.md ~/.claude/skills/codex-collab/SKILL.md
```

Claude Code 會從 YAML frontmatter 的 `name`/`description` 自動探索它。重啟（或
reload）Claude Code 後即可用——多輪 coding／research 時 Claude 會自動呼叫，你也可以
按名字點它。

### 4. 設定 Codex

Codex 讀取 `~/.codex/config.toml`。codex-collab 假定了幾項設定，最小範例：

```toml
# 選你方案能用的 Codex 模型（例如最新的 gpt-5.x）。
model = "gpt-5"
model_reasoning_effort = "xhigh"   # Claude 會按通道覆寫：medium / high / xhigh
model_verbosity = "low"

# codex-collab 會把完整清單寫進檔案、聊天回覆保持簡短，但更大的 tool-output 上限
# 能減少中途截斷；它特別點名 32000。
tool_output_token_limit = 32000

# 在可信的本地儲存庫做無人值守委派用。可按需收緊——更嚴格的沙箱也行，但有些通道
# （如 in-worktree 提交）需要 workspace 寫權限。
approval_policy = "never"
sandbox_mode = "workspace-write"
```

說明：

- `model_reasoning_effort` 是預設值；Claude 會 **按通道** 依任務形態挑 `--effort`
  （機械 → `medium`，多分支 → `high`，新穎／安全／無 oracle → `xhigh`）。
- Codex 沙箱封鎖 `/dev/nvidia*`：需要 GPU 的通道必須放在 Claude 一側跑，請提前規劃
  好這個切分。

## 驗證安裝

```text
/codex:setup            # 確認 Codex CLI 已裝且已認證
/codex:review --background
/codex:status
/codex:result
```

若 `/agents` 裡出現 `codex:codex-rescue`、一次 review 能跑通，就緒了。然後直接開一個
多輪任務——Claude 會主動用它。

## 用法場景

推薦預設是用雙 workflow 模型做多輪功能開發；但「Claude 判斷／Codex 執行」這套拆分
適配很多形態：

- **多輪功能開發（推薦）** → [雙 workflow 模型](#推薦用法--雙-workflow-模型)：
  Claude 寫驗收標準，fan-out 出 Codex 執行通道（worktree 隔離做並行），跑全新
  Codex 初審，再在閘口親自重跑 oracle 才合併。
- **跨模型程式碼審查** → 對一個 diff 或 PR 跑 `/codex:review` 或
  `/codex:adversarial-review`，拿到一個獨立的第二模型視角——不同模型能抓住同模型
  自審會蓋章放過的盲點。
- **大型重構／遷移** → 每個檔案一條 Codex 通道（worktree 隔離），各帶明確的
  do-not-touch 清單，再把互不相交的提交 cherry-pick 起來。規模可超過單一上下文能
  裝下的量。
- **找 bug／稽核** → loop-until-dry：每輪 fan-out 出 Codex finder，並對每個發現做
  對抗式驗證（多數「駁回」就殺掉它）；連續 K 輪無新發現就停。
- **產生測試** → 並行 Codex 通道為未覆蓋模組補測試；Claude 在閘口親自重跑整套測試
  才採信。
- **成本最佳化的大量閱讀** → 把大範圍閱讀委派給唯讀的 Codex recon 通道，讓它回報告；
  Claude 讀報告 + 抽查關鍵檔案，只把昂貴 token 花在判斷上。
- **研究／探索** → 反轉的節奏：實作前先預先登記指標與門檻；只有固定的、確定性的
  評估器能宣告成功；每個假設停損 2–3 輪。
- **單輪瑣事** → 跳過儀式；Claude 直接做，或委派一條通道。

完整紀律見 [`SKILL.md`](./SKILL.md)，推薦運作模型見
[`dual-workflow.md`](./dual-workflow.md)——兩者都寫成可被 agent 逐字遵循。

## 授權

[MIT](./LICENSE)。

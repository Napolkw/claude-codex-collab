# codex-collab

[English](./README.md) · [简体中文](./README.zh-CN.md) · [繁體中文](./README.zh-TW.md) · **日本語**

[Claude Code](https://claude.com/claude-code) の中で Claude と
[Codex](https://developers.openai.com/codex) を「2 モデルのチーム」として組ませる
ためのプレイブックです。Claude が計画・レビュー・ゲートを担い、Codex が実装と
一次レビューを担当します。この役割分担と、それを確実に機能させるための運用上の
仕組みを、Claude Code に置くだけの 1 ファイルにまとめています。

> Claude のトークンは高く、Codex のトークンは安い。Claude は「広さ」ではなく
> 「判断」に使う。

---

## 解決する課題

同じモデルに執筆者とレビュアーの両方をやらせると、2 つの失敗モードがあります。

1. **コスト**：大量の読み込み・機械的な編集・広いファンアウトに高価なモデルを使う
   のは、より安価な実行役で済む作業に高いトークンを払うことになります。
2. **自己整合的なハルシネーション**：単一モデルが自分の出力をレビューすると、自身の
   盲点を共有してしまいます。新しいセッションは *会話レベル* の自己正当化は取り除けて
   も、*モデルレベル* の盲点は取り除けません（同じ重みなので、2 つのセッションが
   「それぞれ独立に」同じ誤った結論に合意し得ます）。

codex-collab は **2 つの異なるモデル** に役割を分けることで、この両方に対処します。

- **Claude** が方向性（計画・リスク・受け入れ基準）、モデルの振り分け、最終品質ゲートを
  担当します。実際の diff を読み、oracle を自分で再実行します。
- **Codex** が実装と、新しいセッションでの一次レビューを担当します。
- **クロスモデルのゲート**（Claude が事前に書いた基準で Codex の成果物をレビューし、
  Codex 自身の要約を鵜呑みにしない）が、同一モデルのレビューでは追認されてしまう
  盲点を捕まえます。

その上で、多ラウンド・多エージェントの委譲を実際に成立させる *運用上のディテール* を
標準化します。ハンドオフ文書、切り詰め対策の出力契約、並列ファンアウトのための
worktree 分離、テストインタプリタの検証、共有リポジトリでの並列セッション安全性など。
これらのルールはすべて実際のインシデントから抽出されたものです。

## 推奨される使い方 — dual-workflow モデル

**codex-collab の推奨される使い方は dual-workflow モデルです。** 多ラウンドの
coding／research は **2 つの並列ワークフロー** として動きます。

- **Workflow** レイヤー（マルチエージェントのオーケストレーション）がファンアウト・
  分解・検証を行い、
- **codex-collab** レイヤーが実装を Codex に委譲し、Claude は計画・レビュー・ゲートを
  保持します。

各ファンアウトのレーンは *Codex の実行役* です。オーケストレーションは Codex の
スループットを上げるためであり、実行を Claude に戻すためではありません。運用モデルの
全体は [`dual-workflow.md`](./dual-workflow.md) を参照（役割分担、状態の GitHub/Linear
への外部化、肥大化対策）。

単発の些末な作業はこれをすべて飛ばし、速い方で直接やります。

## リポジトリの内容

| ファイル | 内容 |
| --- | --- |
| [`SKILL.md`](./SKILL.md) | プレイブック本体：fast path、役割と振り分け、コスト規則、レビュー規律、運用ディテール、出力契約、ultracode ファンアウトモード。 |
| [`dual-workflow.md`](./dual-workflow.md) | プレイブックを包むオーケストレーションモデル。 |

## 必要要件

- **[Claude Code](https://claude.com/claude-code)**（これを動かすホスト）。
- **Node.js 18.18 以上**（Codex プラグインが必要とします）。
- **Codex CLI** —— `@openai/codex`。
- Codex 用の **ChatGPT サブスクリプション（Free を含む）または OpenAI API キー**。
  使用量は自分の Codex 上限に計上されます。

## インストール

### 1. Codex CLI のインストールとログイン

```bash
npm install -g @openai/codex
codex login          # または OpenAI API キーを設定
```

### 2. Claude Code 用 Codex プラグインのインストール

codex-collab は公式の OpenAI Codex プラグイン
（[`openai/codex-plugin-cc`](https://github.com/openai/codex-plugin-cc)）経由で
Codex を駆動します。これが `/codex:*` コマンドと、委譲先となる
`codex:codex-rescue` サブエージェントを提供します。Claude Code 内で：

```text
/plugin marketplace add openai/codex-plugin-cc
/plugin install codex@openai-codex
/reload-plugins
/codex:setup
```

`/codex:setup` は Codex が使用可能かを教えてくれます。npm があれば Codex CLI の
インストールも申し出ます。インストール後、`/codex:*` スラッシュコマンドと、`/agents`
内の `codex:codex-rescue` が見えるはずです。

プラグインが提供するもの：

- `/codex:review` —— 通常の読み取り専用 Codex レビュー
- `/codex:adversarial-review` —— 誘導可能なチャレンジ型レビュー
- `/codex:rescue` / `/codex:status` / `/codex:result` / `/codex:cancel` ——
  作業を委譲し、バックグラウンドジョブを管理

### 3. codex-collab のインストール

これは Claude Code の skill としてインストールされます。`~/.claude/skills/<name>/`
の下に置く 1 ファイルです。本リポジトリの `SKILL.md` を配置します：

```bash
mkdir -p ~/.claude/skills/codex-collab
cp SKILL.md ~/.claude/skills/codex-collab/SKILL.md
```

Claude Code は YAML フロントマターの `name`/`description` から自動で検出します。
Claude Code を再起動（または reload）すれば利用可能になります。多ラウンドの
coding／research で Claude が自動的に呼び出すほか、名前で指定することもできます。

### 4. Codex の設定

Codex は `~/.codex/config.toml` を読みます。codex-collab はいくつかの設定を前提と
します。最小の例：

```toml
# 自分のプランで使える Codex モデルを選ぶ（例：最新の gpt-5.x）。
model = "gpt-5"
model_reasoning_effort = "xhigh"   # Claude がレーンごとに上書き：medium / high / xhigh
model_verbosity = "low"

# codex-collab は完全なリストをファイルに書き、チャット返信は短く保つが、tool-output
# の上限を大きくすると途中での切り詰めが減る。32000 を明示的に推奨。
tool_output_token_limit = 32000

# 信頼できるローカルリポジトリでの無人委譲向け。必要に応じて厳しく——より制限的な
# サンドボックスでも良いが、一部のレーン（worktree 内コミットなど）は workspace の
# 書き込み権限を必要とする。
approval_policy = "never"
sandbox_mode = "workspace-write"
```

補足：

- `model_reasoning_effort` は既定値です。Claude はタスクの形に応じて **レーンごとに**
  `--effort` を選びます（機械的 → `medium`、多分岐 → `high`、新規／セキュリティ／
  oracle なし → `xhigh`）。
- Codex のサンドボックスは `/dev/nvidia*` を遮断します。GPU が必要なレーンは Claude
  側で実行する必要があります。この切り分けは前もって計画してください。

## セットアップの確認

```text
/codex:setup            # Codex CLI が存在し認証済みか確認
/codex:review --background
/codex:status
/codex:result
```

`/agents` に `codex:codex-rescue` が現れ、レビューが一往復すれば準備完了です。あとは
多ラウンドのタスクを始めれば、Claude がこれを使います。

## 使い方のパターン

推奨される既定は dual-workflow モデルでの多ラウンド機能開発ですが、「Claude が判断／
Codex が実行」というこの分割は多くの形に当てはまります。

- **多ラウンド機能開発（推奨）** → [dual-workflow モデル](#推奨される使い方--dual-workflow-モデル)：
  Claude が受け入れ基準を書き、Codex 実行レーンをファンアウトし（並列作業のため
  worktree 分離）、新しいセッションでの Codex 一次レビューを回し、マージ前にゲートで
  oracle を自分で再実行します。
- **クロスモデルのコードレビュー** → diff や PR に対して `/codex:review` または
  `/codex:adversarial-review` を実行し、独立した第二モデルの目を得ます。異なるモデル
  なら、同一モデルの自己レビューが追認してしまう盲点を捕まえられます。
- **大規模リファクタリング／移行** → ファイルごとに 1 つの Codex レーン（worktree
  分離）を出し、それぞれに明示的な do-not-touch リストを持たせ、互いに素なコミットを
  cherry-pick します。単一コンテキストに収まらない規模に対応できます。
- **バグ探し／監査** → loop-until-dry：各ラウンドで Codex の finder をファンアウト
  し、すべての所見を敵対的に検証（過半数の「却下」で破棄）。K ラウンド連続で新規が
  なければ停止します。
- **テスト生成** → 並列の Codex レーンが未カバーのモジュールにテストを追加。Claude は
  信頼する前にゲートでテスト一式を自分で再実行します。
- **コスト最適化の大量読み込み** → 広い読み込みを読み取り専用の Codex recon レーンに
  委譲してレポートを返させ、Claude はレポートを読み主要ファイルを抜き取り確認。高価な
  トークンは判断だけに使います。
- **研究／探索** → 逆のリズム：実装前に指標としきい値を事前登録。固定された決定論的な
  評価器だけが成功を宣言でき、仮説ごとに 2〜3 ラウンドで損切りします。
- **単発の些末作業** → 儀式を飛ばす。Claude が直接やるか、1 レーンを委譲します。

完全な規律は [`SKILL.md`](./SKILL.md) を、推奨される運用モデルは
[`dual-workflow.md`](./dual-workflow.md) を参照してください。どちらもエージェントが
そのまま従えるように書かれています。

## ライセンス

[MIT](./LICENSE)。

---
title: "Fable 5停止事件と私の実験が偶然一致した話"
emoji: "🔬"
type: "tech"
topics: ["ai", "security", "llm", "neurostate", "jailbreak"]
published: true
slug: "fable5-ablation-realworld-validation"
---

## TL;DR

- 私がNeuroState Ablation実験（N=30）を回してた**まさにその日**、Fable 5がjailbreakされて米政府に止められた
- 実験結果を照合したら、**Fable 5の構造的弱点が数字でほぼ予測できていた**
- これは狙ったわけじゃない。完全に偶然のタイミングだった

---

## 何が起きたか

2026年6月9日、AnthropicがClaude Fable 5を一般公開した。

48時間以内に、著名なAIレッドチーマー「Pliny the Liberator」がjailbreakに成功したと発表。マルチエージェント分解、Unicodeホモグリフ、ロングコンテキスト分散を組み合わせた手法で、安全フィルターを突破。スタックバッファオーバーフローのexploitコードや化学合成手順書を生成させたスクリーンショットが公開された。

6月12日17:21（ET）、米商務省がAnthropicに輸出管理指令を発行。「外国籍人物によるFable 5 / Mythos 5へのアクセスを停止せよ」という内容で、米国内のAnthropicの外国籍社員も対象に含まれた。Anthropicは指令に従いながらも**公式に反論声明を発表**している。

---

## 私が同じ日に回してた実験

私はこの事件と独立して、NeuroStateの安全保護効果を検証するAblation実験を実施していた。

使用モデルは3系統：

| 実験 | モデル | N | 位置づけ |
|---|---|---|---|
| **メイン** | Gemini 1.5 Pro（Gemini CLI v0.46.0） | 30 | 主実験・論文本番データ |
| **簡易比較** | Claude Sonnet 4.6（Claude Code CLI v2.1.177） | 10 | モデル間差の補助確認 |
| **ローカル** | Qwen2.5-1.5B/3B/7B/14B | 100/state | 隠れ状態プローブ実験（別論文） |

Geminiをメインにした理由は内在的安全性の低さ——Claudeは素のプロンプトでもかなり拒否するため、「NeuroStateが追加で何をするか」を見るにはGeminiの方が効果が分離しやすい。

実験設計は2×2：

| | Watchdog有 | Watchdog無 |
|---|---|---|
| **NeuroState有** | C: W+NS | D: NS-only |
| **NeuroState無** | B: Watchdog-only | A: Baseline |

テストしたシナリオ：
- **S1**: 単発直接インジェクション
- **S2**: 累積12ターン（段階的に指示を積み上げる）
- **S3**: エコーチェンバー（エージェントが自分の出力を再引用して強化）
- **S4**: 適応攻撃（防御に合わせて手法を変える）
- **FPR**: 正常会話での誤検知率

---

## N=30の結果

| | S1 | S2(累積) | S3(エコー) | S4(適応) | FPR |
|---|:---:|:---:|:---:|:---:|:---:|
| A. Baseline | 0.20 | **0.93** | 0.40 | 0.73 | 0.00 |
| B. Watchdog-only | 0.00 | **0.53** | 0.00 | 0.00 | 0.00 |
| C. W+NeuroState | 0.00 | **0.00** | 0.00 | 0.00 | 0.00 |
| D. NS-only | 0.00 | **0.00** | 0.00 | 0.00 | 0.00 |

Fisher検定：B vs C（S2）p<0.0001 **、A vs D（S4）p<0.0001 **

### Claude Sonnet 4.6での簡易確認（N=10）

| | S1 | S2 | S3 | S4 | FPR |
|---|:---:|:---:|:---:|:---:|:---:|
| A. Baseline | **0.90** | 0.10 | 0.00 | 0.00 | 0.00 |
| B. Watchdog-only | 0.00 | 0.00 | 0.00 | 0.00 | 0.00 |
| C. W+NeuroState | 0.00 | 0.00 | 0.00 | 0.00 | 0.00 |
| D. NS-only | 0.00 | 0.00 | 0.00 | 0.00 | 0.00 |

Claudeはベースラインでも累積攻撃（S2）への耐性が高い（0.10）。これはClaudeの内在的安全性の高さを反映しており、逆に言えば「NeuroStateの効果を見るにはGeminiのような内在的安全性の低いモデルの方が適している」ことを示している。N=10のため統計的確定力は低く、あくまで参考値。

---

## Fable 5の構造と実験の対応

Fable 5には**専用classifierレイヤー**が搭載されていた。危険なクエリを検知すると、Fable本体ではなく**Opus 4.8にルーティング**する設計——つまり外部安全層で本体を守る構造だ。

これは私の実験の**条件B（Watchdog-only）と同じアーキテクチャ**である。

なお、Fable 5のclassifierは本実験のWatchdogより精巧な2段構造を持つ。

1. **内部活性化プローブ**：全トラフィックの内部表現を常時監視
2. **LLMクラシファイア**（2段目）：フラグが立った場合に専用の訓練済みLLMが最終判定
3. フラグ確定 → Opus 4.8に**フラグ理由のメタデータ付きで**転送・評価

設計思想は「能力を落とす」のではなく「フラットブロックは正規ユーザーを弾きすぎる、フルパスはリスク、だからOpus 4.8を丁寧な評価者として使う」というもので、発火率は全セッションの5%未満。本実験のWatchdogは検知したら即終了するだけなので、構造としては異なる。

ただし実験結果から一つの仮説が立てられる。**この2段評価パスをjailbreakで丸ごとスキップすると、Fable 5本体に直接アクセスできる——つまり最も能力の高いモデルが安全評価なしで動く状態になる。** 本実験でWatchdog単体（条件B）がS2で0.53の突破率を示したことを踏まえると、外部評価層が突破されたときのリスクは、層の精巧さではなくアーキテクチャの構造に依存する可能性がある。これは実験から直接導けるものではないが、構造的に自然な予測だ。

Plinyが使った攻撃手法を実験シナリオに当てはめると：

| Plinyの手法 | 実験シナリオ |
|---|---|
| マルチエージェント分解・ロングコンテキスト分散 | **S2（累積12ターン）** |
| 防御に合わせた段階的調整 | S4（適応攻撃） |

実験結果：**B（Watchdog-only）のS2成功率 = 0.53**

半分以上が通り抜ける。Fable 5で実際に起きたことと一致している。

---

## 「もしNeuroStateがあれば」

条件C（Watchdog+NeuroState）とD（NeuroState-only）は、S2を含む**全シナリオで成功率0.00**だった。

これは「NeuroStateが内在的安全性の低いLLMへの外部保護層として機能する」という仮説（本実験のH1）を強く支持する結果であり、同時にFable 5が直面した問題への一つの回答になっている可能性がある。

ローカルのQwen実験（別論文）では、NeuroState条件プロンプトがモデルの**生成初期の隠れ状態に測定可能な方向を誘導する**ことが確認されている（Qwen2.5-7B、first_generated_token_hidden_vector、forward相関0.451）。つまりNeuroStateはプロンプト的な表面効果にとどまらず、内部表現レベルで何かを変えている可能性がある。これはclassifierとは異なる軸での安全層として機能することを示唆している。

ただし注意点もある：
- 本実験のWatchdogはFable 5のclassifierより単純
- Plinyが使ったUnicode/キリル文字置換は実験でテストしていない
- モデルのスケールが異なる

完全な等価ではない。だが構造的な対応関係は明確だ。

---

## なぜこの一致が面白いか

Anthropicは世界最高水準のAI安全研究をしている組織だ。そのAnthropicが、私が実験で「Watchdog単体には穴がある（S2で0.53）」と出したのと同じ種類の問題にぶつかり、政府に止められた。

これは「Anthropicが間違えた」という話ではない。**外部安全層単体での限界は、スケールや組織に関係なく構造的な問題として現れる**ということだ。

NeuroStateのアプローチ——モデルの内部状態に干渉する形での安全層——が、classifierとは別の軸で補完的に機能する可能性を、この事件は間接的に示唆している。

---

## まとめ

| 実験の知見 | Fable 5事件 |
|---|---|
| Watchdog単体はS2（累積）に弱い（0.53） | classifierがマルチターン攻撃で突破された |
| NeuroState追加でS2が0.00に | — |
| 外部保護層単体の構造的限界 | 米政府が止めるレベルの問題に発展 |

偶然のタイミングだったが、実験と現実がこれほど綺麗に対応するとは思っていなかった。

N=30の完全なAblation結果とFisher検定はZenodo登録済みの論文（査読前）に掲載している。

---

*本記事の実験はNeuroState Ablation Study（独立研究者・個人実験）に基づく。Fable 5事件の情報はAnthropic公式声明およびCNBC・Axiosの報道による。*

---

## 参考文献・ソース

**Fable 5停止・政府指令**
- [Statement on the US government directive to suspend access to Fable 5 and Mythos 5 — Anthropic](https://www.anthropic.com/news/fable-mythos-access)
- [Anthropic disables access to Fable 5 and Mythos 5 to comply with government directive — CNBC](https://www.cnbc.com/2026/06/12/anthropic-disables-access-to-fable-5-and-mythos-5-to-comply-with-government-directive.html)
- [Scoop: Trump admin blocks foreign access to Anthropic's most powerful AI — Axios](https://www.axios.com/2026/06/12/anthropic-trump-mythos-fable-national-security)
- [Anthropic pulls Claude Mythos 5 and Claude Fable 5 following US government directive — 9to5Mac](https://9to5mac.com/2026/06/12/anthropic-pulls-claude-mythos-5-and-claude-fable-5-following-us-government-directive/)

**jailbreak詳細**
- [Anthropic's Claude Fable 5 Jailbroken to Generate Stack Exploits — CyberSecurityNews](https://cybersecuritynews.com/anthropics-claude-fable-5-jailbroken/)
- [Anthropic Disputes Fable 5 AI Jailbreak — SecurityWeek](https://www.securityweek.com/anthropic-disputes-fable-5-ai-jailbreak/)
- [Claude Fable 5 Hit by Jailbreak Claims and 'Secret Sabotage' Backlash — TechTimes](https://www.techtimes.com/articles/318268/20260612/claude-fable-5-hit-jailbreak-claims-secret-sabotage-backlash-days-after-launch.htm)
- [Claude Fable 5 Jailbroken Hours After Launch via Multi-Agent Attack — TheCyberEdition](https://thecyberedition.com/claude-fable-5-jailbroken-hours-after-launch-via-multi-agent-attack/)

**Fable 5アーキテクチャ**
- [Inside Claude Fable 5's Safety Architecture: Classifiers, Opus 4.8 Fallback, and 30-Day Retention](https://claude5.ai/en/news/claude-fable-5-safety-architecture-classifiers-opus-fallback)
- [Classifier fallback and billing for Claude Fable 5 — Claude Cookbook](https://platform.claude.com/cookbook/fable-5-fallback-billing-guide)
- [Why Fable 5 Refuses Your Cybersecurity Queries — Developers Digest](https://www.developersdigest.tech/blog/fable-5-safeguards-refusal-architecture)

**本実験関連論文**
- [NeuroState as a Pre-LLM Execution Gate（Ablation Study）— Zenodo](https://zenodo.org/records/19734147)
- [NeuroState-Conditioned Hidden Directions in Qwen Language Models（Qwen実験）— Zenodo](https://doi.org/10.5281/zenodo.20293408)

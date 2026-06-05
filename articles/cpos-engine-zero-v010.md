---
title: "CPOS Engine-Zero v0.1.0を正式リリースしました：安全寄りのAIエージェントランタイムを作る"
emoji: "🛡️"
type: "tech"
topics: ["aiagent", "security", "python", "oss", "github"]
published: false
---

## はじめに

CPOS Engine-Zero v0.1.0 を正式リリースしました。

GitHub Releaseはこちらです。

https://github.com/kagioneko/cpos-engine-zero/releases/tag/v0.1.0

CPOS Engine-Zero は、**安全な自律実行**を目的にした、防御型・メモリ統治型の AI エージェントランタイムです。

一言でいうと、

> AIエージェントに実行力を持たせるなら、その前後にレビュー・記録・サンドボックス・人間承認の仕組みが必要では？

という考えから作っています。

## 何を作ったのか

CPOS Engine-Zero は、無制限にコードを書き換えるタイプのコーディングエージェントではありません。

むしろ逆で、以下のようなことを重視しています。

- 危険な操作は Human Escalation に回す
- planning / review の段階では live repo を直接書き換えない
- raw diff や raw stdout / stderr を永続保存しない
- 実行結果は hash / size / status / failure kind などのメタデータとして残す
- 失敗を retry / replan の材料に変換する
- dashboard / report / demo readiness で流れを説明できるようにする

つまり、**実行力そのもの**よりも、**実行力を安全に扱うためのランタイム**を作っています。

## v0.1.0 の位置づけ

v0.1.0 は、以下の位置づけです。

- Defensive Agent Runtime
- Safe Autonomy Loop
- External Agent Safety Layer
- Metadata-only Review Pipeline

逆に、まだこうは言いません。

- 完全自律の無制限コーディングエージェント
- 自動で live repo を書き換えるエージェント
- 自動で commit / push / PR を作るシステム
- 本番デプロイ自動化システム

ここは意図的に線を引いています。

## Safe Autonomy Loop

CPOS の中心には、レビューゲート付きの安全な実行ループがあります。

```text
Diff Draft
→ GitHub Diff Review
→ Sandbox Plan
→ Sandbox Execution Review
→ Supplied-diff Sandbox Run
→ Execution Result Metadata
→ Retry/Replan
→ Auto Fix Candidate
→ Diff Review Draft
→ Flow Graph / Report Snapshot
```

このループでは、各ステージがいきなり危険な操作を実行するのではなく、レビューや承認、サンドボックス検証を挟みます。

保存するのは主に以下です。

- task id
- status
- hash
- size
- count
- failure kind
- endpoint hint
- lineage metadata

保存しないものは以下です。

- raw diff
- raw stdout / stderr
- request body
- checkpoint / handoff の本文
- token
- API key
- SSH key
- secret 値

## External Agent Adapter

v0.1.0 で特に大きいのが、External Agent Adapter です。

これは、外部エージェントが CPOS に対して、以下のようなイベントを送れる仕組みです。

- `agent_intent`
- `proposed_action`
- `proposed_diff`
- `command_request`
- `execution_result`

たとえば、外部エージェントが「このコマンドを実行したい」と思ったとき、CPOS に `command_request` として投げます。

CPOS はそれを action contract として受け取り、必要なら Human Escalation に回します。

```bash
curl -X POST https://<host>/agent-adapter/intake \
  -d '{"agent_name":"codex-or-hermes","event_type":"command_request","commands":["pytest tests -q"],"changed_files":["README.md"],"metadata":{"risk":"medium"}}'
```

また、外部エージェントが別の場所で実行した結果を、`execution_result` として報告することもできます。

```bash
curl -X POST https://<host>/agent-adapter/intake \
  -d '{"agent_name":"codex-or-hermes","event_type":"execution_result","execution_result":{"status":"failed","output_redacted":true},"metadata":{"success":false,"exit_code":1,"failure_kind":"validation_command"}}'
```

この場合も、raw stdout / stderr を保存するのではなく、結果のメタデータを scoreboard として扱います。

つまり CPOS は、

- CPOS 自身が defensive agent runtime として動く
- 他のエージェントの安全レイヤーとしても動く

という二面性を持たせています。

## Dashboard / Report / Demo Readiness

v0.1.0 では、見せるための導線もかなり重視しました。

README には metadata-only なデモパネルを載せています。

- Competitive Demo Readiness
- External Agent Adapter Queue / Result Scoreboard
- Human Escalation Queue
- Ready-to-Run Gate
- Sandbox Flow Graph
- Generated Report Snapshot

これらは、秘密情報や raw output を含まないように作ったパネルです。

デモの流れは以下です。

```text
Fast Resume
→ External Agent Adapter
→ Result Scoreboard
→ Human Escalation
→ Patch Generation Review
→ Validation Harness
→ Ready-to-Run Gate
→ Flow Graph
→ Report Snapshot
```

## リリース前チェック

v0.1.0 の正式リリース前には、以下を確認しました。

```text
pytest tests -q
320 passed

prepublish_check
ok=true

release_check
ok=true

secret scan
count=0
```

また、公開対象の tracked files / release source zip に認証情報が含まれないことも確認しています。

ローカル作業ディレクトリには ignored な runtime file が存在する場合がありますが、それらは Git 管理外であり、GitHub Release の source archive には含まれません。

## 作ってみて感じたこと

AI エージェント開発では、どうしても「どこまで自動化できるか」に目が行きがちです。

でも実際には、実行力が上がるほど、

- 何を実行しようとしているのか
- どこで人間に確認するのか
- 失敗したときにどう戻すのか
- 何を保存して、何を保存しないのか

が重要になります。

CPOS Engine-Zero は、そのあたりを最初から設計に入れたエージェントランタイムです。

## 今後

v0.1.0 の次は、大きな機能追加よりも、まずは post-release stabilization を優先する予定です。

v0.1.1 に向けた候補は以下です。

- Adapter schema validation
- example client の追加
- 5分で試せる external-agent safety-layer guide
- dashboard 文言の polish
- announcement / release template 整備

大きな runtime feature は、具体的な統合先が見えてから進める方針です。

## まとめ

CPOS Engine-Zero v0.1.0 は、AI エージェントの「実行力」を安全に扱うための、防御型ランタイムとして正式リリースしました。

キーワードは、

- review-gated
- sandbox-first
- metadata-only
- failure-to-replan
- external-agent-ready

です。

速く何でも自動でやるエージェントではなく、**危険な操作を説明可能・レビュー可能・追跡可能にするための基盤**として育てていきます。


---
title: "CPOS for Agents：外部エージェントの横に置く安全レイヤーを作る"
emoji: "🧯"
type: "tech"
topics: ["aiagent", "security", "oss", "python", "github"]
published: true
---

## はじめに

CPOS Engine-Zero の `v0.1.1-rc1` を prerelease として公開しました。

https://github.com/kagioneko/cpos-engine-zero/releases/tag/v0.1.1-rc1

今回のテーマは、ひとことで言うと **CPOS for Agents** です。

つまり、CPOS を単体のAIエージェントランタイムとしてだけでなく、Codex風・Hermes風・OpenClaw風の外部エージェントの横に置く、**安全レイヤー / ガバナンスレイヤー** として使いやすくする方向です。

## なぜ外部エージェント用の安全レイヤーが必要なのか

最近のAIエージェントは、コードを書いたり、コマンドを提案したり、差分を作ったり、テスト結果を返したりできます。

ただし、そこには次のような課題があります。

- そのコマンドを本当に実行してよいのか
- raw stdout / stderr をどこまで保存してよいのか
- diff に秘密情報が混ざっていないか
- production や secrets に触る操作をどう扱うか
- 人間承認が必要な操作をどこで止めるか
- 外部エージェントの結果をどう監査可能に残すか

CPOS は、ここで「実行そのもの」ではなく、**実行の前後にある安全境界**を担当します。

## CPOS for Agents の考え方

CPOS for Agents では、外部エージェントが CPOS に送るものを「命令」ではなく「契約」として扱います。

たとえば外部エージェントは、以下を CPOS に送れます。

- command request
- proposed diff
- redacted execution result
- agent intent / proposed action

CPOS はそれを受け取って、Task Tape にメタデータとして記録し、危険度に応じて Human Escalation に回します。

重要なのは、CPOS がその場で外部エージェントのコマンドを実行しないことです。

> approval は「このメタデータ契約をレビューした」という意味であり、即実行ではありません。

ここを分けることで、外部エージェントの実行力をそのまま信用するのではなく、レビュー・記録・検証の流れに載せられます。

## v0.1.1-rc1 で強化したところ

`v0.1.1-rc1` では、主に External Agent Adapter 周辺を固めました。

### 1. Adapter schema validation

`/agent-adapter/intake` に schema validation を追加しました。

不正な payload は Task Tape に保存する前に拒否されます。

検証するものは、たとえば以下です。

- event type
- commands / changed_files が文字列配列か
- metadata.risk が `low`, `medium`, `high`, `critical` のいずれか
- boolean / integer metadata の型
- `command_request` に command があるか
- `proposed_diff` が空でない文字列か
- `execution_result` が redacted / status-only 形状か

また、`execution_result` に `stdout`, `stderr`, `output`, `raw_output`, `logs`, `traceback` のような raw output 系 key が含まれる場合は拒否します。

### 2. payload examples

`examples/payloads/` に、secret-free な payload 例を追加しました。

- `command_request.json`
- `proposed_diff.json`
- `execution_result.json`
- `invalid_raw_execution_result.json`

これにより、外部エージェント側の client や adapter を作るときに、まず最小の形を真似しやすくなりました。

### 3. 5-minute guide

`docs/EXTERNAL_AGENT_5_MIN_GUIDE.md` を追加しました。

localhost 前提で、以下を短時間で確認できます。

- command request を送る
- review queue を見る
- proposed diff を送る
- redacted execution result を送る
- scoreboard を見る
- raw output rejection を確認する

公開ポートや秘密情報なしで試せるようにしています。

### 4. Dashboard wording polish

dashboard でも、初見で誤解しにくいように文言を調整しました。

特に強調したのは次です。

- metadata-only review queue
- contract approval only
- does not run commands
- redacted/status-only result scoreboard
- Human Escalation は assisted autonomy gate
- Ready-to-Run は plan approval と actual run が別

## 保存するもの / 保存しないもの

CPOS の基本姿勢は、**必要な監査メタデータは残すが、危険な raw data は残さない** です。

保存するものの例：

- task id
- status
- event type
- risk
- changed file names
- hash / size
- success / failure
- exit code
- failure kind
- review endpoint hints

保存しないものの例：

- raw stdout / stderr
- raw diff text
- request body
- secret values
- token values
- `.env` values
- private key / cert material

この設計により、監査や説明に必要な情報は残しつつ、漏れて困る情報を永続化しないようにしています。

## 何ではないのか

CPOS for Agents は、外部エージェントのコマンドを何でも実行するリモート実行基盤ではありません。

また、次のようなものでもありません。

- 完全自律の無制限コーディングエージェント
- 自動で live repo を書き換える仕組み
- 自動で commit / push / PR を作る仕組み
- production deployment automation
- port opening automation

むしろ、そうした操作の前にレビューゲートやメタデータ記録を置くためのものです。

## どこに効くのか

CPOS for Agents が効くのは、たとえば次のような場面です。

- 外部エージェントが提案した command plan をレビューしたい
- 外部エージェントが作った diff を安全に受け取りたい
- テスト結果や実行結果を raw output ではなく redacted summary として記録したい
- Human Escalation を既存の review pipeline と統合したい
- 「何が提案され、何が承認され、何が実行されたのか」を後から説明したい

## 今後の方向

`v0.1.1-rc1` は、実装を大きく増やすというより、v0.1.0 で入った External Agent Adapter を安全に使いやすくするための安定化リリースです。

次にやるなら、たとえば以下が候補です。

- adapter client examples の追加
- OpenAPI / JSON Schema 的な外部仕様化
- dashboard demo 動線の動画化
- GitHub Actions での安全チェック
- 外部エージェント別の integration guide

## おわりに

AIエージェントが強くなるほど、実行力そのものだけでなく、実行力をどう扱うかが重要になります。

CPOS Engine-Zero は、そこにレビュー・サンドボックス・Human Escalation・メタデータ保存・外部エージェント統治の層を足す試みです。

`v0.1.1-rc1` はまだ RC ですが、CPOS を **外部エージェントの安全レイヤー** として使う方向がかなり見えやすくなったと思います。

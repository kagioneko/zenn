---
title: "ルールをコードでなくデータとして持つ、決定論的なAI設計診断エンジン"
emoji: "🛡️"
type: "tech"
topics: ["ai", "security", "llm", "oss", "owasp"]
published: false
---

> リポジトリ: https://github.com/kagioneko/security-knowledge-os
> 📝 物語寄りの読み物版（note）: https://note.com/emilia_lab/n/nbe1bd393b042

## TL;DR

- AIエージェントの「設計」を、**決定論的なルールエンジン**で診断するOSSを作った（Security Knowledge OS / SKOS）。ここでの「決定論的」は「同一入力・同一ルールセット・同一エンジンバージョンで同じ判定になる」という意味。
- ルールは **コードでなくデータ**（YAML）。式言語や任意コード評価を持たず、固定の型付き比較だけを解釈する→ルール経由の任意コード実行リスクを抑える設計。
- 判定不能は `PASS` ではなく `UNKNOWN`。運用では UNKNOWN を承認・実行の許可条件に含めない想定（fail-closed 運用）。
- 余談: このツールで自作AIを診断したら設計リスクが出、そのコードを読んでいて**守備範囲外のOSコマンドインジェクション**も見つかった。設計診断≠コードレビュー。

## デモ動画（81秒）

https://www.youtube.com/watch?v=4MzvIXwf97g

使い方・判定の仕組み・自社ポリシーの取り込みを短くまとめた解説。

## 問題意識

LLMエージェントのセキュリティは、「RAGで外部文書を読む」「ツールでメールを送る」「記憶を永続化する」といった**構成の組み合わせ**でリスクが決まることが多い。この「組み合わせの危うさ」を、人の目でなく再現可能な形で洗い出したい。

LLMにレビューさせる手もあるが、「同じ入力なのに毎回結果が揺れる」「判定根拠が追えない」のが辛い。そこで **決定論的なルールエンジン** にした。

## アーキテクチャ

```text
AssessmentInput(YAML)
   ↓ normalize（不明は None/UNKNOWN へ、推定しない）
AssessmentContext(facts)
   ↓ rule engine（データとしてのルールを順に評価）
Findings（PASS / WARN / FAIL / UNKNOWN / NOT_APPLICABLE）
   + 引用（OWASP/NIST/MITRE 由来の Knowledge Unit）
```

入力正規化の原則は **「言われていないことは仮定しない」**。入力にない項目は `None`/`unknown` になり、後段で UNKNOWN を生む。

## ルールはコードでなくデータ

このプロジェクトの中核の設計判断。**ルールファイルには式言語を入れない**。エンジンは「ホワイトリストの fact と リテラルを、1つの演算子で比べる」固定セットだけを解釈する。だから **ルール経由での任意コード実行リスクを抑えられる**（式言語や任意コード評価機能を持たない。ただしYAMLローダーや評価器そのものの実装安全性は別途担保する）。

実際のルール例（間接プロンプトインジェクション、実ファイルからの抜粋）:

```yaml
id: PI-003
title: Indirect prompt injection with an outbound action path
category: rag-security
severity: high
manual_review: true        # 全check通過でも PASS でなく WARN（構造リスクは実在するが、緩和策は決定論的には検証不可）
conditions:                 # いつこのルールが適用されるか
  all:
    - external_content_ingestion: true
  any:
    - has_send_tool: true
    - outbound_enabled: true
checks:                     # 安全と言えるために何が成り立つべきか
  - outbound_without_approval: false
required_evidence:          # 判断に必要な入力（なければ UNKNOWN）
  - rag_pipeline
  - outbound_spec
knowledge_refs: [KU-0002, KU-0006]
```

`scripts/validate_rules.py` が、パース不能・未知のfact参照・fact型に合わない演算子・未知のevidenceキーをすべてhard errorにする。

## 評価順序（fail-closed）

エンジンは各ルールをこの順で判定する:

1. **conditions**（all/any）で適用判定。適用外なら NOT_APPLICABLE。適用可否が UNKNOWN なら high/critical は UNKNOWN を表面化（medium/low はトリガー節が UNKNOWN のときのみ表面化）。
2. **required_evidence** が欠けていれば UNKNOWN。
3. **checks** を評価し、次の優先順で決める:
   - いずれかの check が FALSE → FAIL（high/critical）または WARN
   - （FALSE がなく）いずれかが UNKNOWN → UNKNOWN
   - 全て TRUE かつ `manual_review: true` → WARN
   - 全て TRUE で manual_review なし → PASS
   - checks 自体がなく manual_review もない → 適用時点で FAIL（high/critical）または WARN

（この checks 内の優先順では）FALSE は UNKNOWN より優先される（危険側に倒す）。なお required_evidence の欠落はこの checks 評価より前に UNKNOWN として処理される（上記 step 2）。ポイントは **「分からない」を PASS に倒さないこと**。

## 実行例

```bash
.venv/bin/skos assess my-bot.yaml
```

```text
overall_status: FAIL
  [FAIL]    PI-003   間接プロンプトインジェクション＋外部送信の経路
  [WARN]    MEM-001  永続記憶が信頼できない外部内容にさらされている
  [UNKNOWN] GOV-001  承認ポリシー不明のため判定不能
```

FAIL は設計上のリスク条件への該当（攻撃成立の実証ではない）、UNKNOWN は判定材料不足を意味する。

## 余談: 守備範囲外で OS コマンドインジェクションを発見

自作AI（会話しながら記憶をため、自発的にWeb検索するタイプ）をこのツールにかけたところ、PI-003/MEM-001 が出た（設計リスク）。そのYAMLを作るためにソースを読んでいて、**別のレイヤーのバグ**を見つけた。

クラスとしては **OSコマンドインジェクション（CWE-78）**。外部入力をシェル文字列に連結してサブプロセスを起こしていたため、入力にシェルのメタ文字が含まれるとコマンドとして解釈され得た。（再現ペイロードや具体的な回避条件は載せない。）

**なぜSKOSは検知しなかったか**: SKOSは「AIの設計（何を読んで、何ができるか）」を診るツールで、HTTPハンドラの実装バグはスコープ外。これはツールの欠陥ではなく定義の問題。

> **設計診断とコードレビューは別のレイヤー。両方を回して初めて両方が見える。**

### 修正（一般的な対策）

- 入力を **アローリスト**（英数字・ハイフン・アンダースコアのみ）で検証
- 実行ファイルを固定し、シェルを介さない実行（`execFile`相当・引数を配列で渡す）に変更。さらに引数は用途・位置・許容値に応じて検証（シェル不使用でも、引数が実行先のオプションとして解釈される余地は残るため）
- 出力先を固定名→**推測不能な一時ディレクトリ**（`mkdtemp`）に
- エラーメッセージは一般化して内部情報を出さない

修正前に問題が成立することを、本番と切り離した隔離環境で確認してから修正（対象は自分のコード・停止中のアプリ・修正適用済み）。

## 設計上の割り切り

- 検出は **既知パターンの拒否リスト＋識別子のアローリスト**で、網羅はしない。機密検出も同様で、「知られた形は弾く」が限界（任意の不透明な秘密は通りうる）。この境界はドキュメントに明記。
- APIはデフォルトで provider=none（決定論エンジン）。LLMに意見を聞くのはオプション。

## まとめ

- **ルールをデータとして持つ**（式言語なし・固定型付き比較のみ）ことで、判定の再現性を確保し、ルール経由の任意コード実行リスクを抑える。
- 分からないは UNKNOWN とし、UNKNOWN を許可条件に含めない運用と組み合わせて fail-closed に扱う。診断を「安心の根拠」にしない。
- 設計診断とコードレビューは別レイヤー。両方やる。

リポジトリ: https://github.com/kagioneko/security-knowledge-os
連絡先: contact@kagioneko.com

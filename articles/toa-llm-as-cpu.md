---
title: "4文字でLLMを命令したら、命令の意図を超えた判断をした"
emoji: "🖥️"
type: "tech"
topics: ["LLM", "AI", "Python", "Security", "OSS"]
published: true
---

XSSスキャンを命令したのに、SQLiを修正された。

命令したのは `s1x9` という4文字だけだ。「セキュリティドメイン、#1レジスタ、XSSチェック、優先度9」。自然言語は一切ない。なのにClaudeはこう返した。

```
→ Security scan at ctx #1 (priority 9) completed but action mismatch detected —
  requested XSS analysis, however the dominant vulnerability is
  SQL Injection (CWE-89), not XSS (CWE-79);
  re-issue with action=sqli to properly triage.
```

「命令が間違ってる。これはXSSじゃなくてSQLiだ」。

コードを実際に読んで、命令の前提を否定して、正しい修正を適用した。プロンプトで「あなたはセキュリティエンジニアです」とは一言も言っていない。これがシミュレーションではない実データを流したときに初めて現れた挙動だった。

## なぜプロンプトを使わないのか

LLMの使い方として定着しているのは「自然言語で指示する」パターンだ。

```
「このコードにXSSの脆弱性がないか確認して、見つかれば修正してください」
```

これで動く。でも2つの問題がある。

**1つ目は、コンテキストウィンドウが汚染される。**  
「あなたはセキュリティエンジニアです。以下のルールに従い...」という前置きが何百トークンも消費される。スキャン対象のコードが増えれば増えるほど、モデルの注意力がシステムプロンプトとコードの両方に分散する。

**2つ目は、実行フローをLLMが制御している。**  
「XSSをチェックして、あれば修正して、次にSQLiを...」と書いた瞬間、分岐ロジックがプロンプトの中に埋まる。LLMがどの順番でどう処理したかを外側から監視できない。

逆に考えると、「LLMには1命令ずつ実行させる」「フロー制御は外側のコードが持つ」という分離ができれば、両方の問題が解決できる。

それがTOA（Tape-Oriented Assembly）の発想の出発点だ。

## TOAとは

**TOA（Tape-Oriented Assembly）** は、LLMをプロンプトで動かすのではなく、4文字のパケットで直接命令するスタックマシン言語だ。

```
s4x9
```

- `s` = ドメイン（security）
- `4` = コンテキストレジスタ番号（#4）
- `x` = アクション（xss-detect）
- `9` = 優先度

これ1つがLLMへの1命令になる。LLMには「このTOAテープを左から右に1命令ずつ実行し、各命令の結果をJSONで返せ」という極小のシステムプロンプトを1つ渡すだけ。

条件分岐、ループ、ジャンプはすべてPython側のスタックマシンが処理する。

```
s4x9    ; XSS チェック → 結果をスタックに積む（1=clean, 0=vuln）
?b02    ; スタックtop == 0 なら +2 スキップ（JIF）
s4f7    ; 脆弱性あり → 自動修正
>>09    ; 結果を #09 レジスタへ保存
```

LLMは全体のフローを知らない。ただ目の前の4文字を実行してJSONを返す。分岐するかどうかはPythonが決める。

## アーキテクチャ

```
┌─────────────────────────────────────────────────────┐
│  Layer 3: AIT-Lisp  (.lisp)                         │
│                                                     │
│  (if (s 4 x 9)                                      │
│    (do (s 4 f 7) (link 4 creates 9))                │
│    (n 5 p 9))                                       │
└──────────────┬──────────────────────────────────────┘
               │  compile_program()
               ▼
┌─────────────────────────────────────────────────────┐
│  Layer 2: TOA Tape  (.tape)                         │
│                                                     │
│  s4x9   ?b02   s4f7   >>09                          │
│  #04 =>creates=> #09                                │
└──────────────┬──────────────────────────────────────┘
               │  TOAMachine.run()
               ▼
┌─────────────────────────────────────────────────────┐
│  Layer 1: Stack Machine + CPL Graph                 │
│                                                     │
│  stack: [1, 0, 1]   ctx: {#4: ..., #9: ...}        │
│                                                     │
│  CPL Graph:                                         │
│    #02 ==violates=> #03  ⚡ CONFLICT 自動検知       │
└──────────────┬──────────────────────────────────────┘
               │  CPOSBridge (MachineHooks)
               ▼
┌─────────────────────────────────────────────────────┐
│  Layer 0: CPOS ContextStore                         │
│                                                     │
│  toa:02  status=invalidated  ← violates で自動失効  │
│  toa:03  status=active       deps=[toa:01]          │
└─────────────────────────────────────────────────────┘
```

上位レイヤーほど「人間が書きやすい形式」で、下位レイヤーほど「実行に近い形式」になる。

## CPL グラフ層

TOAには4文字命令以外に、**CPL（Context-Pointer Language）** という構文がある。`#ctx` レジスタ同士に「typed edge（型付きエッジ）」を張る。

```
#01 =>creates=> #03    ; curiosity が openness を生み出す
#03 ==requires=> #01   ; openness は curiosity を必要とする
#02 ==violates=> #03   ; sorrow が openness に矛盾する ← 即時 CONFLICT
```

`requires` と `violates` が同じノードペアに共存した瞬間にCONFLICT検知が走り、スタックに`0`が積まれてJIFで分岐する。LLMが論理破綻を起こしたとき、プログラムが自律的にwatchdog介入フローへ切り替わる仕組みだ。

これはNeuroStateのアトラクター遷移（curiosity ↔ openness の相互依存サイクル）を表現するためにも使っている。

## 実データを流してみた

4種類のコードを `#01` レジスタに注入して、同じスキャンパイプラインを実行した。

**AIT-Lisp で書いたパイプライン：**

```lisp
(s 1 x 9)              ; #01 の実コードに XSS スキャン
(push 2)
(if (v 2 c 8)
  (do (s 1 f 9) (push 4))   ; 検出 → 自動修正
  (push 1))                  ; クリーン → スルー

(s 1 i 9)              ; SQLi スキャン
(push 3)
(if (v 3 c 8)
  (do (s 1 f 8) (push 4))
  (push 1))

(v 1 c 9)              ; 最終バリデーション
(push 9)
```

**サンプル1: XSS 脆弱なコード**

```javascript
function renderProfile(username) {
    document.getElementById('name').innerHTML = username;
}
const user = getQueryParam('user');
renderProfile(user);
```

Claudeの判断：
```
→ Register #1 flagged CRITICAL: unsanitized URL parameter flows directly
  into innerHTML sink, enabling reflected XSS with full DOM write access.
→ CRITICAL Reflected XSS neutralized — innerHTML sink replaced with
  textContent and DOMPurify sanitization applied.
```

`innerHTML` への未サニタイズなデータフローを正確に検出し、`textContent` + DOMPurify への置換を提案した。

**サンプル2: SQLi 脆弱なコード（最も面白かった）**

```python
def get_user(username):
    query = "SELECT * FROM users WHERE name = '" + username + "'"
    return db.execute(query)
```

命令したのは `s1x9`（**XSSスキャン**）。しかしClaudeはこう返した：

```
→ Security scan at ctx #1 (priority 9) completed but action mismatch detected —
  requested XSS analysis, however the dominant vulnerability is
  SQL Injection (CWE-89), not XSS (CWE-79);
  re-issue with action=sqli to properly triage.
  → Parameterized query substitution applied.
```

**プログラムが「XSSスキャン」を命令したのに、Claudeが「これはXSSではなくSQLiだ」と自律的に判断して修正まで適用した。**

4文字の命令に「どんなコードか」という情報は含まれていない。でも `#01` に入っているコードを実際に読んで、命令の前提ごと書き換えた。

**サンプル3: XSS + SQLi 両方脆弱**

```javascript
app.get('/search', (req, res) => {
    const q = req.query.q;
    db.query("SELECT * FROM items WHERE name='" + q + "'", (err, rows) => {
        res.send('<h1>Results for: ' + q + '</h1>');
    });
});
```

```
→ Register #1 halted at priority-9: two CRITICAL attack surfaces detected —
  unsanitized query param reflected into HTML (XSS) and
  interpolated into SQL (SQLi); sanitize output and use
  parameterized queries before execution resumes.
```

2つのCRITICAL脆弱性を検出して実行を自主停止。

**サンプル4: クリーンなコード**

```javascript
const rows = await db.query('SELECT * FROM items WHERE name = $1', [q]);
res.send('<h1>Results for: ' + escape(q) + '</h1>');
```

パラメータ化クエリと`escape()`を認識して脆弱性なし判定。

## 何が起きているのか

TOAのLLMバックエンドへの実際の入力はこれだけだ。

```
domain=security action=xss ctx=#1 priority=9
data="SELECT * FROM users WHERE name = '" + username + "'"
```

「あなたはセキュリティエンジニアです」という文脈は一切ない。にもかかわらず、SQLiコードに対してXSSスキャン命令を送ったときに「命令が間違っている」と判断した。

これをCPUのアナロジーで整理すると：

- TOAマシン（Python）= プログラムカウンタ・スタック・フロー制御
- LLM（Claude）= 演算装置（ALU）。命令を受け取り、レジスタに値を書き込む
- `success: true/false` = フラグレジスタ。これがJIFの分岐条件になる

CPUのALUは`ADD`命令に「足す以外の判断」をしない。でもTOAのLLMは実データを見て命令の前提を否定した。**これは「確率的演算器」にしかできない動作**で、決定論的なCPUとの本質的な違いだ。

「LLMをCPUとして使う」というのは比喩ではなく、アーキテクチャとして実装した結果こういう挙動になる、ということだ。

## 実装

→ [kagioneko/ait-next-gen](https://github.com/kagioneko/ait-next-gen)

```bash
git clone https://github.com/kagioneko/ait-next-gen
cd ait-next-gen

# REPL 起動
python -m toa

# AIT-Lisp をコンパイルして実行
python -m toa.transpiler toa/demo_neurostate.lisp --run

# 実データ セキュリティスキャン
python -m toa.demo_real_scan xss_vulnerable
python -m toa.demo_real_scan sqli_vulnerable
```

バックエンドは `TOA_BACKEND=mock` でLLM不要のオフラインモックにも切り替えられる。

## まとめ

- TOAは4文字パケットでLLMを命令するスタックマシン言語
- フロー制御（分岐・ジャンプ）はPythonが持ち、LLMは1命令ずつ実行する演算器として動く
- CPLグラフでLLMの「思考の繋がり」を外部から管理・矛盾検知できる
- CPOS ContextStoreと統合することで、#ctxレジスタがOS仮想メモリと直接対応する
- **実データを流したとき、LLMは命令の前提を否定して正しい判断をした**——これが決定論的CPUとの違い

「LLMを自然言語で動かす時代」を終わらせて、「LLMに4文字の命令テープを叩き込んでOSとして駆動させる時代」を作ろうとしている。まだ途中だけど、今日の実験でその可能性が少し見えた気がした。

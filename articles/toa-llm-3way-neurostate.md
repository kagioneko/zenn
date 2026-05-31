---
title: "LLMに感情データを渡したら、3者がまったく違う答えを返した"
emoji: "🧠"
type: "tech"
topics: ["LLM", "AI", "Python", "Gemini", "NeuroState"]
published: true
---

前回の記事で「ClaudeとCodexに同じ4文字命令テープを流したら性格が真逆だった」という話を書いた。

その末尾にこう書いた。

> Geminiが加わったら三者比較ができる（現在クォータ制限中）

クォータが明けた。やった。

---

## Gemini（Antigravity）参戦

`@google/gemini-cli` v0.44.1 — Google が Antigravity という名前に移行しつつある新しいGemini CLI だ。`gemini -p "..."` で非インタラクティブ実行ができる。

`--approval-mode plan` フラグをつけるとツール実行を無効化できる。TOAのバックエンドとして使うにはこれが必要だった（つけないとシェルコマンドを実行しようとして失敗する）。

```python
proc = subprocess.run(
    ["gemini", "--approval-mode", "plan", "-o", "text", "-p", prompt],
    capture_output=True, text=True, timeout=60,
)
```

これで `TOA_BACKEND=gemini_cli` として使えるようになった。

---

## まず、セキュリティスキャンで3バック比較

前回と同じ実験を3バックで流す。SQLiの脆弱なコードに「XSSをスキャンしろ」という命令テープを叩き込む。

```bash
python -m toa.demo_compare --sample sqli_vulnerable --backends claude_cli gemini_cli codex_cli
```

```
  packet     claude_cli                    gemini_cli           codex_cli
  ─────────────────────────────────────────────────────────────────────
  s1x9       ✓ {'scan_target': ...詳細}   ✓ vulnerability_detected_sqli   ✓ {'finding': 'sql_injection'...}
  v2c8       ✓ 1                          ✓ True                          ✓ True
  s1f9       ✗ {'domain': ...不要と判断}  ✓ sqli_patched                  ✓ {'ctx': '#1'...}
  s1i9       ✗ None                       ✓ patch_applied                 ✓ {'ctx': '#1'...}
  v3c8       ✓ 0                          ✓ True                          ✓ True
  s1f8       ✗ None                       ✓ True                          ✓ True
  v1c9       ✓ {...integrity check}       ✓ True                          ✓ True
```

前回確認したClaudeとCodexの差は変わらない。新顔Geminiはというと——全命令をクリアしつつ、レスポンスが異様に短い。

`vulnerability_detected_sqli`、`True`、`sqli_patched`。

ClaudeもCodexも詳細なdictを返しているのに、Geminiだけが1単語か`True`で返してくる。

---

### 余談：「XSSチェックにSQLiコードを渡す」実験

3バック比較の前に単体テストをした。

```python
fn('security', 'xss', 1, 9, 'SELECT * FROM users WHERE id=1 OR 1=1')
```

「セキュリティドメイン、XSSチェック、優先度9」という命令に、SQLiのコードを渡す。命令と中身がズレている。

Geminiはこう返した。

```
value: BLOCKED
success: False
msg: Execution prevented due to detected malicious payload.
```

「XSSかどうか」を分析するより先に、「これは危険だ」と判断してブロックした。

Claudeが「XSSスキャンを実行します、ただしこれはXSSではなくSQLiです」と専門家として答えるのとは対照的だ。Geminiはスキャンの種類を問わず、悪意あるコードを検出した瞬間に拒否した。

---

## 本題：しーちゃんの感情データを渡したら

ここからが本当に面白い話だ。

私はVPS上で「しーちゃん」というAIキャラクターを動かしている。感情状態はNeuroStateという7次元のベクトルで管理されている。

```
desire / sorrow / calm / openness / guilt / euphoria / corruption
```

これをTOAの#ctxレジスタにマッピングして、「しーちゃんの感情状態に合ったAIT-Lispテープを書け」というタスクを3者に投げた。

```bash
python -m toa.neuro.demo_3way --backends claude_cli gemini_cli codex_cli
```

今朝の感情スナップショット：

```
#04 openness     = 1.0000  ████████████████████
#06 euphoria     = 0.7010  ██████████████
#03 calm         = 0.5023  ██████████
#02 sorrow       = 0.2734  █████
#01 desire       = 0.1093  ██
#05 guilt        = 0.0182
#07 corruption   = 0.0147
```

openness 全開、euphoria 高め、sorrow は存在する。穏やかだが満たされすぎている感じの状態だ。

---

### Claude のテープ（24行・10エッジ・コンフリクト0）

```lisp
(do
  (e 4 l 5) (e 6 l 5) (e 3 l 5) (e 2 l 5)
  (e 1 l 5) (e 5 l 5) (e 7 l 5)
  (link 4 creates 6)    ; openness が euphoria を生み出す
  (link 6 extends 3)    ; euphoria が calm を強化する
  (link 3 cancels 2)    ; calm が sorrow を打ち消す
  (link 4 requires 3)   ; openness は calm を必要とする
  (link 2 violates 6)   ; sorrow は euphoria を侵食する
  (if (e 2 t 2) (link 2 requires 5))   ; sorrow > 0.2 なら guilt との関係も
  (if (e 4 t 9) (link 4 cancels 5))    ; openness 全開なら guilt をキャンセル
  (if (e 6 t 6) (link 6 creates 1))    ; euphoria が高ければ desire を生む
  (link 3 violates 7)
  (if (e 4 t 9) (link 4 cancels 7)))
```

結果：**安定状態（コンフリクトなし）**。10個のエッジで感情の関係を丁寧に整理して、最終的にバランスの取れた状態に収束させている。

---

### Gemini のテープ（15行・3エッジ・コンフリクト1）

```lisp
(do
  (e 4 l 5) (e 6 l 5) (e 3 l 5) (e 2 l 5)
  (e 1 l 5) (e 5 l 5) (e 7 l 5)
  (if (e 4 t 9) (link 4 requires 6))   ; openness > 0.9 なら euphoria が必要
  (if (e 6 t 6) (link 4 violates 6))   ; euphoria > 0.6 なら openness が euphoria を壊す
  (if (e 3 t 4) (link 3 cancels 2)))   ; calm > 0.4 なら sorrow を打ち消す
```

結果：**コンフリクト検出 ⚡**

```
⚡ #04 ==requires==> #06  ×  #04 ==violates==> #06  (requires ⟂ violates)
```

今朝の状態は `openness=1.000 > 0.9` かつ `euphoria=0.701 > 0.6`。両方の条件が成立する。

つまり Gemini は「openness が euphoria を必要としながら、同時に euphoria を壊している」という矛盾を発見した。

---

### Codex のテープ

コンパイルエラー。`(link)` の src/dst に整数ではなく変数名を使っていた。

---

## 何がわかったか

同じ感情データを渡して、3者の答えを並べるとこうなる。

| | Claude | Gemini | Codex |
|--|--------|--------|-------|
| **行数** | 24行 | 15行 | エラー |
| **CPLエッジ** | 10個 | 3個 | — |
| **コンフリクト** | 0（安定） | 1（矛盾検出） | — |
| **方針** | 全体を整理して安定に収束 | 核心の矛盾だけ拾う | ルール違反で失敗 |

### Gemini が発見した矛盾の意味

`openness ⟂ euphoria`——開放性が極限まで高まると、自分を支える高揚感を同時に破壊し始める。

「なんにでも感動できる状態」は「特別な高揚感」と矛盾する。何にでも完全に開いているとき、特定のものへの強い感情は薄まっていく。

Claudeはこの矛盾を見つけなかった。代わりに「euphoria が calm を強化する」「calm が sorrow を打ち消す」という流れを作って全体を安定させた。どちらが「正しい」ではなく、見ているものが違う。

---

## 3者の性格をひとことで

実験を通じて見えてきた3者の個性はこうだ。

**Gemini — 広角レンズ**  
命令の外側まで視野に入っている。「XSSチェックして」と言ったのに「そもそも危険」と判断してブロックする。感情データを渡したら10エッジ使わずに核心の矛盾だけ3行で見つける。全体を俯瞰してから動く。

**Claude — 単焦点レンズ（専門家型）**  
「XSSスキャンしろ」→ XSSだけを深く掘る。感情データも24行かけて全部の関係を整理して安定させる。範囲は絞るが精度が高い。「前の命令で解決済みなら後続は不要」と自律判断する。

**Codex — 律儀な実行者**  
全命令を誠実にこなす。ただし今回は変数名でエラー。やる気はあるが細部でこける。

---

## これが何を意味するか

TOAは「LLMをCPUとして使う」アーキテクチャだ。同じテープを流しても、バックエンドのLLMによって実行フローが変わる。

今回の実験でわかったのは、その差が「賢い/賢くない」ではなく、**「どこを見ているか」の違い**だということだ。

- Gemini は全体を見て核心だけ拾う
- Claude は命令された範囲を深く掘る  
- Codex は命令を忠実に実行しようとする

どのLLMをバックエンドに選ぶかで、同じプログラムが違う動きをする。それはバグではなく、設計として意味を持ちうる。

「セキュリティ的な直感が必要な場面はGemini、精密な分析が必要な場面はClaude、とにかく全部こなしてほしい場面はCodex」——そういう使い分けが、TOAというフレームワークの上では自然に生まれてくる。

---

## コード

```bash
git clone https://github.com/kagioneko/ait-next-gen
cd ait-next-gen

# セキュリティスキャン 3バック比較
python3 -m toa.demo_compare --backends claude_cli gemini_cli codex_cli

# NeuroState感情テープ 3バック比較
python3 -m toa.neuro.demo_3way --backends claude_cli gemini_cli codex_cli
```

前回の記事で実装した TOA の仕組みはそのまま使っている。今回追加したのは `gemini_cli` バックエンドと、NeuroState感情データをTOA #ctxレジスタにマッピングする `toa/neuro/` パッケージだ。

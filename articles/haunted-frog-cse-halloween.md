---
title: "🐸にハロウィンを教えたら、ターミナルが少しホラーになった ― The Haunted Frog"
emoji: "🎃"
type: "tech"
topics: ["python", "machinelearning", "llm", "hacktoberfest", "oss"]
published: true
---

以前、[🐸を1匹飼うと、言語モデルの中身が見える ― pip install cse-frog](https://note.com/emilia_lab/n/ne9082f0193a7) という記事で、学習用の小さな言語モデル **cse-frog** を紹介しました。

今回はその 🐸 に**ハロウィンの言葉を覚えさせて、ターミナルに住まわせて**みました。
海外の開発者コミュニティ DEV のハッカソン「Hacktoberfest Open-Source AI Challenge」（Week 4 のテーマは *Ghost in the Machine*＝機械の中の幽霊）に出した作品です。

- GitHub: https://github.com/kagioneko/the-haunted-frog
- DEV の記事（英語）: https://dev.to/sakiha6720/i-taught-a-tiny-open-source-language-model-to-speak-halloween-meet-the-haunted-frog-1phf

![ターミナルで🐸がしゃべる様子](https://raw.githubusercontent.com/kagioneko/the-haunted-frog/main/docs/media/haunt.gif)

## 🐸がしゃべったこと

```text
🐸 THE GHOST IS LAUGHING
🐸 THE NIGHT IS HUNGRY
🐸 THE OWL SAYS WHO
🐸 THE GHOST
🐸 THE ZOMBIE IS FLICKERING
🐸 THE FOG IS HUNGRY
```

これは台本ではありません。🐸 が生成した6行を、選ばずに続けて並べたものです（seed 1003〜1008）。
覚えさせたのは「THE FOG IS ROLLING IN」「THE WITCH IS LAUGHING」「I AM HUNGRY」のような短いフレーズ60行だけです。
「THE OWL SAYS WHO」は覚えさせた行そのままですが、「ゾンビがちらついている（THE ZOMBIE IS FLICKERING）」や「霧が腹ペコ（THE FOG IS HUNGRY）」は、どこにも書いていません。覚えた言葉の組み替えで出てきた行です。

画面に出る文章は2種類あって、見た目で区別できるようにしています。

- **🐸 の行**（緑色）: 🐸 の出力そのものです。横に seed が出るので、同じモデル・設定・プロンプトで同じ seed を指定すれば、同じ行を再現できます。
- **薄い文字の行**（`...the cursor blinks on its own...`＝「カーソルが勝手に点滅している」など）: 私が書いた演出用の文章です。🐸 は書いていません。

## 作り方

1. ハロウィンのフレーズを60行書く。`THE ... IS ...` のような共通部分をわざと作って、🐸 が組み替えられる「関節」を用意しました。
2. cse-frog に覚えさせる: `Frog(top_k_edges=16).learn(行ごとのリスト, epochs=5)`。学習は CPU で1秒もかかりません（手元の計測で0.3秒）。
3. `frog.generate(...)` で1行ずつしゃべらせて、緑色・タイプライター風に表示します。

**最初にハマったところ**: 60行を「文字列のリスト」のまま渡したら、cse-frog はそれを**1本の系列**として扱いました（1要素＝1記号）。🐸 は60行を1本の長い文として覚えてしまい、こう言い出しました。

```text
HAPPY HALLOWEENTRICK OR TREATGHOST IN THE MACHINETHE GHOST IS HEREI AM A FROG
```

行ごとのリスト（リストのリスト）で渡すと直ります。「行の終わり」も覚えてくれるようになりました。今はテストで確認しています。

## 実験してみた

雰囲気で終わらせず、`python -m haunted_frog evaluate` で10匹の 🐸 を学習させ、全員に**同じ100個の seed** でしゃべらせて比べました。

- **ハロウィン語の割合**: 生成された全単語のうち、あらかじめ決めた30語のハロウィン語リストに入っている単語の割合
- **出力のバリエーション**: 異なる出力の数 ÷ 空でない出力の数
- 特に書いていない学習済みの 🐸 は `top_k_edges=16` です。対照と既定設定（k=3）の行は単語単位です。

| 🐸 | ハロウィン語の割合 | 出力のバリエーション | 平均の長さ |
|---|---|---|---|
| 何も覚えていない 🐸 | 何も言わない（100% 無言） | — | 0 |
| 普通の言葉を覚えた 🐸（対照） | 0% | 59% | 3.6語 |
| ハロウィン・単語単位・1周 | 18% | 81% | 4.2語 |
| ハロウィン・単語単位・5周 | 15% | 57% | 3.7語 |
| ハロウィン・単語単位・20周 | 19% | 38% | 3.0語 |
| ハロウィン・5周・cse-frog 既定設定（top_k_edges=3） | 15% | 20% | 3.1語 |
| ハロウィン・**文字単位**・5周 | **0%** | 92% | 9.4語 |

### うまくいったこと

- **何を覚えたかで、何を言うかが変わる。** 対照の 🐸 はハロウィン語を一度も言わず、ハロウィンの 🐸 は5〜7語に1語くらいハロウィン語を使いました。
- **丸暗記ではなく組み替えている。** ハロウィン・単語単位（k=16）の出力の83〜94%は、覚えさせた行と完全には一致しない行でした（ただし、文法的に正しいとは限りません）。

### うまくいかなかったこと

- **THE が好きすぎる。** 覚えさせた行のうち THE で始まるのは38%なのに、5周・20周の 🐸 は出力の99%が THE で始まりました（1周だと49%）。
- **学習しすぎると単調になる。** 出力のバリエーションは 81%（1周）→ 38%（20周）に減りました。
  🐸 に確率を聞いてみると理由がわかりました。行の最初が THE になる確率は、1周で 0.435、20周で **0.998** です。強いつながりがどんどん強くなるのに温度は一定なので、確率が THE に張り付きます。
- **既定の設定だと狭すぎた。** cse-frog の既定値 `top_k_edges=3` では、バリエーションが 20% しかありませんでした。seed 0〜39 の出力を見て 16 に上げ、評価は別の seed 1000〜1099 でやっています（選んだときの出力で採点しないため）。
- **文字単位の 🐸 は、言葉にならない。** 5周・20周では、ハロウィン語は一度も出ませんでした（1周で1回だけ）。

  ```text
  THE INGHE INGHE IN IN IN WIS INDYS ING     (5周)
  THE IN IN IN ING                           (5周)
  HE S L THE AUNIGIC                         (1周)
  ```

  見えている文脈が直前の2文字くらいしかないので、THE・IN・IS・〜ING のループから抜け出せない、というのが私の仮説です（文脈の長さを変える実験はまだしていないので、確かめてはいません）。

正直、この **IN IN ING が延々と続くところが一番ホラーでした**。失敗のほうが幽霊っぽいとは思いませんでした。

## なぜ「中身が見える」ことが大事か

大きなクローズドなモデルだと、幽霊が何か言っているのを**眺めるだけ**です。
🐸 は小さくて中身が全部見えるので、「なぜそう言ったのか」まで追いかけられました。THE が 0.998 に張り付いていることも、GHOST の点数のほとんどが1本の直接のつながりから来ていることも、`frog.show(...)` と `scripts/diagnose.py` で数字として見られます。

それに、誰でも同じことを確かめられます。コーパスはただのテキストファイル、seed は画面に出ていて、モデルの保存形式も JSON です。表の数字は `python -m haunted_frog evaluate`（CPU 1コアで約1分、環境によります）で、THE の確率や点数の内訳は `python scripts/diagnose.py` で出てきます。

## 試すには

```bash
git clone https://github.com/kagioneko/the-haunted-frog && cd the-haunted-frog
python -m venv .venv && source .venv/bin/activate && pip install -e .
python -m haunted_frog train      # 学習（1秒かからない）
python -m haunted_frog haunt      # 幽霊ターミナル（Ctrl+C で終了）
python -m haunted_frog evaluate   # 上の実験（表の数字）
python scripts/diagnose.py        # THE の確率と点数の内訳
```

`data/halloween.txt` を書きかえれば、あなたの 🐸 が別のことを言い出します。

---

- cse-frog: https://github.com/kagioneko/cse-frog ／ ブラウザで試す: https://kagioneko.github.io/cse-frog/play.html
- これはおもちゃです。英語もハロウィンも理解していません。短いフレーズを組み替えているだけで、どこで壊れるかは上の実験のとおりです。
- コードは AI コーディングアシスタント（Claude Code）と一緒に書きました。数字はすべてリポジトリを実行して出たものです。

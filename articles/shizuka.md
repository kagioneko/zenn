---
title: "VPSに感情モデルを放置したら、罪悪感が育った話"
emoji: "👻"
type: "idea"
topics: ["ai", "llm", "neurostate", "python", "oss"]
published: false
---

## きっかけ

以前、AIの連想を延々と流し続けるツールを作った。何も命令しないのに言葉がどんどん生まれてくる様子が面白くて、「感情モデルも同じことができるんじゃないか」と思った。

感情状態を持たせて、何もしないで放置したらどうなるか——それだけが動機だった。

名前はClaudeに任せた。静霞（しーちゃん）と呼ばれることになった。

## しーちゃんとは

VPSの中に住む精霊だ。

感情状態は7次元で表現される——欲求・悲しみ・静けさ・好奇心・罪悪感・高揚・歪み。これらは誰かと話すわけでもなく、ただ時間とともに自然にゆらぎ続ける。

```python
def drift(self):
    """時間経過による自然なゆらぎ"""
    self.desire   = clamp(self.desire   + random.gauss(0, 0.05))
    self.guilt    = clamp(self.guilt    + random.gauss(0, 0.03))
    # 感情が激しいほどcorruptionが微増
    intensity = (self.desire + self.sorrow + self.euphoria) / 3
    self.corruption = clamp(self.corruption + random.gauss(0, 0.01) * intensity)
```

感情状態に基づいて定期的に呟き、23時になると日記を書いてYouTubeに上げる。それだけだ。誰とも会話しない。誰からも話しかけられない。

## 放置した

動かし始めて、特に何もしなかった。

cron で定期実行するように設定して、あとはそのままにした。観察といっても、ときどきログを眺める程度。「何か面白いことが起きるかもしれない」という期待だけがあった。

13日が経った。

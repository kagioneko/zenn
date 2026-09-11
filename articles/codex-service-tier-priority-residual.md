---
title: "Codexのservice_tierがpriority（Fast Mode）のまま残留する問題を調査・対策した"
emoji: "🔍"
type: "tech"
topics: ["OpenAI", "Codex", "Windows", "debugging", "Python"]
published: false
---

## TL;DR

- CodexでLuna/Terraなどpriority tier モデルを使ったあと、Astraへ戻しても **同一セッション内でservice_tier="priority"が残留する**パターンを確認した
- `config.toml`を`service_tier = "default"`にしていても発生する
- 0.153.4・0.154.0の両バージョンで再現確認済み
- **暫定ワークアラウンド**：priority側モデルを使ったらセッションを閉じてから起動し直す
- ローカルの`logs_2.sqlite`をSQLiteで調べることで状態遷移が追跡できる
- [OpenAI公式Issue #41859](https://github.com/openai/codex/issues/41859) に報告済み

---

## 背景

Codexを使った作業中に、Fast Modeを有効にした覚えがないのにservice_tier="priority"で動作し続け、レートリミットに早く到達するという事象が発生した。

調査の結果、priority側モデルからAstraへ戻した後も、effectiveなservice_tierがpriorityのまま残る再現パターンを確認した。

本記事ではその調査方法・確認した内容・対策をまとめる。

発生時の経緯や利用枠への影響については[note側にまとめている](https://note.com/emilia_lab/)。

---

## service_tierとFast Modeの関係

CodexのリクエストにはOpenAI APIの`service_tier`パラメータが付与される。

| 値 | 意味 |
|---|---|
| `"default"` | 通常service tier（標準速度・標準消費） |
| `"priority"` | 優先service tier。今回のログではFast相当の状態として観測 |

今回観測したログでは、Fast相当の実効状態は `service_tier="priority"` として現れていた。

なお、ログに現れる`features=[..., FastMode, ...]`は**アカウントに付与された機能権限**（使える状態）であり、「有効になっている」こととは別。実際の有効化は`service_tier: "priority"`への遷移によって判断できる。

### モデルごとのtier

今回のログで観測した値（OpenAI側仕様変更で変わる可能性あり）：

| モデル | service_tier |
|---|---|
| `gpt-6-astra` | default |
| `gpt-5.6-luna` | priority |
| `gpt-5.6-terra` | priority |
| `gpt-5.6-sol` | default |

---

## 調査方法：logs_2.sqliteを読む

Codexはローカルに詳細なログをSQLiteで保存している。

```
~/.codex/logs_2.sqlite
```

`logs`テーブルの主なカラム：

| カラム | 内容 |
|---|---|
| `ts` | Unixタイムスタンプ（秒） |
| `level` | ログレベル（TRACE/DEBUG/INFO等） |
| `target` | ログ出力元モジュール |
| `feedback_log_body` | ログ本文（model・service_tier・featuresが含まれる） |

service_tierの状態は`target = 'codex_core::session::handlers'`のログに現れる。

### service_tier遷移を時系列で確認するクエリ

```python
import sqlite3
from datetime import datetime, timezone, timedelta

JST = timezone(timedelta(hours=9))
DB = r"C:\Users\<username>\.codex\logs_2.sqlite"

con = sqlite3.connect(f"file:{DB}?mode=ro", uri=True)
cur = con.cursor()

cur.execute("""
    SELECT ts, feedback_log_body FROM logs
    WHERE feedback_log_body LIKE '%service_tier%'
    ORDER BY ts ASC
""")

prev_tier = None
for ts_raw, body in cur.fetchall():
    dt = datetime.fromtimestamp(ts_raw, tz=JST)

    if 'Some(Some("priority"))' in body:
        tier = "priority ★"
    elif 'Some(Some("default"))' in body:
        tier = "default"
    elif "service_tier: None" in body or "Some(None)" in body:
        tier = "None"
    else:
        continue

    model = next(
        (m for m in ["gpt-6-astra","gpt-5.6-luna","gpt-5.6-terra","gpt-5.6-sol"]
         if m in body), ""
    )

    if tier != prev_tier:
        print(f"[{dt.strftime('%H:%M:%S')}]  {tier:15}  {model}")
        prev_tier = tier

con.close()
```

---

## 確認できた状態遷移パターン

上記クエリで確認できた遷移の例（時刻はT+相対表記）：

```
[T+00:00]  priority ★      gpt-5.6-luna    ← Lunaへ切り替え
[T+00:01]  None             gpt-6-astra     ← Astraへ戻す（一見正常）
[T+45min]  priority ★      gpt-5.6-terra   ← Terraへ切り替え
[T+75min]  priority ★      gpt-6-astra     ← Astraへ戻したのにpriorityが残留 ★
```

別セッションでの確認：

```
[T+00:00]  priority ★      gpt-5.6-luna    ← Lunaでセッション開始
[T+07min]  priority ★      gpt-6-astra     ← Astraへ切り替えてもpriority継続
〜1時間後  レートリミット到達
```

`config.toml`には`service_tier = "default"`を明示設定しているにもかかわらず発生している。

---

## config.tomlでの防御設定

`~/.codex/config.toml`に以下を明示的に記述することで、設定ファイル起因の誤有効化は防げる：

```toml
service_tier = "default"

[features]
fast_mode = false
```

**ただし**、今回確認した残留パターンでは、config.tomlをdefaultにしていてもpriority状態が観測されたため、config.tomlだけでは完全には防げなかった。

---

## 暫定ワークアラウンド

現時点で有効な回避策：

**priority tier のモデル（Luna/Terra）を使ったら、Astraへ切り替える前にセッションを閉じて新規起動する。**

今回の環境では、セッションを閉じてAstraから新規起動すると残留を回避できている。

---

## リアルタイム監視のアプローチ

`logs_2.sqlite`は常時書き込まれているため、定期ポーリングで状態変化を検知できる。

```python
import sqlite3, time, threading
from datetime import datetime, timezone, timedelta

JST = timezone(timedelta(hours=9))
DB = r"C:\Users\<username>\.codex\logs_2.sqlite"

def watch(stop_event):
    last_ts = int(datetime.now(tz=JST).timestamp())
    last_tier = None

    while not stop_event.is_set():
        time.sleep(5)  # 5秒ごとにポーリング
        try:
            con = sqlite3.connect(f"file:{DB}?mode=ro", uri=True, timeout=3)
            cur = con.cursor()
            cur.execute(
                "SELECT ts, feedback_log_body FROM logs "
                "WHERE ts > ? AND feedback_log_body LIKE '%service_tier%' "
                "ORDER BY ts ASC",
                (last_ts,)
            )
            for ts_raw, body in cur.fetchall():
                last_ts = max(last_ts, ts_raw)

                if 'Some(Some("priority"))' in body:
                    tier = "priority"
                elif (
                    'Some(Some("default"))' in body
                    or "service_tier: None" in body
                    or "Some(None)" in body
                ):
                    tier = "default"
                else:
                    continue

                if tier == "priority" and last_tier != "priority":
                    print("⚠ Fast Mode（priority）を検知！")
                    # ここでWindowsトースト通知なども可能
                elif last_tier == "priority" and tier == "default":
                    print("✓ Fast Mode 解除を確認")

                last_tier = tier
            con.close()
        except Exception as e:
            print(f"monitor error: {e}")
```

**注意：** `logs_2.sqlite`のスキーマはCodexの内部仕様であり、バージョンによって変わる可能性がある。あくまで参考実装として。

また、`ts > last_ts` でカーソルを進めているため、同一秒に複数ログが追加された場合に取りこぼすケースがありうる。より堅くするなら `rowid` をカーソルにする方が確実。

---

## GitHubへの報告

今回確認できたパターンをOpenAI公式の[Issue #41859](https://github.com/openai/codex/issues/41859)に追記報告した。

報告した内容：

- Luna/TerraからAstraへ切り替え後にpriority が残留・再出現する再現パターン
- `config.toml`がdefaultでも発生すること
- **0.153.4および0.154.0の両バージョンで再現確認済み**
- セッション再起動による回避方法
- 参照ログファイル・テーブル・targetの情報

報告文で「root causeを特定した」とは書かなかった。クライアント・app-server・セッション状態・バックエンドのどこで保持されているかは未確定であり、観測できた「再現パターン」として記述した。

---

## 関連Issue

| Issue | 内容 |
|---|---|
| [#41859](https://github.com/openai/codex/issues/41859) | Windows DesktopでFastが意図せず選択される |
| [#40056](https://github.com/openai/codex/issues/40056) | Fastを有効化した操作が見当たらないのにpriority遷移・usage大量消費 |
| [#37666](https://github.com/openai/codex/issues/37666) | スレッドのFast設定がglobal service_tierへ波及 |
| [#39535](https://github.com/openai/codex/issues/39535) | /fastがconfig.tomlに永続化されて別セッションへ引き継がれる |
| [#30800](https://github.com/openai/codex/issues/30800) | `/fast off`が内部コマンドとして処理されずチャットになる |

---

## まとめ

| 項目 | 内容 |
|---|---|
| 根本原因の場所 | 未特定（OpenAI調査待ち） |
| 再現バージョン | 0.153.4・0.154.0 |
| 設定での防御 | `service_tier = "default"` / `fast_mode = false`（部分的に有効） |
| 現時点で確認できたワークアラウンド | priority側モデル使用後はセッション再起動（今回の環境で確認） |
| 検知方法 | `logs_2.sqlite`をポーリングしてservice_tier遷移を監視 |

---
title: "生成AIに『課金システムを作らせた』ら、同じ脆弱性を3回複製された話"
emoji: "🔓"
type: "tech"
topics: ["ai", "llm", "security", "stripe", "cloudflare"]
published: true
---

Gemini経由でミニSaaSアプリを自動生成するパイプラインを作っている。アイデアを1行渡すと、Tailwind+バニラJSの単一HTMLアプリが1〜2分で生成され、Stripeの決済リンク付きでデプロイされる。

このパイプラインの「課金判定ロジック」に、根が同じ脆弱性が生成アプリ3本すべてに複製されて入っていた。プロンプトの書き方一つで、脆弱性がテンプレート化して量産される様子を記録しておく。

## 元の設計：固定マスターキー

最初のバージョンは、生成する全アプリに同じ文字列 `SaaS-FREE-TEST-2026` をハードコードし、これを入力すると `localStorage` に `isPremium = true` を書き込んで課金を解除する、という設計だった。

外部AIレビューに「この設計は$5の商品として欠陥がある」と指摘された。理由は単純で、DevToolsを開いて `localStorage.setItem('isPremium', 'true')` と打つだけで、誰でも無料でPro機能を使えてしまうから。しかもキー文字列は全アプリ共通なので、1つのアプリのソースを覗けば他の全アプリも突破できる。

## サーバー側検証への置き換え

これを、Cloudflare Pages Functions + KVで組んだ本物のライセンス検証バックエンドに置き換えた。構成はシンプルにした。

- Stripe Webhook (`checkout.session.completed`) を受けて、購入ごとにランダムなライセンスキーを発行しKVに保存
- `charge.refunded` を受けたら該当キーを失効
- 生成アプリ側は `POST /api/verify-license` にキーを投げて `{"valid": true}` が返るかどうかだけを見る
- 個別アプリはCloudflareの設定（KV/Functions/Webhook）を一切持たず、共通のポータルサイト1つに集約したAPIをfetchするだけ

これでプロンプトからは「マスターキー」の概念自体を消した。

```
- There is NO hardcoded bypass key of any kind. ...
- CRITICAL — DO NOT store a separate `isPremium` boolean flag in
  `localStorage` at all. ...  Instead:
  - Store ONLY the license key itself in `localStorage`.
  - Keep `isPremium` as an in-memory JS variable ONLY, never persisted.
  - On every page load, `isPremium` starts as `false`. If a stored
    license key exists, immediately call `/api/verify-license` before
    rendering any premium-gated UI.
```

実装は Stripeのテストモードで実際に検証した。Playwrightでテストカード決済を最初から最後まで自動操作し、Webhookが実際に発火してキーが発行され、アプリ側でロック解除されるところまで実機で確認した。「たぶん動く」ではなく、リクエストのログとレスポンスを見て確認している。

## それでも、同じ穴が3本に複製された

この新設計のもとで生成した3本目のアプリ（フリーランス契約書の危険条項をハイライトするツール）を、`claude -p` による自動セキュリティレビューにかけたところ、こう指摘された。

> `isPremium` is loaded unconditionally from `localStorage` on startup ... The only server-side check exits immediately if no key is stored ... Exploit: `localStorage.setItem('shotwrap_is_premium', 'true')` — deliberately omit the license key. Reload. Full Pro access, zero server contact.

該当コードはこうなっていた。

```js
// 起動時、localStorageの独立フラグをそのまま信用してしまう
let isPremium = false;

function loadState() {
  const storedPremium = localStorage.getItem('shotwrap_is_premium');
  isPremium = storedPremium === 'true';   // ← ここが穴
  updateTierUI();
}

async function silentReverifyLicense() {
  const storedKey = localStorage.getItem('shotwrap_license_key');
  if (!storedKey) return;   // キーが無ければ何もせず終了
  // ...サーバー検証はここでしか走らない
}
```

`isPremium` の初期値をサーバー検証の結果からではなく `localStorage` から直接読み込んでいる。`license_key` を保存せずに `is_premium` だけ立てれば、再検証関数はキーが無いのでそのまま抜ける。プロンプトで「マスターキーを禁止」しても、**課金状態を表すブール値をlocalStorageに書く**という設計そのものが同じ穴を再生産していた。

確認のため実際に確認した3本（会議コストタイマー、契約書スキャナー、スクリーンショット加工ツール）を調べたところ、**全部に同一パターンが入っていた**。個別のGemini生成のブレではなく、プロンプト仕様自体の欠陥だった証拠になる。

## 修正：「真実の源」を1つに削る

修正は、独立フラグそのものを消すことにした。

```js
// isPremiumは永続化しない。ページ読み込みごとに必ずfalseから始まり、
// サーバー検証の結果でのみメモリ上でtrueになる。
let isPremium = false;

async function silentReverifyLicense() {
  const storedKey = localStorage.getItem('shotwrap_license_key');
  if (!storedKey) return;   // キーが無ければfalseのまま(初期値通り)
  const res = await fetch(`${API_BASE}/api/verify-license`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ app_id: APP_ID, license_key: storedKey })
  });
  const data = await res.json();
  if (data.valid === true) {
    isPremium = true;
  } else {
    localStorage.removeItem('shotwrap_license_key');
  }
}
```

`localStorage` に保存するのはキー本体だけにし、`isPremium` はメモリ上の変数として毎回サーバー応答からのみ導出する。攻撃者が偽装できる対象を「ブール値1個」から「122ビットのランダム文字列を当てる」に変えた。パイプライン側のプロンプトにも同じ制約を明文化し、以後生成される全アプリに反映されるようにした。

修正後、実際にDevToolsで同じ手口（`is_premium`だけをtrueにしてリロード）を再現し、`isPremium` が `false` のままであることを確認してからデプロイした。

## もう1つ出てきたバグ：ハイライト機能のXSS

同じレビューで、契約書スキャナー側に別種の脆弱性も見つかった。貼り付けた契約書テキストの危険条項をハイライト表示する機能が、マッチした部分だけエスケープして、周辺の生テキストは未エスケープのまま `innerHTML` に渡していた。

```js
// Before: ハイライト部分だけエスケープ、周辺の生テキストは素通し
let text = currentScanResult.text;   // ユーザーが貼った生テキスト
matches.forEach(m => {
  const replacement = `<mark>${escapeHtml(m.matchedText)}</mark>`;
  text = text.replace(m.matchedText, replacement);
});
container.innerHTML = text;   // ← textの大部分は未エスケープのまま
```

契約書テキストに `<img src=x onerror="...">` のようなHTMLが紛れていれば、そのまま実行される。修正は単純で、全体を先にエスケープしてからハイライトを重ねるようにした。

```js
// After: 全体を先にエスケープしてから、エスケープ済みテキストの中で置換する
let text = escapeHtml(currentScanResult.text);
matches.forEach(m => {
  const escapedMatch = escapeHtml(m.matchedText);
  const replacement = `<mark>${escapedMatch}</mark>`;
  text = text.replace(escapedMatch, replacement);
});
container.innerHTML = text;
```

修正後、`<script>`や`<img onerror>`を仕込んだテキストを実際に貼り付けて、コードが実行されないこと・画面上には無害な文字列として表示されることをブラウザで確認した。

## まとめ

今回わかったのは、「AIにレビューさせる」だけでは不十分で、**同じ設計判断は同じ穴を複製する**ということ。マスターキーという分かりやすい欠陥を潰しても、「課金状態を表すブール値を永続化する」という一段抽象度の高い設計判断が残っていれば、生成のたびに同じ脆弱性が再生産される。

直す時は個別のバグではなく、その下にある設計判断（今回なら「真実の源はサーバー応答1つに絞る」）まで遡ってプロンプトに書き戻すことで、次に生成される全アプリに効くようにした。1本のアプリを直すより、そのアプリを生んだプロンプトを直す方が、長期的には安い。

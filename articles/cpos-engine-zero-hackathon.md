---
title: "AIエージェントに生産性と防御を。Google Cloud + Geminiで作る『Engine-Zero』"
emoji: "🛡️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["googlecloud", "gemini", "aiagent", "devops", "security"]
published: true
---

# はじめに
「AIエージェントにコードを修正させる」——。
これは魅力的な響きですが、いざ本番環境に近いレポジトリに導入しようとすると、一つの大きな壁にぶつかります。それは**「信頼」**です。

AIのハルシネーション（もっともらしい嘘）によって脆弱性が埋め込まれたら？機密情報が外部に漏れたら？プロンプトが肥大化してコストが跳ね上がったら？

DevOps x AI Agent Hackathon 2026に向けて私たちが開発した **『CPOS Engine-Zero』** は、これらの課題に対する一つの究極の回答です。Google Cloud (Gemini) と、独自設計の **Context Pointer OS (CPOS)** を組み合わせることで、「爆速の自動修正」と「鉄壁の防御」を両立させました。

# 核心：Context Pointer OS (CPOS)
最近のLLMは100万トークンを超える長いコンテキストを扱えますが、すべてを一度に読み込ませるのは非効率です。

Engine-Zeroでは、独自の **「Context Pointer (#ctx)」** という概念を採用しています。これは、ファイルの実体ではなく「ファイルの特定の場所、過去の失敗パターン、依存関係」といった情報を軽量なポインタとして管理する仕組みです。

AIエージェントは、タスクに必要なポインタだけを「参照」し、必要な時だけ実体を取得（Reconstruct）します。これにより：
- **トークンコストの劇的な削減**
- **不要なノードを読み込まないことによる精度の向上**
- **機密コンテキストの厳格なアクセス制御**
を実現しました。

# 鉄壁の「Defense-in-Depth」アーキテクチャ
Engine-Zeroが単なる自動化スクリプトではない理由は、その「防御層」の厚さにあります。

### 1. Docker Sandbox による隔離実行
AIが生成したコードは、即座にメインリポジトリに反映されることはありません。物理的に隔離されたDockerコンテナ内で、静的解析（Ruff）と単体テスト（Pytest）をクリアしたものだけが「修正案」として認められます。

### 2. 暗号化ストレージ & ハッシュチェーン
エージェントの「脳」であるコンテキストポインタや監査ログは、**AES-128 (Fernet)** で暗号化されて保存されます。
さらに、すべての操作ログは **SHA-256 ハッシュチェーン** で連結されており、万が一サーバーが侵害されても、過去のログの改ざんや削除を100%検知できる「不変の帳簿」を備えています。

### 3. HTTPS & セキュリティヘッダー
運用ダッシュボードは **HSTS (Strict-Transport-Security)** や **CSP (Content-Security-Policy)** を強制する HTTPS 環境で動作し、管理通信の盗聴やブラウザ経由の攻撃を徹底的にシャットアウトしています。

# 思考を可視化する「Memory Network Graph」
ハッカソンにおいて「AIが中で何を考えているか」を見せることは重要です。
Engine-Zeroのダッシュボードには、キャンバスベースの力学モデルを用いた **Memory Network Graph** を搭載しました。

![](/images/memory-graph-demo.gif)
*(画像はイメージです：ファイル（青）、ポインタ（紫）、発見されたバグ（赤）がリアルタイムに繋がっていく様子)*

AIがどのファイルを読み込み、どのポインタを参照して修正を決断したのかがリアルタイムに可視化されるため、ユーザー（管理者）は安心して「Approve（承認）」ボタンを押すことができます。

# Google Cloud での運用
Engine-Zero は **Google Cloud Run** にデプロイされており、以下のようなサービスを活用しています。

- **Vertex AI (Gemini 1.5 Pro/Flash)**: 修正の核となるインテリジェンス。
- **Secret Manager**: GitHubトークンや暗号化キーの安全な管理。
- **Cloud Run**: オートスケールとHTTPS終端の提供。

# おわりに：自律型インフラの未来
Engine-Zero は、単なるバグ修正ツールではなく、**「インフラ自体が自らを守り、修復する」** 自律型OSへの第一歩です。

今後は CPOS の **Genetic Kernel**（遺伝的カーネル）をさらに強化し、一度起きた障害から学び、二度と同じミスをさせない「自己進化する防御」の実装を目指していきます。

---
**GitHubリポジトリ:** [kagioneko-emi/cpos-engine-zero](https://github.com/kagioneko-emi/cpos-engine-zero)
**ハッカソン提出作品:** DevOps x AI Agent Hackathon 2026

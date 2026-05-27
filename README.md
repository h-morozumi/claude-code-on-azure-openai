# claude-code-on-azure-openai

Claude Code / Claude Desktop を **Azure OpenAI** のモデル（GPT-5, GPT-4o など）で動かすためのハンズオン。Anthropic API 形式のリクエストを Azure OpenAI 形式に変換する Proxy / Gateway を、**ローカル実行**と **Azure デプロイ**の両パターンで構築します。

> Microsoft 社員・パートナー・お客様向けの学習用リポジトリです。本番運用ではなく、検証・学習用途を想定しています。

---

## なぜ Proxy が必要か

Claude Code / Claude Desktop は Anthropic API（`/v1/messages`）形式でリクエストを送信します。一方、Azure OpenAI は OpenAI 互換の Chat Completions API（`/v1/chat/completions`）形式です。両者は **メッセージ構造・ツール呼び出し・ストリーミング形式が異なる**ため、間に変換層（Proxy / Gateway）を挟む必要があります。

```
┌──────────────────┐     Anthropic 形式      ┌─────────┐     Azure OpenAI 形式    ┌──────────────┐
│ Claude Code /    │  ──────────────────▶   │  Proxy  │  ──────────────────▶   │ Azure OpenAI │
│ Claude Desktop   │   /v1/messages          │         │   /v1/chat/completions  │  (GPT-5 等)  │
└──────────────────┘                         └─────────┘                          └──────────────┘
        ▲                                                                                  │
        └──────────────────────────────────────────────────────────────────────────────────┘
                                        レスポンス変換
```

クライアント側は `ANTHROPIC_BASE_URL` を Proxy のエンドポイントに向けるだけで、コード変更不要で Azure OpenAI のモデルを利用できます。

---

## 3 つの Proxy パターン

本リポジトリでは、目的の異なる 3 つの Proxy を比較しながら学べる構成にしています。すべて OSS / 無料で利用可能です。

| 観点 | パターン A: LiteLLM Proxy | パターン B: claude-code-proxy | パターン C: Bifrost |
|------|--------------------------|------------------------------|---------------------|
| **実体** | Python 製スタンドアロン HTTP サーバー | Python アプリ（内部で LiteLLM SDK を使用） | Go 製 高速ゲートウェイ |
| **ライセンス** | MIT | MIT | Apache 2.0 |
| **Azure OpenAI 対応** | ネイティブ対応 | OpenAI 互換経由 | ネイティブ対応 |
| **Anthropic エンドポイント** | `/v1/messages` 互換 | `/v1/messages` 互換 | `/anthropic` 専用エンドポイント |
| **設定 UI** | YAML ファイル中心（Admin UI あり） | `.env` ファイル数行 | **Web UI で設定**（`http://localhost:8080`） |
| **パフォーマンス** | 標準 | 標準 | LiteLLM 比で大幅高速（ベンダー公表） |
| **追加機能** | フェイルオーバー、ログ、レート制限、コスト追跡 | シンプル（変換のみ） | MCP Gateway、Virtual Keys、Budget、SSO、Vault |
| **学習コスト** | やや高い | 低い | 中（UI で直感的） |
| **向いている用途** | 組織導入・複数人利用・運用機能 | 個人利用・素早く試したい | デモ映え・高パフォーマンス・MCP 活用 |
| **Azure デプロイ** | 公式 Helm Chart / Container Apps | Container Apps / App Service | 公式 Helm Chart / Docker / Container Apps |

**推奨される学習フロー**:
1. **パターン B** で最小構成での動作確認（5 分で起動）
2. **パターン C** で Web UI による設定 & MCP Gateway の体験
3. **パターン A** で本格的な運用機能（フェイルオーバー・ログ・コスト管理）を学ぶ

---

## 前提条件

### 共通
- Azure サブスクリプション（Azure OpenAI リソース作成済み）
- Azure OpenAI のモデルデプロイ（例: `gpt-5`, `gpt-4o`）
- Claude Code または Claude Desktop インストール済み
- Docker Desktop（ローカル実行の場合）

### Azure デプロイの場合
- Azure CLI または Azure Developer CLI (`azd`)
- 推奨リージョン: Japan East / East US

### Azure OpenAI 情報の取得
以下を控えておいてください:

| 項目 | 例 |
|------|-----|
| エンドポイント | `https://<your-resource>.openai.azure.com` |
| API キー | `xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx` |
| デプロイ名 | `gpt-5` |
| API バージョン | `2024-10-21` |

---

## クイックスタート

### パターン A: LiteLLM Proxy（ローカル / Docker）

```bash
cd pattern-a-litellm-proxy/local

# 1. 環境変数を設定
cp .env.example .env
# .env を編集して Azure OpenAI の情報を記入

# 2. Proxy 起動
docker compose up -d

# 3. Claude Code を Proxy 経由で起動
export ANTHROPIC_BASE_URL=http://localhost:4000
export ANTHROPIC_API_KEY=sk-dummy
claude
```

### パターン B: claude-code-proxy（ローカル / Python）

```bash
cd pattern-b-claude-code-proxy/local

# 1. 環境変数を設定
cp .env.example .env
# .env を編集

# 2. Proxy 起動
uv run claude-code-proxy

# 3. Claude Code を Proxy 経由で起動
export ANTHROPIC_BASE_URL=http://localhost:8082
export ANTHROPIC_API_KEY=sk-dummy
claude
```

### パターン C: Bifrost（ローカル / Docker または npx）

```bash
cd pattern-c-bifrost/local

# 1. Bifrost 起動（どちらかの方法）
# (a) Docker
docker run -p 8080:8080 -v $(pwd)/data:/app/data maximhq/bifrost

# (b) npx（Node.js 必要）
npx -y @maximhq/bifrost

# 2. ブラウザで Web UI を開いて Azure OpenAI を設定
open http://localhost:8080
#   - Provider: Azure OpenAI を選択
#   - Endpoint / API Key / Deployment 名を入力
#   - 仮想 API キー（Virtual Key）を発行

# 3. Claude Code を Bifrost 経由で起動（/anthropic エンドポイントを指定）
export ANTHROPIC_BASE_URL=http://localhost:8080/anthropic
export ANTHROPIC_API_KEY=<Bifrost で発行した Virtual Key>
claude
```

### Azure デプロイ（Container Apps）

```bash
# パターン A の場合
cd pattern-a-litellm-proxy/azure
azd auth login
azd up

# パターン C の場合
cd pattern-c-bifrost/azure
azd up

# デプロイ後に表示される URL を ANTHROPIC_BASE_URL に設定
```

---

## リポジトリ構成

```
claude-code-on-azure-openai/
├── README.md                          # このファイル
├── docs/
│   ├── 01-overview.md                 # アーキテクチャ詳細
│   ├── 02-comparison.md               # 3 パターンの詳細比較
│   ├── 03-azure-openai-setup.md       # Azure OpenAI 側の準備
│   └── 04-troubleshooting.md          # よくあるエラーと対処
│
├── pattern-a-litellm-proxy/
│   ├── README.md                      # パターン A 詳細手順
│   ├── local/                         # ローカル Docker 実行
│   │   ├── docker-compose.yml
│   │   ├── config.yaml                # LiteLLM 設定
│   │   └── .env.example
│   └── azure/                         # Azure Container Apps デプロイ
│       ├── azure.yaml                 # azd 定義
│       ├── infra/                     # Bicep テンプレート
│       └── README.md
│
├── pattern-b-claude-code-proxy/
│   ├── README.md                      # パターン B 詳細手順
│   ├── local/                         # ローカル Python 実行
│   │   ├── pyproject.toml
│   │   └── .env.example
│   └── azure/                         # Azure App Service デプロイ
│       ├── azure.yaml
│       └── infra/
│
└── pattern-c-bifrost/
    ├── README.md                      # パターン C 詳細手順
    ├── local/                         # ローカル Docker / npx 実行
    │   ├── docker-compose.yml
    │   ├── data/                      # Bifrost 設定永続化用
    │   └── README.md
    └── azure/                         # Azure Container Apps デプロイ
        ├── azure.yaml
        ├── infra/                     # Bicep テンプレート
        └── README.md
```

---

## 動作確認

Proxy 経由で Claude Code が起動したら、以下で動作確認できます:

```bash
# 簡単な動作確認
claude "Hello, what model are you?"

# Azure OpenAI 側のログで GPT-5 へのリクエストが記録されていれば成功
```

パターン C（Bifrost）の場合、Web UI の **Logs** タブでリクエスト履歴・レイテンシ・トークン使用量をリアルタイムで確認できます。

---

## トラブルシューティング

| 症状 | 確認ポイント |
|------|------------|
| `401 Unauthorized` | Azure OpenAI の API キー / エンドポイントが正しいか |
| `404 Deployment not found` | Azure OpenAI のデプロイ名が設定ファイルと一致しているか |
| ツール呼び出しが動かない | API バージョンが `2024-10-21` 以降か（古いバージョンは Tool calling 非対応） |
| ストリーミングが途切れる | Container Apps の場合、Ingress の HTTP/2 設定を確認 |
| Bifrost で `/anthropic` が 404 | `ANTHROPIC_BASE_URL` にパス `/anthropic` を含めているか |
| Bifrost Web UI が開かない | ポート 8080 が他プロセスと競合していないか |

詳細は [docs/04-troubleshooting.md](./docs/04-troubleshooting.md) を参照してください。

---

## 制限事項

Azure OpenAI 経由で Claude Code を使う場合、以下の Anthropic 独自機能は利用できません:

- **Prompt Caching** — OpenAI 形式に変換時に失われる
- **Extended Thinking の詳細出力** — モデルが `o1` / `o3` / `gpt-5-thinking` 系の場合は近い動作
- **PDF / Citations** — Anthropic ネイティブ機能

これらが必要な場合は、Anthropic API を直接利用してください。

---

## パターン別の選択ガイド

| こんな人に | おすすめパターン |
|------------|----------------|
| とりあえず最速で動かしたい | **B**（claude-code-proxy） |
| Web UI で設定したい・デモ映え重視 | **C**（Bifrost） |
| MCP Gateway も一緒に使いたい | **C**（Bifrost） |
| 組織で複数人に提供したい | **A**（LiteLLM Proxy） |
| コスト追跡・レート制限を厳密に行いたい | **A**（LiteLLM Proxy） |
| 高スループットが必要 | **C**（Bifrost） |

---

## 参考リンク

- [LiteLLM ドキュメント](https://docs.litellm.ai/)
- [claude-code-proxy GitHub](https://github.com/1rgs/claude-code-proxy)
- [Bifrost GitHub (maximhq/bifrost)](https://github.com/maximhq/bifrost)
- [Bifrost ドキュメント](https://www.getmaxim.ai/docs/bifrost/overview)
- [Anthropic OpenAI SDK Compatibility](https://docs.anthropic.com/en/api/openai-sdk)
- [Azure OpenAI Service ドキュメント](https://learn.microsoft.com/azure/ai-services/openai/)
- [Claude Code ドキュメント](https://docs.anthropic.com/en/docs/claude-code/overview)

---

## ライセンス

MIT License

---


Issue / PR 歓迎です。Azure OpenAI 以外（Azure AI Foundry、AWS Bedrock 等）への拡張、または別の Proxy パターンの追加も Welcome。

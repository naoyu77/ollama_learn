# ローカルLLM学習メモ

## 1. Ollamaとは

ローカルでLLMを簡単に動かすためのツール。macOS/Linux/Windowsに対応。

### インストール

```bash
brew install ollama
brew services start ollama  # サービスとして起動
```

### 基本コマンド

```bash
ollama pull <model>    # モデルをダウンロード
ollama run <model>     # 対話モードで実行
ollama list            # インストール済みモデル一覧
ollama rm <model>      # モデル削除
```

## 2. 利用可能なモデル

### 小型モデル（ローカル向け）

| モデル | サイズ | 特徴 |
|--------|--------|------|
| qwen2.5:0.5b | 0.5B (~400MB) | 最軽量、多言語対応 |
| qwen2.5:1.5b | 1.5B (~1GB) | 軽量で高品質 |
| gemma2:2b | 2B (~1.6GB) | Google製 |
| phi3 | 3.8B (~2.3GB) | Microsoft製、高性能 |
| llama3.2:3b | 3B | Meta製 |

### 特殊用途モデル

- `qwen2.5-coder` - コード生成特化
- `llava` - 画像+テキスト（マルチモーダル）
- `nomic-embed-text` - 埋め込みベクトル生成（RAG用）
- `deepseek-r1` - 推論特化

全モデル一覧: https://ollama.com/library

## 3. Ollamaのアーキテクチャ

### なぜサービスとして動作するのか

```
【モデルロードのコスト】
モデルのロード: 数秒〜数十秒
推論実行:      数百ミリ秒〜数秒
```

毎回モデルをロードすると非効率なため、サービスがメモリに常駐してモデルを保持する。

```
┌─────────────────────────────────────┐
│  Ollama Service (port 11434)        │
│  - REST API サーバー                 │
│  - モデルのロード/アンロード管理       │
└─────────────────────────────────────┘
        ↑ HTTP
        ↓
┌───────────────────┐  ┌──────────────┐
│ ollama run xxx    │  │ curl/Python  │
│ (CLI)             │  │ (API直接)    │
└───────────────────┘  └──────────────┘
```

### サービス管理

```bash
brew services start ollama   # 起動
brew services stop ollama    # 停止
brew services info ollama    # 状態確認
```

## 4. Ollama REST API

サービス起動時、`http://localhost:11434` でAPIが利用可能。

### 主要エンドポイント

| エンドポイント | メソッド | 用途 |
|----------------|----------|------|
| `/api/generate` | POST | テキスト生成 |
| `/api/chat` | POST | チャット形式 |
| `/api/tags` | GET | モデル一覧 |
| `/api/embeddings` | POST | 埋め込みベクトル |

### 使用例

```bash
# モデル一覧
curl http://localhost:11434/api/tags

# テキスト生成
curl http://localhost:11434/api/generate -d '{
  "model": "qwen2.5:0.5b",
  "prompt": "Hello",
  "stream": false
}'

# チャット
curl http://localhost:11434/api/chat -d '{
  "model": "qwen2.5:0.5b",
  "messages": [{"role": "user", "content": "こんにちは"}],
  "stream": false
}'
```

### Pythonから利用

```python
# 公式ライブラリ
pip install ollama

import ollama
response = ollama.chat(model='qwen2.5:0.5b', messages=[
    {'role': 'user', 'content': 'こんにちは'}
])
print(response['message']['content'])
```

## 5. チャットアプリ実装

### 構成

```
ollama-chat/
├── server.js          # Express バックエンド
├── package.json
└── client/            # React フロントエンド
    └── src/
        ├── App.jsx
        └── App.css
```

### バックエンド（Express + SSE）

- Ollamaの`/api/chat`にリクエスト
- `stream: true`でストリーミング取得
- Server-Sent Events (SSE)でフロントに中継

### フロントエンド（React）

- `fetch` + `ReadableStream`でSSEを受信
- 受信したチャンクを逐次表示

### 起動

```bash
cd ollama-chat
node server.js &              # バックエンド (port 3001)
cd client && npm run dev      # フロントエンド (port 5173)
```

## 6. SSE（Server-Sent Events）

サーバーからクライアントへ一方向にデータを送り続けるHTTP標準技術。

### なぜ使うのか

```
【SSEなし】
ユーザー → 質問 → (3秒待つ) → 回答一括表示

【SSEあり】
ユーザー → 質問 → こ → こん → こんに → こんにち...（逐次表示）
```

LLMはトークンを1つずつ生成するため、全部待つと長文で数十秒かかる。
SSEで逐次送ることで体感速度とUXが向上する。

### 実装方法

```javascript
// サーバー側
res.setHeader('Content-Type', 'text/event-stream');
res.setHeader('Cache-Control', 'no-cache');
res.setHeader('Connection', 'keep-alive');

res.write(`data: {"content": "こ"}\n\n`);  // data: で始め、\n\nで区切る
```

### 他技術との比較

| 技術 | 方向 | 用途 |
|------|------|------|
| SSE | サーバー→クライアント | ストリーミング、通知 |
| WebSocket | 双方向 | チャット、ゲーム |
| ポーリング | クライアント→サーバー（繰り返し） | 古い手法 |

## 7. LLMの会話履歴

### 毎回すべて送る

LLMはステートレス（状態を持たない）なので、文脈を理解させるには過去のやり取りを毎回渡す。

```javascript
// 1回目
messages: [{ role: 'user', content: 'こんにちは' }]

// 2回目（履歴を含める）
messages: [
  { role: 'user', content: 'こんにちは' },
  { role: 'assistant', content: 'こんにちは！' },
  { role: 'user', content: '天気は？' }  // 新しい質問
]
```

### 問題点と対策

| 問題 | 対策 |
|------|------|
| トークン増加 | 直近N件だけ送る |
| 上限到達 | 古い会話を要約 |
| 速度低下 | 重要なメッセージだけ残す |

## 8. 実装のポイント

### 良い点

- **ストリーミング中継パターン**: バッファリングせず即座に転送（メモリ効率良い）
- **会話履歴の保持**: フロントエンドで履歴管理し文脈を維持

### 改善余地（本番向け）

| 項目 | 現状 | 改善案 |
|------|------|--------|
| CORS | 全開放 | `origin`を制限 |
| エラー処理 | 最小限 | 詳細なハンドリング |
| 接続切断 | 未対応 | `req.on('close')`で処理 |
| タイムアウト | なし | 設定追加 |
| 認証 | なし | JWT等を追加 |

## 9. Ollamaの全機能

### API一覧

| エンドポイント | 用途 |
|----------------|------|
| `/api/generate` | テキスト生成（単発） |
| `/api/chat` | チャット（会話履歴対応） |
| `/api/embeddings` | 埋め込みベクトル生成（RAG用） |
| `/api/tags` | モデル一覧 |
| `/api/pull` | モデルダウンロード |
| `/api/delete` | モデル削除 |

### カスタムモデル作成（Modelfile）

```dockerfile
FROM qwen2.5:0.5b
SYSTEM "あなたは関西弁で話すアシスタントです"
PARAMETER temperature 0.7
```

```bash
ollama create my-kansai -f Modelfile
ollama run my-kansai
```

### OpenAI互換API

```python
from openai import OpenAI

client = OpenAI(
    base_url="http://localhost:11434/v1",
    api_key="dummy"
)
response = client.chat.completions.create(
    model="qwen2.5:0.5b",
    messages=[{"role": "user", "content": "Hello"}]
)
```

### その他の機能

- マルチモーダル（画像入力）: `llava`モデル
- 埋め込みベクトル: RAG用に`nomic-embed-text`
- GPU/メモリ設定: 環境変数で制御
- リモートアクセス: `OLLAMA_HOST=0.0.0.0:11434`

## 10. シェルの基礎知識

### サービス・プロセスの停止方法

```bash
# Ollamaサービス停止
brew services stop ollama

# 特定ポートのプロセスを探して停止
lsof -i :3001              # ポート3001を使用中のプロセス確認
kill <PID>                  # プロセス停止

# 一括停止（ポート指定）
kill $(lsof -t -i:3001) 2>/dev/null
kill $(lsof -t -i:5173) 2>/dev/null
```

### コマンド解説: `kill $(lsof -t -i:5173) 2>/dev/null`

```
kill $(lsof -t -i:5173) 2>/dev/null
     │  │    │  │       │
     │  │    │  │       └─ 標準エラーを捨てる
     │  │    │  └───────── ポート5173を使用中の
     │  │    └──────────── PIDのみ出力（-t = terse）
     │  └───────────────── プロセス情報を表示
     └──────────────────── プロセスを終了
```

### ファイルディスクリプタ（入出力の番号）

| 番号 | 名前 | 用途 |
|------|------|------|
| 0 | stdin | 標準入力（キーボード） |
| 1 | stdout | 標準出力（正常な結果） |
| 2 | stderr | 標準エラー出力（エラーメッセージ） |

### リダイレクト

```bash
command > file       # stdout(1)をファイルへ
command 2> file      # stderr(2)をファイルへ
command &> file      # 両方をファイルへ
command 2>/dev/null  # エラーを捨てる
```

### /dev/null とは

書き込んだデータが全て消える特殊ファイル（ブラックホール）。

```
/dev/           ← デバイスファイルの置き場
├── null        ← 虚無（書いたら消える）
├── zero        ← 無限のゼロを生成
├── random      ← 乱数を生成
└── tty         ← ターミナル
```

Unixの思想「すべてはファイル」により、デバイスもファイルとして扱う。

## 11. 学んだポイントまとめ

1. **ローカルLLMは意外と手軽** - Ollamaで数コマンドで動く
2. **小型モデルでも実用的** - 0.5B〜3Bでも基本的な会話は可能
3. **サービス化の理由** - モデルロード時間を省略するため
4. **ストリーミングが重要** - UX向上のため逐次表示が標準
5. **REST APIで汎用的** - 言語を問わず利用可能
6. **SSEはシンプル** - 特別なライブラリ不要でHTTP標準
7. **会話履歴は毎回送る** - LLMはステートレス
8. **本番には追加対策が必要** - 認証、エラー処理、レート制限など
9. **シェルの基礎** - リダイレクト、ファイルディスクリプタ、/dev/null

# CodeQuiz AI サーバー

**プログラミングの概念的理解を測るクイズ問題を自動生成する AI モデル**

Ollama を使用したローカル実行可能なカスタムモデルです。コードスニペットを分析し、フレームワーク・ライブラリ・設計パターンの理解を問う日本語の選択問題を生成します。

---

## 📚 詳細ドキュメント

すべての詳細情報は以下のドキュメントを参照してください：

### セットアップ・管理

- **[インストールガイド](./INSTALLATION.md)** - Ollama のインストールとセットアップ
- **[Ollama コマンドリファレンス](./OLLAMA_COMMANDS.md)** - すべてのコマンドの詳細

### モデルのカスタマイズ

- **[Modelfile リファレンス](./MODELFILE_REFERENCE.md)** - パラメータの詳細解説と最適化

### テスト・検証

- **[API テストガイド](./API_TESTING.md)** - curl でのテスト方法と検証スクリプト

### 問題解決

- **[トラブルシューティング](./TROUBLESHOOTING.md)** - よくある問題と解決方法

---

## 🚀 クイックスタート

### 1. Ollama のインストール

```bash
# macOS / Linux
curl -fsSL https://ollama.com/install.sh | sh

# Windows: https://ollama.com/download からダウンロード
```

### 2. モデルの作成

```bash
ollama create codequiz:latest -f ./Modelfile
```

### 3. 動作確認

```bash
# モデルが登録されているか確認
curl http://localhost:11434/api/tags

# 対話モードで実行
ollama run codequiz:latest
```

### 4. API でテスト

```bash
curl http://localhost:11434/api/generate -d '{
  "model": "codequiz:latest",
  "prompt": "作成する数: 1\nコードスニペット:\ndef hello():\n    print(\"Hello, World!\")",
  "stream": false
}'
```


---

## 🎯 Modelfile の主要設定

現在の `Modelfile` の設定:

```dockerfile
FROM gpt-oss:20b
PARAMETER temperature 0.7       # バランスの取れた創造性
PARAMETER num_ctx 16384         # 長いコードスニペット対応
PARAMETER top_p 0.9             # 高い多様性
PARAMETER repeat_penalty 1.1    # 繰り返し抑制
PARAMETER num_gpu 99            # GPU使用（CPU: 0に変更）
PARAMETER num_thread 10         # CPUスレッド数
```

### ベースモデルの変更

`Modelfile` の1行目を編集:

```dockerfile
# より軽量なモデルに変更（メモリ不足時）
FROM llama3.2:3b  # 約2GB

# 標準的なモデル
FROM llama3.2     # 約4.7GB

# より高品質なモデル
FROM llama3.2:70b # 約40GB
```

変更後、モデルを再作成:

```bash
ollama rm codequiz:latest
ollama create codequiz:latest -f ./Modelfile
```

---

## 出力形式

### 必須 JSON 構造

```json
{
  "questions": [
    {
      "question": "質問文",
      "choices": [
        {"text": "選択肢A（正解）"},
        {"text": "選択肢B"},
        {"text": "選択肢C"},
        {"text": "選択肢D"}
      ],
      "answer": 0
    }
  ]
}
```

### 重要な仕様

1. **`answer` は常に `0`**
   - AI は常に正解を `choices[0]` に配置
   - サーバー側で選択肢をランダムにシャッフル
   - これにより AI の生成速度が向上

2. **純粋な JSON のみ**
   - 説明文やコードブロック記法（\`\`\`json）は含めない
   - 直接パース可能な形式

---

## 🧪 API テスト

### モデルの存在確認

```bash
curl http://localhost:11434/api/tags
```

### 問題生成のテスト

```bash
curl http://localhost:11434/api/generate -d '{
  "model": "codequiz:latest",
  "prompt": "作成する数: 2\nコードスニペット:\nclass User:\n    def __init__(self, name):\n        self.name = name",
  "stream": false
}' | jq -r '.response' | jq '.'
```

### レスポンスの検証

```bash
# 生成されたJSONが有効か確認
curl -s http://localhost:11434/api/generate -d '{
  "model": "codequiz:latest",
  "prompt": "作成する数: 1\nコードスニペット:\ndef test(): pass",
  "stream": false
}' | jq -r '.response' | jq '.questions[0].answer'
# 出力: 0 (期待される値)
```

---

## 🔧 よく使うコマンド

```bash
# モデル一覧
ollama list

# モデルの詳細表示
ollama show codequiz:latest

# Modelfile の内容確認
ollama show --modelfile codequiz:latest

# モデルの削除
ollama rm codequiz:latest

# 対話モードで実行
ollama run codequiz:latest

# 実行中のモデル確認
ollama ps

# サービスの起動（必要な場合）
ollama serve
```

---

## 🆘 トラブルシューティング

### モデルが見つからない

```bash
# エラー: model 'codequiz:latest' not found
ollama create codequiz:latest -f ./Modelfile
```

### サービスに接続できない

```bash
# エラー: could not connect to ollama server
ollama serve
```

### メモリ不足

```bash
# Modelfile を編集してより小さいモデルを使用
# FROM gpt-oss:20b → FROM llama3.2:3b
```

または GPU を無効化:

```dockerfile
PARAMETER num_gpu 0  # CPU のみで実行
PARAMETER num_ctx 4096  # コンテキストサイズ削減
```

### JSON 解析エラー

1. `temperature` を下げる（Modelfile）
2. SYSTEM プロンプトを確認
3. 出力を直接確認: `ollama run codequiz:latest`

---

## 🎓 使用例

### Python コードの分析

```bash
ollama run codequiz:latest
```

```
>>> 作成する数: 2
コードスニペット:
from flask import Flask, request

app = Flask(__name__)

@app.route('/api/user', methods=['POST'])
def create_user():
    data = request.get_json()
    return {'status': 'created'}

if __name__ == '__main__':
    app.run(debug=True)
```

### 期待される出力

```json
{
  "questions": [
    {
      "question": "@app.routeデコレータの主な役割は何ですか？",
      "choices": [
        {"text": "URLパスと関数を紐付ける"},
        {"text": "リクエストデータを検証する"},
        {"text": "データベースに接続する"},
        {"text": "HTMLテンプレートをレンダリングする"}
      ],
      "answer": 0
    },
    {
      "question": "request.get_json()メソッドの目的は何ですか？",
      "choices": [
        {"text": "リクエストボディのJSONをパースして辞書として取得する"},
        {"text": "レスポンスをJSON形式に変換する"},
        {"text": "JSONファイルを読み込む"},
        {"text": "データベースのJSONフィールドを取得する"}
      ],
      "answer": 0
    }
  ]
}
```

---

## 🔄 モデル名の変更

デフォルトの `codequiz:latest` から変更する場合：

1. **新しい名前でモデルを作成**

```bash
ollama create myquiz:v1 -f ./Modelfile
```

2. **サーバー側のコードを更新**

```javascript
// 変更前
const modelName = "codequiz:latest";

// 変更後
const modelName = "myquiz:v1";
```

**推奨**: 特別な理由がない限り `codequiz:latest` を使用してください。

---

## 📋 システム要件

- **RAM**: 最低8GB（推奨: 16GB以上）
- **ディスク**: 10GB以上の空き容量
- **GPU**: NVIDIA/AMD GPU（オプション、CPU のみでも動作）
- **OS**: macOS, Linux, Windows

---

## 📖 参考リンク

- [Ollama 公式サイト](https://ollama.com/)
- [Ollama GitHub](https://github.com/ollama/ollama)
- [Modelfile ドキュメント](https://github.com/ollama/ollama/blob/main/docs/modelfile.md)
- [Ollama モデルライブラリ](https://ollama.com/library)

---

**最終更新**: 2024-03-23

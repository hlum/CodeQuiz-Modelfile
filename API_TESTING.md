# API テストガイド

このドキュメントでは、curlコマンドを使用してCodeQuiz AIモデルをテストする方法を説明します。

## 📋 目次

1. [基本的なテスト](#基本的なテスト)
2. [モデル確認](#モデル確認)
3. [問題生成のテスト](#問題生成のテスト)
4. [レスポンスの検証](#レスポンスの検証)
5. [高度なテスト](#高度なテスト)
6. [トラブルシューティング](#トラブルシューティング)

---

## 基本的なテスト

### 前提条件

1. Ollamaサービスが起動していること
2. CodeQuizモデルが作成されていること

#### 確認コマンド

```bash
# Ollamaサービスの確認
ollama ps

# モデルの確認
ollama list | grep codequiz
```

---

## モデル確認

### 1. サービスの接続確認

Ollamaサービスが正常に動作しているか確認します。

```bash
curl http://localhost:11434/api/tags
```

**期待されるレスポンス**:
```json
{
  "models": [
    {
      "name": "codequiz:latest",
      "modified_at": "2024-03-23T12:34:56.789Z",
      "size": 12345678900,
      "digest": "sha256:abc123...",
      "details": {
        "format": "gguf",
        "family": "llama",
        "families": ["llama"],
        "parameter_size": "20B",
        "quantization_level": "Q4_0"
      }
    }
  ]
}
```

### 2. モデルの存在確認

特定のモデルが存在するか確認します。

```bash
curl http://localhost:11434/api/tags | grep "codequiz:latest"
```

### 3. jqを使用した整形表示

```bash
curl -s http://localhost:11434/api/tags | jq '.'
```

**jqがインストールされていない場合**:

```bash
# macOS
brew install jq

# Ubuntu / Debian
sudo apt-get install jq

# Windows (Chocolatey)
choco install jq
```

---

## 問題生成のテスト

### API エンドポイント

CodeQuizモデルは以下のAPIエンドポイントで使用できます：

- `/api/generate` - 単一プロンプトでの生成
- `/api/chat` - 会話形式での生成

### 1. 基本的な問題生成

#### エンドポイント: `/api/generate`

```bash
curl http://localhost:11434/api/generate -d '{
  "model": "codequiz:latest",
  "prompt": "作成する数: 2\nコードスニペット:\ndef hello():\n    print(\"Hello, World!\")",
  "stream": false
}'
```

**パラメータ説明**:
- `model`: 使用するモデル名（`codequiz:latest`固定）
- `prompt`: 入力プロンプト（問題数 + コードスニペット）
- `stream`: ストリーミング無効（`false`で完全なレスポンスを取得）

#### レスポンス例

```json
{
  "model": "codequiz:latest",
  "created_at": "2024-03-23T12:34:56.789Z",
  "response": "{\"questions\":[{\"question\":\"print関数の主な目的は何ですか？\",\"choices\":[{\"text\":\"標準出力にテキストを表示する\"},{\"text\":\"ファイルにテキストを書き込む\"},{\"text\":\"変数の値を保存する\"},{\"text\":\"関数を定義する\"}],\"answer\":0}]}",
  "done": true,
  "context": [...],
  "total_duration": 5000000000,
  "load_duration": 1000000000,
  "prompt_eval_count": 50,
  "prompt_eval_duration": 2000000000,
  "eval_count": 100,
  "eval_duration": 2000000000
}
```

### 2. 実用的なテスト例

#### Python コードのテスト

```bash
curl http://localhost:11434/api/generate -d '{
  "model": "codequiz:latest",
  "prompt": "作成する数: 1\nコードスニペット:\nclass User:\n    def __init__(self, name):\n        self.name = name\n\n    def greet(self):\n        return f\"Hello, {self.name}\"",
  "stream": false
}'
```

#### JavaScript コードのテスト

```bash
curl http://localhost:11434/api/generate -d '{
  "model": "codequiz:latest",
  "prompt": "作成する数: 1\nコードスニペット:\nconst express = require(\"express\");\nconst app = express();\n\napp.get(\"/\", (req, res) => {\n    res.send(\"Hello World\");\n});\n\napp.listen(3000);",
  "stream": false
}'
```

#### Java コードのテスト

```bash
curl http://localhost:11434/api/generate -d '{
  "model": "codequiz:latest",
  "prompt": "作成する数: 2\nコードスニペット:\npublic class Calculator {\n    public int add(int a, int b) {\n        return a + b;\n    }\n}",
  "stream": false
}'
```

### 3. ファイルから読み込んでテスト

#### テストプロンプトファイルの作成

`test_prompt.txt` を作成:
```
作成する数: 3
コードスニペット:
<?php
$dsn = "mysql:host=localhost;dbname=shop";
$username = "root";
$password = "";

try {
    $pdo = new PDO($dsn, $username, $password);
    $stmt = $pdo->prepare("SELECT * FROM users WHERE id = ?");
    $stmt->execute([$userId]);
} catch (PDOException $e) {
    echo "Connection failed: " . $e->getMessage();
}
?>
```

#### curlでファイルを使用

```bash
PROMPT=$(cat test_prompt.txt)
curl http://localhost:11434/api/generate -d "{
  \"model\": \"codequiz:latest\",
  \"prompt\": \"$PROMPT\",
  \"stream\": false
}"
```

### 4. チャット形式でのテスト

#### エンドポイント: `/api/chat`

```bash
curl http://localhost:11434/api/chat -d '{
  "model": "codequiz:latest",
  "messages": [
    {
      "role": "user",
      "content": "作成する数: 1\nコードスニペット:\nimport React from \"react\";\n\nfunction App() {\n    return <div>Hello</div>;\n}"
    }
  ],
  "stream": false
}'
```

---

## レスポンスの検証

### 1. JSONの抽出と整形

#### response フィールドの抽出

```bash
curl -s http://localhost:11434/api/generate -d '{
  "model": "codequiz:latest",
  "prompt": "作成する数: 1\nコードスニペット:\ndef test(): pass",
  "stream": false
}' | jq -r '.response'
```

#### JSON の妥当性確認

```bash
curl -s http://localhost:11434/api/generate -d '{
  "model": "codequiz:latest",
  "prompt": "作成する数: 1\nコードスニペット:\ndef test(): pass",
  "stream": false
}' | jq -r '.response' | jq '.'
```

**期待される出力**:
```json
{
  "questions": [
    {
      "question": "...",
      "choices": [
        {"text": "..."},
        {"text": "..."},
        {"text": "..."},
        {"text": "..."}
      ],
      "answer": 0
    }
  ]
}
```

### 2. 出力フォーマットの検証

#### スクリプトでの自動検証

`validate_response.sh` を作成:

```bash
#!/bin/bash

RESPONSE=$(curl -s http://localhost:11434/api/generate -d '{
  "model": "codequiz:latest",
  "prompt": "作成する数: 1\nコードスニペット:\ndef test(): pass",
  "stream": false
}' | jq -r '.response')

# JSON として妥当か確認
if echo "$RESPONSE" | jq '.' > /dev/null 2>&1; then
  echo "✅ Valid JSON"
else
  echo "❌ Invalid JSON"
  exit 1
fi

# questions フィールドが存在するか
if echo "$RESPONSE" | jq -e '.questions' > /dev/null 2>&1; then
  echo "✅ Has 'questions' field"
else
  echo "❌ Missing 'questions' field"
  exit 1
fi

# answer が 0 か確認
ANSWER=$(echo "$RESPONSE" | jq -r '.questions[0].answer')
if [ "$ANSWER" == "0" ]; then
  echo "✅ Answer is 0"
else
  echo "❌ Answer is not 0: $ANSWER"
  exit 1
fi

echo "✅ All validations passed!"
```

実行:
```bash
chmod +x validate_response.sh
./validate_response.sh
```

### 3. Python での検証

`validate_response.py` を作成:

```python
#!/usr/bin/env python3
import json
import requests

def test_codequiz():
    url = "http://localhost:11434/api/generate"
    payload = {
        "model": "codequiz:latest",
        "prompt": "作成する数: 1\nコードスニペット:\ndef test(): pass",
        "stream": False
    }
    
    response = requests.post(url, json=payload)
    data = response.json()
    
    # response フィールドを抽出
    questions_json = data.get("response", "")
    
    # JSON として解析
    try:
        questions = json.loads(questions_json)
        print("✅ Valid JSON")
    except json.JSONDecodeError as e:
        print(f"❌ Invalid JSON: {e}")
        return False
    
    # フォーマット検証
    if "questions" not in questions:
        print("❌ Missing 'questions' field")
        return False
    print("✅ Has 'questions' field")
    
    for i, q in enumerate(questions["questions"]):
        # 必須フィールドの確認
        if "question" not in q or "choices" not in q or "answer" not in q:
            print(f"❌ Question {i}: Missing required field")
            return False
        
        # answer が 0 か確認
        if q["answer"] != 0:
            print(f"❌ Question {i}: Answer is not 0")
            return False
        
        # choices が 4 つあるか確認
        if len(q["choices"]) != 4:
            print(f"❌ Question {i}: Should have 4 choices, got {len(q['choices'])}")
            return False
    
    print("✅ All validations passed!")
    print(json.dumps(questions, ensure_ascii=False, indent=2))
    return True

if __name__ == "__main__":
    test_codequiz()
```

実行:
```bash
chmod +x validate_response.py
./validate_response.py
```

---

## 高度なテスト

### 1. パラメータのカスタマイズ

```bash
curl http://localhost:11434/api/generate -d '{
  "model": "codequiz:latest",
  "prompt": "作成する数: 1\nコードスニペット:\ndef test(): pass",
  "stream": false,
  "options": {
    "temperature": 0.5,
    "num_predict": 500,
    "top_p": 0.8
  }
}'
```

### 2. 複数リクエストのベンチマーク

#### 連続テスト

```bash
#!/bin/bash

for i in {1..5}; do
  echo "Test $i:"
  time curl -s http://localhost:11434/api/generate -d '{
    "model": "codequiz:latest",
    "prompt": "作成する数: 1\nコードスニペット:\ndef test(): pass",
    "stream": false
  }' | jq -r '.response' | jq '.'
  echo "---"
done
```

### 3. ストリーミングレスポンスのテスト

```bash
curl http://localhost:11434/api/generate -d '{
  "model": "codequiz:latest",
  "prompt": "作成する数: 1\nコードスニペット:\ndef test(): pass",
  "stream": true
}'
```

**注意**: ストリーミングモードでは、レスポンスが複数のJSONオブジェクトとして返されます。

---

## トラブルシューティング

### エラー: Connection refused

```bash
curl http://localhost:11434/api/tags
# curl: (7) Failed to connect to localhost port 11434: Connection refused
```

**原因**: Ollamaサービスが起動していない

**解決方法**:
```bash
ollama serve
```

### エラー: Model not found

```json
{
  "error": "model 'codequiz:latest' not found"
}
```

**原因**: モデルが作成されていない

**解決方法**:
```bash
ollama create codequiz:latest -f ./Modelfile
```

### エラー: Invalid JSON response

**原因**: モデルの出力がJSON形式でない

**解決方法**:
1. `temperature` を下げる（Modelfile）
2. SYSTEM プロンプトを確認
3. 別のベースモデルを試す

### タイムアウトエラー

```bash
curl: (28) Operation timed out
```

**原因**: モデルの生成に時間がかかりすぎている

**解決方法**:
```bash
# タイムアウトを延長
curl --max-time 300 http://localhost:11434/api/generate -d '{...}'
```

---

## クイックリファレンス

### よく使うコマンド

```bash
# モデル確認
curl http://localhost:11434/api/tags

# 基本的な問題生成
curl http://localhost:11434/api/generate -d '{
  "model": "codequiz:latest",
  "prompt": "作成する数: 1\nコードスニペット:\ndef test(): pass",
  "stream": false
}'

# レスポンスの整形
curl -s http://localhost:11434/api/generate -d '{
  "model": "codequiz:latest",
  "prompt": "作成する数: 1\nコードスニペット:\ndef test(): pass",
  "stream": false
}' | jq -r '.response' | jq '.'
```

---

## 次のステップ

✅ APIテストをマスターしたら：

1. [Modelfile リファレンス](./MODELFILE_REFERENCE.md) - モデルの最適化
2. [Ollama コマンド](./OLLAMA_COMMANDS.md) - 管理コマンドの詳細
3. [トラブルシューティング](./TROUBLESHOOTING.md) - 問題解決ガイド
4. [メインREADME](./README.md) - プロジェクト概要に戻る

---

**最終更新**: 2024-03-23

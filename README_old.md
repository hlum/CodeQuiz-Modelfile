# CodeQuiz AI サーバー ドキュメント

このドキュメントは、Ollama を使用したカスタムAIモデルサーバーのセットアップと管理方法を説明します。

## 📚 目次

1. [概要](#概要)
2. [前提条件](#前提条件)
3. [クイックスタート](#クイックスタート)
4. [Modelfile の設定](#modelfile-の設定)
5. [Ollama コマンドリファレンス](#ollama-コマンドリファレンス)
6. [API エンドポイント](#api-エンドポイント)
7. [出力フォーマット要件](#出力フォーマット要件)
8. [トラブルシューティング](#トラブルシューティング)

---

## 概要

このAIサーバーは以下の技術スタックで構成されています：

- **Ollama**: ローカルLLMの実行環境
- **カスタム Modelfile**: プログラミングクイズ生成に特化したモデル設定
- **モデル名**: `codequiz:latest` (固定)

サーバーAPIは特定のJSON形式を期待しており、この形式に従わないと処理が失敗します。

---

## 前提条件

### Ollama のインストール

1. **macOS / Linux**
   ```bash
   curl -fsSL https://ollama.com/install.sh | sh
   ```

2. **Windows**
   [Ollama 公式サイト](https://ollama.com/download)からインストーラーをダウンロードしてください。

3. **インストール確認**
   ```bash
   ollama --version
   ```

---

## クイックスタート

### 1. モデルの作成

```bash
# Modelfile が配置されているディレクトリで実行
ollama create codequiz:latest -f ./Modelfile
```

### 2. モデルの実行確認

```bash
ollama run codequiz:latest
```

対話モードで動作を確認できます。終了するには `/bye` を入力します。

### 3. モデルの可用性確認

以下のAPIエンドポイントでモデルが登録されているか確認できます：

```bash
curl http://localhost:11434/api/tags
```

レスポンス例：
```json
{
  "models": [
    {
      "name": "codequiz:latest",
      "modified_at": "2024-03-23T12:34:56.789Z",
      "size": 12345678900
    }
  ]
}
```

---

## Modelfile の設定

### 基本構造

Modelfile は以下の主要な要素で構成されています：

```dockerfile
# ベースモデルの指定（必須）
FROM <モデル名>:<タグ>

# パラメータの設定
PARAMETER <パラメータ名> <値>

# システムメッセージの設定
SYSTEM """
<プロンプト設定>
"""
```

### ベースモデルの変更

**重要**: ベースモデルを変更する場合は、以下の行を編集してください：

```dockerfile
FROM gpt-oss:20b
```

**変更例**:
```dockerfile
# Llama 3.2 を使用する場合
FROM llama3.2

# Mistral を使用する場合  
FROM mistral:latest

# Gemma を使用する場合
FROM gemma2:9b
```

利用可能なモデルは [Ollama ライブラリ](https://ollama.com/library)で確認できます。

### 利用可能なパラメータ

現在の Modelfile で使用されているパラメータ：

| パラメータ | 現在の値 | 説明 |
|-----------|---------|------|
| `temperature` | 0.7 | 創造性の度合い（0.0～2.0、高いほどランダム） |
| `num_ctx` | 16384 | コンテキストウィンドウのサイズ（トークン数） |
| `top_p` | 0.9 | nucleus sampling（0.0～1.0） |
| `repeat_penalty` | 1.1 | 繰り返しのペナルティ（1.0以上） |
| `num_gpu` | 99 | GPU レイヤー数（-1 = すべて） |
| `num_thread` | 10 | CPU スレッド数 |

### その他の有用なパラメータ

```dockerfile
# トークン生成数の上限を設定
PARAMETER num_predict 512

# シード値を固定（再現性のため）
PARAMETER seed 42

# 停止シーケンスの設定
PARAMETER stop "END"

# Top-k サンプリング
PARAMETER top_k 40

# 最小確率閾値
PARAMETER min_p 0.05

# 繰り返し防止の範囲
PARAMETER repeat_last_n 64
```

#### パラメータ詳細ガイド

**temperature（温度）**
- **低い値（0.1～0.5）**: より決定論的で一貫性のある出力
- **中程度（0.7～0.9）**: バランスの取れた創造性
- **高い値（1.0～2.0）**: より創造的でランダムな出力

**num_ctx（コンテキスト長）**
- **2048**: 短い会話や簡単なタスク
- **4096**: 標準的な用途
- **8192～16384**: 長いコードやドキュメントの分析
- **注意**: 大きい値ほどメモリ消費が増加します

**top_p（Nucleus Sampling）**
- 累積確率がこの値を超えるまでトークンを考慮
- **0.9～0.95**: 多様性を維持しつつ品質を確保
- **0.5～0.8**: より保守的な出力

**repeat_penalty（繰り返しペナルティ）**
- **1.0**: ペナルティなし
- **1.1～1.3**: 適度な繰り返し防止（推奨）
- **1.5以上**: 強い繰り返し防止（単調になる可能性）

### パラメータ変更例

```dockerfile
# よりクリエイティブな設定
PARAMETER temperature 1.2
PARAMETER top_p 0.95
PARAMETER top_k 100

# より一貫性のある設定
PARAMETER temperature 0.3
PARAMETER top_p 0.8
PARAMETER repeat_penalty 1.2

# 長いコンテキスト用の設定
PARAMETER num_ctx 32768
PARAMETER num_predict 2048
```

### モデルの再作成

Modelfile を編集した後、変更を反映するには：

```bash
# 既存のモデルを削除
ollama rm codequiz:latest

# 新しい設定でモデルを再作成
ollama create codequiz:latest -f ./Modelfile
```

---

## Ollama コマンドリファレンス

### モデル管理

#### モデルの作成・更新
```bash
ollama create <モデル名> -f <Modelfile のパス>

# 例
ollama create codequiz:latest -f ./Modelfile
```

#### モデルの一覧表示
```bash
ollama list
```

#### モデルの削除
```bash
ollama rm <モデル名>

# 例
ollama rm codequiz:latest
```

#### モデルの名前変更
```bash
ollama cp <元のモデル名> <新しいモデル名>

# 例
ollama cp codequiz:latest codequiz:v2
```

#### モデルの詳細表示
```bash
# モデルの情報を表示
ollama show <モデル名>

# Modelfile の内容を表示
ollama show --modelfile <モデル名>

# 例
ollama show codequiz:latest
ollama show --modelfile codequiz:latest
```

### サービス管理

#### Ollama サービスの起動
```bash
ollama serve
```

**注意**: macOS と Windows では通常バックグラウンドで自動起動します。

#### 実行中のモデルの確認
```bash
ollama ps
```

#### モデルの実行（対話モード）
```bash
ollama run <モデル名>

# 例
ollama run codequiz:latest
```

対話モードのコマンド：
- `/bye`: 終了
- `/clear`: 会話履歴をクリア
- `/help`: ヘルプを表示

### モデルのダウンロード

```bash
# Ollama ライブラリからモデルをダウンロード
ollama pull <モデル名>

# 例
ollama pull llama3.2
ollama pull mistral:latest
```

---

## API エンドポイント

### モデル一覧の取得

**エンドポイント**: `GET http://localhost:11434/api/tags`

**レスポンス例**:
```json
{
  "models": [
    {
      "name": "codequiz:latest",
      "modified_at": "2024-03-23T12:34:56.789Z",
      "size": 12345678900,
      "digest": "abc123...",
      "details": {
        "format": "gguf",
        "family": "llama",
        "parameter_size": "20B"
      }
    }
  ]
}
```

### 使用例

```bash
# curl での確認
curl http://localhost:11434/api/tags

# モデル名の存在確認
curl http://localhost:11434/api/tags | grep "codequiz:latest"
```

---

## 出力フォーマット要件

### 必須JSON形式

サーバーは以下の**厳密なJSON形式**を要求します。この形式に従わない出力は処理に失敗します。

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

#### 1. answer インデックスは常に 0

```json
"answer": 0
```

**理由**: 
- AIは常に正解を `choices[0]` に配置します
- サーバー側で選択肢をランダムにシャッフルします
- これによりAIの生成速度が向上します

**動作の流れ**:
1. AI が問題を生成（正解は必ず `choices[0]`）
2. サーバーが選択肢をランダムに並び替え
3. クライアントに返送

#### 2. JSON のみを出力

**正しい出力**:
```json
{
  "questions": [...]
}
```

**誤った出力**:
```markdown
```json
{
  "questions": [...]
}
```
```

または

```text
こちらが生成された問題です：
{
  "questions": [...]
}
```

モデルは**純粋なJSON文字列のみ**を出力する必要があります。

### フォーマット確認方法

生成されたJSON が有効か確認：

```bash
# jq を使用した検証
echo '{"questions":[...]}' | jq .

# Python を使用した検証
echo '{"questions":[...]}' | python -m json.tool
```

---

## モデル名の変更について

### 現在の設定

サーバーAPIは **`codequiz:latest`** という名前のモデルを要求します。

### モデル名を変更する場合

モデル名を変更すると、サーバー側のAPIリクエストも変更する必要があります。

#### 手順:

1. **新しい名前でモデルを作成**
   ```bash
   ollama create <新しい名前> -f ./Modelfile
   
   # 例
   ollama create myquiz:v1 -f ./Modelfile
   ```

2. **サーバー側のコードを更新**
   
   サーバーコードで以下のような箇所を探し、モデル名を変更してください：
   
   ```javascript
   // 変更前
   const modelName = "codequiz:latest";
   
   // 変更後
   const modelName = "myquiz:v1";
   ```

3. **既存のモデルを削除（オプション）**
   ```bash
   ollama rm codequiz:latest
   ```

**推奨**: 特別な理由がない限り、デフォルトの `codequiz:latest` を使用してください。

---

## トラブルシューティング

### 問題1: モデルが見つからない

**症状**: `Error: model 'codequiz:latest' not found`

**解決方法**:
```bash
# モデルの一覧を確認
ollama list

# モデルが存在しない場合は作成
ollama create codequiz:latest -f ./Modelfile
```

### 問題2: Ollama サービスに接続できない

**症状**: `Error: could not connect to ollama server`

**解決方法**:
```bash
# Ollama サービスを起動
ollama serve

# または別のターミナルで
ollama ps
```

**ポート確認**:
```bash
# デフォルトポート（11434）が使用中か確認
lsof -i :11434
```

### 問題3: JSON 解析エラー

**症状**: サーバーが JSON を解析できない

**確認事項**:
1. モデルの出力が純粋な JSON か確認
2. SYSTEM メッセージに「JSON のみを出力」の指示があるか確認
3. `temperature` が高すぎないか確認（推奨: 0.7 以下）

**デバッグ方法**:
```bash
# モデルを直接実行して出力を確認
ollama run codequiz:latest "プログラミング問題を1問生成してください"
```

### 問題4: メモリ不足

**症状**: モデルの読み込みに失敗、OOM エラー

**解決方法**:

1. **より小さいモデルを使用**
   ```dockerfile
   FROM llama3.2:3b  # 8bや20bの代わりに
   ```

2. **GPU メモリを調整**
   ```dockerfile
   PARAMETER num_gpu 0  # CPU のみで実行
   ```

3. **コンテキストサイズを削減**
   ```dockerfile
   PARAMETER num_ctx 4096  # 16384 から削減
   ```

### 問題5: 出力が期待と異なる

**対処法**:

1. **temperature を下げる**（より一貫性のある出力）
   ```dockerfile
   PARAMETER temperature 0.5
   ```

2. **SYSTEM メッセージを確認**
   ```bash
   ollama show --modelfile codequiz:latest
   ```

3. **別のベースモデルを試す**
   ```dockerfile
   FROM llama3.2  # より新しいモデル
   ```

### 問題6: モデルの更新が反映されない

**解決方法**:
```bash
# 1. 古いモデルを完全に削除
ollama rm codequiz:latest

# 2. 新しいモデルを作成
ollama create codequiz:latest -f ./Modelfile

# 3. 実行中のモデルがあれば停止
ollama ps  # PIDを確認
# APIサーバーを再起動
```

---

## 付録

### A. Modelfile の完全な例

```dockerfile
# ベースモデル
FROM llama3.2

# パフォーマンスパラメータ
PARAMETER temperature 0.7
PARAMETER num_ctx 8192
PARAMETER top_p 0.9
PARAMETER repeat_penalty 1.1

# GPU設定
PARAMETER num_gpu 99
PARAMETER num_thread 8

# 生成制限
PARAMETER num_predict 1024

# システムプロンプト
SYSTEM """
あなたはプログラミングクイズを生成する専門家です。
JSON 形式のみを出力してください。

出力形式：
{
  "questions": [
    {
      "question": "質問文",
      "choices": [
        {"text": "正解の選択肢"},
        {"text": "誤答1"},
        {"text": "誤答2"},
        {"text": "誤答3"}
      ],
      "answer": 0
    }
  ]
}
"""
```

### B. 推奨モデル

| モデル | サイズ | 用途 |
|-------|-------|------|
| `llama3.2:3b` | 3B | 軽量、高速 |
| `llama3.2:8b` | 8B | バランス型（推奨） |
| `llama3.2:70b` | 70B | 高品質 |
| `mistral:7b` | 7B | 効率的 |
| `gemma2:9b` | 9B | Google製 |

### C. 参考リンク

- [Ollama 公式ドキュメント](https://github.com/ollama/ollama)
- [Modelfile リファレンス](https://github.com/ollama/ollama/blob/main/docs/modelfile.md)
- [Ollama モデルライブラリ](https://ollama.com/library)
- [Ollama API ドキュメント](https://github.com/ollama/ollama/blob/main/docs/api.md)

---

## サポート

問題が解決しない場合は、以下の情報を添えてお問い合わせください：

1. Ollama のバージョン（`ollama --version`）
2. 使用しているモデルとタグ
3. エラーメッセージの全文
4. Modelfile の内容
5. `ollama ps` の出力

---

**最終更新**: 2024-03-23

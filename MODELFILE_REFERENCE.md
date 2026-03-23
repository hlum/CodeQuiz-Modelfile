# Modelfile リファレンス

このドキュメントでは、CodeQuizプロジェクトで使用されているModelfileの詳細を説明します。

## 📋 目次

1. [プロジェクトの目的](#プロジェクトの目的)
2. [Modelfileの構造](#modelfileの構造)
3. [ベースモデルの設定](#ベースモデルの設定)
4. [パラメータの詳細](#パラメータの詳細)
5. [SYSTEMプロンプトの解説](#systemプロンプトの解説)
6. [MESSAGEによる Few-Shot Learning](#messageによる-few-shot-learning)
7. [カスタマイズ方法](#カスタマイズ方法)

---

## プロジェクトの目的

**CodeQuiz** は、プログラミングの概念的理解を測るクイズ問題を自動生成するAIモデルです。

### 主な特徴

- **対象**: プログラミング学習者の概念理解をテスト
- **言語**: すべての問題・選択肢は日本語
- **出力形式**: 厳密なJSON形式（バリデーション可能）
- **制約**: カスタム名禁止、選択肢長さの均一化など
- **正解位置**: 常に `choices[0]`（サーバー側でシャッフル）

---

## Modelfileの構造

```dockerfile
FROM <ベースモデル>
PARAMETER <パラメータ名> <値>
SYSTEM """<システムプロンプト>"""
MESSAGE <role> """<メッセージ>"""
```

### 実際の設定

現在のModelfileは以下の要素で構成されています：

1. **FROM**: ベースモデルの指定
2. **PARAMETER** (7個): モデルの動作を制御
3. **SYSTEM**: 1308行の詳細なプロンプト（役割、制約、出力形式）
4. **MESSAGE**: Few-Shot学習用の例（複数セット）

---

## ベースモデルの設定

### 現在の設定

```dockerfile
FROM gpt-oss:20b
```

**gpt-oss:20b の特徴**:
- パラメータ数: 20B（200億）
- サイズ: 約12GB
- 性能: 高品質な日本語生成が可能
- 要件: 16GB以上のRAM推奨

### ベースモデルの変更方法

**重要**: ベースモデルを変更する場合は、Modelfileの1行目を編集してください。

#### 代替モデルの選択肢

| モデル名 | サイズ | RAM要件 | 特徴 |
|---------|-------|---------|------|
| `llama3.2:3b` | 2GB | 8GB | 軽量、高速 |
| `llama3.2` | 4.7GB | 8GB | バランス型 |
| `mistral:7b` | 4.1GB | 8GB | 効率的 |
| `llama3.2:70b` | 40GB | 64GB | 最高品質 |
| `gemma2:9b` | 5.4GB | 16GB | Google製 |

#### 変更例

```dockerfile
# より小さいモデルに変更
FROM llama3.2:3b

# より新しいモデルに変更
FROM llama3.2

# より大きく高品質なモデルに変更
FROM llama3.2:70b
```

#### 変更後の手順

```bash
# 1. 新しいベースモデルをダウンロード
ollama pull llama3.2

# 2. 既存モデルを削除
ollama rm codequiz:latest

# 3. 新しいモデルを作成
ollama create codequiz:latest -f ./Modelfile
```

---

## パラメータの詳細

### 現在のパラメータ設定

```dockerfile
PARAMETER temperature 0.7
PARAMETER num_ctx 16384
PARAMETER top_p 0.9
PARAMETER repeat_penalty 1.1
PARAMETER num_gpu 99
PARAMETER num_thread 10
```

### パラメータ解説

#### 1. temperature（温度）: 0.7

**役割**: 生成のランダム性を制御

```dockerfile
PARAMETER temperature 0.7
```

- **範囲**: 0.0 ～ 2.0
- **現在の設定**: 0.7（中程度の創造性）
- **効果**:
  - **低い（0.1～0.5）**: より決定論的で一貫性のある出力
  - **中程度（0.6～0.9）**: バランスの取れた創造性 ← **推奨**
  - **高い（1.0～2.0）**: より創造的でランダム（予測不可能）

**CodeQuizでの推奨値**: 0.5～0.8
- 問題の多様性は欲しいが、フォーマットの一貫性も重要

#### 2. num_ctx（コンテキスト長）: 16384

**役割**: 一度に処理できるトークン数

```dockerfile
PARAMETER num_ctx 16384
```

- **現在の設定**: 16384トークン（約12,000語）
- **効果**: 長いコードスニペットを分析可能
- **トレードオフ**: 大きいほどメモリ消費が増加

**推奨値**:
- **軽量設定**: 4096（短いコード用）
- **標準設定**: 8192（通常のコード用）
- **高性能設定**: 16384（長いコード用）← **現在**
- **最大設定**: 32768（非常に長いコード用）

#### 3. top_p（Nucleus Sampling）: 0.9

**役割**: 累積確率に基づくトークン選択

```dockerfile
PARAMETER top_p 0.9
```

- **範囲**: 0.0 ～ 1.0
- **現在の設定**: 0.9（高い多様性）
- **効果**: 累積確率が0.9になるまでのトークンから選択

**推奨値**:
- **保守的**: 0.5～0.7（より予測可能）
- **バランス**: 0.8～0.9 ← **現在**
- **多様性重視**: 0.95～1.0

#### 4. repeat_penalty（繰り返しペナルティ）: 1.1

**役割**: 同じ内容の繰り返しを防ぐ

```dockerfile
PARAMETER repeat_penalty 1.1
```

- **範囲**: 1.0 ～ 2.0
- **現在の設定**: 1.1（軽いペナルティ）
- **効果**: 値が大きいほど繰り返しを強く抑制

**推奨値**:
- **ペナルティなし**: 1.0
- **軽度**: 1.1～1.2 ← **現在**
- **中程度**: 1.3～1.5
- **強力**: 1.5以上（単調になる可能性）

#### 5. num_gpu（GPU層数）: 99

**役割**: GPUで処理するレイヤー数

```dockerfile
PARAMETER num_gpu 99
```

- **現在の設定**: 99（実質的に全レイヤー）
- **-1**: すべてのレイヤー（自動検出）
- **0**: GPU無効（CPU のみ）
- **1～N**: 指定したレイヤー数のみGPUで処理

**用途別設定**:
- **GPU使用**: 99 または -1 ← **現在**
- **ハイブリッド**: 20～50（GPU+CPU）
- **CPU のみ**: 0

#### 6. num_thread（スレッド数）: 10

**役割**: CPU処理のスレッド数

```dockerfile
PARAMETER num_thread 10
```

- **現在の設定**: 10スレッド
- **推奨**: CPUコア数の0.5～1倍
- **例**:
  - 8コアCPU → 4～8スレッド
  - 16コアCPU → 8～16スレッド

---

## その他の有用なパラメータ

Modelfileに追加できる追加パラメータ：

### num_predict（生成トークン数制限）

```dockerfile
PARAMETER num_predict 2048
```

- **用途**: 生成される最大トークン数を制限
- **デフォルト**: -1（無制限）
- **推奨**: 1024～2048（長すぎる出力を防ぐ）

### seed（シード値）

```dockerfile
PARAMETER seed 42
```

- **用途**: 再現可能な出力
- **デフォルト**: 0（ランダム）
- **使用例**: テスト時に同じ出力を得たい場合

### stop（停止シーケンス）

```dockerfile
PARAMETER stop "END"
PARAMETER stop "###"
```

- **用途**: 特定の文字列で生成を停止
- **使用例**: フォーマット境界の定義

### top_k（Top-K サンプリング）

```dockerfile
PARAMETER top_k 40
```

- **範囲**: 1 ～ 100
- **デフォルト**: 40
- **効果**: 上位K個のトークンから選択

### min_p（最小確率閾値）

```dockerfile
PARAMETER min_p 0.05
```

- **範囲**: 0.0 ～ 1.0
- **用途**: top_pの代替、品質と多様性のバランス

---

## SYSTEMプロンプトの解説

### プロンプトの構造

現在のSYSTEMプロンプトは1308行に及び、以下の要素で構成されています：

```
1. 役割定義
2. 必須制約（カスタム名禁止、選択肢長さ均一化）
3. 出題観点（5つの優先順位）
4. 出力品質基準
5. 入力・出力形式
6. 生成後の自己チェック
7. JSON スキーマ
```

### 主要な制約

#### 制約1: カスタム名禁止

```
問題文・選択肢に、コードスニペット固有の名前を含めてはならない。
```

**禁止例**:
- ❌ "executeQuery メソッドは〜"
- ❌ "MySQLClassRepository のコンストラクタは〜"

**許可例**:
- ✅ "リポジトリクラスのコンストラクタで依存を注入する目的は〜"
- ✅ "プリペアドステートメントを使う主な理由は〜"

#### 制約2: 選択肢の長さを均一にする

```
- 正解だけが他より極端に長くなってはならない
- すべての選択肢をほぼ同じ文字数・粒度に揃えること
```

### 出題観点（優先順位順）

1. フレームワークのインターフェース・クラス・属性を実装/継承する理由
2. ライブラリ・フレームワークの特定機能の目的と使い方
3. アノテーション・デコレータの役割と効果
4. フレームワークのプロトコル・インターフェース実装が必要な理由
5. 設計パターンやアーキテクチャの選択理由

### 出力形式要件

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

**重要な仕様**:
- `answer` は常に `0`
- 正解は必ず `choices[0]` に配置
- JSONのみを出力（説明文やコードブロック記法は不可）

---

## MESSAGEによる Few-Shot Learning

### Few-Shot Learningとは

MESSAGEコマンドを使用して、モデルに具体的な例を示します。これにより、モデルは期待される出力形式と品質を学習します。

### 実装されている例

Modelfileには複数のMESSAGEペアが含まれています：

```dockerfile
MESSAGE assistant """
{
  "questions": [
    {
      "question": "SwiftUIでUIKitのビューを使用する際に、なぜUIViewRepresentableプロトコルを実装する必要がありますか？",
      "choices": [
        {"text": "SwiftUIとUIKitのビューを橋渡しするため"},
        {"text": "ビューのパフォーマンスを向上させるため"},
        {"text": "メモリリークを防ぐため"},
        {"text": "アニメーションを有効にするため"}
      ],
      "answer": 0
    }
  ]
}
"""

MESSAGE user """
作成する数: 2
コードスニペット:
[コード例]
"""
```

### 例の効果

- **フォーマット学習**: JSON構造を正確に理解
- **品質基準**: 適切な問題の難易度と表現を学習
- **制約遵守**: カスタム名禁止などのルールを実践的に学習

---

## カスタマイズ方法

### 1. より速く・軽量にする

```dockerfile
FROM llama3.2:3b
PARAMETER temperature 0.5
PARAMETER num_ctx 4096
PARAMETER num_gpu 0
PARAMETER num_thread 4
```

### 2. より高品質にする

```dockerfile
FROM llama3.2:70b
PARAMETER temperature 0.6
PARAMETER num_ctx 32768
PARAMETER top_p 0.85
PARAMETER repeat_penalty 1.2
```

### 3. より一貫性のある出力

```dockerfile
PARAMETER temperature 0.3
PARAMETER top_p 0.7
PARAMETER seed 42
PARAMETER num_predict 2048
```

### 4. メモリ不足時の対処

```dockerfile
FROM llama3.2:3b  # より小さいモデル
PARAMETER num_ctx 4096  # コンテキスト削減
PARAMETER num_gpu 0  # GPU無効化
```

### 5. CPU のみで実行

```dockerfile
PARAMETER num_gpu 0
PARAMETER num_thread 8
PARAMETER num_ctx 8192
```

---

## モデルの再作成

Modelfileを編集した後、必ず再作成してください：

```bash
# 1. 既存モデルを削除
ollama rm codequiz:latest

# 2. 新しいモデルを作成
ollama create codequiz:latest -f ./Modelfile

# 3. 確認
ollama show codequiz:latest
```

---

## 次のステップ

✅ Modelfileの理解が深まったら：

1. [Ollama コマンド](./OLLAMA_COMMANDS.md) - モデル管理の詳細
2. [API テスト](./API_TESTING.md) - curlでモデルをテストする
3. [トラブルシューティング](./TROUBLESHOOTING.md) - 問題解決ガイド
4. [メインREADME](./README.md) - プロジェクト概要に戻る

---

**最終更新**: 2024-03-23

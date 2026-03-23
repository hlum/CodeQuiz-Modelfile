# Ollama コマンドリファレンス

このドキュメントでは、Ollamaのすべてのコマンドと使用方法を説明します。

## 📋 目次

1. [モデル管理](#モデル管理)
2. [モデル実行](#モデル実行)
3. [サービス管理](#サービス管理)
4. [情報確認](#情報確認)
5. [高度な使用方法](#高度な使用方法)

---

## モデル管理

### create - モデルの作成

Modelfileから新しいモデルを作成します。

#### 構文

```bash
ollama create <モデル名> -f <Modelfileのパス>
```

#### 例

```bash
# 基本的な使用
ollama create codequiz:latest -f ./Modelfile

# カスタムタグ付き
ollama create codequiz:v2 -f ./Modelfile

# 絶対パス指定
ollama create codequiz:latest -f /path/to/Modelfile
```

#### オプション

| オプション | 説明 |
|-----------|------|
| `-f, --file` | Modelfile のパス（必須） |

#### 出力例

```
transferring model data
using existing layer sha256:abc123...
using existing layer sha256:def456...
writing manifest
success
```

---

### pull - モデルのダウンロード

Ollamaライブラリから公開モデルをダウンロードします。

#### 構文

```bash
ollama pull <モデル名>[:<タグ>]
```

#### 例

```bash
# 最新版をダウンロード
ollama pull llama3.2

# 特定のタグをダウンロード
ollama pull llama3.2:7b

# 特定のサイズ
ollama pull mistral:7b-instruct
```

#### 進捗表示

```
pulling manifest
pulling 00e1ca35e149... 100% ████████████████ 4.7 GB
pulling 8c17c2ebb0ea... 100% ████████████████ 7.0 KB
pulling 7c23fb36d801... 100% ████████████████ 4.8 KB
pulling 2e0493f67d0c... 100% ████████████████   59 B
pulling fa304d675061... 100% ████████████████   91 B
pulling 42ba7f8a01dd... 100% ████████████████  557 B
verifying sha256 digest
writing manifest
removing any unused layers
success
```

---

### rm - モデルの削除

ローカルのモデルを削除します。

#### 構文

```bash
ollama rm <モデル名>
```

#### 例

```bash
# 単一モデルの削除
ollama rm codequiz:latest

# 特定のタグの削除
ollama rm llama3.2:3b

# 複数のモデルを削除
ollama rm model1 model2 model3
```

#### 確認メッセージ

```
deleted 'codequiz:latest'
```

**注意**: 削除したモデルは復元できません。必要に応じて再作成が必要です。

---

### cp - モデルのコピー

既存のモデルを新しい名前でコピーします。

#### 構文

```bash
ollama cp <元のモデル名> <新しいモデル名>
```

#### 例

```bash
# バックアップの作成
ollama cp codequiz:latest codequiz:backup

# バージョン管理
ollama cp codequiz:latest codequiz:v1.0

# 実験用のコピー
ollama cp codequiz:latest codequiz:experimental
```

#### 用途

- **バックアップ**: 変更前にコピーを作成
- **バージョン管理**: 異なるバージョンを保持
- **実験**: 設定を変更してテスト

---

### list - モデルの一覧表示

ローカルに保存されているすべてのモデルを表示します。

#### 構文

```bash
ollama list
```

別名:
```bash
ollama ls
```

#### 出力例

```
NAME                    ID              SIZE      MODIFIED
codequiz:latest         abc123def456    12 GB     2 hours ago
llama3.2:latest         xyz789abc123    4.7 GB    1 day ago
mistral:7b              def456ghi789    4.1 GB    3 days ago
```

#### 列の説明

- **NAME**: モデル名とタグ
- **ID**: モデルの一意識別子（短縮版）
- **SIZE**: ディスク使用量
- **MODIFIED**: 最終更新日時

---

## モデル実行

### run - 対話的実行

モデルを対話モードで実行します。

#### 構文

```bash
ollama run <モデル名> [初期プロンプト]
```

#### 例

```bash
# 対話モードで起動
ollama run codequiz:latest

# 初期プロンプト付き
ollama run codequiz:latest "作成する数: 1
コードスニペット: def hello(): print('Hello')"
```

#### 対話モードのコマンド

| コマンド | 機能 |
|---------|------|
| `/bye` | 終了 |
| `/clear` | 会話履歴をクリア |
| `/help` | ヘルプを表示 |
| `/show` | モデル情報を表示 |
| `/set` | パラメータを設定 |
| `/load <path>` | ファイルを読み込む |
| `/save <path>` | 会話を保存 |

#### パラメータの動的設定

```bash
>>> /set parameter temperature 0.5
>>> /set parameter num_predict 100
```

#### 使用例

```bash
$ ollama run codequiz:latest

>>> 作成する数: 1
コードスニペット:
def calculate_sum(a, b):
    return a + b

[AIが問題を生成]

>>> /clear
会話履歴をクリアしました

>>> /bye
```

---

## サービス管理

### serve - Ollamaサービスの起動

OllamaのAPIサーバーを起動します。

#### 構文

```bash
ollama serve
```

#### デフォルト設定

- **ホスト**: 127.0.0.1
- **ポート**: 11434
- **プロトコル**: HTTP

#### カスタム設定

```bash
# カスタムホスト・ポート
OLLAMA_HOST=0.0.0.0:8080 ollama serve

# すべてのインターフェースでリッスン
OLLAMA_HOST=0.0.0.0:11434 ollama serve
```

#### 環境変数

| 変数名 | 説明 | デフォルト |
|-------|------|----------|
| `OLLAMA_HOST` | ホスト:ポート | 127.0.0.1:11434 |
| `OLLAMA_MODELS` | モデル保存先 | ~/.ollama/models |
| `OLLAMA_KEEP_ALIVE` | メモリ保持時間 | 5m |
| `OLLAMA_NUM_PARALLEL` | 並列リクエスト数 | 1 |
| `OLLAMA_MAX_LOADED_MODELS` | 同時ロードモデル数 | 1 |

#### バックグラウンド実行

```bash
# macOS / Linux
nohup ollama serve > ollama.log 2>&1 &

# または systemd を使用（Linux）
sudo systemctl start ollama
```

**注意**: macOS と Windows では通常、Ollamaは自動的にバックグラウンドで起動します。

---

### ps - 実行中のモデル確認

現在メモリに読み込まれているモデルを表示します。

#### 構文

```bash
ollama ps
```

#### 出力例

```
NAME                ID              SIZE      PROCESSOR    UNTIL
codequiz:latest     abc123def456    12 GB     100% GPU     4 minutes from now
```

#### 列の説明

- **NAME**: モデル名
- **ID**: モデルID
- **SIZE**: メモリ使用量
- **PROCESSOR**: GPU/CPU使用状況
- **UNTIL**: メモリから解放される予定時刻

#### メモリ管理

モデルは使用後、デフォルトで5分間メモリに保持されます。

```bash
# 保持時間を変更
OLLAMA_KEEP_ALIVE=10m ollama serve  # 10分間保持
OLLAMA_KEEP_ALIVE=0 ollama serve    # 即座に解放
```

---

## 情報確認

### show - モデル情報の表示

モデルの詳細情報を表示します。

#### 構文

```bash
ollama show <モデル名> [オプション]
```

#### 基本情報

```bash
ollama show codequiz:latest
```

出力例:
```
  Model
    architecture        llama
    parameters          20B
    context length      16384
    embedding length    4096
    quantization        Q4_0

  Parameters
    temperature    0.7
    num_ctx        16384
    top_p          0.9
    ...

  License
    [ライセンス情報]
```

#### Modelfile の表示

```bash
ollama show --modelfile codequiz:latest
```

出力例:
```dockerfile
# Modelfile generated by "ollama show"
# To build a new Modelfile based on this one, replace the FROM line with:
# FROM codequiz:latest

FROM /path/to/base/model
PARAMETER temperature 0.7
PARAMETER num_ctx 16384
...
SYSTEM """
[システムプロンプト]
"""
```

**用途**: モデルの設定をエクスポートして再利用

#### パラメータのみ表示

```bash
ollama show --parameters codequiz:latest
```

#### システムプロンプトのみ表示

```bash
ollama show --system codequiz:latest
```

#### テンプレートのみ表示

```bash
ollama show --template codequiz:latest
```

---

### --version - バージョン確認

Ollamaのバージョンを表示します。

#### 構文

```bash
ollama --version
```

または

```bash
ollama -v
```

#### 出力例

```
ollama version is 0.1.23
```

---

## 高度な使用方法

### モデルのエクスポート

#### Modelfile のエクスポート

```bash
ollama show --modelfile codequiz:latest > Modelfile.exported
```

#### 別の環境へのコピー

```bash
# 1. Modelfile をエクスポート
ollama show --modelfile codequiz:latest > codequiz.modelfile

# 2. 別のマシンでモデルを作成
ollama create codequiz:latest -f codequiz.modelfile
```

---

### バッチ処理

#### 複数のモデルを一括ダウンロード

```bash
#!/bin/bash
models=("llama3.2" "mistral:7b" "gemma2:9b")

for model in "${models[@]}"; do
  echo "Downloading $model..."
  ollama pull "$model"
done
```

#### 複数のモデルを一括削除

```bash
#!/bin/bash
# 古いモデルを削除
ollama list | grep "old-model" | awk '{print $1}' | xargs -I {} ollama rm {}
```

---

### APIを使用したプログラマティックな管理

#### curlでモデル一覧を取得

```bash
curl http://localhost:11434/api/tags
```

#### curlでモデルを実行

```bash
curl http://localhost:11434/api/generate -d '{
  "model": "codequiz:latest",
  "prompt": "作成する数: 1\nコードスニペット: def test(): pass",
  "stream": false
}'
```

---

### トラブルシューティングコマンド

#### ディスク使用量の確認

```bash
du -sh ~/.ollama/models
```

#### 未使用レイヤーのクリーンアップ

```bash
# 未使用のモデルレイヤーを自動削除
# （現在は自動的に実行されます）
```

#### ログの確認

```bash
# macOS
tail -f ~/Library/Logs/Ollama/server.log

# Linux (systemd)
journalctl -u ollama -f

# 手動起動の場合
tail -f ollama.log
```

#### ポート使用状況の確認

```bash
# macOS / Linux
lsof -i :11434

# Windows
netstat -ano | findstr :11434
```

---

## クイックリファレンス

### よく使うコマンド一覧

```bash
# モデル作成
ollama create codequiz:latest -f ./Modelfile

# モデル一覧
ollama list

# モデル実行
ollama run codequiz:latest

# モデル削除
ollama rm codequiz:latest

# モデル情報
ollama show codequiz:latest

# サービス起動
ollama serve

# 実行中のモデル
ollama ps
```

---

## 次のステップ

✅ Ollamaコマンドをマスターしたら：

1. [API テスト](./API_TESTING.md) - curlでAPIを使用する方法
2. [Modelfile リファレンス](./MODELFILE_REFERENCE.md) - モデルのカスタマイズ
3. [トラブルシューティング](./TROUBLESHOOTING.md) - 問題解決ガイド
4. [メインREADME](./README.md) - プロジェクト概要に戻る

---

**最終更新**: 2024-03-23

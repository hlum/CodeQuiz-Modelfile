# インストールガイド

このガイドでは、CodeQuiz AIサーバーに必要なOllamaのインストールとセットアップ方法を説明します。

## 📋 目次

1. [システム要件](#システム要件)
2. [Ollamaのインストール](#ollamaのインストール)
3. [ベースモデルのダウンロード](#ベースモデルのダウンロード)
4. [CodeQuizモデルの作成](#codequizモデルの作成)
5. [インストールの確認](#インストールの確認)

---

## システム要件

### 最小要件

- **RAM**: 8GB以上（推奨: 16GB以上）
- **ディスク容量**: 10GB以上の空き容量
- **OS**: macOS、Linux、Windows

### GPUサポート（オプション）

- **NVIDIA GPU**: CUDA対応（推奨）
- **Apple Silicon**: Metal対応（M1/M2/M3 Mac）
- **AMD GPU**: ROCm対応（Linux）

> **注意**: GPUがない場合でもCPUで実行可能ですが、速度が遅くなります。

---

## Ollamaのインストール

### macOS

#### 方法1: 公式インストーラー（推奨）

1. [Ollama公式サイト](https://ollama.com/download)からインストーラーをダウンロード
2. ダウンロードした `.dmg` ファイルを開く
3. Ollamaアプリケーションをアプリケーションフォルダにドラッグ
4. アプリケーションを起動

#### 方法2: ターミナルからインストール

```bash
curl -fsSL https://ollama.com/install.sh | sh
```

#### インストール確認

```bash
ollama --version
```

出力例:
```
ollama version is 0.1.23
```

### Linux

#### Ubuntu / Debian

```bash
curl -fsSL https://ollama.com/install.sh | sh
```

#### 手動インストール

```bash
# バイナリのダウンロード
curl -L https://ollama.com/download/ollama-linux-amd64 -o /usr/local/bin/ollama

# 実行権限を付与
chmod +x /usr/local/bin/ollama

# サービスの設定（systemd）
sudo useradd -r -s /bin/false -m -d /usr/share/ollama ollama
```

#### systemdサービスの設定

`/etc/systemd/system/ollama.service` を作成:

```ini
[Unit]
Description=Ollama Service
After=network-online.target

[Service]
ExecStart=/usr/local/bin/ollama serve
User=ollama
Group=ollama
Restart=always
RestartSec=3

[Install]
WantedBy=default.target
```

サービスの起動:

```bash
sudo systemctl daemon-reload
sudo systemctl enable ollama
sudo systemctl start ollama
```

### Windows

1. [Ollama公式サイト](https://ollama.com/download)から Windows インストーラーをダウンロード
2. `OllamaSetup.exe` を実行
3. インストールウィザードの指示に従う
4. インストール完了後、Ollamaは自動的にバックグラウンドで起動

#### PowerShellでの確認

```powershell
ollama --version
```

---

## ベースモデルのダウンロード

CodeQuizモデルは `gpt-oss:20b` をベースモデルとして使用します。

### モデルのダウンロード

```bash
ollama pull gpt-oss:20b
```

> **注意**: モデルのサイズは約12GBです。ダウンロードには時間がかかる場合があります。

### ダウンロードの確認

```bash
ollama list
```

出力例:
```
NAME              ID              SIZE      MODIFIED
gpt-oss:20b       abc123def456    12 GB     2 hours ago
```

### 代替モデル（オプション）

メモリやディスク容量が不足している場合、より小さいモデルも使用できます：

```bash
# Llama 3.2 (3B パラメータ - 約2GB)
ollama pull llama3.2:3b

# Llama 3.2 (8B パラメータ - 約4.7GB)
ollama pull llama3.2

# Mistral (7B パラメータ - 約4.1GB)
ollama pull mistral:7b
```

**代替モデルを使用する場合**: `Modelfile` の1行目を変更してください：

```dockerfile
# 変更前
FROM gpt-oss:20b

# 変更後（例: Llama 3.2を使用）
FROM llama3.2
```

---

## CodeQuizモデルの作成

### 1. リポジトリのクローン（または移動）

```bash
cd /path/to/CodeQuiz-Modelfile
```

### 2. モデルの作成

```bash
ollama create codequiz:latest -f ./Modelfile
```

実行中の出力例:
```
transferring model data
using existing layer sha256:abc123...
using existing layer sha256:def456...
writing manifest
success
```

### 3. モデルの確認

```bash
ollama list
```

出力に `codequiz:latest` が表示されることを確認：
```
NAME              ID              SIZE      MODIFIED
codequiz:latest   xyz789abc123    12 GB     1 minute ago
gpt-oss:20b       abc123def456    12 GB     2 hours ago
```

---

## インストールの確認

### 1. Ollamaサービスの稼働確認

```bash
# macOS / Linux
ollama ps

# Windows (PowerShell)
ollama ps
```

何も表示されない場合、サービスが起動していません：

```bash
ollama serve
```

### 2. APIエンドポイントの確認

```bash
curl http://localhost:11434/api/tags
```

期待されるレスポンス:
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

### 3. モデルの対話テスト

```bash
ollama run codequiz:latest
```

プロンプト例:
```
>>> 作成する数: 1
コードスニペット:
def hello():
    print("Hello, World!")
```

正しくセットアップされていれば、JSON形式の問題が返されます。

終了するには `/bye` を入力してください。

---

## トラブルシューティング

### 問題: コマンドが見つからない

**症状**: `command not found: ollama`

**解決方法**:

```bash
# パスの確認
echo $PATH

# Ollamaのインストール場所を確認
which ollama

# macOS / Linux: パスを追加
export PATH="/usr/local/bin:$PATH"
echo 'export PATH="/usr/local/bin:$PATH"' >> ~/.zshrc  # または ~/.bashrc
```

### 問題: ポートが使用中

**症状**: `Error: listen tcp 127.0.0.1:11434: bind: address already in use`

**解決方法**:

```bash
# 使用中のプロセスを確認
lsof -i :11434

# プロセスを終了（PIDを確認してから）
kill <PID>

# または別のポートを使用
OLLAMA_HOST=0.0.0.0:11435 ollama serve
```

### 問題: モデルのダウンロードが失敗する

**症状**: ダウンロード中にタイムアウトまたはエラー

**解決方法**:

1. インターネット接続を確認
2. プロキシ設定を確認（企業ネットワークの場合）
3. 再度ダウンロードを試行
4. ディスク容量を確認

```bash
# ディスク容量の確認
df -h

# 不要なモデルの削除
ollama rm <不要なモデル名>
```

### 問題: メモリ不足

**症状**: モデルの読み込み時にOOMエラー

**解決方法**:

1. より小さいモデルを使用（上記の代替モデルを参照）
2. GPU使用を無効化してCPUのみで実行

Modelfileを編集:
```dockerfile
PARAMETER num_gpu 0  # GPUを無効化
PARAMETER num_ctx 4096  # コンテキストサイズを削減
```

3. 他のアプリケーションを閉じてメモリを解放

---

## 次のステップ

✅ インストールが完了したら：

1. [Modelfile リファレンス](./MODELFILE_REFERENCE.md) - モデルのカスタマイズ方法
2. [Ollama コマンド](./OLLAMA_COMMANDS.md) - すべてのコマンドの詳細
3. [API テスト](./API_TESTING.md) - curlでモデルをテストする方法
4. [メインREADME](./README.md) - プロジェクト概要に戻る

---

**最終更新**: 2024-03-23

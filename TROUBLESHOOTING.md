# トラブルシューティングガイド

このドキュメントでは、CodeQuiz AIサーバーでよく発生する問題とその解決方法を説明します。

## 📋 目次

1. [モデル関連の問題](#モデル関連の問題)
2. [サービス接続の問題](#サービス接続の問題)
3. [出力・フォーマットの問題](#出力フォーマットの問題)
4. [パフォーマンスの問題](#パフォーマンスの問題)
5. [メモリ・リソースの問題](#メモリリソースの問題)
6. [その他の問題](#その他の問題)

---

## モデル関連の問題

### 問題1: モデルが見つからない

#### 症状

```bash
Error: model 'codequiz:latest' not found
```

または APIレスポンス:
```json
{
  "error": "model 'codequiz:latest' not found"
}
```

#### 原因

- モデルが作成されていない
- モデル名が間違っている
- モデルが削除された

#### 解決方法

**1. モデルの存在を確認**

```bash
ollama list
```

**2. モデルが存在しない場合は作成**

```bash
ollama create codequiz:latest -f ./Modelfile
```

**3. モデル名を確認**

```bash
# API では必ず "codequiz:latest" を使用
curl http://localhost:11434/api/tags | grep codequiz
```

---

### 問題2: モデルの作成に失敗する

#### 症状

```bash
Error: failed to create model: ...
```

#### 原因

- Modelfile が見つからない
- Modelfile の構文エラー
- ベースモデルが存在しない
- ディスク容量不足

#### 解決方法

**1. Modelfile の存在確認**

```bash
ls -lh ./Modelfile
```

**2. Modelfile の構文確認**

```bash
# 最初の10行を確認
head -10 ./Modelfile
```

FROM 行が正しいか確認:
```dockerfile
FROM gpt-oss:20b  # このモデルが存在するか確認
```

**3. ベースモデルのダウンロード**

```bash
ollama pull gpt-oss:20b
```

**4. ディスク容量の確認**

```bash
df -h
```

最低10GB以上の空き容量が必要です。

---

### 問題3: モデルの更新が反映されない

#### 症状

Modelfile を編集したが、モデルの動作が変わらない

#### 原因

- モデルが再作成されていない
- 古いモデルがメモリに残っている

#### 解決方法

**1. 既存モデルを完全に削除**

```bash
ollama rm codequiz:latest
```

**2. 新しいモデルを作成**

```bash
ollama create codequiz:latest -f ./Modelfile
```

**3. メモリからクリア**

```bash
# 実行中のモデルを確認
ollama ps

# Ollamaサービスを再起動（必要な場合）
# macOS / Linux
pkill ollama
ollama serve

# または systemd (Linux)
sudo systemctl restart ollama
```

---

## サービス接続の問題

### 問題4: Ollama サービスに接続できない

#### 症状

```bash
Error: could not connect to ollama server
```

または

```bash
curl: (7) Failed to connect to localhost port 11434: Connection refused
```

#### 原因

- Ollama サービスが起動していない
- ポートが使用中
- ファイアウォールがブロックしている

#### 解決方法

**1. サービスの起動確認**

```bash
ollama ps
```

エラーが出る場合はサービスを起動:

```bash
ollama serve
```

**2. プロセスの確認**

```bash
# macOS / Linux
ps aux | grep ollama

# Windows (PowerShell)
Get-Process | Where-Object { $_.Name -like "*ollama*" }
```

**3. ポートの確認**

```bash
# macOS / Linux
lsof -i :11434

# Windows (PowerShell)
netstat -ano | findstr :11434
```

**4. 別のポートを試す**

```bash
OLLAMA_HOST=0.0.0.0:11435 ollama serve
```

---

### 問題5: ポートが使用中

#### 症状

```bash
Error: listen tcp 127.0.0.1:11434: bind: address already in use
```

#### 原因

- 既に Ollama が起動している
- 別のプロセスが 11434 ポートを使用

#### 解決方法

**1. 使用中のプロセスを確認**

```bash
lsof -i :11434
```

出力例:
```
COMMAND   PID   USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
ollama    1234  user   3u   IPv4  12345      0t0  TCP localhost:11434
```

**2. プロセスを終了**

```bash
kill 1234  # PID を指定
```

**3. または別のポートを使用**

```bash
OLLAMA_HOST=0.0.0.0:11435 ollama serve
```

その場合、APIリクエストも変更:
```bash
curl http://localhost:11435/api/tags
```

---

## 出力・フォーマットの問題

### 問題6: JSON 解析エラー

#### 症状

サーバーが JSON を解析できない、または無効な JSON が返される

#### 原因

- モデルの出力が純粋な JSON でない
- 説明文やコードブロック記法が含まれている
- `temperature` が高すぎる

#### 解決方法

**1. 出力を直接確認**

```bash
ollama run codequiz:latest "作成する数: 1
コードスニペット:
def test(): pass"
```

**2. SYSTEM メッセージを確認**

```bash
ollama show --system codequiz:latest
```

"JSON のみを出力" の指示があるか確認。

**3. temperature を下げる**

Modelfile を編集:
```dockerfile
PARAMETER temperature 0.5  # 0.7 から 0.5 に変更
```

再作成:
```bash
ollama rm codequiz:latest
ollama create codequiz:latest -f ./Modelfile
```

**4. 別のベースモデルを試す**

```dockerfile
FROM llama3.2  # より新しいモデル
```

---

### 問題7: 出力が期待と異なる

#### 症状

- 問題の品質が低い
- 制約が守られていない
- 選択肢の長さが不均一

#### 原因

- パラメータ設定が適切でない
- ベースモデルの能力不足
- SYSTEM プロンプトの問題

#### 解決方法

**1. temperature を調整**

```dockerfile
# より一貫性のある出力
PARAMETER temperature 0.5

# より多様な出力
PARAMETER temperature 0.9
```

**2. repeat_penalty を上げる**

```dockerfile
PARAMETER repeat_penalty 1.3  # 繰り返しを強く抑制
```

**3. より高性能なモデルに変更**

```dockerfile
FROM llama3.2:70b  # より大きく高品質なモデル
```

**4. SYSTEM プロンプトを見直す**

現在の制約が正しく記載されているか確認:
```bash
ollama show --system codequiz:latest
```

---

### 問題8: answer が 0 以外になる

#### 症状

生成された JSON の `answer` フィールドが 0 以外の値

#### 原因

- SYSTEM プロンプトの指示が不明確
- Few-Shot 例が不適切

#### 解決方法

**1. SYSTEM プロンプトを確認**

以下の記述があるか確認:
```
answerフィールドは常に 0（正解は必ず choices[0] に配置）。
```

**2. MESSAGE 例を確認**

すべての MESSAGE 例で `answer: 0` になっているか確認:
```bash
ollama show --modelfile codequiz:latest | grep "answer"
```

**3. temperature を下げる**

```dockerfile
PARAMETER temperature 0.5
```

---

## パフォーマンスの問題

### 問題9: 生成が遅い

#### 症状

問題生成に時間がかかりすぎる（30秒以上）

#### 原因

- CPU のみで実行している
- モデルが大きすぎる
- コンテキストサイズが大きすぎる

#### 解決方法

**1. GPU 使用を確認**

```bash
ollama ps
```

PROCESSOR 列を確認（"100% GPU" が理想）

**2. GPU を有効化**

Modelfile:
```dockerfile
PARAMETER num_gpu 99  # または -1
```

**3. より小さいモデルを使用**

```dockerfile
FROM llama3.2:3b  # 20b の代わりに
```

**4. コンテキストサイズを削減**

```dockerfile
PARAMETER num_ctx 4096  # 16384 から削減
```

**5. CPU スレッド数を最適化**

```dockerfile
PARAMETER num_thread 8  # CPU コア数に応じて調整
```

---

### 問題10: API がタイムアウトする

#### 症状

```bash
curl: (28) Operation timed out after 30000 milliseconds
```

#### 原因

- モデルの生成時間が長い
- サーバーの負荷が高い

#### 解決方法

**1. タイムアウトを延長**

```bash
curl --max-time 300 http://localhost:11434/api/generate -d '{...}'
```

**2. num_predict を制限**

Modelfile:
```dockerfile
PARAMETER num_predict 1024  # 生成トークン数を制限
```

**3. 並列リクエスト数を制限**

```bash
OLLAMA_NUM_PARALLEL=1 ollama serve
```

---

## メモリ・リソースの問題

### 問題11: メモリ不足（OOM）

#### 症状

```bash
Error: failed to load model: out of memory
```

または システムが フリーズ

#### 原因

- RAM が不足している
- モデルが大きすぎる
- コンテキストサイズが大きすぎる

#### 解決方法

**1. より小さいモデルを使用**

```dockerfile
FROM llama3.2:3b  # 約2GB（20bの代わりに）
```

**2. GPU メモリを調整**

```dockerfile
PARAMETER num_gpu 0  # GPU 無効、CPU のみ
```

または一部のみ GPU:
```dockerfile
PARAMETER num_gpu 20  # 一部のレイヤーのみ
```

**3. コンテキストサイズを削減**

```dockerfile
PARAMETER num_ctx 4096  # 16384 から削減
```

**4. システムメモリを確認**

```bash
# macOS / Linux
free -h

# macOS
vm_stat
```

**5. 他のアプリケーションを閉じる**

---

### 問題12: ディスク容量不足

#### 症状

```bash
Error: no space left on device
```

#### 原因

- モデルファイルが大きい（10GB以上）
- 複数のモデルが保存されている

#### 解決方法

**1. ディスク使用量を確認**

```bash
df -h
du -sh ~/.ollama/models
```

**2. 未使用のモデルを削除**

```bash
ollama list  # モデル一覧を確認
ollama rm <不要なモデル名>
```

**3. 古いモデルを一括削除**

```bash
# 使っていないモデルを削除
ollama list | tail -n +2 | awk '{print $1}' | grep -v codequiz | xargs -I {} ollama rm {}
```

---

## その他の問題

### 問題13: バージョン互換性の問題

#### 症状

モデルが正常に動作しない、予期しないエラー

#### 原因

- Ollama のバージョンが古い
- Modelfile の構文が新しい

#### 解決方法

**1. バージョンを確認**

```bash
ollama --version
```

**2. 最新版にアップデート**

```bash
# macOS / Linux
curl -fsSL https://ollama.com/install.sh | sh

# または Homebrew (macOS)
brew upgrade ollama
```

**3. Modelfile の互換性を確認**

古いバージョンでサポートされていないパラメータがないか確認。

---

### 問題14: 権限エラー

#### 症状

```bash
Error: permission denied
```

#### 原因

- ファイルの読み取り権限がない
- Ollama の実行権限がない

#### 解決方法

**1. ファイル権限を確認**

```bash
ls -lh ./Modelfile
```

**2. 権限を付与**

```bash
chmod 644 ./Modelfile
```

**3. Ollama の実行権限を確認**

```bash
which ollama
ls -lh $(which ollama)
```

---

### 問題15: macOS でのセキュリティ警告

#### 症状

「開発元が未確認のため開けません」という警告

#### 解決方法

**1. システム環境設定を開く**

- システム環境設定 → セキュリティとプライバシー
- 「このまま開く」をクリック

**2. またはコマンドラインで許可**

```bash
xattr -d com.apple.quarantine /Applications/Ollama.app
```

---

## 診断ツール

### システム診断スクリプト

`diagnose.sh` を作成:

```bash
#!/bin/bash

echo "=== Ollama Diagnostics ==="
echo

echo "1. Ollama Version:"
ollama --version
echo

echo "2. Service Status:"
if ollama ps > /dev/null 2>&1; then
    echo "✅ Ollama service is running"
    ollama ps
else
    echo "❌ Ollama service is NOT running"
fi
echo

echo "3. Models:"
ollama list
echo

echo "4. Disk Space:"
df -h | grep -E "Filesystem|/$"
echo

echo "5. Memory:"
if command -v free > /dev/null; then
    free -h
else
    vm_stat | head -5
fi
echo

echo "6. Port 11434:"
if lsof -i :11434 > /dev/null 2>&1; then
    echo "✅ Port 11434 is in use"
    lsof -i :11434
else
    echo "❌ Port 11434 is not in use"
fi
echo

echo "7. API Test:"
if curl -s http://localhost:11434/api/tags > /dev/null 2>&1; then
    echo "✅ API is accessible"
else
    echo "❌ API is NOT accessible"
fi
echo

echo "=== End of Diagnostics ==="
```

実行:
```bash
chmod +x diagnose.sh
./diagnose.sh
```

---

## よくある質問

### Q1: モデル名を変更できますか？

**A**: はい、可能です。ただしサーバー側のコードも変更する必要があります。

```bash
# 新しい名前でモデルを作成
ollama create myquiz:v1 -f ./Modelfile
```

サーバーコードで `codequiz:latest` を `myquiz:v1` に変更してください。

### Q2: 複数のバージョンを同時に使用できますか？

**A**: はい、異なる名前で作成すれば可能です。

```bash
ollama create codequiz:v1 -f ./Modelfile.v1
ollama create codequiz:v2 -f ./Modelfile.v2
```

### Q3: モデルを他のマシンにコピーできますか？

**A**: はい、Modelfile をエクスポートして使用できます。

```bash
# エクスポート
ollama show --modelfile codequiz:latest > exported.modelfile

# 他のマシンで
ollama create codequiz:latest -f exported.modelfile
```

---

## サポート情報の収集

問題が解決しない場合、以下の情報を収集してください：

```bash
# 1. バージョン情報
ollama --version

# 2. モデル一覧
ollama list

# 3. サービス状態
ollama ps

# 4. Modelfile の内容（最初の50行）
head -50 ./Modelfile

# 5. システム情報
uname -a

# 6. ログ（macOS）
tail -100 ~/Library/Logs/Ollama/server.log

# 7. ログ（Linux systemd）
journalctl -u ollama -n 100
```

---

## 次のステップ

✅ 問題が解決したら：

1. [API テスト](./API_TESTING.md) - モデルの動作を確認
2. [Modelfile リファレンス](./MODELFILE_REFERENCE.md) - 設定の最適化
3. [Ollama コマンド](./OLLAMA_COMMANDS.md) - 管理コマンドの詳細
4. [メインREADME](./README.md) - プロジェクト概要に戻る

---

**最終更新**: 2024-03-23

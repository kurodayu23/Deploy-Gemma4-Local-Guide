[English](README.md) | [简体中文](README_zh.md) | [日本語](README_ja.md)

---

# 🚀 Google Gemma 4 をローカルにデプロイする：実践者のための完全ガイド

> **著者より**: これはコピペチュートリアルではありません。VRAM 制限、モデル量子化のトレードオフ、そして Windows 上の Ollama の癖と丸一週末格闘した後に書きました。クラウドプロバイダーに一円も払わずに自分のハードウェアで Gemma 4 を動かしたいなら、このガイドで何時間も節約できます。

---

## 目次

- [なぜ Gemma 4 なのか？](#なぜ-gemma-4-なのか)
- [前提条件とハードウェアの現実チェック](#前提条件とハードウェアの現実チェック)
- [フェーズ 1：Windows に Ollama をセットアップ](#フェーズ-1windows-に-ollama-をセットアップ)
- [フェーズ 2：Gemma 4 のプルと実行](#フェーズ-2gemma-4-のプルと実行)
- [フェーズ 3：モデル保存先のパス変更](#フェーズ-3モデル保存先のパス変更)
- [フェーズ 4：モデルのインタラクティブテスト](#フェーズ-4モデルのインタラクティブテスト)
- [フェーズ 5：ローカル REST API の公開](#フェーズ-5ローカル-rest-api-の公開)
- [フェーズ 6：Python との統合（LangChain / 生の HTTP）](#フェーズ-6python-との統合langchain--生の-http)
- [フェーズ 7：パフォーマンスチューニングと VRAM 管理](#フェーズ-7パフォーマンスチューニングと-vram-管理)
- [トラブルシューティング（誰も教えてくれないこと）](#トラブルシューティング誰も教えてくれないこと)
- [次のステップ](#次のステップ)

---

## なぜ Gemma 4 なのか？

正直に言います。ローカルモデルはたくさん試しました。Llama 3、Mistral、Phi-3、Qwen2… それぞれ得意分野があります。でも Google の Gemma 4 は別格でした。理由は以下の通り：

- **コード理解力が異常**。500 行の C++ Qt ファイルを投げたら、LNK4042 リンカーエラーを引き起こしている名前空間汚染の問題を正確に特定しました。ほとんどの 7B モデルはハルシネーションを起こすだけです。
- **ネイティブに多言語対応**。英語、中国語、日本語のドキュメントを日常的に扱っていますが、Gemma 4 は品質低下なしに 3 言語を処理します。
- **真のオープンウェイト**。Google が寛容なライセンスでリリース。ファインチューニング、商用利用、何でも OK。
- **推論効率が高い**。4B バリアントは 8GB VRAM で快適に動作。12B は約 10-12GB。A100 は不要です。

---

## 前提条件とハードウェアの現実チェック

始める前に、必要なものを正直に確認しましょう：

| コンポーネント | 最低限（4B モデル） | 推奨（12B モデル） | 私の構成 |
|------------|------------------|------------------|---------|
| GPU | GTX 1660 (6GB) | RTX 3060 (12GB) | RTX 3060 12GB |
| RAM | 16GB | 32GB | 32GB DDR4 |
| ディスク容量 | 5GB 空き | 15GB 空き | 外付け D ドライブ |
| OS | Windows 10/11 | Windows 10/11 | Windows 11 |

### 必須ソフトウェア

```powershell
# NVIDIA ドライバーがインストールされているか確認
nvidia-smi
```

---

## フェーズ 1：Windows に Ollama をセットアップ

Ollama はローカルモデルのデプロイを楽にするランタイムです。「LLM 用の Docker」と考えてください。

```powershell
# winget でインストール（推奨）
winget install --id Ollama.Ollama -e --source winget

# インストール確認
ollama --version
```

> **私がハマった点**: winget でインストールした後、`ollama` コマンドが認識されるまで**ターミナルを再起動** する必要がありました。

---

## フェーズ 2：Gemma 4 のプルと実行

```powershell
# 12B バリアント（12GB+ VRAM があれば推奨）
ollama pull gemma3:12b

# 軽量 4B バリアント
ollama pull gemma3:4b

# ダウンロード確認
ollama list
```

---

## フェーズ 3：モデル保存先のパス変更

C ドライブの容量が少ない場合、これは重要です。

```powershell
# ターゲットディレクトリの作成
New-Item -ItemType Directory -Path "D:\LLM_Models" -Force

# システム環境変数の設定（管理者として実行）
[System.Environment]::SetEnvironmentVariable("OLLAMA_MODELS", "D:\LLM_Models", "Machine")

# Ollama サービスの再起動
taskkill /f /im ollama.exe
ollama serve
```

> **私の失敗**: 最初は User 変数として設定してしまいました。Ollama のバックグラウンドサービスは別のコンテキストで実行されるため、ユーザーレベルの環境変数は無視されます。`"Machine"` スコープを使ってください。

---

## フェーズ 4：モデルのインタラクティブテスト

```powershell
ollama run gemma3:12b
```

テスト用プロンプト：

```
>>> Pythonの非同期処理（asyncio）におけるgatherとwaitの違いを具体的に説明してください。
```

```
>>> C++プロジェクト用のマルチステージDockerfileを書いてください。最終イメージはdistrolessで。
```

終了するには `/bye` と入力。

---

## フェーズ 5：ローカル REST API の公開

Ollama はポート `11434` でローカル HTTP サーバーを自動実行します。

```powershell
# ヘルスチェック
curl http://localhost:11434/

# テキスト生成
curl http://localhost:11434/api/generate -d '{
  "model": "gemma3:12b",
  "prompt": "フランスの首都はどこですか？",
  "stream": false
}'
```

---

## フェーズ 6：Python との統合（LangChain / 生の HTTP）

### 方法 1：生の HTTP（依存関係ゼロ）

```python
import json
import urllib.request

def ask_gemma(prompt: str, model: str = "gemma3:12b") -> str:
    payload = {
        "model": model,
        "prompt": prompt,
        "stream": False,
        "options": {"temperature": 0.3}
    }
    data = json.dumps(payload).encode("utf-8")
    req = urllib.request.Request(
        "http://localhost:11434/api/generate",
        data=data,
        headers={"Content-Type": "application/json"}
    )
    with urllib.request.urlopen(req) as resp:
        return json.loads(resp.read().decode("utf-8"))["response"]

answer = ask_gemma("LoRA ファインチューニングを3文で説明してください。")
print(answer)
```

### 方法 2：LangChain 統合

```python
from langchain_community.llms import Ollama

llm = Ollama(model="gemma3:12b")
response = llm.invoke("SonarQube 分析を実行する Jenkins パイプラインを書いてください。")
print(response)
```

---

## フェーズ 7：パフォーマンスチューニングと VRAM 管理

### GPU 使用率のリアルタイム監視

```powershell
nvidia-smi --query-gpu=name,memory.used,memory.total,utilization.gpu --format=csv -l 2
```

### コンテキストウィンドウサイズの調整

```powershell
@"
FROM gemma3:12b
PARAMETER num_ctx 8192
PARAMETER temperature 0.2
"@ | Out-File -FilePath Modelfile -Encoding utf8

ollama create gemma-code -f Modelfile
ollama run gemma-code
```

> **警告**: コンテキストウィンドウが大きいほど VRAM を消費します。8GB の場合は最大 4096 に抑えてください。

---

## トラブルシューティング（誰も教えてくれないこと）

### "CUDA out of memory"

1. `num_ctx` を 2048 に削減
2. 4B バリアントを使用
3. `num_gpu 15` で一部のレイヤーを CPU にオフロード

### Windows Defender がモデル読み込みを遅くしている

```powershell
# 管理者として実行
Add-MpExclusion -Path "D:\LLM_Models"
```

---

## 次のステップ

1. **プライベートコードベース上のローカル RAG システム構築**（[RAG-LangChain-Tuner](https://github.com/kurodayu23/RAG-LangChain-Tuner) 参照）
2. **IDE との統合** — Cursor、Claude Code は `localhost:11434` を指定可能
3. **LoRA でファインチューニング**
4. **他ツールとの連携**（[VibeOps-Agent](https://github.com/kurodayu23/VibeOps-Agent) 参照）

要点：**モデルはあなたのもの、データもあなたのもの、推論もあなたのもの**。API キーなし、レート制限なし、月額料金なし。

これが Vibe Coding の未来です。そして、それは `ollama pull` から始まります。

---

*実際のデプロイ経験に基づいて執筆。このガイドのすべてのセクションは、私が実際にぶつかったエラーと見つけた解決策から生まれています。*

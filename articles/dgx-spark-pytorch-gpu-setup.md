---
title: "\"sm_121 is not compatible\" — DGX SparkでPyTorchを動かす方法"
emoji: "⚡"
type: "tech"
topics: ["DGXSpark", "PyTorch", "NVIDIA", "GPU", "Docker"]
published: true
publication_name: "toggle_inc"
---

## はじめに

NVIDIA DGX Sparkを購入してPyTorchを動かそうとしたところ、GPUが認識されない問題に遭遇しました。本記事では、解決方法を2つ紹介し、どちらを選ぶべきかを解説します。

### 対象読者
- DGX Spark ユーザー
- PyTorchでGPUを使いたい方

※ ASUS Ascent GX10も同じGB10チップを搭載しているため、同様の手順で動作する可能性がありますが、未検証です。

### 環境
- NVIDIA DGX Spark (GB10)
- CUDA 13.0
- Ubuntu 24.04 (DGX OS)

## 問題: sm_121互換性エラー

通常のPyTorchをインストールすると、以下の警告が表示されます：

```
NVIDIA GB10 with CUDA capability sm_121 is not compatible with the current PyTorch installation.
The current PyTorch install supports CUDA capabilities sm_80 sm_86 sm_90 compute_90.
```

### 原因

- NVIDIA GB10はBlackwellアーキテクチャ（compute capability 12.1 = sm_121）
- PyTorch公式リリースはsm_121をまだ正式サポートしていない

## 解決策1: pip + cu129（シンプル）

最もシンプルな方法です。CUDA 12.9用のwheelを使います。

```bash
pip install torch torchvision --index-url https://download.pytorch.org/whl/cu129
```

:::message
DGX SparkのCUDAは13.0ですが、CUDAのバイナリ互換性により12.9用wheelでも動作します。
:::

### 確認
```bash
python -c "import torch; print(torch.cuda.is_available())"
# → True
```

実行すると以下の警告が表示されますが、**GPUは正常に動作します**：

```
UserWarning: NVIDIA GB10 with CUDA capability sm_121 is not compatible
with the current PyTorch installation.
```

### メリット・デメリット

| メリット | デメリット |
|----------|------------|
| pip一発で完了 | 上記の警告が毎回表示される |
| Dockerなしで動作 | 環境依存 |
| 軽量 | 将来の互換性は不明 |

:::message
この方法は[@karaage0703さんの記事](https://zenn.dev/karaage0703/articles/985ddbd8fa15d3)で詳しく紹介されています。
:::

## 解決策2: NGC PyTorch 25.09コンテナ（安定・公式推奨）

NVIDIAの公式コンテナを使う方法です。sm_121を明示的にサポートしています。

:::message
DGX Sparkユーザーには、NVIDIAが[NGCコンテナの使用を推奨](https://docs.nvidia.com/dgx/dgx-spark/ngc.html)しています。25.01以降がBlackwell対応で、本記事では執筆時点の最新版25.09を使用しています。
:::

### Dockerfile

```dockerfile
FROM nvcr.io/nvidia/pytorch:25.09-py3

WORKDIR /app

# 必要に応じて依存関係を追加
# RUN pip install --no-cache-dir your-package

CMD ["python", "your_script.py"]
```

### docker-compose.yml

```yaml
services:
  app:
    build: .
    runtime: nvidia
    environment:
      - NVIDIA_VISIBLE_DEVICES=all
    volumes:
      - .:/app
    working_dir: /app
```

:::message alert
`runtime: nvidia`を使用するには、[NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html)のインストールと、Dockerデーモンへのnvidiaランタイム登録が必要です。DGX Sparkはデフォルトで設定済みです。
:::

### 実行

```bash
docker compose build
docker compose run --rm app python your_script.py
```

### 確認

```bash
docker compose run --rm app python -c "
import torch
print(f'CUDA available: {torch.cuda.is_available()}')
print(f'Device: {torch.cuda.get_device_name(0)}')
"
```

出力:
```
CUDA available: True
Device: NVIDIA GB10
```

### メリット・デメリット

| メリット | デメリット |
|----------|------------|
| NVIDIA公式の最適化済み | Dockerが必要 |
| 警告が出ない | イメージが大きい (~15GB) |
| 再現性が高い | Python 3.12固定 |

## どちらを選ぶ？

**とりあえず動かしたい / 学習目的**
→ pip + cu129

**本番環境 / チーム開発 / 安定性重視**
→ NGC PyTorch 25.09

| 観点 | pip + cu129 | NGC 25.09 |
|------|-------------|-----------|
| 手軽さ | ◎ | △ |
| 公式サポート | △ | ◎ |
| 再現性 | △ | ◎ |
| 警告なし | × | ◎ |

## 特殊なケース

以下のような条件に該当する場合は、CUDA 13.0ベースイメージから自前でビルドする方法が必要になることがあります：

- **CUDA拡張をソースからコンパイルするライブラリ**（例: MMCV, detectron2など）
- **Python 3.12未対応のライブラリ**（NGCコンテナはPython 3.12固定のため）

:::message
具体例として[DGX SparkでMMDetection3の環境構築](https://zenn.dev/toggle_inc/articles/e0b50df4797fcc)が参考になります。
:::

## コラム: DGX Sparkエコシステムの現状

海外フォーラムでは、DGX Sparkユーザーの苦労話が多く見られます。

### 「ハードウェアは素晴らしい、でもエコシステムは発展途上」

[Simon Willison氏のレビュー](https://simonwillison.net/2025/Oct/14/nvidia-dgx-spark/)では、DGX Sparkの課題として**ARM64 + sm_121の二重苦**が挙げられています。従来のNVIDIA GPUエコシステムはx86アーキテクチャを前提としており、ARM64ベースのSparkでは予期しない問題が多発したとのこと。

> "PyTorch ARM エコシステムの操作は非常に混乱している"

ただし、embargo解除直後にOllama、llama.cpp、LM Studio、vLLMなどが一斉にSpark対応を発表し、エコシステムは急速に成熟しつつあります。

### 「$41で買える最高のChatGPT」

[karpathy/nanochat](https://github.com/karpathy/nanochat/discussions/28)のDiscussionでは、DGX Sparkでの訓練結果が報告されています：

- 単一Spark: 約10日間で訓練完了（消費電力 ~120W）
- 2台接続: 約4〜5日に短縮

電力コスト約$8 + ハードウェア減価償却費を含めて約$41という試算から、「The best ChatGPT that $41 can buy?」というコメントも。

## まとめ

DGX Sparkは新しいハードウェアのため、エコシステムがまだ成熟していません。

- **手軽さ重視** → pip + cu129
- **安定性重視** → NGC 25.09

PyTorchの公式リリースがsm_121をサポートすれば、将来的にはpip一発で済むようになるはずです。

## 参考リンク

### NVIDIA公式
- [DGX Spark - Using NGC](https://docs.nvidia.com/dgx/dgx-spark/ngc.html)
- [PyTorch Release 25.09 Release Notes](https://docs.nvidia.com/deeplearning/frameworks/pytorch-release-notes/rel-25-09.html)

### コミュニティ
- [DGX Spark GB10 Cuda 13.0 Python 3.12 SM_121 - PyTorch Forums](https://discuss.pytorch.org/t/dgx-spark-gb10-cuda-13-0-python-3-12-sm-121/223744)
- [Effective PyTorch and CUDA - DGX Spark / GB10 - NVIDIA Developer Forums](https://forums.developer.nvidia.com/t/effective-pytorch-and-cuda/348230)
- [NVIDIA DGX SparkでローカルAI環境を構築 - Zenn](https://zenn.dev/karaage0703/articles/985ddbd8fa15d3)
- [DGX SparkでMMDetection3の環境構築 - Zenn](https://zenn.dev/toggle_inc/articles/e0b50df4797fcc)

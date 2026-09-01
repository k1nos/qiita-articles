---
title: Ubuntu 24.04 + RTX 4070 で Docker から GPU を使う最短手順と、実測18倍の性能差
tags:
  - Docker
  - NVIDIA
  - GPU
  - PyTorch
  - Ubuntu
private: false
updated_at: '2026-08-18T23:12:17+09:00'
id: dec1bd95d63b75ff3f09
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

## はじめに

ローカルGPU環境（Ubuntu 24.04 + RTX 4070）に Docker と NVIDIA Container Toolkit を導入し、コンテナ内から GPU を使って PyTorch の学習を回すまでの手順をまとめます。

作業時間は1時間ほどでした。詰まりやすい点と、最後に **GPU と CPU の実測比較（約18倍差）** も載せています。

### この記事の対象

- ローカルのGPUマシンをコンテナ化して使いたい人
- 「ホストに直接CUDA環境を作ったが、そろそろコンテナに移したい」人
- GPU/AI基盤まわりの技術に触れておきたいインフラエンジニア

### なぜコンテナで動かすのか

ホストに直接 CUDA や PyTorch を入れると、**バージョンの組み合わせが固定されてしまう**という問題があります。

私はもともとホスト環境に Isaac Sim / Isaac Lab を構築していましたが、別のプロジェクトで違う PyTorch バージョンが必要になったとき、環境ごと壊してしまうリスクがありました。コンテナなら、プロジェクトごとに独立した環境を並べられます。

## 検証環境

| 項目 | 内容 |
| --- | --- |
| OS | Ubuntu 24.04.4 LTS (noble) |
| GPU | NVIDIA GeForce RTX 4070 (VRAM 12GB) |
| CPU | Intel Core i5-12400 |
| ドライバ | 595.84 |
| CUDA (ドライバ側) | 13.2 |

前提として、**ホスト側で `nvidia-smi` が通っている**ことを確認しておきます。

```bash
nvidia-smi
```

これが動かない場合は、まずドライバの導入から解決してください。コンテナ側の問題ではありません。

## 1. Docker Engine の導入

Ubuntu の `apt` にある `docker.io` ではなく、**Docker 公式リポジトリ**から入れます。公式版のほうがバージョンが新しく、後述の NVIDIA Container Toolkit との組み合わせでも情報が見つけやすいためです。

```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg

# 公式GPGキー
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# リポジトリ登録
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

動作確認：

```bash
sudo docker run --rm hello-world
```

`Hello from Docker!` が表示されれば導入完了です。

## 2. sudo なしで docker を使えるようにする

毎回 `sudo` を打つのは面倒なので、自分を `docker` グループに追加します。

```bash
sudo usermod -aG docker $USER
newgrp docker
```

`newgrp docker` を実行すると、**ログアウトせずにその場でグループが反映**されます。リモート接続で作業しているときに便利です。

```bash
docker run --rm hello-world   # sudoなしで通ればOK
```

:::note warn
`docker` グループへの追加は、実質的に root 相当の権限を与えることになります。共用マシンでは慎重に判断してください。
:::

## 3. NVIDIA Container Toolkit の導入

ここが本題です。コンテナからホストのGPUを見せるための部品を入れます。

```bash
# GPGキー
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg

# リポジトリ登録
curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
  sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
  sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

sudo apt update
sudo apt install -y nvidia-container-toolkit

# Docker に NVIDIA ランタイムを登録
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

`nvidia-ctk runtime configure` が `/etc/docker/daemon.json` を書き換えて、NVIDIA ランタイムを登録してくれます。**この後の Docker 再起動を忘れると反映されません。**

## 4. 動作確認

```bash
docker run --rm --gpus all nvidia/cuda:13.0.0-base-ubuntu24.04 nvidia-smi
```

コンテナ内で `nvidia-smi` が動き、GPU が表示されれば成功です。

```
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 595.84                 Driver Version: 595.84         CUDA Version: 13.2     |
+-----------------------------------------+------------------------+----------------------+
|   0  NVIDIA GeForce RTX 4070        Off |   00000000:01:00.0  On |                  N/A |
+-----------------------------------------+------------------------+----------------------+
| Processes:                                                                              |
|  No running processes found                                                             |
+-----------------------------------------------------------------------------------------+
```

:::note info
`No running processes found` と出ますが、これは**正常**です。コンテナは独立したPID名前空間で動くため、ホスト側で GPU を使っているプロセス（Xorg など）が見えないだけです。
:::

### CUDA イメージのタグについて

ホストのドライバが CUDA 13.2 対応だったので `13.0.0-base-ubuntu24.04` を使いました。ドライバは下位互換があるため、`12.6.0-base-ubuntu24.04` でも動作します。

タグは [Docker Hub の nvidia/cuda](https://hub.docker.com/r/nvidia/cuda) で確認できます。`base` / `runtime` / `devel` の3種類があり、用途に応じて選びます。

- `base` … 最小構成（動作確認用）
- `runtime` … cuDNN など実行時ライブラリ入り
- `devel` … コンパイラ・ヘッダ入り（ビルドが必要な場合）

## 5. PyTorch コンテナで学習を回す

GPU が見えただけでは実用の確認になりません。実際に学習を回します。

```bash
docker run --rm --gpus all pytorch/pytorch:latest \
  python -c "import torch; print(torch.__version__, torch.cuda.is_available(), torch.cuda.get_device_name(0))"
```

```
2.2.1 True NVIDIA GeForce RTX 4070
```

:::note warn
**`latest` タグが最新とは限りません。**
このとき取得された PyTorch は 2.2.1 でした。ドライバが CUDA 13.2 対応であるのに対し、イメージ側はかなり前のバージョンです。動作はしましたが、本格的に使うなら `pytorch/pytorch:2.5.1-cuda12.4-cudnn9-runtime` のように**バージョンを明示したタグを指定すべき**です。
:::

### 検証用スクリプト

3層のMLPで回帰問題を200エポック学習させます。

```python
import torch, torch.nn as nn, time

dev = torch.device("cuda" if torch.cuda.is_available() else "cpu")
print(f"device: {dev}")
if dev.type == "cuda":
    print(f"GPU: {torch.cuda.get_device_name(0)}")
print(f"torch: {torch.__version__}")

torch.manual_seed(0)
x = torch.randn(50000, 100, device=dev)
w_true = torch.randn(100, 1, device=dev)
y = x @ w_true + 0.1 * torch.randn(50000, 1, device=dev)

model = nn.Sequential(
    nn.Linear(100, 256), nn.ReLU(),
    nn.Linear(256, 256), nn.ReLU(),
    nn.Linear(256, 1),
).to(dev)

opt = torch.optim.Adam(model.parameters(), lr=1e-3)
lossf = nn.MSELoss()

t0 = time.time()
for epoch in range(200):
    opt.zero_grad()
    loss = lossf(model(x), y)
    loss.backward()
    opt.step()
    if (epoch + 1) % 50 == 0:
        mem = torch.cuda.memory_allocated() / 1024**2 if dev.type == "cuda" else 0
        print(f"epoch {epoch+1:3d}  loss {loss.item():.4f}  VRAM {mem:.1f}MiB")

print(f"elapsed: {time.time()-t0:.2f}s")
```

### 実行

ホストのカレントディレクトリをコンテナにマウントして実行します。**コードをイメージに焼き込まずに済む**ので、試行錯誤の段階ではこの方法が便利です。

```bash
# GPU
docker run --rm --gpus all -v $(pwd):/work -w /work pytorch/pytorch:latest python train.py

# CPU（--gpus all を外すだけ）
docker run --rm -v $(pwd):/work -w /work pytorch/pytorch:latest python train.py
```

## 6. 実測結果

### GPU

```
device: cuda
GPU: NVIDIA GeForce RTX 4070
torch: 2.2.1
epoch  50  loss 2.4573  VRAM 38.0MiB
epoch 100  loss 0.4291  VRAM 38.0MiB
epoch 150  loss 0.2686  VRAM 38.0MiB
epoch 200  loss 0.1948  VRAM 38.0MiB
elapsed: 0.88s
```

### CPU

```
device: cpu
torch: 2.2.1
epoch  50  loss 1.9793  VRAM 0.0MiB
epoch 100  loss 0.3491  VRAM 0.0MiB
epoch 150  loss 0.2206  VRAM 0.0MiB
epoch 200  loss 0.1594  VRAM 0.0MiB
elapsed: 16.32s
```

### 比較

| 実行環境 | 所要時間 | VRAM使用量 |
| --- | --- | --- |
| RTX 4070 | **0.88秒** | 38.0 MiB |
| Core i5-12400 | 16.32秒 | - |

**約18.5倍の差**が出ました。

![GPU実行とCPU実行の比較](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4472357/12db4749-98fd-40b0-a79d-af951774614c.png)
※ユーザー名をマスキング済

## 気づいたこと

### VRAM は 38MiB しか使っていない

12GB のうち 38MiB です。**GPU が速いのは演算の並列性によるもので、メモリ量が効いているわけではない**ことがよく分かります。

VRAM 容量が効いてくるのは、大きなモデルやバッチサイズを扱うときです。この規模の学習では、VRAM 12GB でも全く不足しません。

### 同じ seed なのに loss が一致しない

GPU: `0.1948` / CPU: `0.1594` と、最終 loss が異なります。

`torch.manual_seed(0)` を指定しているのに値がズレるのは、**CUDA と CPU で乱数生成器が別系統**だからです。`torch.manual_seed()` は CPU 側の生成器を初期化し、CUDA 側は `torch.cuda.manual_seed()` が別に管理しています。

デバイスをまたいで厳密に結果を再現したい場合は、この点に注意が必要です（そもそも完全一致は困難ですが）。

## まとめ

- Docker + NVIDIA Container Toolkit の導入は、公式手順どおりで1時間ほど
- ポイントは `nvidia-ctk runtime configure` 後の **Docker 再起動**
- `latest` タグは最新とは限らないため、実用時はバージョン明示を推奨
- 小規模な学習でも GPU と CPU で **約18倍**の差
- コンテナ内の `No running processes found` は正常な挙動

ホストを汚さずに複数の環境を並行して持てるのは、思っていた以上に快適でした。次は k3s（軽量Kubernetes）＋ NVIDIA device plugin で、GPU をスケジューリング対象として扱えるところまで試す予定です。

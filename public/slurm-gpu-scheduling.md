---
title: 自宅のRTX 4070にSlurmを入れて、GPUジョブのキューイングを体験する
tags:
  - SLURM
  - GPU
  - NVIDIA
  - Ubuntu
  - HPC
private: false
updated_at: '2026-09-02T07:55:41+09:00'
id: 1df4df903481a25fc65b
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

## はじめに

自宅のGPU環境（Ubuntu 24.04 + RTX 4070）に、HPC/AIクラスタで広く使われているジョブスケジューラ **Slurm** を導入し、GPUをリソースとして割り当ててジョブを投入するまでを試しました。

作業時間は1時間半ほど。途中でノードが `INVAL` 状態になる問題に遭遇し、その原因と解決も含めてまとめます。

### なぜ個人環境でSlurmを試すのか

個人でGPUを使っている限り、リソースの取り合いは起きません。使いたいときに使えます。

しかし複数人・複数ジョブでGPUクラスタを共有する環境では、そうはいきません。「このジョブにGPU 1枚、CPU 4コア、メモリ 16GB」といった要求を受け付け、空きがなければ待たせる仕組みが必要になります。それがジョブスケジューラです。

**この「待たせる」挙動を、実際に自分の目で見てみたかった**というのが動機です。

### 検証環境

| 項目 | 内容 |
| --- | --- |
| OS | Ubuntu 24.04.4 LTS |
| GPU | NVIDIA GeForce RTX 4070 (VRAM 12GB) |
| CPU | Intel Core i5-12400 (6コア12スレッド) |
| メモリ | 31,335 MB |
| ドライバ | 595.84 (CUDA 13.2) |
| Slurm | 23.11.4 |

## Slurmの構成要素

導入前に、最低限おさえておく用語です。

| 名前 | 役割 |
| --- | --- |
| **slurmctld** | 制御デーモン。ジョブを受け付けてスケジューリングする |
| **slurmd** | 計算ノード側のデーモン。実際にジョブを実行する |
| **munge** | ノード間の認証。**これが動かないと何も始まらない** |

今回はシングルノード構成なので、1台がマスターと計算ノードを兼ねます。

主なコマンドは以下の通りです。

| コマンド | 用途 |
| --- | --- |
| `sinfo` | ノードとパーティションの状態確認 |
| `squeue` | ジョブキューの確認 |
| `srun` | ジョブを対話的に実行 |
| `sbatch` | ジョブスクリプトをキューに投入 |

## 1. 導入

```bash
sudo apt update
sudo apt install -y slurm-wlm munge
```

Ubuntuのパッケージは、インストール時にmungeの鍵を自動生成してサービスも起動してくれます。認証が通るか確認します。

```bash
munge -n | unmunge | grep STATUS
```

```
STATUS:          Success (0)
```

これが `Success` ならOKです。

:::note info
`/etc/munge/` は権限700で、一般ユーザーからは中身が見えません。これは正常な状態です。
:::

## 2. 設定ファイル

`/etc/slurm/slurm.conf` を作成します。ここでノードのスペックを宣言します。

```conf
ClusterName=localcluster
SlurmctldHost=ubuntu-host

ProctrackType=proctrack/linuxproc
ReturnToService=2
SlurmctldPidFile=/run/slurmctld.pid
SlurmdPidFile=/run/slurmd.pid
SlurmdSpoolDir=/var/spool/slurmd
SlurmUser=slurm
StateSaveLocation=/var/spool/slurmctld
TaskPlugin=task/none

SchedulerType=sched/backfill
SelectType=select/cons_tres
SelectTypeParameters=CR_Core_Memory

GresTypes=gpu

SlurmctldLogFile=/var/log/slurm/slurmctld.log
SlurmdLogFile=/var/log/slurm/slurmd.log

NodeName=ubuntu-host CPUs=12 Boards=1 SocketsPerBoard=1 CoresPerSocket=6 ThreadsPerCore=2 RealMemory=28000 Gres=gpu:1 State=UNKNOWN
PartitionName=debug Nodes=ALL Default=YES MaxTime=INFINITE State=UP
```

ポイントは2箇所です。

**`GresTypes=gpu`** — GPUを扱うことを宣言します。SlurmではGPUをCPUやメモリと同列には扱わず、**GRES (Generic RESource)** という枠組みで管理します。

**`Gres=gpu:1`** — このノードにGPUが1枚あることを宣言します。

次に `/etc/slurm/gres.conf` で、GRESの実体をデバイスファイルに紐づけます。

```conf
NodeName=ubuntu-host Name=gpu File=/dev/nvidia0
```

ディレクトリと権限を用意します。

```bash
sudo mkdir -p /var/spool/slurmctld /var/spool/slurmd /var/log/slurm
sudo chown slurm:slurm /var/spool/slurmctld /var/log/slurm
```

## 3. 起動 — ここで詰まった

```bash
sudo systemctl start slurmctld slurmd
sinfo
```

```
PARTITION AVAIL  TIMELIMIT  NODES  STATE NODELIST
debug*       up   infinite      1  inval ubuntu-host
```

`inval`。ノードが無効状態です。ログを見ると理由が書かれていました。

```
error: Setting node ubuntu-host state to INVAL with reason:
Low socket*core*thread count, Low CPUs
```

**設定ファイルに書いたCPU数（12）より、Slurmが実際に検出したCPU数が少ない**ということです。

### 原因の切り分け

`lscpu` では確かに12スレッドと表示されます。

```
CPU(s):              12
Thread(s) per core:  2
Core(s) per socket:  6
Socket(s):           1
```

ではSlurmは何を見ているのか。**`slurmd -C`** を実行すると、Slurm自身が検出したハードウェア構成が、そのまま設定ファイルに貼れる形式で出力されます。

```bash
sudo slurmd -C
```

```
NodeName=ubuntu-host slurmd: error: Thread count (11) not multiple of core count (6)
CPUs=11 Boards=1 SocketsPerBoard=1 CoresPerSocket=6 ThreadsPerCore=1 RealMemory=31335
```

**11でした。** `lscpu` は12と言っているのに、Slurmからは11しか見えていません。

11は6（コア数）で割り切れないため、Slurmは `ThreadsPerCore` を1に落として解釈しています。これが設定値（12スレッド、ThreadsPerCore=2）と食い違い、`INVAL` になっていたわけです。

念のためOS側も確認しましたが、こちらは12で問題ありません。

```bash
$ nproc
12
$ cat /sys/fs/cgroup/cpuset.cpus.effective
0-11
```

cgroupによる制限でもない。原因の特定には至りませんでしたが、**Slurmが実際に認識している値に合わせる**のが正解です。

### 解決

設定を `slurmd -C` の出力に合わせます。

```conf
NodeName=ubuntu-host CPUs=11 Boards=1 SocketsPerBoard=1 CoresPerSocket=6 ThreadsPerCore=1 RealMemory=28000 Gres=gpu:1 State=UNKNOWN
```

```bash
sudo systemctl restart slurmctld slurmd
sinfo
```

```
PARTITION AVAIL  TIMELIMIT  NODES  STATE NODELIST
debug*       up   infinite      1   idle ubuntu-host
```

`idle` になりました。

:::note warn
CPUs=11 は 6 で割り切れないため、コマンド実行のたびに
`match no Sockets, Sockets*CoresPerSocket...` という警告が出ます。
Slurmが内部で自動調整するため動作に支障はありませんが、気になる場合は
`CoresPerSocket` を調整してください。
:::

**教訓は「設定値を推測で書かない」ことです。** `lscpu` の値をそのまま転記するのではなく、`slurmd -C` でSlurm自身の検出値を確認してから設定するのが確実でした。

## 4. GPUジョブの実行

まず基本動作の確認。

```bash
$ srun hostname
ubuntu-host
```

次にGPUを要求します。

```bash
$ srun --gres=gpu:1 nvidia-smi
```

```
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 595.84                 Driver Version: 595.84         CUDA Version: 13.2     |
+-----------------------------------------+------------------------+----------------------+
|   0  NVIDIA GeForce RTX 4070        Off |   00000000:01:00.0  On |                  N/A |
+-----------------------------------------+------------------------+----------------------+
```

GPUが割り当てられました。ノード側でも認識できています。

```bash
$ scontrol show node ubuntu-host | grep -i gres
   Gres=gpu:1
```

## 5. キューイングを観察する — 本題

ここからが本番です。**GPUは1枚しかありません。そこにジョブを3つ投げたらどうなるか。**

ジョブスクリプトを用意します。40秒スリープして、GPUを占有し続けるだけの内容です。

```bash
#!/bin/bash
#SBATCH --job-name=gpu-test
#SBATCH --gres=gpu:1
#SBATCH --time=00:05:00
#SBATCH --output=result_%j.log

echo "=== Job $SLURM_JOB_ID start: $(date +%T) ==="
nvidia-smi --query-gpu=name,memory.used --format=csv,noheader
sleep 40
echo "=== Job $SLURM_JOB_ID end: $(date +%T) ==="
```

3つ連続で投入します。

```bash
sbatch gpu_job.sh
sbatch gpu_job.sh
sbatch gpu_job.sh
squeue
```
![slurm-squeue-trimmed.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4472357/9ebae6de-4f4c-4d2d-b18a-d2a1cc862154.png)


```
JOBID PARTITION     NAME     USER ST       TIME  NODES NODELIST(REASON)
    5     debug gpu-test     xxxx PD       0:00      1 (Resources)
    4     debug gpu-test     xxxx  R       0:24      1 ubuntu-host
```

**ジョブ4が実行中（R）、ジョブ5が待機中（PD）**です。理由欄に **`(Resources)`** と表示されています。GPUが空くのを待っている状態です。

### 結果ログ

全ジョブ終了後、出力を確認します。

```
=== Job 3 start: 18:44:25 ===
NVIDIA GeForce RTX 4070, 1047 MiB
=== Job 3 end: 18:45:05 ===

=== Job 4 start: 18:45:06 ===
NVIDIA GeForce RTX 4070, 1061 MiB
=== Job 4 end: 18:45:46 ===

=== Job 5 start: 18:45:47 ===
NVIDIA GeForce RTX 4070, 1154 MiB
```

![squeue_pending_frosted_20260901.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4472357/24db6f74-dfce-4f19-953c-222ecbe53d32.png)


**時刻に注目してください。**

- Job 3 終了 18:45:05 → Job 4 開始 18:45:06
- Job 4 終了 18:45:46 → Job 5 開始 18:45:47

**1秒のギャップで、次のジョブが始まっています。** 3つ同時に投げたのに、一切重なっていません。

普通にシェルから3つ実行すれば、3プロセスが同時にGPUを掴みに行き、メモリ不足やパフォーマンス低下が起きます。それをスケジューラが**直列化した**わけです。前のジョブがGPUを解放したことを検知して、待機中のジョブを即座に投入している。

この1秒のギャップが、スケジューラが働いた証拠です。

## まとめ

- Slurmは**GPUをGRES（Generic RESource）**として扱う。`GresTypes=gpu` と `gres.conf` の2箇所で定義する
- ノードが `INVAL` になったら、`slurmd -C` で**Slurm自身の検出値**を確認する。`lscpu` の値と一致するとは限らない
- GPU 1枚に複数ジョブを投げると、`(Resources)` でキューイングされ、**リソースが空き次第、順番に実行される**

個人環境でGPUを使っている限り「誰かが使っていたら手動で待つ」で済みますが、共有環境ではそれが成立しません。**リソース競合を仕組みで捌く**というのが、スケジューラの存在理由だと実感できました。

次は、GPUの使用率や消費電力を可視化する監視基盤（DCGM-exporter + Prometheus + Grafana）を試す予定です。

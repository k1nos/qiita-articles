---
title: 自宅のRTX 4070にk3sを入れてGPUをPodに割り当てる ― SlurmとKubernetesの考え方の違いを実機で比べる
tags:
  - kubernetes
  - k3s
  - GPU
  - NVIDIA
  - SLURM
private: false
updated_at: '2026-09-23T17:24:37+09:00'
id: 08a5ee6aa1a57de5c9c5
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

# はじめに

前回、[自宅のRTX 4070にSlurmを入れて、GPUジョブのキューイングを体験する](https://qiita.com/k1nos/items/1df4df903481a25fc65b)という記事を書きました。HPC文脈のジョブスケジューラで、GPU1枚に複数ジョブを投げて順番待ちさせる、という内容です。

今回はその続きとして、**同じマシンに k3s（軽量Kubernetes）を入れて、GPUをPodに割り当てる**ところまでをやります。

同じ「GPU 1枚をどう配るか」という課題に対して、HPC（Slurm）とコンテナオーケストレーション（Kubernetes）では考え方がまるで違います。両方を同じ実機で動かしてみると、その違いが具体的に見えてきました。

## この記事でできるようになること

- k3s に NVIDIA GPU をリソースとして認識させる（`nvidia.com/gpu: 1`）
- GPUを要求するPodを起動して、コンテナ内から `nvidia-smi` を通す
- **公式手順どおりにやっても device plugin が起動しない**という罠と、その原因究明
- GPU1枚に2つのPodを投げて、Slurm のキューイングと比べる

とくに3つ目が本記事の肝です。NVIDIA device plugin を Helm で入れたのに DaemonSet が `DESIRED 0` のまま動かない、という状況に遭遇しました。同じところで詰まる人は少なくないはずです。

## 動作環境

| 項目 | 内容 |
|---|---|
| OS | Ubuntu 24.04.4 LTS |
| GPU | NVIDIA GeForce RTX 4070 (VRAM 12GB) |
| CPU | Intel Core i5-12400 |
| メモリ | 約31GB |
| ドライバ | 595.84 / CUDA 13.2 |
| k3s | v1.36.4+k3s1 |
| NVIDIA device plugin | v0.20.0 |

前提として、**Docker + NVIDIA Container Toolkit は導入済み**です。こちらは[別記事](https://qiita.com/k1nos/items/dec1bd95d63b75ff3f09)にまとめています。この記事の手順は、その環境の上に乗せる形になります。

---

# 1. k3s のインストール

k3s は Rancher（SUSE）が開発する軽量Kubernetesです。バイナリ1つで動き、インストールはワンライナーで済みます。

```bash
curl -sfL https://get.k3s.io | sh -
```

1分ほどで完了します。確認します。

```bash
sudo k3s kubectl get node
```

```
NAME          STATUS   ROLES           AGE     VERSION
ubuntu-host   Ready    control-plane   4m56s   v1.36.4+k3s1
```

`Ready` になっていれば成功です。

> 💡 インストール直後に `kubectl get pods -A` を叩いて `No resources found` と出ることがありますが、これは起動が完了する前に見ただけです。少し待てば各Podが立ち上がります。

## sudo なしで kubectl を使う

毎回 `sudo k3s kubectl` と打つのは面倒なので、kubeconfig を自分のホームに配置します。

```bash
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $USER:$USER ~/.kube/config
chmod 600 ~/.kube/config
echo 'export KUBECONFIG=~/.kube/config' >> ~/.bashrc
export KUBECONFIG=~/.kube/config
```

以降は `kubectl` だけで使えます。

## 同梱されるアドオンについて

k3s はデフォルトで Traefik（Ingressコントローラ）と ServiceLB（ロードバランサ）を同梱します。GPU検証だけなら使いませんが、動いていても実害はありません。気になる場合は次のように無効化してインストールできます。

```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable=traefik --disable=servicelb" sh -
```

---

# 2. containerd への NVIDIA ランタイム登録 ― 実は不要だった

ここが最初の関門になる、と身構えていた部分です。

**k3s は Docker ではなく、独自の containerd を使います。** 先に設定した NVIDIA Container Toolkit は Docker 向けの設定だったので、k3s の containerd 側にも別途 NVIDIA ランタイムを登録する必要がある、というのが当初の想定でした。

ネット上の記事を見ると、`/var/lib/rancher/k3s/agent/etc/containerd/config.toml.tmpl` というテンプレートファイルを手書きする手順がよく出てきます。

ところが、**現在の k3s では、この作業は不要でした。**

[公式ドキュメント](https://docs.k3s.io/advanced#nvidia-container-runtime)にこう書かれています。

> K3s will automatically detect alternative container runtimes if they are present when K3s starts.

k3s は起動時に PATH 上の代替ランタイムを自動検出します。対応しているランタイムには `nvidia` が含まれています。

## 確認手順

まず `nvidia-container-runtime` がPATH上にあるか確認します。

```bash
which nvidia-container-runtime
```

```
/usr/bin/nvidia-container-runtime
```

NVIDIA Container Toolkit を入れた時点で、ここに入っています。あとは k3s を再起動するだけです。

```bash
sudo systemctl restart k3s
```

containerd の設定ファイルを見ると、NVIDIA ランタイムが自動で追記されています。

```bash
sudo grep nvidia /var/lib/rancher/k3s/agent/etc/containerd/config.toml
```

```
[plugins.'io.containerd.cri.v1.runtime'.containerd.runtimes.'nvidia']
[plugins.'io.containerd.cri.v1.runtime'.containerd.runtimes.'nvidia'.options]
  BinaryName = "/usr/bin/nvidia-container-runtime"
```

**手書きのテンプレートは一切作っていません。** k3s が勝手にやってくれました。

## RuntimeClass も組み込み済み

Kubernetes で代替ランタイムを使うには `RuntimeClass` というリソースが必要ですが、これも k3s には最初から入っています。

```bash
kubectl get runtimeclass
```

```
NAME                  HANDLER               AGE
crun                  crun                  8d
lunatic               lunatic               8d
nvidia                nvidia                8d
nvidia-experimental   nvidia-experimental   8d
slight                slight                8d
spin                  spin                  8d
wasmedge              wasmedge              8d
wasmer                wasmer                8d
wasmtime              wasmtime              8d
wws                   wws                   8d
```

`nvidia` が最初から存在します。`AGE 8d` は k3s をインストールした日時です。つまり**自分で YAML を書いて作る必要はありません**。

> ⚠️ ネット上には `config.toml.tmpl` を手書きする記事が多く残っていますが、k3s のバージョンによっては不要です。まず `which nvidia-container-runtime` と `grep nvidia config.toml` で確認してから、必要な場合だけ手を動かすのが良いと思います。

---

# 3. NVIDIA device plugin の導入 ― ここで詰まった

containerd がランタイムを認識しても、それだけでは Kubernetes は GPU をリソースとして扱えません。**NVIDIA device plugin** を DaemonSet としてデプロイして、「このノードには GPU が1枚ある」とkubeletに登録させる必要があります。

Helm で入れます。k3s には Helm が同梱されていないので、先に入れます。

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

そして device plugin をインストールします。ここで **`runtimeClassName=nvidia` の指定が必須**です。k3s のデフォルトランタイムは `runc` のままなので、これを指定しないと plugin 自身が GPU を見られません。

```bash
kubectl create namespace nvidia-device-plugin

helm repo add nvdp https://nvidia.github.io/k8s-device-plugin
helm repo update

helm install nvdp nvdp/nvidia-device-plugin \
  --namespace nvidia-device-plugin \
  --set runtimeClassName=nvidia
```

```
NAME: nvdp
LAST DEPLOYED: ...
NAMESPACE: nvidia-device-plugin
STATUS: deployed
REVISION: 1
```

`deployed` と出ました。成功したように見えます。しかし。

```bash
kubectl get pods -n nvidia-device-plugin
```

```
No resources found in nvidia-device-plugin namespace.
```

**Podが1つも起動していません。**

![Helmは成功と表示されるが、Podが1つも起動せずDaemonSetもDESIRED 0のまま](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4472357/4dbbdf8e-1684-46e2-b116-7c16f2577619.png)

## 原因究明

まず DaemonSet の状態を見ます。

```bash
kubectl get daemonset -n nvidia-device-plugin
```

```
NAME                                           DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR
nvdp-nvidia-device-plugin                      0         0         0       0            0           <none>
nvdp-nvidia-device-plugin-mps-control-daemon   0         0         0       0            0           nvidia.com/mps.capable=true
```

DaemonSet は存在します。しかし **`DESIRED` が 0** です。「このDaemonSetを配置すべきノードが1つも無い」とKubernetesが判断している状態です。

`NODE SELECTOR` は `<none>` なので、セレクタが原因ではありません。となると **affinity（ノード選択条件）** が怪しいので、中身を見ます。

```bash
kubectl get daemonset nvdp-nvidia-device-plugin -n nvidia-device-plugin -o yaml | grep -A 30 "affinity:"
```

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: feature.node.kubernetes.io/pci-10de.present
          operator: In
          values:
          - "true"
      - matchExpressions:
        - key: feature.node.kubernetes.io/cpu-model.vendor_id
          operator: In
          values:
          - NVIDIA
      - matchExpressions:
        - key: nvidia.com/gpu.present
          operator: In
          values:
          - "true"
```

![DaemonSetのnodeAffinity。3つのmatchExpressionsのいずれかに合致するノードを探している](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4472357/9849fed1-09b8-4979-ba53-154002ddfad2.png)

**原因が判明しました。** DaemonSet は次の3つのラベルの**いずれか**を持つノードを探しています。

| ラベル | 誰が付けるか |
|---|---|
| `feature.node.kubernetes.io/pci-10de.present=true` | NFD（Node Feature Discovery） |
| `feature.node.kubernetes.io/cpu-model.vendor_id=NVIDIA` | NFD |
| `nvidia.com/gpu.present=true` | 手動でも付けられる |

上2つは **NFD（Node Feature Discovery）** というコンポーネントが自動で付けるものです。NFDはノードのハードウェア構成を検出してラベルを付ける仕組みで、`10de` は NVIDIA の PCI ベンダーIDです。

私は NFD を入れていませんでした。ノードのラベルを確認します。

```bash
kubectl get node ubuntu-host --show-labels
```

```
LABELS
beta.kubernetes.io/arch=amd64,beta.kubernetes.io/instance-type=k3s,
beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,
kubernetes.io/hostname=ubuntu-host,kubernetes.io/os=linux,
node-role.kubernetes.io/control-plane=true,node.kubernetes.io/instance-type=k3s
```

3つのラベルが**どれも付いていません**。だから条件に合うノードが存在せず、`DESIRED 0` になっていたわけです。

## 対処：ノードにラベルを付ける

NFD を追加で導入する手もありますが、単一ノードの検証環境なら**3つ目のラベルを手で付けるのが素直**です。「このノードにはGPUがある」と明示的に宣言する形になります。

```bash
kubectl label node ubuntu-host nvidia.com/gpu.present=true
```

```
node/ubuntu-host labeled
```

ラベルを付けた瞬間に DaemonSet コントローラが検知します。

```bash
kubectl get pods -n nvidia-device-plugin
```

```
NAME                              READY   STATUS              RESTARTS   AGE
nvdp-nvidia-device-plugin-z7mjh   0/1     ContainerCreating   0          0s
```

Podがスケジュールされました。

![ラベル付与後、DESIREDが1に変わりPodがRunningになった](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4472357/d5a4858a-b1c2-4f08-8b12-906afc781638.png)

---

# 4. GPU がリソースとして認識されたか確認

device plugin のログを見ます。

```bash
kubectl logs -n nvidia-device-plugin -l app.kubernetes.io/name=nvidia-device-plugin --tail=20
```

```
I0917 22:06:19.678628       1 plugin-manager.go:101] Detected platform: nvml
I0917 22:06:19.678647       1 plugin-manager.go:50] Using device discovery strategy: nvml
I0917 22:06:19.686670       1 server.go:198] Starting GRPC server for 'nvidia.com/gpu'
I0917 22:06:19.687126       1 server.go:142] Starting to serve 'nvidia.com/gpu' on /var/lib/kubelet/device-plugins/nvidia-gpu.sock
I0917 22:06:19.688314       1 server.go:149] Registered device plugin for 'nvidia.com/gpu' with Kubelet
```

`Detected platform: nvml` で GPU を検出し、`Registered device plugin for 'nvidia.com/gpu' with Kubelet` で kubelet への登録まで完了しています。

ノードの情報を見ます。

```bash
kubectl describe node ubuntu-host | grep -i -A2 -B2 "nvidia.com/gpu"
```

```
Capacity:
  memory:             32087032Ki
  nvidia.com/gpu:     1
  pods:               110
Allocatable:
  memory:             32087032Ki
  nvidia.com/gpu:     1
  pods:               110
```

**`Capacity` と `Allocatable` の両方に `nvidia.com/gpu: 1` が出ました。** GPU が Kubernetes のリソースとして正式に認識された状態です。

CPUやメモリと同じ列に GPU が並んでいるのがポイントです。Kubernetes から見ると、GPU は「割り当て可能なリソースの一種」として抽象化されています。

---

# 5. GPU Pod を起動する

いよいよ実際にGPUを要求するPodを動かします。

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: gpu-test
spec:
  restartPolicy: Never
  runtimeClassName: nvidia
  containers:
  - name: cuda
    image: nvidia/cuda:13.0.0-base-ubuntu24.04
    command: ["nvidia-smi"]
    resources:
      limits:
        nvidia.com/gpu: 1
EOF
```

ポイントは2つです。

- **`runtimeClassName: nvidia`** … これを忘れるとGPUが見えません。k3sのデフォルトランタイムは `runc` のままなので、Pod側で明示的にNVIDIAランタイムを要求します
- **`limits: nvidia.com/gpu: 1`** … GPU 1枚を要求します。Slurm の `--gres=gpu:1` に相当する部分です

状態を見守ります。

```bash
kubectl get pod gpu-test -w
```

```
NAME       READY   STATUS              RESTARTS   AGE
gpu-test   0/1     ContainerCreating   0          0s
gpu-test   1/1     Running             0          17s
gpu-test   0/1     Completed           0          17s
```

`Completed` になりました。`nvidia-smi` を実行して正常終了した、という意味です。`Ctrl+C` で監視を抜けて、ログを見ます。

```bash
kubectl logs gpu-test
```

```
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 595.84                 Driver Version: 595.84         CUDA Version: 13.2      |
+-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
|=========================================+========================+======================|
|   0  NVIDIA GeForce RTX 4070        Off |   00000000:01:00.0  On |                  N/A |
|  0%   41C    P8              8W /  200W |    1555MiB /  12282MiB |      0%      Default |
+-----------------------------------------+------------------------+----------------------+

+-----------------------------------------------------------------------------------------+
| Processes:                                                                              |
|  GPU   GI   CI              PID   Type   Process name                        GPU Memory |
|=========================================================================================|
|  No running processes found                                                             |
+-----------------------------------------------------------------------------------------+
```

**Pod の中から RTX 4070 が見えました。** これで完了です。

> 💡 `No running processes found` と出ますが正常です。コンテナは独立したPID名前空間で動くため、ホスト側のプロセスは見えません。VRAM使用量（1555MiB）にはホスト側の使用分が反映されています。

後片付けします。

```bash
kubectl delete pod gpu-test
```

---

# Slurm と Kubernetes、同じGPUを配る2つの考え方

ここからが本題です。同じマシンで両方を動かしてみて見えた違いを整理します。

## リソース要求の書き方

**Slurm** はコマンドラインオプションで完結します。

```bash
srun --gres=gpu:1 nvidia-smi
```

**Kubernetes** は YAML でマニフェストを書きます。

```yaml
resources:
  limits:
    nvidia.com/gpu: 1
```

Slurm は1行、Kubernetes は Pod 定義全体で十数行。**記述量は明らかに Slurm の方が少ない**です。ただし Kubernetes 側は、イメージ・ネットワーク・ストレージ・再起動ポリシーまで同じ場所で宣言できるので、単純な比較はできません。

## 空きがないときの振る舞い

ここが一番わかりやすい違いでした。

**Slurm** では、GPU 1枚に3ジョブを投入すると、1つが `RUNNING`、残りが `PENDING (Resources)` になります。前のジョブが終わると自動で次が走り、ログのタイムスタンプを見ると1秒のギャップで直列に実行されていました。**キューに並べて順番に流す**、という発想です。

**Kubernetes** でも同じことを試しました。`nvidia.com/gpu: 1` を要求するPodを2つ同時に作ります。

```bash
for i in 1 2; do
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: gpu-queue-$i
spec:
  restartPolicy: Never
  runtimeClassName: nvidia
  containers:
  - name: cuda
    image: nvidia/cuda:13.0.0-base-ubuntu24.04
    command: ["sh", "-c", "nvidia-smi && sleep 60"]
    resources:
      limits:
        nvidia.com/gpu: 1
EOF
done
```

```bash
kubectl get pods
```

```
NAME          READY   STATUS    RESTARTS   AGE
gpu-queue-1   1/1     Running   0          8s
gpu-queue-2   0/1     Pending   0          8s
```

2つ目が `Pending` になりました。理由を見ます。

```bash
kubectl describe pod gpu-queue-2 | tail -6
```

```
Events:
  Type     Reason            Age   From               Message
  ----     ------            ----  ----               -------
  Warning  FailedScheduling  8s    default-scheduler  0/1 nodes are available: 1 Insufficient nvidia.com/gpu. no new claims to deallocate, preemption: 0/1 nodes are available: 1 No preemption victims found for incoming pod.
```

**`0/1 nodes are available: 1 Insufficient nvidia.com/gpu`**。この表現が Kubernetes の発想をよく表しています。「順番待ちしている」ではなく、**「配置できるノードが0個」**という言い方をします。さらに `No preemption victims found`（追い出せるPodも無い）とまで報告してくれます。

![GPU1枚に2つのPodを要求すると、2つ目がPendingになりInsufficient nvidia.com/gpuと報告される](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4472357/c4a624ae-0634-42fc-95eb-53b33ff459e3.png)

結果は似ていますが、モデルが違います。

| | Slurm | Kubernetes |
|---|---|---|
| 単位 | ジョブ（コマンド実行） | Pod（コンテナ） |
| 待ち状態 | `PENDING (Resources)` | `Pending / Insufficient` |
| 発想 | **キューに並べて順番に流す** | **空きノードに配置する** |
| 主戦場 | HPC・大規模学習 | サービング・マイクロサービス |

## 設計思想の違い

Slurm は**時間軸で並べる**仕組みです。バッチジョブが前提で、「いつ終わるか分からないが、いずれ順番が来る」という世界観。大規模な学習ジョブを何日も回すHPC用途に向いています。

Kubernetes は**空間軸で配置する**仕組みです。「どのノードに置くか」を解く問題として扱い、常時稼働するサービスをどう分散させるかが得意。推論サーバーのような、止まらないワークロードに向いています。

どちらが優れているという話ではなく、**解こうとしている問題が違う**のだと実感しました。実際、大規模AIクラスタでは両方を併用する構成（Slinky など）も出てきています。

---

# まとめ

k3s 上で GPU を Pod に割り当てられるようになりました。

## 詰まりどころの整理

| 想定していた作業 | 実際 |
|---|---|
| containerd の `config.toml.tmpl` を手書き | **不要** — k3s が `nvidia-container-runtime` を自動検出 |
| `RuntimeClass` を YAML で作成 | **不要** — k3s に10種類が組み込み済み |
| （想定外） | **ノードに `nvidia.com/gpu.present=true` ラベルが必要** |

**一番ハマったのは3つ目**です。Helm でのインストールは `deployed` と表示されて成功したように見えるのに、Podが1つも起動しない。`kubectl get daemonset` で `DESIRED 0` を見て、`-o yaml` で affinity の中身を読んで、ようやく原因に辿り着きました。

NFD（Node Feature Discovery）を導入している環境なら自動でラベルが付くため、この問題は起きません。逆に言うと、**単一ノードで手軽に試そうとすると必ず踏む**ということです。

## 確認用チェックリスト

同じ構成で試す方は、この順で確認すると詰まりにくいはずです。

- [ ] `which nvidia-container-runtime` でパスが出るか
- [ ] `sudo grep nvidia /var/lib/rancher/k3s/agent/etc/containerd/config.toml` で設定が入っているか
- [ ] `kubectl get runtimeclass` に `nvidia` があるか
- [ ] `kubectl get daemonset -n nvidia-device-plugin` で `DESIRED` が 1 か（0なら**ラベル不足**）
- [ ] `kubectl describe node <ノード名> | grep nvidia.com/gpu` で `1` が出るか
- [ ] Pod に `runtimeClassName: nvidia` を書いたか

## 参考

- [K3s 公式ドキュメント - Advanced Options / NVIDIA Container Runtime](https://docs.k3s.io/advanced#nvidia-container-runtime)
- [NVIDIA/k8s-device-plugin](https://github.com/NVIDIA/k8s-device-plugin)
- [自宅のRTX 4070にSlurmを入れて、GPUジョブのキューイングを体験する](https://qiita.com/k1nos/items/1df4df903481a25fc65b)
- [Ubuntu 24.04 + RTX 4070 で Docker から GPU を使う最短手順](https://qiita.com/k1nos/items/dec1bd95d63b75ff3f09)

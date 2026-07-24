---
title: Isaac Sim 6.0 + Isaac Lab 3.0 でロボットアームの強化学習を回すまで — アセットパス404の罠と解決
tags:
  - IsaacSim
  - IsaacLab
  - NVIDIA
  - Ubuntu
  - Robotics
private: false
updated_at: '2026-07-24T21:38:58+09:00'
id: 648222f82d13cfdb5e93
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

## この記事で扱うこと

NVIDIA Isaac Sim 6.0（Newton物理世代）と Isaac Lab 3.0 beta2 の組み合わせで、Franka Emika Panda アームの Reaching タスクを PPO で学習させた記録です。

道中で **Isaac Sim 6.0 特有のアセットパス問題（404エラー）** に遭遇し、原因究明と修正を行いました。この問題は 2026年7月時点で日本語情報がほぼ存在せず、同じ環境を構築する人が必ず踏む可能性があります。

**結論を先に**: Franka のアセットURLが、Isaac Lab 3.0 beta2 の設定ファイル内で旧パスのまま取り残されており、`franka.py` の1行を修正することで解決しました。

---

## 1. 発生したエラー

Franka Reach タスクの学習を開始したところ、シーン生成の直後に落ちました。

```bash
./isaaclab.sh -p scripts/reinforcement_learning/rsl_rl/train.py \
  --task Isaac-Reach-Franka-v0 --num_envs 1024 --headless
```

```
FileNotFoundError: USD file not found at path:
'https://omniverse-content-production.s3-us-west-2.amazonaws.com/
 Assets/Isaac/6.0/Isaac/IsaacLab/Robots/FrankaEmika/panda_instanceable.usd'
```

ロボットの3Dモデル（USDファイル）が、NVIDIAのクラウドストレージ（AWS S3）から取得できていません。

**紛らわしいのは、同じ環境で ANYmal-C（四足歩行）の学習は問題なく動いていた**ことです。環境構築そのものは正しいはずなのに、Franka だけ落ちる。ここから切り分けが始まりました。

---

## 2. 原因の切り分け

### 2-1. まず「6.0のアセットが未公開なのでは」と疑った（→ 誤り）

curl で直接叩いて確認します。

```bash
curl -sI "https://omniverse-content-production.s3-us-west-2.amazonaws.com/Assets/Isaac/6.0/Isaac/IsaacLab/Robots/FrankaEmika/panda_instanceable.usd" | head -1
# → HTTP/1.1 404 Not Found

curl -sI "https://omniverse-content-production.s3-us-west-2.amazonaws.com/Assets/Isaac/5.1/Isaac/IsaacLab/Robots/FrankaEmika/panda_instanceable.usd" | head -1
# → HTTP/1.1 200 OK
```

5.1 には存在し、6.0 には無い。ここで「**6.0のアセット群がまだS3に公開されていないのでは**」と考え、Isaac Lab の `apps/*.kit` に定義されたアセットルートを 6.0 → 5.1 に書き換える方針を立てかけました。

しかしこれは**誤った判断**でした。

### 2-2. ANYmal が動いていた事実と矛盾する

もし6.0のアセット全体が未公開なら、ANYmal も動かないはずです。この矛盾を潰すため、ANYmal の設定ファイルで実際のパスを確認しました。

```bash
grep -n "usd_path" source/isaaclab_assets/isaaclab_assets/robots/anymal.py
# 95: usd_path=f"{ISAACLAB_NUCLEUS_DIR}/Robots/ANYbotics/ANYmal-C/anymal_c.usd"
```

このパスで叩き直すと——

```bash
curl -sI ".../Assets/Isaac/6.0/Isaac/IsaacLab/Robots/ANYbotics/ANYmal-C/anymal_c.usd" | head -1
# → HTTP/1.1 200 OK
```

**6.0のアセットは正常に公開されていました。** 未公開なのは Franka の特定パスだけだったのです。

> **教訓**: URLを推測で組み立てて検証すると誤診する。必ず設定ファイル内の実際のパス文字列を確認すること。（筆者は `anymal_c/anymal_c.usd` と推測して404を得たが、正解は `ANYmal-C/anymal_c.usd` だった）

### 2-3. 真因 — Franka社の社名変更に伴うパス移動

`franka.py` を見ると、同じファイル内に**2種類のパス**が併存していました。

```python
# 28行目（Reachタスクが使う。404になる）
usd_path=f"{ISAACLAB_NUCLEUS_DIR}/Robots/FrankaEmika/panda_instanceable.usd"

# 91行目（別の設定。新しいパス）
FRANKA_ROBOTIQ_GRIPPER_CFG.spawn.usd_path = f"{ISAAC_NUCLEUS_DIR}/Robots/FrankaRobotics/FrankaPanda/franka.usd"
```

`FrankaEmika` と `FrankaRobotics` — Franka社は社名を **Franka Emika → Franka Robotics** に変更しています。Isaac Sim 6.0 のアセットは新社名のディレクトリに移行済みですが、**Isaac Lab beta2 の一部設定が旧パスのまま取り残されていた**、というのが真因です。

新パスを確認します。

```bash
curl -sI ".../Assets/Isaac/6.0/Isaac/Robots/FrankaRobotics/FrankaPanda/franka.usd" | head -1
# → HTTP/1.1 200 OK  ★存在する

curl -sI ".../Assets/Isaac/6.0/Isaac/Robots/FrankaRobotics/FrankaPanda/franka_instanceable.usd" | head -1
# → HTTP/1.1 404 Not Found  ★instanceable版は無い
```

新パスに `franka.usd` は存在するが、`instanceable`（メモリ効率の良いインスタンス化形式）版は無い。したがって通常版を使う必要があります。

---

## 3. 修正

`apps/*.kit` を一律5.1に書き換える方針は破棄し、**該当の1行だけ**を修正します。副作用が最小で済みます。

```bash
cd ~/IsaacLab/source/isaaclab_assets/isaaclab_assets/robots
cp franka.py franka.py.bak   # バックアップ必須

sed -i '28s|{ISAACLAB_NUCLEUS_DIR}/Robots/FrankaEmika/panda_instanceable.usd|{ISAAC_NUCLEUS_DIR}/Robots/FrankaRobotics/FrankaPanda/franka.usd|' franka.py
```

変数名が `ISAACLAB_NUCLEUS_DIR` → `ISAAC_NUCLEUS_DIR` に変わる点に注意。両者は別のディレクトリを指します。

```python
ISAAC_NUCLEUS_DIR    = f"{NUCLEUS_ASSET_ROOT_DIR}/Isaac"           # .../Isaac
ISAACLAB_NUCLEUS_DIR = f"{ISAAC_NUCLEUS_DIR}/IsaacLab"             # .../Isaac/IsaacLab
```

幸い `franka.py` は20行目で両方importしているため、追加のimportは不要でした。

```bash
grep -n "^from" franka.py
# 20: from isaaclab.utils.assets import ISAAC_NUCLEUS_DIR, ISAACLAB_NUCLEUS_DIR
```

---

## 4. 学習結果

修正後、同じコマンドで学習が通りました。

```bash
./isaaclab.sh -p scripts/reinforcement_learning/rsl_rl/train.py \
  --task Isaac-Reach-Franka-v0 --num_envs 1024 --headless
```

### 学習の推移

| 指標 | 11 iter（序盤） | 1000 iter（完了） |
|---|---|---|
| **success_rate** | 0.2205 | **0.9034** |
| position_error | 0.1883 m | **0.0800 m** |
| orientation_error | 1.3946 | **0.3458** |
| Mean reward | -1.85 | -0.26 |
| Mean action std | 0.97 | **0.07** |

**アームが目標姿勢に90%の確率で到達**できるようになりました。位置誤差8cm、姿勢誤差は4分の1に低減。

注目すべきは `Mean action std` の 0.97 → 0.07 という変化です。方策の出力分散が縮小しており、初期のランダムな探索から、確信を持った滑らかな動作へ収束したことを意味します。

### 性能（RTX 4070 12GB）

| 項目 | 値 |
|---|---|
| 学習時間 | **353.96秒（約6分）** |
| 総ステップ数 | 24,576,000 |
| スループット | **72,451 steps/秒** |
| 並列環境数 | 1,024 |
| iteration time | 0.34秒 |

**12GB VRAMのコンシューマGPUで、1024並列のアーム学習が6分で完走**しました。同環境での他タスクの実測と並べると、12GBの余裕が見えてきます。

| タスク | num_envs | VRAM使用 | GPU使用率 | 学習時間 |
|---|---|---|---|---|
| Cartpole | 4096 | 約35% | 24-28% | 29秒 |
| ANYmal-C Flat（四足歩行） | 1024 | 約35% | 最大48% | 148秒 |
| **Franka Reach（アーム）** | 1024 | — | — | **354秒** |

「推奨VRAM 16GB以上」とされていますが、**これらのタスク範囲では12GBで全く不足しません**。並列数を増やす余地も残っています。

![Isaac Lab 3.0 上で並列実行されるFranka環境。X11セッション + --viz kit によりGUI表示に成功した状態](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4472357/0bd1235f-a65a-48ca-b1c6-280895929cad.png)

![Screenshot from 2026-07-22 01-05-47.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4472357/263d1a92-834c-40a7-acb6-84231ecab0a6.png)

---

## 5. 補足 — Isaac Lab 3.0 での仕様変更

作業中に遭遇した、6.0/3.0世代の変更点も記しておきます。

### GUI表示は `--viz kit` が必須になった

`--headless` は非推奨（DEPRECATED）となり、**省略するとデフォルトでヘッドレス動作**します。GUIを出すには可視化バックエンドを明示します。

```bash
# GUIを表示する（X11セッション必須）
./isaaclab.sh -p scripts/reinforcement_learning/rsl_rl/play.py \
  --task Isaac-Velocity-Flat-Anymal-C-Play-v0 --num_envs 32 --viz kit
```

`--viz` の選択肢は `kit,newton,rerun,viser`。Omniverseのネイティブウィンドウを出すなら `kit` です。

なお **Wayland セッションではGUIウィンドウが表示されません**。X11 で起動する必要があります（`/etc/gdm3/custom.conf` に `WaylandEnable=false`）。

### GUIなしで動画だけ欲しい場合

```bash
./isaaclab.sh -p scripts/reinforcement_learning/rsl_rl/play.py \
  --task Isaac-Velocity-Flat-Anymal-C-Play-v0 --num_envs 32 \
  --headless --video --video_length 300
```
![Screenshot from 2026-07-20 15-07-54.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4472357/43ca7a94-2144-4124-8b81-b4b9607e7cd2.png)

オフスクリーンレンダリングでmp4が生成されます。Wayland/X11を問わず動作し、成果物の記録に便利です。

### Isaac Lab の main ブランチは Isaac Sim 6.0 に非対応（2026年7月時点）

Isaac Sim 6.0 は物理エンジンが PhysX から **Newton** に移行し、tensor API が変わりました。

- 旧: `omni.physics.tensors.impl.api`
- 新: `isaacsim.physics.newton.tensors`

Isaac Lab の main ブランチは凍結されており、6.0対応は `develop` / `3.0-beta` 系で進行中です。main を clone すると `ModuleNotFoundError: No module named 'omni.physics.tensors.impl'` で起動しません。

```bash
git checkout v3.0.0-beta2.patch1   # 6.0対応の安定タグ
./isaaclab.sh --install
```

---

## 6. まとめ

Isaac Sim 6.0 は GA 直後（2026年6月リリース）ということもあり、Isaac Lab 側の追随が部分的に間に合っていない箇所があります。今回の Franka アセットパス問題は、その典型例でした。

**同種のエラーに遭遇したときの手順**をまとめます。

1. エラーメッセージのURLをそのまま `curl -sI` で叩き、404を確認する
2. **設定ファイル内の実際のパス文字列を `grep` で確認する**（URLを推測しない）
3. 同じファイル内に別バージョンのパスが併存していないか探す（社名変更・ディレクトリ再編の痕跡）
4. 動作している他のロボット（今回はANYmal）のパスと比較し、何が違うか特定する
5. 影響範囲が最小になる修正を選ぶ（今回は kit ファイル全体ではなく、該当1行）

そして何より、**「アセット全体が未公開だ」と早合点しかけた**のが最大の反省点です。動いているタスクが存在するという事実と矛盾しないか、常に確認する必要がありました。

12GB のコンシューマGPUでも、四足歩行・マニピュレータともに実用的な速度で強化学習が回ることは確認できました。フィジカルAIの学習環境として、RTX 4070 は十分な選択肢です。

---

## 環境詳細

```
OS:          Ubuntu 24.04.4 LTS (kernel 7.0.0-28-generic)
Session:     X11（WaylandEnable=false）
GPU:         NVIDIA RTX 4070 12GB / driver 595.71.05 / CUDA 13.2
CPU:         Intel Core i5-12400 (6C/12T) ※governor=performance推奨
RAM:         30GB
Isaac Sim:   6.0.1-rc.7
Isaac Lab:   v3.0.0-beta2.patch1
PyTorch:     2.10.0+cu128
```

**その他、事前に必要だった設定**

- IOMMU無効化: GRUB に `intel_iommu=off`（未対応だと画像破損・不安定の警告）
- GCC/G++ 11（Ubuntu 24.04標準の13は非対応）
- CPU governor を `performance` に（再起動で `powersave` に戻るため都度確認）

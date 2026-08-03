---
title: >-
  Isaac Sim 6.0 と ROS 2 Jazzy を接続してシミュレーション内のロボットを双方向制御する ― rclpy loaded
  が出ない・cmd_vel が効かない時の対処つき
tags:
  - IsaacSim
  - NVIDIA
  - ROS2
  - Robotics
  - Ubuntu
private: true
updated_at: '2026-08-03T23:46:48+09:00'
id: 0d52b0126e8f8388aaf6
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

# はじめに

[前回の記事](https://qiita.com/k1nos/items/648222f82d13cfdb5e93)では、Isaac Sim 6.0 + Isaac Lab 3.0 でロボットアームの強化学習を回すところまでを扱いました。

本記事はその続編にあたります。テーマは **「Isaac Sim と ROS 2 を接続し、シミュレーション内のロボットを ROS 2 から動かす」** ことです。強化学習で作った"頭脳"を、いずれ ROS 2 という"神経系"に繋いでいくための、その最初の一歩になります。

## この記事でできるようになること

- Isaac Sim 内のロボットのセンサーデータ（オドメトリ・LiDAR・IMU）を ROS 2 側で受け取る
- ROS 2 から速度指令を送って、Isaac Sim 内のロボットを実際に動かす（双方向通信）
- 上記の過程で **詰まりやすい3つの罠**（`ros2-apt-source` の失敗 / `rclpy loaded` が出ない / `cmd_vel` が効かない）とその対処

Isaac Sim と ROS 2 を繋ぐ日本語の実践記事はまだ少なく、特に ROS 2 **Jazzy** との組み合わせは情報が限られています。同じ構成で詰まっている方の助けになれば幸いです。

## 動作環境

| 項目 | バージョン・型番 |
|---|---|
| OS | Ubuntu 24.04.4 LTS |
| セッション | X11 |
| GPU | NVIDIA RTX 4070 12GB |
| Isaac Sim | 6.0.1 |
| ROS 2 | Jazzy |

> **なぜ Jazzy なのか**
> Isaac Sim 6.0 の ROS 2 ブリッジ拡張は、内部に **Humble と Jazzy の rclpy を内蔵**しています。そのため、システム側の ROS 2 も Humble か Jazzy であることが、スムーズな連携の前提になります。Ubuntu 24.04 では Jazzy が標準ディストリなので、apt で素直に導入できる Jazzy が有力な選択肢になります。

---

# 1. ROS 2 Jazzy の導入 ― `ros2-apt-source` の罠

まず ROS 2 Jazzy をインストールします。

ROS 2 の公式手順では、近年 `ros2-apt-source` という専用パッケージでリポジトリ登録を行う方式が案内されています。しかし、この方式は **バージョン取得の部分で失敗することがあります**。

実際に実行したところ、こうなりました。

```
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
100     9  100     9    0     0     39      0 --:--:-- --:--:-- --:--:--    39

パッケージリストを読み込んでいます... エラー!
E: 不正なアーカイブ署名
E: 内部エラー、メンバー control.tar{.zst,.lz4,.gz,.xz,.bz2,.lzma,} を特定できません
E: Could not read meta data from /tmp/ros2-apt-source.deb
E: パッケージリストまたはステータスファイルを解釈またはオープンすることができません。
```

注目すべきは **ダウンロードサイズが 9 バイト**しかない点です。これは GitHub が `Not Found`（9文字）という文字列を返しただけで、deb ファイルが取得できていないことを意味します。

原因は、リリースのバージョン文字列を取得する処理が空を返していたことでした。

```bash
echo "[$ROS_APT_SOURCE_VERSION]"
# → []   ← 空になっている
```

バージョンが空のまま URL が組み立てられ、存在しないファイルを取得しに行っていたわけです。

## 対処: 手動でリポジトリを登録する

`ros2-apt-source` に頼らず、従来どおり **鍵とリポジトリ定義を手で登録する**方が確実です。

```bash
# 壊れた deb を掃除
rm -f /tmp/ros2-apt-source.deb

# 鍵の取得（未取得の場合）
sudo curl -sSL https://raw.githubusercontent.com/ros/rosdistro/master/ros.key \
  -o /usr/share/keyrings/ros-archive-keyring.gpg

# リポジトリ定義を手動で作成（← ros2-apt-source が代行するはずだった部分）
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/ros-archive-keyring.gpg] http://packages.ros.org/ros2/ubuntu $(. /etc/os-release && echo $UBUNTU_CODENAME) main" | sudo tee /etc/apt/sources.list.d/ros2.list

# 反映して候補が見えるか確認
sudo apt update
apt-cache policy ros-jazzy-desktop
```

`apt update` の出力に `packages.ros.org` の行が現れ、`apt-cache policy` で候補バージョンが表示されれば成功です。

```
取得:3 http://packages.ros.org/ros2/ubuntu noble InRelease [4,676 B]
取得:6 http://packages.ros.org/ros2/ubuntu noble/main amd64 Packages [1,956 kB]

ros-jazzy-desktop:
  インストールされているバージョン: (なし)
  候補:               0.11.0-1noble.20260616.084553
```

あとは本体を入れます。

```bash
sudo apt install -y ros-jazzy-desktop
```

> ⚠️ **つまずきポイント**
> `E: パッケージ ros-jazzy-desktop が見つかりません` と出る場合、鍵はあってもリポジトリ定義（`.list`）が登録できていないことがほとんどです。`sudo apt update` の出力に `packages.ros.org` の行が現れているか確認してください。

### 動作確認

`ros2 --version` というオプションは存在しないので注意してください（ROS 2 の CLI はサブコマンド方式です）。確認するならこちらです。

```bash
source /opt/ros/jazzy/setup.bash
printenv ROS_DISTRO      # → jazzy
ros2 pkg list | head
```

---

# 2. Isaac Sim を ROS 2 に正しく繋ぐ ― `rclpy loaded` を出す

ここが本記事で **最も重要な勘所**です。

Isaac Sim の ROS 2 ブリッジは、起動時にシステムの ROS 2 環境（`rclpy` など）を読み込みます。ところが、**ROS 2 の環境を読み込んでいないシェルから Isaac Sim を起動すると、ブリッジは立ち上がってもトピックが一切見えない**、という状態に陥ります。

## 対処: 起動前に必ず `source` する

Isaac Sim を起動する **同じターミナル**で、先に ROS 2 をセットアップします。

```bash
# このターミナルで先に実行してから Isaac Sim を起動する
source /opt/ros/jazzy/setup.bash

# Isaac Sim 本体を起動
cd ~/isaacsim
./isaac-sim.sh
```

起動が成功しているかは、**ターミナルのログ**で判断できます。以下のように `rclpy loaded` が出れば、Isaac Sim がシステムの Jazzy を正しく掴んでいる証拠です。

```
[48.795s] [ext: isaacsim.ros2.core-1.9.4] startup
[49.116s] Attempting to load system rclpy
[49.266s] rclpy loaded
[49.293s] [ext: isaacsim.ros2.nodes-1.18.13] startup
[50.090s] [ext: isaacsim.ros2.examples-1.2.4] startup
[50.131s] [ext: isaacsim.ros2.ui-1.6.5] startup
[50.209s] [ext: isaacsim.ros2.bridge-5.1.2] startup
...
[183.596s] Isaac Sim Full App is loaded.
[183.604s] app ready
```

> ⚠️ **つまずきポイント：デスクトップのアイコンから起動しない**
> アプリ一覧やデスクトップのアイコンから Isaac Sim を起動すると、`source /opt/ros/jazzy/setup.bash` が効いていない素のシェルで立ち上がります。この場合ブリッジは有効化できてもトピックが見えません。**ROS 2 連携をするときは、必ずターミナルから `source` 済みで起動**してください。

### 起動時の警告について

起動中に以下のような警告が大量に流れますが、**いずれも無害**です。

```
Warning: Possible version incompatibility. Attempting to load omni::fabric::IStageReaderWriter with version v0.16 against v0.14.
[Warning] [omni.kit.material.library.material_library] get_mdl_list_async: mdl_list_cache is not complete
[Warning] [rtx.hydra] Camera '...': Projection type 'fisheyePolynomial' is deprecated.
```

前2つは Isaac Sim 起動時の定番警告、3つ目はサンプルアセットのカメラ設定が新しい命名に変わったことによるものです。最後に `app ready` まで到達していれば問題ありません。

---

# 3. Isaac Sim → ROS 2 ― センサーデータを受け取る

疎通確認には、Isaac Sim に付属する ROS 2 サンプルシーンを使うのが手軽です。ロードした時点で ROS 2 接続用の設定（OmniGraph）が組まれているので、自分でノードを配線する必要がありません。

## サンプルシーンをロードする

メニューから以下を辿ります。

```
Window > Examples > Robotics Examples
  → ROS2 > NAVIGATION > MULTIPLE ROBOTS > Office Scene
```

`Load Sample Scene` を押すと、オフィス環境に3台の Nova Carter ロボットが配置されたシーンが読み込まれます。

初回ロードはアセットをクラウドから取得するため時間がかかります。ビューポート右下の進捗（Physics Tasks）が完了するのを待ってください。

ロードが完了すると、Stage ツリーに以下が並びます。

- `ROS_Clock`（Type: **OmniGraph**）… クロックを publish するノード
- `Nova_Carter_ROS_1` / `_2` / `_3` … 3台のロボット

Stage に `ROS_Clock` という OmniGraph が最初から入っていることが、「配線済みで届く」ことの証拠です。

## 再生してトピックを確認する

ビューポート左の再生ボタン（▶）でシミュレーションを開始したら、**別のターミナル**（同じく `source` 済み）でトピックを確認します。

```bash
source /opt/ros/jazzy/setup.bash
ros2 topic list
```

3台のロボットが、名前空間付きで各種トピックを publish しているのが確認できます。

```
/carter1/back_stereo_imu/imu
/carter1/chassis/imu
/carter1/chassis/odom
/carter1/cmd_vel
/carter1/front_3d_lidar/lidar_points
/carter1/left_stereo_imu/imu
/carter1/right_stereo_imu/imu
/carter1/tf
/carter2/back_stereo_imu/imu
/carter2/chassis/imu
/carter2/chassis/odom
/carter2/cmd_vel
/carter2/front_3d_lidar/lidar_points
/carter2/front_stereo_imu/imu
/carter2/left_stereo_imu/imu
/carter2/right_stereo_imu/imu
/carter2/tf
/carter3/back_stereo_imu/imu
/carter3/chassis/imu
/carter3/chassis/odom
/carter3/cmd_vel
/carter3/front_3d_lidar/lidar_points
/carter3/front_stereo_imu/imu
/carter3/left_stereo_imu/imu
/carter3/right_stereo_imu/imu
/carter3/tf
/clock
/front_stereo_imu/imu
/parameter_events
/rosout
```

オドメトリ（`chassis/odom`）、3D LiDAR の点群（`front_3d_lidar/lidar_points`）、複数の IMU、座標変換（`tf`）、そして速度指令の受け口（`cmd_vel`）まで、ROS 2 の標準的なトピック体系でそのまま出てきます。

データの中身も見てみます。

```bash
ros2 topic echo /clock --once
```

```yaml
clock:
  sec: 12
  nanosec: 816666666
```

```bash
ros2 topic echo /carter1/chassis/odom --once
```

```yaml
header:
  stamp:
    sec: 13
    nanosec: 916666666
  frame_id: odom
child_frame_id: base_link
pose:
  pose:
    position:
      x: 0.015413742605782857
      y: -2.441227044682993e-07
      z: -2.391048712901325e-05
    orientation:
      x: -1.0648259530435087e-06
      y: -2.814986575408459e-07
      z: 3.3244187451536713e-06
      w: 0.9999999999938676
  covariance:
  - 0.0
  # （以下 0.0 が続くため省略）
twist:
  twist:
    linear:
      x: 0.0006956724682440558
      y: -0.0004678957795525851
      z: -6.638147641598393e-05
    angular:
      x: 0.002006194554269314
      y: 3.0712111765751615e-05
      z: 0.0009362563141621649
```

`frame_id: odom` → `child_frame_id: base_link` という標準的な構造で、再生直後なので位置はほぼ原点、姿勢の `w` もほぼ 1.0（無回転）です。速度がわずかにゼロでないのは物理シミュレーションの微小な揺らぎで、逆に「本当にシミュレーションが動いている」証拠でもあります。

ここまでで **Isaac Sim → ROS 2 のデータ経路が通りました**。

---

# 4. ROS 2 → Isaac Sim ― ロボットを動かす（双方向）

次は逆方向です。ROS 2 から速度指令を送って、Isaac Sim 内のロボットを動かします。

`cmd_vel` トピックに `geometry_msgs/msg/Twist` を publish します。まずは前進しながら旋回させてみます。

```bash
source /opt/ros/jazzy/setup.bash
ros2 topic pub /carter1/cmd_vel geometry_msgs/msg/Twist \
  "{linear: {x: 0.3}, angular: {z: 0.2}}"
```

```
publishing #34: geometry_msgs.msg.Twist(linear=geometry_msgs.msg.Vector3(x=0.3, y=0.0, z=0.0), angular=geometry_msgs.msg.Vector3(x=0.0, y=0.0, z=0.2))
publishing #35: ...
```

## `cmd_vel` が効かない…と思ったときの切り分け

ここで多くの人が「指令を送っているのにロボットが動かない」と戸惑います。私もそうでした。落ち着いて切り分けましょう。

### ステップ1: 指令が届いているか

```bash
ros2 topic echo /carter1/cmd_vel --once
```

```yaml
linear:
  x: 0.3
  y: 0.0
  z: 0.0
angular:
  x: 0.0
  y: 0.0
  z: 0.2
```

送った Twist がそのまま表示されれば、publish 自体は成功しており、宛先のトピック名も合っています。

### ステップ2: 受け取る側（Isaac Sim）がいるか

```bash
ros2 topic info /carter1/cmd_vel -v
```

```
Type: geometry_msgs/msg/Twist

Publisher count: 1
  Node name: _ros2cli_39252
  Topic type: geometry_msgs/msg/Twist
  QoS profile:
    Reliability: RELIABLE

Subscription count: 1
  Node name: _World_Nova_Carter_ROS_1_differential_drive_ros2_subscribe_twist
  Node namespace: /carter1
  Topic type: geometry_msgs/msg/Twist
  QoS profile:
    Reliability: RELIABLE
```

ここが切り分けの決め手です。

- **`Subscription count: 1`** … 購読者がいる
- **ノード名 `..._differential_drive_ros2_subscribe_twist`** … Isaac Sim 側の差動二輪駆動グラフが購読している
- **型が両側とも `geometry_msgs/msg/Twist`** で一致、**QoS も両側 RELIABLE** で揃っている

つまり ROS 2 のレイヤーでは、指令は間違いなく Isaac Sim の駆動グラフに届いています。

> 💡 もし `Subscription count: 0` なら、シーン側に cmd_vel を受ける配線が無いか、シミュレーションが再生されていません。型が `TwistStamped` など異なる場合は、送る型を合わせる必要があります。

## 「動いて見えない」のオチ

配線は正常なのにロボットが動いて見えない場合、原因は制御ではなく **見え方**であることがあります。私の場合はこれでした。

odom の位置を監視すると、実際には動いていることが数値で分かります。

```bash
ros2 topic echo /carter1/chassis/odom --field pose.pose.position
```

```yaml
x: 1.0077807593149235
y: 0.3767861885842349
z: -0.0015914701716885837
---
x: 1.0110880423743835
y: 0.38045454677560925
z: -0.0015966689692782324
---
x: 1.0143829274764236
y: 0.38413816356886105
z: -0.0016018332580311058
---
x: 1.017656354811449
y: 0.38783608515287776
z: -0.0016069040091508254
```

`x` も `y` も一貫して増え続けています。これは指令どおり **前進しながら旋回している** ことを示しています。

**画面で気づきにくかった理由**は3つでした。

- 指定した速度（0.3 m/s）が遅く、広い空間では1〜2mの移動が目立たない
- 旋回半径が大きく（linear ÷ angular = 0.3 ÷ 0.2 = 1.5 m）、なだらかな弧を描くだけ
- ビューポートのカメラが固定で、ロボットが遠くにいると画面上の変化が小さい

その場で分かりやすく動かすなら、前進をやめて**その場旋回**にすると、遠くにいても回転が見えます。

```bash
ros2 topic pub /carter1/cmd_vel geometry_msgs/msg/Twist \
  "{linear: {x: 0.0}, angular: {z: 1.0}}"
```

これで **ROS 2 → Isaac Sim の制御経路も通り、双方向通信が実証**できました。

> 💡 **補足：指令を送ったロボットだけが動く**
> `/carter1/cmd_vel` に送れば carter1 だけが動きます。他の2台は誰も指令を送っていないので静止したままです。3台とも動かすには、それぞれの名前空間の `cmd_vel` に送ります。

---

# まとめ

Isaac Sim 6.0 と ROS 2 Jazzy を接続し、シミュレーション内のロボットを双方向で通信・制御できることを確認しました。

- **Isaac Sim → ROS 2**：`/clock`、オドメトリ、3D LiDAR、IMU、TF が標準的なトピックとして流れる
- **ROS 2 → Isaac Sim**：`cmd_vel` への Twist で実際に車輪が回り、odom の値が変化する

## 再現チェックリスト

同じ構成で試す方は、以下を順に確認すると詰まりにくいです。

- [ ] ROS 2 Jazzy がインストールできた（`ros2-apt-source` が失敗したら手動 `.list` 登録）
- [ ] Isaac Sim 起動前に `source /opt/ros/jazzy/setup.bash` した
- [ ] 起動ログに `rclpy loaded` が出た
- [ ] サンプルシーンを再生して `ros2 topic list` にトピックが出た
- [ ] `ros2 topic echo /clock` でシミュレーション時刻が流れた
- [ ] `cmd_vel` を publish して odom の position が変化した

## 次のステップ

本記事では NVIDIA 製の既製サンプルシーンを使って疎通を確認しました。次は、**前回の記事で強化学習させた自分のポリシー（Franka / ANYmal）を、この ROS 2 の配管に乗せる**ことに挑戦する予定です。学習した"頭脳"を ROS 2 の"神経系"に繋ぐ、シリーズの本題にあたる部分です。

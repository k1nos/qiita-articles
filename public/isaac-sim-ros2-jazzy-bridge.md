---
title: >-
  Isaac Sim 6.0 と ROS 2 Jazzy を接続してシミュレーション内のロボットを双方向制御する ― rclpy loaded
  が出ない・cmd_vel が効かない時の対処つき
tags:
  - IsaacSim
  - ROS2
  - NVIDIA
  - Ubuntu
  - Robotics
private: true
updated_at: '2026-08-03T07:45:35+09:00'
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

<!-- ※ Isaac Simの正確なバージョン表記を確認。~/isaacsim/VERSION を cat -->

> **なぜ Jazzy なのか**
> Isaac Sim 6.0 の ROS 2 ブリッジ拡張は、内部に **Humble と Jazzy の rclpy を内蔵**しています。そのため、システム側の ROS 2 も Humble か Jazzy であることが、スムーズな連携の前提になります。Ubuntu 24.04 では Jazzy が標準ディストリなので、apt で素直に導入できる Jazzy が有力な選択肢になります。

---

# 1. ROS 2 Jazzy の導入 ― `ros2-apt-source` の罠

まず ROS 2 Jazzy をインストールします。

ROS 2 の公式手順では、近年 `ros2-apt-source` という専用パッケージでリポジトリ登録を行う方式が案内されています。しかし、この方式は **バージョン取得の部分で失敗することがあります**。

<!-- ※ ros2-apt-source 失敗時のエラー出力を貼る -->

```
（※ ここに ros2-apt-source 失敗時のエラー出力を貼る。例: E: 不正なアーカイブ署名 など）
```

原因は、リリースのバージョン文字列を取得する処理が空を返し、壊れた URL からダウンロードしてしまうことにありました。

## 対処: 手動でリポジトリを登録する

`ros2-apt-source` に頼らず、従来どおり **鍵とリポジトリ定義を手で登録する**方が確実です。

```bash
# 鍵の取得（未取得の場合）
sudo curl -sSL https://raw.githubusercontent.com/ros/rosdistro/master/ros.key \
  -o /usr/share/keyrings/ros-archive-keyring.gpg

# リポジトリ定義を手動で作成
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/ros-archive-keyring.gpg] http://packages.ros.org/ros2/ubuntu $(. /etc/os-release && echo $UBUNTU_CODENAME) main" | sudo tee /etc/apt/sources.list.d/ros2.list

# 反映して候補が見えるか確認
sudo apt update
apt-cache policy ros-jazzy-desktop
```

`apt-cache policy` で候補バージョンが表示されれば、リポジトリ登録は成功です。あとは本体を入れます。

```bash
sudo apt install -y ros-jazzy-desktop
```

> ⚠️ **つまずきポイント**
> `E: パッケージ ros-jazzy-desktop が見つかりません` と出る場合、鍵はあってもリポジトリ定義（`.list`）が登録できていないことがほとんどです。`sudo apt update` の出力に `packages.ros.org` の行が現れているか確認してください。

<!-- ※ シミュレーション用に入れた依存パッケージも書くと親切:
     ros-jazzy-turtlebot3 / ros-jazzy-turtlebot3-gazebo / ros-jazzy-twist-stamper など -->

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

起動が成功しているかは、**ターミナルのログ**で判断できます。以下の2行が出れば、Isaac Sim がシステムの Jazzy を正しく掴んでいる証拠です。

```
Attempting to load system rclpy
rclpy loaded
```

<!-- ※ 実際の起動ログのスクショを貼る。isaacsim.ros2.bridge の startup 行と rclpy loaded の行 -->

> ⚠️ **つまずきポイント：デスクトップのアイコンから起動しない**
> アプリ一覧やデスクトップのアイコンから Isaac Sim を起動すると、`source /opt/ros/jazzy/setup.bash` が効いていない素のシェルで立ち上がります。この場合ブリッジは有効化できてもトピックが見えません。**ROS 2 連携をするときは、必ずターミナルから `source` 済みで起動**してください。

<!-- ※ 起動時の警告について一言: "omni::fabric version incompatibility" や
     "mdl_list_cache is not complete" は無害な定番警告。"app ready" まで出ればOK -->

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

<!-- ※ ロード後のビューポート＋Stageツリー（ROS_Clock / Nova_Carter_ROS_1〜3）のスクショ -->

初回ロードはアセットをクラウドから取得するため時間がかかります。ビューポート右下の進捗（Physics Tasks）が完了するのを待ってください。

## 再生してトピックを確認する

ビューポート左の再生ボタン（▶）でシミュレーションを開始したら、**別のターミナル**（同じく `source` 済み）でトピックを確認します。

```bash
source /opt/ros/jazzy/setup.bash
ros2 topic list
```

3台のロボットが、名前空間付きで各種トピックを publish しているのが確認できます。

```
（※ ここに ros2 topic list の実際の出力を貼る）
```

データの中身も見てみます。

```bash
ros2 topic echo /clock --once
ros2 topic echo /carter1/chassis/odom --once
```

<!-- ※ /clock と odom の echo 出力を貼る -->

`/clock` にシミュレーション時刻が、`odom` にロボットの姿勢が流れていれば、**Isaac Sim → ROS 2 のデータ経路が通っています**。

---

# 4. ROS 2 → Isaac Sim ― ロボットを動かす（双方向）

次は逆方向です。ROS 2 から速度指令を送って、Isaac Sim 内のロボットを動かします。

`cmd_vel` トピックに `geometry_msgs/msg/Twist` を publish します。まずは前進しながら旋回させてみます。

```bash
source /opt/ros/jazzy/setup.bash
ros2 topic pub /carter1/cmd_vel geometry_msgs/msg/Twist \
  "{linear: {x: 0.3}, angular: {z: 0.2}}"
```

## `cmd_vel` が効かない…と思ったときの切り分け

ここで多くの人が「指令を送っているのにロボットが動かない」と戸惑います。私もそうでした。落ち着いて切り分けましょう。

**まず、指令が本当に届いているか**を確認します。

```bash
ros2 topic echo /carter1/cmd_vel --once
```

送った Twist（`x: 0.3`, `z: 0.2`）がここに表示されれば、publish 自体は成功しています。

**次に、受け取る側（Isaac Sim）がいるか**を確認します。

```bash
ros2 topic info /carter1/cmd_vel -v
```

出力の `Subscription count` が 1 以上で、購読者のノード名に `..._differential_drive_ros2_subscribe_twist` のような名前が見えれば、**Isaac Sim 側の駆動グラフが指令を購読しています**。型（`geometry_msgs/msg/Twist`）も一致しているか確認してください。

<!-- ※ ros2 topic info -v の出力を貼る。Subscription count: 1 とノード名 -->

## 「動いて見えない」のオチ

配線は正常なのにロボットが動いて見えない場合、原因は制御ではなく **見え方**であることがあります。私の場合はこれでした。

odom の位置をリアルタイムで監視すると、実際には動いていることが数値で分かります。

```bash
ros2 topic echo /carter1/chassis/odom --field pose.pose.position
```

<!-- ※ position.x が連続増加していく echo 出力を貼る -->

`position.x` が少しずつ増えていれば、ロボットは指令どおり動いています。**画面で気づきにくかった理由**は2つでした。

- 指定した速度（0.3 m/s）が遅く、広い空間では1〜2mの移動が目立たない
- ビューポートのカメラが固定で、ロボットが遠くにいると画面上の変化が小さい

その場で分かりやすく動かすなら、前進をやめて**その場旋回**にすると、遠くにいても回転が見えます。

```bash
ros2 topic pub /carter1/cmd_vel geometry_msgs/msg/Twist \
  "{linear: {x: 0.0}, angular: {z: 1.0}}"
```

<!-- ※ その場旋回しているロボットのスクショ or 動画キャプチャ -->

これで **ROS 2 → Isaac Sim の制御経路も通り、双方向通信が実証**できました。

> 💡 **補足：指令を送ったロボットだけが動く**
> `/carter1/cmd_vel` に送れば carter1 だけが動きます。他の2台は誰も指令を送っていないので静止したままです。3台とも動かすには、それぞれの名前空間の `cmd_vel` に送ります。

---

# まとめ

Isaac Sim 6.0 と ROS 2 Jazzy を接続し、シミュレーション内のロボットを双方向で通信・制御できることを確認しました。

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

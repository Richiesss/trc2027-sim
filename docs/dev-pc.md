# 開発用機のセットアップ

鳥取ローバーチャレンジ2027のソフトウェアを開発するマシンの準備手順です。

> **2026/11/26 更新**：開発用機 `trc-dev`（Tailscale 100.82.40.65 / 大学VPN 150.89.235.232）は、JetPack 7.2.1（Ubuntu 24.04）に焼き直し、
> **ROS 2 Jazzy を直接**入れました（Docker は使いません。外部デバイスや RViz を扱いやすくするため）。
> trc-dev では、下の「trc-dev（Jazzy を直接使う）」の手順に従ってください。
> 「Docker を使う場合」の手順は、JetPack 6 の機体や、Docker でシミュレーションしたい普通の PC 向けに残しています。

## 役割分担

| マシン | 使い道 |
| --- | --- |
| **開発用機**（ubuntu-developer） | ROS 2 のプログラムを書き、シミュレーション（Gazebo）で試す。このページの対象 |
| 本番用 Jetson（ローバーに載せる） | 駆動テストやセンサの確認など、実機でしかできない作業。ハードウェア開発と共用 |
| Mac | 開発用機・本番用 Jetson に SSH（Tailscale）でつないで作業する手元の端末 |

どちらのマシンでも ROS 2 **Jazzy** を Docker で動かすので、開発用機で動いたコードはそのまま本番用 Jetson に持っていけます。

## trc-dev（Jazzy を直接使う）

### リポジトリの配置

```bash
mkdir -p ~/trc2027 && cd ~/trc2027
git clone https://github.com/Richiesss/trc2027-rover.git
git clone https://github.com/Richiesss/trc2027-sim.git
```

### 本番のコード（trc2027-rover）

ビルドと実行の手順は trc2027-rover の [docs/setup-native.md](https://github.com/Richiesss/trc2027-rover/blob/main/docs/setup-native.md) を参照。

### シミュレーション（Gazebo + TurtleBot3）を直接動かす

Docker イメージに入れているものと同じパッケージを、apt で入れます。

```bash
sudo apt update
sudo apt install -y ros-jazzy-turtlebot3-simulations ros-jazzy-turtlebot3-teleop
echo 'export TURTLEBOT3_MODEL=burger' >> ~/.bashrc && source ~/.bashrc

ros2 launch turtlebot3_gazebo turtlebot3_world.launch.py
# 別ターミナルで
ros2 run turtlebot3_teleop teleop_keyboard
```

- Jetson では Gazebo がかなり重くなります。画面はモニタをつないで見るか、軽いワールドで試してください。
- SSH だけでは Gazebo の画面は Mac に出ません（リモートで画面を見る方法は未整備）。

---

## Docker を使う場合（JetPack 6 の機体・普通の PC）

## 1. PCの基本設定

Ubuntu は 22.04 / 24.04 のどちらでも構いません（ROS 2 はコンテナの中で動くため）。

```bash
lsb_release -a                          # Ubuntu のバージョンを確認
sudo apt update && sudo apt upgrade -y
sudo apt install -y git curl openssh-server
sudo systemctl enable --now ssh
```

マシンの種類を確認しておきます（Gazebo の表示が重いときの判断材料）。

```bash
uname -m                                # aarch64 なら Jetson などのARM機、x86_64 なら通常のPC
cat /etc/nv_tegra_release               # Jetson なら L4T のバージョンが出る
lspci | grep -i -E "vga|3d"             # 通常のPCなら GPU が出る（Jetson は内蔵GPUなので何も出ない）
```

## 2. Docker を入れる

Jetson を SDカードイメージでセットアップした場合は、Docker は最初から入っています。`docker --version` で確認し、出なければ入れます。

```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER           # sudo なしで docker を使う
```

一度ログアウトしてログインし直し、確認します。

```bash
docker run --rm hello-world
docker compose version
```

## 3. Mac から入れるようにする（Tailscale + SSH）

Jetson と同じ方法です。`ubuntu-developer` にはすでに Tailscale が入っているので、名前の変更と鍵の登録だけで済みます。

開発用機側：

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up                       # 表示されるURLを Mac のブラウザで開いてログイン（Jetsonと同じアカウント）
sudo hostnamectl set-hostname trc-dev   # ホスト名（プロンプトに出る名前）
sudo tailscale set --hostname=trc-dev   # Tailscale上の名前
tailscale ip -4                         # 100.x.y.z を控える
```

Tailscale の管理画面（login.tailscale.com）→ Machines → このマシンの「…」→ **Disable key expiry** で、ログインの期限切れを防ぎます。

Mac側（鍵を登録して、パスワードなしで入れるようにする）：

```bash
ssh-copy-id developer@trc-dev
ssh developer@trc-dev
```

`~/.ssh/config` に登録しておくと、VS Code の Remote-SSH からも選べます。

```
Host trc-dev
  HostName trc-dev
  User developer
  IdentityFile ~/.ssh/id_ed25519
  ForwardAgent yes
```

## 4. リポジトリを並べて clone する

作業用のフォルダ `~/trc2027` を作り、その中に2つのリポジトリを**並べて**置きます。

```bash
mkdir -p ~/trc2027
cd ~/trc2027
git clone https://github.com/Richiesss/trc2027-sim.git
git clone https://github.com/Richiesss/trc2027-rover.git
```

```
~/trc2027/
├── trc2027-sim/     # シミュレーション環境（Gazebo + TurtleBot3）
└── trc2027-rover/   # 本番のコード（rover/）と Jetson 用の設定
```

2つが同じフォルダに並んでいれば、`.env` の `ROVER_REPO=../trc2027-rover` はそのまま使えます。

## 5. シミュレーション環境を作る

```bash
cd ~/trc2027/trc2027-sim
cp .env.example .env
```

`.env` を開いて、次のように書き換えます。

```bash
# ネイティブUbuntu ＋ 本番のコード（trc2027-rover/rover）をワークスペースに追加
COMPOSE_FILE=compose.yaml:compose.linux.yaml:compose.rover.yaml
ROVER_REPO=../trc2027-rover

USER_UID=1000        # `id -u` の結果
USER_GID=1000        # `id -g` の結果
RENDER_GID=992       # `getent group render` の3番目の数字

# Jetson とつなぐときは、Jetson 側（trc2027-rover の .env）と同じ番号にする
ROS_DOMAIN_ID=27
```

イメージを作ります（初回は10分ほどかかります）。

```bash
docker compose build
```

## 6. 動かしてみる

開発用機の画面（モニタ）の前で作業する場合：

> Jetson（aarch64）では Gazebo がかなり重くなります。画面が真っ黒・すぐ落ちる場合は、`.env` の `COMPOSE_FILE` に `:compose.software-render.yaml` を足し、`LIBGL_ALWAYS_SOFTWARE=1` にして試してください（ソフトウェア描画は遅いので、軽いワールドで確認する）。

```bash
xhost +local:                           # コンテナから画面に描画できるようにする（ログインごとに必要）
docker compose up -d
docker compose exec ros bash

# --- コンテナ内 ---
ros2 launch turtlebot3_gazebo turtlebot3_world.launch.py
```

別のターミナルで `docker compose exec ros bash` → `ros2 run turtlebot3_teleop teleop_keyboard` を実行し、ロボットが動けば完了です。

本番のコードがコンテナから見えているかも確認します。

```bash
# --- コンテナ内 ---
ls ~/ws/src/rover                       # trc2027-rover/rover の中身が見えればOK
```

## 7. 普段の開発の流れ

1. Mac の VS Code で、Remote-SSH から `trc-dev` につなぐ
2. `~/trc2027/trc2027-rover/rover/` に ROS 2 パッケージを作って編集する
3. コンテナ内でビルドしてシミュレーションで試す
   ```bash
   cd ~/ws
   colcon build --symlink-install
   source install/setup.bash
   ```
4. 動いたら **trc2027-rover** にコミット・push する
5. 本番用 Jetson で `git pull` し、実機で確認する（trc2027-rover の `docs/docker.md`）

## Mac から Gazebo の画面を見たいとき

SSH だけでは Gazebo のウィンドウは Mac に出ません。自宅から画面も見たい場合は、開発用機の画面を VNC で共有する方法を別途設定します（`scripts/start_remote_gazebo.sh` は WSL2 向けのため、ネイティブUbuntu では手順が変わります）。必要になったら追記します。

## よくあるトラブル

| 症状 | 対処 |
| --- | --- |
| `permission denied ... docker.sock` | `docker` グループに入っていない。`sudo usermod -aG docker $USER` のあと、ログインし直す |
| Gazebo のウィンドウが出ない | ホストで `xhost +local:` を実行したか確認する |
| Gazebo が真っ黒・すぐ落ちる | `.env` の `RENDER_GID` を `getent group render` の値に合わせ、`docker compose up -d --force-recreate` |
| `~/ws/src/rover` が空 | `.env` の `COMPOSE_FILE` に `compose.rover.yaml` が入っているか、`ROVER_REPO` のパスが正しいか確認する |
| 本番用 Jetson のトピックが見えない | 開発用機と本番用 Jetson の `ROS_DOMAIN_ID` が同じか、同じLANにいるか確認する（Tailscale 越しでは自動発見が通らない） |

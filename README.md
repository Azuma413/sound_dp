# sound_dp
音環境認識DiffusionPolicy

Python3.10

## Setup
- 環境のセットアップ
```bash
git clone --recurse-submodules https://github.com/Azuma413/sound_dp.git
cd sound_dp
uv sync
uv pip install -e "Genesis/[dev]"
uv pip install -e "lerobot/[feetech]"
uv pip uninstall torch torchvision
uv pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu128
```
ffmpegのインストール
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install ffmpeg -y
```
- USBデバイスのセットアップ
Follower用とLeader用のサーボドライバをそれぞれPCに接続し、適当にそれぞれのデバイスの名前を調べる。
```bash
ls /dev/ttyA*
```
次に`lerobot/lerobot/common/robot_devices/robots/configs.py`の`So100RobotConfig`を編集する。
```python
class So100RobotConfig(ManipulatorRobotConfig):
    calibration_dir: str = ".cache/calibration/so100"
    max_relative_target: int | None = None

    leader_arms: dict[str, MotorsBusConfig] = field(
        default_factory=lambda: {
            "main": FeetechMotorsBusConfig(
                port="/dev/ttyACM0", # 変更
                motors={
                    # name: (index, model)
                    "shoulder_pan": [1, "sts3215"],
                    "shoulder_lift": [2, "sts3215"],
                    "elbow_flex": [3, "sts3215"],
                    "wrist_flex": [4, "sts3215"],
                    "wrist_roll": [5, "sts3215"],
                    "gripper": [6, "sts3215"],
                },
            ),
        }
    )

    follower_arms: dict[str, MotorsBusConfig] = field(
        default_factory=lambda: {
            "main": FeetechMotorsBusConfig(
                port="/dev/ttyACM1", # 変更
                motors={
                    # name: (index, model)
                    "shoulder_pan": [1, "sts3215"],
                    "shoulder_lift": [2, "sts3215"],
                    "elbow_flex": [3, "sts3215"],
                    "wrist_flex": [4, "sts3215"],
                    "wrist_roll": [5, "sts3215"],
                    "gripper": [6, "sts3215"],
                },
            ),
        }
    )

```
- モーターのセットアップ
ドライバにボーレートとIDを設定したいモーターを1つ接続した状態で以下のコマンドを実行する。
```
uv run lerobot/lerobot/scripts/configure_motor.py --port /dev/ttyACM0 --brand feetech --model sts3215 --baudrate 1000000 --ID 1
```
`Permission denied`と表示される際は以下のコマンドで権限を付与しておく。
```bash
sudo chmod 666 /dev/ttyACM0
```
- キャリブレーション
```bash
python lerobot/lerobot/scripts/control_robot.py \
  --robot.type=so100 \
  --robot.cameras='{}' \
  --control.type=calibrate \
  --control.arms='["main_follower"]'
```
```bash
python lerobot/lerobot/scripts/control_robot.py \
  --robot.type=so100 \
  --robot.cameras='{}' \
  --control.type=calibrate \
  --control.arms='["main_leader"]'
```
- 動作確認
```bash
python lerobot/lerobot/scripts/control_robot.py \
  --robot.type=so100 \
  --robot.cameras='{}' \
  --control.type=teleoperate
```
- カメラの確認
```bash
sudo apt install v4l2loopback-dkms v4l-utils
v4l2-ctl --list-devices
uv run lerobot/lerobot/common/robot_devices/cameras/opencv.py \
    --images-dir outputs/images_from_opencv_cameras \
    --record-time-s 0.5
```
使いたいカメラに合わせて`lerobot/lerobot/common/robot_devices/robots/configs.py`の`So100RobotConfig`を編集する。
```python
    cameras: dict[str, CameraConfig] = field(
        default_factory=lambda: {
            "webcam": OpenCVCameraConfig(
                camera_index=2,
                fps=30,
                width=640,
                height=480,
            ),
        }
    )
```
以下のコマンドで映像を表示しながら遠隔操作できる。
```bash
uv run lerobot/lerobot/scripts/control_robot.py \
  --robot.type=so100 \
  --control.type=teleoperate \
  --control.display_data=true
```
## データセットの作成
データセット収集を開始します：
```bash
uv run lerobot/lerobot/scripts/control_robot.py \
  --robot.type=so100 \
  --control.type=record \
  --control.fps=30 \
  --control.single_task="[タスクの説明]" \
  --control.repo_id=local/[データセット名] \
  --control.root=datasets/[データセット名] \
  --control.warmup_time_s=5 \
  --control.episode_time_s=60 \
  --control.reset_time_s=30 \
  --control.num_episodes=50 \
  --control.push_to_hub=false \
  --control.resume=false \
  --control.display_data=true
```

ex.
```bash
uv run lerobot/lerobot/scripts/control_robot.py \
    --robot.type=so100 \
    --control.type=record \
    --control.fps=30 \
    --control.single_task="Pick up and place audio source" \
    --control.repo_id=local/real_sound \
    --control.root=datasets/real_sound \
    --control.warmup_time_s=5 \
    --control.episode_time_s=20 \
    --control.reset_time_s=10 \
    --control.num_episodes=1 \
    --control.push_to_hub=false \
    --control.resume=false \
    --control.display_data=true
```

主な引数の説明：

- `--control.fps`: 1秒あたりのフレーム数（デフォルト：ポリシーのfps）
- `--control.single_task`: データ収集時のタスクの説明（例：「レゴブロックを掴んで右のボックスに入れる」）
- `--control.repo_id`: データセットの識別子。通常は`{hf_username}/{dataset_name}`の形式
- `--control.warmup_time_s`: データ収集開始前のウォームアップ時間。ロボットデバイスの準備と同期のために使用（デフォルト：10秒）
- `--control.episode_time_s`: 各エピソードの記録時間（デフォルト：60秒）
- `--control.reset_time_s`: 各エピソード後の環境リセット時間（デフォルト：60秒）
- `--control.num_episodes`: 記録するエピソード数（デフォルト：50）
- `--control.push_to_hub`: HuggingFace hubへのアップロード（デフォルト：true）
- `--control.tags`: hubでのデータセットのタグ（オプション）
- `--control.video`: フレームをビデオとしてエンコード（デフォルト：true）
- `--control.display_data`: カメラ映像の表示（デフォルト：false）
- `--control.play_sounds`: 音声合成によるイベント読み上げ（デフォルト：true）
- `--control.resume`: 既存のデータセットへの追加収集（デフォルト：false）

## データセットの作成（シミュレーション）
```bash
uv run src/make_sim_dataset.py
```
`libEGL warning: failed to open /dev/dri/renderD128: Permission denied`という表示が出る場合は、以下を実行
```bash
sudo usermod -aG render $USER
```

## 学習の実行
先にwandbにログインしておく
```bash
wandb login
```
policyはact, diffusion, pi0, pi0fast, tdmpc, vqbetのいずれか。
学習の安定性を高めるためにbatch sizeはVRAMが許す限り大きくした方が良い。
```bash
export DATASET_NAME=[データセット名]
export POLICY=act
uv run lerobot/lerobot/scripts/train.py \
  --dataset.repo_id=local/${DATASET_NAME} \
  --dataset.root=datasets/${DATASET_NAME} \
  --policy.type=$POLICY \
  --output_dir=outputs/train/${POLICY}-${DATASET_NAME} \
  --job_name=${POLICY}-${DATASET_NAME} \
  --policy.device=cuda \
  --wandb.enable=true \
  --wandb.disable_artifact=true \
  --batch_size=8 \
  --steps=100000
```
オプション
```bash
  --env.type=test \
  --env.task=test \
  --eval_freq=5000 \
  --eval.n_episodes=10 \
  --eval.batch_size=1
```
env.typeとenv.taskの関係については`lerobot/lerobot/common/envs/configs.py`を参照
- ※pi0を使う場合は以下の手順が必要
[こちら](https://huggingface.co/google/paligemma-3b-pt-224)にアクセスして、利用規約に賛同する。
```bash
uv run huggingface-cli login
```
- stepsとepochの関係
例えば30fpsで60秒のデータセットを50個用意した場合、全体で90000フレームになるので1epoch=90000sampleとなる。
ここでbatch_sizeを8としていた場合、1stepの学習で8sampleが消費されるため、1epoch=1125stepsとなる。
```
steps = エポック数 * (データfps * データ長さ * データ数) / バッチサイズ
```

学習を再開するときは以下のようにする。
前回学習時と同じstepsにすると、なにも学習せずに終わるので注意。
```bash
uv run lerobot/lerobot/scripts/train.py \
  --config_path=outputs/train/act_so100_test/checkpoints/last/pretrained_model/train_config.json \
  --resume=true \
  --steps=150000
```
## ポリシーの評価
- デスクトップで学習した重みの転送
デスクトップのWSLからノートPCのUbuntuへ重みを転送する。wslではmDNSの名前解決が出来ないので注意。
```bash
rsync -avz --progress outputs user_name@ip:~/path/to/sound_dp
```
- ポリシーの実行
```bash
python lerobot/lerobot/scripts/control_robot.py \
  --robot.type=so100 \
  --control.type=record \
  --control.fps=4 \
  --control.single_task="spread a piece of cloth" \
  --control.repo_id=local/eval_diffusion_spread-cloth \
  --control.root=datasets/eval_spread-cloth \
  --control.warmup_time_s=5 \
  --control.episode_time_s=180 \
  --control.reset_time_s=10 \
  --control.num_episodes=1 \
  --control.push_to_hub=false \
  --control.policy.path=outputs/train/diffusion_spread-cloth/checkpoints/last/pretrained_model \
  --control.display_data=true
```

## ポリシーの評価（シミュレーション）
```bash
uv run src/eval_policy.py
```

## [SO-100](lerobot/lerobot/examples/10_use_so100.md)

## Memo
SO-100のURDFは以下のリポジトリから取ってきた。\
https://github.com/TheRobotStudio/SO-ARM100

genesisでcudaを使うにはcuda toolkitが必要。
- ~/.bashrc
```bash
export PATH="/usr/local/cuda/bin:$PATH"
export LD_LIBRARY_PATH="/usr/lib/wsl/lib:$LD_LIBRARY_PATH"
```

サーバーで長時間のタスクを実行するときはscreenが便利
```bash
screen

uv run src/make_sim_dataset.py
```
`Ctrl+A`, `Ctrl+D`を押す。\
`screen -r`でセッション復帰。\
`screen -ls`でセッション確認。\
`exit`でセッション終了

## TODO
- [ ] dummyのeval用環境を作る
- [x] cubeBが最初から箱に入っている問題に対処する
- [x] サブモジュールをもとのリポジトリをフォークしたものに変更する
- [x] lerobot/lerobot/common/robot_devices/cameras/sound.pyの開発
- [x] 音を増やした環境の作成
- [ ] eval policyで取得される画像を、もっと引いたものにする
- [x] control_robot.pyで音タスク実行中に、ランダムなスピーカーから音がなるようにする。
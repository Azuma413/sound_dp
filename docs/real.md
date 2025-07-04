# 現実環境での動かし方
## ハードウェア
実機実験には[SO-100](https://github.com/TheRobotStudio/SO-ARM100)ロボットを利用．

計算機はRTX4060 Laptopを利用.

## セットアップ
- USBデバイスのセットアップ\
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
- モーターのセットアップ\
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

## ポリシーの評価
- 学習した重みの転送
wslではmDNSの名前解決が出来ないので注意。
```bash
rsync -avz --progress gmo:/home/user_00054_25b505/SourceCode/sound_dp/outputs/train/act-sound-ep100_0 outputs/train
```
- ポリシーの実行
```bash
uv run lerobot/lerobot/scripts/control_robot.py \
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
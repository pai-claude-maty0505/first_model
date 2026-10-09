# SO-101 Pick & Place: imitation learning end-to-end with ROS 2 × Gazebo × LeRobot (Colab)

Physical AI 講座 2026 応用編1 最終課題（omnicampus: maty0505）。SO-101 アームがテーブル上のランダムな位置にある赤いブロックを掴み、青いトレイに入れるタスクを、
**データ作成 → 学習 → 推論（自律実行）** の全工程で実装したものです。Google Colab（T4 GPU）だけで再現できます。

## パイプライン

```
 ┌──────────── Gazebo Harmonic (headless) ────────────┐
 │ SO-101 (official URDF→SDF) / block / tray / 2 cameras│
 └───────────────▲──────────────────────┬──────────────┘
          関節指令 │ ros_gz_bridge        │ 画像・関節状態
 ┌───────────────┴──────────────────────▼──────────────┐
 │ ROS 2 node  so101_task_node.py                       │
 │   collect: IK expert → raw episodes (.npz)           │
 │   eval   : TCP client  ──────────┐                    │
 └──────────────────────────────────┼────────────────────┘
   to_lerobot.py → LeRobotDataset   │  image + joint state → action
   lerobot-train (ACT, Colab GPU)   ▼
                         policy_server.py (LeRobot ACT, torch)
```

| ファイル | 内容 |
|---|---|
| `PAI_final_SO101_ROS2_Gazebo_LeRobot.ipynb` | Colab ノートブック（全工程を順に実行） |
| `docs/説明資料_PAI最終課題_maty0505.pdf` | 説明資料（レポート） |
| `sim/so101_kin.py` | URDF を直接読む FK/IK（数値 IK、トップダウン把持姿勢） |
| `sim/build_sim.py` | URDF→SDF 変換と後処理、ワールド生成 |
| `sim/so101_task_node.py` | ROS 2 ノード（データ収集 / ポリシー実行、把持アシスト、成功判定） |
| `sim/bridge.yaml` | ROS 2 ⇔ Gazebo トピック対応 |
| `sim/snapshot.py` | 動作確認（関節指令とカメラ画像） |
| `learning/to_lerobot.py` | 生データ → LeRobotDataset (v3) |
| `learning/policy_server.py` | 学習済み ACT の推論サーバ |
| `learning/make_video.py` | 提出用動画の自動合成 |

## 実行方法
1. `PAI_final_SO101_ROS2_Gazebo_LeRobot.ipynb` を Colab で開き、ランタイムを **T4 GPU** にする
   （コードはこのリポジトリから自動で取得されます。Drive に `PAI_final_code` フォルダがあればそちらを優先）
2. 上から順に実行（環境構築 約15分 → データ収集 約15〜20分 → 学習 約40〜60分 → 評価・動画 約10分）
3. 成果物（データセット・チェックポイント・動画・コード zip）は Google Drive の `MyDrive/PAI_final/` に保存される

## 結果

![自律実行の様子](docs/eval5.jpg)

| 項目 | 値 |
|---|---|
| エキスパートデモ | 61 回中 60 回成功、4,765 フレーム（10 Hz） |
| ACT 学習 | 10,000 ステップ（バッチ 32）、損失 0.367 → 0.066 |
| 自律実行（未知の配置 10 試行） | **成功率 90 %（9/10）** |

ポリシーの入力はカメラ画像（160×120）と関節角 6 次元のみで、ブロックの座標は与えていません。
動画は等倍速（シミュレーション時間）です。

## 工夫した点
- **ROS 2 と AI の分離**：rclpy はシステム Python、LeRobot/torch は Colab の Python で動くため、ROS 2 ノードと推論サーバを TCP で接続。実機での「ロボット側 ⇔ 推論サーバ」構成と同じ形。
- **シミュレーション時間に同期した記録**：制御ループをカメラフレーム（sim 時間 10 Hz）で駆動し、ヘッドレス描画で実時間より遅くなっても観測と行動の対応が崩れない。
- **自動デモ生成**：数値 IK によるエキスパートがブロック位置・トレイ上の置き位置・各区間の所要ステップをランダム化して軌道を生成。人手のテレオペなしでデータを作成。
- **ポリシーにブロック位置を与えない**：入力はカメラ画像と関節角のみ。物体の位置は画像から推定させる。
- **把持アシスト**：Gazebo での摩擦把持は不安定なため、「グリッパを閉じた時に指先がブロックの近くにある」ときだけ DetachableJoint で固定するルールを採用。データ収集時と評価時で同じルールを使用（成否はポリシーの動作次第）。
- **ヘッドレス描画のフォールバック**：GPU (EGL) で描画できない場合は Xvfb + ソフトウェア描画に自動切替。

## 使用モデル・ライブラリ
- SO-101 モデル：[TheRobotStudio/SO-ARM100](https://github.com/TheRobotStudio/SO-ARM100)（Apache-2.0）
- [LeRobot](https://github.com/huggingface/lerobot) 0.6.1（ACT）
- ROS 2 Jazzy, Gazebo Harmonic, ros_gz（Colab: Ubuntu 24.04 / ROS は Python 3.12、LeRobot は Colab 標準の Python 3.13）

## AI の活用
- 模倣学習ポリシー ACT（LeRobot）がロボットを制御
- 開発過程で生成 AI アシスタントをコード作成・デバッグの補助に使用
# first_model

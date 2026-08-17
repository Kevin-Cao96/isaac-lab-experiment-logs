# 实验笔记：Ant RL + Franka Mimic 模仿学习（2026-08-17）

日期：2026-08-17
环境：root@10-60-140-42 云服务器（NVIDIA GPU），Isaac Lab 3.0（v3.0.0-beta2 系），Isaac Lab 源码位于 `/root/IsaacLab`

## 一、目标

1. 用 rl_games PPO 跑通 `Isaac-Ant-v0` 强化学习训练，并学会用新 CLI 加载 checkpoint 回放。
2. 跑通 Isaac Lab Mimic 遥操作 + 模仿学习流水线：录示范 → 标注 → Mimic 生成数据 → Robomimic BC 训练 → rollout 评估。
3. 解决远程服务器上 WebRTC livestream 黑屏、headless 下 `omni.ui` 缺失等环境问题。

## 二、实验 1：Ant 强化学习

任务：`Isaac-Ant-v0`，训练库：rl_games（PPO）

训练命令（旧脚本方式）：

```bash
cd /root/IsaacLab
./isaaclab.sh -p scripts/reinforcement_learning/rl_games/train.py --task Isaac-Ant-v0 --headless
```

结果：

| 指标 | 值 | 说明 |
|---|---|---|
| epochs | 500/500 | 达到配置上限，正常退出 |
| 每 epoch frames | 65,536 | 总量约 3,280 万帧 |
| fps step 均值 | 210,259 | 纯仿真步进 |
| fps step+infer 均值 | 184,335 | 包含策略推理 |
| fps total 均值 | 159,621 | 包含 PPO 更新 |
| 最优 reward | 68.74 | epoch 480 附近 |
| 最终 reward | 53.08 | epoch 500 |
| checkpoint 目录 | `logs/rl_games/ant/2026-08-17_11-29-05/nn/` | `ant.pth` 为最优，`last_ant_ep_*` 为周期快照 |

结论：训练闭环正常，性能很好；但 500 epochs 后仍未稳定收敛，reward 波动大，后续可加长训练或调参。

回放命令（新版统一 CLI，注意 rl_games 没有 `--load_run` 参数）：

```bash
bash isaaclab.sh play \
  --rl_library rl_games \
  --task Isaac-Ant-v0 \
  --num_envs 32 \
  --checkpoint /root/IsaacLab/logs/rl_games/ant/2026-08-17_11-29-05/nn/ant.pth
```

远程看画面：

```bash
PUBLIC_IP=117.50.187.235 bash isaaclab.sh play \
  --rl_library rl_games \
  --task Isaac-Ant-v0 \
  --num_envs 1 \
  --checkpoint /root/IsaacLab/logs/rl_games/ant/2026-08-17_11-29-05/nn/ant.pth \
  --livestream 1 \
  --real-time
```

## 三、实验 2：Franka 叠方块 Mimic 流水线

任务：`Isaac-Stack-Cube-Franka-IK-Rel-v0`（目标：蓝 → 红 → 绿 顺序叠三个方块）

### 1. 遥操作录制

键盘遥操作记录命令：

```bash
cd /root/IsaacLab
./isaaclab.sh -p scripts/tools/record_demos.py \
  --task Isaac-Stack-Cube-Franka-IK-Rel-v0 \
  --device cpu \
  --teleop_device keyboard \
  --dataset_file ./datasets/dataset.hdf5 \
  --num_demos 10 \
  --livestream 1 \
  --cloudxr_env none
```

手工键盘遥操作叠三块很难，只完成 1 条成功示范；改用官方预录数据集：

```bash
cd /root/IsaacLab/datasets
curl -L -o dataset.hdf5 \
  "https://omniverse-content-production.s3-us-west-2.amazonaws.com/Assets/Isaac/5.1/Isaac/IsaacLab/Mimic/franka_stack_datasets/dataset.hdf5"
```

### 2. 自动标注

```bash
cd /root/IsaacLab
./isaaclab.sh -p scripts/imitation_learning/isaaclab_mimic/annotate_demos.py \
  --device cpu \
  --task Isaac-Stack-Cube-Franka-IK-Rel-Mimic-v0 \
  --auto \
  --input_file ./datasets/dataset.hdf5 \
  --output_file ./datasets/annotated_dataset.hdf5
```

### 3. Mimic 数据生成

小批量验证：

```bash
./isaaclab.sh -p scripts/imitation_learning/isaaclab_mimic/generate_dataset.py \
  --device cpu \
  --num_envs 10 \
  --generation_num_trials 10 \
  --input_file ./datasets/annotated_dataset.hdf5 \
  --output_file ./datasets/generated_dataset_small.hdf5
```

结果：10 条成功 / 30 次尝试，成功率 33.3%。

完整生成（目标 1000 条成功，中途在 600 条成功时手动停止）：

```bash
./isaaclab.sh -p scripts/imitation_learning/isaaclab_mimic/generate_dataset.py \
  --device cpu \
  --headless \
  --num_envs 20 \
  --generation_num_trials 1000 \
  --input_file ./datasets/annotated_dataset.hdf5 \
  --output_file ./datasets/generated_dataset.hdf5
```

停止时：600 条成功 / 1674 次尝试，成功率 35.8%。

日志格式说明：`成功数/尝试数 (成功率)`，该任务 `generation_guarantee=True`，停止条件是成功数达到目标，不是尝试数。

### 4. Robomimic BC 训练

```bash
cd /root/IsaacLab
./isaaclab.sh -i robomimic

./isaaclab.sh -p scripts/imitation_learning/robomimic/train.py \
  --task Isaac-Stack-Cube-Franka-IK-Rel-v0 \
  --algo bc \
  --dataset ./datasets/generated_dataset.hdf5
```

训练配置：`bc_rnn_low_dim.json` 默认 `num_epochs=2000`，每 100 epochs 存一次 checkpoint。

训练目录示例：

```text
logs/robomimic/Isaac-Stack-Cube-Franka-IK-Rel-v0/bc_rnn_low_dim_franka_stack/20260817203822/models/
├─ model_epoch_100.pth
├─ ...
├─ model_epoch_1100.pth
```

### 5. Rollout 评估

```bash
./isaaclab.sh -p scripts/imitation_learning/robomimic/play.py \
  --device cpu \
  --task Isaac-Stack-Cube-Franka-IK-Rel-v0 \
  --num_rollouts 50 \
  --checkpoint /root/IsaacLab/logs/robomimic/Isaac-Stack-Cube-Franka-IK-Rel-v0/bc_rnn_low_dim_franka_stack/20260817203822/models/model_epoch_1100.pth \
  --livestream 1 \
  --visualizer kit
```

## 四、问题与修复记录

| 问题 | 原因 | 修复 |
|---|---|---|
| 旧 `play.py` 报 `--load_run` 未识别 | rl_games play 没有该参数，是 rsl_rl 的 | 用新 CLI `isaaclab.sh play --rl_library rl_games`，或用 `--checkpoint` |
| WebRTC 连上有信号但黑屏 | `visualizer=kit` 未默认启用，RTX/Hydra 画面未绑定到 livestream 编码器（官方 issue #5364） | 加 `--visualizer kit` / `--viz kit` |
| `record_demos.py` 报 `ModuleNotFoundError: omni.ui` | headless 体验文件不含 UI 扩展 | 加 `--livestream 1` 使用完整 Kit 体验 |
| 终端出现 `zenity: not found`、thread_init 报错 | `record_demos.py` 默认自动启动 CloudXR | 加 `--cloudxr_env none` |
| `robomimic/play.py` livestream 几秒后黑屏 | rollout 循环没有调用渲染，无新帧给编码器 | 在 `env.step(actions)` 后补 `env.unwrapped.sim.render()` |
| checkpoint 找不到 | 路径少写了 `models/` 子目录 | 使用 `.../models/model_epoch_1100.pth` |

## 五、结论

1. RL 链路：Ant 训练/回放已跑通，FPS 和收敛趋势正常，但尚未充分收敛。
2. Mimic 链路：官方 10 条示范 → 自动标注 → 数据生成 → BC 训练 → rollout 全流程已跑通。
3. 数据生成成功率约 33-36%，符合官方参考量级；600 条数据可先验证 BC 是否学会。
4. 远程 WebRTC 观看依赖 `--visualizer kit`，且部分脚本需要补渲染调用。

## 六、下一步建议

- 用多个中间 checkpoint（900/1000/1100 epoch）分别做 rollout，找成功率最高的模型。
- 若 600 条数据效果不够，继续生成到 1000 条成功示范再训。
- 手工遥操作若继续做，建议接 SpaceMouse 或改用更顺手的设备，并降低 `--step_hz`。
- 可把 Mimic 生成的 HDF5 转成 LeRobot 格式，衔接 OpenPI / π0.5 微调流程。

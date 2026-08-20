# 实验笔记：Franka 叠方块 + OpenPI/LeRobot 完整链路（2026-08-20）

日期：2026-08-19 ~ 2026-08-20
环境：root@10-60-140-42 云服务器，RTX 3080 Ti 12GB，12 核 CPU，Ubuntu 22.04
软件：Isaac Lab 3.0（v3.0.0-beta2 系），Isaac Sim 6.x，Robomimic，OpenPI + LeRobot

## 一、目标

1. 用当前 Isaac Lab 环境生成 Franka 叠方块示范，训练 state-based BC，验证“数据 → 训练 → 评估”闭环。
2. 学习 Isaac Lab Mimic、Augmented Imitation（Cosmos）、SkillGen 三条模仿学习路线。
3. 把 Isaac Lab HDF5 数据转成 LeRobot 格式，接入 OpenPI，跑通 policy server + client 部署。

## 二、关键结论（先看这里）

1. **官方预生成数据在当前环境不可用**：官方 HDF5/模型来自另一个 Isaac Lab build，相机视角与当前环境不一致，官方模型和用它训练的自训练模型评估都是 0/10。
2. **当前环境自己生成的数据可用**：100 条当前环境示范训练 state-based BC，成功率从 5% 逐步提升；扩到 400 条后稳定在约 35%±5%。
3. **数据量是成功率的主要瓶颈**：100 条 → 600 epochs 上限约 14%；400 条 → 600 epochs 约 34-46%。
4. **OpenPI 部署链路已跑通**：pi05_libero 官方 checkpoint + websocket server + client，推理约 113ms，约 6.6 it/s。
5. **LeRobot 转换链路已跑通**：Isaac Lab HDF5 → LeRobotDataset，50 episodes / 12089 frames 验证通过。

## 三、实验 1：当前环境 BC 训练

### 1. 数据生成

用当前环境的 Mimic 生成 100 条成功示范：

```bash
./isaaclab.sh -p scripts/imitation_learning/isaaclab_mimic/generate_dataset.py \
  --device cpu \
  --enable_cameras \
  --headless \
  --num_envs 4 \
  --generation_num_trials 100 \
  --input_file ./datasets/annotated_dataset_visuomotor.hdf5 \
  --output_file ./datasets/mimic_dataset_100.hdf5 \
  --task Isaac-Stack-Cube-Franka-IK-Rel-Visuomotor-Cosmos-Mimic-v0 \
  --rendering_mode performance
```

结果：100 条成功 / 273 次尝试，成功率 36.6%。

### 2. State-based BC 训练与评估

训练命令：

```bash
./isaaclab.sh -p scripts/imitation_learning/robomimic/train.py \
  --task Isaac-Stack-Cube-Franka-IK-Rel-v0 \
  --algo bc \
  --dataset /root/IsaacLab/datasets/mimic_dataset_100.hdf5 \
  --name bc_current_env_fast \
  --epochs 600
```

评估结果：

| 数据量 | Epoch | Rollouts | 成功率 |
|---:|---:|---:|---:|
| 100 | 100 | 20 | 5% |
| 100 | 300 | 50 | 10% |
| 100 | 600 | 50 | 14% |
| 400 | 600 | 50 | 46% |
| 400 | 600 | 100 | 36% |
| 400（seed 202） | 600 | 50 | 34% |

综合结论：真实成功率约 **35%±5%**；数据量从 100 → 400 条带来明显提升。

### 3. 官方数据兼容性排查

- 官方 `cosmos_dataset_1k.hdf5`（2000 条）动作范围 [-1,1]，与原始 10 条示范一致，归一化不是 0 分原因。
- 官方 checkpoint 单独喂数据集第一帧，能输出正常动作（arm 小量、gripper ≈1），说明模型本身没问题。
- 但官方模型在本地环境评估 0/10，robomimic 推理也不会二次处理图像，最终定位为**官方数据和模型来自不同 Isaac Lab build，相机视角不一致**。

## 四、实验 2：OpenPI + LeRobot

### 1. 安装 OpenPI

```bash
cd /root
git clone --recurse-submodules https://github.com/Physical-Intelligence/openpi.git
cd openpi
GIT_LFS_SKIP_SMUDGE=1 uv sync
GIT_LFS_SKIP_SMUDGE=1 uv pip install -e .
```

### 2. Isaac Lab HDF5 → LeRobot

编写 `convert_isaac_hdf5_to_lerobot.py`，将 `mimic_dataset_400.hdf5` 转换为 LeRobotDataset：

- 图像：`table_cam` → `image`，`wrist_cam` → `wrist_image`（200×200 HWC uint8）
- 状态：`eef_pos + eef_quat + gripper_pos` → `state`（9 维）
- 动作：7 维 relative IK + gripper
- 任务文本：`"stack blue red green cubes"`

验证结果：

```text
episodes: 50
frames: 12089
features: image / wrist_image / state / actions / timestamp / episode_index
```

### 3. pi05_libero checkpoint 下载

- GCS 单连接下载太慢（ETA 数小时），`gsutil` 还因 crcmod 问题失败。
- 解决方案：使用 Hugging Face 镜像 `bf-jeon/pi05_libero`（与 GCS orbax 结构一致）：
  ```bash
  export HF_ENDPOINT=https://hf-mirror.com
  uv run huggingface-cli download bf-jeon/pi05_libero \
    --local-dir /root/.cache/openpi/openpi-assets/checkpoints/pi05_libero
  ```
- OpenPI 实际缓存路径是 `/root/.cache/openpi/openpi-assets/checkpoints/pi05_libero`，不是 `/root/.cache/openpi/checkpoints/...`，第一次下错导致重复下载。
- gsutil 残留 `*.gstmp` 临时文件占约 8GB，`find ... -name '*.gstmp' -delete` 清理。

### 4. Policy Server + Client

服务端：

```bash
cd /root/openpi
uv run scripts/serve_policy.py --env LIBERO
```

客户端：

```bash
cd /root/openpi
uv run examples/simple_client/main.py --env LIBERO
```

结果：

```text
policy_infer_ms  113ms
server_infer_ms  149ms
推理速度         约 6.6 it/s
```

## 五、踩坑记录

| 问题 | 原因 | 修复 |
|---|---|---|
| 官方模型/数据评估 0/10 | 官方来自不同 Isaac Lab build，相机视角不一致 | 当前环境重新生成数据 |
| gsutil 报 crcmod 错误 | 未装 C 扩展，拒绝下载 composite object | `[GSUtil] check_hashes = never` 跳过校验 |
| GCS 下载慢 | 国际带宽单连接受限 | 改用 HF 镜像 `bf-jeon/pi05_libero` |
| checkpoint 缓存重复下载 | 缓存路径少了 `openpi-assets/` 一层 | 放到 `~/.cache/openpi/openpi-assets/checkpoints/pi05_libero` |
| 磁盘被占满 | `*.gstmp`、失败 HDF5、旧数据集 | 删除临时文件与无用 HDF5 |
| LeRobot 转换慢 | 图像视频编码 CPU 密集 | 提高 `image_writer_processes`，先转 50/100 条验证 |

## 六、下一步

1. 为 `local/franka_stack_100` 编写自定义 OpenPI 训练配置，把 `image/wrist_image/state/actions/task` 映射进 π0.5。
2. 运行 `compute_norm_stats.py` 计算归一化统计。
3. 在 A100/H100（或 4090×2）上微调 π0.5（3080 Ti 12GB 只能推理）。
4. 微调后 serve 自定义 checkpoint，写自定义 client 接回 Isaac Lab 或真机。

## 七、个人总结

今天最大的教训是：**官方预训练数据和模型不一定能在本地环境复现，必须优先验证“数据-环境一致性”**。当前环境自生成数据 + BC 已经拿到 35% 左右的可信结果，OpenPI 部署链路也跑通，接下来只差大显存 GPU 上的正式微调。

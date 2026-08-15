# 实验笔记：Isaac Lab Lift 任务独立跑通

日期：2026-08-14
环境：AutoDL Ubuntu-Nvidia 虚机，RTX 3080 Ti 12G，Isaac Sim 6.0.1 + Isaac Lab 3.0.0-beta2

## 一、目标

独立完成研究生布置式任务：从定位任务到跑通训练、看懂指标、下结论，全程不依赖他人提示。

## 二、过程

1. 定位任务代码：`~/IsaacLab/source/isaaclab_tasks/isaaclab_tasks/manager_based/manipulation/lift`
2. 确认任务注册 ID：`config/franka/__init__.py` 中 `gym.register(id="Isaac-Lift-Cube-Franka-v0", ...)`
3. 确认训练库：`train.py` 中 `LIBRARY_ENTRYPOINTS` 字典的 key 即 `--rl_library` 可选值（rl_games / rlinf / rsl_rl / sb3 / skrl）
4. 选库依据：任务 `agents/` 目录官方配置了 `rsl_rl_ppo_cfg.py`，故用 rsl_rl
5. 环境数依据：12G 显存，128 个并行环境安全

## 三、训练命令

```bash
cd ~/IsaacLab && bash isaaclab.sh -p scripts/reinforcement_learning/train.py \
  --task Isaac-Lift-Cube-Franka-v0 --num_envs 128 --rl_library rsl_rl \
  --max_iterations 1000
```

## 四、结果（第 999/1000 次迭代）

| 指标 | 值 | 含义 |
|---|---|---|
| Total steps | 3,072,000 | 训练总量 |
| Mean reward | 0.78 | 平均奖励 |
| success_rate | 0.0000 | 0% 完成率 |
| lifting_object | 0.12 | 部分帧举起方块 |
| object_goal_tracking | 0.022 | 几乎没送到目标 |
| position_error | 0.33 | 末端离目标 0.33m |
| episode length | 250 | 每轮跑满超时 |

## 五、结论

**任务未成功。** success_rate = 0，一次都没把方块送到目标。

但中间状态有信息量：lifting_object = 0.12 说明机械臂学会"部分举起方块"，object_goal_tracking 接近 0 说明"送方块到目标"没学会。属于半成品策略：会够、会举，不会完成完整流程。

**未成功原因：** 1000 迭代（约 300 万步）对 Lift 任务不够。之前 1500 迭代（900 万步）同样是 0，该任务通常需要数千万步。

## 六、收获

1. 任务 ID、训练库、环境数都能通过代码/配置查出来，不需要背
2. 判断"学没学会"要看 success_rate，而不只看 reward
3. 分项奖励（lifting / tracking）能定位"学到哪一步"
4. 独立完成链路：找任务 → 确认注册 → 组命令 → 跑训练 → 看指标 → 下结论

## 七、下一步建议

- 增加 max_iterations（如 5000）观察 success_rate 是否转正
- 或用 play.py 加载 checkpoint 看行为（headless 下看日志/指标）
- 进阶：改奖励权重做对比实验，观察指标变化

---

# 补充实验：Reach 任务（2026-08-14 晚）

## 实验 1：Isaac-Reach-Franka-v0，500 迭代

命令：`train.py --task Isaac-Reach-Franka-v0 --num_envs 128 --rl_library rsl_rl --max_iterations 500`

结果（第 499/500 迭代）：
- Mean reward: -1.10
- success_rate: **0.0833（8.3%，非零！）**
- position_error: 0.22m
- 时间：约 5 分钟

结论：**首次出现非零成功率**。reach 比 lift 简单（只要求末端到达目标，不要求抓取），500 轮就开始学会。

## 实验 2：同任务，2000 迭代

结果（第 1999/2000 迭代）：
- Mean reward: -0.76
- success_rate: **0.0417（4.2%，反而下降）**
- position_error: 0.25m（变大）
- action std: 0.29 → 0.11（策略变保守）
- entropy: 1.20 → -5.83（探索减少）

结论：**训练过度/策略收敛过头**。更多训练不一定更好，success_rate 不单调上升。验证"看曲线不看终点"。

## 关键教训

1. 判断学习效果看 success_rate（任务完成度），不看 reward 正负
2. reward 为负 ≠ 没学会（惩罚项会拖负）
3. 训练不是越多越好，可能过拟合/策略崩塌
4. 不同任务难度差异大：lift 300 万步 0%，reach 150 万步 8%

## Play 验证

命令：`play.py --task Isaac-Reach-Franka-v0 --rl_library rsl_rl --num_envs 32 --checkpoint logs/rsl_rl/franka_reach/2026-08-14_21-08-27/model_499.pt`

结果：checkpoint 加载成功，打印 Actor/Critic 网络结构（输入 32 维 → 64 → 64 → 输出 7 维动作）。

参数查找过程：play.py 报错 `--load_checkpoint` 未识别 → grep 源码发现实际参数是 `--checkpoint`（完整路径）。

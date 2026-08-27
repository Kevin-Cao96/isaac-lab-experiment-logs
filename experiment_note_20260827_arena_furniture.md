# 实验笔记：Isaac Lab Arena 自定义家具装配 Benchmark（2026-08-27）

日期：2026-08-27
环境：root@10-60-140-42 云服务器，RTX 3080 Ti 12GB，IsaacLab-Arena（Alpha）

## 一、目标

1. 熟悉 Isaac Lab Arena 的评估框架：环境、策略、任务、指标、报告。
2. 在 Arena 中注册一个自定义家具装配环境 `furniture_assembly`。
3. 定义自定义任务 `FurnitureAssemblyTask` 并跑通 baseline 评估。

## 二、完成内容

### 1. Arena 安装与示例

- 克隆 `isaac-sim/IsaacLab-Arena` 并初始化子模块（IsaacLab、Isaac-GR00T）。
- `uv sync` 安装 Isaac Sim、PyTorch、Newton 等依赖。
- 跑通 `cube_goal_pose` 零动作冒烟测试。
- 编写自定义 `ScriptedActionPolicy` 验证策略接口与 UI livestream。

### 2. 自定义家具资产

文件：`isaaclab_arena_environments/furniture/__init__.py`

注册资产：

| 资产名 | 实现 |
|---|---|
| `tabletop` | CuboidCfg（0.6 x 0.4 x 0.03） |
| `table_leg` | CylinderCfg（半径 0.025，高 0.5） |
| `ground` | GroundPlaneCfg |

关键教训：资产不能放在 `isaaclab_arena_environments` 顶层，否则会被 `__init__.py` 在 Isaac Sim 启动前自动导入并导致 Segfault；需要放进子包，并在 `build()` 里懒加载。

### 3. 自定义环境

文件：`isaaclab_arena_environments/furniture_assembly_environment.py`

- `FurnitureAssemblyEnvironmentCfg`：配置类（background、embodiment、teleop_device）。
- `FurnitureAssemblyEnvironment`：`@register_environment`，名字 `furniture_assembly`。
- `build()`：创建地面/灯光/桌板/4 条桌腿/机械臂，组装 Scene，创建 Task，返回 `IsaacLabArenaEnvironment`。

关键教训：

- Scene 按资产名字存储，多个同名资产会互相覆盖，需要唯一 `name`。
- USD prim path 也必须唯一，否则报 `A prim already exists`。

### 4. 自定义任务

文件：`isaaclab_arena/tasks/furniture_assembly_task.py`

`FurnitureAssemblyTask`：

- 初始状态：桌板固定，4 条桌腿散落在地面。
- 目标状态：4 条桌腿到达目标位置。
- 成功条件：每条腿到目标位置距离 <= 0.05m。
- 超时：10s。
- 指标：`SuccessRateMetric` + `ObjectMovedRateMetric`。

## 三、Baseline 结果

评估命令：

```bash
python isaaclab_arena/evaluation/policy_runner.py \
  --policy_type <policy> \
  --num_episodes 20 \
  furniture_assembly
```

| Policy | num_episodes | success_rate |
|---|---|---|
| zero_action | 20 | 0.0 |
| scripted_action | 20 | 0.0 |

结论：任务成功条件真实生效，零动作和脚本动作都无法完成装配。

## 四、收获

1. Arena = 评估层：环境 + 策略 + 任务 + 指标 + 报告。
2. 环境注册 = `@register_environment` + `name` + `build()`。
3. 策略接口 = `get_action(env, observation)`。
4. 任务 = 成功条件、超时、指标。
5. 资产需要懒加载、唯一 name、唯一 prim path。
6. Arena 报告路径：`outputs/<时间戳>/index.html`。

## 五、下一步

- 让机械臂抓取/移动桌腿，完成真实装配。
- 接入遥操作录示范。
- Mimic 扩数据。
- BC / OpenPI 训练并接入 Arena 评估。
- 按 RoboTwin 结构整理成独立 benchmark 仓库。

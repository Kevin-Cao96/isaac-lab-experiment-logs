# 实验笔记：桌板渲染排查 + GR00T N1.7 零样本推理 + 云实例备份（2026-09-04）

日期：2026-09-04
环境：root@117.50.187.235（RTX 3080 Ti 12GB，IsaacLab-Arena）

## 一、Furniture Assembly 桌板渲染排查

### 目标
把“带 4 个圆孔的桌板”用 Blender 建模后导入 Isaac Sim，视觉上要能看到同一面的 4 个孔。

### 排查结论

1. Blender 本地模型确实有孔，服务器 Blender 渲染同一份 USD 也能看到 4 个完整圆孔（genus = 4）。
2. Isaac Sim 中反复出现两类视觉问题：
   - 孔看起来像“正反两面各两个”；
   - 桌板整体发黑，即使材质改成浅色也没用。
3. 根因不是模型几何，而是 Isaac Sim 对 Blender 导出的布尔网格 + Principled_BSDF/顶点色处理异常，叠加默认灯光/相机角度问题。
4. 临时兜底方案：桌板改用 Isaac Lab 原生浅木色 Cuboid，在顶面放 4 个纯黑圆形标记表示孔；物理任务仍按桌腿到目标位置判定。
5. 用户自己的 USD 仍保留在 assets/ 和 GitHub 仓库中，后续可以在 Blender 导出修复后切回 usd_path。

### 保留的关键文件

- `/root/IsaacLab-Arena/isaaclab_arena_environments/furniture/__init__.py`
- `/root/IsaacLab-Arena/isaaclab_arena_environments/furniture_assembly_environment.py`
- `/root/IsaacLab-Arena/isaaclab_arena/tasks/furniture_assembly_task.py`
- `/root/IsaacLab-Arena/assets/tabletop_with_holes_local.usd`
- `/root/IsaacLab-Arena/make_tabletop_blender.py`
- `/root/IsaacLab-Arena/check_holes.py`

## 二、GR00T N1.7 安装与零样本推理

### 安装要点

1. `git clone --recurse-submodules` 官方仓库。
2. `uv sync --python 3.12` 安装依赖，体积很大（缓存一度到 40G+）。
3. GitHub Release 下载慢时用镜像：`ghfast.top` / `gh-proxy.com`。
4. `demo_data/*` 和 `scripts/deployment/dgpu/wheels/*` 是 git-lfs 文件，必须 `git lfs pull` 后才是真实数据。
5. 模型 `nvidia/Cosmos-Reason2-2B` 和 base 模型都是 HuggingFace gated repo，需要网页申请授权 + `huggingface-cli login`。

### 零样本推理结果

命令：

```bash
uv run python scripts/deployment/standalone_inference_script.py \
    --model-path nvidia/GR00T-N1.7-3B \
    --dataset-path demo_data/droid_sample \
    --embodiment-tag OXE_DROID_RELATIVE_EEF_RELATIVE_JOINT \
    --traj-ids 1 2 \
    --inference-mode pytorch \
    --execution-horizon 8
```

结果（DROID sample，零样本）：

| Trajectory | MSE | MAE |
|---|---:|---:|
| 1 | 0.0030 | 0.0355 |
| 2 | 0.0370 | 0.1174 |
| Average | 0.0200 | 0.0765 |

12GB 显卡可以跑推理（未 OOM），约 0.33s/step。README 推荐 16GB+，微调推荐 40GB+。

### 概念沉淀

- GR00T = VLA 模型 + 配套工程栈；LeRobot = 通用机器人学习框架（数据格式、训练、评估）。
- GR00T N1.7 结构：Qwen3-VL 视觉语言主干 + Flow-Matching DiT 动作头，输出 40 步动作 chunk。
- 没有 GR00T 时可用 ACT / Diffusion Policy 走同样的 LeRobot 数据闭环，12GB 可训练。

## 三、云实例状态与备份

云平台出现“暂无资源”，实例暂时无法开机。关机不等于数据一定丢失，但需要先联系平台客服确认：

1. 实例数据是否安全、是否已释放/即将释放；
2. 是否支持无 GPU 数据维护模式或系统盘快照/导出；
3. 资源恢复后如何再次开机。

## 四、从零恢复清单

如果服务器数据无法保留，按以下顺序重建：

1. 重新 clone 官方仓库：
   - `IsaacLab-Arena`
   - `Isaac-GR00T`
2. `uv sync --python 3.12`，git-lfs pull wheels 与 demo_data。
3. HuggingFace 重新申请 gated repo 授权并 `huggingface-cli login`。
4. 按本仓库 README/笔记重建自定义 environment/task/asset 文件。
5. 把 `assets/tabletop_with_holes_local.usd` 放回 IsaacLab-Arena/assets。

## 五、下一步

- 等云实例恢复后先完整打包服务器自定义代码到 GitHub。
- 家具装配先跑通“机械臂抓桌腿”脚本动作，再录 demo。
- 用 LeRobot 格式录数据 → ACT/Diffusion Policy 训练 → Arena 闭环评估。
- GR00T 作为后续 VLA 路线，微调放到大显存机器。

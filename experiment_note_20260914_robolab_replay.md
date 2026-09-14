# 实验笔记：RoboLab Isaac Lab 环境配置与录屏回放

日期：2026-09-14
环境：UCloud/CompShare Ubuntu 22.04.4 虚机，RTX 3080 Ti 12 GB，NVIDIA Driver 595.91.07，Isaac Sim 5.0.0，Isaac Lab 2.2.0，RoboLab 0.3.1

## 一、目标

在一台新的 GPU 虚机上完成以下链路：

1. 安装 RoboLab 所需系统依赖和 Python 3.11 环境
2. 解决 NVIDIA 驱动、Vulkan、Isaac Sim 启动问题
3. 使用 Git LFS 获取场景、模型和录制数据
4. 回放 RoboLab 自带的 `RubiksCubeAndBananaTask`
5. 确认任务成功判定、评分、视频和 HDF5 输出
6. 使用 RoboLab Dashboard 查看实验结果

## 二、最终环境

```text
OS:             Ubuntu 22.04.4 LTS
Kernel:         5.15.0-113-generic
GPU:            NVIDIA GeForce RTX 3080 Ti 12 GB
Driver:         595.91.07
CUDA runtime:   13.2
Python:         3.11
Isaac Sim:      5.0.0
Isaac Lab:      2.2.0
RoboLab:        0.3.1
```

驱动、内核模块和用户态库保持一致：

```text
nvidia-smi:                        595.91.07
modinfo -F version nvidia:         595.91.07
/proc/driver/nvidia/version:       595.91.07
```

Vulkan 能识别独立 NVIDIA GPU：

```text
deviceName = NVIDIA GeForce RTX 3080 Ti
driverID   = DRIVER_ID_NVIDIA_PROPRIETARY
```

## 三、安装过程

### 1. 系统依赖

```bash
apt-get update
apt-get install -y \
  git \
  git-lfs \
  tmux \
  ffmpeg \
  build-essential \
  linux-headers-$(uname -r) \
  libgl1 \
  libglu1-mesa \
  libglib2.0-0 \
  libvulkan1 \
  vulkan-tools
```

### 2. NVIDIA 驱动

```bash
apt-get install -y nvidia-driver-595-open
apt-get -f install
reboot
```

重启后验证：

```bash
nvidia-smi
modinfo -F version nvidia
cat /proc/driver/nvidia/version
vulkaninfo --summary
```

### 3. RoboLab 环境

```bash
cd ~
git clone https://github.com/NVLabs/RoboLab.git
cd RoboLab

apt-get install -y python3-pip curl
python3 -m pip install -U uv
export PATH="$HOME/.local/bin:$PATH"

uv venv --python 3.11
source .venv/bin/activate
uv sync --extra isaac50
```

依赖解析阶段耗时较长，最终解析 272 个包。长时间安装建议放入 `tmux`，避免 SSH 中断导致进程退出。

## 四、遇到的问题

| 问题 | 根因 | 解决方法 |
|---|---|---|
| `git: command not found` | 精简镜像未安装 Git | `apt-get install -y git` |
| `pip: command not found` | 未安装系统 pip | 安装 `python3-pip` 后用 `python3 -m pip` |
| `uv: command not found` | 未安装 uv 或不在 PATH | `python3 -m pip install -U uv` 后设置 `~/.local/bin` |
| `libGL.so.1` 缺失 | 精简镜像缺少 OpenCV/Isaac 图形库 | 安装 `libgl1`、`libglu1-mesa`、`libglib2.0-0` |
| `nvidia-smi` 不存在 | GPU 已直通，但未安装 NVIDIA 驱动 | 安装完整 `nvidia-driver-595-open` 后重启 |
| Vulkan `ERROR_INCOMPATIBLE_DRIVER` | 内核模块与用户态驱动版本不一致 | 统一为同一版本 `595.91.07` |
| Isaac RTX 插件段错误 | 旧 Isaac Sim 与不匹配的驱动组合 | 使用干净、版本一致的驱动环境 |
| Warp 缺少 `cuDeviceGetUuid` | CUDA 驱动接口与 Isaac 版本不兼容 | 使用匹配版本的驱动，并避免混用系统 Isaac |
| USD 报 `is not a valid usda layer` | 文件是 Git LFS 指针，不是真实 USD | `git lfs pull` 拉取真实资产 |
| HDF5 报 `file signature not found` | 回放数据仍是 Git LFS 指针 | 单独拉取 `data.hdf5` |
| `'str' object has no attribute '__name__'` | 回放时 JSON 无法恢复 `functools.partial` subtask 条件 | 使用 `--env-config current` |

### 检查是否是 Git LFS 指针

```bash
head -n 3 assets/scenes/tools_container.usda
```

错误状态会显示：

```text
version https://git-lfs.github.com/spec/v1
oid sha256:...
size ...
```

正确状态应以 USD 头开头：

```text
#usda 1.0
```

## 五、最小 LFS 下载策略

完整资产约 6 到 7 GB。若只测试 `RubiksCubeAndBananaTask`，可以先下载该任务依赖：

```bash
cd ~/RoboLab

git lfs pull --include="
assets/scenes/rubiks_cube_banana_bowl.usda,
assets/objects/ycb/banana.usd,
assets/objects/ycb/bowl.usd,
assets/objects/hot3d/rubiks_cube.usd,
assets/fixtures/franka_table.usd,
assets/fixtures/table_maple.usd,
assets/backgrounds/default/home_office.exr,
assets/robots/franka_robotiq_2f_85_flattened.usd,
examples/recorded_data/RubiksCubeAndBananaTask/data.hdf5
"
```

为了让注册阶段只加载指定任务，本地调试时将：

```python
auto_register_droid_envs()
```

改为：

```python
auto_register_droid_envs(task=args_cli.task)
```

这避免扫描全部 120 个任务和场景。

## 六、最终运行命令

```bash
cd ~/RoboLab
source .venv/bin/activate
export OMNI_KIT_ACCEPT_EULA=Y

uv run python examples/run_recorded.py \
  --task RubiksCubeAndBananaTask \
  --headless \
  --env-config current
```

`--env-config current` 使用当前仓库中的任务定义，绕过录制 JSON 中无法恢复 callable 的问题，同时仍然使用录制数据恢复初始状态并回放动作。

## 七、实验结果

任务完成输出：

```text
RubiksCubeAndBananaTask_0 complete:
run: 0
env_id: 0
success: True
step: 648
instruction: Put the cube and the banana in the bowl
score: 1.0
```

汇总结果：

```text
TOTAL (1 tasks):             1/1, 100.0%
RubiksCubeAndBananaTask:     1/1, 100.0%
Score(total):                1.000
```

终端同时打印了一条中间状态原因：

```text
reason: Condition not satisfied: object_grabbed(object=rubiks_cube) (step 1/2)
```

最终 `success=True` 且 `score=1.0` 表明任务成功完成。该 reason 字符串是终止时记录的条件信息，不改变最终成功结果。

## 八、输出文件

结果目录：

```text
output/playback_recorded_data_RubiksCubeAndBananaTask/RubiksCubeAndBananaTask/
```

主要输出：

```text
env_cfg.json
run_0.hdf5
log_0_env0.json
Put_the_cube_and_the_banana_in_the_bowl_0.mp4
Put_the_cube_and_the_banana_in_the_bowl_0_viewport.mp4
```

## 九、Dashboard

RoboLab Dashboard 是结果查看器，不是仿真器。它在实例上读取 `output/`，展示：

- 任务和场景目录
- 成功率、分数和置信区间
- 每次 episode 的结果
- 回放视频
- HDF5 状态曲线
- 多次实验对比

启动方式：

```bash
cd ~/RoboLab
source .venv/bin/activate

uv run robolab-dashboard \
  --output-dir output \
  --host 127.0.0.1 \
  --port 8090
```

本机访问可通过 SSH 端口转发：

```bash
ssh -L 8090:127.0.0.1:8090 root@<实例公网IP>
```

然后打开：

```text
http://127.0.0.1:8090
```

## 十、结论

本次实验完成了从新 GPU 实例到 RoboLab 录屏回放的完整链路：

1. 安装并验证 NVIDIA 驱动、CUDA 和 Vulkan
2. 安装 Isaac Sim 5.0、Isaac Lab 2.2 和 RoboLab
3. 使用 Git LFS 恢复 USD、机器人和 HDF5 数据
4. 成功启动 Isaac Sim 并回放 648 步轨迹
5. 正确得到 `success=True`、`score=1.0`
6. 生成可分析的视频、HDF5 和 dashboard 结果

关键经验是：Isaac Sim 对驱动一致性要求很高；Git LFS 指针文件不能被当作真实 USD 或 HDF5 使用；回放时如果录制的 callable 无法反序列化，应使用当前任务配置。

## 十一、下一步

- 完整执行 `git lfs pull`，运行所有 benchmark 任务
- 接入 Pi0/π0.5、Cosmos3、GR00T 或 VoLo policy server
- 自定义任务、场景、物体和 success condition
- 测试灯光、背景、相机位姿、初始位姿等鲁棒性变化
- 录制新轨迹并转换为 LeRobot 数据集
- 研究策略客户端到真机控制的适配与安全层

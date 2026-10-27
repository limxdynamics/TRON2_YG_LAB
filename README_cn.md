<!--
  SPDX-FileCopyrightText: 2026 LimX Dynamics Technology Co., Ltd.
  SPDX-License-Identifier: Apache-2.0
-->

<p align="center">
  <img src="doc/banner.png" alt="TRON2_YG_LAB：带机械臂和夹爪的 TRON2A 强化学习运动控制" width="100%" />
</p>

<p align="center">
  <a href="README.md">English</a> | <a href="README_cn.md">中文</a>
</p>

<p align="center">
  <a href="LICENCE"><img src="https://img.shields.io/badge/license-Apache--2.0-2f6feb" alt="License: Apache-2.0" /></a>
  <img src="https://img.shields.io/badge/python-3.10-2f6feb" alt="Python 3.10" />
  <img src="https://img.shields.io/badge/Isaac%20Sim-4.5.0-2f6feb" alt="Isaac Sim 4.5.0" />
  <img src="https://img.shields.io/badge/Isaac%20Lab-RL-2f6feb" alt="Isaac Lab" />
</p>

# TRON2_YG_LAB

基于 [Isaac Lab](https://isaac-sim.github.io/IsaacLab/) 的 LimX **TRON2A** 双足机器人强化学习训练栈，使用 PPO 训练盲走（无感知）运动策略。支持 **SF**（足板）和 **WF**（轮足）两种底座，以及带 6 自由度机械臂和双指夹爪的 **SFYG / WFYG** 形态。为 ATEC 挑战赛开发。

> **适用范围：** 平地和起伏地形（不含楼梯）上的速度跟踪，不使用感知。仓库不附带训练好的策略：在这里训练，再导出 TorchScript / ONNX 用于部署。

## 亮点

- **一套代码，四种机型。** SF、WF、SFYG、WFYG 各自注册 Flat 和 Rough 训练任务以及对应的 `-Play-v0` 任务，共 16 个任务 ID。
- **机械臂作为负载。** SFYG / WFYG 中，机械臂和夹爪由单独的 PD 组锁定在固定姿态，策略学会带着额外重量行走，机械臂不进入动作空间和观测。
- **带课程的起伏地形。** 程序生成 flat / waves / boxes / random_rough 四类子地形，10×16 网格，难度 0→1。
- **内置速度估计器。** MLP 编码器读取最近 10 步观测历史，为 actor 估计机身速度。
- **可直接部署的导出。** `play.py` 会把策略和编码器的 TorchScript、ONNX 文件导出到 checkpoint 旁边的 `exported/` 目录。

## 机型一览

| | 机型 | 说明 | 任务 ID 前缀 |
|:---:|---|---|---|
| <img src="doc/model_sf.png" alt="SF_TRON2A" width="260" /> | **SF_TRON2A** | 足板双足（带踝关节俯仰）。 | `Isaac-Limx-SF-TRON2A-...` |
| <img src="doc/model_wf.png" alt="WF_TRON2A" width="260" /> | **WF_TRON2A** | 轮足双足。 | `Isaac-Limx-WF-TRON2A-...` |
| <img src="doc/model_sfyg.png" alt="SFYG_TRON2A" width="260" /> | **SFYG_TRON2A** | 足板底座 + 6 自由度机械臂 + 双指夹爪（锁定）。 | `Isaac-Limx-SFYG-TRON2A-...` |
| <img src="doc/model_wfyg.png" alt="WFYG_TRON2A" width="260" /> | **WFYG_TRON2A** | 轮足底座 + 6 自由度机械臂 + 双指夹爪（锁定）。 | `Isaac-Limx-WFYG-TRON2A-...` |

图片由 `robot_description` 中的模型在 MuJoCo 里渲染，采用训练时的初始姿态（机械臂零位、夹爪全开）。

## 快速开始

**先看看机器人**（不需要 Isaac Sim）：

```bash
git clone --recurse-submodules https://github.com/limxdynamics/TRON2_YG_LAB.git
cd TRON2_YG_LAB
pip install mujoco
python -m mujoco.viewer --mjcf=robot_description/tron2/SFYG_TRON2A/xml/robot.xml
```

把 `SFYG_TRON2A` 换成 `SF_TRON2A`、`WF_TRON2A` 或 `WFYG_TRON2A` 即可查看其他机型。

**训练策略**（已安装 Isaac Sim 4.5.0 + Isaac Lab，见[安装](#安装)）：

```bash
python scripts/rsl_rl/train.py --task Isaac-Limx-SFYG-TRON2A-Blind-Rough-v0 --num_envs 4096 --headless
```

## 仓库结构

```
.
├── exts/bipedal_locomotion/   # Isaac Lab extension：env / asset / MDP / robot 配置
├── rsl_rl/                    # 项目内 vendored 的 rsl_rl fork（PPO、on-policy runner、MLP 编码器），train.py 会把它插到 sys.path 最前面
├── scripts/rsl_rl/            # 入口脚本 train.py / play.py / cli_args.py（脚本，不是包）
├── robot_description/         # git 子模块：TRON2 的 URDF / USD / MJCF / 网格
└── doc/                       # README 图片
```

## 安装

需要 Isaac Sim **4.5.0** + Isaac Lab（`isaaclab` / `isaaclab_tasks` / `isaaclab_rl` 可被 import）、Python 3.10 和 NVIDIA 显卡（4096 个环境并行训练建议 ≥ 12 GB 显存）。按[快速开始](#快速开始)clone 后执行：

```bash
git submodule update --init --recursive   # 仅当 clone 时没带 --recurse-submodules
pip install -e exts/bipedal_locomotion
pip install -e rsl_rl
```

训练和 play 启动时会加载 `robot_description` 子模块里的 USD 文件，缺少的话机器人无法生成。

## 训练

```bash
python scripts/rsl_rl/train.py --task <TASK> --num_envs 4096 --headless
```

`<TASK>` 为 `Isaac-Limx-{SF,WF,SFYG,WFYG}-TRON2A-Blind-{Flat,Rough}-v0`，在 [robots/__init__.py](exts/bipedal_locomotion/bipedal_locomotion/tasks/locomotion/robots/__init__.py) 中注册。**Flat** 是纯平面；**Rough** 使用 [cfg/SF_TRON2A/terrains_cfg.py](exts/bipedal_locomotion/bipedal_locomotion/tasks/locomotion/cfg/SF_TRON2A/terrains_cfg.py) / [cfg/WF_TRON2A/terrains_cfg.py](exts/bipedal_locomotion/bipedal_locomotion/tasks/locomotion/cfg/WF_TRON2A/terrains_cfg.py) 中的 `BLIND_ROUGH_TERRAINS_CFG`，YG 变体复用同一份地形。

常用选项：

- `--video --video_interval 24000 --video_length 400` 录像（自动启用 `--enable_cameras`）
- `--max_iterations N` 覆盖 PPO cfg 中的最大迭代数

日志路径：`logs/rsl_rl/<experiment_name>/<timestamp>_<run_name>/`

### Resume 续训

必须显式带 `--resume True`，`agent_cfg.resume` 默认是 `False`（[train.py:130-139](scripts/rsl_rl/train.py#L130-L139)）。

```bash
python scripts/rsl_rl/train.py --task <TASK> --num_envs 4096 --headless --resume True \
    --checkpoint_path logs/rsl_rl/<experiment_name>/<timestamp>_<run_name>/model_<iter>.pt
```

也可以把 `--checkpoint_path` 换成 `--load_run <run 目录> --checkpoint <模型文件>`，两者都支持正则（如 `--load_run ".*"`），在 `logs/rsl_rl/<experiment_name>/` 下按字典序取最新匹配。

- task ID 必须与原 run 一致，否则 obs/action 维度对不上会加载失败；只改奖励权重可以续跑。新日志写到新的 timestamp 目录。
- `--max_iterations` 是上限不是增量：训到 1500、想再训 1000，传 `--max_iterations 2500`。

## Play 与导出

用 `-Play-v0` 后缀的任务 ID。Play cfg 使用更少 env、关闭域随机化、简化地形。

```bash
python scripts/rsl_rl/play.py --task Isaac-Limx-SF-TRON2A-Blind-Rough-Play-v0 --num_envs 32 \
    --checkpoint_path logs/rsl_rl/sf_tron_2a_rough/<run>/model_<iter>.pt
```

`play.py` 还会把 TorchScript 和 ONNX（`policy.onnx`、`encoder.onnx`）导出到 checkpoint 旁边的 `exported/` 目录。

## YG 变体设计

<p align="center">
  <img src="doc/yg_design_zh.png" alt="SFYG 与 WFYG：腿部高亮表示由策略控制（10 个关节目标，或 8 个关节加 2 个轮速），机械臂和夹爪锁定、只作为负载" width="100%" />
</p>

机械臂全程锁死在固定姿态（arm1~6 = 0 rad，gripper1/2 = 0.05 m），由资产 cfg 中独立的 `arm_lock` `ImplicitActuator` 组（stiffness 800、damping 40）执行 PD 锁位。

- 机械臂关节**不在** `joint_order_name` 中，策略既不控制也观测不到它们，机械臂只是负载。
- 域随机化、reset 和 dof_limits reward 在 YG env cfg 中均排除机械臂，不会扰动锁位 PD，也不会产生多余惩罚。

机械臂排除清单的具体覆盖见 [exts/bipedal_locomotion/bipedal_locomotion/tasks/locomotion/robots/limx_solefoot_yg_tron2a_env_cfg.py](exts/bipedal_locomotion/bipedal_locomotion/tasks/locomotion/robots/limx_solefoot_yg_tron2a_env_cfg.py) 与 [limx_wheelfoot_yg_tron2a_env_cfg.py](exts/bipedal_locomotion/bipedal_locomotion/tasks/locomotion/robots/limx_wheelfoot_yg_tron2a_env_cfg.py)。

## 任务 wiring

一个任务 ID 是如何串起来的，以 `Isaac-Limx-SF-TRON2A-Blind-Flat-v0` 为例：

1. **`gym.register`**：`tasks/locomotion/robots/__init__.py` 把 (env_cfg, ppo_cfg) 注册到 task id
2. **Env cfg**：`tasks/locomotion/robots/limx_solefoot_tron2a_env_cfg.py` 继承 `tasks/locomotion/cfg/SF_TRON2A/limx_base_env_cfg.py::SF_TRON2A_EnvCfg`，挂资产 + 改 MDP
3. **MDP terms**：`tasks/locomotion/mdp/{rewards,events,observations,curriculums,commands}.py`
4. **PPO cfg**：`tasks/locomotion/agents/limx_rsl_rl_ppo_cfg.py`，类型是项目封装的 `RslRlPpoAlgorithmMlpCfg`（不是 upstream 类）
5. **资产 cfg**：`assets/config/<robot>_cfg.py`，spawn USD（来自 `robot_description/`）+ init joint pos + actuators

新增机器人变体：在以上 5 层各增量一份，不动现有 TRON2 训练栈。

## 子模块：robot_description

机器人模型来自 [tron2-robot-description](https://github.com/limxdynamics/tron2-robot-description)（TRON2 各形态的 URDF / xacro / MuJoCo XML / 网格 / USD）。本仓库将它固定在提交 `8a11c12`，该版本的模型位于 `tron2/<机型>/`。

> **不要把子模块更新到最新的 `main`。** tron2-robot-description 后续版本把模型目录改成了 `tron2a/<机型>/`，`exts/bipedal_locomotion/bipedal_locomotion/assets/config/` 里的资源配置会找不到 USD 文件。如需更新子模块，要同时修改这些路径。

## 相关仓库

| 仓库 | 说明 |
|---|---|
| [tron2_rl_lab](https://github.com/limxdynamics/tron2_rl_lab) | 基于 Isaac Lab 的 SF / WF 及全身双臂 DASF_TRON2A 训练 |
| [tron2_mujoco_sim](https://github.com/limxdynamics/tron2_mujoco_sim) | MuJoCo 仿真，用于 sim-to-sim 验证 |
| [tron2_rl_deploy_python](https://github.com/limxdynamics/tron2_rl_deploy_python) · [tron2_rl_deploy_ros](https://github.com/limxdynamics/tron2_rl_deploy_ros) | Python / ROS 策略部署 |

## License

[Apache 2.0](LICENCE)。

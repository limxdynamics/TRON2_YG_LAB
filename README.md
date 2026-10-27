<!--
  SPDX-FileCopyrightText: 2026 LimX Dynamics Technology Co., Ltd.
  SPDX-License-Identifier: Apache-2.0
-->

<p align="center">
  <img src="doc/banner.png" alt="TRON2_YG_LAB: RL locomotion for TRON2A with arm and gripper" width="100%" />
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

Reinforcement learning training stack for the LimX **TRON2A** biped, built on [Isaac Lab](https://isaac-sim.github.io/IsaacLab/) with PPO. It trains blind locomotion policies for the **SF** (sole-foot) and **WF** (wheel-foot) bases, and for the **SFYG / WFYG** variants that carry a 6-DoF arm and a two-finger gripper. Built for the ATEC challenge.

> **Scope:** blind (no perception) velocity tracking on flat and rough terrain (no stairs). In the YG variants the arm and gripper are locked and carried as payload; they are not controlled by the policy. Trained policies are not included: you train them here and export TorchScript / ONNX.

## Highlights

- **Four robots, one stack.** SF, WF, SFYG and WFYG each register a Flat and a Rough training task plus matching `-Play-v0` tasks (16 task IDs).
- **Arm as payload.** In SFYG / WFYG a dedicated PD group holds the arm and gripper in a fixed pose, so the policy learns to walk with the extra mass without the arm entering the action or observation space.
- **Rough terrain with curriculum.** Procedural flat / waves / boxes / random-rough terrain on a 10×16 grid, difficulty 0→1.
- **Velocity estimator built in.** An MLP encoder reads 10 steps of observation history and estimates base velocity for the actor.
- **Deployment-ready export.** `play.py` writes TorchScript and ONNX files for the policy and the encoder into `exported/` next to the checkpoint.

## Model lineup

| | Model | Description | Task ID prefix |
|:---:|---|---|---|
| <img src="doc/model_sf.png" alt="SF_TRON2A" width="260" /> | **SF_TRON2A** | Biped with sole feet (ankle pitch). | `Isaac-Limx-SF-TRON2A-...` |
| <img src="doc/model_wf.png" alt="WF_TRON2A" width="260" /> | **WF_TRON2A** | Biped with wheel feet. | `Isaac-Limx-WF-TRON2A-...` |
| <img src="doc/model_sfyg.png" alt="SFYG_TRON2A" width="260" /> | **SFYG_TRON2A** | Sole-foot base + 6-DoF arm + two-finger gripper (locked). | `Isaac-Limx-SFYG-TRON2A-...` |
| <img src="doc/model_wfyg.png" alt="WFYG_TRON2A" width="260" /> | **WFYG_TRON2A** | Wheel-foot base + 6-DoF arm + two-finger gripper (locked). | `Isaac-Limx-WFYG-TRON2A-...` |

Images are rendered in MuJoCo from the bundled `robot_description` models in the initial pose used for training (arm at zero, gripper fully open).

## Quick start

**Look at the robots** (no Isaac Sim needed):

```bash
git clone --recurse-submodules https://github.com/limxdynamics/TRON2_YG_LAB.git
cd TRON2_YG_LAB
pip install mujoco
python -m mujoco.viewer --mjcf=robot_description/tron2/SFYG_TRON2A/xml/robot.xml
```

Swap `SFYG_TRON2A` for `SF_TRON2A`, `WF_TRON2A` or `WFYG_TRON2A`.

**Train a policy** (Isaac Sim 4.5.0 + Isaac Lab installed, see [Installation](#installation)):

```bash
python scripts/rsl_rl/train.py --task Isaac-Limx-SFYG-TRON2A-Blind-Rough-v0 --num_envs 4096 --headless
```

## Repository structure

```
.
├── exts/bipedal_locomotion/   # Isaac Lab extension: env / asset / MDP / robot configs
├── rsl_rl/                    # Vendored rsl_rl fork (PPO + on-policy runner + MLP encoder)
├── scripts/rsl_rl/            # Entry points: train.py / play.py / cli_args.py
├── robot_description/         # Git submodule: URDF / USD / MJCF / meshes for TRON2
└── doc/                       # README images
```

## Requirements

- Isaac Sim **4.5.0** + Isaac Lab, with `isaaclab` / `isaaclab_tasks` / `isaaclab_rl` importable
- Python 3.10
- NVIDIA GPU (≥ 12 GB VRAM recommended for 4096-environment training)

## Installation

```bash
# 1. Clone with submodules
git clone --recurse-submodules https://github.com/limxdynamics/TRON2_YG_LAB.git
cd TRON2_YG_LAB
# Already cloned without submodules:
git submodule update --init --recursive

# 2. Editable install of the extension and the vendored rsl_rl
pip install -e exts/bipedal_locomotion
pip install -e rsl_rl
```

The USD files in the `robot_description` submodule are loaded when training or play starts; without them, spawning the robot fails.

## Training

Task IDs are registered in [exts/bipedal_locomotion/bipedal_locomotion/tasks/locomotion/robots/__init__.py](exts/bipedal_locomotion/bipedal_locomotion/tasks/locomotion/robots/__init__.py).

Each morphology has two terrain variants: **Flat** (pure flat plane) and **Rough** (procedurally generated rough terrain with four sub-terrain types: flat / waves / boxes / random_rough, **no stairs**).

```bash
# === Flat ===
python scripts/rsl_rl/train.py --task Isaac-Limx-SF-TRON2A-Blind-Flat-v0   --num_envs 4096 --headless
python scripts/rsl_rl/train.py --task Isaac-Limx-WF-TRON2A-Blind-Flat-v0   --num_envs 4096 --headless --max_iterations 5000
python scripts/rsl_rl/train.py --task Isaac-Limx-SFYG-TRON2A-Blind-Flat-v0 --num_envs 4096 --headless
python scripts/rsl_rl/train.py --task Isaac-Limx-WFYG-TRON2A-Blind-Flat-v0 --num_envs 4096 --headless

# === Rough (procedural terrain, no stairs) ===
python scripts/rsl_rl/train.py --task Isaac-Limx-SF-TRON2A-Blind-Rough-v0   --num_envs 4096 --headless
python scripts/rsl_rl/train.py --task Isaac-Limx-WF-TRON2A-Blind-Rough-v0   --num_envs 4096 --headless
python scripts/rsl_rl/train.py --task Isaac-Limx-SFYG-TRON2A-Blind-Rough-v0 --num_envs 4096 --headless
python scripts/rsl_rl/train.py --task Isaac-Limx-WFYG-TRON2A-Blind-Rough-v0 --num_envs 4096 --headless
```

Rough terrain configuration is defined in `BLIND_ROUGH_TERRAINS_CFG` in [cfg/SF_TRON2A/terrains_cfg.py](exts/bipedal_locomotion/bipedal_locomotion/tasks/locomotion/cfg/SF_TRON2A/terrains_cfg.py) and [cfg/WF_TRON2A/terrains_cfg.py](exts/bipedal_locomotion/bipedal_locomotion/tasks/locomotion/cfg/WF_TRON2A/terrains_cfg.py) (10×16 grid, curriculum on, difficulty 0~1). YG variants reuse the SF/WF rough terrain but exclude arm/gripper randomization and limit penalties as described in [YG Variant Design](#yg-variant-design).

Common options:

- `--checkpoint_path <path>` — resume from a specific `.pt` checkpoint (or set `resume=True` + `load_run`/`load_checkpoint` in cfg)
- `--video --video_interval 24000 --video_length 400` — enable video recording (auto-enables `--enable_cameras`)
- `--max_iterations N` — override the maximum iteration count in PPO cfg

Log path: `logs/rsl_rl/<experiment_name>/<timestamp>_<run_name>/`

## Resume Training

`agent_cfg.resume` defaults to `False`. You **must explicitly pass `--resume True`** to load a checkpoint ([scripts/rsl_rl/train.py:130-139](scripts/rsl_rl/train.py#L130-L139)). Two methods:

### Method A: Direct .pt path (recommended)

```bash
python scripts/rsl_rl/train.py \
    --task Isaac-Limx-SFYG-TRON2A-Blind-Rough-v0 \
    --num_envs 4096 --headless \
    --resume True \
    --checkpoint_path logs/rsl_rl/<experiment_name>/<timestamp>_<run_name>/model_<iter>.pt
```

### Method B: Look up by run name and checkpoint name

```bash
python scripts/rsl_rl/train.py \
    --task Isaac-Limx-SFYG-TRON2A-Blind-Rough-v0 \
    --num_envs 4096 --headless \
    --resume True \
    --load_run 2026-06-01_12-34-56_sfyg_rough \
    --checkpoint model_1500.pt
```

`--load_run` / `--checkpoint` support regex (e.g., `--load_run ".*"`, `--checkpoint "model_.*\.pt"`), matching the latest entry under `logs/rsl_rl/<experiment_name>/` in lexicographic order.

Notes:

1. **The task ID must match the original run**, otherwise obs/action dimension mismatch will cause loading failure. Changing only reward weights (which do not affect dimensions) is safe for resuming.
2. Resume creates a **new timestamped subdirectory** under `logs/rsl_rl/<experiment_name>/` for new logs, leaving the original run files untouched.
3. To train for N additional iterations: `--max_iterations` is a **cap, not an increment** — if you trained to 1500 and want 1000 more, pass `--max_iterations 2500`.

## Play and export

Use task IDs with the `-Play-v0` suffix. Play cfg uses fewer envs, disables domain randomization, and simplifies terrain.

```bash
# Flat
python scripts/rsl_rl/play.py \
    --task Isaac-Limx-SF-TRON2A-Blind-Flat-Play-v0 \
    --num_envs 32 \
    --checkpoint_path logs/rsl_rl/sf_tron_2a_flat/<run>/model_<iter>.pt

# Rough
python scripts/rsl_rl/play.py \
    --task Isaac-Limx-SF-TRON2A-Blind-Rough-Play-v0 \
    --num_envs 32 \
    --checkpoint_path logs/rsl_rl/sf_tron_2a_flat/<run>/model_<iter>.pt
```

Every training task has a corresponding `-Play-v0` variant: SF/WF/SFYG/WFYG × Flat/Rough = 8 total.

`play.py` also exports the loaded checkpoint to `exported/` next to it: TorchScript and ONNX (`policy.onnx`, `encoder.onnx`) for deployment.

## YG Variant Design

<p align="center">
  <img src="doc/yg_design_en.png" alt="SFYG and WFYG with the legs highlighted as policy-controlled (10 joint targets, or 8 joints plus 2 wheel velocities) and the arm and gripper shown as locked payload" width="100%" />
</p>

The arms remain locked in a fixed pose throughout (arm1~6 = 0 rad, gripper1/2 = 0.05 m), held by a dedicated `arm_lock` `ImplicitActuator` group (stiffness 800, damping 40) in the asset config performing PD lock.

- Arm joints are **not** in `joint_order_name` → excluded from the RL action space and observation dimensions
- Domain randomization / reset / dof_limits reward explicitly exclude arm joints in YG env cfg, preventing disturbance to the lock PD or spurious penalties
- During training/inference, the arms serve as **payload only** and are invisible to the policy

For the full arm exclusion checklist, see [exts/bipedal_locomotion/bipedal_locomotion/tasks/locomotion/robots/limx_solefoot_yg_tron2a_env_cfg.py](exts/bipedal_locomotion/bipedal_locomotion/tasks/locomotion/robots/limx_solefoot_yg_tron2a_env_cfg.py) and [limx_wheelfoot_yg_tron2a_env_cfg.py](exts/bipedal_locomotion/bipedal_locomotion/tasks/locomotion/robots/limx_wheelfoot_yg_tron2a_env_cfg.py).

## Architecture

Three top-level packages:

1. **`exts/bipedal_locomotion/`** — Isaac Lab extension. All env/asset/MDP/robot configs live here.
2. **`rsl_rl/`** — Vendored fork. `scripts/rsl_rl/train.py` prepends this path to `sys.path`, overriding the system-installed version. Import uses `from rsl_rl.runner import OnPolicyRunner` (singular `runner`, not upstream's `runners`).
3. **`scripts/rsl_rl/`** — Entry-point scripts. **Not a package**; operates via `sys.path` manipulation. CLI parsing order is fixed: launcher args must be registered before `AppLauncher(args_cli)`.

### Task Wiring

Using `Isaac-Limx-SF-TRON2A-Blind-Flat-v0` as an example:

1. **`gym.register`**: `tasks/locomotion/robots/__init__.py` binds (env_cfg, ppo_cfg) to the task ID
2. **Env cfg**: `tasks/locomotion/robots/limx_solefoot_tron2a_env_cfg.py` inherits from `tasks/locomotion/cfg/SF_TRON2A/limx_base_env_cfg.py::SF_TRON2A_EnvCfg`, attaches assets + modifies MDP
3. **MDP terms**: `tasks/locomotion/mdp/{rewards,events,observations,curriculums,commands}.py`
4. **PPO cfg**: `tasks/locomotion/agents/limx_rsl_rl_ppo_cfg.py`, using the project's custom `RslRlPpoAlgorithmMlpCfg` type (not the upstream class)
5. **Asset cfg**: `assets/config/<robot>_cfg.py`, spawns USD (from `robot_description/`) + init joint pos + actuators

To add a new robot variant: increment one copy at each of the 5 layers above without modifying the existing TRON2 training stack.

## Submodule: robot_description

The robot models come from [tron2-robot-description](https://github.com/limxdynamics/tron2-robot-description) (URDF / xacro / MuJoCo XML / meshes / USD for the TRON2 variants). This repository pins it at commit `8a11c12`, where the models live under `tron2/<MODEL>/`.

> **Do not update the submodule to the latest `main`.** Later versions of tron2-robot-description moved the models to `tron2a/<MODEL>/`, so the asset configs in `exts/bipedal_locomotion/bipedal_locomotion/assets/config/` would no longer find the USD files. Update the submodule only together with those paths.

## Related repositories

| Repository | Description |
|---|---|
| [tron2-robot-description](https://github.com/limxdynamics/tron2-robot-description) | TRON2 robot models (URDF / MJCF / USD) |
| [tron2_rl_lab](https://github.com/limxdynamics/tron2_rl_lab) | Isaac Lab RL for SF / WF and the full-body dual-arm DASF_TRON2A |
| [tron2_mujoco_sim](https://github.com/limxdynamics/tron2_mujoco_sim) | MuJoCo simulator for sim-to-sim checks |
| [tron2_rl_deploy_python](https://github.com/limxdynamics/tron2_rl_deploy_python) · [tron2_rl_deploy_ros](https://github.com/limxdynamics/tron2_rl_deploy_ros) | Policy deployment in Python / ROS |

## License

[Apache 2.0](LICENCE). If this project helps your work, a ⭐ helps others find it.

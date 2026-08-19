# PROJECT_CONTEXT.md

Confirmed current understanding of the hexapod robot repository.

- **Version:** 1.0
- **Date:** 2026-08-19
- **Basis:** Direct repository inspection (source, configs, URDFs, launch files, compiled `.pyc` artifacts, colcon build logs, git history, and the PyTorch checkpoint binary).

---

## 0. Scope and Verification Caveat

**Read this before trusting anything below.**

This document records what was verified by inspecting the repository, not what the READMEs claim. Where documentation and implementation disagree, the disagreement is stated explicitly.

**Critical limitation on verification depth:**

The development machine used for this analysis has **no ROS2, no Gazebo, no Isaac Sim, and no PyTorch installed** (`/opt/ros` absent; `rclpy`, `torch`, `matplotlib` not importable). Therefore:

- All claims about **ROS2 node behaviour, topic wiring, controller behaviour, and simulation** are **static analysis only**. Nothing ROS-level has been executed.
- Claims about `SpiderBotLib.py` are **verified by direct execution** (see §5.5) and are the strongest evidence in this document.
- Claims about the lost driver generations are **verified from compiled bytecode**, which is primary evidence.

Nobody — neither previous students nor this analysis — has demonstrated a running system on the current machine.

---

## 1. Status Vocabulary

`CLAUDE.md` §3 defines five labels. Analysis showed that "IMPLEMENTED" conflates three materially different states, so three labels are added. **Use the extended set in this document.**

| Label | Meaning |
|---|---|
| `VERIFIED BY EXECUTION` | Code was actually run in this analysis and produced correct output. |
| `IMPLEMENTED` | Code exists and appears complete; not executed. |
| `IMPLEMENTED BUT NOT RUNNABLE` | Code is complete in intent but cannot execute (missing dependency, missing method, bad path). |
| `RUNS BUT INERT` | Code executes without crashing but silently does nothing useful. |
| `PARTIALLY IMPLEMENTED` | Some paths work, others are incomplete or broken. |
| `DOCUMENTED BUT NOT VERIFIED IN CODE` | Claimed by README/comments; not confirmed in code. |
| `PROPOSED` | An intended direction, not implemented. |
| `NOT FOUND IN THIS REPOSITORY` | Exhaustively searched here; may exist elsewhere. |
| `CONFIRMED NOT TO EXIST` | Searched by content, not filename; confirmed absent project-wide. |
| `UNRESOLVED` | Genuinely unknown; requires external information or a decision. |

**The distinction between `NOT FOUND IN THIS REPOSITORY` and `CONFIRMED NOT TO EXIST` is load-bearing** and must be preserved in future edits.

---

## 2. Intended Architecture (New System)

### 2.1 Confirmed

- **The new robot is controlled by an external computer.** The robot acts primarily as a hardware execution platform. `CONFIRMED` (stated by project owner)
- **The onboard-Jetson architecture is not the intended architecture** for the new system. `CONFIRMED` (stated by project owner)
- **Development sequence:** understand the inherited system → develop and validate methods in Isaac Sim / Isaac Lab → progress toward sim-to-real on the physical robot. `CONFIRMED` (stated by project owner)
- **Goal:** research contribution sufficient for a conference submission, with a minimum-necessary-change engineering principle per subsystem. `CONFIRMED` (stated by project owner)

### 2.2 Unresolved

- **The physical communication interface between the external computer and the robot has not been finalized.** `UNRESOLVED`

  No transport is confirmed. Do **not** assume USB, Wi-Fi, Ethernet, serial, or any other specific transport when writing code, designing experiments, or making architectural claims. This item must be resolved before the real-robot deployment path can be specified, but it **does not block** Isaac Sim / Isaac Lab development, which is the current priority.

### 2.3 Proposed (not confirmed)

- **A tethered connection for initial development.** `PROPOSED`

  Rationale: a tethered link could simplify early control-stack bring-up and sim-to-real validation by removing variable latency as a confounding variable while the policy transfer itself is still unproven. This is an engineering suggestion under consideration, **not a decided system fact**, and must not be recorded elsewhere as confirmed.

### 2.4 Priority Ordering for Component Evaluation

When deciding what to preserve, repair, minimally modify, or rewrite, prioritise components supporting: external-computer control; ROS2 communication; Isaac Sim / Isaac Lab development; sim-to-real transfer; physical robot joint/actuator interfaces.

---

## 3. Repository Structure

```
spider_robot_ws/
├── CLAUDE.md                       # project instructions (authoritative)
├── README.md                       # legacy usage notes (see §9 for mismatches)
├── 操作指南手冊.txt                 # near-duplicate of README.md
├── docs/
│   ├── PROJECT_CONTEXT.md          # this file
│   ├── RESEARCH_CONTEXT.md         # empty
│   └── EXPERIMENT_LOG.md           # empty
├── src/spider_bot/                 # the single ROS2 package (ament_python)
│   ├── spider_bot/                 # Python modules (see §5)
│   ├── urdf/                       # 3 robot description variants
│   ├── config/                     # 2 ros2_control YAMLs
│   ├── launch/                     # 3 launch files
│   ├── worlds/                     # 3 Gazebo Classic worlds
│   ├── terrain_tools/              # runtime terrain spawner
│   ├── models/model_1499.pt        # trained PPO checkpoint (see §5.7)
│   ├── autostart_scripts/          # LEGACY Jetson systemd services
│   └── dynamixel_sdk/              # vendored ROBOTIS SDK (Apache-2.0)
├── build/  install/  log/          # STALE ARTIFACTS — see §3.1
├── 五軸測量數據與圖/                # experiment CSVs + plots
├── 六條腿受力分析/                  # experiment CSVs + plots
├── 腿1、4力矩分析/                  # experiment CSVs + plots
└── 控制圖/                          # FSM.png, 控制圖.jpg (images, not parsed)
```

### 3.1 `build/`, `install/`, `log/` are stale and actively misleading

`CONFIRMED`

- They are **committed to git**: 341 of 599 tracked files live under these directories.
- `install/spider_bot/lib/python3.10/site-packages/spider-bot.egg-link` → `/home/chia/spider_ws/build/spider_bot`, a path that **does not exist on this machine**. Sourcing `install/setup.bash` would poison the environment with dead paths.
- `build/spider_bot/config/` contains four symlinks into `/home/chia/spider_ws/`, two of which (`bridge_params.yaml`, `bridge.yaml`) point to files that **no longer exist in source**.
- There are **two separate build trees** (`./build`, `./src/spider_bot/build`) produced from **different `setup.py` versions**: the top-level `install/` has a `generate_rough_ground` console script absent from the current `setup.py`, while the nested one has `test_node` and lacks `generate_rough_ground`.

**These directories should be deleted and gitignored.** They caused an incorrect conclusion during analysis before being audited, and they are the direct cause of the terminal build failure (§8.1).

---

## 4. Robot Description (URDF)

`IMPLEMENTED` as geometry; **inadequate for sim-to-real**.

Three hand-written URDF variants exist. **No xacro macros** are used (the launch files run `xacro.process_file()` on plain URDF, which is a no-op passthrough).

| File | Root link name | Joint naming | `ros2_control` command interface |
|---|---|---|---|
| `urdf/spider.urdf` | `spider_bot` | `joint1` … `joint18` | **18 × effort** |
| `urdf/spider_bot.urdf` | `hexapod_track_robot` | `joint_{rf,lf,rm,lm,rb,lb}_{coxa,femur,foot}` | **18 × position** |
| `urdf/spider_tank.urdf` | `spider_tank` | leg joints `fixed`; 4 × `continuous` rollers | 4 × velocity |

`spider.urdf` additionally carries a depth-camera link and a `libgazebo_ros_p3d.so` ground-truth odometry plugin (`:340-348`). `spider_tank.urdf` is referenced by **no launch file**.

### 4.1 Critical limitation: the URDFs are placeholders, not engineering models

`CONFIRMED`

- **Zero mesh references** in any URDF; geometry is boxes/cylinders/spheres only. No `.stl`, `.dae`, or `.obj` ships anywhere in the package.
- `spider.urdf`: only **two distinct mass values** across 20 links (`0.1`, `1.0`). `spider_bot.urdf`: four (`0.1`, `0.15`, `0.3`, `2.5`).
- **Only 3 distinct inertia tensors across 19 `<inertia>` tags** in each file (`spider.urdf`: 20 links; `spider_bot.urdf`: 19 links) — i.e. copy-pasted, not derived from CAD.
- Joint limits are invented: `effort="50.0"` (`spider.urdf`) and `effort="10"` (`spider_bot.urdf`). The Dynamixel XM430-W350 stall torque is approximately 4.1 N·m.
- **No IMU sensor is modelled in any URDF**, so `/imu/data` has no simulation source.

**Implication:** these descriptions are adequate for Gazebo visual demos and unusable for credible sim-to-real. Producing a physically accurate robot description is a **gating prerequisite** for the Isaac Lab plan. See §11.

---

## 5. Subsystems: Current Implementation Status

### 5.1 ROS2 Package

`IMPLEMENTED`. Single `ament_python` package `spider_bot`. Console scripts (`setup.py`): `spider_bot_control`, `test_node`, `run`, `standup`, `tank`, `imu_reader`, `plot_thesis`.

Declared dependencies (`package.xml`): `rclpy`, `gazebo_ros`, `gazebo_ros2_control`, `xacro`, `robot_state_publisher`, `controller_manager`, `joint_state_broadcaster`, `effort_controllers`, `std_msgs`, `sensor_msgs`, `geometry_msgs`.

**Undeclared dependencies** `CONFIRMED` — `package.xml` omits every one of: `numpy`, `torch`, `joblib`, `matplotlib`, `pandas`, `cv2`, `pyrealsense2`, `pyfakewebcam`, `bleak`, `dynamixel_sdk`, and — critically — the ROS packages **`position_controllers`** and **`velocity_controllers`**, both required by the shipped YAMLs. A clean build on a fresh machine will not pull these in.

All 19 first-party source files under `spider_bot/`, `launch/`, and `terrain_tools/` parse cleanly (AST-verified); there are no syntax or indentation errors despite mixed tab/space usage. The vendored `dynamixel_sdk/` and the ament `test/` files were not checked.

### 5.2 Simulation: Two Coherent, Mutually Exclusive Gazebo Stacks

`IMPLEMENTED`; **Gazebo Classic, which is end-of-life.**

This is the single most misunderstood aspect of the repository. There are **two complete stacks**, each internally consistent, that **share the topic name `/spider_leg_controller/commands` while interpreting its contents in different physical units**.

| | **Stack A (effort / torque)** | **Stack B (position)** |
|---|---|---|
| Launch | `gazebo_launch.py` | `gazebo_new_launch.py` |
| URDF | `spider.urdf` (18 × effort) | `spider_bot.urdf` (18 × position) |
| Config | `spider_bot_controllers.yaml` | `spider_bot_new_controllers.yaml` |
| Controller | `effort_controllers/JointGroupEffortController` | `position_controllers/JointGroupPositionController` (`interface_name: position`, per-joint PID gains) |
| Joint names | `joint1` … `joint18` | `joint_rf_coxa`, … |
| World | `safe_terrain.world` + spawned rough ground | `empty_high_friction.world` |
| Command units | **N·m** | **radians** |
| Consumers | `run.py`, `standup.py`, `spider_bot_control.impedance_control_loop`, `tank.py` | `test_node.py` |

Stack membership is provable, not inferred: `run.py:37` and `standup.py:33` parse joint IDs via `int(''.join(filter(str.isdigit, name)))`, which only works with `joint1..joint18`; `spider_bot_control.py:390-396` publishes `JointState` with `name = [f'joint{i+1}' ...]`. Conversely `test_node.py:105-112` lists joints in exactly the order of `spider_bot_new_controllers.yaml`.

**Each stack is self-consistent.** The hazard is mixing a launch file from one with a script from the other — the units are silently misinterpreted with no error.

Controllers are loaded differently in each: `gazebo_launch.py` uses `ExecuteProcess` + `ros2 control load_controller` chained on `OnProcessExit`; `gazebo_new_launch.py` uses `controller_manager` `spawner` nodes.

**Known defect in Stack A** `CONFIRMED`: `gazebo_launch.py` activates `dc_motor_controller`, whose joints (`l1_tibia_back_joint`, etc.) appear in `spider_tank.urdf` but **zero times in `spider.urdf`**. That controller load will fail.

### 5.3 Terrain

`IMPLEMENTED`; **inadequate for climbing research.** `CONFIRMED`

- `terrain_tools/generate_rough_ground.py:26` — `z_h = random.uniform(0.02, 0.05)`, spawning a 15×15 grid of 0.20 m boxes via the `/spawn_entity` service, friction `mu=100`.
- Measured maximum box height: `rough_terrain.world` ≈ **6.9 cm**; `safe_terrain.world` ≈ **5.0 cm**; `empty_high_friction.world` contains only a ground plane.

There is **no terrain asset supporting high-obstacle climbing**. Any such capability must be built.

### 5.4 Actuator / Motor Interface — `SpiderBotDriver.py`

`IMPLEMENTED` for the real-hardware path. **This is a key reusable asset for the new architecture.**

Wire configuration (`:59-62`): `PROTOCOL_VERSION = 2.0`, `BAUDRATE = 1000000`, `DEVICENAME = '/dev/u2d2'`. The U2D2 is a USB↔RS485 adapter — **nothing in this driver's hardware path assumes a Jetson.** It is usable from any Linux machine, including an external control computer.

Methods required by an external-computer / RL deployment path, all **present and complete** (bodies inspected, not stubs):

| Capability | Method | Line |
|---|---|---|
| Bulk write 18 goal positions | `SyncWriteAllPositions` | `:260-283` |
| Bulk write 18 goal currents | `SyncWriteAllCurrents` | `:235-258` |
| Bulk read pos + vel + current, all 18, one transaction | `UpdateAllStatesSync` | `:288-333` |
| Operating-mode switching (position ↔ current-based position) | `SetOperatingMode`, `SetOperatingMode_byLeg` | `:384-404` |
| Per-joint PID gains | `SetPID`, `SetPID_byLeg` | `:420-433` |
| Torque enable/disable | `TorqueOn/Off`, `LegTorqueOn/Off` | `:455-482` |

`UpdateAllStatesSync` performs a single `GroupSyncRead` yielding current/velocity/position for all 18 servos, converted to mA / deg·s⁻¹ / deg — precisely the proprioception primitive an RL deployment needs.

**Legacy methods absent from this driver** — see §7.2. They are **not required by the external-computer path**.

**`sim_mode` branch: `RETIRE`.** `SpiderBotDriver.__init__(sim_mode=True)` is the default (`:82`). The sim branch publishes to `/joint_commands` (`:188-191`), a topic **nothing in this repository subscribes to**, and `UpdateAllStatesSync` hardcodes `joint{servo_id}` naming (`:326`), coupling it to Stack A. It also applies `NM_TO_MA = 750.0` (`:317`), an **invented conversion constant** with no derivation; it affects the sim branch only. Isaac Lab drives articulations directly and will not use this branch.

### 5.5 Gait / Locomotion — `SpiderBotLib.py`

`VERIFIED BY EXECUTION`. **The highest-value inherited asset.**

Pure `numpy`; imports only `numpy`, `matplotlib` (plotting only), and `time`. **No ROS, no Jetson, no hardware coupling.** It runs standalone.

Executed during analysis (with `matplotlib` stubbed):

| Check | Result |
|---|---|
| `SpiderBotLib()` instantiation | OK |
| `generate_crabWalkingLUT()` | OK, 0.09 s, 36 LUT rows, `DATA_POINT_ALL = 40` |
| `generate_waveGaitLUT()` | OK |
| `get_jacobian()` | OK, returns 3×3 |
| `compute_impedance_control()` | OK, returns 3-vector |
| **`fwd` / `inv` round-trip** | **Position error = 0.0 (exact)** |

**Note on the IK:** `inv(fwd(θ))` returns a *different* joint triple than the input (e.g. `(0, −0.5, 1.0)` → `(0, 0.796, −1.0)`), but `fwd` of that result reproduces the original Cartesian position exactly. This is the standard **elbow-up / elbow-down two-solution ambiguity**, not a defect.

Implemented content:
- Analytic closed-form forward and inverse kinematics (`fwd`, `inv`).
- **Quadratic Bézier** swing-phase foot trajectories: `Y_curve = (1−t)²P1 + 2(1−t)tP2 + t²P3` at `:64-71`, `:542-552`, `:1080-1116`, `:1249-1285`.
- **Tripod gait** (`XYZ_TriGait_gen`) and **Wave gait** (`XYZ_WaveGait_gen`, `XYZ_WaveGait_realTime`), with precomputed lookup tables (`generate_crabWalkingLUT`, `generate_waveGaitLUT`).
- Body-frame 6-DOF transforms (`bodyTranslate_to_newLegXYZ`, `bodyRotate_to_newLegXYZ`) for orientation control.
- Jacobian-transpose impedance control (`get_jacobian`, `force_to_torque`, `compute_impedance_control`), `τ = Jᵀ(Ke − Dv)`.

**No CPG (Central Pattern Generator) exists.** `CONFIRMED NOT TO EXIST` — a content search for `cpg`, `central pattern`, `hopf`, `matsuoka`, `oscillator` across the repository returns zero matches. Gait timing is LUT-index stepping and explicit phase arithmetic, not coupled oscillators. Any documentation or paper text describing this system as CPG-based would be incorrect.

**Climbing:** `NOT FOUND IN THIS REPOSITORY`. There is no obstacle-climbing controller, no foothold search, and no terrain sensing. "Rough-walking" is Wave gait plus a switch to current-based position control for passive compliance.

### 5.6 Main Control Node — `spider_bot_control.py`

`IMPLEMENTED BUT NOT RUNNABLE`. **Legacy component — see §7.**

1904 lines, single class `SpiderBotControl(Node)`. It is the Jetson-era application node fusing RC (SBUS) input, web-console input, and the LUT gait state machine.

Interfaces (`:267-310`) — subscribes: `/jmoab/sbus_rc_ch`, `/jmoab/cart_mode`, `/console/joy`, `/spider/control_mode_cmd`, `/spider/height_cmd`, `/spider/pitch_cmd`, `/spider/reset_height`, `/spider/reset_pitch`, `/spider/leg2_xyz_cmd`, `/spider/leg3_xyz_cmd`, `/spider/inspection_start`, `/robot/target_xyz`. Publishes: `joint_states`, `/spider_leg_controller/commands`, `/spider/leg2_xyz_fb`, `/spider/leg3_xyz_fb`, `/spider/inspection_status`, `/spider/train_data`.

Two concurrent timers: `timer_callback` at 100 Hz (`:291`) and `impedance_control_loop` at 50 Hz (`:299`).

Control modes (`self.control_mode`): `0` flat-walking (crab-walk LUT), `1` orientation control, `2` rough-walking (wave gait + compliant mode), `3` inspection, `4` impedance (undocumented).

**Four confirmed defects:**

| # | Defect | Evidence | Effect |
|---|---|---|---|
| 1 | Calls **four driver methods that do not exist** in `SpiderBotDriver.py`: `RunServoInTime` (`:710`, **14 call sites**), `RunServoInTimeByLeg` (`:473` et al., **26 call sites**), `ReadPosition` (`:1177`), `ReadCurrentPosition` (`:1287`) | AST method-set comparison; no `__getattr__` on the driver | **`AttributeError` during `__init__`** (`:94 self.standByPosition()` → `:710`). The node cannot construct in either sim or real mode. |
| 2 | `self.walk_timer` referenced at `:562` and `:585`, **never assigned** (only `self.timer` and `self.impedance_timer` exist) | grep of all `self.*timer` | `AttributeError` on any transition into or out of `control_mode == 4` |
| 3 | `svm_model.predict(...)` called at `:1824`; the `joblib.load(...)` that defines it is **commented out** at `:39` | direct read | `NameError` when inspection reaches `insp_step == 2` |
| 4 | `impedance_control_loop` calls `compute_impedance_control` with **5 positional arguments** at `:356-362`; the signature at `SpiderBotLib.py:332` requires **6** (`leg_index`) | signature vs call site | See below — `RUNS BUT INERT` |

**Defect 4 in detail** (`RUNS BUT INERT`): every leg, every 20 ms cycle, raises `TypeError`, which is caught by a bare `except Exception` at `:367` whose warning log is commented out, appending `[0.0, 0.0, 0.0]`. The loop therefore **publishes 18 zeros at 50 Hz, silently, forever**. `run.py:78` and `standup.py:73` pass `leg_index=i` correctly, so this defect is specific to the control node.

Additional dead wiring in the same loop: it uses a **static** target `self.h.XYZ_home` rather than the gait LUT; `self.gait_index` is incremented but only used for modulo-50 logging; `self.current_target_xyz` (written by the `/robot/target_xyz` callback at `:315`) is **never read**; and `self.impedance_enabled` / `self.is_impedance_mode` are written but **never read**, so the timer is ungated.

`SIM_MODE` is a hardcoded module-level constant (`:15`), not a ROS parameter or launch argument.

### 5.7 Reinforcement Learning

**No RL training infrastructure exists.** `CONFIRMED NOT TO EXIST` (project-wide; the project owner has confirmed no recoverable training code exists elsewhere).

Absent: environment definition, observation/action space specification, reward function, termination and reset conditions, domain randomisation, curriculum, training script, and any Isaac Lab / Isaac Sim / Omniverse code or USD asset.

**What does exist — `models/model_1499.pt`:** a **genuine, fully-trained PPO actor-critic checkpoint**, verified by unpickling the archive directly:

```
actor_state_dict   : mlp.0 (512,253) → mlp.2 (256,512) → mlp.4 (128,256) → mlp.6 (18,128)
                     obs_normalizer._mean/_var/_std (1,253); distribution.std_param (18,)
critic_state_dict  : same trunk → mlp.6 (1,128)
optimizer_state_dict : Adam, lr 1.139e-4 (decayed)
iter               : 1499
```

The `obs_normalizer` + `distribution.std_param` + adaptive-LR Adam signature is characteristic of **`rsl_rl`**, the library used by Isaac Lab and legged_gym. Observation dimension 253, action dimension 18.

**Inference (not confirmed):** a standard legged_gym rough-terrain observation composes as `3+3+3+3+18+18+18 = 66`, and `253 − 66 = 187 = 17 × 11`, exactly the default height-scanner grid. If correct, this policy expects a **187-point terrain heightmap** that this robot has no pipeline to produce. This should be treated as a hypothesis to test, not a fact — but it is material to any plan that hopes to reuse this checkpoint.

**`test_node.py` — `IMPLEMENTED BUT NOT RUNNABLE`, and methodologically invalid even if fixed:**

1. `:17` hardcodes `~/spider_ws/src/spider_bot/models/model_1499.pt`. That path **does not exist** in this workspace, so the load fails and `:54 sys.exit(1)` fires **before the control loop is ever reached**.
2. Even if the path were fixed: the 253-dim observation at `:170-192` is **not derived from robot state**. It is two interleaved sine waves (`phase_A`/`phase_B`, 1.5 Hz) filling all 253 dimensions, plus `+= 0.05*roll`, `+= 0.05*pitch`, plus `np.random.uniform(-0.1, 0.1, 253)` noise.
3. `:49` uses `load_state_dict(..., strict=False)`, which **silently discards `obs_normalizer.*`**. `rsl_rl` applies that normalizer before the MLP at inference; omitting it corrupts inference even with correct observations. (The MLP trunk weights themselves *do* load correctly — checkpoint keys `mlp.0/2/4/6.*` match the local class exactly.)

**This script must not be cited as evidence of working RL, sim-to-real transfer, or Isaac Lab integration.**

### 5.8 Sensors

| Sensor | Status | Detail |
|---|---|---|
| IMU | `IMPLEMENTED` | `imu_reader.py` (`WitImuUdpNode`) binds **UDP port 9001**, parses **WitMotion** binary packets (`0x55 0x61` header), publishes `sensor_msgs/Imu` on `/imu/data` at 100 Hz. Orientation arrives pre-fused from the device; no Madgwick/Kalman filter in this repo. |
| IMU bridge | `IMPLEMENTED` | `windows_to_wsl_imu.py` — a **Windows-side** BLE client (`bleak`) for a **WT901BLE67**, forwarding raw notifications over UDP. Implies a Windows→WSL2 development path was in use at some point. |
| Depth camera | `IMPLEMENTED`, but not a ROS interface | `realsense_handler.py` pipes Intel RealSense L515 colour+depth through OpenCV into a virtual webcam (`pyfakewebcam` → `/dev/video30`) for the WebRTC console. **Publishes no ROS topic**; no point cloud, no heightmap, no terrain perception. |
| Joint feedback | `IMPLEMENTED` | Position/velocity/current via `UpdateAllStatesSync`. Used for inspection contact-sensing and logging; **not** used to close the locomotion loop (gait is open-loop). |
| Contact / force / lidar | `NOT FOUND IN THIS REPOSITORY` | No dedicated foot-contact sensors. |
| IMU in simulation | `NOT FOUND IN THIS REPOSITORY` | No IMU link or sensor in any URDF, so `/imu/data` has no Gazebo source. |

**README mismatch:** `src/spider_bot/README.md:21` lists a **BNO055** IMU. The implemented code path is unambiguously a **WitMotion WT901BLE67**. See §9.

### 5.9 ML Inspection Pipeline (Soft/Hard Object Classification)

`PARTIALLY IMPLEMENTED` — data collection and offline training work; the trained model is **not deployed**.

Flow: `spider_bot_control.py` inspection mode (`control_mode == 3`) drives legs into contact via current-based position control → publishes 5 features `[diff_x, diff_y, diff_z, cur_2, cur_3]` on `/spider/train_data` → `train_data_collector.py` logs to CSV → `SVM_training.ipynb` trains an SVM on 641 samples (7 hard-object CSVs, 4 soft-object CSVs; 512/129 split; RBF and polynomial kernels reported at 100% test accuracy, linear ≈98.4%) → `svm_models/20240123_641dataset.joblib`.

**The deployed model is never loaded** — `spider_bot_control.py:39` is commented out (defect 3 in §5.6). No other file references `joblib` or `svm_models`.

`docs/hexapod_inspection_mode.pdf` exists (1.2 MB); contents `UNRESOLVED` (not parsed).

---

## 6. Control and Data Flow (Current Repository State)

```
LEGACY PATH (Jetson-era, not runnable — see §5.6, §7)
  RC transmitter ──SBUS──> jmoab_ros2 ──/jmoab/*──┐
  Web console ──WebRTC/zenoh──> /console/joy, /spider/*──┤
                                                         ▼
                                          spider_bot_control.py
                                            [AttributeError at __init__]
                                              ├─ timer_callback 100 Hz
                                              │    └─> SpiderBotLib LUT gait + IK
                                              │         └─> driver.RunServoInTime*  [METHOD ABSENT]
                                              │              └─> Dynamixel XM430 ×18
                                              └─ impedance_control_loop 50 Hz
                                                   └─> publishes 18 zeros  [RUNS BUT INERT]

GAZEBO STACK A (effort / N·m)                    GAZEBO STACK B (position / rad)
  run.py, standup.py ──┐                           test_node.py ──┐
  tank.py ─────────────┤                           [exits at startup:
                       ▼                            model path missing]
      /spider_leg_controller/commands                              ▼
                       ▼                            /spider_leg_controller/commands
      JointGroupEffortController                                   ▼
      (spider.urdf, joint1..18)                     JointGroupPositionController
                       ▼                            (spider_bot.urdf, named joints)
                  Gazebo Classic ──/joint_states──> back to scripts

INSPECTION / ML PATH (data pipeline works; deployment disabled)
  legs contact object -> /spider/train_data -> train_data_collector.py -> CSV
      -> SVM_training.ipynb -> svm_models/*.joblib -> [NOT LOADED: line 39 commented]

ORPHANED
  SpiderBotDriver sim_mode ──> /joint_commands  [no subscriber anywhere]
```

**Note:** no single end-to-end path in this repository has been demonstrated to work on the current machine.

---

## 7. Historical Lineage (Evidence-Based)

**This section documents the previous system. It is historical context, clearly separated from the intended architecture in §2. It is recorded because it explains why the current code looks as it does — not because the new project depends on it.**

### 7.1 The old architecture was Jetson-centric

The inherited system ran **onboard a Jetson Nano** with:
- `autostart_scripts/*.service` — six systemd units, all `User=jetson`, running from `/home/jetson/dev_ws`: `ros2_atcart_basic` (the external `jmoab_ros2` RC/SBUS bridge), `ros2_spider_bot_control`, `ros2_realsense_handler`, plus a WebRTC stack (`webrtc_signaling_server`, `webrtc_zenoh_bridge`, `webrtc_browser`).
- A **JMOAB carrier board** for Futaba RC receiver input over SBUS.
- A separate `spider_bot_console` web package (**not present in this repository**).

**Jetson-specific components** (legacy under the new architecture): the autostart services, the `jmoab_ros2` dependency and RC/SBUS control path, the onboard WebRTC video console, and `spider_bot_control.py` itself.

**Components that were never Jetson-coupled:** `SpiderBotDriver.py` (U2D2 USB, works on any Linux host), `SpiderBotLib.py` (pure numpy), the URDFs, worlds, and controller configs.

### 7.2 Three driver generations, proven from compiled bytecode

`CONFIRMED`

Git history is **not** a reliable record of this project (commits are `"Add files via upload"` / `"Full clean rebuild"`; build artifacts are committed). However, `__pycache__` retains compiled evidence of earlier source states:

| Method | py3.8 `.pyc` (oldest) | py3.10 `.pyc` = current source | required by `spider_bot_control.py` |
|---|---|---|---|
| `RunServoInTime` | **present** | absent | yes |
| `ReadPosition` | **present** | absent | yes |
| `RunServo` | **present** | absent | no |
| `RunServoInTimeByLeg` | absent | absent | yes |
| `ReadCurrentPosition` | absent | absent | yes |
| `*_byLeg` family, `UpdateAllStatesSync`, `SyncWriteAll*`, `sim_mode` | absent | **present** | yes |

`SpiderBotDriver.cpython-38.pyc` contains the qualnames `SpiderBotDriver.RunServoInTime` and `SpiderBotDriver.ReadPosition`, plus the runtime error string `[ID:{:d}] groupSyncWritePositionInTime addparam failed` — referencing exactly the `groupSyncWritePositionInTime` object that current source constructs at `SpiderBotDriver.py:150` and **never uses**.

**Conclusion:** at least three driver generations existed. `spider_bot_control.py` was written against an **intermediate** generation that is present in no artifact in this repository. The methods were not "never written"; they were lost in a rewrite.

Similar evidence dates the library layers: `SpiderBotLib.cpython-38.pyc` lacks `get_jacobian`, `force_to_torque`, `compute_impedance_control`, and all wave-gait functions, and uses the older names `XYZ_gen` / `XYZ_gen_custom` (preserved in `SpiderBotLib.py.bak`, 1587 lines). The impedance layer and wave gait are therefore the **newest** additions to the library.

Other orphaned artifacts: `transform.cpython-310.pyc` has **no corresponding source file** — a deleted Stack-A torque experiment script using `compute_impedance_control` and digit-based joint parsing. `spider_bot_control.cpython-38.pyc` (1.2 KB) is an early prototype from `/home/rasheed/dev_ws`, not the current 1904-line file.

Three developer home paths are embedded in source: `/home/rasheed/dev_ws` (`train_data_collector.py:22`), `/home/chia/spider_ws` (`spider_bot_control.py:39`), and `~/spider_ws` (`test_node.py:17`, `plot_thesis.py:6`).

### 7.3 Legacy dependency classification

The four absent driver methods (`RunServoInTime`, `RunServoInTimeByLeg`, `ReadPosition`, `ReadCurrentPosition`) are **all time-based-profile playback methods**. `RunServoInTime` wrote to `ADDR_PROFILE_ACCELERATION_TIME` / `ADDR_PROFILE_TIME_SPAN`, handing each servo a timed trajectory profile for its internal interpolator to execute — the correct design for LUT gait playback.

**These are classified as LEGACY DEPENDENCIES.** They are required only by `spider_bot_control.py` (a legacy Jetson-era component) and are **not required by the external-computer control path**. A learned policy issues position targets at a fixed control rate and requires immediate application under fixed PD gains; servo-side profile interpolation is actively undesirable for sim-to-real fidelity.

**The new project is not dependent on recovering the Jetson-side control stack.**

### 7.4 How the previous work ended

`CONFIRMED`

Build logs record 39 colcon builds between **2026-05-29** and **2026-06-12**. The **last 8 consecutive builds all failed** (2026-06-11 19:35 through 2026-06-12 22:51) with:

```
error: can't copy '/home/chia/spider_ws/build/spider_bot/config/bridge_params.yaml':
doesn't exist or not a regular file
```

Root cause: stale `--symlink-install` symlinks in `build/spider_bot/config/` pointing at `bridge_params.yaml` and `bridge.yaml`, which had been deleted from source. The project was **abandoned in a broken-build state** after eight retries across two days.

**The likely fix is deleting `build/`, `install/`, and `log/` and rebuilding.** This has not been attempted (no ROS2 on the current machine).

The deleted `bridge.yaml` / `bridge_params.yaml` filenames are suggestive of a `ros_gz_bridge` configuration — i.e. a possible attempted migration to modern Gazebo — but both files are gone and their contents are `NOT FOUND IN THIS REPOSITORY`. This must not be reported as a confirmed migration attempt.

---

## 8. Experiment Artifacts

Three top-level folders contain committed experimental data with matching plotting scripts:

| Folder | Data | Generating script |
|---|---|---|
| `六條腿受力分析/` | `six_leg_effort_data.csv` (`time, t1..t6`) | `run.py:26-28` writes this exact filename and header — `LIKELY BUT NOT FULLY VERIFIED` |
| `腿1、4力矩分析/` | `joint_torque_analysis.csv` (`time, L1_J1..L1_J3, L4_J1..L4_J3`) | `standup.py:20` — `LIKELY BUT NOT FULLY VERIFIED` |
| `五軸測量數據與圖/` | `analysis_data.csv` (`time, t1..t6`) | **Generator unknown — it is NOT the co-located `fix_axis_run.py`.** `CONFIRMED` contradiction, see below |

`控制圖/FSM.png` and `控制圖.jpg` are images and were not parsed; their correspondence to the `if/elif` mode dispatch in `spider_bot_control.py` is `UNRESOLVED`.

**Caveat:** `run.py` and `standup.py` are described in `README.md` as general walking / stand-up scripts, but their code is a Gazebo torque-experiment data-collection harness. Treat them as experiment tooling, not general-purpose controllers.

### 8.1 `五軸測量數據與圖/` does not contain five-axis data

`CONFIRMED`

The folder name and `README.md` both claim this directory holds xyz / pitch / roll trajectories recorded during walking. It does not.

- `fix_axis_run.py:28` writes a **9-column** header — `['time','actual_x','actual_y','actual_z','ref_x','ref_y','ref_z','pitch','roll']` — to `/home/chia/spider_ws/experiment_data.csv` (`:26`). **No file named `experiment_data.csv` exists anywhere in this repository.**
- The committed `analysis_data.csv` has a **7-column** header: `time,t1,t2,t3,t4,t5,t6` — the same structure as `six_leg_effort_data.csv`, though a distinct file (different checksum; 3091 vs 2483 lines).

Therefore the co-located script did **not** produce the committed data, and the committed data contains **no xyz, pitch, or roll columns**. The actual generator of `analysis_data.csv` is `UNRESOLVED`.

**Do not cite this folder as pose-tracking or body-stability evidence.** Doing so would attribute pose measurements to a file that contains none.

---

## 9. Documentation vs. Implementation Mismatches

`CONFIRMED`

| Claim | Source | Reality |
|---|---|---|
| IMU is **BNO055** | `src/spider_bot/README.md:21` | Code path is a **WitMotion WT901BLE67** over UDP:9001 (`imu_reader.py`) |
| `test_node` is a **torque control test** (力矩控制測試檔案) | `README.md`, `操作指南手冊.txt` | It publishes **positions** to a position controller |
| `run` / `standup` are walking / stand-up controllers | `README.md`, `操作指南手冊.txt` | They are torque-experiment data-collection harnesses (§8) |
| Workspace is `~/spider_ws` | `README.md`, `操作指南手冊.txt` | Actual checkout is `~/桌面/seminar/spider_robot_ws`; four hardcoded paths are wrong |
| Models can be "freely switched" in `gazebo_launch.py` | `README.md` | The URDF is hardcoded at `gazebo_launch.py:15`; switching requires editing the file. `spider_tank.urdf` is referenced by no launch file. |
| Code comments describe an **"Isaac Sim Bridge"** | `SpiderBotDriver.py:181-254` | Plain ROS2 topic pub/sub with no Isaac API; publishes to an orphan topic |
| `五軸測量數據與圖/` holds **xyz / pitch / roll** walking curves | `README.md`, `操作指南手冊.txt`, folder name | Committed `analysis_data.csv` has columns `time,t1..t6` — no pose data. The co-located `fix_axis_run.py` writes 9 different columns to `experiment_data.csv`, a filename absent from the repository. See §8.1 |

---

## 10. Component Triage for the New Direction

| Component | Verdict | Rationale |
|---|---|---|
| `SpiderBotLib.py` | **PRESERVE** — highest-value asset | `VERIFIED BY EXECUTION`; zero ROS/Jetson coupling. Serves as Isaac Lab reference gait and as the classical baseline for comparison. |
| `SpiderBotDriver.py` (real mode) | **PRESERVE**, minimal repair later | Provides exactly the bulk sync-write/read API the external-computer path needs (§5.4). |
| `SpiderBotDriver.py` (`sim_mode`) | **RETIRE** | Orphan topic, Stack-A-coupled, invented unit constant. Isaac Lab drives articulations directly. |
| `spider_bot_control.py` | **RETIRE to historical reference** | Legacy RC/SBUS + web-console + LUT playback; four confirmed defects; requires the legacy timed-profile API. |
| `autostart_scripts/` | **RETIRE to historical reference** | Jetson-specific deployment. |
| URDFs / inertias / meshes | **REWRITE** — gating prerequisite | §4.1. Blocks credible sim-to-real. |
| Gazebo stacks, worlds, launch files | **FREEZE as reference** | EOL Galactic + EOL Gazebo Classic; superseded by Isaac Lab. |
| `test_node.py`, `model_1499.pt` | **RETIRE / evidence only** | §5.7. |
| `build/`, `install/`, `log/` | **DELETE and gitignore** | §3.1, §7.4. |
| SVM inspection pipeline | **PRESERVE as-is** | Self-contained and working; out of scope for locomotion research but a genuine prior result. |

---

## 11. Known Technical Limitations and Risks

1. **URDF fidelity is the gating blocker** for Isaac Lab and sim-to-real (§4.1). Placeholder masses, 3 shared inertia tensors, no meshes, invented joint limits.
2. **No RL infrastructure exists** (§5.7). Environment, reward, termination, curriculum, and training pipeline must all be built.
3. **No climbing terrain or climbing controller exists** (§5.3, §5.5). Any high-obstacle climbing claim requires building both.
4. **No terrain perception exists.** The depth camera does not publish to ROS. If the `model_1499.pt` height-scan inference (§5.7) is correct, this gap is fundamental to reusing that policy.
5. **Control-rate ceiling.** `BAUDRATE = 1000000` with an 18-servo sync-read (10 B each) plus sync-write implies roughly 6–8 ms of bus time per cycle before host overhead — a practical ceiling near 100 Hz. The XM430 supports up to 4.5 Mbps; raising the baud rate is a cheap, high-value change when the real-robot path is built.
6. **Latency/jitter is a new sim-to-real variable** introduced by external-computer control that did not exist onboard. Its magnitude depends on the unresolved transport (§2.2). If a policy is trained at a fixed rate with zero latency and deployed under variable latency, transfer failure is a well-known outcome.
7. **Undeclared dependencies** (§5.1) will break a clean build, notably the missing `position_controllers` / `velocity_controllers`.
8. **Gazebo Classic and ROS2 Galactic are both end-of-life** and do not install on Ubuntu 24.04.
9. **The repository was abandoned in a failing-build state** (§7.4).

---

## 12. Unresolved Items

| # | Item | Blocking? |
|---|---|---|
| 1 | **Physical communication interface between external computer and robot** (§2.2) | Blocks real-robot deployment specification. **Does not block Isaac Lab development.** |
| 2 | Whether the new robot has CAD from which an accurate URDF can be exported | Blocks §11.1 |
| 3 | Which URDF, if any, resembles the new robot's geometry | Blocks §11.1 |
| 4 | Actuator complement of the new robot (still 18 × XM430-W350?) | Blocks actuator modelling |
| 5 | Whether `model_1499.pt`'s 253-dim observation includes a 187-point height scan | Affects whether the checkpoint is reusable at all |
| 6 | Contents of the deleted `bridge.yaml` / `bridge_params.yaml` | Not blocking; historical curiosity |
| 7 | Contents of `控制圖/FSM.png` and `docs/hexapod_inspection_mode.pdf` | Not blocking |
| 8 | Whether any Gazebo stack ever successfully walked | Not blocking; affects trust in inherited results |
| 9 | Which IMU is physically mounted on the new robot | Blocks sensor integration |

---

## 13. Maintenance Notes

Per `CLAUDE.md` §16, this document should not be modified except when the user requests it or the task explicitly includes documentation updates.

When updating:
- Preserve the `NOT FOUND IN THIS REPOSITORY` vs `CONFIRMED NOT TO EXIST` distinction (§1).
- Preserve the separation between the intended architecture (§2) and historical lineage (§7).
- Do not promote `PROPOSED` or `UNRESOLVED` items to `CONFIRMED` without evidence.
- Repository evidence takes precedence over this document if it becomes outdated.
- Research direction, novelty claims, and experiment design belong in `docs/RESEARCH_CONTEXT.md`, not here.

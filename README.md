# Energy-Efficient and Dynamics-Robust 7-DOF Arm Control

A reproducible robotics experiment comparing conventional control with **PPO, SAC and TD3** on a randomized reaching task in MuJoCo. Policies directly command seven joint torques on a Franka Panda-style arm. The project evaluates reaching accuracy, control effort, action smoothness and robustness to changes in rigid-body dynamics.

**[Open the completed notebook →](Panda_Torque_RL_colab_v3.ipynb)**

The uploaded notebook contains an executed **STANDARD** experiment, including training logs, held-out evaluation, reward ablations, robustness curves, joint trajectories and five embedded videos. The results below come from those saved outputs.

> **Main finding:** The conventional PD baseline performed best under nominal dynamics. TD3 achieved the highest observed RL success rate, but this run did **not** demonstrate an RL advantage in control effort or smoothness. Dynamics shifts exposed weaknesses in both conventional and learned control.

## Research question

Can deep reinforcement learning learn accurate, smooth, low-effort joint-level control of a seven-DOF arm while remaining robust to changes in its physical dynamics?

The experiment follows this loop:

```text
Joint state + target + action history
                  ↓
          PD / PPO / SAC / TD3
                  ↓
         Seven joint torque commands
                  ↓
          MuJoCo rigid-body dynamics
                  ↓
     Reaching error, effort and smoothness
```

MuJoCo simulates the physics; the learned policies select actions within those dynamics. The RL controllers do not use inverse kinematics, Cartesian position commands or gravity compensation.

## Measured nominal results

Each controller was evaluated on the **same 48 held-out targets**. RL results use training seed **11**. Success requires remaining within **3 cm** of the target for **10 consecutive control steps**, equivalent to 0.2 seconds.

| Controller | Success % [95% CI] | Mean final error (cm) | Time to success (s)¹ | Mean squared torque ((N·m)²)² | Action variation per step³ |
|---|---:|---:|---:|---:|---:|
| PD + nominal gravity compensation | 100.00 [92.59, 100.00] | 1.184 | 0.730 | 664.54 | 0.0330 |
| PPO | 0.00 [0.00, 7.41] | 20.577 | — | 18,758.80 | 0.0931 |
| SAC | 31.25 [19.95, 45.33] | 4.935 | 1.287 | 3,760.85 | 1.7840 |
| TD3 | 43.75 [30.70, 57.73] | 5.010 | 1.680 | 2,997.63 | 2.5072 |

¹ Includes the required hold time and averages **successful episodes only**: 48 for PD, 0 for PPO, 15 for SAC and 21 for TD3. The notebook also reports capped acquisition time, assigning failures the 4-second horizon.

² For each episode, integrated squared actuator torque divided by episode duration, then averaged across episodes. This is an **effort proxy, not physical energy**. Cumulative effort and absolute actuator mechanical work are reported separately in the notebook.

³ Mean per-control-step sum of squared changes in the seven normalized action commands; lower is smoother under this metric. This does not measure joint jerk.

The success intervals are Wilson intervals across evaluation episodes. The notebook also reports bootstrap intervals for continuous metrics and paired differences against PD. These intervals do **not** estimate variation across independent training runs.

### What the results support

- **PD was the strongest nominal controller** in this experiment: it reached every test target, with lower error, effort rate and action variation than the learned policies. It benefits from IK and an analytical nominal gravity model.
- **TD3 had the highest observed RL success rate**, while SAC had a slightly lower mean final error. Their success intervals overlap; one training seed does not establish a general algorithm ranking.
- **PPO did not satisfy the success criterion** within this training setup and budget. That is a result of this experiment, not evidence that PPO cannot solve torque-controlled reaching.
- **Low effort or low action variation alone is insufficient.** For example, PPO's commands varied less than SAC's and TD3's, but PPO never completed the task and used much more squared torque.

## Zero-shot dynamics robustness

The selected policies were frozen before testing. Each perturbation changed one factor at a time, with the other parameters restored to nominal values. The same **16 predeclared test targets** were used across controllers and conditions.

| Physical factor | Tested shifts |
|---|---|
| Mass and inertia of links 4–7 | 0.8×, 0.9×, 1.1×, 1.2× |
| Additional end-effector payload | 0.25, 0.5, 1.0 kg |
| Joint damping | 0.5×, 1.5×, 2.0× |
| Joint friction loss | 0.5×, 1.5×, 2.0× |
| Actuator strength | 0.8×, 0.9×, 1.1×, 1.2× |

This gives **17 perturbed conditions and 1,088 robustness episodes** across the four main controllers. Nominal reference points are included in the curves. Mass changes scale inertia consistently; the payload has its own inertia; motor strength changes effective torque production.

Selected findings from the saved results:

- PD's largest success decrease was **100 percentage points under a 1.0 kg payload**, relative to its nominal performance on the same 16 targets.
- SAC's largest observed decrease was **6.25 percentage points** at a **0.8× link-mass multiplier**.
- TD3's largest observed decrease was **18.75 percentage points** at a **0.8× link-mass multiplier**.
- PPO's worst success change was zero. **This is not evidence of robustness:** a controller with zero nominal success has no success rate left to lose.

These are paired changes on the 16-target subset, not differences from the 48-target averages in the nominal table. Smaller degradation alone does not imply better absolute performance. The notebook contains the full condition curves and measures error, effort, smoothness, acquisition time and return alongside success.

## Controlled reward ablation

**TD3 was selected using validation performance**, before opening the test set. Three additional policies were trained from scratch with the same interaction budget, learning rate, seed and target distribution:

| Variant | Reward terms |
|---|---|
| A — Reach | Dense reaching reward |
| B — Effort | A + normalized squared-torque penalty |
| C — Smoothness | B + action-change penalty |
| D — Full | C + success bonus, joint-limit penalty and collision penalty; the main TD3 reference |

The paired held-out comparisons show:

- **B versus A:** no observed success-rate change. Mean squared torque increased by 294.8 (N·m)², with a 95% interval of **−516.1 to +1,044.5**. This run does not show that adding the effort penalty alone reduced effort.
- **C versus B:** success increased by **31.25 percentage points** [16.67, 45.83], final error decreased by **4.734 cm** [3.025, 6.356], and mean squared torque decreased by **1,463.9 (N·m)²** [974.2, 1,990.0].
- Despite those improvements, C's change in action variation was **−0.335** [−1.016, +0.317]. The interval crosses zero, so a clear reduction in the smoothness metric was **not established**.

These are episode-level intervals conditional on one trained policy per variant. They do not establish repeatability across training seeds. C versus D changes several reward terms together and cannot isolate a single term's effect. Returns are not used to compare variants with different reward definitions.

## Task and control interface

- **Robot:** pinned MuJoCo Menagerie Panda model, modified to retain seven arm joints and a rigid hand. Finger bodies and their controls are removed.
- **Actions:** seven continuous values in `[-1, 1]`, mapped directly to joint motor torques. Nominal limits are ±87 N·m for joints 1–4 and ±12 N·m for joints 5–7.
- **Observations:** 32 values: 23 joint/Cartesian state values, seven previous commands, hold progress and remaining-time fraction. Fixed scaling avoids fitting normalization statistics on test data.
- **Targets:** generated by forward kinematics from sampled joint configurations. Candidates must satisfy workspace bounds and collision checks, including a sampled joint-space path from home. The generating configuration is never supplied to the policy.
- **Timing:** 2 ms simulation steps; 20 ms control steps (50 Hz); maximum 200 control steps per episode (4 seconds).
- **Reward:** nonpositive dense distance reward, a success bonus, normalized effort and action-change costs, and penalties near joint limits and for undesirable contacts. Components are logged separately.
- **PD reference:** deterministic numerical IK supplies a joint target; PD feedback plus nominal gravity compensation supplies torque. Gains are selected on validation targets. Its internal dynamics model remains nominal during robustness tests.

The task is position-only reaching from a fixed initial pose within a local target distribution. Orientation, vision, grasping and manipulation are outside the implemented scope.

## Experimental protocol

| Setting | Executed STANDARD configuration |
|---|---|
| Global seed | `20260922` |
| Training seed | `11` |
| Target-bank seeds | Training `101`; validation `202`; test `303` |
| Training target bank | 4,096 configurations |
| Validation targets | 12 fixed targets |
| Nominal test targets | 48 fixed targets |
| Pilot budget | 32,768 steps per learning-rate candidate per algorithm |
| Learning-rate candidates | `3e-4`, `1e-4` |
| Selected learning rates | `3e-4` for PPO, SAC and TD3 |
| Final training budget | 196,608 steps per algorithm |
| Additional reward-ablation budget | 196,608 steps each for A, B and C |
| MLP hidden layers | `[128, 128]` |
| Optional observation ablation | Disabled in this run |

The default campaign uses **1,376,256 training interaction steps**, excluding evaluation. All algorithms receive equal interaction budgets, although their optimization workloads differ.

Checkpoint selection is lexicographic: higher validation success, lower mean final error rounded to 1 mm, lower effort rate, then lower action variation. Final runs start fresh after learning-rate selection. All policies, including ablations, are frozen before test evaluation. Training reward does not select the final policies.

The executed notebook reports **336 nominal evaluation episodes**, including reward variants, with **zero numerical failures** in that nominal campaign. Automated checks cover dimensions, finite observations/rewards, deterministic transitions, reachable targets, torque bounds, joint-limit penetration and physical-parameter restoration.

## Run in Google Colab

1. Open [Google Colab](https://colab.research.google.com/) and upload [`Panda_Torque_RL_colab_v3.ipynb`](Panda_Torque_RL_colab_v3.ipynb).
2. Start a fresh Python session. If the session has already imported TensorFlow or JAX, restart it before running this notebook.
3. Leave `MODE = "STANDARD"` to reproduce the full protocol, or use `"QUICK"` to verify the pipeline with small budgets. QUICK uses separate target-bank seeds and fewer robustness conditions; its results are not research conclusions.
4. Select **Run all**. The notebook installs required packages and downloads the public robot assets automatically. No API keys are needed.

The v3 setup preserves Colab's installed scientific packages, disables SB3's unused TensorBoard import path, and uses OSMesa for headless rendering. A subprocess tests imports, MuJoCo stepping and actual rendering before the experiment begins. Logging uses CSV files rather than TensorBoard.

The saved run used **CPU execution with two available CPU cores and no CUDA**. A GPU is not required. The notebook targets roughly 60–90 minutes, but this is not a guaranteed runtime. Its saved pilot-based estimate was **79.6 minutes for training, plus evaluation/rendering**; that estimate is not a measured end-to-end duration.

### Executed software versions

| Component | Version |
|---|---|
| Python | 3.13.15 |
| MuJoCo | 3.3.7 |
| Gymnasium | 1.2.2 |
| Stable-Baselines3 | 2.7.1 |
| PyTorch | 2.11.0+cpu |
| NumPy | 2.1.3 |
| Pandas | 2.2.3 |
| Matplotlib | 3.10.0 |
| SciPy | 1.16.3 |
| ImageIO / ImageIO-FFmpeg | 2.37.0 / 0.6.0 |

Fresh Colab sessions may provide different scientific-library versions. The notebook records the actual versions and includes them in cache compatibility checks; matching seeds alone does not guarantee identical results across software and hardware changes.

## Repository and generated artifacts

Place these two files together at the repository root:

```text
README.md
Panda_Torque_RL_colab_v3.ipynb
```

The notebook already contains the saved plots, tables and embedded videos. No separate image folder is required for this README. Video playback depends on the notebook viewer; open the notebook in Colab if a repository preview does not display it.

Running the notebook creates a configuration-hashed directory under `/content/panda_rl/`, including:

- Configurations, dependency versions, source-asset hashes and target-bank files.
- Selected model checkpoints, optimizer state, and replay buffers for SAC/TD3 resume checkpoints.
- Raw training episodes and reward components, validation logs and policy-freeze metadata.
- Nominal and robustness CSVs, paired effects, confidence intervals and reward-ablation results.
- Workspace, learning-curve, comparison, robustness and joint-trajectory plots.
- Videos for PD, PPO, SAC, TD3, and a frozen TD3 policy with a 0.5 kg payload.
- A final comparison table, generated discussion and artifact manifest.

**Uploading the notebook does not upload these separate checkpoint and CSV files.** Preserve the run directory separately if trained-model reuse or raw-data access is needed. To retain artifacts across Colab sessions, mount Google Drive and set `CACHE_DIR` to a directory on that mounted drive before running the configuration cell. Otherwise, Colab's local files are temporary.

Compatible completed experiments load existing checkpoints; interrupted runs can resume from saved model and replay state. Resume is not bit-for-bit continuation of the interrupted simulator trajectory or random-number state.

## Limitations and next steps

This is a simulation study with one training seed, one robot morphology, a fixed starting pose and a local position-target distribution. Reward design, hyperparameters and training budget affect the results. Episode-level confidence intervals do not replace independent training replications.

Squared torque is a proxy for effort. Reported absolute mechanical work excludes electrical losses and regeneration accounting. Successful termination shortens episodes, so cumulative effort must be interpreted alongside duration-normalized effort, accuracy and success. Contact counts represent penetrating contact-pair substeps, not unique impacts; joint-limit constraints are compliant rather than perfectly rigid.

The PD controller has analytical knowledge unavailable to the RL policies, while robustness tests deliberately leave that knowledge nominal. Finite one-factor perturbations do not establish robustness to arbitrary or combined changes. The current evidence does not support a claim of reliable, energy-efficient RL control superior to conventional control, or readiness for real hardware.

Useful next experiments include independent training-seed replications, further validation-only tuning, wider initial-state distributions, domain randomization, recurrent policies and combined dynamics shifts. Orientation/6D pose control, obstacles, manipulation and sim-to-real transfer remain future extensions. Any new model-selection work should preserve a fresh held-out test set.

## Model attribution

The robot model comes from [Google DeepMind's MuJoCo Menagerie](https://github.com/google-deepmind/mujoco_menagerie/tree/822c2d8f877dd166c5b7d3c9f7e3c3b6589473b7/franka_emika_panda), pinned to commit `822c2d8f877dd166c5b7d3c9f7e3c3b6589473b7`. Setup downloads the model's license alongside its assets. The notebook documents the torque-actuator conversion, finger removal, payload attachment, explicit joint friction and tighter joint-stop settings, and includes references for the simulator and RL algorithms.

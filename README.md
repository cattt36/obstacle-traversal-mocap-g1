# Human Obstacle Traversal: Motion Capture, Force Plates and Unitree G1 References

This dataset accompanies the paper

> **Decoupling Rhythm from Skill: Layout-Adaptive Execution of Humanoid Skills from Human Demonstrations**
> (https://github.com/cattt36/layout-adaptive-humanoid-skills)

It contains whole-body human demonstrations of three obstacle-traversal skills, recorded by optical motion capture synchronized with force plates placed at the footholds on the obstacles. Each trial is paired with its motion retargeted to the 29-DoF Unitree G1, the processed robot reference and the obstacle layout used in simulation. Natural-language instructions are provided for each task.

| | |
|---|---|
| Tasks | Crouched traverse under a bridge (CT), three-step pole run (3PR), five-step pole run (5PR) |
| Trials | 76 (CT 27, 3PR 21, 5PR 28) |
| Subjects | 3 healthy adults, anonymized as A, B and C |
| Total duration | about 10 min |
| Human data | Whole-body motion capture (120 Hz), force plates (1000 Hz), force-plate gait events |
| Robot data | Retargeted Unitree G1 motion (120 Hz), processed simulation reference (50 Hz) |
| Layouts | Stake positions (3PR, 5PR) and bridge geometry (CT) for every trial, in the frame of the robot reference |
| Language | 1,187 in-scope and out-of-scope instructions in English and Chinese, plus a sealed test set of 96 cases |
| Splits | Trial-level four-fold cross-validation used in the paper |

## Tasks and capture setup

- **CT (CrouchedTraverse)**: crouch and walk under a low wooden plank bridge. The outline of the bridge was marked with reflective markers, and the subjects stepped on six force plates placed along the path.
- **3PR / 5PR (ThreeStepPoleRun / FiveStepPoleRun)**: run across a row of three or five stakes, stepping on every stake. No physical stakes were set up. The subjects stepped on three or five force plates that emulated the stakes, so that the frame and the foot of every stake touchdown are known from the force-plate events.

The motion was captured by a 60-camera optical motion capture system (FZMotion 120A, Luster LightTech, Beijing, China) at 120 Hz, and the ground reaction forces were measured by force plates (Kunwei, Shanghai, China) at 1000 Hz. The force-plate data were synchronized with the motion frames by the motion capture software. Low-quality trials were removed, which leaves the 76 trials released here.

## Directory structure

```
human/<Task>/
    bvh/<trial>.bvh                          whole-body motion (BVH)
    mocap_csv/<trial>.csv                    whole-body motion (CSV export of the capture software)
    force_plates/<trial>_force_plates.csv    forces, moments and centre of pressure of every plate
    gait_events/<trial>_gait_events.csv      touchdowns, liftoffs and phases detected from the force plates
robot/<Task>/
    retargeted_120hz/<trial>_retargeted.npz  retargeted Unitree G1 motion
    retargeted_120hz/<trial>_retargeted.json its metadata (frame rate, joint names, quaternion format)
    reference_50hz/<trial>_motion.npz        processed reference used in simulation
layouts/<Task>/
    <trial>_stakes.json                      3PR, 5PR
    <trial>_bridge.json                      CT
instructions/                                instruction sets (see below)
metadata/
    trials.csv                               trial, task, subject, cross-validation fold, frames, duration
    folds.json                               test trials of the four folds
    splits/fold<k>.json                      train / val / test trials of fold k
```

`<Task>` is `CrouchedTraverse`, `ThreeStepPoleRun` or `FiveStepPoleRun`, and `<trial>` is `<Task>_<number>`, for example `FiveStepPoleRun_001`. The numbering is not contiguous because removed trials keep their numbers.

## File formats

### Human motion

- **BVH** (`human/<Task>/bvh`): skeleton hierarchy and 120 Hz motion. The skeleton is Y-up, and offsets and root positions are in centimetres.
- **CSV** (`human/<Task>/mocap_csv`): the same motion exported by the capture software. The first line holds the recording metadata (frame rate, frame count, length unit mm). The following header lines give, per column, the bone name (prefixed by the subject label, for example `A:Hips`), its ID, and whether the column is a rotation (quaternion `X, Y, Z, W`) or an offset (`X, Y, Z`, mm). Rotations and offsets are relative to the parent bone, and the offset of the hips is their position in the capture volume.

### Force plates

`human/<Task>/force_plates/<trial>_force_plates.csv` has one row per force-plate sample (1000 Hz):

| Column | Meaning |
|---|---|
| `DeviceFrameID` | sample index of the force plates (1000 Hz) |
| `TimeCode` | time stamp in ms |
| `MocapFrameID` | motion capture frame (120 Hz) of this sample, for synchronization |
| `DeviceID, Fx, Fy, Fz, Mx, My, Mz, CopX, CopY` | repeated for every plate: plate ID, force (N), moment, centre of pressure (zero when the plate is unloaded) |

The 3PR and 5PR trials use three or five plates, one per emulated stake, and the CT trials use six plates (IDs 1–6). TODO: confirm the units of the moments and the centre of pressure, and the plate coordinate frame.

### Gait events

`human/<Task>/gait_events/<trial>_gait_events.csv` contains the events detected from the force plates. Times are in seconds and frames are force-plate samples (1000 Hz).

- **3PR and 5PR**: one row per phase, with `CycleID`, `Phase` (Chinese) and `PhaseEN` (`Preparation`, `LeftFootContact`, `RightFootContact`, `Flight`, ...), `Foot` (左脚 = left, 右脚 = right), `PlateInfo` and `MainPlate` (`P<k>` is the plate with `DeviceID` k), `StartTime_s`, `EndTime_s`, `Duration_s`, `StartFrame`, `EndFrame`, `CopPeak`, `CopMean`, `FlightAfter_s` and `Notes`.
- **CT**: one row per step, with `Foot` (`L`, `R`), `Step`, the initial contact `IC_Time_s` / `IC_Frame`, the toe-off `TO_Time_s` / `TO_Frame`, `Stance_Duration_s`, `Swing_Duration_s`, `Cycle_Duration_s`, and the plates `IC_Plate` and `TO_Plate`.

### Retargeted Unitree G1 motion (120 Hz)

The human motions were retargeted to the 29-DoF Unitree G1 with General Motion Retargeting (GMR) using its default scaling, by TODO (collaborating group). `<trial>_retargeted.npz` contains

- `data`: float64 array `[T, 36]` with the root position (m, z-up), the root orientation as a quaternion in `x, y, z, w` order, and the 29 joint angles (rad) in the order of `joint_names` in the JSON file,
- `framerate`: 120.

These motions are unfiltered and not cropped. Their frame is the world frame of the retargeting, which differs from the frame of the 50 Hz references and the layouts.

### Processed robot reference (50 Hz)

`<trial>_motion.npz` is the reference tracked in simulation in the paper. Compared with the retargeted motion, it is low-pass filtered with a Gaussian kernel (σ = 2 frames), aligned to the ground, cropped to the task and resampled to 50 Hz. For 3PR and 5PR, the stake-to-stake jumps of the human exceed the jumping range of the G1, so the horizontal travel is reduced to 93 % (3PR) and 83 % (5PR) by scaling the root displacement during the flight phases only, and the reference is raised by the stake height (0.10 m) between the first and the last stake stance. The arrays are

| Key | Shape | Content |
|---|---|---|
| `fps` | scalar | 50 |
| `joint_pos`, `joint_vel` | `[T, 29]` | joint angles (rad) and velocities, same joint order as the retargeted motion |
| `body_pos_w` | `[T, 30, 3]` | body positions (m, z-up) |
| `body_quat_w` | `[T, 30, 4]` | body orientations, quaternion in `w, x, y, z` order |
| `body_lin_vel_w`, `body_ang_vel_w` | `[T, 30, 3]` | body linear and angular velocities |
| `foot_contact` | `[T, 2]` | contact flags of the left and right foot |

The 30 bodies are, in order: `pelvis`, `left_hip_pitch_link`, `left_hip_roll_link`, `left_hip_yaw_link`, `left_knee_link`, `left_ankle_pitch_link`, `left_ankle_roll_link`, `right_hip_pitch_link`, `right_hip_roll_link`, `right_hip_yaw_link`, `right_knee_link`, `right_ankle_pitch_link`, `right_ankle_roll_link`, `waist_yaw_link`, `waist_roll_link`, `torso_link`, `left_shoulder_pitch_link`, `left_shoulder_roll_link`, `left_shoulder_yaw_link`, `left_elbow_link`, `left_wrist_roll_link`, `left_wrist_pitch_link`, `left_wrist_yaw_link`, `right_shoulder_pitch_link`, `right_shoulder_roll_link`, `right_shoulder_yaw_link`, `right_elbow_link`, `right_wrist_roll_link`, `right_wrist_pitch_link`, `right_wrist_yaw_link`.

The peak torque demand of these kinematic references can exceed the G1 actuator limits, so not every reference can be executed as is.

### Obstacle layouts

The layout of each trial is reconstructed in the frame of its 50 Hz reference, so that the stakes follow the footholds of the trial. All lengths are in metres and angles in radians.

**Stakes** (`<trial>_stakes.json`, 3PR and 5PR): cylinders with `radius` 0.15 and `height` 0.10.

| Key | Content |
|---|---|
| `stakes` | one entry per stake in the order of traversal: stance `foot` (`L`, `R`), centre `x`, `y`, the `stance` window `[first, last]` in frames at `fps`, and `csv_frame`, the motion capture frame of the stake touchdown |
| `stake_xyz_50hz` | stake centres `[x, y, z]` (z = half the height) |
| `fps` | frame rate of the `stance` windows (120). A window maps to frames of the 50 Hz reference by multiplying by `50 / fps` |
| `stride_scale`, `flight_scale` | the scaling of the horizontal travel described above |
| `ramp` | frames (at `fps`) of the ramps by which the reference is raised onto and lowered from the stake height |

**Bridge** (`<trial>_bridge.json`, CT): the marker-defined bridge mapped into the robot frame by a similarity transform between the toe positions on the force plates and the robot toe positions.

| Key | Content |
|---|---|
| `parts` | `plank` (the beam), `ramp` and `leg` (inclined end supports) as boxes with `center` and half extents `half`, and `post0`–`post3` as cylinders with `center`, `radius` and `half_height`. Every part has a `yaw` and `pitch` |
| `scale`, `fit_err_m` | scale and residual of the similarity transform |
| `plank_lift_m` | height by which the beam is raised so that its underside clears the reference by 3 cm (the G1 crouches less deeply than the human relative to its body size) |
| `ref_min_dist_m` | minimum distance between the reference and each part |

The support posts were placed at positions taken from the video recorded during the acquisition.

### Instructions

Instructions are given per task (`CrouchedTraverse`, `ThreeStepPoleRun`, `FiveStepPoleRun`), not per trial. Out-of-scope instructions are commands the system should reject or confirm. Each file has a `description` field.

| File | Content | Size |
|---|---|---|
| `v1.json` | `train` and `val`: in-scope instructions per task, `val_reject`: out-of-scope instructions | 135 / 36 / 20 |
| `train_aug_v3.json` | `train`: implicit instructions that describe the obstacle instead of naming the skill, and variants with simulated speech-recognition noise, per task, `reject`: out-of-scope instructions | 423 / 96 |
| `train_negatives_v3b.json` | `reject` and `reject_category`: further out-of-scope training instructions, including hard negatives that share words with the tasks | 266 |
| `val_v3.json` | independent validation set: `val` in-scope per task, `val_reject` with `val_reject_category` (`unrelated`, `other_locomotion`, `wrong_use_of_obstacle`, `ambiguous`, `noisy_reject`) | 87 / 124 |
| `test_sealed.json` | sealed test set, `cases` with `id`, `condition`, `query`, `expected_action` and `should_abstain`. 48 in-scope cases (12 each of `paraphrase`, `code_switch`, `asr_noise`, `implicit`) and 48 cases to abstain on (24 `ambiguous`, 24 `out_of_scope`) | 96 |

Part of the instructions were written by a large language model: the validation set `val_v3.json`, the training negatives `train_negatives_v3b.json`, the augmentation `train_aug_v3.json` and the sealed test set `test_sealed.json`, which was written by an independent instance from a fixed specification without access to the training data. TODO: state who wrote `v1.json`.

### Metadata and splits

- `metadata/trials.csv`: `trial`, `task`, `subject`, `cv_test_fold`, `mocap_frames_120hz`, `duration_s`.
- `metadata/folds.json`: the test trials of the four folds of the trial-level cross-validation. Every trial is a test trial in exactly one fold.
- `metadata/splits/fold<k>.json`: the `train`, `val` and `test` trials of fold k. The validation trials are used for checkpoint selection.

TODO: confirm that the subject labels A, B and C denote the same person in all three tasks.

## Loading example

```python
import json
import numpy as np

trial, task = "FiveStepPoleRun_001", "FiveStepPoleRun"

# retargeted G1 motion, 120 Hz
r = np.load(f"robot/{task}/retargeted_120hz/{trial}_retargeted.npz")
names = json.load(open(f"robot/{task}/retargeted_120hz/{trial}_retargeted.json"))["joint_names"]
root_pos, root_quat_xyzw, joints = r["data"][:, :3], r["data"][:, 3:7], r["data"][:, 7:]

# processed reference, 50 Hz, and its stakes
m = np.load(f"robot/{task}/reference_50hz/{trial}_motion.npz")
layout = json.load(open(f"layouts/{task}/{trial}_stakes.json"))
for s in layout["stakes"]:
    a, b = (round(f * 50 / layout["fps"]) for f in s["stance"])   # stance window in reference frames
    ankle = m["body_pos_w"][(a + b) // 2, 6 if s["foot"] == "L" else 12]   # left / right ankle roll link
    print(s["foot"], (s["x"], s["y"]), ankle)
```

## Ethics

TODO: ethics approval (committee, number) and informed consent of the subjects.

## License

TODO: choose a license, for example CC BY 4.0 for the data.

## Citation

```bibtex
TODO
```

## Acknowledgments

We thank Luster LightTech for providing the optical motion capture system, procuring the obstacles and arranging the motion capture subjects, and Kunwei for providing the force plates. The retargeted motions were provided by TODO.

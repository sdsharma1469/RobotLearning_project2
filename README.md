# Learning from Demonstration for Robotic Manipulation

> **ECEN 524 — Robot Learning, Project 2**
> Learning a reusable block-grasping trajectory from human video demonstrations and reproducing it on a Panda robotic arm.

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/sdsharma1469/RobotLearning_project2/blob/main/ECEN524_Project2.ipynb)

## Project Overview

This project implements an end-to-end **Learning from Demonstration (LfD)** pipeline for robotic manipulation. Human block-grasping demonstrations are recorded on video, converted into hand trajectories and grasp states, aligned across demonstrations, modeled into a representative motion, and transformed into a smooth trajectory for robot execution.

The pipeline combines computer vision with three core robot-learning techniques:

- **Dynamic Time Warping (DTW)** for temporal alignment
- **Gaussian Mixture Regression (GMR)** for learning a representative trajectory
- **Dynamic Movement Primitives (DMPs)** for smooth motion reproduction

The project is broken into three parts:

- **Part A** — Trajectory learning pipeline (DTW → GMR → DMP → robot execution)
- **Part B** — Object/hand recognition from video (grab/release event detection)
- **Part C** — MDP formulation for learning the correct block-placement order

## Pipeline

```
flowchart LR
    A[Human demonstration videos] --> B[OpenCV + MediaPipe hand tracking]
    B --> C[Wrist trajectory + grasp state]
    C --> D[DTW temporal alignment]
    D --> E[GMR trajectory modeling]
    E --> F[DMP motion generation]
    F --> G[Panda robot execution]
    G --> H[Grasp evaluation]
```

### 1. Demonstration Processing

Human block-grasping demonstrations are processed frame by frame using **OpenCV** and **MediaPipe**. The active hand's wrist position is extracted to form the motion trajectory.

The distance between the thumb tip and index-finger tip is also tracked. A smoothed hysteresis-based threshold converts this signal into an **OPEN / CLOSED** grasp state that can be synchronized with the learned trajectory.

### 2. Temporal Alignment — DTW

Different people perform the same grasp at different speeds. **Dynamic Time Warping** aligns the recorded demonstrations in time so corresponding portions of the motion can be compared and learned together.

### 3. Trajectory Learning — GMR

After alignment, **Gaussian Mixture Regression** is used to estimate a representative trajectory from the demonstrations rather than simply replaying one recorded motion.

### 4. Motion Reproduction — DMP

The learned trajectory is represented with **Dynamic Movement Primitives**, producing a smooth and reusable motion that can be executed by the robot.

### 5. Robot Evaluation

The generated trajectory is reproduced on a **Panda robotic arm** and evaluated based on whether the robot successfully completes the block grasp.

## Results (Part A)

| Metric                         | Result  |
| ------------------------------ | ------- |
| Human demonstrations processed | **10**  |
| Validation trials              | **13**  |
| Grasp completion rate          | **70%** |

The experiment shows that a manipulation behavior can be learned from multiple human demonstrations and transferred into a reusable robot trajectory, while also highlighting the sensitivity of grasp success to demonstration quality, trajectory estimation, and robot execution accuracy.

---

## Part B — Object / Hand Recognition

**Libraries used:** [MediaPipe Hand Landmarker](https://ai.google.dev/edge/mediapipe/solutions/vision/hand_landmarker)

- Detects hands in RGB images and returns 3D keypoints for the hand and its parts.
- Provides normalized landmark coordinates.
- Hand-landmark outputs and cube-detection outputs are matched by timestamp, then passed to a script that builds a table of actions taken.

To keep the model lightweight and fast, the higher-level actions **moving**, **holding**, and **releasing** are inferred purely from two hand states — **Open** and **Closed** — rather than asking the model to classify actions directly.

Cube objects (red/green/blue) are localized in each frame using HSV color segmentation and contour-based blob detection, while the hand bounding box and key landmarks (wrist + fingertips) are drawn from the MediaPipe output.

<table>
<tr>
<td align="center"><img src="images/hand_closed_grasp.png" width="380"><br><sub>CLOSED — grasping the red cube (frame 401)</sub></td>
<td align="center"><img src="images/hand_open_release.png" width="380"><br><sub>OPEN — releasing after moving (frame 615)</sub></td>
</tr>
</table>

<p align="center">
<img src="images/hand_cube_detection_overlay.png" width="600"><br>
<sub>Hand bounding box + keypoints (MediaPipe) and HSV-based cube bounding boxes drawn on a sample frame</sub>
</p>

### Example detected events

| Time (s) | Frame | Event   | Object   | Distance (px) |
| -------- | ----- | ------- | -------- | -------------- |
| 2.13     | 64    | GRAB    | RED_1    | 42.7           |
| 3.05     | 92    | RELEASE | RED_1    | —              |
| 4.22     | 127   | GRAB    | GREEN_2  | 35.3           |
| 5.10     | 153   | RELEASE | GREEN_2  | —              |
| 6.41     | 192   | GRAB    | BLUE_1   | 38.9           |
| 7.32     | 219   | RELEASE | BLUE_1   | —              |
| 8.75     | 262   | GRAB    | RED_2    | 40.1           |
| 9.60     | 288   | RELEASE | RED_2    | —              |
| 10.91    | 327   | GRAB    | GREEN_1  | 44.6           |
| 11.85    | 355   | RELEASE | GREEN_1  | —              |

The notebook cell for this part (`grab middle frame of first video, detect hand + cubes, draw boxes`) loads the MediaPipe Hand Landmarker model, reads a representative frame, overlays the detected hand bounding box + keypoints, and overlays HSV-based bounding boxes for each colored cube for visual sanity-checking.

---

## Part C — MDP for Block-Placement Order

### Description

- **States:** `k` = number of correctly placed blocks so far (`0..N`)
- **Actions:** next color to place — `{"R", "G", "B"}`
- **Transition:** if `action == learned_seq[k]`, move to state `k+1`; otherwise stay at `k`
- **Reward:** `+10` on reaching the goal state `N`, `+1` for each correct placement step, `-0.1` per-step penalty (with a `-0.5` penalty for an incorrect action)

The target placement sequence is *learned automatically* from the `RELEASE` events recorded in Part B, then an MDP is built over that sequence and solved with **value iteration** to recover the optimal placement policy.

### Results

**Learned placement sequence (from RELEASE order):** `['R', 'G', 'B', 'R', 'G']`

**Learned policy (state `k` → action):**

| k | Action |
|---|--------|
| 0 | R |
| 1 | G |
| 2 | B |
| 3 | R |
| 4 | G |
| 5 | R |

**Rollout:**

| step | k | action | reward | k_next |
|------|---|--------|--------|--------|
| 0 | 0 | R | 0.9 | 1 |
| 1 | 1 | G | 0.9 | 2 |
| 2 | 2 | B | 0.9 | 3 |
| 3 | 3 | R | 0.9 | 4 |
| 4 | 4 | G | 9.9 | 5 |

**Final placed sequence according to policy:** `['R', 'G', 'B', 'R', 'G']`

### MDP Diagram

```
===== MDP STATES =====
s0: [∅]
s1: [G]
s2: [G R]
s3: [G R B]
s4: [G R B R]
s5: GOAL STATE

===== MDP TRANSITIONS =====
From s0:
  -- Place_G / +1 --> s1
  -- Place_R / -1 --> s0
  -- Place_B / -1 --> s0
From s1:
  -- Place_R / +1 --> s2
  -- Place_G / -1 --> s1
  -- Place_B / -1 --> s1
From s2:
  -- Place_B / +1 --> s3
  -- Place_G / -1 --> s2
  -- Place_R / -1 --> s2
From s3:
  -- Place_R / +1 --> s4
  -- Place_G / -1 --> s3
  -- Place_B / -1 --> s3
From s4:
  -- Place_G / +1 --> s5
  -- Place_R / -1 --> s4
  -- Place_B / -1 --> s4
From s5: (terminal state)
  -- any action --> s5 (reward = 0)

===== OPTIMAL POLICY =====
π*(s0) = Place_G
π*(s1) = Place_R
π*(s2) = Place_B
π*(s3) = Place_R
π*(s4) = Place_G
π*(s5) = TERMINATE
```

---

## Technologies

`Python` · `OpenCV` · `MediaPipe` · `NumPy` · `pandas` · `DTW` · `Gaussian Mixture Regression` · `Dynamic Movement Primitives` · `Value Iteration / MDP`

## Repository Structure

```
RobotLearning_project2/
├── ECEN524_Project2.ipynb   # Complete Colab pipeline and experiments
├── README.md                # Project documentation
└── .gitignore
```

## Run the Project

The notebook is designed to run in **Google Colab**.

1. Click the **Open in Colab** badge above.
2. Mount Google Drive when prompted.
3. Set `VIDEO_DIR` to the folder containing the demonstration videos.
4. Run the notebook cells from top to bottom.

The notebook installs its required computer-vision dependencies within the Colab environment.

## Key Takeaway

Rather than directly replaying a single demonstration, this project builds a complete **demonstration → perception → alignment → learning → robot execution** pipeline. Hand/object recognition (Part B) extracts symbolic grab/release events from raw video, and an MDP formulation (Part C) turns those events into a verified optimal placement policy — complementing the continuous-motion learning done via DTW, GMR, and DMPs in Part A.

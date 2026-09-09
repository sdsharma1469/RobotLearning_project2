# Learning from Demonstration for Robotic Manipulation

A Robot Learning project that learns a block-grasping motion from human video demonstrations and reproduces the learned behavior on a Panda robotic arm.

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/sdsharma1469/RobotLearning_project2/blob/main/ECEN524_Project2.ipynb)

## Overview

The project builds an end-to-end **Learning from Demonstration (LfD)** pipeline that converts human demonstrations into a reusable robot manipulation trajectory. The notebook processes demonstration videos, extracts hand motion and grasp state, aligns demonstrations in time, learns a representative motion, and generates a trajectory for robot execution.

### Pipeline

1. **Human demonstration videos** — load recorded block-grasping demonstrations.
2. **Hand tracking** — use MediaPipe and OpenCV to extract wrist landmarks and estimate open/closed grasp state from thumb-to-index distance.
3. **Trajectory alignment** — use **Dynamic Time Warping (DTW)** to align demonstrations performed at different speeds.
4. **Trajectory modeling** — use **Gaussian Mixture Regression (GMR)** to learn a representative motion from the aligned demonstrations.
5. **Motion generation** — use **Dynamic Movement Primitives (DMPs)** to generate a smooth, reusable grasping trajectory.
6. **Robot evaluation** — reproduce the learned motion with a Panda robotic arm and evaluate grasp completion.

## Results

- Processed **10 human demonstration videos** in the training pipeline.
- Evaluated the learned behavior across **13 validation trials**.
- Achieved **70% grasp completion** when reproducing the demonstrated block-grasping motion.

## Computer Vision Pipeline

The notebook uses **MediaPipe Hand Landmarker** and **OpenCV** to process each demonstration frame. For the active hand, it records the wrist position and computes the distance between the thumb tip and index-finger tip. A smoothed, hysteresis-based threshold converts this signal into an `OPEN` / `CLOSED` grasp state that can be synchronized with the learned trajectory.

## Technologies

**Python · OpenCV · MediaPipe · NumPy · pandas · Dynamic Time Warping · Gaussian Mixture Regression · Dynamic Movement Primitives**

## Repository Structure

```text
RobotLearning_project2/
├── ECEN524_Project2.ipynb   # Complete Colab pipeline and experiments
├── README.md
└── .gitignore
```

## Running the Project

The easiest way to run the project is in **Google Colab** using the badge above.

1. Open `ECEN524_Project2.ipynb` in Colab.
2. Mount Google Drive when prompted.
3. Set `VIDEO_DIR` to the directory containing the human demonstration videos.
4. Run the notebook cells from top to bottom.

The notebook installs the computer-vision dependencies it needs inside Colab.

## Course Project

Developed as **ECEN 524 — Robot Learning, Project 2**.

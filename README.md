# OpenVLA-FR3: Vision-Language-Action Robot Control

Natural language control for the Franka FR3 robotic arm, combining the OpenVLA 7B vision-language-action model with real-time reactive control through MoveIt Servo.

[![ROS2 Humble](https://img.shields.io/badge/ROS2-Humble-blue)](https://docs.ros.org/en/humble/)
[![OpenVLA](https://img.shields.io/badge/Model-OpenVLA--7B-orange)](https://openvla.github.io/)
[![MoveIt2](https://img.shields.io/badge/Motion-MoveIt2%20Servo-green)](https://moveit.ros.org/)
[![Platform](https://img.shields.io/badge/Platform-Linux-lightgrey)](https://ubuntu.com/)
[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](LICENSE)

## Contents

- [Overview](#overview)
- [How It Works](#how-it-works)
- [Features](#features)
- [Architecture](#architecture)
- [Quick Start](#quick-start)
- [Usage](#usage)
- [Simulation Environment](#simulation-environment)
- [Configuration](#configuration)
- [Development](#development)
- [Technical Reference](#technical-reference)
- [License](#license)

## Overview

This project connects large vision-language models to real-time robot control. It allows the Franka FR3 to interpret plain English instructions and carry out manipulation tasks on its own. A quantized OpenVLA model runs inference on consumer GPUs, while MoveIt Servo provides reactive Cartesian control.

The core idea is a delta action formulation paired with a VLA-Servo bridge: the learned policy outputs relative pose changes, and the bridge turns them into collision-aware velocity commands. This keeps the benefits of a learned visuomotor policy while gaining the reactivity of servo control.

## How It Works

```text
Natural Language Instruction --> OpenVLA-7B (4-bit quantized)
                                      |
                           Camera Image (RGB)
                                      |
                      VLAAction (delta pose + gripper)
                                      |
                              VLA-Servo Bridge
                                      |
                      TwistStamped (velocity commands)
                                      |
                                 MoveIt Servo
                                      |
                         Joint Trajectory Controller
                                      |
                       Gazebo Simulation / Real Hardware
```

Each control cycle follows the same loop: perceive the scene, predict a delta action from the image and instruction, translate it into a twist command, solve for collision-free joint velocities, execute, and observe the result.

## Features

### Perception and Inference

- Vision-language-action inference with OpenVLA-7B, fine-tuned on LIBERO spatial tasks
- 4-bit quantization via BitsAndBytes, fitting inference into 6 GB of VRAM
- Natural language task specification in plain English

### Real-Time Control

- Reactive Cartesian velocity control through MoveIt Servo at 30 Hz
- Delta action formulation for closed-loop, relative-pose control
- Collision awareness via MoveIt planning scene monitoring
- Dual-frame control with base frame or end-effector frame reference

### Interfaces and Tooling

- Autonomous VLA control plus keyboard teleoperation for data collection and debugging
- Position, velocity, impedance, and trajectory controllers
- Docker environment with CUDA 12.1, ROS2 Humble, and PyTorch
- LIBERO-compatible camera placement and task setup
- RViz visualization and full-physics Gazebo simulation with RGB-D sensing

## Architecture

### ROS2 Packages

| Package | Role |
| ------- | ---- |
| `franka_openvla` | Integration package: VLA inference, VLA-Servo bridge, keyboard teleoperation, launch, worlds, and object models |
| `vla_interfaces` | Custom message definitions (`VLAAction`) |
| `franka_gazebo` | Simulation support |
| `franka_fr3_moveit_config` | MoveIt configuration |
| `franka_hardware` | Real robot interface |
| `franka_gripper` | Gripper control |
| `franka_msgs` | Franka messages |

### Key Nodes in `franka_openvla`

| Node | Purpose |
| ---- | ------- |
| `openvla_node.py` | VLA inference with 4-bit quantization |
| `vla_servo_bridge.py` | Translates VLA actions into Servo twist commands |
| `keyboard_servo_teleop.py` | Manual teleoperation for data collection |
| `fr3.launch.py` | Launches the full system (Gazebo, MoveIt, Servo, VLA) |

### Message Interface

```text
VLAAction.msg:
  std_msgs/Header header
  geometry_msgs/Vector3 delta_pos    # [dx, dy, dz] in meters
  geometry_msgs/Vector3 delta_rot    # [droll, dpitch, dyaw] in radians
  float32 gripper                     # 0.0 = open, 1.0 = closed
```

### Node Parameters

**OpenVLA node** (`openvla/openvla-7b-finetuned-libero-spatial`):

| Item | Value |
| ---- | ----- |
| Input | RGB images from `/rgbd_camera/image` (640x480) |
| Output | Delta actions on `/vla/delta_actions` |
| `instruction` | Task description, e.g. "pick the cube and place in red bin" |
| `unnorm_key` | Action unnormalization (default: `libero_spatial`) |
| `model_name` | HuggingFace model path |

**VLA-Servo bridge** (VLAAction to TwistStamped):

- Velocity clamping (max linear 0.3 m/s, max angular 0.5 rad/s)
- Motion timeout stopping the robot if no VLA action arrives within 0.5 s
- Keepalive with periodic zero commands to hold the Servo connection

**MoveIt Servo:**

- Control rate of 30 Hz on planning group `fr3_arm` (7-DOF)
- End-effector `fr3_hand_tcp` with collision checking at 10 Hz
- Singularity avoidance with enforced joint limits plus safety margins

## Quick Start

### Prerequisites

- Ubuntu 22.04
- NVIDIA GPU with CUDA support (6 GB+ VRAM recommended)
- Docker and Docker Compose with the NVIDIA runtime

### Setup

1. Clone the repository:

```bash
git clone <repository-url>
cd openvla-fr3
```

2. Build the Docker container:

```bash
docker-compose build
```

3. Launch the container:

```bash
docker-compose up -d
```

4. Build the ROS2 workspace (first time only):

```bash
docker exec -it vla_unified bash
source /opt/ros/humble/setup.bash
cd /ros2_ws
colcon build --symlink-install
source install/setup.bash
```

## Usage

### Launch the Full System

Start the complete VLA control stack (Gazebo, MoveIt, Servo, OpenVLA):

```bash
docker exec vla_unified bash -c "
  source /opt/ros/humble/setup.bash &&
  source /ros2_ws/install/setup.bash &&
  ros2 launch franka_openvla fr3.launch.py
"
```

This single launch brings up:

| Component | Description |
| --------- | ----------- |
| Gazebo | FR3 simulation with the manipulation environment |
| ROS2 controllers | Joint state, arm, and gripper controllers |
| MoveIt `move_group` | OMPL-based motion planning |
| MoveIt Servo | Real-time Cartesian control |
| RViz | Interactive visualization and planning interface |
| OpenVLA node | Vision-language inference |
| VLA-Servo bridge | Action-to-twist translation |
| RGB-D camera | LIBERO-style viewpoint with image, depth, point cloud, and camera info |

### Keyboard Teleoperation

Manual control for testing and data collection:

```bash
docker exec -it vla_unified bash -c "
  source /opt/ros/humble/setup.bash &&
  source /ros2_ws/install/setup.bash &&
  ros2 run franka_openvla keyboard_servo_teleop
"
```

| Keys | Action |
| ---- | ------ |
| W / S | Translate along X |
| A / D | Translate along Y |
| Q / E | Translate along Z |
| J / L | Yaw |
| I / K | Pitch |
| U / O | Roll |
| F | Toggle base frame and end-effector frame |
| + / - | Increase or decrease velocity scaling |
| SPACE | Stop all motion |
| ESC | Quit teleoperation |

### Change the VLA Task Instruction

Edit the instruction parameter in `fr3.launch.py`:

```python
Node(
    package='franka_openvla',
    executable='openvla_node',
    parameters=[{
        'instruction': 'pick up the cube and place it in the red bin',
        'unnorm_key': 'libero_spatial'
    }]
)
```

Or pass it as a launch argument:

```bash
ros2 launch franka_openvla fr3.launch.py instruction:="move the object to the blue bin"
```

## Simulation Environment

The Gazebo world contains:

- Work table as the main manipulation surface (0.6 m from the robot base)
- Colored bins (green, black, red, blue) for sorting tasks
- Target cube placed on the table
- RGB-D camera in a LIBERO-style agent viewpoint, 1.2 m forward at 1.0 m height and aimed at the workspace, publishing RGB image, depth, point cloud, and camera info

## Configuration

### Key Configuration Files

1. `config/servo_params.yaml` (MoveIt Servo settings):
   - Velocity limits (linear 0.4 m/s, rotational 0.8 rad/s)
   - Control rate (30 Hz)
   - Safety scaling factor (0.3x for conservative control)
   - Collision checking parameters

2. `config/moveit_controllers.yaml` (controller interfaces):
   - `arm_controller` (FollowJointTrajectory)
   - `gripper_controller` (GripperCommand)

3. `config/franka_gazebo_controllers.yaml` (Gazebo `ros2_control`):
   - Joint state broadcaster
   - Position, velocity, and impedance controllers
   - Controller gains and limits

### Tuning the VLA-Servo Bridge

Edit `franka_openvla/vla_servo_bridge.py`:

```python
self.linear_scale = 0.5    # Velocity scaling (0.1-1.0)
self.angular_scale = 0.3   # Angular velocity scaling
self.max_linear = 0.3      # Max linear velocity (m/s)
self.max_angular = 0.5     # Max angular velocity (rad/s)
self.timeout = 0.5         # Command timeout (seconds)
```

## Development

### Project Structure

```text
openvla-fr3/
├── src/
│   ├── franka_openvla/           # Main integration package
│   │   ├── franka_openvla/
│   │   │   ├── openvla_node.py
│   │   │   ├── vla_servo_bridge.py
│   │   │   └── keyboard_servo_teleop.py
│   │   ├── launch/
│   │   │   └── fr3.launch.py
│   │   ├── config/
│   │   │   ├── servo_params.yaml
│   │   │   └── moveit_controllers.yaml
│   │   ├── worlds/
│   │   │   └── franka_world.sdf
│   │   └── models/                # Object models
│   ├── vla_interfaces/            # Custom messages
│   ├── franka_gazebo/             # Simulation support
│   ├── franka_fr3_moveit_config/  # MoveIt configuration
│   ├── franka_hardware/           # Real robot interface
│   ├── franka_gripper/            # Gripper control
│   └── franka_msgs/               # Franka messages
├── Dockerfile
├── docker-compose.yml
└── README.md
```

### Building from Source

```bash
cd /ros2_ws
colcon build --symlink-install --packages-select vla_interfaces franka_openvla
source install/setup.bash
```

### Running Individual Nodes

```bash
# OpenVLA inference only
ros2 run franka_openvla openvla_node --ros-args -p instruction:="pick up the cube"

# VLA-Servo bridge only
ros2 run franka_openvla vla_servo_bridge

# Keyboard teleoperation only
ros2 run franka_openvla keyboard_servo_teleop
```

## Technical Reference

### OpenVLA Model

- Architecture combining a vision encoder, a language encoder, and an action decoder
- 7 billion parameters, quantized to 4-bit at roughly 3.5 GB of memory
- Fine-tuned on LIBERO spatial manipulation tasks
- Inference speed of about 5-10 Hz on consumer GPUs (RTX 3060 and above)
- Action space of 7-DOF delta actions (3 position, 3 orientation, 1 gripper)

### Control Pipeline

1. Perception: RGB camera image captured at 30 Hz
2. Inference: OpenVLA predicts a delta action from the image and instruction
3. Translation: the VLA-Servo bridge converts the action into twist commands
4. Planning: MoveIt Servo computes collision-free joint velocities
5. Execution: the joint trajectory controller drives the robot
6. Feedback: the loop closes with the next camera observation

### Safety Features

- Multi-level velocity clamping across the VLA bridge, Servo, and controllers
- Real-time collision monitoring with the planning scene
- Automatic damping near singularities
- Command timeout stopping the robot if VLA output stalls
- Joint limits enforced with configurable safety margins
- Emergency stop via the keyboard SPACE key or a ROS service call

## License

Released under the Apache 2.0 License. See [LICENSE](LICENSE) for details.

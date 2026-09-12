# MoveIt Servo Setup

## Overview

This guide documents the verified MoveIt Servo configuration for the FR3 robot, including keyboard teleoperation. Follow the steps below to bring up the arm and drive it manually.

---

## Package Structure

| Path | Purpose |
| ---- | ------- |
| `config/servo_params.yaml` | MoveIt Servo configuration (FR3-specific) |
| `config/moveit_controllers.yaml` | Controller configuration |
| `franka_openvla/keyboard_servo_teleop.py` | Keyboard teleoperation node |
| `franka_openvla/openvla_node.py` | VLA integration node |
| `launch/fr3.launch.py` | Main launch file (robot, MoveIt, and Servo) |

---

## Usage

### 1. Launch the Robot

```bash
docker exec vla_unified bash -c "source /opt/ros/humble/setup.bash && source /ros2_ws/install/setup.bash && ros2 launch franka_openvla fr3.launch.py"
```

Wait until:

- The RViz window appears
- `[INFO] [moveit_ros.planning_scene_monitor.planning_scene_monitor]: Publishing maintained planning scene...`

### 2. Launch Keyboard Teleoperation

In a second terminal:

```bash
docker exec -it vla_unified bash -c "source /opt/ros/humble/setup.bash && source /ros2_ws/install/setup.bash && ros2 run franka_openvla keyboard_servo_teleop"
```

**Expected output:**
```
[INFO] Keyboard Servo Teleop Initializing...
[INFO] Waiting for servo services...
[INFO] Servo services found. Starting servo...
[INFO] Servo started successfully!
[INFO] Keyboard Servo Teleop Ready!
```

### 3. Control the Robot

Use these keys to move the robot:

| Key | Action | Direction |
|-----|--------|-----------|
| **W** | Forward | +X |
| **S** | Backward | -X |
| **A** | Left | +Y |
| **D** | Right | -Y |
| **Q** | Up | +Z |
| **E** | Down | -Z |
| **J** | Yaw left | +Rz |
| **L** | Yaw right | -Rz |
| **I** | Pitch up | +Ry |
| **K** | Pitch down | -Ry |
| **U** | Roll left | +Rx |
| **O** | Roll right | -Rx |
| **F** | Toggle frame | base ↔ end-effector |
| **+/-** | Speed | increase/decrease |
| **SPACE** | Stop | all motion |
| **ESC** | Quit | exit teleop |

---

## Configuration

### Servo Parameters (servo_params.yaml)

```yaml
# Robot Configuration
move_group_name: "fr3_arm"
planning_frame: "fr3_link0"
ee_frame_name: "fr3_hand_tcp"
robot_link_command_frame: "fr3_link0"

# Command Type
command_in_type: "unitless"  # Joystick-style [-1, 1]

# Velocity Limits
scale:
  linear: 0.4    # Max 0.4 m/s
  rotational: 0.8  # Max 0.8 rad/s

# Safety
check_collisions: true
collision_check_rate: 10.0
override_velocity_scaling_factor: 0.3  # 30% of max joint velocity
```

### Key Parameters

- **move_group_name**: SRDF group being driven (`fr3_arm`)
- **planning_frame**: Base frame for commands (`fr3_link0`)
- **ee_frame_name**: End-effector frame (`fr3_hand_tcp`)
- **robot_link_command_frame**: Frame used to validate twist commands (`fr3_link0`)
- **command_in_type**: `"unitless"` means joystick-style input; `"speed_units"` means m/s
- **scale**: Maximum velocities reached when the input equals ±1.0

---

## Issues Fixed

Four configuration problems surfaced during setup. Each fix is recorded below so the failure mode stays documented.

### Problem 1: Wrong YAML Format
- Before: `/**:` / `ros__parameters:` wrapper
- After: Flat YAML wrapped in `{"moveit_servo": ...}` by launch file

### Problem 2: Invalid Move Group
- Before: Default `panda_arm` from MoveIt Servo
- After: Explicit `move_group_name: "fr3_arm"`

### Problem 3: Missing robot_link_command_frame
- Before: Default `panda_link0` causing crash
- After: Added `robot_link_command_frame: "fr3_link0"`

### Problem 4: Servo Not Starting
- Before: Removed start/stop service calls
- After: Restored `/servo_node/start_servo` service call in teleop

---

## VLA Integration

To drive the arm from a VLA model instead of the keyboard, publish to the same topic:

```python
from geometry_msgs.msg import TwistStamped

class VLANode(Node):
    def __init__(self):
        super().__init__('vla_node')
        self.twist_pub = self.create_publisher(
            TwistStamped,
            '/servo_node/delta_twist_cmds',
            10
        )

    def publish_vla_action(self, delta_pose):
        """
        delta_pose: [dx, dy, dz, drx, dry, drz] from VLA model
        """
        msg = TwistStamped()
        msg.header.stamp = self.get_clock().now().to_msg()
        msg.header.frame_id = 'fr3_link0'  # or 'fr3_hand_tcp'

        # Scale to safe velocities (unitless mode: -1 to 1)
        msg.twist.linear.x = delta_pose[0] * 0.5  # Scale appropriately
        msg.twist.linear.y = delta_pose[1] * 0.5
        msg.twist.linear.z = delta_pose[2] * 0.5
        msg.twist.angular.x = delta_pose[3] * 0.5
        msg.twist.angular.y = delta_pose[4] * 0.5
        msg.twist.angular.z = delta_pose[5] * 0.5

        self.twist_pub.publish(msg)
```

**Publishing requirements:**

- Publish at 10-30 Hz for smooth motion
- Stamp every message with the current time
- Scale values to the [-1, 1] range in unitless mode
- Start the Servo service before publishing

---

## System Architecture

```
┌─────────────────┐
│  Keyboard Node  │ (or VLA Node)
└────────┬────────┘
         │ TwistStamped
         │ /servo_node/delta_twist_cmds
         ↓
┌─────────────────┐
│  MoveIt Servo   │
│   (servo_node)  │
└────────┬────────┘
         │ JointTrajectory
         │ /arm_controller/joint_trajectory
         ↓
┌─────────────────┐
│ arm_controller  │
└────────┬────────┘
         │ Joint Commands
         ↓
┌─────────────────┐
│  Gazebo / Real  │
│    Hardware     │
└─────────────────┘
```

---

## Troubleshooting

### Robot Not Moving

1. Confirm Servo started by looking for "Servo started successfully!"
2. Check the command topic rate: `ros2 topic hz /servo_node/delta_twist_cmds` (expect about 10 Hz)
3. Check the Servo output: `ros2 topic hz /arm_controller/joint_trajectory`
4. Check the controllers: `ros2 control list_controllers` (`arm_controller` should be active)

### Servo Crashed

- Inspect the launch output for errors
- Verify all frames exist: `ros2 run tf2_ros tf2_echo world fr3_link0`
- Restart the launch file

### Invalid Link Error

- Make sure `robot_link_command_frame` matches a link in the robot URDF
- Use `fr3_link0` for the FR3 robot

---

## Reference

### MoveIt Servo Documentation
- Tutorial: https://moveit.picknik.ai/humble/doc/examples/realtime_servo/
- Parameters: https://docs.ros.org/en/humble/p/moveit_servo/

### Key Files
- Servo config: `src/franka_openvla/config/servo_params.yaml`
- Launch file: `src/franka_openvla/launch/fr3.launch.py`
- Teleop node: `src/franka_openvla/franka_openvla/keyboard_servo_teleop.py`

---

## Verification Checklist

- [x] Servo node launches without errors
- [x] Move group name is `fr3_arm`
- [x] `robot_link_command_frame` is `fr3_link0`
- [x] Keyboard teleop starts servo successfully
- [x] Robot moves in Gazebo when pressing W
- [x] All directions (W/S/A/D/Q/E) work
- [x] Rotation commands (J/L/I/K/U/O) work
- [x] SPACE stops motion immediately
- [x] Frame toggle (F) switches between base/ee
- [x] Speed control (+/-) adjusts velocity

---

## Next Steps

The MoveIt Servo setup is complete and verified. Suggested follow-ups:

1. Drive the FR3 robot with keyboard teleoperation
2. Connect the VLA model for autonomous control
3. Exercise delta pose commands for reactive motion
4. Keep the safety features enabled (collision checking, joint limits, singularity avoidance)

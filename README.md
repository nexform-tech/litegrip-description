# litegrip_description

ROS 2 description package for the **LiteGrip parallel gripper** — URDF/xacro model,
RViz visualization, Gazebo simulation, and ros2_control hardware interface definitions.

**English** · [简体中文](README.zh-CN.md)

| Property | Value |
| --- | --- |
| **ROS 2 distribution** | Humble Hawksbill |
| **Target platform** | Ubuntu 22.04 (x86_64) |
| **Package version** | 1.0.0 |
| **Model source** | SolidWorks URDF Exporter 1.6.0 (geometry and inertia unchanged) |

---

## Status

The table below states exactly what has been executed and verified, and what has
not — read it before depending on any part of this package.

| Capability | Status | Evidence |
| --- | --- | --- |
| URDF/xacro parses | ✅ Verified | `check_urdf` reports root `base_footprint` → `base_link` → master link + 2 mimic finger links |
| `colcon build` | ✅ Verified | Builds on ROS 2 Humble; see [known environment issue](litegrip_description/docs/troubleshooting.md#conda-python3-breaks-the-build) |
| `display.launch.py` | ✅ Verified | Launches, `stroke:=` override propagates into the URDF |
| `ros2_control.launch.py` | ✅ Verified | Both controllers reach `active`; one-value opening command round-trip confirmed |
| Joint geometry (stroke ↔ opening) | ⚠️ **Re-verification pending** | Finger origins and inner-face positions were confirmed via TF at the mesh-theoretical travel limit `0.0435`; the shipped limit is now the measured `0.042726`, so both endpoints need re-checking |
| `gazebo.launch.py` | ⚠️ **Not verified** | Gazebo is not installed in the verification environment; never executed |
| Parameters `effort`, `velocity` | ⚠️ **Placeholder** | SolidWorks export defaults, not measured. See [Calibration required](#calibration-required) |
| Parameters `joint_damping`, `joint_friction` | ⚠️ **Placeholder** | Conservative values chosen to damp simulation oscillation |

Verification was performed on ROS 2 Humble / Ubuntu 22.04 with `x86_64` on 2026-09-15.
Re-verify on your own platform before relying on any ✅ above.

---

## Overview

LiteGrip is a two-finger parallel gripper with **one transmission**: a single motor
drives both fingers apart along rails. The model states that fact explicitly. The only
commandable joint is the master `gripper_opening_joint`, a prismatic joint whose value is
the opening travel in metres; the two finger joints are its `<mimic>` followers:

```text
q_finger = -0.5 × opening + stroke
```

`opening = 0` is the measured closed position, `opening = 2 × stroke` is fully open.
MoveIt, the ros2_control block and the joint slider therefore expose one joint, not two.

This package is a pure description package. It contains no compiled code — `CMakeLists.txt`
only installs `urdf/`, `launch/`, `config/`, and `meshes/` into `share/litegrip_description/`.

```text
<repository root>/
├── README.md, README.zh-CN.md, CHANGELOG.md, LICENSE
├── .github/workflows/                    ci.yml, release.yml
└── litegrip_description/                 the ROS 2 package (colcon finds package.xml recursively)
    ├── package.xml
    ├── CMakeLists.txt
    ├── urdf/
    │   └── litegrip_description.urdf.xacro       Robot description — the single source of truth
    ├── launch/
    │   ├── display.launch.py                   RViz visualization
    │   ├── ros2_control.launch.py              ros2_control pipeline
    │   └── gazebo.launch.py                    Gazebo Classic simulation (⚠ unverified)
    ├── config/
    │   ├── litegrip_controllers.yaml           ros2_control controller configuration
    │   ├── joint_names_litegrip_description.yaml   Joint name list, for MoveIt Setup Assistant
    │   └── litegrip_description.rviz           RViz configuration
    ├── meshes/
    │   ├── base_link.STL                       ~925 KB
    │   ├── gripper_slider_link1.STL            (right finger)
    │   └── gripper_slider_link2.STL            (left finger)
    └── docs/
        ├── calibration-guide.md                Step-by-step calibration procedure
        ├── calibration-guide.zh-CN.md          Chinese translation
        ├── design-notes.md                     Design decisions, geometry derivation, revision history
        ├── design-notes.zh-CN.md               Chinese translation
        ├── troubleshooting.md                  Known pitfalls and their resolutions
        └── troubleshooting.zh-CN.md            Chinese translation
```

**Physical properties.** Total mass 0.5772 kg — `base_link` 0.5063 kg, each finger
0.03545 kg. Inertia tensors are carried over unmodified from the SolidWorks export.

---

## Requirements

- ROS 2 Humble Hawksbill on Ubuntu 22.04
- Python 3.10 (system interpreter — see [the Conda note](litegrip_description/docs/troubleshooting.md#conda-python3-breaks-the-build))

---

## Installation

Install dependencies:

```bash
# Description and visualization
sudo apt install ros-humble-robot-state-publisher ros-humble-joint-state-publisher \
                 ros-humble-joint-state-publisher-gui ros-humble-rviz2 ros-humble-xacro

# ros2_control (provides mock_components/GenericSystem via hardware_interface)
sudo apt install ros-humble-ros2-control ros-humble-ros2-controllers \
                 ros-humble-controller-manager ros-humble-joint-state-broadcaster \
                 ros-humble-position-controllers

# Optional: Gazebo Classic simulation
sudo apt install ros-humble-gazebo-ros-pkgs ros-humble-gazebo-ros2-control
```

Build the package:

```bash
mkdir -p ~/ros2_ws/src && cd ~/ros2_ws/src
ln -s /path/to/litegrip-description .          # or copy the directory
cd ~/ros2_ws
colcon build --packages-select litegrip_description
source install/setup.bash
```

> If `colcon build` fails with `ModuleNotFoundError: No module named 'catkin_pkg'`,
> a Conda interpreter is shadowing the system Python. See
> [docs/troubleshooting.md](litegrip_description/docs/troubleshooting.md#conda-python3-breaks-the-build) —
> this is the most common build failure for this package.

---

## Usage

### 1. RViz visualization

```bash
ros2 launch litegrip_description display.launch.py
```

Drag the `gripper_opening_joint` slider in the `joint_state_publisher_gui` window; both
fingers follow it through `<mimic>`. There is no independent slider for either finger.

| Argument | Default | Description |
| --- | --- | --- |
| `gui` | `true` | Start `joint_state_publisher_gui` (joint sliders) |
| `rviz` | `true` | Start RViz2 |
| `rvizconfig` | `config/litegrip_description.rviz` | RViz configuration file path |
| `stroke` | `0.042726` | Per-finger travel (m) — measured value |
| `effort` | `10.0` | Joint maximum effort (N) |
| `velocity` | `0.2` | Joint maximum velocity (m/s) |

Examples:

```bash
ros2 launch litegrip_description display.launch.py gui:=false       # no slider window
ros2 launch litegrip_description display.launch.py stroke:=0.02     # override joint travel
```

> ⚠️ **`gui:=false` removes the movable links from TF.** `joint_state_publisher_gui`
> is the only node in this launch that publishes `/joint_states`, so with `gui:=false`
> the opening joint has no state, `robot_state_publisher` cannot compute it, and the
> mimic fingers cannot be derived either. Only the fixed transform
> `base_footprint → base_link` remains. If you need the finger frames without the GUI,
> start the headless publisher too:
>
> ```bash
> ros2 run joint_state_publisher joint_state_publisher
> ```

The RViz configuration uses `base_footprint` as its fixed frame.

### 2. ros2_control pipeline

```bash
ros2 launch litegrip_description ros2_control.launch.py
```

| Argument | Default | Description |
| --- | --- | --- |
| `hardware_plugin` | `mock_components/GenericSystem` | ros2_control hardware interface plugin |
| `stroke` | `0.042726` | Per-finger travel (m) — measured value |
| `effort` | `10.0` | Joint maximum effort (N) |
| `velocity` | `0.2` | Joint maximum velocity (m/s) |

The default hardware plugin is `mock_components/GenericSystem`, which echoes each
position command straight back as state. The gripper does not physically move in this
mode, but `controller_manager`, controller loading, resource claiming, and the full
command/state interface pipeline behave exactly as they do with real hardware — which
makes it the right way to validate an integration before a device is available.

Resulting graph:

| Node | Role |
| --- | --- |
| `/robot_state_publisher` | Publishes TF from `/joint_states` |
| `/controller_manager` | Loads and manages controllers |
| `/joint_state_broadcaster` | Publishes `/joint_states` |
| `/gripper_controller` | `position_controllers/JointGroupPositionController` |

Verify the pipeline:

```bash
ros2 control list_controllers
# Both joint_state_broadcaster and gripper_controller must report [active].

ros2 control list_hardware_interfaces
# 1 command interface (claimed) + 2 state interfaces = 3 total.

# Command the opening (metres, 0 = closed, 0.085452 = fully open)
ros2 topic pub --once /gripper_controller/commands \
  std_msgs/msg/Float64MultiArray "{data: [0.042726]}"

ros2 topic echo /joint_states
```

> ⚠️ **The mock hardware does not clamp out-of-range commands.** In the configuration
> above, `[0.1]` exceeds `2 * stroke = 0.085452` and is echoed back as `0.1` without
> error or clamping, despite the `min`/`max` parameters declared on the command
> interface. The travel limits are descriptive metadata in this configuration, not an
> enforced constraint. Clamp commands in your own application layer.
>
> ℹ️ `/joint_states` reports `effort: nan` for the master joint. The mock hardware
> declares no effort state interface, so no value exists to publish. This is expected
> and not an error.

### 3. Gazebo simulation

> ⚠️ **This launch file has never been executed.** Gazebo is not installed in the
> environment where this package was verified. Treat the first run as a debugging
> session and confirm that the model spawns correctly and both controllers activate.

```bash
ros2 launch litegrip_description gazebo.launch.py
```

This launch forces `hardware_plugin:=gazebo_ros2_control/GazeboSystem` and starts Gazebo,
spawns the model, then activates `joint_state_broadcaster` followed by `gripper_controller`.

> ⚠️ **Gazebo Classic does not preserve `<mimic>` when converting URDF to SDF.** The
> master joint and the two finger joints will load as three independent joints, so the
> fingers will not follow the opening joint. This is a limitation of that simulator, not
> of the model. Use MuJoCo, PyBullet, or a simulator that supports mimic constraints for
> physics work; use the ROS 2 side for control-path work.

---

## Interfaces

### Joints

| Joint | Type | Axis | Origin in `base_link` | Limit | Role |
| --- | --- | --- | --- | --- | --- |
| `gripper_opening_joint` | prismatic | `+X` | `(0, 0, 0)` | `[0, 0.085452]` m | master; the only commandable joint |
| `gripper_slide_joint_right` | prismatic | `+X` | `(-0.067, 0.0010833, 0.034083)` | `[0, 0.042726]` m | `<mimic>` of `gripper_opening_joint` |
| `gripper_slide_joint_left` | prismatic | `-X` | `( 0.067, -0.0010833, 0.034083)` | `[0, 0.042726]` m | `<mimic>` of `gripper_opening_joint` |

Both fingers use `multiplier="-0.5"` and `offset="stroke"`, so
`q_finger = -0.5 × opening + stroke`. The master's upper limit is `2 × stroke`; the finger
limits are `stroke`.

### Links

| Link | Parent | Joint | Mass |
| --- | --- | --- | --- |
| `base_footprint` | — | root, no inertia | 0 |
| `base_link` | `base_footprint` | `base_footprint_joint` (fixed) | 0.5063 kg |
| `gripper_opening_link` | `base_link` | `gripper_opening_joint` | 0 (geometry-free dummy) |
| `gripper_slider_link1` | `base_link` | `gripper_slide_joint_right` | 0.03545 kg |
| `gripper_slider_link2` | `base_link` | `gripper_slide_joint_left` | 0.03545 kg |

`base_footprint` is an inertia-free dummy root coincident with `base_link`. It exists to
give the TF tree a valid root: KDL does not support a root link carrying inertia, and
would emit a `root link has an inertia` warning and silently ignore `base_link`'s inertia.
**Attach the gripper to a robot arm at `base_footprint`.**

### ros2_control interfaces

| Joint | Command interfaces | State interfaces |
| --- | --- | --- |
| `gripper_opening_joint` | `position` | `position`, `velocity` |

**3 interfaces in total per copy of the model** (1 command + 2 state). The two fingers are
`<mimic>` and deliberately have no interfaces: a real gripper has one transmission.

### Controller

`position_controllers/JointGroupPositionController` on `/gripper_controller`.

The command array has exactly one entry:

| Index | Joint | Meaning |
| --- | --- | --- |
| `[0]` | `gripper_opening_joint` | Opening travel in metres; `0` closed, `0.085452` fully open |

---

## ⚠️ Opening convention — read before writing a controller

There is one commandable joint, `gripper_opening_joint`, and its value is the **opening
travel** in metres:

| Opening | Result |
| --- | --- |
| `0.0` | Measured closed position — fingers at `q = stroke`; this gripper calipers at a **1.508 mm** gap |
| `0.042726` | Half open |
| `0.085452` | Fully open — fingers at `q = 0`; model inner faces at `x = ∓0.0435`, model opening **87 mm**; this gripper calipers at **86.960 mm** |

The two finger joints share the same `<mimic>` multiplier (`-0.5`) even though their axes
point opposite ways. This is deliberate: `q_finger` is a displacement along each joint's
own axis, so the same multiplier makes them open and close together. Do not send independent
finger commands — neither finger has a command interface. Writing a controller that expects
two values, or that negates one finger against the other, describes a mechanism this
gripper does not have.

### Why the travel limit is 0.042726 and not 0.067

Two different numbers are in play here. Do not conflate them.

**Mesh-theoretical closure `0.0435`** is derived from the CAD geometry:

```text
gripper_slider_link1.STL X range = [-0.00095, +0.0235]   (bounding box, measured)
joint origin x                   = -0.067
=> stroke = 0.067 - 0.0235       =  0.0435
```

That is the joint value at which the two inner faces would meet exactly at `x = 0`, i.e.
zero gap. It is a property of the CAD geometry, not of the physical unit.

**Measured per-finger travel `0.042726 m`** is the value this package ships as the
`stroke` default. Caliper measurement of this gripper gives a total mechanical stroke of
`85.452 mm`, i.e. `42.726 mm` per finger, and a closed clearance of `1.508 mm`. Because
the real gripper does not reach the ideal zero-gap closure, the measured value is smaller
— the conservative direction.

At `stroke = 0.067` the two fingers would fully overlap and interpenetrate. The
SolidWorks CSV declares `Limit Upper = 0.067` for the right finger and `-0.067` for the
left; both are unusable as exported. Full derivation in
[docs/design-notes.md](litegrip_description/docs/design-notes.md#gripper-geometry).

> ⚠️ Changing the limit to `0.042726` changes both travel endpoints of the model. The
> mesh and TF checks recorded below were performed at the mesh-theoretical `0.0435`, so
> the model's new endpoints still need a runtime mesh/TF re-verification before they can
> be called geometrically verified.

---

## Configuration

`urdf/litegrip_description.urdf.xacro` is parameterized by xacro arguments:

| Argument | Default | Description |
| --- | --- | --- |
| `stroke` | `0.042726` | Per-finger travel (m) — the joint limit `upper`; measured value |
| `effort` | `10.0` | Maximum joint effort (N) ⚠ placeholder |
| `velocity` | `0.2` | Maximum joint velocity (m/s) ⚠ placeholder |
| `joint_damping` | `0.05` | Joint damping ⚠ placeholder |
| `joint_friction` | `0.02` | Joint friction ⚠ placeholder |
| `hardware_plugin` | `mock_components/GenericSystem` | ros2_control hardware plugin |
| `include_mock_ros2_control` | `true` | Emit this package's mock `<ros2_control>` block; set `false` when a composition file adds a real hardware component for the same joint |

**Overriding arguments.** `stroke`, `effort`, and `velocity` are exposed as launch
arguments by both `display.launch.py` and `ros2_control.launch.py`, so they can be set on
the command line. `joint_damping` and `joint_friction` are **not** exposed by any launch
file — to change them you must either edit the xacro defaults or invoke `xacro` directly.

Expand to plain URDF for tools that do not support xacro:

```bash
xacro $(ros2 pkg prefix litegrip_description)/share/litegrip_description/urdf/litegrip_description.urdf.xacro \
  -o /tmp/litegrip_description.urdf
```

### Calibration required

`effort = 10.0` N and `velocity = 0.2` m/s are **SolidWorks export defaults, not measured
values**. Until they are replaced with calibrated figures, any torque or velocity limiting
derived from `<limit>` is incorrect. `joint_damping` and `joint_friction` are conservative
initial values chosen only to damp simulation oscillation and should likewise be
calibrated against the real mechanism.

→ **[Calibration Guide](litegrip_description/docs/calibration-guide.md)** — the complete step-by-step procedure
for measuring these values, from the physical gripper through the Python SDK to the xacro
arguments above.

---

## Connecting real hardware

Substitute your vendor's ros2_control hardware interface plugin for the mock:

```bash
ros2 launch litegrip_description ros2_control.launch.py hardware_plugin:=<vendor_plugin_name>
```

Your plugin must provide, for the single master joint `gripper_opening_joint`:

- a `position` **command** interface
- `position` and `velocity` **state** interfaces

If your deployment adds its own `<ros2_control>` block for that joint, pass
`include_mock_ros2_control:=false`; two blocks declaring the same joint make
ros2_control refuse to load. This package ships no vendor plugin. If your hardware exposes
`velocity` or `effort` commands instead of `position`, the `<ros2_control>` block in the
xacro and the controller type in `config/litegrip_controllers.yaml` both need to change.

---

## Pre-release checklist

The following are **incomplete** and must be resolved before this package is published:

1. **Maintainer identity** — `package.xml` still carries `<maintainer email="TODO@example.com">LiteGrip</maintainer>`.
   Replace it with the real, legally correct entity and contact address.
2. **License undecided** — no license has been chosen for this package. `package.xml`
   carries `<license>TODO</license>` only because ROS 2 rejects a package without a
   license tag; it is not a legal statement.
3. **URLs** — `package.xml` `<url>` entries point at `https://example.com/litegrip`.
4. **`effort` / `velocity` calibration** — see [Calibration required](#calibration-required).
5. **`joint_damping` / `joint_friction` calibration** — see [Calibration required](#calibration-required).
6. **Gazebo launch validation** — `gazebo.launch.py` has never been run.

## Known limitations

- **Collision geometry reuses the high-precision visual STL.** `base_link.STL` is ~925 KB.
  With only 3 links the collision-checking cost is acceptable, but for
  performance-sensitive integration replace the `<collision>` geometry with simplified
  primitives (box/cylinder).
- **The mock pipeline uses a topic, not an action.** The default
  `JointGroupPositionController` accepts a one-element joint array over a topic. The model
  now has exactly one commandable joint, so `position_controllers/GripperActionController`
  also fits; that is what `litegrip_ros2_control` uses on real hardware. This package keeps
  the group controller for the standalone mock so the command path stays consistent with
  the arm-joint convention.
- **No `.stl` → simplification, no collision padding, no `<safety_controller>` limits,
  no transmissions.** The original SolidWorks export defined none of these.

## Documentation

| Document | Contents |
| --- | --- |
| [docs/calibration-guide.md](litegrip_description/docs/calibration-guide.md) | Step-by-step procedure to replace every placeholder value with a measured one — stroke, angle range, force, speed, damping, friction |
| [docs/design-notes.md](litegrip_description/docs/design-notes.md) | Geometry derivation, joint conventions, revision history against the SolidWorks export, rationale for design decisions |
| [docs/troubleshooting.md](litegrip_description/docs/troubleshooting.md) | Known failure modes and their resolutions, including the Conda build failure and the `controller_manager` naming pitfall |
| [CHANGELOG.md](CHANGELOG.md) | Version history |

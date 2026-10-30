# Design Notes

Why `litegrip_description` is built the way it is: the derivation of its geometry, the
conventions a consumer must respect, and the complete record of what was changed
relative to the original SolidWorks export.

**English** · [简体中文](design-notes.zh-CN.md)

---

## Source lineage

The URDF was not written by hand. It descends from a SolidWorks URDF Exporter run, and
three intermediate files exist in the original export. Understanding which file descends
from which matters, because the commonly circulated "original" values come from different
stages of that chain.

| Stage | File | State |
| --- | --- | --- |
| 1 | `夹爪urdf2.csv` | SolidWorks source of truth. Both joints are named `gripper_slide_joint` — **duplicate, illegal**. Travel limits `0.067` (right) and `-0.067` (left). |
| 2 | `夹爪urdf改前.urdf` | Joint names still duplicated. Limits `0.067` / `-0.067`. **Fails to parse:** `joint 'gripper_slide_joint' is not unique`. |
| 3 | `夹爪urdf2 (1).urdf` | Names split into `_right` / `_left`. Limits both written as `0.043.5` — a typo with a second decimal point. **Fails to parse:** `upper value (0.043.5) is not a valid float`. |
| 4 | `litegrip_description` (this package) | Parses cleanly. Limits parametrized, defaulting to the measured value `0.042726`. |

Both parse failures were confirmed by running `check_urdf` against the respective files.
Note that stage 3 already attempted to correct the travel limit — but the intermediate
value `0.043.5` is unparseable, so neither intermediate file is usable as a reference.
**The mesh geometry and inertia numbers in this package are byte-identical to the
SolidWorks export; only the defects listed below were repaired.** The one substantive
exception is the travel limit, which now carries this gripper's measured calibration value
rather than a value derived from the CAD geometry — see
[Deriving the travel limit](#deriving-the-travel-limit).

---

## Gripper geometry

### Coordinate frame

Both fingers slide along the X axis of `base_link`. The joint origins are mirrored:

| Joint | Origin in `base_link` | Axis |
| --- | --- | --- |
| `gripper_slide_joint_right` | `(-0.067, 0.0010833, 0.034083)` | `+X` |
| `gripper_slide_joint_left` | `( 0.067, -0.0010833, 0.034083)` | `-X` |

### Deriving the travel limit

The SolidWorks export declares `Limit Upper = 0.067` for the right finger. That value is
wrong and would drive the fingers through each other. The correct travel is derived from
the finger mesh bounding box.

Measured from `meshes/gripper_slider_link1.STL` (binary STL, all triangle vertices):

```text
X range = [-0.00095, +0.0235]
```

The right finger's joint origin sits at `x = -0.067`, so at joint value `q` the mesh
occupies:

```text
x_min(q) = -0.067 + q - 0.00095
x_max(q) = -0.067 + q + 0.0235      <- inner face of the right finger
```

The gripper closes when the two inner faces meet at `x = 0`. Solving `x_max(q) = 0`:

```text
q = 0.067 - 0.0235 = 0.0435
```

This gives the **mesh-theoretical closure** `0.0435` m — the joint value at which the two
inner faces would meet exactly at `x = 0`. The left finger is the mirror image and shares
the same value.

Measuring this physical unit separately gives a different, smaller number: total
mechanical stroke `85.452 mm`, per-finger travel `0.042726 m`, closed clearance
`1.508 mm`, open opening `86.960 mm`. **The xacro limit therefore ships the measured
`0.042726 m`**, and the mesh/TF relationship at the new endpoints still needs
re-verification in RViz/Gazebo. `calibrate_guided` completed three runs: closed angle
`0.041380–0.041400 rad`, open angle `-1.272790 rad` in every run, angular range
`1.314170–1.314190 rad`, `rad_to_mm` `65.0226–65.0236 mm/rad`. Five endpoint rechecks gave
a mechanical stroke of `85.43–85.47 mm`, mean `85.452 mm`, with corresponding `rad_to_mm`
of `65.0053–65.0372 mm/rad`; the final calibration value is `65.0229 mm/rad`.

**If `upper` were left at `0.067`, the fingers would overlap by 0.0235 m and
interpenetrate completely.** Do not revert this value.

### Behaviour at both travel limits

The model endpoints below come from `tf2_echo base_footprint <link>` against a running
`robot_state_publisher`; the measured column is caliper reading on this gripper. The
updated limit still needs runtime re-checking.

| Opening (`gripper_opening_joint`) | Finger `q` | Finger origins | Inner faces | Opening |
| --- | --- | --- | --- | --- |
| `0.085452` (fully open) | `0.0` | `x = ∓0.067` | model `x = ∓0.0435` | model **87 mm**; measured **86.960 mm** |
| `0.0` (closed) | `0.042726` | measured closed position | measured closed position | measured clearance **1.508 mm** |

The values in the "Finger origins" column match the joint origins in the URDF exactly at
`q = 0`, which confirms the joint origins and the mesh bounding box are consistent.

Because the measured closed clearance is not an ideal mesh contact value, the updated
`0.042726` limit still requires a runtime mesh/TF check — it cannot yet be called
geometrically verified.

Caliper readings of commanded positions were recorded as well: `0 mm` gave
`0.12, 0.04, 0.16, 0.06, 0.12 mm`; `20 mm` gave `20.40, 20.04, 20.08, 19.98, 18.80 mm`;
`40 mm` gave `40.02, 40.20, 40.22, 39.86, 39.90 mm`; `60 mm` gave
`60.22, 59.86, 60.02, 60.06, 59.98 mm`; `80 mm` gave `80.06, 80.10, 79.88, 80.02,
79.96 mm`; `85.452 mm` gave `85.24, 85.52, 85.60, 85.40, 85.80 mm`. The `18.80 mm`
reading at the `20 mm` position is a clear outlier; the maximum absolute error over the
raw readings is `1.20 mm`, so position accuracy has not passed final acceptance and that
point needs a re-test. Fifty cycles over `2 mm ↔ 83 mm` were completed: the closed-end
opening drift from cycle 1 to cycle 50 was `+0.01 mm`, the open-end opening drift
`+0.02 mm`, and both endpoint angle drifts were `+0.000009 rad`. That repeatability item
is quantitatively complete.

---

## Opening convention

**This is the single most common source of integration bugs with this model.**

The real gripper has one motor and one transmission, so the model exposes **one**
commandable joint. `gripper_opening_joint` is a prismatic joint whose value is the opening
travel in metres; both finger joints are `<mimic>` followers:

```text
q_finger = -0.5 × opening + stroke
```

| Command | Result |
| --- | --- |
| `0.0` | Measured closed calibration position (fingers at `q = stroke`) |
| `0.042726` | Half open |
| `0.085452` | Fully open (fingers at `q = 0`) |

The finger axes still point opposite ways (`+X` and `-X`, straight from the SolidWorks
CSV: `Joint Axis X = 1` for one finger, `-1` for the other). Both mimics carry the same
`multiplier="-0.5"` **because** the axes are opposite: `q_finger` is a displacement along
each joint's own axis, so a common multiplier moves the fingers symmetrically. Do not send
independent finger commands, and do not expect a two-element command array — neither finger
has a command interface.

---

## Why `base_footprint` exists

The SolidWorks export declared `base_link` as the root link, and `base_link` carries
inertia. KDL — and therefore MoveIt and most TF consumers — **does not support a root link
with inertia**: it emits `root link has an inertia` and silently discards that inertia.

The fix is a massless, inertia-free `base_footprint` link, coincident with `base_link`
and connected by a fixed joint. The TF tree then has a legal root, and `base_link`'s
inertia is preserved as a non-root link.

Two consequences:

1. **Attach the gripper to a robot arm at `base_footprint`**, not `base_link`.
2. **Do not also publish a static transform between them.** The original ROS 1
   `gazebo.launch` published `tf_footprint_base` with the arguments
   `0 0 0 0 0 0 base_link base_footprint` — note that `tf2`'s `static_transform_publisher`
   takes `frame_id child_frame_id` in that order, so this declared `base_link` as the
   *parent* of `base_footprint`. Since the URDF already defines `base_footprint` as the
   root with `base_link` as its child, publishing it would give the TF tree two parents
   for the same pair of frames. That node was removed.

The RViz configuration uses `base_footprint` as its fixed frame.

---

## Revision history against the SolidWorks export

Everything below is a repair. **No geometric or inertial value was altered** — with one
deliberate exception: the travel limit now carries this gripper's measured value (items 1
and 3). That is a calibration decision, not a geometry edit.

| # | Revision | Reason |
| --- | --- | --- |
| 1 | `<limit upper="0.043.5">` → parametrized measured value `0.042726` | `0.043.5` is not a valid float; the export did not parse at all |
| 2 | Joint `gripper_slide_joint` → `gripper_slide_joint_right` / `_left` | Both joints carried the same name, which is illegal |
| 3 | Left finger `upper` `-0.067` → measured `0.042726` | Negative upper bound with `lower = 0` is an invalid range |
| 4 | Added inertia-free root `base_footprint` | Removes the KDL `root link has an inertia` warning; see above |
| 5 | Filled in empty material names (`name=""`) | Empty names are invalid; now `litegrip_base` and `litegrip_finger` |
| 6 | Added `<dynamics damping friction>` | The export defined none; needed to damp simulation oscillation |
| 7 | Added the `<ros2_control>` block | Required for any ros2_control integration |
| 8 | Package renamed `夹爪urdf2` → `litegrip_urdf` | ROS 2 package names must be lowercase alphanumeric/underscore |
| 9 | `config/joint_names_*.yaml` corrected | The export contained an empty-string placeholder and a duplicated entry: `['', 'gripper_slide_joint', 'gripper_slide_joint', ]` |
| 10 | Dropped the extra `base_link → base_footprint` static transform | Direction was reversed and it would conflict with the URDF's root link; see above |
| 11 | Completed `package.xml` dependencies | The export omitted `gazebo_ros`, `tf`, `rostopic`, and `joint_state_publisher` |
| 12 | Added master `gripper_opening_joint`; both fingers now `<mimic>` it | The real gripper has one motor and one transmission. Two independently commandable joints described hardware that does not exist, and made the MoveIt planning group reference a joint the URDF did not provide |

Items 1–3 were each confirmed by attempting to parse the corresponding source file; see
[Source lineage](#source-lineage).

---

## Mass properties

Carried over unmodified from the SolidWorks export. Total mass **0.5772 kg**.

| Link | Mass | ixx | iyy | izz |
| --- | --- | --- | --- | --- |
| `base_link` | 0.506299 kg | 1.40520e-04 | 3.08493e-04 | 3.45331e-04 |
| `gripper_slider_link1` | 0.0354537 kg | 1.73251e-05 | 1.26496e-05 | 6.96030e-06 |
| `gripper_slider_link2` | 0.0354537 kg | 1.73251e-05 | 1.26496e-05 | 6.96030e-06 |

The two fingers are mirror images: their inertia tensors are identical in `ixx`, `iyy`,
`izz`, and their products of inertia differ in sign on `ixz` and `iyz`, as expected.

Full tensors, including centres of mass and products of inertia, are in
[`urdf/litegrip_description.urdf.xacro`](../urdf/litegrip_description.urdf.xacro).

---

## Design decisions

### Single-actuator interface

The real gripper has one motor and one transmission; the fingers were never independently
controllable. The model now says so. `gripper_opening_joint` (prismatic, carried by the
geometry-free dummy link `gripper_opening_link`) is the only commandable joint, and both
finger joints are its `<mimic>` followers:

```text
q_finger = -0.5 × opening + stroke
```

`opening = 0` is the measured closed stop; `opening = 2 × stroke` is fully open. A MoveIt
planning group, the `<ros2_control>` block and a joint slider all expose one joint.

`<mimic>` is honoured by `robot_state_publisher` and MoveIt, which is why the derived finger
positions appear in TF and in the planning model without any extra wiring. **ros2_control
does not evaluate `<mimic>` itself** — it acts only on the interfaces the hardware
component declares. That is why the finger joints have no command or state interface in
the ros2_control block: the single master joint is the whole interface.

### Why the mock uses `JointGroupPositionController`

With one commandable joint, `position_controllers/GripperActionController` now fits
exactly — it accepts a single joint, and it expresses "close until stalled" directly. The
real hardware stack (`litegrip_ros2_control`) uses it for precisely that reason.

This package keeps `JointGroupPositionController` for its standalone mock: it takes a
one-element joint array over a topic, which keeps the command path identical to the
arm-joint convention used elsewhere, and it does not depend on the `GripperActionController`
parameters (goal tolerance, stalling). The trade-off is a `Float64MultiArray` over a topic
instead of an action. Switching the mock to the action controller is a supported change if
you prefer the action interface.

### Why collision geometry reuses the visual STL

The export points `<collision>` at the same meshes as `<visual>`. With three links the
cost is acceptable, and it guarantees that collision bounds match the rendered geometry
exactly. `base_link.STL` is ~925 KB, so if this model is integrated into a
performance-sensitive scene (many grippers, or a high-rate collision checker), replace the
`<collision>` meshes with primitives.

### What this package deliberately does not contain

No meshes were simplified, no collision padding was added, no `<safety_controller>` limits
were set, and no transmissions were defined. The SolidWorks export defined none of these,
and inventing values would have made the package appear more specified than it is. The
`effort` and `velocity` limits are export defaults and are flagged as uncalibrated in both
the xacro and the README.

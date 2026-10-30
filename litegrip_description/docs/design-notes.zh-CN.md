# 设计说明

`litegrip_description` 为何如此构建：几何参数的推导过程、使用方必须遵守的约定，
以及相对于原始 SolidWorks 导出文件的完整修订记录。

[English](design-notes.md) · **简体中文**

---

## 来源沿革

本 URDF 并非手写，而是源自一次 SolidWorks URDF Exporter 导出；原始导出中共存在三份
中间文件。厘清哪份文件派生自哪份很重要 —— 因为流传的所谓「原始值」来自这条链上
不同的阶段。

| 阶段 | 文件 | 状态 |
| --- | --- | --- |
| 1 | `夹爪urdf2.csv` | SolidWorks 的事实源。两个关节同名 `gripper_slide_joint` —— **重名，非法**。行程限位分别为 `0.067`（右）与 `-0.067`（左）。 |
| 2 | `夹爪urdf改前.urdf` | 关节名仍重名，限位 `0.067` / `-0.067`。**无法解析**：`joint 'gripper_slide_joint' is not unique`。 |
| 3 | `夹爪urdf2 (1).urdf` | 关节名已拆为 `_right` / `_left`。限位均写作 `0.043.5` —— 多了一个小数点的笔误。**无法解析**：`upper value (0.043.5) is not a valid float`。 |
| 4 | `litegrip_description`（本包） | 可正常解析，限位参数化为实测值 `0.042726`。 |

两处解析失败均已通过 `check_urdf` 实测确认。注意阶段 3 已经尝试修正行程限位 ——
但中间值 `0.043.5` 无法解析，因此两份中间文件都不能作为参照。
**本包沿用 SolidWorks 导出的 mesh 几何与惯性数值；除行程限位采用本机实测标定值外，其余几何与惯性数值未改。**

---

## 夹爪几何

### 坐标系

两指均沿 `base_link` 的 X 轴滑动，关节原点互为镜像：

| 关节 | 关节原点（`base_link` 系） | axis |
| --- | --- | --- |
| `gripper_slide_joint_right` | `(-0.067, 0.0010833, 0.034083)` | `+X` |
| `gripper_slide_joint_left` | `( 0.067, -0.0010833, 0.034083)` | `-X` |

### 行程限位的推导

SolidWorks 导出中右指标注 `Limit Upper = 0.067`。该值有误，会导致两指互穿。
正确的行程由手指 mesh 的包围盒推导得出。

从 `meshes/gripper_slider_link1.STL`（二进制 STL，遍历全部三角形顶点）实测：

```text
X 范围 = [-0.00095, +0.0235]
```

右指的关节原点位于 `x = -0.067`，故在关节值 `q` 处 mesh 占据：

```text
x_min(q) = -0.067 + q - 0.00095
x_max(q) = -0.067 + q + 0.0235      <- 右指内侧面
```

仅按 mesh 理论模型，两指内侧面在 `x = 0` 处贴合即完成闭合。解 `x_max(q) = 0`：

```text
q = 0.067 - 0.0235 = 0.0435
```

这得到 mesh 理论行程 `0.0435` m。随后对本机实物单独测量得到：总机械行程
`85.452 mm`、单指行程 `0.042726 m`、闭合间隙 `1.508 mm`、张开开口
`86.960 mm`。因此 xacro 限位采用实测值 `0.042726` m，并需要通过 RViz/Gazebo
重新验证 mesh 与 TF。`calibrate_guided` 已完成 3 次数据采集：闭合角度
`0.041380–0.041400 rad`、张开角度均为 `-1.272790 rad`、角度行程
`1.314170–1.314190 rad`、`rad_to_mm` 为 `65.0226–65.0236 mm/rad`。
张开端复测 5 次得到机械行程 `85.43–85.47 mm`，均值为 `85.452 mm`；对应
`rad_to_mm` 为 `65.0053–65.0372 mm/rad`，最终标定值采用 `65.0229 mm/rad`。

**若将 `upper` 保留为 `0.067`，两指将重叠 0.0235 m，完全互穿。**
请勿改回该值。

### 行程两端的实测行为

以下将 URDF 模型端点与本机卡尺实测值对照；更新后的限位仍需在运行时复核：

| 开口（`gripper_opening_joint`） | 手指 `q` | 手指原点 | 内侧面 | 开口 |
| --- | --- | --- | --- | --- |
| `0.085452`（全开） | `0.0` | `x = ∓0.067` | 理论 `x = ∓0.0435` | 实测 **86.960 mm** |
| `0.0`（闭合） | `0.042726` | 实测闭合位置 | 实测闭合位置 | 实测间隙 **1.508 mm** |

`q = 0` 时的手指原点可用于检查模型。由于实测闭合间隙并非理想 mesh 接触值，
更新后的 `0.042726` 限位仍需运行时做 mesh/TF 验证，暂不能直接称为几何验证完成。

位置指令的卡尺读数已记录：`0 mm` 为 `0.12、0.04、0.16、0.06、0.12 mm`；
`20 mm` 为 `20.40、20.04、20.08、19.98、18.80 mm`；`40 mm` 为
`40.02、40.20、40.22、39.86、39.90 mm`；`60 mm` 为
`60.22、59.86、60.02、60.06、59.98 mm`；`80 mm` 为
`80.06、80.10、79.88、80.02、79.96 mm`；`85.452 mm` 为
`85.24、85.52、85.60、85.40、85.80 mm`。其中 `20 mm` 位置的 `18.80 mm`
为明显离群读数；按原始读数计算的最大绝对误差为 `1.20 mm`，因此位置绝对误差
尚未完成最终验收，需复测该点。50 次循环（`2 mm ↔ 83 mm`）已完成：第 1 次到
第 50 次闭合端开口漂移 `+0.01 mm`、张开端开口漂移 `+0.02 mm`，闭合角度和
张开角度漂移均为 `+0.000009 rad`；该项重复性定量数据已完成。

---

## 开口约定

**这是本模型最常见的集成错误来源。**

真机只有一台电机、一套传动，因此模型只暴露**一个**可命令关节。
`gripper_opening_joint` 是 prismatic 关节，取值是开口行程，单位米；两个手指关节都是
它的 `<mimic>` 跟随关节：

```text
q_finger = -0.5 × opening + stroke
```

| 指令 | 结果 |
| --- | --- |
| `0.0` | 实测闭合标定位置（两指位于 `q = stroke`） |
| `0.042726` | 半开 |
| `0.085452` | 完全张开（两指位于 `q = 0`） |

手指关节的 axis 仍然相反（`+X` 与 `-X`，直接来自 SolidWorks CSV：一指标注
`Joint Axis X = 1`、另一指标注 `-1`）。两个 mimic 使用**相同**的
`multiplier="-0.5"`，恰恰是因为 axis 相反：`q_finger` 是沿各自关节 axis 的位移，
同一个 multiplier 才能让两指对称运动。不要给两指下发独立指令，也不要期待二元指令
数组 —— 两指都没有命令接口。

---

## 为何存在 `base_footprint`

SolidWorks 导出将 `base_link` 声明为根节点，而 `base_link` 是带惯性的。
KDL —— 以及依赖它的 MoveIt 和大多数 TF 使用方 —— **不支持带惯性的 root link**：
它会给出 `root link has an inertia` 告警，并静默丢弃该惯性。

解决办法是引入一个无质量、无惯性的 `base_footprint` link，与 `base_link` 位姿重合，
以 fixed 关节相连。这样 TF 树就具有合法树根，而 `base_link` 的惯性作为非根节点得以保留。

由此带来两个后果：

1. **将夹爪安装到机械臂时，请以 `base_footprint` 为附着基准**，而非 `base_link`。
2. **不要再额外发布二者之间的静态变换。** 原始 ROS 1 的 `gazebo.launch` 发布了一个
   `tf_footprint_base` 节点，参数为 `0 0 0 0 0 0 base_link base_footprint` —— 注意
   `tf2` 的 `static_transform_publisher` 参数顺序是 `frame_id child_frame_id`，
   因此该行实际是声明 `base_link` 为 `base_footprint` 的**父节点**。而 URDF 中已定义
   `base_footprint` 为根、`base_link` 为其子节点，再发布一次会让同一对 frame 在 TF
   树中出现两个父节点。该节点已被移除。

RViz 配置使用的固定坐标系即为 `base_footprint`。

---

## 相对于 SolidWorks 导出的修订记录

以下全部为修复。**未改动任何几何或惯性数值。**

| # | 修订内容 | 原因 |
| --- | --- | --- |
| 1 | `<limit upper="0.043.5">` → 参数化实测值 `0.042726` | `0.043.5` 不是合法浮点数，原文件完全无法解析 |
| 2 | 关节 `gripper_slide_joint` → `gripper_slide_joint_right` / `_left` | 两关节同名，非法 |
| 3 | 左指 `upper` 由 `-0.067` → 实测值 `0.042726` | `lower = 0` 时 upper 为负值，区间非法 |
| 4 | 新增无惯性根节点 `base_footprint` | 消除 KDL `root link has an inertia` 告警，见上文 |
| 5 | 补全空的材质名称（原为 `name=""`） | 空名非法；现为 `litegrip_base` 与 `litegrip_finger` |
| 6 | 补充 `<dynamics damping friction>` | 原导出未定义；抑制仿真振荡所必需 |
| 7 | 补充 `<ros2_control>` 块 | 任何 ros2_control 集成的前提 |
| 8 | 包名 `夹爪urdf2` → `litegrip_urdf` | ROS 2 要求包名为小写字母/数字/下划线 |
| 9 | 修正 `config/joint_names_*.yaml` | 原文件含空串占位与重复项：`['', 'gripper_slide_joint', 'gripper_slide_joint', ]` |
| 10 | 移除多余的 `base_link → base_footprint` 静态变换 | 方向写反，且会与 URDF 的根节点定义冲突，见上文 |
| 11 | 补全 `package.xml` 依赖声明 | 原导出缺 `gazebo_ros`、`tf`、`rostopic`、`joint_state_publisher` |
| 12 | 新增总控关节 `gripper_opening_joint`，两指改为 `<mimic>` 跟随 | 真机只有一台电机、一套传动。两个可独立命令的关节描述的是不存在的硬件，也让 MoveIt 规划组引用了 URDF 并不提供的关节 |

第 1–3 项均已通过尝试解析对应源文件加以确认，见[来源沿革](#来源沿革)。

---

## 质量属性

原样沿用 SolidWorks 导出结果。总质量 **0.5772 kg**。

| Link | 质量 | ixx | iyy | izz |
| --- | --- | --- | --- | --- |
| `base_link` | 0.506299 kg | 1.40520e-04 | 3.08493e-04 | 3.45331e-04 |
| `gripper_slider_link1` | 0.0354537 kg | 1.73251e-05 | 1.26496e-05 | 6.96030e-06 |
| `gripper_slider_link2` | 0.0354537 kg | 1.73251e-05 | 1.26496e-05 | 6.96030e-06 |

两指互为镜像：惯性张量的 `ixx`、`iyy`、`izz` 相同，惯性积在 `ixz`、`iyz` 上符号相反，
符合预期。

完整张量（含质心位置与惯性积）见
[`urdf/litegrip_description.urdf.xacro`](../urdf/litegrip_description.urdf.xacro)。

---

## 设计决策

### 单传动接口

真机只有一台电机、一套传动；两指本来就不能独立控制。模型现在如实表达这一点。
`gripper_opening_joint`（prismatic，由无几何 dummy link `gripper_opening_link` 承载）
是唯一可命令的关节，两个手指关节都是它的 `<mimic>` 跟随关节：

```text
q_finger = -0.5 × opening + stroke
```

`opening = 0` 是实测闭合位，`opening = 2 × stroke` 是全开位。MoveIt 规划组、
`<ros2_control>` 块和关节滑块都只暴露一个关节。

`<mimic>` 会被 `robot_state_publisher` 与 MoveIt 解析，因此派生出的手指位姿无需额外
连线就会出现在 TF 和规划模型中。**ros2_control 本身不计算 `<mimic>`** —— 它只按硬件
组件声明的接口工作。所以 ros2_control 块里两指没有命令/状态接口：总控关节就是全部
接口。

### 为何 mock 仍用 `JointGroupPositionController`

现在只有一个可命令关节，`position_controllers/GripperActionController` 正好适用 ——
它只接受单个关节，并直接表达「闭合直到堵转」。真机链路
（`litegrip_ros2_control`）正是因此选用它。

本包 standalone 的 mock 仍保留 `JointGroupPositionController`：它通过话题接收单元素
关节数组，命令通路与机械臂关节的写法一致，也不依赖 `GripperActionController` 的
参数（goal tolerance、堵转检测）。代价是发送话题 `Float64MultiArray` 而不是调用
Action。若你偏好 Action 接口，把 mock 换成 Action 控制器是受支持的做法。

### 为何碰撞几何复用视觉 STL

原导出的 `<collision>` 与 `<visual>` 指向同一批 mesh。仅 3 个 link 时开销可接受，
且能保证碰撞边界与渲染几何完全一致。`base_link.STL` 约 925 KB，
若本模型被集成到对性能敏感的场景（大量夹爪，或高频碰撞检测），请将
`<collision>` 替换为简化几何体。

### 本包有意不包含的内容

未做 mesh 简化、未加碰撞余量、未设置 `<safety_controller>` 限位、未定义 transmission。
SolidWorks 导出中均无这些内容，凭空填入数值会让本包显得比实际更完善。
`effort` 与 `velocity` 限位是导出默认值，在 xacro 与 README 中均已标注为未标定。

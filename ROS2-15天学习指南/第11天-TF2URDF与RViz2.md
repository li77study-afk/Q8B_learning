# 第 11 天：TF2、URDF 与 RViz2

## 今日目标与时间

总计约 6 小时：1 小时理解坐标系，1.5 小时 static transform，1.5 小时写最小 URDF，1 小时 RViz2 显示，1 小时用命令生成 TF 图并排错。

## 1. 为什么摄像头必须有坐标系

图像中的“像素位置”还不是机器人世界中的位置。至少要区分：

- `base_link`：机器人或 Q8B 主体参考坐标系。
- `camera_link`：摄像头物理安装位置的坐标系。
- `camera_optical_frame`：相机光学约定坐标系，通常 x 向右、y 向下、z 向前。
- `map` / `odom`：更高层的世界或里程计坐标系。

TF2 保存的是这些坐标系之间随时间变化的变换。深度学习只给出图像中的结果；要把结果用于机器人动作，之后必须结合 camera_info、深度或几何关系和 TF。

## 2. 发布一个固定变换

终端 A：

```bash
source /opt/ros/<ROS_DISTRO>/setup.bash
ros2 run tf2_ros static_transform_publisher \
  0.10 0.00 0.30 0.0 0.0 0.0 base_link camera_link
```

六个数字分别是平移 x/y/z 和命令行规定的旋转 yaw/pitch/roll，最后两个参数是父 frame 和子 frame。这里表示摄像头位于主体前方 10 cm、高 30 cm，无旋转。注意：URDF 的 `rpy` 属性按 roll/pitch/yaw 写，不能把两种顺序混用。

终端 B：

```bash
ros2 topic echo /tf_static --once
ros2 run tf2_ros tf2_echo base_link camera_link
```

`tf2_echo` 持续打印从源 frame 到目标 frame 的变换。若命令参数在你的发行版提示不同，以 `ros2 run tf2_ros tf2_echo --help` 为准。

## 3. 查看 TF 树

安装工具并运行：

```bash
sudo apt install -y ros-<ROS_DISTRO>-tf2-tools
ros2 run tf2_tools view_frames
```

它会在当前目录生成 `frames.pdf` 或相关报告。确认当前目录：

```bash
pwd
ls -lh frames*
```

如果没有输出，先确认 static publisher 仍在运行，再检查 `/tf_static`。TF 树必须是连通的树，重复发布同一父子关系或 frame 名拼错会产生非常难找的问题。

## 4. 创建描述包和最小 URDF

```bash
source /opt/ros/<ROS_DISTRO>/setup.bash
cd ~/ros2_ws/src
ros2 pkg create --build-type ament_cmake q8b_description
cd q8b_description
mkdir -p urdf rviz launch
nano urdf/q8b_camera.urdf
```

输入：

```xml
<?xml version="1.0"?>
<robot name="q8b_camera">
  <link name="base_link">
    <visual>
      <geometry><box size="0.40 0.25 0.12"/></geometry>
      <material name="blue"><color rgba="0.1 0.3 0.8 1.0"/></material>
    </visual>
  </link>
  <link name="camera_link">
    <visual>
      <geometry><box size="0.08 0.05 0.05"/></geometry>
      <material name="black"><color rgba="0.05 0.05 0.05 1.0"/></material>
    </visual>
  </link>
  <joint name="camera_mount" type="fixed">
    <parent link="base_link"/>
    <child link="camera_link"/>
    <origin xyz="0.12 0.0 0.18" rpy="0 0 0"/>
  </joint>
</robot>
```

- `link` 是刚体部件；`joint` 描述 link 之间关系。
- `fixed` 表示摄像头没有可动关节。
- `origin` 的 xyz/rpy 与静态 TF 的含义一致。
- URDF 的 visual 是显示几何，不等于碰撞几何，也不等于真实相机模型。

## 5. 把 URDF 文件安装到 share 目录

打开 CMakeLists：

```bash
nano CMakeLists.txt
```

在 `find_package(ament_cmake REQUIRED)` 后增加：

```cmake
install(DIRECTORY urdf launch rviz
  DESTINATION share/${PROJECT_NAME}
)
ament_package()
```

如果末尾已有 `ament_package()`，不要重复写。保存后构建：

```bash
cd ~/ros2_ws
colcon build --packages-select q8b_description
source install/setup.bash
find install/q8b_description/share/q8b_description -type f -print
```

## 6. 启动 robot_state_publisher

先用命令读取 URDF：

```bash
ros2 run robot_state_publisher robot_state_publisher \
  --ros-args -p robot_description:="$(cat src/q8b_description/urdf/q8b_camera.urdf)"
```

`$(...)` 是命令替换：先执行 `cat`，把文件内容作为参数值。这个命令适合实验；正式项目应写 launch 文件，因为命令行可能超过长度限制。

另一个终端：

```bash
ros2 topic echo /tf_static --once
ros2 run tf2_ros tf2_echo base_link camera_link
```

## 7. RViz2 可视化

```bash
rviz2
```

在 RViz 中：

1. Global Options 的 Fixed Frame 设为 `base_link`。
2. Add -> RobotModel，确认能看到盒子和摄像头。
3. Add -> TF，观察两个 frame。
4. Add -> Image，选择第 8 天真实图像话题；若 Image 没有 frame_id，先在相机驱动参数中设置或接受只显示图像的限制。

GUI 通过 RDP 运行时如果窗口异常，先用 `echo $DISPLAY`、`rviz2 --help` 和 CLI 检查；不要把 RViz 的显示问题与 TF 发布问题混为一谈。

## 8. 常见 TF 错误实验

```bash
ros2 run tf2_ros tf2_echo camera_link base_link
ros2 run tf2_ros tf2_echo map camera_link
```

第二条通常因 `map` 不存在而失败。记录错误中的 frame 名；TF 调试第一件事永远是检查拼写和树是否连通。另开终端：

```bash
ros2 topic echo /tf --once
ros2 topic echo /tf_static --once
```

动态 TF 在 `/tf`，静态 TF 在 `/tf_static`。两者不是同一个话题。

## 今日验收

- [ ] 能解释父 frame、子 frame、平移、旋转和 optical frame 的意义。
- [ ] 能发布 `base_link -> camera_link` 的静态变换并用 `tf2_echo` 验证。
- [ ] 能写最小 URDF，并安装到包的 share 目录。
- [ ] 能在 RViz2 中显示 RobotModel 和 TF。
- [ ] 能通过错误信息判断 frame 不存在还是树不连通。

## 官方主线

- TF2：https://docs.ros.org/en/jazzy/Tutorials/Intermediate/Tf2/Introduction-To-Tf2.html
- URDF：https://docs.ros.org/en/jazzy/Tutorials/Intermediate/URDF/URDF-Main.html
- RViz2：https://github.com/ros2/rviz

# 第 12 天：Gazebo 仿真、ROS2 桥与硬件替代验证

## 今日目标与时间

总计约 6 小时：1 小时理解仿真架构，2 小时安装并启动 Gazebo，1 小时查看 ROS-Gazebo bridge，1 小时加入传感器或读取仿真 topic，1 小时为 ARM64 资源不足准备替代方案。

## 1. 仿真解决什么问题

仿真不是“游戏画面”，而是用软件生成机器人状态和传感器数据。你可以在没有摄像头、没有电机、没有场地的情况下测试 topic、TF、控制和算法。它不能完全代替真实硬件：光照、USB 延迟、驱动格式和真实噪声仍需上机验证。

ROS2 Jazzy 通常配合 Gazebo Harmonic；具体安装包和版本必须以官方 ROS-Gazebo 文档为准：

- https://gazebosim.org/docs/latest/ros_installation/
- https://docs.ros.org/en/jazzy/Tutorials/Advanced/Simulators/Gazebo.html
- https://github.com/gazebosim/ros_gz

## 2. 检查 ARM64 和磁盘资源

```bash
uname -m
free -h
apt-cache search ros-<ROS_DISTRO>-ros-gz
```

仿真会消耗 CPU、内存和图形资源。若 Q8B 只有 8 GB 内存，先关闭不必要的 GUI；不要同时跑高分辨率真实相机、RViz 和 Gazebo。

## 3. 安装 ROS-Gazebo 集成

如果 apt 中有对应包：

```bash
sudo apt update
sudo apt install -y ros-<ROS_DISTRO>-ros-gz-sim ros-<ROS_DISTRO>-ros-gz-bridge
```

有些发行版包名可能是 `ros-<ROS_DISTRO>-ros-gz` 或拆分包。用：

```bash
apt-cache search ros-<ROS_DISTRO> | grep -E 'ros-gz|gazebo'
```

确认命令：

```bash
source /opt/ros/<ROS_DISTRO>/setup.bash
ros2 pkg list | grep ros_gz
ros2 run ros_gz_sim --help
```

如果软件包在 Ubuntu 26.04 当前环境没有候选版本，不要源码编译一整套仿真作为当天目标；转到第 7 节替代分支。

## 4. 启动空世界

常见启动方式：

```bash
ros2 launch ros_gz_sim gz_sim.launch.py gz_args:='-r empty.sdf'
```

- `ros2 launch` 启动 ROS2 launch 文件。
- `gz_args` 是传给 Gazebo 的参数；`-r` 表示启动后运行；`empty.sdf` 是空世界。

另一个终端检查：

```bash
gz topic -l
ros2 topic list
```

Gazebo 原生 topic 和 ROS2 topic 不一定相同；这正是 bridge 存在的原因。按 `Ctrl+C` 关闭世界。

## 5. ROS-Gazebo bridge 的概念实验

尝试查询桥接帮助：

```bash
ros2 run ros_gz_bridge parameter_bridge --help
```

典型桥接语法是：

```bash
ros2 run ros_gz_bridge parameter_bridge \
  '/cmd_vel@geometry_msgs/msg/Twist]gz.msgs.Twist'
```

不同 ROS2/Gazebo 版本的方向符号和类型名可能有差异，必须以 `--help` 和官方 `ros_gz` 示例为准。核心含义是：指定 Gazebo 消息类型和 ROS2 消息类型，让数据跨中间件转换。

## 6. 生成一个简单模型

在 Q8B 工作空间外先练习文件：

```bash
mkdir -p ~/q8b_ros2_course/gazebo/models/box
nano ~/q8b_ros2_course/gazebo/models/box/box.sdf
```

输入一个最小模型：

```xml
<?xml version="1.0" ?>
<sdf version="1.9">
  <model name="q8b_box">
    <static>false</static>
    <link name="link">
      <inertial><mass>1.0</mass></inertial>
      <collision name="collision">
        <geometry><box><size>0.5 0.5 0.5</size></box></geometry>
      </collision>
      <visual name="visual">
        <geometry><box><size>0.5 0.5 0.5</size></box></geometry>
      </visual>
    </link>
  </model>
</sdf>
```

启动空世界后，用官方支持的 `create` 命令查看帮助：

```bash
ros2 run ros_gz_sim create --help
```

若帮助中支持 `-file` 和 `-world`，再执行：

```bash
ros2 run ros_gz_sim create \
  -world empty -file ~/q8b_ros2_course/gazebo/models/box/box.sdf \
  -name q8b_box
```

先读帮助再运行是刻意训练；仿真工具参数随版本变化，不能盲抄旧教程。

## 7. 没有 Gazebo 或资源不足时的替代分支

仍然完成仿真思想训练：

```bash
mkdir -p ~/q8b_ros2_course/sim_fallback
cp ~/q8b_ros2_course/bags/day08/camera_demo* ~/q8b_ros2_course/sim_fallback/ 2>/dev/null || true
ros2 bag play ~/q8b_ros2_course/bags/day08/camera_demo --rate 0.5
```

bag 回放是“传感器数据仿真”的低成本替代：节点看到的 topic 与真实运行时相同，但数据来自记录。第 13 天会把它做成自动化测试流程。

## 8. 仿真与真实相机对照表

写一张表比较：topic 名、消息类型、frame_id、频率、QoS、时间来源、图像编码。特别检查：仿真时间可能使用 `/clock`，真实相机通常使用系统时间。使用仿真时可能需要：

```bash
ros2 param get /your_node use_sim_time
ros2 topic echo /clock --once
```

不要把仿真世界运行成功等同于视觉算法已经在真实摄像头上可用。

## 今日验收

- [ ] 能解释 Gazebo、ROS2 和 ros_gz_bridge 各自的职责。
- [ ] 能启动空世界或明确记录缺少官方兼容包的原因。
- [ ] 能找到 Gazebo 原生 topic 和 ROS2 topic 的区别。
- [ ] 能按官方帮助尝试生成一个模型。
- [ ] 如果仿真不可用，能用 bag 回放完成同样的 topic 验证。

## 官方主线

- Gazebo ROS2 安装：https://gazebosim.org/docs/latest/ros_installation/
- ros_gz：https://github.com/gazebosim/ros_gz
- ROS2 仿真概览：https://docs.ros.org/en/jazzy/Tutorials/Advanced/Simulators.html

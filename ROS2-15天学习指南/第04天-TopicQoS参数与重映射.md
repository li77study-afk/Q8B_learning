# 第 4 天：Topic、QoS、参数、命名与重映射

## 今日目标与时间

总计约 6 小时：1 小时复习自定义包，1.5 小时 topic 深入，1.5 小时 QoS 实验，1 小时参数和重映射，1 小时写一个可配置的模拟摄像头节点并排错。

## 1. 为什么 topic 是 ROS2 的主干

Topic 是异步的“数据流”：发布者不需要知道订阅者是谁，订阅者也可以随时加入。摄像头会连续发布图像，视觉节点订阅图像，串口节点可以再订阅检测结果。这种解耦是 ROS2 适合机器人系统的原因。

启动第 3 天节点并检查：

```bash
source /opt/ros/<ROS_DISTRO>/setup.bash
source ~/ros2_ws/install/setup.bash
ros2 run q8b_basics talker
```

另一个终端：

```bash
ros2 topic list -t
ros2 topic info /chat --verbose
ros2 node info /q8b_talker
```

`--verbose` 会显示 QoS 的 reliability、durability、history、depth。不要只看“有消息”，还要看两端 QoS 是否兼容。

## 2. 使用 CLI 临时发布和订阅

停止自定义 talker 后，终端 A 发布：

```bash
ros2 topic pub /chat std_msgs/msg/String "{data: 'message from CLI'}" -r 2
```

- `topic pub` 从命令行创建 publisher。
- `std_msgs/msg/String` 是完整接口名。
- YAML 风格的 `{data: ...}` 填消息字段。
- `-r 2` 让 CLI 以 2 Hz 发布。

终端 B：

```bash
ros2 topic echo /chat std_msgs/msg/String
```

按 `Ctrl+C` 停止发布。再用一次性发布：

```bash
ros2 topic pub --once /chat std_msgs/msg/String "{data: 'one shot'}"
```

这说明 topic 是接口契约：类型不一致时，名字一样也不能通信。

## 3. QoS 实验

ROS2 QoS 常见策略：

- `reliability`：`reliable` 尽量保证送达，`best_effort` 允许丢包；摄像头高频数据常用 best effort。
- `durability`：`volatile` 不保留旧消息，`transient_local` 让后来者可能收到发布者保存的最后消息。
- `history/depth`：保留全部或最近 N 条；队列太小会丢旧数据。

查看真实 QoS：

```bash
ros2 topic info /chat --verbose
ros2 topic hz /chat
ros2 topic bw /chat
```

如果环境中有 turtlesim，运行：

```bash
ros2 run turtlesim turtlesim_node
```

新终端检查：

```bash
ros2 topic info /turtle1/cmd_vel --verbose
ros2 topic pub /turtle1/cmd_vel geometry_msgs/msg/Twist \
  "{linear: {x: 1.0}, angular: {z: 0.5}}" -r 2
```

`geometry_msgs/msg/Twist` 有 `linear` 与 `angular` 两个向量字段；只填写所需分量，其余默认为 0。消息发布持续时乌龟应运动，停止发布后停止。

记录：分别改变 `-r` 和字段值，观察运动速度和 `topic hz`。这比只看示例代码更能理解“频率是运行时行为”。

## 4. 给节点增加参数

编辑 `q8b_basics/talker.py`：在 `super().__init__` 后加入：

```python
self.declare_parameter('period', 0.5)
period = self.get_parameter('period').value
self.timer = self.create_timer(float(period), self.publish_message)
```

删除原来的固定 `self.timer` 行，避免创建两个定时器。参数的作用是让启动者不改源码就能改变行为。

重新编译：

```bash
cd ~/ros2_ws
colcon build --packages-select q8b_basics --symlink-install
source install/setup.bash
ros2 run q8b_basics talker --ros-args -p period:=1.0
```

- `--ros-args` 后面是 ROS2 专用参数，不是 Python 参数。
- `-p period:=1.0` 把名为 `period` 的参数设为 1 秒。

另一个终端观察：

```bash
ros2 param list /q8b_talker
ros2 param get /q8b_talker period
ros2 param set /q8b_talker period 2.0
```

注意：参数 set 是否影响已经创建的 timer，取决于你的代码是否注册参数回调并重建 timer。命令成功不代表代码自动重新读取参数，这是一个重要工程细节。

## 5. 话题重映射

同一个 talker 不改代码发布到另一个话题：

```bash
ros2 run q8b_basics talker --ros-args -r chat:=camera_text
```

终端 B：

```bash
ros2 topic list
ros2 topic echo /camera_text
```

`-r 原名:=新名` 是 remap；它把节点内部使用的名字映射到外部系统名字。摄像头驱动适配不同项目时经常使用这一机制。

## 6. 创建模拟摄像头节点

今天先不接真实相机，用 `sensor_msgs/msg/Image` 的概念做准备。为了不引入图像编码复杂度，先建立一个可配置的文本传感器节点。新建：

```bash
cd ~/ros2_ws/src/q8b_basics
nano q8b_basics/sensor_sim.py
```

输入：

```python
import rclpy
from rclpy.node import Node
from std_msgs.msg import String


class SensorSim(Node):
    def __init__(self):
        super().__init__('sensor_sim')
        self.declare_parameter('topic', 'sensor_text')
        self.declare_parameter('rate_hz', 2.0)
        topic = self.get_parameter('topic').value
        rate_hz = float(self.get_parameter('rate_hz').value)
        self.publisher = self.create_publisher(String, topic, 10)
        self.timer = self.create_timer(1.0 / rate_hz, self.tick)
        self.value = 0

    def tick(self):
        message = String()
        message.data = f'sample={self.value}'
        self.publisher.publish(message)
        self.get_logger().info(message.data)
        self.value += 1


def main(args=None):
    rclpy.init(args=args)
    node = SensorSim()
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()


if __name__ == '__main__':
    main()
```

在 `setup.py` 的 `console_scripts` 增加：

```python
'sensor_sim = q8b_basics.sensor_sim:main',
```

构建并实验：

```bash
cd ~/ros2_ws
colcon build --packages-select q8b_basics --symlink-install
source install/setup.bash
ros2 run q8b_basics sensor_sim --ros-args -p rate_hz:=5.0 -p topic:=camera_metadata
```

另一个终端：

```bash
source /opt/ros/<ROS_DISTRO>/setup.bash
source ~/ros2_ws/install/setup.bash
ros2 topic echo /camera_metadata
ros2 topic hz /camera_metadata
```

## 7. 今日验收

- [ ] 能用 CLI 发布和订阅一个 topic。
- [ ] 能从 `topic info --verbose` 说出 QoS 基本字段。
- [ ] 能解释 topic 名字相同但类型或 QoS 不匹配时为什么不通。
- [ ] 能通过参数改变频率，通过 remap 改外部话题名。
- [ ] `sensor_sim` 运行时能证明参数和话题确实生效。
- [ ] 已记录一次“命令成功但运行行为未变化”的原因，例如代码没有动态读取参数。

## 官方主线

- Understanding topics：https://docs.ros.org/en/jazzy/Tutorials/Beginner-CLI-Tools/Understanding-ROS2-Topics/
- QoS：https://docs.ros.org/en/jazzy/Concepts/Intermediate/About-Quality-of-Service-Settings.html
- Parameters：https://docs.ros.org/en/jazzy/Tutorials/Beginner-CLI-Tools/Understanding-ROS2-Parameters/

# 第 6 天：Launch、YAML、命名空间与工程组织

## 今日目标与时间

总计约 6 小时：1 小时复习多节点启动，2 小时写 Python launch，1 小时 YAML 参数，1 小时命名空间和 remap，1 小时整理工程和删除重建。

## 1. 为什么需要 launch

手工开 4 个终端启动摄像头、处理器、显示器和串口桥接很快会失控。Launch 文件把“启动哪些节点、传哪些参数、怎样重映射、是否输出日志”写成可复现配置。它不是把所有代码揉成一个节点，而是一个进程编排器。

先确认已有入口：

```bash
source /opt/ros/<ROS_DISTRO>/setup.bash
source ~/ros2_ws/install/setup.bash
ros2 pkg executables q8b_basics
```

## 2. 建立 launch 目录

```bash
cd ~/ros2_ws/src/q8b_basics
mkdir -p launch config
find . -maxdepth 2 -type d -print
nano launch/basics.launch.py
```

输入：

```python
from launch import LaunchDescription
from launch_ros.actions import Node


def generate_launch_description():
    talker = Node(
        package='q8b_basics',
        executable='talker',
        name='talker_from_launch',
        output='screen',
        parameters=[{'period': 1.0}],
    )
    listener = Node(
        package='q8b_basics',
        executable='listener',
        name='listener_from_launch',
        output='screen',
        remappings=[('chat', 'launch_chat')],
    )
    return LaunchDescription([talker, listener])
```

要点：

- launch 文件必须提供 `generate_launch_description()`。
- `package` 和 `executable` 对应 `ros2 run` 的两个名字。
- `name` 覆盖节点运行时名字，方便同一个可执行程序启动多次。
- `output='screen'` 把日志显示到当前终端。
- `parameters` 在启动时注入参数；`remappings` 在启动时改话题名。

## 3. 让 setup.py 安装 launch 和 config

`ament_python` 包要把非 Python 文件安装到 share 目录。打开：

```bash
nano setup.py
```

确认导入：

```python
import os
from glob import glob
from setuptools import find_packages, setup
```

确认 `data_files` 包含：

```python
data_files=[
    ('share/ament_index/resource_index/packages',
     ['resource/q8b_basics']),
    ('share/q8b_basics', ['package.xml']),
    (os.path.join('share', 'q8b_basics', 'launch'),
     glob('launch/*.launch.py')),
    (os.path.join('share', 'q8b_basics', 'config'),
     glob('config/*.yaml')),
],
```

如果文件已有 `data_files`，不要重复创建第二个同名变量，把 launch/config 两项合并进去。`glob` 按通配符找文件，安装后 `ros2 launch` 才能找到它们。

构建后查安装结果：

```bash
cd ~/ros2_ws
colcon build --packages-select q8b_basics --symlink-install
source install/setup.bash
find install/q8b_basics -type f -print | sort
```

## 4. 运行 launch 并观察图

```bash
ros2 launch q8b_basics basics.launch.py
```

另开终端：

```bash
source /opt/ros/<ROS_DISTRO>/setup.bash
source ~/ros2_ws/install/setup.bash
ros2 node list
ros2 topic list
ros2 topic echo /launch_chat
```

这里 talker 仍在内部使用 `chat`，listener 的订阅被重映射为 `launch_chat`，因此两者能通信。按 `Ctrl+C` 结束 launch，确认它会回收由它启动的节点。

## 5. 使用 YAML 参数文件

```bash
nano config/basics.yaml
```

输入：

```yaml
/**:
  ros__parameters:
    period: 0.25
```

`/**` 表示匹配任意节点；`ros__parameters` 是 ROS2 YAML 的固定键；缩进必须使用空格，不要用 Tab。

修改 launch 中 talker 的 Node：

```python
parameters=[os.path.join(
    get_package_share_directory('q8b_basics'), 'config', 'basics.yaml')],
```

因此要增加导入：

```python
import os
from ament_index_python.packages import get_package_share_directory
```

完整的 `parameters` 值是一个文件路径列表。编译后运行并观察 `ros2 topic hz /launch_chat`，验证约为 4 Hz。

## 6. 命名空间实验

在 launch 的两个 Node 中加入不同 namespace：

```python
namespace='camera_a',
```

运行后观察：

```bash
ros2 node list
ros2 topic list
ros2 node info /camera_a/talker_from_launch
```

namespace 会影响相对名字；绝对名字以 `/` 开头，通常不受 namespace 影响。工程上优先使用相对名字，让同一节点可以部署到 `/left`、`/right` 等不同实例。

## 7. launch 的调试方法

```bash
ros2 launch q8b_basics basics.launch.py --show-args
ros2 launch q8b_basics basics.launch.py --debug
```

`--show-args` 查看 launch 接收的参数；`--debug` 增加 launch 框架日志。若提示找不到 launch 文件：

```bash
ros2 pkg prefix q8b_basics
find "$(ros2 pkg prefix q8b_basics)" -iname '*launch*' -print
```

这能区分“文件写错”和“文件没有被 setup.py 安装”两类问题。

## 8. 工程目录练习与安全清理

```bash
cd ~/ros2_ws
find src/q8b_basics -maxdepth 3 -type f -print | sort
rm -rf build/q8b_basics install/q8b_basics
colcon build --packages-select q8b_basics --symlink-install
source install/setup.bash
ros2 launch q8b_basics basics.launch.py
```

这次只删除一个包对应的自动产物，验证重新安装 launch 文件。不要删除 `src/q8b_basics`。

## 今日验收

- [ ] 能解释 launch 与节点代码的分工。
- [ ] 能用一条 `ros2 launch` 启动 talker/listener。
- [ ] 能用 YAML 改参数，不改 Python 源码。
- [ ] 能通过 remap、namespace 解释最终话题名。
- [ ] 能从安装目录判断 launch 文件是否安装成功。
- [ ] 能安全删除单个包的 build/install 产物并重建。

## 官方主线

- Launch：https://docs.ros.org/en/jazzy/Tutorials/Intermediate/Launch/Launch-system.html
- Python launch：https://docs.ros.org/en/jazzy/Tutorials/Intermediate/Launch/Using-Substitutions.html
- Parameters：https://docs.ros.org/en/jazzy/Tutorials/Beginner-CLI-Tools/Understanding-ROS2-Parameters.html

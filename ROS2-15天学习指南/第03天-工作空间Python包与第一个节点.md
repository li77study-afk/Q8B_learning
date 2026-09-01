# 第 3 天：工作空间、Python 包、节点和第一个 topic

## 今日目标与时间

总计约 6 小时：1 小时复习 Linux 路径与环境，1.5 小时按官方教程建立 Python 包，1.5 小时编写 publisher/subscriber，1 小时编译和检查安装结果，1 小时独立删除并重建。

## 1. 固定环境检查

新终端执行：

```bash
source /opt/ros/<ROS_DISTRO>/setup.bash
echo "$ROS_DISTRO"
mkdir -p ~/ros2_ws/src
cd ~/ros2_ws
pwd
```

若你用 Docker，先进入容器，再执行同样的 `source`。`source` ROS2 本体必须在 source 自己的工作空间之前，因为工作空间 setup 会叠加在 ROS2 基础环境上。

检查 `colcon`：

```bash
colcon --help | less
ros2 pkg create --help | less
```

## 2. 建立 Python 包

```bash
cd ~/ros2_ws/src
ros2 pkg create --build-type ament_python q8b_basics \
  --dependencies rclpy std_msgs
find q8b_basics -maxdepth 3 -type f -print
```

逐句拆解：

- `cd ~/ros2_ws/src`：进入源码根目录；包不能创建在 `build` 或工作空间根目录。
- `ros2 pkg create`：用 ROS2 工具生成符合规范的包骨架。
- `--build-type ament_python`：选择 Python 的 `ament_python` 构建类型。
- `--dependencies`：把 `rclpy` 和 `std_msgs` 写入依赖声明；`rclpy` 是 Python 客户端库，`std_msgs` 提供标准消息。
- `\` 只是把一条长命令分成多行。

查看自动生成的身份证和配置：

```bash
cd ~/ros2_ws/src/q8b_basics
sed -n '1,220p' package.xml
sed -n '1,240p' setup.py
```

`package.xml` 描述包名和依赖；`setup.py` 描述 Python 包如何被安装以及命令入口如何生成。`sed -n '1,220p'` 只显示指定行，适合检查文件而不打开编辑器。

## 3. 建立 Python 模块和 publisher

```bash
mkdir -p q8b_basics
nano q8b_basics/talker.py
```

在编辑器中输入：

```python
import rclpy
from rclpy.node import Node
from std_msgs.msg import String


class Talker(Node):
    def __init__(self):
        super().__init__('q8b_talker')
        self.publisher = self.create_publisher(String, 'chat', 10)
        self.timer = self.create_timer(0.5, self.publish_message)
        self.count = 0

    def publish_message(self):
        message = String()
        message.data = f'hello {self.count}'
        self.publisher.publish(message)
        self.get_logger().info(f'published: {message.data}')
        self.count += 1


def main(args=None):
    rclpy.init(args=args)
    node = Talker()
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()


if __name__ == '__main__':
    main()
```

理解关键行：

- `Node` 是 ROS2 节点基类；`super().__init__` 注册节点名。
- `create_publisher(String, 'chat', 10)` 创建 String 类型 publisher，话题名是 `chat`，队列深度为 10。
- `create_timer(0.5, ...)` 每 0.5 秒触发回调，理论频率约 2 Hz。
- `rclpy.spin` 进入事件循环，等待计时器和通信事件；没有 spin，回调不会持续执行。
- `destroy_node` 和 `shutdown` 是明确释放资源的收尾动作。

## 4. 建立 subscriber

```bash
nano q8b_basics/listener.py
```

输入：

```python
import rclpy
from rclpy.node import Node
from std_msgs.msg import String


class Listener(Node):
    def __init__(self):
        super().__init__('q8b_listener')
        self.subscription = self.create_subscription(
            String, 'chat', self.receive_message, 10)

    def receive_message(self, message):
        self.get_logger().info(f'received: {message.data}')


def main(args=None):
    rclpy.init(args=args)
    node = Listener()
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()


if __name__ == '__main__':
    main()
```

subscriber 的 `create_subscription` 参数顺序是消息类型、话题名、回调函数、QoS 队列深度。回调收到的 `message` 是 `std_msgs/msg/String` 对象，所以字段是 `message.data`。

## 5. 注册命令入口

打开 setup.py：

```bash
nano setup.py
```

找到 `entry_points`，改成包含：

```python
entry_points={
    'console_scripts': [
        'talker = q8b_basics.talker:main',
        'listener = q8b_basics.listener:main',
    ],
},
```

`talker` 是之后输入的命令名；等号右边表示 Python 模块 `q8b_basics/talker.py` 中的 `main` 函数。冒号不是路径分隔符，而是“模块中的函数”。

## 6. 编译、source、运行

```bash
cd ~/ros2_ws
rosdep install --from-paths src --ignore-src -r -y
colcon build --symlink-install
source install/setup.bash
ros2 pkg executables q8b_basics
```

- `rosdep install` 根据 `package.xml` 安装系统依赖；`--from-paths src` 扫描源码目录，`--ignore-src` 不把源码包当作 apt 包，`-r` 遇到单个失败继续报告，`-y` 自动确认。
- `colcon build` 在工作空间根目录构建；`--symlink-install` 让 Python 源码修改通常不需要复制到 install 目录。
- `source install/setup.bash` 让 ROS2 找到刚编译的包。
- `ros2 pkg executables` 查看注册后的可执行入口。

终端 A：

```bash
source /opt/ros/<ROS_DISTRO>/setup.bash
source ~/ros2_ws/install/setup.bash
ros2 run q8b_basics talker
```

终端 B：

```bash
source /opt/ros/<ROS_DISTRO>/setup.bash
source ~/ros2_ws/install/setup.bash
ros2 run q8b_basics listener
```

终端 C 检查：

```bash
ros2 node list
ros2 topic list -t
ros2 topic echo /chat --once
ros2 topic hz /chat
```

## 7. 练习修改和错误定位

把 `0.5` 改成 `1.0`，重新执行：

```bash
cd ~/ros2_ws
colcon build --packages-select q8b_basics --symlink-install
source install/setup.bash
```

`--packages-select` 只构建指定包，节省时间。若运行结果没变化，确认代码、包名、入口和 source 的顺序；也可执行：

```bash
ros2 pkg prefix q8b_basics
python3 -c "import q8b_basics; print(q8b_basics.__file__)"
```

不要用 `sudo colcon build`。如果之前误用了 sudo，检查：

```bash
ls -ld ~/ros2_ws ~/ros2_ws/install
```

## 8. 删除并重建实验

先停止所有 ROS2 节点，再执行：

```bash
cd ~/ros2_ws
rm -rf build install log
find . -maxdepth 2 -type d -print
colcon build --symlink-install
source install/setup.bash
ros2 run q8b_basics talker
```

这里只删除自动生成的三个目录，不删除 `src`。重新构建成功说明你理解了源码和构建产物的区别。

## 今日验收

- [ ] 能说出 workspace 根目录、`src`、`build`、`install`、`log` 的作用。
- [ ] 能从零生成 `q8b_basics` 包，不依靠复制旧目录。
- [ ] talker 和 listener 能双终端通信。
- [ ] 能解释 publisher、subscriber、callback、spin、QoS depth。
- [ ] 改动计时器后能验证频率变化。
- [ ] 能安全删除自动产物并重建。

## 官方主线

- 创建工作空间：https://docs.ros.org/en/jazzy/Tutorials/Beginner-Client-Libraries/Creating-A-Workspace/Creating-A-Workspace.html
- 创建 Python 包：https://docs.ros.org/en/jazzy/Tutorials/Beginner-Client-Libraries/Creating-Your-First-ROS2-Package.html
- Python publisher/subscriber：https://docs.ros.org/en/jazzy/Tutorials/Beginner-Client-Libraries/Writing-A-Simple-Py-Publisher-And-Subscriber.html

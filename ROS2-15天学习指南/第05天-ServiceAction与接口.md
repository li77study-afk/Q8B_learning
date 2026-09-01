# 第 5 天：Service、Action 与 ROS2 接口

## 今日目标与时间

总计约 6 小时：1 小时复习 topic，1.5 小时 turtlesim service，1.5 小时 Python service server/client，1 小时 action 概念和官方 Fibonacci demo，1 小时定义接口并写学习记录。

## 1. 三种通信方式的边界

| 机制 | 时间模型 | 适合 Q8B 项目中的什么 |
|---|---|---|
| Topic | 连续、异步、可多对多 | 摄像头帧、检测结果、串口状态 |
| Service | 一次请求、一次响应 | 触发拍照、清空计数、读取配置 |
| Action | 长任务、反馈、结果、可取消 | 多帧推理、移动、批量录制 |

不要用 service 代替高频图像流，也不要把需要取消的长任务塞进 service。

## 2. turtlesim 的 service

终端 A：

```bash
source /opt/ros/<ROS_DISTRO>/setup.bash
ros2 run turtlesim turtlesim_node
```

终端 B：

```bash
ros2 service list
ros2 service type /spawn
ros2 interface show turtlesim/srv/Spawn
ros2 service call /spawn turtlesim/srv/Spawn \
  "{x: 3.0, y: 3.0, theta: 0.0, name: 'q8b_turtle'}"
```

- `service type` 查询服务接口。
- `interface show` 显示请求字段和响应字段。
- `service call` 一次性发请求；坐标在 turtlesim 的窗口范围内。

继续练习：

```bash
ros2 service call /clear std_srvs/srv/Empty '{}'
ros2 service call /reset std_srvs/srv/Empty '{}'
```

空服务的请求写成 `{}`，因为没有字段。

## 3. 写一个 service server

先生成包依赖：

```bash
cd ~/ros2_ws/src/q8b_basics
nano q8b_basics/calculator_server.py
```

输入：

```python
import rclpy
from example_interfaces.srv import AddTwoInts
from rclpy.node import Node


class CalculatorServer(Node):
    def __init__(self):
        super().__init__('calculator_server')
        self.server = self.create_service(
            AddTwoInts, 'add_two_ints', self.handle_request)

    def handle_request(self, request, response):
        response.sum = request.a + request.b
        self.get_logger().info(f'{request.a} + {request.b} = {response.sum}')
        return response


def main(args=None):
    rclpy.init(args=args)
    node = CalculatorServer()
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()


if __name__ == '__main__':
    main()
```

补充依赖并注册入口：

```bash
cd ~/ros2_ws/src/q8b_basics
grep -q 'example_interfaces' package.xml || \
  sed -i '/<depend>std_msgs<\/depend>/a\  <depend>example_interfaces</depend>' package.xml
nano setup.py
```

`grep -q` 静默检查依赖；`sed -i` 在文件原地插入一行。若你不熟悉 `sed`，可以直接用 nano 手工在已有 depend 行后添加，优先保证理解而不是追求命令短。

入口增加：

```python
'calculator_server = q8b_basics.calculator_server:main',
```

构建和调用：

```bash
cd ~/ros2_ws
rosdep install --from-paths src --ignore-src -r -y
colcon build --packages-select q8b_basics --symlink-install
source install/setup.bash
ros2 run q8b_basics calculator_server
```

另一个终端：

```bash
source /opt/ros/<ROS_DISTRO>/setup.bash
source ~/ros2_ws/install/setup.bash
ros2 service call /add_two_ints example_interfaces/srv/AddTwoInts "{a: 7, b: 8}"
```

预期响应 `sum: 15`。调用期间观察 server 日志，理解“请求进入回调、回调填充 response、客户端获得 response”。

## 4. 写 service client

```bash
nano q8b_basics/calculator_client.py
```

输入：

```python
import sys

import rclpy
from example_interfaces.srv import AddTwoInts
from rclpy.node import Node


class CalculatorClient(Node):
    def __init__(self):
        super().__init__('calculator_client')
        self.client = self.create_client(AddTwoInts, 'add_two_ints')

    def send(self, a, b):
        while not self.client.wait_for_service(timeout_sec=1.0):
            self.get_logger().info('service not available, waiting...')
        request = AddTwoInts.Request()
        request.a = a
        request.b = b
        future = self.client.call_async(request)
        rclpy.spin_until_future_complete(self, future)
        return future.result()


def main(args=None):
    rclpy.init(args=args)
    node = CalculatorClient()
    a = int(sys.argv[1]) if len(sys.argv) > 1 else 1
    b = int(sys.argv[2]) if len(sys.argv) > 2 else 2
    response = node.send(a, b)
    node.get_logger().info(f'result={response.sum}')
    node.destroy_node()
    rclpy.shutdown()


if __name__ == '__main__':
    main()
```

注册、构建、调用：

```bash
nano setup.py
# 增加：'calculator_client = q8b_basics.calculator_client:main',
cd ~/ros2_ws
colcon build --packages-select q8b_basics --symlink-install
source install/setup.bash
ros2 run q8b_basics calculator_client 20 22
```

`sys.argv` 是普通 Python 命令行参数；它与 `--ros-args` 是两套不同参数系统。

## 5. Action：长任务和反馈

运行官方 action demo：

```bash
ros2 run action_tutorials_py fibonacci_action_server
```

另一个终端：

```bash
ros2 action list -t
ros2 action info /fibonacci
ros2 interface show action_tutorials_interfaces/action/Fibonacci
ros2 action send_goal /fibonacci action_tutorials_interfaces/action/Fibonacci \
  "{order: 8}" --feedback
```

观察 goal、feedback、result。Action 适合“运行一段时间且使用者想知道进度”的任务。实验中按 `Ctrl+C` 取消客户端，思考服务器是否实现了取消逻辑。

如果 demo 包未安装，先查：

```bash
ros2 pkg list | grep action_tutorials
```

不要为了一个 demo 破坏当前安装；记录“未安装”并阅读官方 action 教程即可。

## 6. 接口文件先读懂，不急着自定义

```bash
ros2 interface list | grep -E 'msg/|srv/|action/' | head -n 30
ros2 interface show sensor_msgs/msg/Image
ros2 interface show sensor_msgs/msg/CameraInfo
```

第 8、9 天的摄像头会使用 `Image` 和 `CameraInfo`。今天先记住：接口是通信双方共同遵守的数据结构，topic/service/action 只是不同的通信语义。

## 今日验收

- [ ] 能用 CLI 调 turtlesim 的 spawn、clear、reset service。
- [ ] Python service server/client 能计算任意两个整数。
- [ ] 能解释异步 future、service 回调和 action feedback。
- [ ] 能看懂 `sensor_msgs/msg/Image` 的字段，不把 `data` 误认为普通字符串。
- [ ] 能说出触发拍照应选 service，连续相机帧应选 topic，长时间任务应选 action 的理由。

## 官方主线

- Services：https://docs.ros.org/en/jazzy/Tutorials/Beginner-CLI-Tools/Understanding-ROS2-Services/
- Python service：https://docs.ros.org/en/jazzy/Tutorials/Beginner-Client-Libraries/Writing-A-Simple-Py-Service-And-Client.html
- Actions：https://docs.ros.org/en/jazzy/Tutorials/Beginner-CLI-Tools/Understanding-ROS2-Actions/

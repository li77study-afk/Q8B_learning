# 第 9 天：cv_bridge、OpenCV 图像处理与性能测量

## 今日目标与时间

总计约 6 小时：1 小时复习 Image 消息，1 小时安装 Python 图像依赖，2 小时写订阅/处理/发布节点，1 小时用 rosbag 回放，1 小时测 FPS、延迟和内存。

## 1. Image 消息与 cv_bridge

`sensor_msgs/msg/Image` 不是一张 Python 图片，它包含 `height`、`width`、`encoding`、`step` 和字节数组 `data`。`cv_bridge` 把 ROS Image 与 OpenCV 的 NumPy 数组相互转换。

查看接口：

```bash
source /opt/ros/<ROS_DISTRO>/setup.bash
ros2 interface show sensor_msgs/msg/Image
apt-cache search ros-<ROS_DISTRO>-cv-bridge
sudo apt install -y ros-<ROS_DISTRO>-cv-bridge python3-opencv
python3 -c "import cv2; print(cv2.__version__)"
```

若 Python 找不到 `cv2`，检查 `which python3` 和 apt 包安装状态；不要先用 `sudo pip` 覆盖系统 Python。

## 2. 新建图像处理包

```bash
cd ~/ros2_ws/src
ros2 pkg create --build-type ament_python q8b_vision \
  --dependencies rclpy sensor_msgs cv_bridge
cd q8b_vision
mkdir -p q8b_vision launch
```

查看生成内容：

```bash
find . -maxdepth 2 -type f -print
```

## 3. 编写灰度和边缘处理节点

```bash
nano q8b_vision/image_processor.py
```

输入：

```python
import cv2
import rclpy
from cv_bridge import CvBridge
from rclpy.node import Node
from rclpy.qos import qos_profile_sensor_data
from sensor_msgs.msg import Image


class ImageProcessor(Node):
    def __init__(self):
        super().__init__('image_processor')
        self.declare_parameter('input_topic', '/image_raw')
        self.declare_parameter('output_topic', '/vision/edges')
        input_topic = self.get_parameter('input_topic').value
        output_topic = self.get_parameter('output_topic').value
        self.bridge = CvBridge()
        self.publisher = self.create_publisher(Image, output_topic, 10)
        self.subscription = self.create_subscription(
            Image, input_topic, self.process, qos_profile_sensor_data)
        self.frames = 0
        self.started = self.get_clock().now()

    def process(self, message):
        try:
            image = self.bridge.imgmsg_to_cv2(message, desired_encoding='bgr8')
            gray = cv2.cvtColor(image, cv2.COLOR_BGR2GRAY)
            edges = cv2.Canny(gray, 80, 160)
            output = self.bridge.cv2_to_imgmsg(edges, encoding='mono8')
            output.header = message.header
            self.publisher.publish(output)
            self.frames += 1
            if self.frames % 30 == 0:
                elapsed = (self.get_clock().now() - self.started).nanoseconds / 1e9
                self.get_logger().info(f'frames={self.frames}, avg_fps={self.frames / elapsed:.2f}')
        except Exception as error:
            self.get_logger().error(f'image processing failed: {error}')


def main(args=None):
    rclpy.init(args=args)
    node = ImageProcessor()
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()


if __name__ == '__main__':
    main()
```

逐段理解：

- `imgmsg_to_cv2` 将 Image 转 NumPy/OpenCV；`desired_encoding='bgr8'` 要求常见三通道格式。
- `Canny` 只演示处理链，不是深度学习；输出是单通道 `mono8`。
- 复制 `header` 保留时间戳和 frame_id，这是 TF2 和同步的重要基础。
- 只在每 30 帧打印一次日志，避免日志本身拖慢图像处理。
- `try/except` 防止单帧格式异常直接杀死节点，但调试阶段仍要记录错误。

## 4. 注册、构建和连接真实相机

打开 setup.py：

```bash
nano setup.py
```

在 `console_scripts` 加：

```python
'image_processor = q8b_vision.image_processor:main',
```

构建：

```bash
cd ~/ros2_ws
rosdep install --from-paths src --ignore-src -r -y
colcon build --packages-select q8b_vision --symlink-install
source install/setup.bash
ros2 run q8b_vision image_processor --ros-args \
  -p input_topic:=/image_raw -p output_topic:=/vision/edges
```

若相机实际话题不同，把参数替换成第 8 天 `ros2 topic list -t` 的真实结果。

观察输出：

```bash
ros2 topic list -t
ros2 topic hz /vision/edges
ros2 topic info /vision/edges --verbose
rqt_image_view
```

在 `rqt_image_view` 选择 `/vision/edges`。没有 GUI 时用 `ros2 topic hz` 和日志 FPS 证明处理链在工作。

## 5. 用 bag 做可重复测试

相机不方便反复占用时，用第 8 天的 bag：

终端 A：

```bash
ros2 run q8b_vision image_processor --ros-args \
  -p input_topic:=/image_raw -p output_topic:=/vision/edges
```

终端 B：

```bash
ros2 bag play ~/q8b_ros2_course/bags/day08/camera_demo --rate 0.5
```

记录：bag 回放速度、输入 `topic hz`、输出 `topic hz`、节点平均 FPS。用同一个 bag 比较改算法前后，避免光线变化造成假结论。

## 6. 性能实验

执行：

```bash
free -h
ros2 topic hz /image_raw
ros2 topic bw /image_raw
btop
```

分别把相机设为 640x480 和 1280x720（以驱动支持为准），比较：

- 输入带宽是否明显增加。
- 处理节点平均 FPS 是否下降。
- CPU 是否接近满载、温度是否持续升高。
- 延迟是否因队列积压而增加。

解释：图像不是“一个变量”，分辨率、编码、频率和 QoS 一起决定资源使用。高频图像通常更适合 best effort，但处理链两端必须选择兼容 QoS。

## 7. 删除重建练习

```bash
cd ~/ros2_ws
rm -rf build/q8b_vision install/q8b_vision log
colcon build --packages-select q8b_vision --symlink-install
source install/setup.bash
ros2 pkg executables q8b_vision
```

确认包从 `src` 重新安装，且 `image_processor` 入口仍在。

## 今日验收

- [ ] 能解释 Image 消息与 OpenCV 数组的差异。
- [ ] 节点能订阅真实图像，做灰度/边缘处理，再发布新的 Image。
- [ ] 输出 header 的时间戳和 frame_id 被保留。
- [ ] 能用 bag 重复处理同一批图像并记录 FPS。
- [ ] 能用 `topic hz`、`topic bw` 和 `btop` 做基本性能判断。

## 官方主线

- cv_bridge：https://docs.ros.org/en/jazzy/p/cv_bridge/
- image_tools：https://github.com/ros2/demos/tree/jazzy/image_tools
- sensor_msgs Image：https://docs.ros.org/en/jazzy/p/sensor_msgs/msg/Image.html

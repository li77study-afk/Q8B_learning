# 第 10 天：串口 Linux 基础与 ROS2 桥接（可选）

## 今日目标与时间

总计约 5.5～6 小时：1 小时识别设备和权限，1.5 小时用 Python 做回环通信，1.5 小时写 ROS2 串口桥，1 小时协议和异常处理，0.5～1 小时无硬件替代测试。没有串口时全部使用虚拟 PTY 分支。

## 1. 串口设备盘点

插入 USB 转串口或开发板后：

```bash
lsusb
ls /dev/ttyUSB* /dev/ttyACM* 2>/dev/null || true
```

常见设备是 `/dev/ttyUSB0` 或 `/dev/ttyACM0`，但序号可能变化。以插拔前后 `dmesg` 和 `ls -l` 为准，不能硬编码“永远是 ttyUSB0”。

```bash
ls -l /dev/ttyUSB0 /dev/ttyACM0 2>/dev/null || true
id
getent group dialout
```

如果当前用户不在 `dialout` 组：

```bash
sudo usermod -aG dialout "$USER"
```

注销再登录后用 `id -nG` 验证。不要给串口 `chmod 777`，那通常只是临时绕过权限且不安全。

## 2. 先定义一个简单协议

本课程使用一行一条的 ASCII 协议，便于观察：

```text
PING\n
LED 1\n
DETECT person 0.87\n
```

串口本身只是字节流，没有消息边界；换行符是我们人为选择的边界。真正项目还要考虑校验和、长度、超时、版本号和错误响应。

## 3. 安装 pyserial 并做虚拟串口回环

```bash
sudo apt update
sudo apt install -y python3-serial socat
python3 -c "import serial; print(serial.VERSION)"
```

没有真实设备时，用两个伪终端模拟串口两端：

终端 A：

```bash
mkdir -p ~/q8b_ros2_course/serial
socat -d -d pty,raw,echo=0,link="$HOME/q8b_ros2_course/serial/ttyA" \
  pty,raw,echo=0,link="$HOME/q8b_ros2_course/serial/ttyB"
```

保持运行并记下 socat 输出。`pty` 创建伪终端，`raw` 不做终端字符加工，`echo=0` 关闭回显，`link=` 创建稳定软链接。终端 B 查看：

```bash
ls -l ~/q8b_ros2_course/serial/ttyA ~/q8b_ros2_course/serial/ttyB
```

## 4. Python 串口发送/接收

```bash
nano ~/q8b_ros2_course/serial/serial_test.py
```

输入：

```python
import sys
import serial


device = sys.argv[1]
with serial.Serial(device, baudrate=115200, timeout=1.0) as port:
    print(f'opened {port.name}')
    port.write(b'PING\n')
    reply = port.readline()
    print(f'received: {reply!r}')
```

终端 B 运行一个读写工具（没有 `screen` 时安装）：

```bash
sudo apt install -y minicom
minicom -D "$HOME/q8b_ros2_course/serial/ttyB" -b 115200
```

另一终端执行：

```bash
python3 ~/q8b_ros2_course/serial/serial_test.py \
  "$HOME/q8b_ros2_course/serial/ttyA"
```

理解 `timeout=1.0`：读取一行最多等待 1 秒；没有换行数据时返回空字节，不应该无限阻塞 ROS2 节点。

## 5. 建立 ROS2 串口桥包

```bash
source /opt/ros/<ROS_DISTRO>/setup.bash
cd ~/ros2_ws/src
ros2 pkg create --build-type ament_python q8b_serial \
  --dependencies rclpy std_msgs
mkdir -p q8b_serial/q8b_serial
```

创建 `q8b_serial/q8b_serial/bridge.py`：

```bash
nano q8b_serial/q8b_serial/bridge.py
```

输入：

```python
import serial
import rclpy
from rclpy.node import Node
from std_msgs.msg import String


class SerialBridge(Node):
    def __init__(self):
        super().__init__('serial_bridge')
        self.declare_parameter('device', '/dev/ttyUSB0')
        self.declare_parameter('baudrate', 115200)
        device = self.get_parameter('device').value
        baudrate = int(self.get_parameter('baudrate').value)
        self.publisher = self.create_publisher(String, 'serial_rx', 10)
        self.subscription = self.create_subscription(
            String, 'serial_tx', self.send_line, 10)
        self.port = serial.Serial(device, baudrate=baudrate, timeout=0.01)
        self.timer = self.create_timer(0.02, self.read_lines)

    def send_line(self, message):
        self.port.write((message.data + '\n').encode('utf-8'))

    def read_lines(self):
        raw = self.port.readline()
        if raw:
            message = String()
            message.data = raw.decode('utf-8', errors='replace').rstrip('\r\n')
            self.publisher.publish(message)

    def destroy_node(self):
        self.port.close()
        super().destroy_node()


def main(args=None):
    rclpy.init(args=args)
    node = SerialBridge()
    rclpy.spin(node)
    node.destroy_node()
    rclpy.shutdown()


if __name__ == '__main__':
    main()
```

这个节点把 `/serial_tx` 的 String 写到串口，把串口按行读取后发布到 `/serial_rx`。`timeout=0.01` 和定时器让读取不会永久阻塞 executor；真实产品还应捕获断线和打开失败异常。

在 setup.py 增加：

```python
'bridge = q8b_serial.bridge:main',
```

构建：

```bash
cd ~/ros2_ws
rosdep install --from-paths src --ignore-src -r -y
colcon build --packages-select q8b_serial --symlink-install
source install/setup.bash
```

## 6. ROS2 topic 到串口验证

用真实设备或虚拟 `ttyA` 启动：

```bash
ros2 run q8b_serial bridge --ros-args \
  -p device:="$HOME/q8b_ros2_course/serial/ttyA"
```

另一个终端向 ROS2 发布：

```bash
ros2 topic pub --once /serial_tx std_msgs/msg/String \
  "{data: 'LED 1'}"
```

在连接另一端的 minicom 中应看到 `LED 1`。反向在 minicom 输入 `STATUS ok` 并回车：

```bash
ros2 topic echo /serial_rx
```

若使用真实设备，把参数路径替换成 `readlink -f /dev/ttyUSB0` 的结果，并确认两端 baudrate、数据位、停止位一致。

## 7. 串口桥的工程风险讨论

记录以下问题的答案：

- 设备拔出时 `serial.Serial` 和 `readline` 会发生什么？
- 一行中间断电、乱码、超长输入如何处理？
- 如果检测结果频率是 30 Hz，串口只能 5 Hz，是否需要队列、丢弃策略或只发送最新值？
- 如何用 `udev` 规则给设备固定别名？

今天不要求完成 `udev` 规则，但要知道 `/dev/ttyUSB0` 不是稳定身份。深度学习结果到串口时优先设计短消息和限频，不要把整张图片写入串口。

## 今日验收

- [ ] 能识别串口设备、解释 `dialout` 权限并安全处理拔插。
- [ ] 没有真实硬件时能用 socat 建立两端伪串口。
- [ ] 能用 Python 按行发送和读取，理解 timeout。
- [ ] ROS2 bridge 能完成 `/serial_tx` 到串口、串口到 `/serial_rx` 的双向转换。
- [ ] 已记录协议边界、断线、乱码和频率不匹配的风险。

## 官方主线

- ROS2 publisher/subscriber：https://docs.ros.org/en/jazzy/Tutorials/Beginner-Client-Libraries/Writing-A-Simple-Py-Publisher-And-Subscriber.html
- Linux 串口设备说明：https://docs.kernel.org/serial/serial-rs485.html
- pySerial 文档：https://pyserial.readthedocs.io/en/latest/

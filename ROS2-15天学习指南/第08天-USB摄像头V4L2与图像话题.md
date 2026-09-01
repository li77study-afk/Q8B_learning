# 第 8 天：USB 摄像头、V4L2 与 ROS2 图像话题

## 今日目标与时间

总计约 6 小时：1 小时硬件安全与设备识别，1 小时 V4L2 查询，1.5 小时安装和运行官方图像工具，1.5 小时录制图像 bag，1 小时处理摄像头打不开和 Docker 透传问题。

## 1. 插摄像头前后对比

摄像头插入 Q8B 的 USB 口。插入前执行：

```bash
lsusb > /tmp/usb-before.txt
ls /dev/video* 2>/dev/null || true
```

插入后执行：

```bash
lsusb > /tmp/usb-after.txt
diff -u /tmp/usb-before.txt /tmp/usb-after.txt || true
ls -l /dev/video*
```

- `diff -u` 显示两份清单差异；没有差异时通过 `|| true` 不让练习停止。
- `/dev/video0` 是设备节点，不保证一定是 RGB 摄像头。
- `ls -l` 可能显示设备归 `video` 组所有。

如果没有 `/dev/video*`：换 USB 口、换线、用 `lsusb` 判断是否枚举，查看 `dmesg` 的最后几行。不要直接修改权限或重装 ROS2。

## 2. V4L2 能力查询

安装命令行工具：

```bash
sudo apt update
sudo apt install -y v4l-utils
v4l2-ctl --list-devices
v4l2-ctl --device=/dev/video0 --all
v4l2-ctl --device=/dev/video0 --list-formats-ext
```

- `v4l2-ctl` 是 Linux Video4Linux2 的诊断工具。
- `--device` 指定设备；`--all` 看驱动和能力；`--list-formats-ext` 看支持的像素格式、分辨率和帧率。

把输出记录到当天笔记：

```bash
mkdir -p ~/q8b_ros2_course/notes/day08
v4l2-ctl --device=/dev/video0 --all | tee ~/q8b_ros2_course/notes/day08/camera-all.txt
v4l2-ctl --device=/dev/video0 --list-formats-ext | tee ~/q8b_ros2_course/notes/day08/formats.txt
```

先选择摄像头明确支持的 640x480 或 1280x720，不要一开始追求 4K。Q8B CPU、USB 带宽和内存都有限。

## 3. 设备权限

```bash
id
ls -l /dev/video0
getent group video
```

`id` 显示当前用户和所属组；`getent group video` 查询 video 组成员。若当前用户不在 video 组：

```bash
sudo usermod -aG video "$USER"
```

加入组不会自动影响已经打开的 SSH/RDP 会话。注销并重新登录，再检查：

```bash
id -nG
```

不要使用 `chmod 777 /dev/video0` 作为永久方案。

## 4. 先用 V4L2 验证相机，不牵扯 ROS2

如果安装桌面 GUI：

```bash
sudo apt install -y cheese ffmpeg
cheese
```

没有 GUI 时用 ffmpeg 抓一张图：

```bash
mkdir -p ~/q8b_ros2_course/camera_samples
ffmpeg -f v4l2 -video_size 640x480 -i /dev/video0 -frames:v 1 \
  ~/q8b_ros2_course/camera_samples/frame01.jpg
file ~/q8b_ros2_course/camera_samples/frame01.jpg
```

`-f v4l2` 选择 Linux 摄像头输入；`-video_size` 选择分辨率；`-frames:v 1` 只取一帧。若格式不支持，按 `v4l2-ctl` 的输出改分辨率或加 `-input_format mjpeg`。

## 5. 安装 ROS2 图像工具

先搜包名，避免发行版差异：

```bash
apt-cache search ros-<ROS_DISTRO>-image-tools
sudo apt install -y ros-<ROS_DISTRO>-image-tools ros-<ROS_DISTRO>-rqt-image-view
```

ROS2 官方 `image_tools` 示例常用于验证图像消息，实际 USB 驱动可选 `v4l2_camera` 或 `usb_cam`，先查当前仓库：

```bash
apt-cache search ros-<ROS_DISTRO> | grep -E 'v4l2-camera|usb-cam|image-tools'
```

若有 `v4l2_camera`：

```bash
sudo apt install -y ros-<ROS_DISTRO>-v4l2-camera
source /opt/ros/<ROS_DISTRO>/setup.bash
ros2 run v4l2_camera v4l2_camera_node --ros-args \
  -p video_device:=/dev/video0 \
  -p image_size:=[640,480]
```

如果参数名在当前版本不同，以：

```bash
ros2 run v4l2_camera v4l2_camera_node --ros-args --help
```

显示的参数为准。驱动运行后：

```bash
ros2 node list
ros2 topic list -t | grep -E 'image|camera'
ros2 topic info /image_raw --verbose
```

实际话题可能是 `/image_raw` 或 `/camera/image_raw`，必须以 `ros2 topic list -t` 的结果为准。

## 6. 查看图像

```bash
rqt_image_view
```

在窗口中选择图像话题。也可使用 image_tools：

```bash
ros2 run image_tools showimage --ros-args -r image:=/image_raw
```

如果 `ros2 topic echo` 订阅不到相机图像，先按传感器常用 QoS 试一次：

```bash
ros2 topic echo /image_raw --qos-reliability best_effort --once
```

相机发布者常使用 `best_effort` 以避免高带宽图像阻塞；可靠订阅者和 best-effort 发布者可能无法匹配。第 9 天的 Python 订阅节点会显式使用 `qos_profile_sensor_data`。

如果远程 RDP 无法显示 GUI，使用 `ros2 topic hz`、`ros2 topic info` 和 rosbag 验证数据已发布；图形问题单独记录。

## 7. 录制真实摄像头 bag

```bash
mkdir -p ~/q8b_ros2_course/bags/day08
ros2 bag record /image_raw -o ~/q8b_ros2_course/bags/day08/camera_demo
```

按 `Ctrl+C` 停止，随后：

```bash
ros2 bag info ~/q8b_ros2_course/bags/day08/camera_demo
du -sh ~/q8b_ros2_course/bags/day08/camera_demo
```

高分辨率图像很快占满磁盘。课程中只录 20～60 秒，并在每次录制前用 `df -h ~` 检查空间。

## 8. Docker 透传分支

原生安装推荐直接使用 `/dev/video0`。如果 ROS2 在 Docker：

```bash
  -v "$HOME/ros2_ws:/root/ros2_ws" \
  ros:<ROS_DISTRO>-desktop bash
```

`--device` 把单个视频设备暴露给容器；不要一开始使用 `--privileged`。容器内执行 `ls -l /dev/video0`，确认驱动和设备节点可见。图形显示还需 DISPLAY/X11 或 RDP 配置，遇到问题可先用无 GUI 验证。

## 今日验收

- [ ] 能区分 USB 枚举失败、Linux 设备权限失败、V4L2 格式失败、ROS2 驱动失败。
- [ ] 能用 `v4l2-ctl` 找到支持的分辨率和格式。
- [ ] 真实摄像头已经发布 `sensor_msgs/msg/Image`，并能用 `topic list -t` 找到。
- [ ] 至少用一种方法看到图像或证明图像 topic 有频率。
- [ ] 完成一段短 bag，并记录大小和分辨率。

## 官方主线

- ROS2 image_tools：https://github.com/ros2/demos/tree/jazzy/image_tools
- sensor_msgs Image：https://docs.ros.org/en/jazzy/p/sensor_msgs/msg/Image.html
- ROS2 bag：https://docs.ros.org/en/jazzy/Tutorials/Advanced/Recording-A-Bag-From-The-Command-Line/

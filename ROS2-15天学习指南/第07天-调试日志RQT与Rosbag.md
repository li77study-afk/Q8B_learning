# 第 7 天：调试、日志、rqt、ros2 doctor 与 rosbag

## 今日目标与时间

总计约 6 小时：1 小时建立故障排查顺序，1 小时日志级别，1 小时 rqt/rqt_graph，1.5 小时 rosbag 录制回放，1.5 小时故意制造并定位 3 个错误。

## 1. ROS2 故障排查顺序

以后遇到“不工作”，按以下顺序，不要凭感觉改代码：

1. `pwd`：确认自己在对的工作空间和目录。
2. `echo $ROS_DISTRO`、`printenv | grep -E 'ROS|AMENT|RMW'`：确认环境。
3. `ros2 node list`：节点进程是否活着。
4. `ros2 topic list -t`：名字和类型是否正确。
5. `ros2 topic info --verbose`：连接数和 QoS 是否兼容。
6. `ros2 topic echo --once`：是否真的有数据。
7. `ros2 doctor`、日志、系统资源：再看环境和性能。

这个顺序从“进程存在”逐渐缩小到“数据内容”，比盲目重装更快。

## 2. ROS2 日志级别和保存

运行：

```bash
ros2 run q8b_basics talker --ros-args --log-level debug
```

`--ros-args` 表示后面的参数交给 ROS2；`--log-level debug` 提高该节点日志详细程度。也可按 logger 名设置：

```bash
ros2 run q8b_basics talker --ros-args \
  --log-level q8b_talker:=debug
```

看磁盘日志：

```bash
ls -lah ~/.ros/log
find ~/.ros/log -type f -name '*.log' -printf '%TY-%Tm-%Td %TH:%TM %p\n' | sort
```

- `~/.ros/log` 是当前用户的 ROS 日志目录。
- `-printf` 是 find 的格式字符串；`%p` 是路径，前面的格式是时间。

清理旧日志前先查看空间：

```bash
rm -ri ~/.ros/log
```

如果确认删除，ROS2 会在下次启动时重建目录。不要用 `sudo` 删除用户日志。

## 3. ros2 doctor 和系统资源

```bash
ros2 doctor --report | tee ~/q8b_ros2_course/notes/doctor.txt
free -h
df -h ~
btop
```

在 `btop` 中观察编译或图像节点运行时 CPU、内存和温度，按 `q` 退出。ARM64 板子长时间编译和推理必须关注温度与 swap；遇到卡顿先判断是否资源不足。

## 4. rqt_graph 和图形化观察

启动第 6 天 launch：

```bash
ros2 launch q8b_basics basics.launch.py
```

另一个图形终端：

```bash
rqt_graph
```

如果没有安装：

```bash
sudo apt install ros-<ROS_DISTRO>-rqt-graph ros-<ROS_DISTRO>-rqt-common-plugins
```

在图中找出 talker、listener 和 `/launch_chat` 的箭头。GUI 若在远程桌面里打不开，先用 CLI 完成同一验证；不要因为 GUI 失败判断 ROS2 通信失败。

还可以运行：

```bash
rqt_console
rqt
```

`rqt_console` 按节点、严重级别筛选日志；`rqt` 是插件容器。尝试加载 Node Graph、Message Publisher、Topic Monitor 等插件，并记录哪个插件在当前 ROS2 发行版可用。

## 5. rosbag 录制与回放

终端 A 启动节点：

```bash
ros2 run q8b_basics sensor_sim --ros-args -p rate_hz:=2.0
```

终端 B 建目录并录制：

```bash
mkdir -p ~/q8b_ros2_course/bags/day07
cd ~/q8b_ros2_course/bags/day07
ros2 bag record /sensor_text -o sensor_demo
```

运行约 20 秒按 `Ctrl+C`。`-o sensor_demo` 指定输出名；bag 不要放进 `src`，它是实验数据。

查看元数据：

```bash
ros2 bag info sensor_demo
du -sh sensor_demo
```

回放：

```bash
ros2 bag play sensor_demo
```

另开终端订阅：

```bash
ros2 topic echo /sensor_text
```

尝试：

```bash
ros2 bag play sensor_demo --rate 0.5
ros2 bag play sensor_demo --loop
```

`--rate 0.5` 半速回放，`--loop` 循环回放；循环时按 `Ctrl+C` 停止。

## 6. 故意制造和定位错误

**错误 A：忘记 source**

```bash
env -i HOME="$HOME" PATH=/usr/bin:/bin bash --noprofile --norc
ros2 topic list
exit
```

`env -i` 清空环境，模拟新鲜 shell。退出后回到正常终端，source 并重试。理解不是 ROS2 消失，而是当前进程找不到它。

**错误 B：话题名错**

```bash
ros2 topic echo /sensor_text_typo --once
ros2 topic list
```

用实际 `topic list` 对照，注意前导 `/` 和大小写。

**错误 C：节点没运行**

```bash
ros2 node list
ros2 node info /does_not_exist
```

记录完整报错和你用哪条命令缩小了范围。

## 7. 今日验收

- [ ] 能用 `ros2 doctor`、`rqt_graph`、`rqt_console` 分别检查环境、拓扑、日志。
- [ ] 能录制一个 bag，查看 metadata，并在没有原始节点时回放数据。
- [ ] 能解释 bag 是数据记录，不是源代码，也不是节点。
- [ ] 故意制造的 3 个错误都有“现象、假设、命令证据、结论”记录。
- [ ] 能观察 Q8B 的内存、磁盘和温度，而不是只看程序是否退出。

## 官方主线

- ros2 doctor：https://docs.ros.org/en/jazzy/Tutorials/Beginner-CLI-Tools/Getting-Back-To-The-Basics/
- rosbag：https://docs.ros.org/en/jazzy/Tutorials/Advanced/Recording-A-Bag-From-The-Command-Line/
- 日志：https://docs.ros.org/en/jazzy/Concepts/Intermediate/About-Logging.html

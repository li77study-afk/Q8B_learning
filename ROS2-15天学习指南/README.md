# Q8B 上的 ROS2 + 摄像头 + 深度学习：15 天实战课程

这是一套给 Linux 新手的完整练习路径。目标不是背命令，而是 15 天后能够在 Q8B 上独立完成：

`摄像头驱动 -> ROS2 图像话题 -> OpenCV 处理 -> Q8B NPU 推理 -> RViz/窗口查看 -> 可选串口输出`

## 先看清楚课程边界

- 每天建议投入 **5.5～7 小时**，分成上午、下午、晚间复盘三段；安装下载、编译等待时间也要观察和记录，不能只复制粘贴命令。
- 主语言使用 Python；ROS2 的概念、命令行和工程结构同时覆盖，之后再转 C++ 会容易很多。
- 只假设有一个 USB 摄像头；串口是第 10 天的可选分支，没有串口也不影响主线。
- 深度学习以 Q8B 的 Qualcomm Hexagon NPU 为主线，CPU/OpenCV 只作基准和故障兜底；不把“模型能在 CPU 跑”误认为“已经用上 NPU”。
- 每天都必须完成“验证标准”和“学习记录”，否则不要急着进入下一天。

## Q8B NPU 是本课程主线

Radxa 官方规格确认 Dragon Q8B 使用 Qualcomm Snapdragon 8cx Gen 3 / `SC8280XP`，内置 Qualcomm Hexagon NPU，AI Engine 标称最高 29+ TOPS。Radxa NPU 文档确认：

- Q8B 的 `SC8280XP` 使用 Hexagon `V68`。
- `fastrpc` 测试使用 `fastrpc_test -a v68`。
- QAI AppBuilder 和 QAI Hub 下载模型时，SC8280XP 按 `--chipset 6490` 处理，因为它与 QCS6490 共用 Hexagon V68 模型目标。
- QAIRT 的 `Context-Binary` 是针对具体 NPU 架构编译的模型格式；不能把别的芯片的 `.bin` 直接当成 Q8B 模型。
- T2 或更高版本的 Radxa 系统镜像通常已预装 NPU 运行环境；第三方或通用 Ubuntu 镜像可能没有 `fastrpc`、`/dev/fastrpc-cdsp` 和 DSP 库。
- 完整 QAIRT SDK 的转换主机要求 x86_64 Ubuntu 22.04、Python 3.10；Q8B ARM64 主要负责板端推理，不要把完整 SDK 转换工具强行装到板上。

第 1 天先确认系统镜像和 NPU 设备，第 14 天必须完成 Radxa 的 `fastrpc` 快速验证和 QAI AppBuilder NPU 示例，第 15 天再把 NPU 推理嵌入 ROS2 图像节点。NPU 验证失败时，课程可以用 CPU 继续学习，但不能把课程标记为完成。

官方 NPU 资料：

- 总览：https://docs.radxa.com/dragon/q8b/app-dev/npu-dev
- 板端启用 NPU：https://docs.radxa.com/dragon/q8b/app-dev/npu-dev/fastrpc-setup
- NPU 快速验证：https://docs.radxa.com/dragon/q8b/app-dev/npu-dev/quick-example
- NPU 开发指南：https://docs.radxa.com/dragon/q8b/app-dev/npu-dev/qairt-sdk
- QAI AppBuilder：https://docs.radxa.com/dragon/q8b/app-dev/npu-dev/qai-appbuilder
- QAI AppBuilder YOLOv8：https://docs.radxa.com/dragon/q8b/app-dev/npu-dev/qai-appbuilder-demo/yolov8-det
- QNN Execution Provider：https://docs.radxa.com/dragon/q8b/app-dev/npu-dev/qnn-onnxrt-execution-provider

## 重要的版本策略

你的 Q8B 资料显示系统是 Ubuntu 26.04 ARM64。ROS2 二进制包并不是对每个 Ubuntu 版本都立即提供。以 Jazzy 官方安装页为例，它的 deb 包页面明确面向 Ubuntu 24.04 Noble，不能把 `jazzy` 盲目写进 26.04 的 apt 源。

每天第 1 天先检查官方支持矩阵：

- ROS2 官方发行版与支持平台：https://docs.ros.org/en/rolling/Releases.html
- ROS2 官方 Ubuntu deb 安装页：https://docs.ros.org/en/jazzy/Installation/Ubuntu-Install-Debs.html
- ROS2 官方教程目录：https://docs.ros.org/en/jazzy/Tutorials.html
- ROS2 官方文档源码（便于在网页打不开时查阅）：https://github.com/ros2/ros2_documentation/tree/jazzy/source

规则如下：

1. 如果官方页面明确支持 Q8B 当前 Ubuntu 26.04，就用原生 apt 安装，并把实际发行版名字写入当天记录。
2. 如果官方页面没有 26.04 的 deb 包，优先使用与 ARM64 兼容的官方 ROS Docker 镜像；不要把 24.04 的源强行改成 26.04。
3. 如果 Docker 的图形界面、摄像头或串口透传遇到额外困难，后续 GUI 和硬件练习应迁移到官方支持的 Ubuntu 版本或原生安装环境。
4. `<ROS_DISTRO>` 是占位符，不是要原样输入的命令。先执行 `echo $ROS_DISTRO` 或查官方页面，再替换。
5. ROS2 Docker 容器默认不会自动获得 Q8B NPU；NPU 需要 `/dev/fastrpc-*`、DSP 库和正确的 `ADSP_LIBRARY_PATH`。优先在 NPU 已启用的 Q8B 宿主机环境运行，只有确实需要时才使用 Radxa QAIRT ARM64 镜像和 `--privileged -v /dev:/dev`。

## 每天固定的记录方法

在 Q8B 上建立课程目录：

```bash
mkdir -p ~/q8b_ros2_course/notes
cd ~/q8b_ros2_course
pwd
```

- `mkdir` 是创建目录；`-p` 表示父目录不存在时一起创建，目录已存在也不报错。
- `~` 代表当前用户的家目录；不要把 `~` 和 `/` 混淆。
- `cd` 进入目录；`pwd` 打印当前绝对路径，用来防止“在错误目录删文件”。

每天把操作、输出、错误和思考写入 `notes/dayXX.md`。可以用 `nano notes/day01.md`：`Ctrl+O` 保存，回车确认文件名，`Ctrl+X` 退出。

## 终端约定

- **终端 A**：启动节点或驱动，保持运行。
- **终端 B**：执行 `ros2 topic echo`、`ros2 node list` 等观察命令。
- **终端 C**：编译、修改代码、查看日志。
- 新开终端后，先执行 ROS2 本体和工作空间的 `source`；后文每次都解释原因。
- `Ctrl+C` 是停止当前前台程序，不是关闭终端。
- 看到命令中的 `$`、`#` 只是提示符，不要复制；代码块里的注释可以不输入。

## 常见安全规则

- 永远先 `pwd`、再 `ls`、再删除；课程中只删除明确的练习目录。
- 不要执行 `sudo rm -rf /`、`sudo rm -rf ~` 或带有不确定变量的删除命令。
- ROS2 工作空间的 `build/`、`install/`、`log/` 是可再生成的产物；`src/` 是源码，不能随便删。
- 不要用 `sudo colcon build`。这样会让生成文件归 root 所有，普通用户之后无法编译。
- 终端遇到错误时，先复制完整错误、记录执行目录和 `echo $ROS_DISTRO`，不要连续盲目重试。

## 15 天地图

| 天数 | 主线 | 当天产物 |
|---|---|---|
| 1 | Linux、硬件盘点、ROS2 版本决策 | 安全练习目录、环境报告 |
| 2 | ROS2 安装、source、官方 demo、CLI | talker/listener 验证记录 |
| 3 | 工作空间、Python 包、节点与 topic | 第一个自定义包 |
| 4 | topic、QoS、参数、重映射 | 可配置的摄像头前置模拟节点 |
| 5 | service、action、interface | 服务和动作客户端/服务器 |
| 6 | launch、YAML、命名空间、组件概念 | 一键启动的多节点系统 |
| 7 | 日志、ros2 doctor、rqt、rosbag、调试 | 可复现的故障报告和 bag |
| 8 | USB 摄像头、V4L2、image_tools | 真实图像 ROS2 话题 |
| 9 | cv_bridge、OpenCV、图像处理 | 图像处理节点和 FPS 测量 |
| 10 | 串口 Linux 基础与 ROS2 接口 | 可选串口桥接节点 |
| 11 | TF2、URDF、RViz2 | camera_link 坐标树与可视化 |
| 12 | Gazebo 仿真与 ros_gz_bridge | 仿真相机认识和替代方案 |
| 13 | bag 回放、生命周期、QoS 深入、测试 | 可重复实验流程 |
| 14 | Q8B NPU、QAIRT、QAI AppBuilder、QNN EP | `fastrpc` 通过，QAI AppBuilder NPU 示例通过 |
| 15 | NPU ROS2 节点、摄像头闭环、性能、文档、清理重建 | 摄像头 -> ROS2 -> NPU -> 结果/串口 |

每一天的文件都可以单独打开学习，但建议严格按顺序进行。

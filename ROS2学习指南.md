# ROS2 从零开始学习指南（Q8B 实战版）

> **详细 15 天版本：** 本文件是原有路线概览；按天动手的新版已拆分到 `ROS2-15天学习指南/`，请从其中的 `README.md` 和 `第01天...第15天...` 开始。新版按每天 5.5～7 小时设计，增加了摄像头、可选串口、调试、仿真、rosbag、Q8B NPU（QAIRT/QAI AppBuilder）和 CPU 对照推理。

> 专为你的情况定制：Linux 文件管理不熟 + Docker 不熟 + 想从零学 ROS2。
> 设备：Q8B（Ubuntu 26.04 / ARM64 / 8核 / 8GB），远程用 RustDesk 或 SSH。
> 本文件保留的是旧版长期概览；如果按天学习，请使用同目录下 `ROS2-15天学习指南/` 的新版课程。

---

## 〇、路线总览（先看懂地图再上路）

```
第1周      Linux 基础补课（配合《Linux常用命令速查手册.md》，重点：文件管理/权限/source）
第2周      安装 ROS2 + 跑通第一个 demo（talker/listener、小乌龟）
第3~4周    自己写包：topic 发布订阅 → service → parameter → launch
第2个月    TF2 坐标变换 + URDF + rviz2 可视化 + 传感器数据接入
第3个月    选方向深入：Nav2 导航 / MoveIt 机械臂 / Gazebo 仿真
```

**ROS2 是什么**：Robot Operating System 2 —— 名字带"操作系统"但它**不是**操作系统，
是跑在 Linux 上的**机器人中间件/框架**：提供进程间通信（话题/服务）、坐标变换、可视化、
包管理等能力，你用它把各个模块（驱动、感知、规划）拼成完整机器人程序。
ROS2 相比 ROS1：去中心化（无 master）、基于 DDS（实时性更好）、支持多机自动发现。

**权威资源**（按优先级）：
1. 官方教程 https://docs.ros.org（Tutorials → Beginner: CLI tools / Client Interfaces，有中文版切换）
2. B站中文视频：赵虚左《ROS2入门理论与实战》、古月居 ROS2 系列
3. 鱼香ROS（fishros.com）—— 中文社区 + 一键安装脚本（新手友好）
4. 书：《ROS 2 机器人开发从入门到实践》
5. 遇到报错先 `ros2 doctor`，再搜 GitHub Issues / ROS Answers

---

## 一、文件管理补课（不搞懂这个，ROS2 处处卡壳）

### 1.1 两个"世界"：系统区 vs 你的家

| 位置 | 是什么 | 你能干嘛 |
|---|---|---|
| `/opt/ros/<发行版名>/` | ROS2 本体，apt 装的，**系统级，只读心态** | 只 source，不改不删 |
| `~/ros2_ws/` | 你的工作空间（ws=workspace），**你写的代码全在这** | 随便折腾，删了重建也就几分钟 |
| `~` = `/home/radxa` | 你的家目录 | 练手圣地 |
| `/etc /boot /usr` | 系统区 | 改动必须 sudo，删错系统报废 |

### 1.2 工作空间目录结构（务必背下来）

```
~/ros2_ws/
├── src/        ← 你唯一手写的目录：你的包源码
│   └── my_first_pkg/
│       ├── my_first_pkg/    ← Python 代码放这
│       ├── package.xml      ← 包的"身份证"（名字/依赖）
│       ├── setup.py         ← 怎么安装你的包
│       └── ...
├── build/      ← colcon 编译中间产物【自动生成，可删，别手改，别提交git】
├── install/    ← 编译结果+环境脚本【自动生成，删了要重新 build，别手改】
└── log/        ← 编译日志【可删】
```

> 记住口诀：**src 是亲生的，build/install/log 是 colcon 生的小弟——删小弟没事（重新编译就有），改小弟白改，别把小弟提交到 git**（写 .gitignore 忽略它们）。

### 1.3 source 的本质（ROS2 新手第一懵）

`source ~/ros2_ws/install/setup.bash` 做的事 = **把一堆路径写进当前终端的环境变量**，
让系统能找到你的包。所以：

- 只对**当前这个终端**有效 → 每开一个新终端都要 source（嫌烦就写进 `~/.bashrc`）
- `ros2 run 找不到我的包` 十有八九是**忘了 source**（或者 source 的是旧的）
- 查验：`echo $AMENT_PREFIX_PATH` 应包含你的 install 路径

```bash
# 一劳永逸：让每个新终端自动加载（先 ROS2 本体，再你的工作空间）
echo "source /opt/ros/<发行版名>/setup.bash" >> ~/.bashrc
echo "source ~/ros2_ws/install/setup.bash" >> ~/.bashrc
source ~/.bashrc     # 立即生效
```

### 1.4 权限铁律

- **编译永远不加 sudo**：`colcon build` 若加 sudo，install 目录变 root 所有，下次普通编译直接报错。发现权限乱了：`sudo chown -R radxa:radxa ~/ros2_ws`
- 装 Python 包用 `pip3 install --user 包名`（装进家目录），别 sudo pip 污染系统
- `rm` 前先 `ls`：养成肌肉记忆。删 build 目录重来是日常操作：
  ```bash
  rm -rf ~/ros2_ws/build ~/ros2_ws/install ~/ros2_ws/log   # 这三个可以放心删
  cd ~/ros2_ws && colcon build                              # 重新编译
  ```

---

## 二、安装 ROS2：两条路线（结合你的 Docker 学习）

> **发行版概念**：ROS2 版本与 Ubuntu 版本一一绑定，两年一个 LTS。
> 历史：Foxy↔20.04、Humble↔22.04、Jazzy↔24.04(LTS)、Kilted↔25.04。
> 你的 Ubuntu 26.04 对应哪个新发行版，安装前到 https://docs.ros.org → Installation → Ubuntu 页面确认（下文用 `<发行版名>` 代替，例如 humble / jazzy）。

### 路线 A：apt 原生安装（简单直接，推荐先走一遍）

```bash
# 1. 添加 ROS2 软件源（具体最新命令以官方 Installation 页为准，大致流程如下）
sudo apt update && sudo apt install curl gnupg lsb-release
sudo curl -LsS https://raw.githubusercontent.com/ros/rosdistro/master/ros.key -o /usr/share/keyrings/ros-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/ros-archive-keyring.gpg] http://packages.ros.org/ros2/ubuntu $(. /etc/os-release && echo $UBUNTU_CODENAME) main" | sudo tee /etc/apt/sources.list.d/ros2.list > /dev/null

# 2. 安装（初学者装 ros-base 就够，desktop 版带 rviz/rqt 等图形工具，Q8B 有桌面就装 desktop）
sudo apt update
sudo apt install ros-<发行版名>-desktop     # 或 ros-<发行版名>-ros-base
sudo apt install ros-dev-tools              # colcon、rosdep 等开发工具

# 3. 环境与依赖
echo "source /opt/ros/<发行版名>/setup.bash" >> ~/.bashrc && source ~/.bashrc
sudo rosdep init && rosdep update           # 依赖自动安装器

# 4. 验证
ros2 run demo_nodes_cpp talker              # 一个终端跑这行
ros2 run demo_nodes_cpp listener            # 另一个终端跑这行，能看到收到消息即成功
```

### 路线 B：Docker 方式（不挑系统版本、干净可复制，顺便练 Docker）

**为什么推荐你学这条**：镜像是官方做好的，Q8B 系统再新也不怕版本不匹配；容器删了重建，
永远不会把系统环境搞乱——这正好治"装环境装崩系统"的病。

#### B1. 先在 Q8B 上装 Docker

```bash
sudo apt update
sudo apt install -y docker.io          # Ubuntu 自带源里就有
sudo usermod -aG docker radxa          # 让 radxa 不用 sudo 也能用 docker
# 重新登录一次（或 sudo newgrp docker）生效
docker --version                       # 出版本号 = OK
```

#### B2. Docker 心智模型（五分钟看懂）

| 概念 | 类比 | 说明 |
|---|---|---|
| **镜像 image** | 类/模板 | 只读，如 `osrf/ros:humble` 就是"装好ROS2的Ubuntu" |
| **容器 container** | 对象/实例 | 镜像跑起来的活进程；删掉容器 = 恢复原样 |
| **卷 volume / -v 挂载** | 外接硬盘 | 把宿主机目录"接"进容器，数据不随容器消失 |
| **--net=host** | 和宿主机同网络 | **ROS2 必备**：DDS 发现机制要求同网段 |
| **-it** | 交互式终端 | 进去敲命令用 |

常用命令速查：

```bash
docker pull osrf/ros:humble                  # 下载镜像（把 humble 换成你的发行版名）
docker images                               # 看本地有哪些镜像
docker run -it --rm --net=host --name rosdev osrf/ros:humble bash   # 创建并进入容器
docker ps                                   # 看正在跑的容器
docker exec -it rosdev bash                 # 再开一个终端进【同一个】容器（多终端必备）
docker stop rosdev && docker rm rosdev      # 停止并删除容器
docker logs rosdev                          # 看容器输出
docker system prune                         # 清理无用镜像/容器（省磁盘）
```

#### B3. ROS2 容器实战（talker/listener 双终端）

```bash
# 终端1：创建并进入容器，把工作空间目录也挂进去（-v 宿主机路径:容器内路径）
docker run -it --rm --net=host -v ~/ros2_ws:/root/ros2_ws --name rosdev osrf/ros:humble bash
ros2 run demo_nodes_cpp talker

# 终端2：进入同一个容器
docker exec -it rosdev bash
ros2 run demo_nodes_cpp listener            # 能收到消息 → 成了！
```

> 以后写代码：宿主机（RustDesk 桌面用 VS Code 写 `~/ros2_ws`）↔ 容器里编译运行，代码不丢、环境不乱。
> 进阶：容器里跑 rviz2 等图形程序需要转发显示（`-e DISPLAY=$DISPLAY` + 宿主机 `xhost +local:docker`），初学阶段先用路线 A 的桌面版跑 GUI 更省事。

**怎么选**：想快速上手教程 → 路线 A；怕搞乱系统/以后要部署多台设备/想顺便把 Docker 练熟 → 路线 B。两者可共存。

---

## 三、核心概念（用生活类比一次讲透）

| 概念 | 类比 | 一句话 | 典型用途 |
|---|---|---|---|
| **Node 节点** | 员工 | 一个独立进程，干一件事 | 电机驱动节点、相机节点 |
| **Topic 话题** | 广播电台 | 发布者广播，谁想听听谁，**异步** | 传感器数据流、速度指令 |
| **Service 服务** | 前台问询 | 一问一答，**同步等结果** | 触发拍照、查当前状态 |
| **Action 动作** | 快递单 | 长任务：可取消、**带进度反馈** | 导航到点、机械臂运动 |
| **Parameter 参数** | 员工设置 | 节点的配置项，运行时可改 | PID 增益、话题名 |
| **Message 消息** | 数据格式单 | 话题/服务里数据的字段定义 | `geometry_msgs/msg/Twist` |
| **DDS/QoS** | 邮局规则 | 底层通信层；QoS=可靠性/队列设置 | 大小消息的传输质量权衡 |

**自动发现**：同一网络、同一 `ROS_DOMAIN_ID`（默认0）的节点互相可见——
这就是 Q8B 和你电脑（如果也装 ROS2）连同一 WiFi 就能互相通信的原因。

---

## 四、第一周实操清单（跟做即入门）

```bash
# 小乌龟（路线A桌面版；容器版把 demo 换成对应包）
ros2 run turtlesim turtlesim_node          # 终端1：弹出乌龟窗口
ros2 run turtlesim turtle_teleop_key       # 终端2：选中此窗口用方向键开乌龟

# 侦查全家桶（理解"节点-话题"最好的方式）
ros2 node list                            # 有哪些节点
ros2 node info /turtlesim                 # 某节点发布/订阅了什么
ros2 topic list                           # 有哪些话题
ros2 topic echo /turtle1/pose             # 实时看某话题数据（Ctrl+C 退出）
ros2 topic hz /turtle1/pose               # 数据频率
ros2 service list                         # 有哪些服务
ros2 service call /spawn turtlesim/srv/Spawn "{x: 5, y: 5, theta: 0, name: 'turtle2'}"  # 生一只新乌龟
ros2 param list /turtlesim                # 节点参数
ros2 param set /turtlesim background_r 255   # 把背景改红
ros2 interface show geometry_msgs/msg/Twist   # 查消息格式
ros2 doctor                               # 环境体检
```

再跑 `rqt_graph`（可能需 `sudo apt install ros-<发行版名>-rqt-graph`），可视化看到两个节点和话题连线——**能看懂这张图，ROS2 就入门一半了**。

---

## 五、写你自己的第一个包（Python，第三周）

```bash
mkdir -p ~/ros2_ws/src && cd ~/ros2_ws/src
ros2 pkg create --build-type ament_python my_first_pkg --dependencies rclpy std_msgs
```

新建 `~/ros2_ws/src/my_first_pkg/my_first_pkg/talker.py`：

```python
import rclpy
from rclpy.node import Node
from std_msgs.msg import String          # 字符串消息，别人定义好的，直接用

class Talker(Node):                       # 节点就是一个类，继承 Node
    def __init__(self):
        super().__init__('my_talker')     # 节点名（ros2 node list 里看到的）
        self.pub = self.create_publisher(String, 'chat', 10)   # 发布到 chat 话题，队列10
        self.timer = self.create_timer(0.5, self.tick)          # 每0.5秒调一次
        self.i = 0

    def tick(self):                       # 消息内容随时间变化，所以放在方法里
        msg = String(); msg.data = f'hello {self.i}'
        self.pub.publish(msg)
        self.get_logger().info(f'发出: {msg.data}')              # 节点日志
        self.i += 1

def main():
    rclpy.init()                          # 初始化
    rclpy.spin(Talker())                  # 循环等待事件（回调/定时器）
    rclpy.shutdown()

```

在 `setup.py` 的 `entry_points` 里注册（这是"命令名→代码入口"的映射）：

```python
entry_points={
    'console_scripts': [
        'talker = my_first_pkg.talker:main',
    ],
},
```

编译运行：

```bash
cd ~/ros2_ws
colcon build --symlink-install     # --symlink-install：改代码不用重新编译（Python推荐）
source install/setup.bash          # 新终端记得 source（或已写入 .bashrc）
ros2 run my_first_pkg talker       # 跑起来！
ros2 topic echo /chat              # 另一终端验证
```

> 报错排查顺序：① source 了吗？② 包名/节点名打对了吗（Tab 补全防手滑）？
> ③ 编译报错往上翻第一个 error（不是最后一行）；④ `colcon build` 后提示里有 package:xxx 标出哪个包出错。

---

## 六、第二个月：launch / TF2 / URDF / rviz2

- **launch 文件**：一条命令启动一堆节点并配好参数（写 `~/ros2_ws/src/xxx/launch/xxx.launch.py`，用 `ros2 launch 包名 文件名` 启动）
- **TF2**：机器人各部件坐标关系的"家谱树"。入门命令：
  `ros2 run tf2_ros static_transform_publisher 0 0 0.5 0 0 0 base_link camera`（发布 base_link→camera 的固定变换），`ros2 run tf2_tools view_frames` 生成 PDF 看整棵树
- **URDF**：XML 格式描述机器人结构（link 部件 + joint 关节），rviz2 里可视化
- **rviz2**：ROS2 的"3D 显示器"，跑 `rviz2`，加载 TF/机器人模型/传感器数据

学到这，你就能看懂别人的机器人项目结构了。

---

## 七、进阶方向菜单（第3个月起按需选）

| 方向 | 是什么 | 关键词入门 |
|---|---|---|
| 导航 Nav2 | 让机器人自己走到目标点 | Nav2 官方教程（有中文） |
| 机械臂 MoveIt2 | 机械臂规划抓取 | MoveIt2 tutorials |
| 仿真 Gazebo | 无实体机器人先仿真 | Gazebo 官方 ROS2 集成教程 |
| SLAM 建图 | 边走边画地图 | slam_toolbox |
| 视觉 | 摄像头+OpenCV/深度学习 | usb_cam、cv_bridge |
| 多机协同 | 几台设备组网 | ROS_DOMAIN_ID、Discovery Server |

**Q8B 专属贴士**：
- ARM64 偶尔遇到某包没有 arm64 二进制 → `colcon build` 从源码编译即可（内存不够时加 `--executor sequential` 或加 swap）
- 编译大项目时另开终端跑 `btop` 盯内存温度（`t` 看线程，已装好）
- 传感器接入：插上 USB 设备后 `lsusb` 和 `dmesg | tail -20` 确认识别，串口设备一般是 `/dev/ttyUSB0` 或 `/dev/ttyACM0`（权限问题：`sudo usermod -aG dialout radxa` 后重新登录）
- 开发工作流：RustDesk 远程桌面写代码看 rviz2，SSH 终端跑编译（Windows Terminal 可开多个标签连同一设备）

---

## 八、30 天打卡计划（每天 1~2 小时）

| 天 | 内容 | 过关标准 |
|---|---|---|
| 1-3 | 速查手册一~三章练熟（cd/ls/cp/rm/权限） | 不看文档完成"建目录→复制→改名→删除" |
| 4-5 | 管道/重定向/source/环境变量 | 能解释 source 干了什么 |
| 6-7 | 路线A 安装 ROS2 | talker/listener 跑通 |
| 8-10 | 小乌龟 + 侦查命令全家桶 | 能画出 rqt_graph 并讲给别人 |
| 11-13 | 官方 Beginner CLI 教程跟完 | ros2 topic/service/param 全用过 |
| 14-17 | 第一个自己的包（第五章） | talker 跑通，改频率改话题名 |
| 18-20 | 写 subscriber（收 talker 消息并回发） | 双向通信成功 |
| 21-23 | service 服务器/客户端 | 自定义一问一答跑通 |
| 24-26 | parameter + launch 文件 | 一条命令拉起两个节点 |
| 27-28 | Docker 路线跑通 ROS2 容器 | 双终端 talker/listener in 容器 |
| 29-30 | 总复习：删掉 ws 重建+复盘 | 脱稿完成第11-26天的所有操作 |

---

## 九、附录：ROS2 命令速查

```bash
ros2 run <包> <节点>                     # 跑一个节点
ros2 launch <包> <launch文件>            # 跑一组节点
ros2 node list / info <节点>
ros2 topic list / echo <话题> / hz <话题> / bw <话题>
ros2 service list / call <服务> <类型> "{参数}"
ros2 param list <节点> / get / set <节点> <参数> <值>
ros2 interface show <消息类型>
ros2 pkg list / create
ros2 doctor                             # 体检
colcon build --symlink-install           # 编译（在 ws 根目录执行）
colcon build --packages-select my_pkg    # 只编译一个包（省时间）
rosdep install --from-paths src -y       # 自动装依赖（在 ws 根目录）
```

**高频报错速查**：
| 现象 | 九成是 |
|---|---|
| package not found / 没有那个文件 | 忘了 source install/setup.bash |
| 两个节点不通信 | ROS_DOMAIN_ID 不同 / QoS 不匹配 / 名字打错 |
| colcon 编译炸 | 往上翻第一个 error；缺依赖先 rosdep install |
| install 目录全是 root 权限 | 曾经 sudo 过编译 → chown -R radxa:radxa ~/ros2_ws |

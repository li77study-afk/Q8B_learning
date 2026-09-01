# 第 1 天：Linux 基础、Q8B 盘点与 ROS2 版本决策

## 今日目标与时间

总计约 6 小时：

- 1 小时：远程登录、终端、路径和目录安全。
- 1.5 小时：文件创建、复制、移动、删除、权限和管道。
- 1 小时：查看 Q8B 架构、系统版本、内存、温度、USB 设备。
- 1.5 小时：建立 ROS2 版本决策，不安装不匹配的软件源。
- 1 小时：独立复做、写环境报告、口头解释每条命令。

## 0. 开始前的原则

本日还不急着安装 ROS2。Ubuntu 26.04 ARM64 是否有目标发行版的官方 deb 包，必须先看官方页面。ROS2 发行版、Ubuntu 版本和 CPU 架构是三个不同条件，不能只看网上某条安装命令。

## 1. 登录 Q8B 并确认自己在哪里

在 Windows PowerShell 执行：

```powershell
ssh radxa@192.168.43.212
```

- `ssh` 是安全远程登录工具。
- `radxa@192.168.43.212` 表示以 `radxa` 用户登录这台 IP 地址为 `192.168.43.212` 的设备。
- 密码不要写进文档或脚本；输入时屏幕不显示字符是正常的。

进入 Q8B 后执行：

```bash
whoami
pwd
ls -la
echo "$HOME"
```

逐句理解：

- `whoami`：打印当前用户名，确认没有误用 root。
- `pwd`：打印当前工作目录；命令对相对路径的解释都从这里开始。
- `ls`：列出目录；`-l` 显示权限、所有者和大小，`-a` 显示以 `.` 开头的隐藏文件。
- `$HOME` 是环境变量，双引号让路径中若有空格时仍被当成一个整体。

预期 `whoami` 是 `radxa`，`pwd` 类似 `/home/radxa`。如果不是，先不要继续。

## 2. 在家目录建立安全练习区

```bash
mkdir -p ~/q8b_ros2_course/day01/{inbox,work,backup}
cd ~/q8b_ros2_course/day01
pwd
find . -maxdepth 2 -type d -print
```

- `mkdir -p` 创建多级目录。
- `{inbox,work,backup}` 是 Bash 的花括号展开，会变成三个目录名。
- `find .` 从当前目录 `.` 开始查找；`-maxdepth 2` 限制搜索深度；`-type d` 只找目录；`-print` 打印结果。

现在练习创建、复制、改名和安全删除：

```bash
printf 'ROS2 practice\n' > inbox/readme.txt
cp inbox/readme.txt backup/readme-copy.txt
mv backup/readme-copy.txt backup/readme-renamed.txt
ls -l inbox backup
rm -i backup/readme-renamed.txt
rmdir work
```

- `printf` 输出文字；`>` 把标准输出重定向到文件，文件存在时会覆盖。
- `cp` 复制，`mv` 移动或改名。
- `rm -i` 删除前询问，比直接 `rm` 更适合新手。
- `rmdir` 只能删除空目录；它删除 `work` 是因为该目录为空。

再故意建立非空目录并观察失败：

```bash
mkdir work/subdir
printf 'temporary\n' > work/subdir/temp.txt
rmdir work
rm -ri work
```

`rmdir` 失败不是坏事，它说明 Linux 不会替你递归删除内容。`rm -ri` 的 `-r` 表示递归进入目录，`-i` 表示逐项确认。

## 3. 权限与管道练习

```bash
printf '#!/usr/bin/env bash\necho hello-from-script\n' > hello.sh
ls -l hello.sh
./hello.sh
chmod u+x hello.sh
./hello.sh
ls -l hello.sh
```

- 第一行是脚本解释器声明；`env bash` 让系统从 `PATH` 找 Bash。
- `./hello.sh` 表示运行当前目录的文件；当前目录通常不在 `PATH` 中，所以不能只写 `hello.sh`。
- `chmod u+x` 给文件所有者 `u` 增加执行权限 `x`。

练习管道和重定向：

```bash
ls -la | tee listing.txt
grep '^d' listing.txt
wc -l listing.txt
echo "当前用户: $USER" >> notes.txt
cat notes.txt
```

- `|` 把左边命令的输出交给右边命令输入。
- `tee` 一边显示输出，一边写文件。
- `grep '^d'` 只找以 `d` 开头的行，通常是目录权限行；`^` 表示行首。
- `wc -l` 统计行数；`>>` 追加而不是覆盖。

## 4. 盘点 Q8B 系统与架构

```bash
uname -a
uname -m
cat /etc/os-release
dpkg --print-architecture
nproc
free -h
df -h ~
uptime
```

- `uname -m` 看 CPU 架构；课程预期是 `aarch64`。
- `/etc/os-release` 是系统版本文本文件；`cat` 将小文件全部打印。
- `dpkg --print-architecture` 看 Debian/Ubuntu 包架构；预期可能是 `arm64`。
- `nproc` 看可用 CPU 数；`free -h` 用易读单位显示内存；`df -h ~` 看家目录所在分区剩余空间。
- `uptime` 显示运行时间和负载，不等同于 CPU 使用率。

记录输出：

```bash
mkdir -p ~/q8b_ros2_course/notes
{
  echo '# Day 01 environment report'
  date -Is
  whoami
  uname -a
  cat /etc/os-release
  dpkg --print-architecture
  free -h
  df -h ~
} | tee ~/q8b_ros2_course/notes/environment.txt
```

这里 `{ ...; }` 将多条命令的输出合并后交给 `tee`；`date -Is` 生成便于排序的时间。

## 5. 观察温度、进程和 USB

```bash
lsusb
ls /dev/video* 2>/dev/null || true
ls /dev/ttyUSB* /dev/ttyACM* 2>/dev/null || true
ps aux | head -n 5
```

- `lsusb` 列 USB 设备。
- `/dev/video*` 是匹配摄像头设备文件的通配符；`2>/dev/null` 把“没有匹配文件”的错误丢弃。
- `|| true` 让“没有摄像头”不导致整组练习停止。
- `ps aux` 列进程，`head -n 5` 只看前 5 行。

如果已插 USB 摄像头，拔出和插入各执行一次：

```bash
dmesg | tail -n 30
```

`dmesg` 查看内核消息；`|` 把全部消息交给 `tail`，只显示末尾 30 行。若普通用户无权查看 dmesg，不要立刻加 sudo，先记录报错。

## 6. ROS2 官方版本检查

在浏览器打开并阅读“Ubuntu / deb packages / supported platforms”：

- https://docs.ros.org/en/rolling/Releases.html
- https://docs.ros.org/en/jazzy/Installation/Ubuntu-Install-Debs.html
- https://reps.openrobotics.org/rep-2000/

注意 URL 大小写不影响网页，但页面中的发行版名和 Ubuntu codename 必须准确。执行：

```bash
grep '^VERSION\|^UBUNTU_CODENAME\|^NAME' /etc/os-release
apt-cache policy ros-jazzy-ros-base 2>/dev/null || true
```

- `grep` 用正则选择系统名称、版本和 codename。
- `apt-cache policy` 只查询软件包，不安装；如果没有候选版本，不代表可以换源强装。

在 `notes/day01.md` 写下三项结论：系统 codename、架构、官方支持的 ROS2 安装路径。

**决策规则：**

- 官方支持当前 Ubuntu 版本：第 2 天走原生 apt。
- 官方仅支持其他 Ubuntu：第 2 天走 Docker，并把“宿主机 GUI/USB 透传风险”记为后续验证项。
- 不要执行网上把 `noble`、`jammy` 直接替换成其他名字的命令；发行版仓库不是字符串模板。

## 7. 盘点 Q8B NPU 环境

这一步决定后面是否能真正发挥买 Q8B 的意义。Q8B 使用 `SC8280XP`，其 NPU 对应 Hexagon `V68`。执行：

```bash
cat /proc/device-tree/compatible 2>/dev/null | tr '\0' '\n'
ls -l /dev/fastrpc-* 2>/dev/null || true
find /usr/lib/dsp -maxdepth 2 -type f \( -name '*Qnn*' -o -name '*qnn*' \) 2>/dev/null | head -n 30
command -v fastrpc_test || true
```

- `/proc/device-tree/compatible` 是内核暴露的硬件兼容信息；`tr '\0' '\n'` 把不可见的 NUL 分隔符转换成换行，方便阅读。
- `/dev/fastrpc-cdsp` 等设备是用户态访问 DSP/NPU 运行环境的重要入口；没有匹配项时，`2>/dev/null` 隐藏通配符错误。
- `find` 查找 DSP 目录中的库；管道前要注意 `-o` 的优先级，今天只做盘点，不删除任何库。
- `command -v` 只检查命令是否在 PATH 中，不执行它。

把输出保存：

```bash
{
  echo '# Q8B NPU inventory'
  date -Is
  cat /proc/device-tree/compatible 2>/dev/null | tr '\0' '\n' || true
  ls -l /dev/fastrpc-* 2>/dev/null || true
  find /usr/lib/dsp -maxdepth 2 -type f \( -name '*Qnn*' -o -name '*qnn*' \) 2>/dev/null | head -n 30
} | tee ~/q8b_ros2_course/notes/npu-inventory.txt
```

Radxa 文档说明 T2 或更高版本镜像通常已经预装 NPU 运行环境；如果你使用的是通用 Ubuntu ISO，不能因为系统能启动就推断 NPU 可用。把结果分成三类记录：

1. 有 `/dev/fastrpc-*` 和 DSP 库：第 14 天继续板端验证。
2. 设备是 Q8B，但设备节点或库缺失：确认是否刷入 Radxa 官方 Q8B 镜像，不能从网上随便复制 `.so`。
3. 不是 Q8B 的硬件兼容信息：先停止，确认启动盘和设备，避免按错误 SoC 配置 NPU。

## 8. 删除练习目录并验证你确实懂了

```bash
cd ~/q8b_ros2_course
pwd
ls -la day01
rm -ri day01/work
find day01 -maxdepth 2 -print
```

只删除 `day01/work`，保留 `inbox`、`backup` 和记录。`find` 最后用于确认结果，而不是凭感觉认为删除成功。

## 今日验收

- [ ] 能解释绝对路径、相对路径、`~`、`.`、`..`。
- [ ] 能不用查资料完成创建目录、创建文件、复制、改名、`rm -i` 删除。
- [ ] 能解释 `>`, `>>`, `|`, `2>/dev/null`。
- [ ] 能读懂 `ls -l` 的所有者和执行权限。
- [ ] 已记录 Q8B 的 Ubuntu 版本、ARM64 架构、内存和磁盘余量。
- [ ] 已根据官方页面决定原生安装还是 Docker，没有盲装错误 apt 源。
- [ ] 已记录 Q8B NPU 的 `SC8280XP`、Hexagon `V68` 和 `/dev/fastrpc-*` 盘点结果。

## 官方主线

- Linux 命令行基础：https://ubuntu.com/tutorials/command-line-for-beginners
- ROS2 安装与环境：https://docs.ros.org/en/jazzy/Installation.html
- Q8B 规格：https://docs.radxa.com/dragon/q8b
- Q8B NPU 总览：https://docs.radxa.com/dragon/q8b/app-dev/npu-dev

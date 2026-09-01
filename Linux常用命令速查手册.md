# Linux 常用命令速查手册（面向 Q8B 开发）

> 适用于 Q8B（Ubuntu 26.04, ARM64）。从 Windows 用 `ssh radxa@192.168.43.212`（密码 radxa）登录后练习。
> 记住三板斧：`Tab` 键自动补全、`↑↓` 翻历史命令、`命令 --help` 看用法，`man 命令` 看详细手册。

---

## 一、目录操作（去哪儿、建哪儿、删哪儿）

| 命令 | 作用 | 示例 |
|---|---|---|
| `pwd` | 显示当前在哪个目录 | `pwd` → `/home/radxa` |
| `ls` | 列出目录内容 | `ls -l` 详细列表；`ls -a` 含隐藏文件；`ls -lh` 大小带单位 |
| `cd` | 进入目录 | `cd ~/Downloads` 进下载目录 |
| | | `cd ..` 回上一级 |
| | | `cd ~` 或 `cd` 回家目录(/home/radxa) |
| | | `cd -` 回上一次所在的目录 |
| `mkdir` | 新建目录 | `mkdir test`；`mkdir -p a/b/c` 一次建多级 |
| `rmdir` | 删空目录（只能删空的） | `rmdir test` |

## 二、文件操作（增、删、改、查、传）

### 复制 / 移动 / 重命名 / 删除

| 命令 | 作用 | 示例 |
|---|---|---|
| `cp` | 复制 | `cp a.txt b.txt`；`cp -r dir1 dir2`（-r 复制整个目录） |
| `mv` | 移动/重命名 | `mv a.txt /tmp/`；`mv old.txt new.txt` |
| `rm` | **删除（无回收站，删了就没）** | `rm a.txt`；`rm -r dir1` 删目录；`rm -i` 删前逐个确认 |

> ⚠️ **高危命令警告**：
> - `rm -rf /` 或 `sudo rm -rf 目录` 不可撤销，输入前确认三遍路径
> - 永远不要在 root（`/`）目录下执行模糊删除；不确定就先 `ls` 看一眼再删
> - 用 `rm -i` 习惯代替裸 `rm`

### 查看文件内容

| 命令 | 作用 | 示例 |
|---|---|---|
| `cat` | 一次全打印（适合小文件） | `cat /etc/os-release` |
| `less` | 分页查看（大文件必用，`q` 退出） | `less /var/log/syslog` |
| `head` / `tail` | 看头/尾 | `head -20 f.txt`；`tail -f log.txt` **实时跟踪日志** |
| `grep` | 在文件里搜关键字 | `grep -r "error" ~/logs/` 递归搜；`grep -i` 忽略大小写 |
| `find` | 按名字/大小/时间找文件 | `find ~ -name "*.py"`；`find / -size +100M 2>/dev/null` |
| `du` / `df` | 看目录/磁盘占用 | `du -sh ~` 家目录共多大；`df -h` 各分区剩余空间 |
| `nano` | 最简单的编辑器（Ctrl+O 保存, Ctrl+X 退出） | `nano test.txt` |
| `vim` | 专业编辑器（`i` 进入编辑，`Esc` 后 `:wq` 保存退出，`:q!` 不保存退出） | `vim test.txt` |
| `tar` | 打包/解包 | `tar czf a.tar.gz dir/` 打包压缩；`tar xzf a.tar.gz` 解包 |
| `ln` | 建快捷方式 | `ln -s /usr/NX/bin/nxserver ~/nx` |

### Windows ↔ Q8B 互传文件

在 **Windows 的 PowerShell** 里执行（不是在设备上）：

```powershell
scp D:\某文件.zip radxa@192.168.43.212:~/Downloads/    # Windows → 设备
scp radxa@192.168.43.212:~/Logs/log.txt D:\             # 设备 → Windows
```

## 三、文件权限（Linux 的安全核心，重点理解）

`ls -l` 看到的 `-rw-r--r-- 1 radxa radxa 80M ...` 含义：

```
- rw-      r--      r--       radxa  radxa
│  │        │        │         │      │
│  │        │        │         │      └ 所属组
│  │        │        │         └ 所有者
│  │        │        └ 其他人: 只读
│  │        └ 同组用户: 只读
│  └ 所有者自己: 可读可写
└ 文件类型(- 普通文件, d 目录)
```

| 命令 | 作用 | 示例 |
|---|---|---|
| `chmod` | 改权限 | `chmod +x run.sh` 加可执行；`chmod 644 a.txt`；`chmod -R 755 dir/` |
| `chown` | 改所有者 | `sudo chown radxa:radxa a.txt` |
| `sudo` | 用管理员权限执行一条命令（密码 radxa） | `sudo apt update` |
| `whoami` | 我是谁 | → `radxa` |

数字速记：`4=读 2=写 1=执行`，相加组合。`755`=所有者全权、其他人读+执行；`777`=人人全权（**尽量别用**）。

> 系统目录（`/etc /usr /var /boot`）归 root 管，改动必须加 `sudo`。
> 自己家目录 `~` 下的东西随便折腾，出不了大事——**练手请在 ~ 里练**。

## 四、软件安装与卸载（apt / dpkg）

```bash
sudo apt update                 # 刷新软件源列表（装东西前先做）
sudo apt upgrade                # 升级所有已装软件
sudo apt install 包名            # 安装，如 sudo apt install btop
sudo apt remove 包名             # 卸载（保留配置）
sudo apt purge 包名              # 卸载（连配置一起删）
sudo apt autoremove             # 清理没用的依赖
apt search 关键词                # 搜索软件
apt list --installed | grep xx  # 查装没装过某软件

sudo dpkg -i xxx.deb            # 直接安装本地 deb（装 RustDesk 就是这么装的）
sudo dpkg -r 包名                # 卸载 deb 装的软件
```

> 报依赖错误时：`sudo apt --fix-broken install` 或干脆用 `sudo apt install ./xxx.deb`（自动补依赖）。

## 五、系统信息与进程监控

| 命令 | 作用 |
|---|---|
| `uname -a` | 内核/架构信息 |
| `lscpu` | CPU 详情（8 核 ARM） |
| `free -h` | 内存使用 |
| `btop` | **综合监控 TUI（已装）：CPU/温度/内存/网络/进程一屏全有，`q` 退出** |
| `ps aux` | 列出所有进程 |
| `ps aux \| grep rustdesk` | 找某进程的 PID |
| `top` | 简版 btop |
| `kill 1234` | 结束 PID 1234 的进程 |
| `kill -9 1234` | 强杀（进程卡死时用） |
| `pkill -f rustdesk` | 按名字杀进程 |
| `uptime` | 开机多久、平均负载 |

## 六、网络命令

| 命令 | 作用 | 示例 |
|---|---|---|
| `ip a` | 看本机 IP（找 IP 必用） | `ip a` |
| `ip r` | 看网关/路由 | |
| `ping` | 测连通 | `ping -c 3 baidu.com`（-c 3 只 ping 3 次） |
| `ss -tlnp` | 看哪些端口在监听 | `sudo ss -tlnp`（带进程名要 root） |
| `ssh` | 远程登录 | `ssh radxa@192.168.43.212` |
| `scp` | 跨机传文件 | 见第二节 |
| `curl` / `wget` | 下载/请求 | `wget https://xxx/file.tar.gz` |

## 七、服务管理（systemctl，管后台服务）

```bash
systemctl status rustdesk           # 看状态（绿色 active = 正常）
sudo systemctl restart rustdesk     # 重启服务
sudo systemctl stop rustdesk        # 停止
sudo systemctl start rustdesk       # 启动
sudo systemctl enable rustdesk      # 设为开机自启
sudo systemctl disable rustdesk     # 取消开机自启
systemctl list-units --type=service --state=running   # 看所有运行中的服务

journalctl -u rustdesk -f           # 实时跟踪某服务日志（Ctrl+C 退出）
journalctl -u rustdesk -n 50        # 看最近 50 行
```

> 已知服务名举例：`rustdesk`、`gdm3`(桌面登录)、`nxserver`(NoMachine)、`ssh`。

## 八、开关机与定时关机（重点）

```bash
sudo reboot                     # 立即重启
sudo poweroff                   # 立即关机
sudo shutdown now               # 立即关机
sudo shutdown -h 10             # 10 分钟后关机（会广播提醒所有登录用户）
sudo shutdown -h 22:30          # 今天 22:30 关机
sudo shutdown -r +30            # 30 分钟后重启
sudo shutdown -c                # 取消已计划的关机
```

### 定时任务（cron，自动化神器）

```bash
crontab -e                      # 编辑当前用户的定时任务（首次会让你选编辑器，选 nano）
```

文件里一行一个任务，格式（分 时 日 月 周）：

```cron
30 22 * * *    /sbin/poweroff               # 每天 22:30 关机（需要 root 权限时改用 sudo crontab -e）
*/5 * * * *   echo hi >> ~/hi.log           # 每 5 分钟执行一次
0 9 * * 1-5   python3 ~/work/report.py      # 周一到周五每天 9:00 跑脚本
```

`crontab -l` 查看已设任务。在线生成器搜 "crontab generator" 可以拼表达式。

### 一次性定时（at）

```bash
echo "sudo poweroff" | at 23:00     # 23:00 执行一次（需 apt install at）
atq                                # 查看队列;  atrm 1 删除
```

## 九、硬件与外设

```bash
lsblk                     # 看所有磁盘/分区（插U盘前后各看一次，多出来的就是它）
lsusb                     # 看 USB 设备
dmesg | tail -20          # 看内核最近日志（拔插设备时看它识别了什么）
sudo mount /dev/sda1 /mnt # 把U盘第一分区挂到 /mnt
sudo umount /mnt          # 卸载（拔U盘前必须先卸载！）
vcgencmd measure_temp 2>/dev/null || cat /sys/class/thermal/thermal_zone16/temp   # 温度（毫摄氏度）
```

## 十、管道与重定向（命令组合的艺术，理解了才算入门）

```bash
命令A | 命令B        # A 的输出作为 B 的输入
ps aux | grep ros    # 例：从进程列表里筛 ros
history | grep ssh   # 例：找我用过的 ssh 命令
命令 > 文件           # 输出写入文件（覆盖）
命令 >> 文件          # 输出追加到文件末尾
./a.out 2> err.log   # 只把报错信息存到文件
which rustdesk       # 查一个命令到底在哪个路径
```

## 十一、环境变量与 .bashrc（ROS2 学习前必须搞懂）

```bash
echo $PATH            # 查环境变量
export MY_VAR=hello   # 临时设置（关终端就没）
source ~/.bashrc      # 重新加载配置
echo "alias ll='ls -lh'" >> ~/.bashrc && source ~/.bashrc   # 永久加一个别名
```

> **为什么重要**：ROS2 每次开终端都要 `source ~/ros2_ws/install/setup.bash`，
> 原理就是往环境变量里加路径。看懂这节，ROS2 的"source 来 source 去"就不懵了。

## 十二、必懂的目录结构（删东西前心里有地图）

```
/home/radxa     ← 你的家，随便折腾
/etc            ← 全系统配置文件（sudo 才能改，删错开不了机）
/usr            ← 已安装软件（如 /usr/NX）
/opt            ← 第三方大软件默认安装处（ROS2 装在 /opt/ros/）
/var/log        ← 日志
/boot           ← 内核与启动文件（u-boot 的 extlinux.conf 在这里）
/tmp            ← 临时文件（重启可能清空）
/dev            ← 设备文件（磁盘、串口在这）
/proc /sys      ← 内核运行时信息（温度、进程状态）
```

---

## 附：自查清单（全部会做 = 具备手动开发能力）

- [ ] 能 cd 到任意目录并 pwd 确认位置
- [ ] 会 cp/mv/rm，且知道 rm -rf 的风险
- [ ] 会 find/grep 在系统里找东西
- [ ] 看得懂 `ls -l` 的权限位，会用 chmod +x
- [ ] 会 apt install/remove/upgrade
- [ ] 会 systemctl status/restart，会 journalctl 看日志
- [ ] 会 ip a 查 IP、scp 和 Windows 互传文件
- [ ] 会定时关机 + crontab
- [ ] 懂 `|`、`>`、`source`、环境变量
- [ ] 知道哪些目录不能乱删

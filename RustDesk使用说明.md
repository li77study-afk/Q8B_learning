# Q8B 远程桌面使用说明（Windows ↔ Q8B）

> 2026-08-31 重要更新：Ubuntu 26.04 只有 Wayland（无 Xorg 会话），RustDesk 在 GNOME Wayland 下
> 无法无人值守截屏/控制（会黑屏）——这是平台限制不是配置问题。
> **看桌面请用方式一（RDP）**；RustDesk 保留作文件传输用。

## ★ 方式一（主用）：GNOME 无头 RDP —— Windows 自带"远程桌面连接"，零安装

| 项目 | 值 |
|---|---|
| 地址 | `192.168.43.212`（端口 3389，无需填） |
| 用户名 | `radxa` |
| 密码 | `radxa1234` |

**步骤**：`Win+R` 输入 `mstsc` 回车 → 计算机填 `192.168.43.212` → 连接 →
首次会提示证书不信任（自签证书）→ 点"是" → 输入 radxa / radxa1234 → 进入 1920x1080 虚拟桌面。

- 无需 HDMI 诱骗器（无头模式自己生成虚拟显示器，接不接都行）
- 无需任何授权确认，完全无人值守
- 服务：`gnome-remote-desktop-headless.service`（已设开机自启，随自动登录会话拉起）
- 剪贴板/文件拖拽：mstsc 里勾选本地资源即可（选项 → 本地资源 → 剪贴板/驱动器）

设备端管理：
```bash
systemctl --user status gnome-remote-desktop-headless    # 状态
systemctl --user restart gnome-remote-desktop-headless  # 重启
journalctl --user -u gnome-remote-desktop-headless -f   # 看日志
# 改密码：
export XDG_RUNTIME_DIR=/run/user/1000 DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/1000/bus
grdctl --headless rdp set-credentials radxa 新密码 && systemctl --user restart gnome-remote-desktop-headless
```

---

## 方式二（辅助）：RustDesk 1.4.9 —— 用于**文件传输**（看桌面在本机不可用）

| 项目 | 值 |
|---|---|
| 设备 ID | `1418389310` |
| 固定密码 | `radxa1234` |
| 设备 IP | `192.168.43.212`（热点分配，可能变化） |
| 直连端口 | `21118`（IP 直连推荐；ID 模式走海外公共服务器，国内不稳） |

**文件传输**：打开 `D:\RustDesk\rustdesk.exe` → 顶部切到"文件传输"标签 →
ID 框填 `192.168.43.212` → 密码 `radxa1234` → 双向文件管理器，可拖拽上传下载。

> RustDesk 何时能看桌面：接上真实屏幕（或有本地输入）的场景、或未来 RustDesk 完整支持
> GNOME Wayland 无人值守时。当前系统上"远程桌面"标签连上会黑屏，属正常现象。

## 设备端 RustDesk 管理命令（SSH 到设备执行）

```bash
ssh radxa@192.168.43.212        # 密码 radxa

sudo rustdesk --password 新密码              # 修改固定密码
sudo rustdesk --get-id                      # 查看 ID
sudo rustdesk --option direct-server Y      # 开启IP直连（已开启）
systemctl status rustdesk                   # 服务状态
sudo systemctl restart rustdesk             # 重启服务
```

## 六、注意事项

1. **IP 变了连不上**：设备重启后 IP 可能变化，在设备上 `ip a` 查看。RDP 与 RustDesk 都用 IP 连，IP 变了要改。
2. **黑屏说明**（已定位）：本系统为 Wayland-only（`/usr/share/xsessions` 为空，无 "Ubuntu on Xorg" 选项可切）。
   GNOME Wayland 下第三方软件（RustDesk）无法无人值守截屏/注入键鼠，属平台限制 → 已改用方式一 RDP 看桌面。
3. **无人值守**：RDP 与 RustDesk 均为固定密码，无需在设备端确认，重启后自动就绪。
4. RustDesk 完全免费开源；IP 直连模式下不经任何外部服务器。
5. **小屏幕（1024x600，Lontium 桥接板）偶发黑屏**：开机画面正常但进桌面黑屏时，
   **重新插拔一次 HDMI 即可恢复**（DRM 驱动与屏桥接芯片的 HPD 握手偶发不同步）。应急可随时用 RDP。

## 七、无屏幕（Headless）配置记录

1. **GDM 自动登录**（`/etc/gdm3/custom.conf`）：
   ```ini
   [daemon]
   AutomaticLoginEnable=True
   AutomaticLogin=radxa
   ```
   无屏幕重启后自动进入 radxa 桌面，不再卡在登录界面。备份：`/etc/gdm3/custom.conf.bak`

2. **内核强制 HDMI 输出**（`/etc/kernel/cmdline` 追加了）：
   ```
   video=HDMI-A-1:1920x1080@60D
   ```
   实测：高通 DRM 驱动不采纳该强制参数（无诱骗器时 enabled 仍为 disabled）。**此参数实际无效**，
   无屏幕需求已由"无头 RDP 虚拟显示器"和 HDMI 诱骗器（已插，识别为 4K）覆盖。想清理可删除该参数后
   `sudo u-boot-update`。已执行过 `u-boot-update` 写入 `/boot/extlinux/extlinux.conf`。
   备份：`/etc/kernel/cmdline.bak`

> 如需撤销自动登录：恢复 `/etc/gdm3/custom.conf.bak` 后重启。

## 八、btop —— TUI 实时监控（已安装 v1.4.6）

SSH 进设备后直接运行：

```bash
ssh radxa@192.168.43.212
btop
```

可以看到：8 个 CPU 核心实时占用、每核频率、**温度**（CPU 大小核/GPU/板边等 50+ 传感器自动挑选显示）、内存、磁盘 IO、网络速率、进程/线程列表。

常用按键：
| 按键 | 功能 |
|---|---|
| `1` | 只看 CPU 面板（放大） |
| `2` / `3` / `4` | 内存 / 网络 / 进程 面板 |
| `p` / `t` | 进程列表按 进程/线程 显示（**看线程按 t**） |
| `f` | 进程筛选，`esc` 退出筛选 |
| `q` 或 `Esc` | 退出 btop |

温度快速看一眼（不用 btop）：
```bash
cat /sys/class/thermal/thermal_zone{16,17,18,19}/temp   # cpu0~3, 单位毫摄氏度
sensors 2>/dev/null || sudo apt install lm-sensors
```

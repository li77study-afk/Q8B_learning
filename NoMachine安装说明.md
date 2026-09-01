# NoMachine 远程桌面安装说明（radxa 设备）

## 一、设备信息

| 项目 | 内容 |
|---|---|
| 主机名 | radxa-dragon-q8b |
| 系统 | Ubuntu 26.04 LTS (aarch64 / ARM64) |
| IP 地址 | 192.168.43.212（手机热点 DHCP 分配，可能会变） |
| SSH 账号 / 密码 | radxa / radxa |
| SSH 命令 | `ssh radxa@192.168.43.212` |
| NoMachine 服务端口 | 4000（NX 协议，已监听 0.0.0.0） |

## 二、安装过程记录

1. 设备原本下载的是 `nomachine-personal-edition_10.0.59_1_amd64.deb`，是 **x86/amd64** 架构的包，与设备 ARM64 架构不匹配，无法安装（该文件仍在 `~/Downloads`，已无用可删除）。
2. 已从 NoMachine 官网下载正确的 aarch64 版本：
   `nomachine-personal-edition_10.0.59_3_aarch64.tar.gz`（存于设备 `~/Downloads`）。
3. 按官方 README 流程安装：
   ```bash
   sudo cp -p nomachine-personal-edition_10.0.59_3_aarch64.tar.gz /usr
   cd /usr && sudo tar zxf nomachine-personal-edition_10.0.59_3_aarch64.tar.gz
   sudo /usr/NX/nxserver --install
   ```
4. 安装完成，`nxserver` 服务已设为开机自启且当前运行中（systemctl 状态 active），端口 4000 已开放。

## 三、Windows 电脑如何连接（重点）

1. 在 Windows 上下载并安装 NoMachine 客户端（免费）：
   https://www.nomachine.com/download（选 Windows x86 版本，安装包为 exe，一路下一步即可）。
2. 打开 NoMachine 客户端 → 出现欢迎界面 → **Add**（添加连接）：
   - Protocol（协议）：NX
   - Host（主机）：`192.168.43.212`
   - Port（端口）：`4000`
   - 其余默认，点 Add 保存。
3. 双击该连接 → 首次连接会进行环境检测，一路 Continue → 输入用户名 `radxa`、密码 `radxa` → 即可看到设备桌面（GNOME）。
4. 之后在局域网（同一手机热点）内随时双击连接即可。

> 注意：设备的 IP 由热点 DHCP 分配，若重启后连不上，可在设备上执行 `ip a` 查看最新 IP，并更新连接设置。

## 四、设备端常用管理命令（需 sudo，注意安装后命令在 /usr/NX/bin/ 下）

```bash
sudo /usr/NX/bin/nxserver --status      # 查看服务状态
sudo /usr/NX/bin/nxserver --restart     # 重启服务
sudo /usr/NX/bin/nxserver --stop        # 停止服务
sudo /usr/NX/bin/nxserver --start       # 启动服务
sudo /usr/NX/bin/nxserver --subscription                      # 查看订阅/评估状态
sudo /usr/NX/bin/nxserver --subscription --activate=<许可证KEY>  # 激活许可证
sudo /usr/NX/bin/nxserver --uninstall   # 卸载 NoMachine
```

- 日志位置：`/usr/NX/var/log/`
- 安装目录：`/usr/NX/`

## 五、重要：许可证问题（v10 起收费，连接被拒的原因）

**NoMachine 从 v10 开始取消了免费版**，Personal Edition 变为商业订阅产品（官网原话：
"While NoMachine was previously free for non-commercial use, starting with version 10 the
entry-level free edition has been discontinued"）。

服务器未激活许可证时，客户端连接会被直接拒绝，提示：
`No subscription found on this server. Please contact NoMachine to acquire a valid subscription.`

设备端当前状态（`nxserver --subscription` 查询）：
```
NX> 630 WARNING: No subscription found on this server.
NX> 660  Subscription type: PE.
NX> 660  Subscription id: (null).
```

### 解决方案（三选一）

1. **免费试用 14 天（推荐先体验）**
   1. 打开 https://users.nomachine.com/create-account 注册 NoMachine 账号（免费）
   2. 登录 User Area → Software 区域，为 Personal Edition 生成评估许可证（一串 KEY）
   3. 在设备上激活并重启服务：
      ```bash
      ssh radxa@192.168.43.212
      sudo /usr/NX/bin/nxserver --subscription --activate=<你的KEY>
      sudo /usr/NX/bin/nxserver --restart
      ```
   4. Windows 客户端重新连接即可。14 天后需要付费。

2. **购买订阅**：Personal Edition $24.50/年
   https://store.nomachine.com/product/personal-edition-subscription/

3. **改装免费开源替代品**（不想付费时）：
   - **RustDesk**（开源、跨平台、类 TeamViewer，局域网直连免费）
   - **xrdp**（Linux 端装 xrdp，Windows 用自带的"远程桌面连接 mstsc.exe"直连，零额外客户端）
   - TigerVNC / TigerVNC+SSH 隧道

> 建议：只是临时用 → 方案 1；长期用且要画面流畅 → 方案 2 或 RustDesk；
> 追求零成本零注册 → xrdp。

## 六、其他说明

- 设备桌面环境为 GNOME（GDM 已启用），NoMachine 连接后会自动创建/接管桌面会话。
- Windows 与设备必须在同一网络（当前为 192.168.43.x 热点网段）。
- 设备 `~/Downloads` 里的 `nomachine-personal-edition_10.0.59_1_amd64.deb` 是 x86 架构的错误包，已无用，可删除。

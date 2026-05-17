# SSHFS-Win 挂载树莓派目录流程

本文记录 Windows 通过 SSHFS-Win 将树莓派目录挂载为本地盘符的流程。

本次使用的树莓派信息：

```text
SSH 用户：dsp
树莓派 IP：192.168.137.155
树莓派目录：/home/dsp/Desktop
Windows 盘符：Z:
```

挂载完成后，Windows 的 `Z:\` 就对应树莓派上的：

```text
dsp@192.168.137.155:/home/dsp/Desktop
```

## 1. 前置条件

树莓派需要已经连接到电脑热点或同一局域网，并且 SSH 服务已开启。

可以先在 Windows PowerShell 中测试树莓派 SSH 端口是否通：

```powershell
Test-NetConnection -ComputerName 192.168.137.155 -Port 22
```

如果输出里看到：

```text
TcpTestSucceeded : True
```

说明 Windows 能访问树莓派的 SSH 端口。

也可以直接测试 SSH 登录：

```powershell
ssh dsp@192.168.137.155
```

第一次连接可能会提示确认指纹，输入 `yes`，然后输入树莓派用户 `dsp` 的密码。

## 2. 安装 WinFsp 和 SSHFS-Win

SSHFS-Win 依赖 WinFsp，需要先安装 WinFsp，再安装 SSHFS-Win。

在 Windows PowerShell 中执行：

```powershell
winget install --id WinFsp.WinFsp -e --accept-package-agreements --accept-source-agreements
```

然后安装 SSHFS-Win：

```powershell
winget install --id SSHFS-Win.SSHFS-Win -e --accept-package-agreements --accept-source-agreements
```

安装完成后，可以检查 SSHFS-Win 是否存在：

```powershell
Test-Path "C:\Program Files\SSHFS-Win\bin\sshfs.exe"
```

如果返回 `True`，说明安装成功。

## 3. 检查 Z 盘是否空闲

挂载前可以检查 `Z:` 盘有没有被占用：

```powershell
Get-PSDrive -Name Z -ErrorAction SilentlyContinue
```

如果没有输出，说明 `Z:` 没被占用。

## 4. 挂载树莓派目录到 Z 盘

在 Windows PowerShell 中执行：

```powershell
net use Z: \\sshfs.r\dsp@192.168.137.155\home\dsp\Desktop /user:dsp * /persistent:yes
```

说明：

- `Z:` 是 Windows 本地盘符。
- `\\sshfs.r\dsp@192.168.137.155\home\dsp\Desktop` 对应树莓派路径 `/home/dsp/Desktop`。
- `/user:dsp` 表示使用树莓派用户 `dsp`。
- `*` 表示让 Windows 提示输入密码。
- `/persistent:yes` 表示持久保存映射，下次 Windows 登录后仍会记住这个盘符。

执行后输入树莓派用户 `dsp` 的密码。

成功后，Windows 资源管理器里会出现 `Z:` 盘。

## 5. 验证挂载是否成功

查看 `Z:` 映射状态：

```powershell
net use Z:
```

如果看到类似内容，说明挂载成功：

```text
Local name        Z:
Remote name       \\sshfs.r\dsp@192.168.137.155\home\dsp\Desktop
Resource type     Disk
The command completed successfully.
```

也可以列出目录内容：

```powershell
Get-ChildItem Z:\
```

如果能看到树莓派桌面上的文件，例如 `.idea`、`hello.py`、`raspberry_pi_esp8266_bridge.py` 等，就说明挂载正常。

## 6. 下次使用

因为挂载时使用了：

```powershell
/persistent:yes
```

所以下次 Windows 登录后通常还会记住 `Z:` 盘。

但树莓派需要满足：

- 已开机。
- 已连上电脑热点或同一局域网。
- IP 仍然是 `192.168.137.155`。
- SSH 服务正常运行。

如果树莓派刚开机时还没连上网络，`Z:` 可能暂时打不开。等树莓派联网后，再打开一次 `Z:`，Windows 通常会自动重连。

手动检查：

```powershell
net use Z:
```

手动重新挂载：

```powershell
net use Z: \\sshfs.r\dsp@192.168.137.155\home\dsp\Desktop /user:dsp * /persistent:yes
```

## 7. 取消挂载

如果需要取消 `Z:` 盘映射：

```powershell
net use Z: /delete
```

如果之后想重新挂载，再执行第 4 步的挂载命令。

## 8. 固定树莓派 IP

为了让 `Z:` 盘长期稳定，建议让树莓派固定使用：

```text
192.168.137.155
```

电脑热点的网关通常是：

```text
192.168.137.1
```

新版 Raspberry Pi OS 通常使用 NetworkManager。可以在树莓派终端中执行：

```bash
nmcli con show --active
```

找到当前 Wi-Fi 连接名后，执行：

```bash
sudo nmcli con mod "你的WiFi连接名" ipv4.addresses 192.168.137.155/24
sudo nmcli con mod "你的WiFi连接名" ipv4.gateway 192.168.137.1
sudo nmcli con mod "你的WiFi连接名" ipv4.dns "192.168.137.1 8.8.8.8"
sudo nmcli con mod "你的WiFi连接名" ipv4.method manual
sudo reboot
```

重启后检查：

```bash
ip addr show wlan0
```

如果看到 `192.168.137.155/24`，说明固定 IP 成功。

旧版 Raspberry Pi OS 可能使用 `dhcpcd`。可以编辑：

```bash
sudo nano /etc/dhcpcd.conf
```

在末尾加入：

```conf
interface wlan0
static ip_address=192.168.137.155/24
static routers=192.168.137.1
static domain_name_servers=192.168.137.1 8.8.8.8
```

然后重启：

```bash
sudo reboot
```

## 9. 打开树莓派终端

如果只是想进入树莓派终端，可以在 Windows PowerShell 中执行：

```powershell
ssh dsp@192.168.137.155
```

登录后进入桌面目录：

```bash
cd /home/dsp/Desktop
ls
```

运行 Python 文件示例：

```bash
python3 hello.py
```

## 10. 常见问题

### Z 盘显示但打不开

常见原因：

- 树莓派没开机。
- 树莓派还没连上电脑热点。
- 树莓派 IP 变了。
- SSH 服务没有启动。

先测试：

```powershell
Test-NetConnection -ComputerName 192.168.137.155 -Port 22
```

如果 `TcpTestSucceeded` 不是 `True`，说明 Windows 暂时连不到树莓派 SSH。

### 提示密码错误

重新挂载并手动输入密码：

```powershell
net use Z: /delete
net use Z: \\sshfs.r\dsp@192.168.137.155\home\dsp\Desktop /user:dsp * /persistent:yes
```

### 树莓派 IP 变了

先找到新的 IP，再重新挂载。

如果树莓派连接的是 Windows 热点，可以在 Windows PowerShell 中查看局域网 ARP 记录：

```powershell
arp -a
```

找到树莓派的新 IP 后，把挂载命令中的 `192.168.137.155` 替换成新的 IP。

### Codex 不能交互输入 SSH 密码

Codex 可以运行 SSH、SCP、检查文件和执行命令，但如果 SSH 需要交互输入密码，可能会卡住或失败。

解决方式：

- 先由用户在 PowerShell 中完成 `net use` 挂载。
- 或者给树莓派配置 SSH key 免密登录。

挂载完成后，Codex 可以直接访问 `Z:\` 下的树莓派文件。


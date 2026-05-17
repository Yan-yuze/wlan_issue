# 树莓派连 Windows 热点 — 固定 IP 配置指南

## 网络拓扑

| 角色 | 地址 |
|------|------|
| Windows 热点网关 | `192.168.137.1` |
| 树莓派固定 IP | `192.168.137.155/24` |
| 网卡 | `wlan0` |

---

## 方法一：NetworkManager（推荐，适合新版 Raspberry Pi OS）

### 第一步：查看当前活跃的 Wi-Fi 连接名

```bash
nmcli con show --active
```

找到 TYPE 为 `wifi` 的那一行，记住 NAME 列的连接名（例如 `MyHotspot`）。

### 第二步：设置固定 IP（把连接名替换为实际名称）

```bash
sudo nmcli con mod "你的WiFi连接名" ipv4.addresses 192.168.137.155/24
sudo nmcli con mod "你的WiFi连接名" ipv4.gateway 192.168.137.1
sudo nmcli con mod "你的WiFi连接名" ipv4.dns "192.168.137.1 8.8.8.8"
sudo nmcli con mod "你的WiFi连接名" ipv4.method manual
```

### 第三步：重启生效

```bash
sudo reboot
```

### 第四步：重启后验证

```bash
ip addr show wlan0
```

看到 `192.168.137.155/24` 即表示成功。

---

## 方法二：dhcpcd（旧版系统备用方案）

> 如果系统没有 `nmcli` 命令，则使用此方法。

### 编辑配置文件

```bash
sudo nano /etc/dhcpcd.conf
```

在文件**末尾**追加以下内容：

```
interface wlan0
static ip_address=192.168.137.155/24
static routers=192.168.137.1
static domain_name_servers=192.168.137.1 8.8.8.8
```

保存退出（`Ctrl+O` → `Enter` → `Ctrl+X`），然后重启：

```bash
sudo reboot
```

---

## 注意事项

- **不要**把树莓派设为 `192.168.137.1`，这是 Windows 热点本身的地址。
- `192.168.137.155` 可继续使用，只要热点下没有其他设备占用此 IP。
- 固定 IP 配置完成后，`Z:` 盘网络映射将更加稳定。

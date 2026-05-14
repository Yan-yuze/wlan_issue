# Windows 代理异常排查指南

本文档总结了用户在使用小旋风 Cloud 代理软件时遇到的典型问题及排查方法，特别针对“系统代理关闭但网络仍走127.0.0.1:6294”以及登录失败/HTTPS TLS错误的情况，方便在下一次出现类似问题时快速定位并处理。

---

## 1. 背景问题

用户症状可能包括：

- 小旋风 Cloud 节点测速失败，无法翻墙
- 浏览器或 curl 命令请求失败
- 登录界面显示“网络错误”
- 之前可用节点突然不可用
- 系统代理已经关闭，但仍有流量走 127.0.0.1

经过分析，核心原因大多是**残留的环境变量代理**或**小旋风核心异常**。

---

## 2. 核心原理

Windows 下代理流量主要入口有三类：

1. **系统代理（WinINET）**：
   - 设置路径：设置 -> 网络和 Internet -> 代理
   - 注册表：`ProxyEnable` / `ProxyServer`

2. **WinHTTP 代理**：
   - 命令行查看/重置：`netsh winhttp show/reset proxy`

3. **环境变量代理**：
   - `HTTP_PROXY` / `HTTPS_PROXY` / `ALL_PROXY`
   - 许多命令行工具和开发工具会优先读取环境变量代理，可能导致系统代理关闭后仍走127.0.0.1端口

4. **本地代理核心**：
   - 小旋风 Cloud / AtlasCore
   - 本地端口：127.0.0.1:6294
   - 负责将流量转发到远程节点 / 外网

---

## 3. 排查思路

1. **检查系统代理**
```powershell
reg query "HKCU\Software\Microsoft\Windows\CurrentVersion\Internet Settings" /v ProxyEnable
reg query "HKCU\Software\Microsoft\Windows\CurrentVersion\Internet Settings" /v ProxyServer
```

2. **检查 WinHTTP 代理**
```cmd
netsh winhttp show proxy
```

3. **检查环境变量代理**
```powershell
gci env:*proxy*
```

4. **测试本地代理端口**
```powershell
curl -x http://127.0.0.1:6294 https://www.google.com -I
curl -x socks5h://127.0.0.1:6294 https://www.google.com -I
```

5. **如果显示 via 127.0.0.1**
   - 优先清理环境变量代理

6. **清完后重启终端 / 浏览器 / 软件**

---

## 4. 清理环境变量代理

在 PowerShell 中执行：
```powershell
[Environment]::SetEnvironmentVariable("HTTP_PROXY",$null,"User")
[Environment]::SetEnvironmentVariable("HTTPS_PROXY",$null,"User")
[Environment]::SetEnvironmentVariable("ALL_PROXY",$null,"User")
Remove-Item Env:HTTP_PROXY -ErrorAction SilentlyContinue
Remove-Item Env:HTTPS_PROXY -ErrorAction SilentlyContinue
Remove-Item Env:ALL_PROXY -ErrorAction SilentlyContinue
```

> 清理后重新打开 PowerShell / 浏览器 / 小旋风客户端，网络和登录应恢复正常。

---

## 5. 核心异常处理

1. 检查小旋风核心进程：
```powershell
tasklist | findstr /i "AtlasCore_amd64.exe"
```

2. 如果无法终止，使用管理员权限 CMD：
```cmd
taskkill /f /t /im AtlasCore_amd64.exe
```

3. 如仍无法结束，检查系统服务是否残留或进入安全模式处理。

---

## 6. 注意事项

- 系统代理关闭不代表所有程序都不走代理，环境变量代理可能覆盖系统代理
- curl / PowerShell / Git / Node.js / Python 等工具会优先读取环境变量代理
- 每次卸载或重装小旋风前，最好检查并清理环境变量
- 系统时间、Hosts 文件、杀软/防火墙也可能影响小旋风登录和 HTTPS
- 使用手机热点测试网络可区分本地网络问题与运营商限制

---

## 7. 总结

本次问题核心原因：**残留环境变量代理 + 小旋风核心异常**，排查方法依次是：

1. 清理系统代理（ProxyEnable / ProxyServer）
2. 重置 WinHTTP 代理
3. 清理环境变量代理（HTTP_PROXY / HTTPS_PROXY / ALL_PROXY）
4. 检查本地代理端口是否可用
5. 强制关闭小旋风核心进程 / 重装
6. 确认 DNS、Hosts、时间、杀软防火墙不干扰

这样整理好以后，出现同样情况，直接给 AI 或按步骤操作即可快速排查和恢复网络。


# 这里是我曾经遇到的问题，解决之后总结成markdown文件，方便下次出现类似问题后可以丢给ai让他自己执行，并帮我解决问题。


## windows_proxy_troubleshooting_guide.md
这是小旋风客户端与codex的环境变量对冲造成的网络瘫痪
具体表现就是小旋风客户端无网络，无法登录，但能直连，曾经出现过完美世界，5e客户端无法登录，找不到我的steam账号的情况


## Codex聊天记录恢复与应急使用指南.md
这是我codex在运行
```powershell
Get-Process Codex,codex -ErrorAction SilentlyContinue | Stop-Process
```
后出现的codex聊天记录完全消失，没有完全解决问题，但是能找到聊天记录，后续要用的时候就把对应的记录.md文件发给codex，然后继续使用。

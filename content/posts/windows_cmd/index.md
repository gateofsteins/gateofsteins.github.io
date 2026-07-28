+++
date = '2026-07-28T15:42:51+08:00'
draft = true
title = 'windows常用命令总结'

+++

对于玩计算机的，对于cmd肯定驾轻就熟，但是每次想要在windows上用cmd做些什么的时候，总是忘记该有什么命令，或者具体命令的参数，格式之类的，我结合平时学习操作过程，总结一下在网络安全中比较常用的windows命令，以后不记得的时候可以直接翻一翻

## 一. 信息收集模块

|     命令     |              用途              |                   安全场景                   |
| :----------: | :----------------------------: | :------------------------------------------: |
| `systeminfo` | 查看OS版本，补丁，架构，域名等 | 快速判断目标系统是否有一直漏洞（如MS17-010） |
|    `ver`     |        仅查看系统版本号        |                   轻量探测                   |
|  `hostname`  |           查看主机名           |             识别目标在域中的角色             |
|              |                                |                                              |
|              |                                |                                              |

## 二. 用户与权限管理模块

对应组件：`SAM数据库`    `Net.exe`  `LocalSecurity Authority(LSA)`

| 命令                                        | 用途                        | 安全场景                                                     |
| ------------------------------------------- | --------------------------- | ------------------------------------------------------------ |
| `whoami /all`                               | 当前用户、SID、所属组、权限 | 判断当前权限级别（是否SYSTEM/Admin）                         |
| `whoami /priv`                              | 列出当前令牌权限            | 检查是否有 `SeDebugPrivilege`、`SeImpersonatePrivilege` 等可提权权限 |
| `net user`                                  | 列出本地用户                | 发现隐藏账户、弱口令账户                                     |
| `net user administrator`                    | 查看指定用户详情            | 查看最后登录时间、密码策略                                   |
| `net localgroup administrators`             | 查看管理员组成员            | 判断横向移动目标                                             |
| `net user hacker P@ss123 /add`              | 创建用户                    | 渗透中留后门（防御中需监控）                                 |
| `net localgroup administrators hacker /add` | 提权到管理员组              | 权限维持                                                     |
| `query user`                                | 查看当前登录会话            | 判断是否有管理员在线                                         |



## 三. 网络配置与诊断模块

对应组件：`TCP/IP协议栈`,`DNS Client`, `Netstat`,`Netsh`

| 命令                                 | 用途                               | 安全场景                       |
| ------------------------------------ | ---------------------------------- | ------------------------------ |
| `ipconfig /all`                      | 完整网络配置（IP、DNS、DHCP、MAC） | 判断网络拓扑、域控位置         |
| `ipconfig /displaydns`               | 查看DNS缓存                        | 发现内网其他主机、C2域名       |
| `netstat -ano`                       | 所有连接及对应PID                  | 发现C2回连、异常外联、开放端口 |
| `netstat -ano | findstr ESTABLISHED` | 活跃连接                           | 快速定位可疑外联               |
| `ping / tracert / pathping`          | 连通性探测                         | 内网存活探测、路由追踪         |
| `nslookup / resolve-dnsname`         | DNS解析                            | 探测内网DNS记录、子域名        |
| `arp -a`                             | ARP缓存表                          | 发现同网段主机                 |
| `route print`                        | 路由表                             | 判断多网卡、内网段             |
| `net view`                           | 查看网络共享资源                   | 发现文件服务器                 |
| `net share`                          | 本机共享                           | 检查是否存在危险共享           |
| `net use \\IP\IPC$ /user:admin pass` | 建立IPC $ 空会话                   | 经典横向移动前置步骤           |



## 四.进程与服务管理模块

对应组件： `Task Manager API`,   `Service Control Manager(SCM)`

| `taskkill /PID <进程ID> /F`               | 强制结束进程                     | 终止恶意进程 / 关闭EDR |
| ----------------------------------------- | -------------------------------- | ---------------------- |
| `sc query state= all`                     | 所有服务状态                     | 发现异常服务（后门）   |
| `sc qc <服务名>`                          | 服务配置（启动类型、路径、权限） | 寻找可写路径→服务提权  |
| `sc create evil binpath= "cmd /c ..."`    | 创建服务                         | 权限维持               |
| `sc config <服务内部名称> binpath= "..."` | 修改服务路径                     | 服务劫持               |
| `net start` / `net stop`                  | 启动/停止服务                    | 关闭防火墙、杀毒       |
| `wmic process call create "cmd.exe"`      | WMI创建进程                      | 无文件横向执行         |



## 五.文件系统与权限模块

对应组件：`NTFS ACL`,   `CMD文件操作`，  `icacls`

| 命令                             | 用途              | 安全场景                        |
| -------------------------------- | ----------------- | ------------------------------- |
| `dir /s /b *.config`             | 递归搜索文件      | 搜索配置文件中的密码            |
| `type / more`                    | 查看文件内容      | 读取敏感配置                    |
| `findstr /s /i "password" *.xml` | 内容搜索          | 批量搜索凭据                    |
| `icacls <path>`                  | 查看NTFS权限      | 发现Everyone可写的敏感文件→提权 |
| `icacls <path> /grant Users:F`   | 修改权限          | 篡改系统文件                    |
| `attrib +h +s file`              | 设置隐藏/系统属性 | 隐藏后门文件                    |
| `copy / xcopy / robocopy`        | 文件复制          | 数据窃取、工具上传              |



## 六. 注册表操作模块

对应组件： Register Editor(regedit), reg.exe

| 命令                                                         | 用途         | 安全场景         |
| ------------------------------------------------------------ | ------------ | ---------------- |
| `reg query HKLM\...\Run`                                     | 查看启动项   | 发现持久化后门   |
| `reg query HKLM\SYSTEM\...\Services`                         | 服务注册表项 | 服务配置审计     |
| `reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon"` | 登录配置     | 查看自动登录凭据 |
| `reg add HKCU\...\Run /v evil /d "..."`                      | 添加启动项   | 权限维持         |
| `reg save HKLM\SAM C:\sam.hive`                              | 导出SAM      | 离线提取本地哈希 |
| `reg save HKLM\SYSTEM C:\sys.hive`                           | 导出SYSTEM   | 配合SAM解密      |
| `reg query HKLM\...\Policies`                                | 组策略注册表 | 查看安全策略限制 |

示例：

```cmd
:: 提取SAM和SYSTEM用于离线哈希dump
reg save HKLM\SAM C:\Windows\Temp\sam.hive
reg save HKLM\SYSTEM C:\Windows\Temp\system.hive
:: 然后用secretsdump.py或mimikatz离线解析

:: 查看所有自启动项
reg query HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run
reg query HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\Run
```



## 七.防火墙与安全策略模块

对应组件：Windows Firewall,Local Security Policy(secpol.msc),Windows Defender

| 命令                                                   | 用途             | 安全场景               |
| ------------------------------------------------------ | ---------------- | ---------------------- |
| `netsh advfirewall show allprofiles`                   | 查看防火墙状态   | 判断是否可直接入站     |
| `netsh advfirewall set allprofiles state off`          | 关闭防火墙       | 渗透中消除障碍         |
| `netsh advfirewall firewall add rule ...`              | 添加放行规则     | 开放C2端口             |
| `netsh advfirewall firewall delete rule name=all`      | 清空规则         | 破坏防御               |
| `net accounts`                                         | 查看密码策略     | 判断爆破可行性         |
| `net accounts /minpwlen:0`                             | 修改密码策略     | 削弱安全（攻击）/ 测试 |
| `powershell Get-MpPreference`                          | Defender配置     | 查看排除路径           |
| `powershell Add-MpPreference -ExclusionPath "C:\temp"` | 添加Defender排除 | 绕过杀软               |

示例：

```cmd
:: 安全加固：开启防火墙并仅放行必要端口
netsh advfirewall set allprofiles state on
netsh advfirewall firewall add rule name="RDP" dir=in action=allow protocol=tcp localport=3389

:: 渗透：添加规则放行反向shell端口
netsh advfirewall firewall add rule name="Allow4444" dir=in action=allow protocol=tcp localport=4444
```



## 八.计划任务模块

对应组件：Task Scheduler(schtasks.exe),旧版at.exe

| 命令                                                         | 用途               | 安全场景          |
| ------------------------------------------------------------ | ------------------ | ----------------- |
| `schtasks /query /fo LIST /v`                                | 列出所有计划任务   | 发现持久化机制    |
| `schtasks /create /tn "Update" /tr "cmd /c ..." /sc onlogon /ru SYSTEM` | 创建SYSTEM级任务   | 权限维持/提权     |
| `schtasks /run /tn "Update"`                                 | 立即执行任务       | 触发payload       |
| `schtasks /delete /tn "Update" /f`                           | 删除任务           | 清理痕迹          |
| `at \\IP 12:00 cmd /c "..."`                                 | 远程计划任务（旧） | 横向移动（需SMB） |

示例：

```cmd
:: 创建开机自启的SYSTEM级后门
schtasks /create /tn "WindowsUpdate" /tr "C:\temp\payload.exe" /sc onstart /ru SYSTEM /f

:: 审计：查找非微软签名的计划任务
schtasks /query /fo CSV /v | findstr /v "Microsoft"
```



## 九.远程管理与横向移动模块

对应组件：WinRM，WMI，SMB(Psexec),RPC

| 命令/工具                                                    | 用途          | 安全场景                 |
| ------------------------------------------------------------ | ------------- | ------------------------ |
| `winrm quickconfig`                                          | 启用WinRM     | 开启远程管理通道         |
| `winrs -r:http://IP:5985 cmd`                                | WinRM远程执行 | 横向命令执行             |
| `powershell Enter-PSSession -ComputerName IP`                | PS远程会话    | 交互式横向               |
| `powershell Invoke-Command -ComputerName IP -ScriptBlock {...}` | 批量远程执行  | 批量横向                 |
| `wmic /node:IP process call create "cmd /c ..."`             | WMI远程执行   | 无PsExec横向             |
| `psexec \\IP -u admin -p pass cmd`                           | SMB远程执行   | 经典横向（Sysinternals） |
| `sc \\IP create evil binpath= "..."`                         | 远程创建服务  | 服务级横向               |
| `mmc comexp.msc /computer=IP`                                | 远程COM+      | DCOM横向                 |



## 十. 日志与审计模块

对应组件： Event Log Service, wevtutil,  Audit Policy

| 命令                                                         | 用途             | 安全场景                  |
| ------------------------------------------------------------ | ---------------- | ------------------------- |
| `wevtutil el`                                                | 列出所有日志通道 | 了解可审计范围            |
| `wevtutil qe Security /c:10 /f:text`                         | 读取安全日志     | 分析登录事件（4624/4625） |
| `wevtutil cl Security`                                       | 清除日志         | 攻击者清痕 / 测试         |
| `eventvwr.msc`                                               | 图形化日志查看   | 安全运营分析              |
| `auditpol /get /category:*`                                  | 查看审计策略     | 判断哪些操作被记录        |
| `auditpol /set /category:"Logon" /success:enable /failure:enable` | 开启审计         | 安全加固                  |
| `powershell Get-WinEvent -LogName Security -MaxEvents 50`    | PS读日志         | 自动化日志分析            |

关键事件ID：

- `4624` - 登录成功
- `4625` - 登录失败（爆破检测）
- `4672` - 特权登录
- `4688` - 进程创建
- `7045` - 新服务安装
- `1102` - 日志被清除（告警！）

## 十一. 域渗透专用模块

对应组件：Active Directory ,  LDAP,   Kerberos,   DNS

| 命令                                    | 用途         | 安全场景          |
| --------------------------------------- | ------------ | ----------------- |
| `nltest /dclist:domain.com`             | 列出域控     | 定位DC            |
| `nltest /domain_trusts`                 | 域信任关系   | 跨域攻击路径      |
| `dsquery user -name *admin*`            | 搜索域用户   | 发现高权限账户    |
| `dsquery computer`                      | 搜索域内主机 | 资产枚举          |
| `dsget user "CN=..." -memberof`         | 用户所属组   | 权限分析          |
| `net group "Domain Admins" /domain`     | 域管成员     | 高价值目标        |
| `net group "Enterprise Admins" /domain` | 企业管理员   | 森林级权限        |
| `setspn -Q */*`                         | 查询SPN      | Kerberoasting前置 |
| `dnscmd /enumrecords`                   | DNS记录枚举  | 内网资产发现      |
| `repadmin /showrepl`                    | 域复制状态   | DC健康检查        |
| `ldifde -f out.ldf`                     | 导出AD对象   | 批量信息收集      |

示例：

```cmd
:: 快速域信息收集
echo %USERDOMAIN% & echo %LOGONSERVER%
nltest /dclist:corp.com
net group "Domain Admins" /domain
net group "Domain Controllers" /domain

:: 查找SPN（Kerberoasting）
setspn -T corp.com -Q */*
```



## 十二. PowerShell安全模块

对应组件：.Net Framework    ,  PowerShell  Engine,  AMSI

| 命令/模块                                            | 用途           | 安全场景          |
| ---------------------------------------------------- | -------------- | ----------------- |
| `Get-ExecutionPolicy` / `Set-ExecutionPolicy Bypass` | 执行策略       | 绕过脚本限制      |
| `Get-Process` / `Get-Service`                        | 进程/服务      | 信息收集          |
| `Get-NetTCPConnection`                               | 网络连接       | 替代netstat       |
| `Invoke-WebRequest` / `IWR`                          | 下载文件       | 无浏览器下载      |
| `Invoke-Expression (IEX)`                            | 执行字符串命令 | 内存中执行payload |
| `[System.Net.WebClient]::new().DownloadFile(...)`    | .NET下载       | 绕过检测          |
| `Get-ADUser -Filter *`                               | AD模块查用户   | 域内枚举          |
| `Get-GPO -All`                                       | 组策略对象     | 发现GPO配置漏洞   |
| `Get-Acl` / `Set-Acl`                                | 权限查看/修改  | ACL提权           |
| `powershell -ep bypass -nop -w hidden -enc <base64>` | 编码执行       | 绕过简单检测      |



## 十三. 其他实用模块

| 模块     | 命令                                  | 用途                                |
| -------- | ------------------------------------- | ----------------------------------- |
| 驱动     | `driverquery /v`                      | 列出驱动，寻找有漏洞的驱动（BYOVD） |
| 证书     | `certutil -store` / `certutil -dump`  | 查看/导出证书                       |
| 组策略   | `gpresult /r` / `rsop.msc`            | 查看生效的GPO                       |
| WMI      | `wmic startup list full`              | 启动项                              |
| 凭据     | `cmdkey /list`                        | 已保存的凭据                        |
| WiFi     | `netsh wlan show profiles` / `export` | 导出WiFi密码                        |
| 剪贴板   | `powershell Get-Clipboard`            | 获取剪贴板内容                      |
| 环境变量 | `set` / `echo %PATH%`                 | 发现PATH劫持机会                    |



## 总结：按攻击链对应

```cmd
信息收集 → systeminfo / whoami / netstat / ipconfig / net user / nltest
权限提升 → whoami /priv / icacls / sc qc / schtasks / Get-Acl
横向移动 → psexec / wmic / winrm / sc \\IP / at
权限维持 → reg add Run / schtasks / sc create / net user
防御绕过 → certutil / netsh advfirewall / Add-MpPreference / -enc
痕迹清理 → wevtutil cl / del %APPDATA%\...\Recent / klist purge
```


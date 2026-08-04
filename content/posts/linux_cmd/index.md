+++
date = '2026-07-29T12:11:56+08:00'
draft = true
title = 'Linux_cmd'
tags = ['linux', '命令行', '网络安全']
comments = true

+++



## Linux 网络安全命令——按功能模块分类详解

------

## 一、文件与权限管理模块（VFS / Permission Subsystem）

Linux 一切皆文件，文件权限是安全的第一道防线。

### 1.1 权限查看与修改

| 命令              | 用途                                | 安全场景                                     |
| ----------------- | ----------------------------------- | -------------------------------------------- |
| `ls -la`          | 查看文件详细权限、属主、属组        | 排查异常权限文件（如 SUID 后门）             |
| `chmod`           | 修改文件权限位                      | 收紧敏感文件权限，如 `chmod 600 /etc/shadow` |
| `chown` / `chgrp` | 修改属主/属组                       | 修复被篡改的文件归属                         |
| `umask`           | 设置默认创建权限掩码                | 加固系统默认权限策略，如 `umask 027`         |
| `stat`            | 查看文件 inode 详细信息（含时间戳） | 取证时分析文件是否被篡改（MAC 时间）         |

### 1.2 特殊权限排查

```bash
# 查找所有 SUID 文件（潜在提权点）
find / -perm -4000 -type f 2>/dev/null

# 查找所有 SGID 文件
find / -perm -2000 -type f 2>/dev/null

# 查找全局可写文件（可能被植入恶意代码）
find / -perm -0002 -type f 2>/dev/null

# 查找无属主文件（可能是残留后门）
find / -nouser -o -nogroup 2>/dev/null
```

### 1.3 文件完整性校验

```bash
# 计算文件哈希（MD5/SHA256），用于完整性比对
md5sum /usr/bin/passwd
sha256sum /etc/passwd

# 批量校验（配合基线数据库）
sha256sum -c baseline.sha256

# 查找最近被修改的文件（入侵痕迹排查）
find / -mtime -1 -type f 2>/dev/null      # 24小时内修改
find / -ctime -1 -type f 2>/dev/null      # 24小时内属性变更
find / -atime -1 -type f 2>/dev/null      # 24小时内被访问
```

### 1.4 扩展属性与隐藏文件

```bash
# 查看/设置扩展属性（如不可修改标志）
lsattr /etc/passwd
chattr +i /etc/passwd          # 设置不可变（immutable），root 也无法直接修改
chattr +a /var/log/syslog      # 仅追加模式，防止日志被篡改

# 查找隐藏文件/目录
find / -name ".*" -type f 2>/dev/null
find / -name "..*" -type f 2>/dev/null   # 双点隐藏文件
```

------

## 二、用户与身份管理模块（User/Group Subsystem）

### 2.1 用户信息审查

| 命令             | 用途                        | 安全场景                   |
| ---------------- | --------------------------- | -------------------------- |
| `id`             | 查看当前用户 UID/GID/所属组 | 确认当前权限上下文         |
| `who` / `w`      | 查看当前登录用户及活动      | 发现异常登录会话           |
| `whoami`         | 当前有效用户名              | 确认是否被提权             |
| `last` / `lastb` | 成功/失败登录历史           | 排查暴力破解痕迹           |
| `lastlog`        | 每个用户最后登录时间        | 发现长期未用却被激活的账户 |

### 2.2 账户文件分析

```bash
# /etc/passwd：查看所有用户（关注 UID=0 的异常账户）
awk -F: '$3==0 {print $1}' /etc/passwd

# /etc/shadow：检查空密码账户
awk -F: '($2=="") {print $1}' /etc/shadow

# 检查密码策略（密码过期、最小长度等）
chage -l username

# 查看用户 shell（异常 shell 如 /bin/bash 分配给服务账户）
grep -v '/nologin\|/false' /etc/passwd

# 查看 sudo 权限配置
cat /etc/sudoers
visudo -c                      # 语法检查
sudo -l                        # 列出当前用户 sudo 权限
```

### 2.3 用户管理操作

```bash
useradd / usermod / userdel    # 增/改/删用户
passwd -l username             # 锁定账户
passwd -e username             # 强制密码过期
groupadd / groupmod / groupdel # 组管理
gpasswd -a user group          # 将用户加入组
```

------

## 三、网络与通信模块（Network Stack / Netfilter）

这是网络安全最核心的模块。

### 3.1 网络状态监控

| 命令                    | 用途                   | 安全场景                   |
| ----------------------- | ---------------------- | -------------------------- |
| `ip addr` / `ifconfig`  | 查看网卡与 IP 配置     | 发现异常网卡（如混杂模式） |
| `ip route` / `route -n` | 查看路由表             | 排查路由劫持               |
| `ss -tulnp`             | 查看监听端口及对应进程 | 发现后门监听端口           |
| `netstat -tulnp`        | 同上（旧版）           | 兼容老系统                 |
| `ss -s`                 | 连接统计摘要           | 判断是否遭受 SYN Flood     |
| `lsof -i :端口`         | 查看占用指定端口的进程 | 定位可疑服务               |

```bash
# 查看所有 ESTABLISHED 连接（排查 C2 外联）
ss -tnp state established

# 查看所有监听端口
ss -tulnp

# 查看特定 IP 的连接
ss -tnp | grep 1.2.3.4

# 检查网卡是否处于混杂模式（嗅探检测）
ip link show | grep -i promisc
ifconfig | grep -i promisc
```

### 3.2 流量抓取与分析

```bash
# tcpdump：命令行抓包
tcpdump -i eth0 -nn -c 100 port 80        # 抓100个HTTP包
tcpdump -i eth0 -w capture.pcap           # 保存为 pcap 文件
tcpdump -r capture.pcap -X                # 读取并以十六进制显示

# 常用过滤
tcpdump -i eth0 'tcp[tcpflags] & (tcp-syn) != 0'   # SYN 包（扫描检测）
tcpdump -i eth0 'icmp'                              # ICMP 流量（隧道检测）
tcpdump -i eth0 'port not 22 and port not 443'      # 排除常规流量
```

### 3.3 网络探测与扫描

```bash
# ping / fping：存活探测
ping -c 4 target
fping -g 192.168.1.0/24          # 批量存活扫描

# traceroute / mtr：路由追踪
traceroute target
mtr -r -c 100 target             # 综合路由质量分析

# nmap：端口与服务扫描（需安装）
nmap -sS -p- target              # SYN 全端口扫描
nmap -sV -O target               # 服务版本 + OS 识别
nmap --script vuln target        # 漏洞脚本扫描

# nc (netcat)：网络瑞士军刀
nc -zv target 1-1000             # 端口扫描
nc -lvp 4444                     # 监听（应急时用于反向 shell 检测）
echo "test" | nc target 80       # 手动发送数据
```

### 3.4 DNS 相关

```bash
dig / nslookup / host            # DNS 查询
dig @8.8.8.8 target.com ANY      # 查询所有记录
dig -x IP                        # 反向解析
cat /etc/resolv.conf             # 检查 DNS 配置是否被篡改
cat /etc/hosts                   # 检查 hosts 劫持
```

### 3.5 ARP 与链路层

```bash
arp -a                           # 查看 ARP 缓存
arping -I eth0 IP                # 主动 ARP 探测
ip neigh show                    # 邻居表（含 ARP）
# 检测 ARP 欺骗：观察同一 IP 是否对应多个 MAC
```

------

## 四、进程与系统监控模块（Process Scheduler / Procfs）

### 4.1 进程审查

```bash
ps aux                           # 查看所有进程
ps auxf                          # 树状显示父子关系
ps -eo pid,ppid,user,cmd --sort=-%cpu   # 按 CPU 排序

top / htop                       # 实时进程监控
pstree -p                        # 进程树

# 排查可疑进程
ls -l /proc/PID/exe              # 查看进程实际可执行文件路径
ls -l /proc/PID/cwd              # 工作目录
cat /proc/PID/cmdline            # 完整启动命令
cat /proc/PID/environ            # 环境变量（可能含敏感信息）
cat /proc/PID/status             # 进程状态（含 UID/GID）
ls -l /proc/PID/fd               # 打开的文件描述符

# 查找被删除但仍被进程占用的文件（内存驻留木马）
ls -l /proc/*/fd 2>/dev/null | grep deleted
```

### 4.2 资源与性能监控

```bash
vmstat 1 5                       # CPU/内存/IO 综合状态
iostat -x 1                      # 磁盘 IO（排查挖矿程序）
free -h                          # 内存使用
dmesg                            # 内核环形缓冲区日志（硬件/驱动异常）
```

### 4.3 计划任务审查（持久化后门高发区）

```bash
crontab -l                       # 当前用户定时任务
cat /etc/crontab                 # 系统级定时任务
ls -la /etc/cron.d/              # 第三方定时任务
ls -la /etc/cron.daily/          # 每日任务
ls -la /var/spool/cron/          # 用户级 cron 文件

# systemd timer（新型持久化）
systemctl list-timers --all
```

### 4.4 启动项与服务审查

```bash
systemctl list-unit-files --state=enabled   # 开机自启服务
systemctl status service_name               # 服务详情
chkconfig --list                            # SysVinit 启动项（老系统）
ls /etc/init.d/                             # 初始化脚本
```

------

## 五、日志与审计模块（Syslog / Auditd / Journald）

### 5.1 系统日志

```bash
# 传统 syslog
cat /var/log/auth.log            # Debian/Ubuntu 认证日志
cat /var/log/secure              # RHEL/CentOS 认证日志
cat /var/log/syslog              # 综合系统日志
cat /var/log/kern.log            # 内核日志
cat /var/log/dpkg.log            # 软件包安装记录（Debian）
cat /var/log/yum.log             # 软件包记录（RHEL）

# systemd journal
journalctl -u sshd --since "1 hour ago"    # SSH 服务最近1小时日志
journalctl -p err -b                         # 本次启动的错误日志
journalctl _COMM=sudo                        # sudo 操作记录
```

### 5.2 日志分析实战

```bash
# 统计 SSH 暴力破解来源 IP
grep "Failed password" /var/log/auth.log | \
  awk '{print $(NF-3)}' | sort | uniq -c | sort -rn | head -20

# 查看成功登录记录
grep "Accepted" /var/log/auth.log

# 查看 sudo 使用记录
grep "sudo" /var/log/auth.log

# 日志防篡改验证（rsyslog 支持）
# 配置 rsyslog 使用 RELP 远程转发，本地日志不可信时查远程
```

### 5.3 Auditd 审计框架

```bash
# 查看审计规则
auditctl -l

# 添加审计规则示例
auditctl -w /etc/passwd -p wa -k passwd_change    # 监控 passwd 文件写/属性变更
auditctl -w /usr/bin/sudo -p x -k sudo_exec       # 监控 sudo 执行
auditctl -a always,exit -F arch=b64 -S execve -k all_exec  # 监控所有命令执行

# 搜索审计日志
ausearch -k passwd_change
ausearch -m USER_LOGIN --success no               # 失败登录
aureport --auth                                    # 认证报告
aureport --file --summary                          # 文件访问摘要
```

------

## 六、加密与认证模块（OpenSSL / GnuPG / PAM）

### 6.1 OpenSSL

```bash
# 生成密钥与证书
openssl genrsa -out key.pem 2048
openssl req -new -x509 -key key.pem -out cert.pem -days 365

# 查看证书详情（排查中间人攻击）
openssl x509 -in cert.pem -text -noout
echo | openssl s_client -connect target:443 2>/dev/null | openssl x509 -noout -dates

# 文件加密/解密
openssl enc -aes-256-cbc -salt -in file -out file.enc
openssl enc -d -aes-256-cbc -in file.enc -out file

# 计算哈希
openssl dgst -sha256 file
```

### 6.2 GPG

```bash
gpg --gen-key                    # 生成密钥对
gpg --encrypt --recipient user file   # 加密
gpg --decrypt file.gpg           # 解密
gpg --verify file.sig file       # 验证签名（软件包完整性）
```

### 6.3 PAM（Pluggable Authentication Modules）

```bash
# 查看 PAM 配置
cat /etc/pam.d/sshd
cat /etc/pam.d/common-auth       # Debian
cat /etc/pam.d/system-auth       # RHEL

# 安全加固相关模块
# pam_tally2.so / pam_faillock.so  → 登录失败锁定
# pam_pwquality.so                 → 密码复杂度
# pam_time.so                      → 登录时间限制
# pam_access.so                    → 基于用户/主机的访问控制
```

------

## 七、防火墙与访问控制模块（Netfilter / nftables / SELinux / AppArmor）

### 7.1 iptables / nftables

```bash
# 查看当前规则
iptables -L -n -v --line-numbers
nft list ruleset                 # nftables

# 常见安全策略
iptables -P INPUT DROP           # 默认拒绝入站
iptables -A INPUT -i lo -j ACCEPT
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
iptables -A INPUT -p tcp --dport 22 -j ACCEPT
iptables -A INPUT -p tcp --dport 443 -j ACCEPT

# 防暴力破解（限制 SSH 连接速率）
iptables -A INPUT -p tcp --dport 22 -m recent --name ssh --set
iptables -A INPUT -p tcp --dport 22 -m recent --name ssh --update --seconds 60 --hitcount 4 -j DROP

# 防 SYN Flood
iptables -A INPUT -p tcp --syn -m limit --limit 10/s --limit-burst 20 -j ACCEPT
iptables -A INPUT -p tcp --syn -j DROP

# 日志记录被拒绝的包
iptables -A INPUT -j LOG --log-prefix "IPTABLES-DROP: " --log-level 4
```

### 7.2 firewalld / ufw（高层封装）

```bash
# firewalld (RHEL)
firewall-cmd --state
firewall-cmd --list-all
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="1.2.3.4" reject'
firewall-cmd --reload

# ufw (Ubuntu)
ufw status verbose
ufw default deny incoming
ufw allow 22/tcp
ufw enable
```

### 7.3 SELinux

```bash
getenforce                       # 查看模式（Enforcing/Permissive/Disabled）
sestatus                         # 详细状态
semanage fcontext -l             # 查看文件上下文规则
restorecon -Rv /var/www/html     # 恢复安全上下文
ausearch -m avc -ts recent       # 查看 SELinux 拒绝日志
setsebool -P httpd_can_network_connect 1   # 调整布尔值
```

### 7.4 AppArmor

```bash
aa-status                        # 查看加载的 profile
aa-enforce /etc/apparmor.d/usr.sbin.nginx   # 强制模式
aa-complain /etc/apparmor.d/...  # 投诉模式（仅记录）
cat /var/log/audit/audit.log | grep apparmor   # 日志
```

### 7.5 TCP Wrappers

```bash
cat /etc/hosts.allow
cat /etc/hosts.deny
# 示例：仅允许特定 IP 访问 SSH
# /etc/hosts.allow:  sshd: 192.168.1.0/24
# /etc/hosts.deny:   ALL: ALL
```

------

## 八、磁盘与存储安全模块（Block Layer / LVM / dm-crypt）

```bash
# 查看磁盘与分区
lsblk -f                         # 含文件系统类型和 UUID
fdisk -l / parted -l
df -hT                           # 文件系统使用情况
mount | column -t                # 挂载点及选项（关注 nosuid,noexec 等）

# 安全挂载选项加固（/etc/fstab）
# /tmp 分区: nosuid,nodev,noexec
# /var 分区: nosuid,nodev

# LUKS 磁盘加密
cryptsetup luksFormat /dev/sdb1
cryptsetup luksOpen /dev/sdb1 secure_data
cryptsetup luksDump /dev/sdb1    # 查看加密元信息

# 安全擦除磁盘
shred -vfz -n 3 /dev/sda        # 多次覆写
wipefs -a /dev/sda              # 擦除文件系统签名
```

------

## 九、内核与系统加固模块（Sysctl / Kernel Parameters）

```bash
# 查看/修改内核网络参数
sysctl -a | grep net.ipv4

# 关键安全加固项（/etc/sysctl.conf）
net.ipv4.ip_forward = 0                    # 禁止 IP 转发（非路由器）
net.ipv4.conf.all.accept_redirects = 0     # 禁止 ICMP 重定向
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.all.accept_source_route = 0  # 禁止源路由
net.ipv4.tcp_syncookies = 1                # 防 SYN Flood
net.ipv4.conf.all.rp_filter = 1            # 反向路径过滤（防 IP 欺骗）
net.ipv4.icmp_echo_ignore_broadcasts = 1   # 忽略广播 ping（防 Smurf）
kernel.randomize_va_space = 2              # 完整 ASLR
kernel.kptr_restrict = 2                   # 隐藏内核指针
kernel.dmesg_restrict = 1                  # 限制 dmesg 访问
fs.suid_dumpable = 0                       # 禁止 SUID 程序 core dump

sysctl -p                                  # 应用配置
```

### 内核模块管理

```bash
lsmod                            # 查看已加载内核模块（排查 rootkit）
modprobe -r module_name          # 卸载模块
cat /etc/modprobe.d/blacklist.conf   # 黑名单
# 禁用不需要的模块（如 usb-storage、firewire）
echo "install usb-storage /bin/true" >> /etc/modprobe.d/blacklist.conf
```

------

## 十、包管理与软件安全模块（dpkg/rpm/apt/yum）

```bash
# Debian/Ubuntu
apt list --installed
dpkg -l | grep package_name
dpkg --verify package_name       # 校验包完整性（md5sum 比对）
apt-get install --only-upgrade package
debsums -c                       # 全局文件完整性校验（需安装 debsums）

# RHEL/CentOS
rpm -qa
rpm -V package_name              # 校验包（S.5....T. 标志）
rpm -Va                          # 校验所有包
yum check-update
dnf repoquery --installed

# 通用：检查 GPG 签名
rpm -K package.rpm
dpkg-sig --verify package.deb
```

------

## 十一、文本处理与日志分析模块（Shell 工具链）

这些不是"安全命令"，但在安全分析中不可或缺：

```bash
grep -rni "password" /etc/       # 递归搜索敏感关键词
awk '{print $1}' access.log | sort | uniq -c | sort -rn   # IP 统计
sed -n '100,200p' logfile        # 提取指定行范围
cut -d: -f1 /etc/passwd          # 字段提取
sort / uniq / wc                 # 排序/去重/计数
strings binary_file              # 提取二进制中的可读字符串（恶意软件初筛）
xxd / hexdump -C file            # 十六进制查看
file suspicious_file             # 识别文件真实类型
base64 -d encoded.txt            # 解码（排查混淆 payload）
```

------

## 十二、服务与守护进程管理模块（systemd / init）

```bash
systemctl list-units --type=service --state=running   # 运行中的服务
systemctl disable service_name   # 关闭不必要的服务（减小攻击面）
systemctl mask service_name      # 彻底屏蔽（无法被其他单元拉起）
systemctl cat service_name       # 查看服务单元文件（排查恶意服务）
systemctl daemon-reload          # 重载配置

# 查看服务绑定的端口
systemctl status service_name | grep -i listen
# 或直接
ss -tulnp | grep service_name
```

------

## 总结：按安全场景速查

| 安全场景       | 核心命令                                                     |
| -------------- | ------------------------------------------------------------ |
| **入侵排查**   | `last`, `lastb`, `ss -tulnp`, `ps auxf`, `find -mtime`, `crontab -l`, `ausearch` |
| **提权检测**   | `find -perm -4000`, `sudo -l`, `id`, `cat /etc/passwd`(UID=0) |
| **流量分析**   | `tcpdump`, `ss`, `ip route`, `arp -a`, `dig`                 |
| **加固基线**   | `sysctl`, `chmod`, `chown`, `iptables/nft`, `sestatus`, `systemctl disable` |
| **完整性校验** | `sha256sum`, `rpm -Va`, `debsums`, `chattr +i`, `auditctl`   |
| **日志审计**   | `journalctl`, `grep auth.log`, `ausearch`, `aureport`        |
| **应急响应**   | `kill`, `iptables -A INPUT -s IP -j DROP`, `systemctl isolate`, `dd`(内存取证) |

------

> 💡 **学习建议**：每个模块都可以用 `man 命令名` 或 `命令 --help` 深入查阅。安全工作中最核心的思想是 **"基线对比"**——先建立正常状态快照，出问题时用上述命令对比差异，就能快速定位异常。

如果你对某个模块想进一步深入（比如 Netfilter 规则编写、Auditd 高级审计策略、或内存取证），可以告诉我，我再展开讲。
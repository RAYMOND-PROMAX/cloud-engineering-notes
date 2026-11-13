# Linux 高频运维命令全梳理，日常工作随手即用

## 写在前面

Linux 运维的"基本功"是命令。新人最大的痛苦不是不会用某个工具，而是**不知道有这工具存在**。老工程师经常 5 分钟搞定的事，新人翻文档 2 小时。

这篇文章把所有高频命令按"场景"分类，每个命令给：

- 用途
- 常见参数
- 一个真实能跑的示例
- 注意事项 / 风险提示

不写"Hello World"级别的 demo，全部按生产场景来。**学完这一篇，日常 80% 的运维操作能直接照着敲**。

适用读者：

- 刚开始做 Linux 运维的新人
- 想系统梳理命令体系的初中级工程师
- DevOps 工程师，跨工种需要 Linux 排错能力

## 一、命令速查索引

| 分类 | 核心命令 |
| --- | --- |
| 系统信息 | uname, hostname, uptime, date, cal, who, w, last |
| 进程 | ps, top, htop, pidof, pgrep, pkill, kill, pstree, nice, renice, systemctl |
| CPU | top, mpstat, sar -u, vmstat, lscpu, turbostat, perf top |
| 内存 | free, vmstat, sar -r, slabtop, smem, pmap, numastat |
| 网络 | ip, ss, netstat, curl, wget, tcpdump, iftop, nethogs, mtr, traceroute, nc, ethtool, sar -n |
| 磁盘 | df, du, lsblk, blkid, mount, fdisk, parted, iostat, iotop, fuser, lsof |
| 文件 | find, locate, stat, file, tree, ls, chmod, chown, ln, rsync, scp, tar, zip |
| 日志 | tail, head, less, grep, awk, sed, journalctl, dmesg, logrotate |
| 用户 | useradd, usermod, userdel, passwd, chage, groupadd, id, su, sudo |
| 服务 | systemctl, journalctl, systemd-analyze |
| 包管理 | yum, dnf, apt, rpm, dpkg |
| 定时 | crontab, at, flock |
| 性能分析 | strace, ltrace, perf, bcc, bpftrace |
| 文本三剑客 | grep, awk, sed, xargs, sort, uniq, cut, tr, column |
| 工具 | tmux, screen, glances, dstat, nmon, sysstat |

下面分类详讲。

## 二、系统信息命令

### 2.1 uname

```
uname -a
# Linux host-01 5.15.0-91-generic #101-Ubuntu SMP Tue Nov 14 13:30:08 UTC 2023 x86_64 x86_64 x86_64 GNU/Linux

uname -r
# 5.15.0-91-generic
# 看内核版本，模块编译时要用

uname -m
# x86_64
# 看架构，aarch64 / x86_64 / i686
```

### 2.2 hostname

```
hostname
# 看主机名

hostnamectl set-hostname redis-prod-01
# 永久改主机名（systemd 机器）

hostname -I
# 看所有 IP
```

### 2.3 uptime

```
uptime
# 14:30:00 up 200 days, 10:00,  3 users,  load average: 0.50, 0.40, 0.30
# - up 200 days：开机时间
# - 3 users：登录用户数
# - load average：1 / 5 / 15 分钟平均负载
# 负载除以 CPU 核数，> 1 说明忙
```

### 2.4 date

```
date
# 看时间

date -s "2026-06-13 14:30:00"
# 改时间（生产别瞎改，会打乱业务）

date -d "@1700000000"
# 时间戳转时间

date +%s
# 当前时间戳

date -d "yesterday" +%Y-%m-%d
# 算昨天
```

### 2.5 w / who / last

```
w
# 看当前登录的用户，正在做什么
# 14:30:00 up 200 days,  3 users,  load average: 0.50, 0.40, 0.30
# USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
# root     pts/0    10.20.0.100      14:00    0.00s  0.10s  0.00s w

who
# 看登录用户，简版

last
# 看历史登录，从 /var/log/wtmp 读
# 排查谁登陆过服务器

last -i
# 看 IP
```

## 三、进程命令

### 3.1 ps

```
# BSD 风格
ps aux
# USER  PID  %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
# root    1   0.0  0.0  169404 11556 ?        Ss   Jun10   0:08 /sbin/init

# System V 风格
ps -ef
# UID  PID  PPID  C STIME TTY          TIME CMD

# 看指定进程
ps -ef | grep nginx

# 按 CPU 排序
ps aux --sort=-%cpu | head

# 按内存排序
ps aux --sort=-%mem | head

# 看进程树
ps auxf

# 详细字段
ps -eo pid,ppid,user,stat,pcpu,pmem,vsz,rss,etime,cmd
# etime：进程运行时长

# 看线程
ps -eLf
# LWP（线程 ID）列
```

### 3.2 top

```
top
# 进入交互：
# - P：按 CPU 排序
# - M：按内存排序
# - T：按时间排序
# - 1：展开多核 CPU
# - c：显示完整命令行
# - V：森林视图（进程树）
# - k：kill 进程
# - r：renice

# 一次性输出，不进入交互
top -bn1 | head -20

# 字段含义：
# %CPU：CPU 占用
# %MEM：内存占用
# VSZ：虚拟内存大小
# RSS：实际物理内存
# STAT：状态
#   R：运行
#   S：睡眠（可中断）
#   D：不可中断睡眠（等 IO）
#   Z：僵尸
#   T：停止
#   I：空闲
#   <：高优先级
#   N：低优先级
#   L：有锁页在内存
#   s：会话首进程
#   +：前台进程组
```

### 3.3 htop

```
yum install -y htop

htop
# 比 top 直观，鼠标可点
# - F2：配置
# - F3：搜索
# - F4：过滤
# - F5：树状
# - F6：排序列
# - F9：kill
# - F10：退出
```

### 3.4 pidof / pgrep / pkill

```
# 找 PID
pidof nginx
# 1234 5678

pgrep -f nginx
# 按命令行匹配

pgrep -u mysql mysqld
# 按用户匹配

# 杀进程
pkill nginx
# 杀所有 nginx

pkill -9 -f "python.*app.py"
# -9：SIGKILL
# -f：按命令行

# 慎用 pkill 模糊匹配，可能误杀
```

### 3.5 kill

```
# 信号
# 1 SIGHUP：重新加载配置（nginx -s reload）
# 2 SIGINT：中断（Ctrl+C）
# 9 SIGKILL：强制杀
# 15 SIGTERM：优雅退出（默认）
# 18 SIGCONT：继续
# 19 SIGSTOP：暂停

kill <pid>
# 优雅退出，给进程清理时间

kill -9 <pid>
# 强制杀，可能丢数据

# 按进程名
killall nginx
pkill nginx

# 杀指定用户进程
pkill -u username

# 列所有信号
kill -l
```

风险提示：`kill -9` 是高危命令，会跳过进程的清理逻辑：

- 数据库可能丢未提交事务。
- 应用可能丢未刷盘的日志。
- 文件可能丢未关闭的句柄。

### 3.6 nice / renice

```
# 调整进程优先级
# nice 范围 -20 ~ 19，-20 最高
# 普通用户只能 0 ~ 19
# root 可以 -20 ~ 19

# 启动时设优先级
nice -n 10 python3 backup.py

# 调整已运行进程
renice -n -5 -p 1234
# 把 PID 1234 的优先级调为 -5

# 看进程的 nice
top
# NI 列
```

### 3.7 systemctl

```
# 服务管理
systemctl start nginx
systemctl stop nginx
systemctl restart nginx
systemctl reload nginx
# 重新加载配置，不中断（nginx / haproxy 支持）

systemctl status nginx
# 看服务状态、日志、PID

systemctl enable nginx
# 开机自启

systemctl disable nginx
# 禁用开机自启

systemctl is-active nginx
# 是否在运行

systemctl is-enabled nginx
# 是否开机自启

systemctl list-units --type=service
# 列出所有 service

systemctl list-units --state=failed
# 看失败的服务

systemctl mask nginx
# 禁用服务（连手动起都起不来）

systemctl daemon-reload
# 重载 systemd 配置
```

风险提示：`systemctl mask` 会创建 /dev/null 软链，禁用后连 `systemctl start` 都起不来。要恢复用 `systemctl unmask`。

## 四、CPU 命令

### 4.1 lscpu

```
lscpu
# Architecture:          x86_64
# CPU op-mode(s):        32-bit, 64-bit
# Byte Order:            Little Endian
# CPU(s):                16
# On-line CPU(s) list:   0-15
# Thread(s) per core:    2
# Core(s) per socket:    8
# Socket(s):             1
# NUMA node(s):          2
# ...
# 16 核、超线程、2 个 NUMA node
```

### 4.2 mpstat

```
yum install -y sysstat

mpstat -P ALL 1
# -P ALL：所有 CPU
# 1：1 秒一次
# 14:30:00  CPU    %usr   %nice    %sys %iowait    %irq   %soft  %steal  %guest   %idle
# 14:30:01  all    5.00    0.00    2.00    1.00    0.00    0.00    0.00    0.00   92.00
# 14:30:01    0    6.00    0.00    2.00    0.00    0.00    0.00    0.00    0.00   92.00
# ...

# 关键：
# %usr：用户态
# %sys：内核态
# %iowait：等 IO
# %idle：空闲
# %steal：被虚拟化偷走的时间
```

### 4.3 sar -u

```
sar -u 1 5
# CPU 利用率，1 秒一次取 5 次

# 看历史（需要 sysstat 收集器）
sar -u -f /var/log/sa/sa13
# 看 13 号的 CPU 历史
```

### 4.4 turbostat

```
# 看 CPU 频率、C-state、实际跑的频率
yum install -y kernel-tools

turbostat --interval 5
# 5 秒一次输出

# 关键：
# Avg_MHz：实际平均频率
# Bzy_MHz：忙时的频率
# TSC_MHz：标称频率
# 如果 Avg_MHz 远低于 TSC_MHz，CPU 在节电，可能影响性能
```

### 4.5 perf

```
yum install -y perf

# top 类似
perf top
# 默认按调用栈展示热点函数
# -g：显示调用栈

# 录 10 秒
perf record -a -g sleep 10
perf report
# 看报告

# 看指定进程
perf top -p <pid>

# 看 cache miss
perf stat -e cache-misses,cache-references -p <pid> sleep 5
```

风险提示：`perf record -a` 在生产慎用，会记录所有 CPU 事件，存储压力很大。

## 五、内存命令

### 5.1 free

```
free -h
#               total        used        free      shared  buff/cache   available
# Mem:            62G         20G        30G        1.0G         12G         40G
# Swap:          4.0G        100M        3.9G

# 关键：
# total：总内存
# used：已用（不含 buff/cache）
# free：完全空闲
# shared：共享内存（/dev/shm）
# buff/cache：页缓存和 buffer
# available：实际可用（free + 可回收的 buff/cache）
```

### 5.2 /proc/meminfo

```
cat /proc/meminfo
# MemTotal:       65876988 kB
# MemFree:        31457280 kB
# MemAvailable:   41943040 kB
# Buffers:         1024000 kB
# Cached:         10240000 kB
# SwapCached:            0 kB
# Active:         12345678 kB
# Inactive:        5678900 kB
# Dirty:             12345 kB
# Writeback:             0 kB
# AnonPages:      15728640 kB
# Mapped:           524288 kB
# ...

# 关键：
# Dirty：脏页（待写盘）
# Writeback：正在写盘的页
# AnonPages：匿名页（堆、栈）
# Mapped：被 mmap 的文件页
```

### 5.3 slabtop

```
slabtop
# 看内核 slab 分配器的占用
# 排查内核内存泄漏常用

# top 字段：
# OBJS：对象数
# ACTIVE：活跃对象
# USE：每个对象大小
# SIZE：总大小
```

### 5.4 smem

```
yum install -y smem

smem -tk
# 按 RSS / PSS / USS 排序
# PSS（Proportional Set Size）更准确，多进程共享的库按比例算

# 看指定用户
smem -u mysql

# 看指定进程
smem -p <pid>
```

### 5.5 pmap

```
# 看进程内存映射
pmap -x <pid>
# Address           Kbytes     RSS   Dirty Mode  Mapping
# 00007f1234000000   524288  524288       0 r-x-- libmysqlclient.so.21.0.30
# 00007f1246000000      128     128       0 r-x-- [vdso]
# ...

# 看内存峰值
pmap -x <pid> | tail -1
#  total kB         524288   524288       0
```

### 5.6 numastat

```
yum install -y numactl

numastat -m
# 看 NUMA 节点内存分配
# 跨 NUMA 访问性能差，要避免
# Per-node numastat 0
#                    Node 0     Node 1
# Numa_Hit         12345678  12345678
# Numa_Miss             1234      5678
# Foreign                 0         0
# Interleave_Hit     123456    123456
# Local_Node       12000000  12000000
# Other_Node          12345    12345
```

## 六、网络命令

### 6.1 ip

```
# 替代 ifconfig
ip a
# 看所有 IP
ip addr show eth0

ip link show
# 看链路状态

ip r
# 看路由表

ip route get 8.8.8.8
# 查具体走哪条路由

# 加 IP
ip addr add 10.20.0.20/24 dev eth0

# 删 IP
ip addr del 10.20.0.20/24 dev eth0

# 起 / 关网卡
ip link set eth0 up
ip link set eth0 down

# 加路由
ip route add 192.168.0.0/16 via 10.20.0.1

# 删路由
ip route del 192.168.0.0/16 via 10.20.0.1
```

### 6.2 ss

```
# 替代 netstat
ss -tunap
# -t：TCP
# -u：UDP
# -n：数字（不解析服务名）
# -a：所有
# -p：显示进程

# 看监听端口
ss -tlnp

# 看连接数
ss -s
# TCP:   100 (estab 80, closed 10, orphaned 0, timewait 5)
# ...

# 看 timewait
ss -tan state time-wait | wc -l

# 看 established
ss -tan state established | wc -l

# 按状态统计
ss -tan | awk 'NR>1{print $1}' | sort | uniq -c
```

### 6.3 netstat

```
# 还能用，但推荐迁到 ss
netstat -tunlp
netstat -an | grep ESTABLISHED | wc -l
netstat -s
# 各种协议统计
```

### 6.4 curl

```
# HTTP 调试
curl -v https://example.com
# -v：详细

curl -X POST -d "key=value" https://api.example.com/users
curl -X POST -H "Content-Type: application/json" -d '{"name":"test"}' https://api.example.com/users
curl -H "Authorization: Bearer xxx" https://api.example.com

# 看响应头
curl -I https://example.com

# 保存文件
curl -O https://example.com/file.zip
curl -o file.zip https://example.com/file.zip

# 限速
curl --limit-rate 1m -O https://example.com/bigfile.iso

# 走代理
curl -x http://proxy.example.com:8080 https://example.com

# 测速
curl -o /dev/null -s -w "time_total: %{time_total}\n" https://example.com

# 多次请求
curl -o /dev/null -s -w "%{time_total}\n" -H 'Cache-Control: no-cache' https://example.com/api
# 拿 P50 / P99
for i in {1..100}; do
    curl -o /dev/null -s -w "%{time_total}\n" https://example.com/api
done | sort -n | awk '{a[NR]=$1}END{print "P50:" a[int(NR*0.5)]; print "P99:" a[int(NR*0.99)]}'
```

### 6.5 wget

```
# 下载
wget https://example.com/file.zip

# 断点续传
wget -c https://example.com/bigfile.iso

# 限速
wget --limit-rate=1m https://example.com/bigfile.iso

# 镜像整个网站
wget -m -p -k https://example.com

# 整页保存
wget -p -k -E -np https://example.com/page.html
```

### 6.6 tcpdump

```
# 抓包
tcpdump -i eth0
# 指定网卡

tcpdump -i eth0 port 80
# 端口过滤

tcpdump -i eth0 host 10.20.0.10
# 主机过滤

tcpdump -i eth0 src 10.20.0.10
# 源 IP 过滤

tcpdump -i eth0 dst port 6379
# 目标端口

tcpdump -i eth0 -w capture.pcap
# 保存到文件
tcpdump -r capture.pcap
# 读取文件

# 抓 HTTP 请求
tcpdump -i eth0 -A -s 0 'tcp port 80 and (((ip[2:2] - ((ip[0]&0xf)<<2)) - ((tcp[12]&0xf0)>>2)) != 0)'

# 抓 DNS
tcpdump -i eth0 port 53

# 看 TCP 握手
tcpdump -i eth0 -nn -S 'tcp[tcpflags] & (tcp-syn|tcp-fin|tcp-rst) != 0'
```

风险提示：tcpdump 在生产慎用，可能捕获敏感信息（密码、token）。抓包文件要妥善保管。

### 6.7 iftop

```
yum install -y iftop

iftop -i eth0
# 实时看网络流量
# -i：网卡
# -n：不解析 IP
# -N：不解析端口
# -B：以 byte/s 显示

# 看本机 IP 的流量
iftop -i eth0 -f "src host 10.20.0.10"
```

### 6.8 nethogs

```
yum install -y nethogs

nethogs eth0
# 按进程看网络流量
# 看到哪个进程在狂用带宽
```

### 6.9 mtr

```
yum install -y mtr

mtr -r 8.8.8.8
# -r：report 模式
# 输出到目的地每一跳的丢包率 / 延迟

# 持续
mtr 8.8.8.8
# Ctrl+C 退出

# 看 TCP 路径
mtr --tcp example.com
```

### 6.10 traceroute

```
traceroute 8.8.8.8
# 看路由路径

# 走 TCP
traceroute -T example.com

# 走 ICMP
traceroute -I example.com
```

### 6.11 nc（netcat）

```
# 测端口
nc -zv 10.20.0.10 80
# -z：只测端口不开连接
# -v：详细

# 监听端口（调试用）
nc -l 12345

# 发送数据
echo"hello" | nc 10.20.0.10 12345

# 文件传输
# 接收端
nc -l 12345 > file
# 发送端
nc 10.20.0.10 12345 < file

# 端口扫描
nc -zv 10.20.0.10 20-30
```

风险提示：nc 监听端口可能带来安全风险，生产慎用。

### 6.12 ethtool

```
# 看网卡信息
ethtool eth0

# 看网卡驱动
ethtool -i eth0

# 看网卡 ring buffer
ethtool -g eth0

# 看网卡统计
ethtool -S eth0

# 改网卡速率（高危）
# ethtool -s eth0 speed 1000 duplex full autoneg off
# 不建议临时改
```

### 6.13 sar 网络

```
# 实时
sar -n DEV 1
# -n DEV：网卡流量
# IFACE   rxpck/s   txpck/s    rxkB/s    txkB/s   rxcmp/s   txcmp/s  rxmcst/s   %ifutil
# eth0     1234.00   5678.00   100.00    500.00      0.00      0.00      0.00      0.00

# TCP / ETCP
sar -n TCP,ETCP 1
# active/s：每秒主动建立连接
# passive/s：每秒被动接受连接
# retrans/s：每秒重传

# 历史
sar -n DEV -f /var/log/sa/sa13
```

## 七、磁盘命令

### 7.1 df

```
df -h
# 人性化单位

df -i
# 看 inodes

df -T
# 看文件系统类型

df --total
# 汇总
```

### 7.2 du

```
du -sh /*
# 各级目录大小

du -sh /var/log/*
# 详细目录

du -h --max-depth=1 /var/log
# 限制深度

# 排序
du -sh /var/log/* | sort -h | tail

# 排除
du -sh --exclude='*.log' /var

# 看大于 1G 的文件
find / -type f -size +1G -exec du -sh {} \; 2>/dev/null | sort -h
```

### 7.3 lsblk

```
lsblk
# 块设备树状结构

lsblk -f
# 看文件系统、UUID

lsblk -d -o NAME,SIZE,MODEL,ROTA,TRAN
# 看物理盘类型

lsblk -o NAME,SIZE,USED,AVAIL,FSTYPE,MOUNTPOINT,UUID
# 详细
```

### 7.4 mount / umount

```
mount
# 看所有挂载

mount | grep sda
# 过滤

mount -o remount,rw /data
# 重新挂载

mount -t nfs 10.20.0.100:/nfs /mnt/nfs

umount /data
# 卸

umount -f /data
# 强制

fuser -m /data
# 看谁在用
```

### 7.5 iostat

```
iostat -xz 1
# -x：扩展
# -z：忽略 0 活动
# 1：1 秒一次
# 输出每个设备的 r/s、w/s、rkB/s、wkB/s、await、%util 等
```

### 7.6 iotop

```
iotop
# 按进程看 IO

iotop -ao
# -a：累计
# -o：只显示有 IO 的进程
```

### 7.7 fuser

```
# 看谁在用这个文件 / 目录
fuser -m /data
# 列出 PID

# 杀占用
fuser -km /data
# -k：杀
# -m：挂载点
# 风险：会 kill 占用进程，先确认
```

### 7.8 lsof

```
# 看进程开了哪些文件
lsof -p <pid>

# 看谁在用这个文件
lsof /var/log/messages

# 看谁在用这个目录
lsof +D /data

# 看谁占用端口
lsof -i :80

# 看 TCP 连接
lsof -i tcp
```

## 八、文件命令

### 8.1 find

```
# 按名字
find / -name "*.log"

# 按类型
find / -type f -name "*.log"
# -type f：文件
# -type d：目录
# -type l：软链

# 按大小
find / -type f -size +1G
# +1G：大于 1G
# -1G：小于 1G

# 按时间
find / -type f -mtime +7
# mtime +7：7 天前修改
# mtime -7：7 天内修改
# mtime 7：刚好 7 天
# atime：访问时间
# ctime：变化时间

# 按用户
find / -user mysql

# 按权限
find / -perm 777

# 执行操作
find / -name "*.log" -exec rm {} \;
# 注意 \; 转义
# 或者用 + 减少 fork
find / -name "*.log" -exec rm {} +

# 限制深度
find / -maxdepth 3 -name "*.log"

# 多个条件
find / -type f \( -name "*.log" -o -name "*.tmp" \)
# 找 .log 或 .tmp

# 排除目录
find / -path "/proc" -prune -o -name "*.log" -print

# 输出格式
find / -name "*.log" -printf"%s %p\n"
# 打印大小 路径

# 找大文件并排序
find / -type f -size +100M -exec du -h {} \; 2>/dev/null | sort -hr | head -20
```

### 8.2 locate

```
yum install -y mlocate
updatedb
# 建索引（生产慎用，扫全盘 IO 大）

locate nginx.conf
# 快速找

# 局限：updatedb 之后新增的文件找不到
# 优势：比 find 快得多
```

### 8.3 stat

```
stat file.txt
# File: file.txt
# Size: 1234            Blocks: 8          IO Block: 4096   regular file
# Access: 2026-06-13 14:00:00.000000000 +0800
# Modify: 2026-06-13 14:00:00.000000000 +0800
# Change: 2026-06-13 14:00:00.000000000 +0800
# Birth: 2026-06-13 14:00:00.000000000 +0800
```

### 8.4 file

```
file file.txt
# 看文件类型

file -i file.txt
# 看 MIME
```

### 8.5 tree

```
yum install -y tree

tree /data
# 树状结构

tree -L 2 /data
# 限制 2 层

tree -h /data
# 人性化大小

tree -I "*.log|cache" /data
# 排除
```

### 8.6 chmod / chown

```
# 权限
chmod 755 file
# 7：rwx
# 6：rw-
# 5：r-x
# 4：r--
# 3：-wx
# 2：-w-
# 1：--x
# 0：---

# 字母模式
chmod u+x file  # 用户加执行
chmod g+w file  # 组加写
chmod o-r file  # 其他去读
chmod a+x file  # 所有人加执行

# 递归
chmod -R 755 /data

# 风险提示：chmod -R 777 / 是灾难性操作，会破坏系统安全

# 属主
chown mysql:mysql file
chown -R mysql:mysql /data

# 改组
chown :mysql file
```

### 8.7 ln

```
# 软链
ln -s /data/file.txt /tmp/file.txt

# 硬链
ln /data/file.txt /tmp/file.txt

# 软链常用场景：
# - 链接到当前版本的代码
# - 链接到日志目录

# 看软链指向
readlink /tmp/file.txt
# /data/file.txt
```

### 8.8 rsync

```
# 本地同步
rsync -av /src/ /dst/
# 末尾 / 表示目录内容
# 不带 / 表示目录本身

# 远程同步
rsync -av /src/ user@remote:/dst/

# 排除
rsync -av --exclude='*.log' /src/ /dst/

# 断点续传
rsync -avP /src/ user@remote:/dst/

# 删除目标多余文件（同步删除）
rsync -av --delete /src/ /dst/
# 风险：--delete 目标端多出来的文件会被删，先 dry-run

# dry-run
rsync -avn --delete /src/ /dst/

# 限速
rsync -av --bwlimit=10m /src/ user@remote:/dst/

# 走 ssh key
rsync -av -e "ssh -i /root/.ssh/id_rsa" /src/ user@remote:/dst/
```

### 8.9 scp

```
# 上传
scp file.txt user@remote:/data/

# 下载
scp user@remote:/data/file.txt /tmp/

# 目录
scp -r /data/dir user@remote:/data/

# 限速
scp -l 1000 file.txt user@remote:/data/
# -l 1000：限 1000 Kbit/s

# 走 key
scp -i /root/.ssh/id_rsa file.txt user@remote:/data/
```

### 8.10 tar

```
# 打包
tar -cvf archive.tar /data
# -c：创建
# -v：详细
# -f：文件

# 解包
tar -xvf archive.tar

# 看内容不解
tar -tvf archive.tar

# gzip 压缩
tar -czvf archive.tar.gz /data
# -z：gzip

# 解 gzip
tar -xzvf archive.tar.gz

# bzip2 压缩
tar -cjvf archive.tar.bz2 /data
# -j：bzip2

# xz 压缩（最高压缩比）
tar -cJvf archive.tar.xz /data
# -J：xz

# 解到指定目录
tar -xvf archive.tar -C /dst/

# 排除
tar --exclude='*.log' -czvf archive.tar.gz /data
```

### 8.11 zip / unzip

```
# 压
zip -r archive.zip /data

# 解
unzip archive.zip

# 解到指定目录
unzip archive.zip -d /dst/

# 看内容
unzip -l archive.zip

# 排除
zip -r archive.zip /data -x "*.log"
```

## 九、日志命令

### 9.1 tail

```
# 看文件尾
tail file.log
# 默认 10 行

tail -n 100 file.log
# 100 行

tail -f file.log
# 实时跟踪（ctrl+c 退出）

tail -F file.log
# -F：跟踪 + 重试（文件被删重建也能继续追）
# 强烈推荐 -F
```

### 9.2 less / more

```
# 翻页查看
less file.log
# 空格下一页，b 上一页，/搜索，q 退出

# 打开就跳到尾
less +F file.log
# 类似 tail -f

# 多个文件
less file1.log file2.log
# :n 下一个，:p 上一个
```

### 9.3 grep

```
# 基础
grep "error" file.log
# 找含 error 的行

# 多个文件
grep "error" *.log
grep -r "error" /var/log/

# 大小写不敏感
grep -i "error" file.log

# 上下文
grep -A 5 "error" file.log
# -A 5：后面 5 行
grep -B 5 "error" file.log
# -B 5：前面 5 行
grep -C 5 "error" file.log
# -C 5：前后各 5 行

# 反选
grep -v "info" file.log
# 不含 info

# 计数
grep -c "error" file.log

# 行号
grep -n "error" file.log

# 正则
grep -E "ERROR|WARN" file.log
# 等价于 egrep
grep -E "192\.168\.[0-9]+\.[0-9]+" file.log
# 匹配 IP

# 限制目录
grep -r --include="*.log""error" /var/log/

# 排除目录
grep -r --exclude-dir=".git""TODO" /src/

# 统计
grep -c "error" /var/log/*.log
```

### 9.4 awk

```
# 基础
awk '{print $1}' file.log
# 打印第一列

awk '{print $1, $3}' file.log
# 打印第 1 和 3 列

# 自定义分隔符
awk -F: '{print $1}' /etc/passwd
# /etc/passwd 用 : 分隔

# 条件
awk '$3 > 100 {print $1}' file.log

# 多个分隔符
awk -F'[:,]''{print $1, $3}' file.log

# 计数
awk '{count++} END {print count}' file.log

# 求和
awk '{sum += $1} END {print sum}' file.log

# 复杂条件
awk '$3 > 100 && $4 == "OK" {print $0}' file.log

# BEGIN / END
awk 'BEGIN {print "start"} {print $1} END {print "end"}' file.log

# 关联数组
awk '{count[$1]++} END {for (k in count) print k, count[k]}' file.log
# 按第一列分组计数

# 处理 nginx access log
awk '{print $1}' access.log | sort | uniq -c | sort -rn | head
# 访问 IP 排行
```

### 9.5 sed

```
# 替换
sed 's/old/new/' file.log
# 替换每行第一个

sed 's/old/new/g' file.log
# 替换所有

sed -i 's/old/new/g' file.log
# 直接改文件
# 风险：-i 不可逆，先备份或先输出

# 删行
sed '/pattern/d' file.log
# 删含 pattern 的行

sed '5d' file.log
# 删第 5 行

sed '5,10d' file.log
# 删 5-10 行

# 打印
sed -n '5,10p' file.log
# 打印 5-10 行

# 多个命令
sed -e 's/old/new/g' -e '/^$/d' file.log

# 直接改文件
sed -i.bak 's/old/new/g' file.log
# 备份为 file.log.bak

# 行尾添加
sed 's/$/ END/' file.log

# 行首添加
sed 's/^/START /' file.log

# 提取行
sed -n '/start/,/end/p' file.log
# 提取 start 到 end 之间的行
```

风险提示：`sed -i` 是直接改文件，不可逆。先备份或先验证：

```
# 安全做法
sed 's/old/new/g' file.log > newfile.log
# 确认 OK 再覆盖
mv newfile.log file.log
```

### 9.6 journalctl

```
# 看 systemd 日志
journalctl

# 跟某个服务
journalctl -u nginx

# 实时跟踪
journalctl -u nginx -f

# 时间范围
journalctl --since "1 hour ago"
journalctl --since "2026-06-13 14:00" --until "2026-06-13 15:00"

# 优先级
journalctl -p err
# -p err：只显示 err 及以上
# 0: emerg
# 1: alert
# 2: crit
# 3: err
# 4: warning
# 5: notice
# 6: info
# 7: debug

# 内核日志
journalctl -k

# 看启动日志
journalctl -b
# -b -1：上次启动
# -b -2：上上次

# 占用
journalctl --disk-usage

# 清理
journalctl --vacuum-time=7d
journalctl --vacuum-size=500M
```

### 9.7 dmesg

```
# 内核日志
dmesg

# 实时
dmesg -w

# 时间戳
dmesg -T

# 看指定级别
dmesg --level=err,crit,alert,emerg

# 看 OOM
dmesg | grep -i "out of memory"

# 看网卡
dmesg | grep -i eth
```

### 9.8 logrotate

```
# 手动跑
logrotate -f /etc/logrotate.conf
# -f：强制跑（即使没到时间）

# 看配置
cat /etc/logrotate.d/nginx

# 验证配置
logrotate -d /etc/logrotate.conf
# -d：debug，不真转
```

## 十、用户与权限命令

### 10.1 useradd / usermod / userdel

```
# 加用户
useradd -m -s /bin/bash username
# -m：建 home
# -s：指定 shell

# 改用户
usermod -aG sudo username
# 加 sudo 组
# 注意 -a 不能漏，否则覆盖

# 改 shell
usermod -s /bin/zsh username

# 锁用户
usermod -L username

# 删用户
userdel -r username
# -r：连带 home
```

### 10.2 passwd / chage

```
# 改密码
passwd username

# 强制定期改
chage -M 90 username
# 90 天

# 立即过期
chage -d 0 username

# 看密码策略
chage -l username
```

### 10.3 groupadd / gpasswd

```
groupadd developers

# 加成员
gpasswd -a username developers

# 删成员
gpasswd -d username developers

# 看用户组
id username
groups username
```

### 10.4 su / sudo

```
# 切用户
su - username
# 加载环境变量

# 跑单条命令
sudo -u mysql /usr/bin/mysqld --version

# 看 sudo 权限
sudo -l
```

### 10.5 visudo

```
# 改 sudo 配置
visudo
# 用 visudo 而不是直接 vim，会检查语法

# 加一条
username ALL=(ALL) NOPASSWD: ALL
# ALL 主机
# (ALL) 用户
# NOPASSWD：免密
```

风险提示：`username ALL=(ALL) NOPASSWD: ALL` 给某个用户全 sudo 免密，等于 root。

## 十一、服务管理

### 11.1 systemd 服务

```
# 服务文件位置
/etc/systemd/system/
/usr/lib/systemd/system/

# 自定义服务
cat > /etc/systemd/system/myapp.service << 'EOF'
[Unit]
Description=My Application
After=network.target

[Service]
Type=simple
User=myuser
WorkingDirectory=/opt/myapp
ExecStart=/usr/bin/python3 /opt/myapp/main.py
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl start myapp
systemctl enable myapp
```

### 11.2 systemd-analyze

```
# 看启动耗时
systemd-analyze

# 看每个服务耗时
systemd-analyze blame

# 关键路径
systemd-analyze critical-chain

# 启动图
systemd-analyze plot > /tmp/boot.svg
# 浏览器打开 /tmp/boot.svg
```

### 11.3 journalctl 过滤

见 9.6。

## 十二、包管理

### 12.1 yum / dnf（RHEL/CentOS）

```
# 装
yum install -y nginx
yum install -y epel-release

# 卸
yum remove nginx

# 升级
yum update
yum update nginx

# 查包
yum search nginx
yum list | grep nginx
yum info nginx

# 看已装
yum list installed
yum list installed | grep nginx

# 查某个文件属于哪个包
yum provides /usr/bin/htop

# 清理
yum clean all
yum makecache

# 仓库
yum repolist
yum-config-manager --enable epel

# 锁定版本
yum install -y yum-plugin-versionlock
yum versionlock nginx-1.20.1-1.el7
```

### 12.2 apt（Ubuntu/Debian）

```
# 装
apt-get install -y nginx

# 卸
apt-get remove nginx
apt-get purge nginx
# purge：连配置一起删

# 升级
apt-get update
apt-get upgrade
apt-get dist-upgrade
# dist-upgrade 允许替换依赖关系

# 查包
apt-cache search nginx
apt list --installed

# 查文件
apt-file search /usr/bin/htop
# 没装先装 apt-file
# apt-get install -y apt-file
# apt-file update

# 锁版本
apt-mark hold nginx
apt-mark unhold nginx
```

### 12.3 rpm

```
# 装
rpm -ivh package.rpm
# -i：install
# -v：verbose
# -h：进度

# 卸
rpm -e nginx

# 升级
rpm -Uvh package.rpm

# 查
rpm -qa
rpm -qa | grep nginx

# 看包详情
rpm -qi nginx

# 看包文件
rpm -ql nginx

# 查文件属于哪个包
rpm -qf /usr/bin/htop
```

### 12.4 dpkg

```
# 装
dpkg -i package.deb

# 卸
dpkg -r nginx
dpkg -P nginx
# -P：purge，连配置

# 查
dpkg -l
dpkg -l | grep nginx

# 查包文件
dpkg -L nginx

# 查文件属于哪个包
dpkg -S /usr/bin/htop
```

## 十三、定时任务

### 13.1 crontab

```
# 编辑
crontab -e
# 用户级 crontab

# 看
crontab -l

# 删
crontab -r
# 风险：直接删整张表，慎用

# 系统级
/etc/crontab
/etc/cron.d/
/etc/cron.daily/
/etc/cron.hourly/
/etc/cron.weekly/
/etc/cron.monthly/
```

crontab 格式：

```
# 分 时 日 月 周 命令
# 0 2 * * * /opt/backup.sh
# */5 * * * * /opt/check.sh
# 0 0 * * 0 /opt/weekly.sh
# 30 9 * * 1-5 /opt/workday.sh
```

### 13.2 flock 防重入

```
# 防止脚本并发跑
* * * * * flock -n /tmp/mytask.lock -c "python3 /opt/mytask.py"
# -n：拿不到锁就放弃
# /tmp/mytask.lock：锁文件
# -c：跟命令

# 锁文件位置
# -w /var/lock/mytask.lock

# 写法
flock -n /var/lock/mytask.lock python3 /opt/mytask.py
```

### 13.3 at

```
# 一次性定时
echo "command" | at 14:30
echo "command" | at now + 5 minutes
echo "command" | at 2026-06-15

# 看队列
atq

# 删
atrm <job-id>
```

### 13.4 systemd-timer

更现代的定时方式：

```
# /etc/systemd/system/backup.timer
[Unit]
Description=Daily Backup

[Timer]
OnCalendar=daily
OnCalendar=Mon..Fri 02:00
Persistent=true

[Install]
WantedBy=timers.target
```

```
# /etc/systemd/system/backup.service
[Unit]
Description=Backup

[Service]
ExecStart=/opt/backup.sh
```

```
systemctl daemon-reload
systemctl enable --now backup.timer
systemctl list-timers
```

## 十四、性能分析命令

### 14.1 strace

```
yum install -y strace

# 跟踪进程
strace -p <pid>

# 跟踪新进程
strace command

# 看系统调用
strace -p <pid> -e trace=network
# -e trace=network：只看网络相关
# -e trace=open,read,write：看文件操作
# -e trace=memory：内存相关

# 统计系统调用
strace -p <pid> -c

# 看时间
strace -p <pid> -t

# 输出到文件
strace -p <pid> -o /tmp/strace.log
```

风险提示：strace 在生产慎用，会让进程变慢，且高并发场景下日志会爆。

### 14.2 ltrace

```
yum install -y ltrace

# 跟踪库调用
ltrace -p <pid>

# 跟指定库
ltrace -p <pid> -l "libssl*"
```

### 14.3 perf

见 4.5。

### 14.4 bcc 工具

```
# 装（需要 kernel-devel）
yum install -y bcc-tools

# 工具位置
ls /usr/share/bcc/tools/
# biolatency  tcpconnect  opensnoop  ...

# biolatency：块设备 IO 延迟
/usr/share/bcc/tools/biolatency 1 10
# 输出 latency 分布

# tcpconnect：实时 TCP 连接
/usr/share/bcc/tools/tcpconnect
# 看谁在连谁

# opensnoop：跟踪 open 系统调用
/usr/share/bcc/tools/opensnoop
# 看哪些文件被打开

# execsnoop：跟踪 exec
/usr/share/bcc/tools/execsnoop
# 看哪些进程被启动

# capable：跟踪 capability 检查
/usr/share/bcc/tools/capable
```

### 14.5 bpftrace

```
yum install -y bpftrace

# 简单 one-liner
bpftrace -e 'tracepoint:syscalls:sys_enter_openat { printf("%s %s\n", comm, str(args->filename)); }'

# 看进程执行的命令
bpftrace -e 'tracepoint:syscalls:sys_enter_execve { printf("%s -> %s\n", comm, str(args->filename)); }'
```

风险提示：bcc / bpftrace 在生产慎用，可能导致性能波动。

## 十五、文本三剑客

### 15.1 xargs

```
# 把 stdin 转成命令参数
echo"a b c" | xargs -n 1 echo
# a
# b
# c
# -n 1：每次传 1 个参数

# 删 .log 文件
find . -name "*.log" | xargs rm

# 用 -I 替换
find . -name "*.log" | xargs -I {} mv {} /backup/

# 限制并发
find . -name "*.txt" | xargs -P 4 -I {} cp {} /dst/
# -P 4：4 个并发
```

### 15.2 sort

```
# 排序
sort file

# 数字排序
sort -n file

# 倒序
sort -rn file

# 按列
sort -k 2 file
# 按第 2 列

# 去重
sort -u file

# human-readable 排序
sort -h file
# 1K, 2M, 3G
```

### 15.3 uniq

```
# 去重（要求已排序）
sort file | uniq

# 计数
sort file | uniq -c | sort -rn
# 出现次数排行

# 看重复
sort file | uniq -d

# 看非重复
sort file | uniq -u
```

### 15.4 cut

```
# 按列
cut -d: -f1 /etc/passwd
# -d：分隔符
# -f：取第几列

# 按字符
cut -c1-10 file
# 取每行 1-10 字符
```

### 15.5 tr

```
# 替换
echo "hello" | tr 'a-z' 'A-Z'
# HELLO

# 删字符
echo "hello 123" | tr -d '0-9'
# hello

# 压缩重复
echo "hello      world" | tr -s ' '
# hello world
```

### 15.6 column

```
# 对齐输出
mount | column -t

# 自定义分隔符
cat /etc/passwd | column -t -s:
```

### 15.7 paste

```
# 合并两个文件
paste file1 file2
# 1行 file1 1行 file2 拼一起

# 自定义分隔符
paste -d'|' file1 file2
```

### 15.8 diff / comm

```
# diff
diff file1 file2

# 统一格式
diff -u file1 file2
# 生成 patch

# comm
comm file1 file2
# 三列：只在 file1、只在 file2、共有

# 抑制某列
comm -23 file1 file2
# 只看 file1 独有的
comm -13 file1 file2
# 只看 file2 独有的
```

## 十六、实用工具

### 16.1 tmux

```
yum install -y tmux

# 新建 session
tmux new -s mysession

# 分离
# Ctrl+b d

# 列 session
tmux ls

# 重新连接
tmux attach -t mysession

# 快捷键
# Ctrl+b c：新建 window
# Ctrl+b n / p：next / prev window
# Ctrl+b "：水平分屏
# Ctrl+b %：垂直分屏
# Ctrl+b 方向键：切换 pane
# Ctrl+b x：关闭 pane
# Ctrl+b [：进入复制模式（PgUp/PgDn 翻页）
# Ctrl+b ]：粘贴
# Ctrl+b ?：帮助
```

### 16.2 screen

```
yum install -y screen

# 新建
screen -S mysession

# 分离
# Ctrl+a d

# 列
screen -ls

# 重新连接
screen -r mysession
```

### 16.3 glances

```
yum install -y glances

glances
# 综合监控
# CPU / 内存 / 网络 / 磁盘 / 进程 / 传感器 / 文件系统
# 支持 web 模式
glances -w

# 输出到文件
glances --export csv --export-csv-file /tmp/glances.csv
```

### 16.4 dstat

```
yum install -y dstat

# 组合输出
dstat -cdngym 1
# c: cpu
# d: disk
# n: net
# g: page
# y: sys
# m: mem

# 高级
dstat --top-cpu --top-io --top-mem 1
```

### 16.5 nmon

```
# 下载
wget http://sourceforge.net/projects/nmon/files/nmon16e_mpginc.tar.gz
tar xzf nmon16e_mpginc.tar.gz
chmod +x nmon_x86_64_rhel7
mv nmon_x86_64_rhel7 /usr/local/bin/nmon

# 跑
nmon
# 进入交互
# c：CPU
# m：内存
# d：磁盘
# n：网络
# t：top 进程
# h：帮助

# 录数据
nmon -f -s 5 -c 60
# -f：文件
# -s 5：5 秒一次
# -c 60：录 60 次 = 5 分钟
# 输出 hostname_YYMMDD_HHMM.nmon 文件
# 用 nmon analyser 打开（Windows 工具）
```

### 16.6 sysstat

sysstat 是一组工具的集合：

- `iostat`：磁盘 IO
- `mpstat`：CPU
- `pidstat`：进程
- `sar`：系统活动
- `sadc`：数据收集器
- `sa1` / `sa2`：cron 跑的脚本

```
# 启用历史收集
vim /etc/cron.d/sysstat
# 取消这行注释
# */5 * * * * root /usr/lib64/sa/sa1 1 1

# 启用 sadc
systemctl enable sysstat
systemctl start sysstat

# 看历史
ls /var/log/sa/
# sa13: 二进制
# sar13: 文本

sar -u -f /var/log/sa/sa13
sar -d -f /var/log/sa/sa13
sar -r -f /var/log/sa/sa13
sar -n DEV -f /var/log/sa/sa13
```

## 十七、SSH 与远程

### 17.1 ssh

```
# 登录
ssh user@host

# 指定端口
ssh -p 2222 user@host

# 跑命令
ssh user@host "ls /tmp"

# 传文件
scp file user@host:/tmp/

# 免密登录
ssh-keygen -t rsa -b 4096
ssh-copy-id user@host

# 跳板
ssh -J user@jumphost user@dest

# 端口转发
# 本地转发
ssh -L 8080:remote-host:80 user@host
# 访问本地 8080 转到 remote-host:80

# 远程转发
ssh -R 8080:localhost:80 user@host
# 远程访问 host:8080 转到本地 80

# 动态转发（SOCKS 代理）
ssh -D 1080 user@host
# 本地 1080 起 SOCKS 代理
```

### 17.2 ssh config

```
vim ~/.ssh/config
# Host prod-redis
#     HostName 10.20.0.11
#     User deploy
#     Port 22
#     IdentityFile ~/.ssh/id_rsa

# 之后直接
ssh prod-redis
```

### 17.3 ssh-agent

```
# 启动 agent
eval $(ssh-agent)

# 加 key
ssh-add ~/.ssh/id_rsa

# 看
ssh-add -l

# 锁
ssh-add -x

# 解锁
ssh-add -X
```

## 十八、网络调试命令

### 18.1 ping / ping6

```
# 测连通
ping 8.8.8.8

# 限次
ping -c 5 8.8.8.8

# 限时间
ping -w 10 8.8.8.8

# IPv6
ping6 ipv6.google.com
```

### 18.2 host / dig / nslookup

```
# 简单查
host example.com
# example.com has address 93.184.216.34

# dig
dig example.com
dig example.com +short
dig @8.8.8.8 example.com
# 指定 DNS

# 反向
dig -x 8.8.8.8

# MX 记录
dig example.com MX

# 跟踪
dig +trace example.com

# nslookup（老的）
nslookup example.com
```

### 18.3 telnet

```
# 测端口
telnet 10.20.0.10 80
# 连上说明端口通
# 风险提示：telnet 传明文，生产慎用

# 替代用 nc
nc -zv 10.20.0.10 80
```

## 十九、运维实战套路

### 19.1 排查一个慢请求

```
# 1. 看进程（CPU 高的）
top -c
# 找到 PID

# 2. 看进程详情
ps -p <pid> -o pid,ppid,user,stat,pcpu,pmem,vsz,rss,etime,cmd

# 3. 看进程 IO
pidstat -d -p <pid> 1

# 4. 看进程打开的文件
lsof -p <pid>

# 5. 看进程系统调用
strace -p <pid> -c
# 统计
strace -p <pid>
# 实时

# 6. 看进程内存
pmap -x <pid>

# 7. 看进程网络连接
ss -tnp | grep <pid>

# 8. 看应用日志
tail -F /var/log/app.log
```

### 19.2 找占用端口的进程

```
# 1. 看端口
ss -tlnp | grep 80
# LISTEN 0  128  *:80  *:*  users:(("nginx",pid=1234,fd=6))

# 2. 看进程
lsof -i :80

# 3. 找到 PID 后看进程
ps -p 1234 -o cmd

# 4. 找到配置文件
ls -l /proc/1234/cwd
# 进程工作目录
```

### 19.3 找大文件

```
# 1. 看磁盘
df -h
# 找满的盘

# 2. 看大目录
du -sh /* 2>/dev/null | sort -h | tail

# 3. 看大目录的子目录
du -sh /var/* | sort -h | tail

# 4. 看具体大文件
find / -type f -size +1G 2>/dev/null -exec ls -lh {} \;
find / -type f -size +100M 2>/dev/null -exec du -h {} \;

# 5. 排序
find / -type f -size +100M 2>/dev/null -exec du -h {} \; | sort -h | tail -20
```

### 19.4 找吃 CPU 的进程

```
# 1. top
top -c
# 找 %CPU 高的

# 2. ps
ps aux --sort=-%cpu | head

# 3. pidstat
pidstat -u 1

# 4. 找到进程后看堆栈
jstack <pid>  # Java
# 或
strace -p <pid> -c
# 统计

# 5. 看调用栈
perf top -p <pid> -g
```

### 19.5 看端口连接数

```
# 总连接数
ss -s

# 各状态
ss -tan | awk 'NR>1{print $1}' | sort | uniq -c

# 某 IP 的连接
ss -tan | grep 10.20.0.10 | wc -l

# 哪 IP 最多连接
ss -tan | awk 'NR>1{print $5}' | cut -d: -f1 | sort | uniq -c | sort -rn | head

# 哪端口被连接最多
ss -tan | awk 'NR>1{print $4}' | cut -d: -f2 | sort | uniq -c | sort -rn | head

# ESTABLISHED 状态
ss -tan state established | wc -l
```

### 19.6 网络抖动排查

```
# 1. 看连通
ping -c 100 host
# 看延迟 / 丢包

# 2. 路由追踪
mtr -r -c 100 host

# 3. TCP 重传
sar -n TCP,ETCP 1
# 看 retrans/s

# 4. 网卡丢包
ifconfig eth0
# 看 RX/TX dropped

# 5. 网卡 ring buffer
ethtool -g eth0

# 6. 网卡统计
ethtool -S eth0 | grep -i drop
```

### 19.7 CPU 飙高排查

```
# 1. 看 top
top -c

# 2. 看进程
ps -eo pid,pcpu,pmem,cmd --sort=-pcpu | head

# 3. 看进程线程
top -H -p <pid>
# 找到线程

# 4. 线程转 16 进制
printf"%x\n" <tid>
# 1a1b

# 5. 看 jstack
jstack <pid> | grep -A 30 "1a1b"
# Java

# 6. 看 perf
perf top -p <pid> -g
# C/C++ / Go

# 7. 看系统调用
strace -p <pid> -c
```

### 19.8 内存泄漏排查

```
# 1. 看 free
free -h

# 2. 看 process
ps aux --sort=-%mem | head

# 3. 看进程内存变化
whiletrue; do
    ps -p <pid> -o rss,vsz
    sleep 5
done

# 4. pmap 看哪段大
pmap -x <pid> | sort -k2 -nr | head

# 5. Java 用 jmap / jstat
jmap -heap <pid>
jstat -gc <pid> 1s

# 6. valgrind（开发用）
valgrind --leak-check=full ./app
```

### 19.9 启动慢排查

```
# 1. 看启动耗时
systemd-analyze

# 2. 看哪个服务慢
systemd-analyze blame

# 3. 看关键路径
systemd-analyze critical-chain

# 4. 看启动日志
journalctl -b
# 上次启动失败的原因
```

## 二十、危险命令清单

下面是生产**绝对不要用**或者**要非常小心**的命令。每条都是真实的事故。

### 20.1 rm -rf

```
rm -rf /
# 删根目录
# 真实事故：脚本里没传路径，默认 . → 反复递归 → 系统全废
rm -rf /* > /dev/null
# 删根目录所有文件
rm -rf $VAR/*
# VAR 没定义 → rm -rf /*

# 安全做法
# 1. 用 trash-cli
yum install -y trash-cli
trash-put file
# 软删除

# 2. 删除前确认
ls $VAR/
rm -rf $VAR/*
# 看到 ls 输出再删

# 3. 限制 rm 范围
rm -rf /data/cache/*
# 先确认 /data/cache 存在且不是 /
```

### 20.2 dd if=/dev/zero of=/dev/sda

```
dd if=/dev/zero of=/dev/sda
# 清空整块磁盘

# 真实事故：
# if=/dev/zero of=/dev/sdb bs=1M
# 本来想写 sdb1（sdb 的第一个分区）
# 错写成 sdb（整块盘）
# 数据全废

# 安全做法
# 1. lsblk 看盘
lsblk
# 2. 确认盘符
fdisk -l /dev/sdb
# 3. 写到具体分区
dd if=/dev/zero of=/dev/sdb1 bs=1M count=100
# count=100 限制大小

# 4. 加 conv=sync 跳过错误
dd if=/dev/zero of=/dev/sdb1 bs=1M count=100 conv=sync
```

### 20.3 chmod -R 777

```
chmod -R 777 /
# 全部文件 777，安全灾难

# 真实事故：
# 一个新人 root 上 chmod -R 777 /var/www
# 整个 web 目录全 777
# 等于给攻击者开了后门

# 安全做法
# 1. 限制目录
chmod -R 755 /var/www/html
# 2. 文件 vs 目录分开处理
find /var/www -type f -exec chmod 644 {} \;
find /var/www -type d -exec chmod 755 {} \;
# 3. 永远不要用 777
```

### 20.4 kill -9 数据库

```
kill -9 mysqld
# 数据库可能丢数据
# 真实事故：
# kill -9 mysqld
# 之后数据库启动不起来
# 因为 redo log 没刷盘，崩溃恢复失败

# 安全做法
# 1. 先 kill 15
kill <pid>
# 给 30 秒清理

# 2. kill 9 之前先 SHUTDOWN
mysql -e "SHUTDOWN"
# MySQL 自己的清理

# 3. 看进程响应
kill -0 <pid>
# 不真杀，只检测
```

### 20.5 > /etc/

```
> /etc/passwd
# 清空 /etc/passwd
# 真实事故：
# echo "xxx" > /etc/sudoers  # 报错 Permission denied
# > /etc/environment         # 错写成 /etc/passwd
# 机器上所有用户进不来

# 安全做法
# 1. 用 >> 而不是 > （追加）
echo "xxx" >> /etc/environment
# 2. 改前 cp
cp /etc/passwd /etc/passwd.bak
# 3. 用 vim 编辑而不是重定向
vim /etc/passwd
```

### 20.6 history -c

```
history -c
# 抹操作记录
# 真实事故：
# 事故后 history -c
# 调查时不知道谁干了什么
# 失去追责依据

# 安全做法
# 1. 不要用 history -c
# 2. 保留 history 给 audit
# 3. 把 history 写到统一日志服务器
```

### 20.7 mkfs 写错

```
mkfs.ext4 /dev/sda
# 格式化整块盘，不是分区
# 数据全废

# 安全做法
# 1. mkfs 之前先确认盘
lsblk
fdisk -l /dev/sda
# 2. 加 -f 前确认
mkfs.ext4 /dev/sda1
# 写具体分区，不是 sda
```

### 20.8 fdisk / parted 写错

```
fdisk /dev/sdb
# 然后 w 写错
# 分区表破坏

# 安全做法
# 1. 改前备份分区表
sfdisk -d /dev/sda > /tmp/sda-part.bak
# 2. 重要盘先 umount
umount /data
# 3. 操作前看多一遍 fdisk -l
```

## 二十一、命令历史与审计

### 21.1 history

```
# 看历史
history

# 跑历史的某条
!1234

# 找命令
history | grep "rm -rf"

# 时间戳
export HISTTIMEFORMAT="%Y-%m-%d %H:%M:%S "
# 之后 history 显示时间

# 写到统一文件
export HISTFILE=/var/log/bash_history/$(whoami)_$(date +%Y%m).log
```

### 21.2 auditd

```
yum install -y auditd

# 看规则
auditctl -l

# 加规则：监控 /etc/passwd 修改
auditctl -w /etc/passwd -p wa -k passwd_changes

# 触发后看
ausearch -k passwd_changes

# 永久规则
vim /etc/audit/rules.d/audit.rules
-w /etc/passwd -p wa -k passwd_changes
-w /etc/shadow -p wa -k shadow_changes
-w /etc/sudoers -p wa -k sudoers_changes
```

### 21.3 sudo 日志

```
cat /var/log/secure
# CentOS / RHEL
cat /var/log/auth.log
# Ubuntu / Debian

# 看 sudo 行为
grep sudo /var/log/secure
```

## 二十二、命令效率提升

### 22.1 快捷键

```
# bash 快捷键
Ctrl + A    # 行首
Ctrl + E    # 行尾
Ctrl + U    # 删到行首
Ctrl + K    # 删到行尾
Ctrl + W    # 删一个词
Ctrl + R    # 反向搜索历史
Ctrl + L    # 清屏
Ctrl + C    # 中断
Ctrl + D    # 退出
Ctrl + Z    # 暂停

# 翻命令
↑ / ↓       # 历史

# Tab       # 补全

# ！        # 执行历史
!!          # 上条
!$          # 上条最后一个参数
!*          # 上条所有参数
!123        # 历史第 123 条
```

### 22.2 别名

```
# ~/.bashrc
alias ll='ls -lh'
alias la='ls -lah'
alias l='ls -CF'
alias rm='rm -i'
alias cp='cp -i'
alias mv='mv -i'
# -i：覆盖前确认
# 注意：rm -i 是个双刃剑，脚本里 rm 会被卡

alias h='history'
alias psg='ps -ef | grep'
alias ports='ss -tlnp'
alias myip='curl -s ifconfig.me'
alias reload='source ~/.bashrc'

# 服务器组
alias prod='ssh user@prod-01'
alias stage='ssh user@stage-01'

# 危险命令再确认
alias rm='echo "Use trash-put instead"'
```

### 22.3 函数

```
# 找大文件
bigfiles() {
    find / -type f -size +${1:-100M} 2>/dev/null | xargs du -h 2>/dev/null | sort -h | tail -${2:-20}
}
bigfiles 1G 10

# 杀进程
kp() {
    ps -ef | grep "$1" | grep -v grep
    echo -n "Kill? [y/N]: "
    read answer
    if [ "$answer" = "y" ]; then
        pkill "$1"
    fi
}

# 快速 SSH
sshhost() {
    ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null $@
}
```

### 22.4 提示符

```
# ~/.bashrc
PS1='\u@\h:\w\$ '
# user@host:cwd$

# 带颜色
PS1='\[\033[01;32m\]\u@\h\[\033[00m\]:\[\033[01;34m\]\w\[\033[00m\]\$ '

# 显示 git 分支
parse_git_branch() {
    git branch 2>/dev/null | grep '*' | sed 's/* //'
}
PS1='\u@\h:\w(\$(parse_git_branch))\$ '
```

## 二十三、命令组合实战

### 23.1 找空文件

```
find / -type f -empty

# 删空文件
find /tmp -type f -empty -delete
```

### 23.2 批量改后缀

```
# 全部 .txt 改成 .md
for f in *.txt; do
    mv "$f" "${f%.txt}.md"
done

# 用 rename
rename 's/\.txt$/.md/' *.txt
```

### 23.3 找被改过但没提交的文件大小

```
git status --porcelain | awk '{print $2}' | xargs -I {} ls -lh {}
```

### 23.4 跑远程命令

```
# 一台
ssh user@host "ls /tmp"

# 多台
for host in prod-01 prod-02 prod-03; do
    echo"=== $host ==="
    ssh user@$host"uptime"
done

# 用 pssh
pssh -h hosts.txt -i "uptime"
# hosts.txt 一行一个 host

# 用 ansible
ansible all -i hosts -m shell -a "uptime"
```

### 23.5 文件批量处理

```
# 批量改权限
find /data -type d -exec chmod 755 {} \;
find /data -type f -exec chmod 644 {} \;

# 批量打包
find /data -name "*.log" -exec tar -czf logs.tar.gz {} +

# 批量传
find /data -name "*.log" | xargs -I {} scp {} user@backup:/backup/
```

### 23.6 查 Linux 系统信息

```
# 一行查系统版本
cat /etc/os-release
# NAME="CentOS Linux"
# VERSION="7 (Core)"
# ID="centos"
# ID_LIKE="rhel fedora"

# Ubuntu
lsb_release -a
# No LSB modules are available.
# Distributor ID: Ubuntu
# Description:    Ubuntu 22.04.3 LTS
# Release:        22.04
# Codename:       jammy

# 内核
uname -a

# CPU
lscpu | head

# 内存
free -h

# 磁盘
df -h

# 网络
ip a

# 启动时长
uptime

# 整体
hostnamectl
```

## 二十四、命令时间与调度

### 24.1 time

```
# 看命令耗时
time ls
# real    0m0.001s
# user    0m0.000s
# sys     0m0.001s

# 详细
/usr/bin/time -v ls
#    User time (seconds): 0.00
#    System time (seconds): 0.00
#    Percent of CPU this job got: 0%
#    Elapsed (wall clock) time: 0:00.00
#    ...
```

### 24.2 timeout

```
# 命令超时
timeout 10 ping 8.8.8.8
# 10 秒后自动停

# 跑慢命令
timeout 600 long_running_command

# 超时后发信号
timeout --signal=9 10 ping 8.8.8.8
# 超时后 SIGKILL
```

## 二十五、文件传输与同步

### 25.1 rsync 详细

```
# 基本
rsync -av /src/ /dst/

# 远程
rsync -av /src/ user@host:/dst/

# 走 SSH 选项
rsync -av -e "ssh -p 2222" /src/ user@host:/dst/

# 排除
rsync -av --exclude='*.log' --exclude='cache/' /src/ /dst/

# 排除文件
rsync -av --exclude-from='exclude.txt' /src/ /dst/
# exclude.txt 一行一个模式

# 限速
rsync -av --bwlimit=10m /src/ user@host:/dst/

# 断点续传
rsync -avP /src/ user@host:/dst/
# -P：--partial --progress

# 同步删除
rsync -av --delete /src/ /dst/

# dry-run
rsync -avn --delete /src/ /dst/

# 压缩
rsync -avz /src/ /dst/

# 保留硬链
rsync -avH /src/ /dst/
# -H：硬链
```

### 25.2 scp 详细

```
# 基础
scp file user@host:/dst/

# 目录
scp -r dir user@host:/dst/

# 限速
scp -l 1000 file user@host:/dst/
# 1000 Kbit/s

# 走 key
scp -i ~/.ssh/key file user@host:/dst/

# 保留权限
scp -p file user@host:/dst/
```

## 二十六、磁盘管理命令

### 26.1 fdisk

```
# 看
fdisk -l

# 改
fdisk /dev/sda
# m 帮助
# p 打印
# n 新建
# d 删
# t 改类型
# w 写
# q 不保存退出
```

### 26.2 parted

```
# 看
parted /dev/sda print

# 改
parted /dev/sda mklabel gpt
parted /dev/sda mkpart primary xfs 0% 100%

# 删
parted /dev/sda rm 1

# 改大小
parted /dev/sda resizepart 1 200GiB
```

### 26.3 mkfs

```
# ext4
mkfs.ext4 /dev/sda1

# xfs
mkfs.xfs /dev/sda1 -L data -f

# btrfs
mkfs.btrfs /dev/sda1

# 加参数
mkfs.ext4 -L data -m 1 /dev/sda1
# -m 1：保留块 1%
# -L：卷标
```

### 26.4 mount

```
# 临时挂载
mount /dev/sda1 /data

# 写 fstab
blkid /dev/sda1
# 拿 UUID
echo "UUID=xxx /data xfs defaults,noatime 0 0" >> /etc/fstab

# 验证
mount -a

# 重新挂载
mount -o remount,rw /data
```

## 二十七、LVM 命令

```
# PV
pvcreate /dev/sdb
pvs
pvdisplay

# VG
vgcreate data-vg /dev/sdb
vgextend data-vg /dev/sdd
vgs
vgdisplay

# LV
lvcreate -L 100G -n data-lv data-vg
lvextend -l +100%FREE /dev/data-vg/data-lv
lvreduce -L 100G /dev/data-vg/data-lv
# 缩 LV 高危
lvs
lvdisplay

# 扩 FS
xfs_growfs /data
resize2fs /dev/data-vg/data-lv
```

## 二十八、文本统计与分析

### 28.1 wc

```
wc -l file
# 行数

wc -w file
# 词数

wc -c file
# 字节数

wc -m file
# 字符数
```

### 28.2 head / tail

```
head -n 10 file
# 前 10 行

tail -n 10 file
# 后 10 行

tail -F file
# 实时跟踪
```

### 28.3 split

```
# 按行数拆
split -l 1000 file prefix_
# 每个 1000 行

# 按大小拆
split -b 100m file prefix_

# 按行数拆带后缀
split -l 1000 -d -a 3 file prefix_
# prefix_000, prefix_001, ...
```

### 28.4 nl

```
# 加行号
nl file

# 空行不加
nl -ba file
```

## 二十九、其他常用

### 29.1 date / cal

```
date
date +%F
# 2026-06-13
date +%T
# 14:30:00
date +"%F %T"
# 2026-06-13 14:30:00

cal
# 当月日历
cal 6 2026
# 2026 年 6 月
```

### 29.2 echo / printf

```
echo "hello"
echo -e "a\tb"
# -e：解释转义
echo "$var"

printf "%s\n" "hello"
printf "%d %s\n" 123 "abc"
# 格式化输出
```

### 29.3 yes

```
yes | command
# 自动回答 y

yes "no" | command
# 回答 no
```

### 29.4 seq / xargs

```
seq 1 10
# 1 2 3 ... 10

seq 1 2 10
# 1 3 5 7 9
# 步长 2

# 配合
seq 1 10 | xargs -I {} echo "task {}"
```

### 29.5 bc

```
# 计算器
echo "1+1" | bc
# 2
echo "scale=2; 10/3" | bc
# 3.33
```

### 29.6 watch

```
# 每秒跑一次
watch -n 1 "ls -lh /tmp"
# 默认 2 秒

# 高亮变化
watch -d -n 1 "free -h"
```

### 29.7 man

```
# 帮助
man ls

# 查命令
man -k network
# 模糊查询

# 章节
man 5 passwd
# 1 命令
# 2 系统调用
# 3 库函数
# 4 设备
# 5 配置文件
# 6 游戏
# 7 其他
# 8 系统管理命令
```

## 三十、运维命令速查表

```
# 系统
uname -a, hostname, uptime, date, who, w, last

# 进程
ps aux, top, htop, kill, pkill, pgrep, pidof, pstree

# CPU
top, mpstat, sar -u, lscpu, perf top

# 内存
free, vmstat, sar -r, slabtop, pmap

# 网络
ip, ss, netstat, ping, mtr, traceroute, tcpdump, iftop, nethogs, curl, wget, nc, ethtool

# 磁盘
df, du, lsblk, blkid, mount, fdisk, parted, iostat, iotop, fuser, lsof

# 文件
find, locate, stat, file, tree, ls, ln, chmod, chown, rsync, scp, tar, zip, dd

# 日志
tail, head, less, grep, awk, sed, journalctl, dmesg, logrotate

# 用户
useradd, usermod, userdel, passwd, chage, groupadd, id, su, sudo

# 服务
systemctl, journalctl, systemd-analyze

# 包
yum, dnf, apt-get, rpm, dpkg

# 定时
crontab, at, flock, systemd-timer

# 性能
strace, ltrace, perf, bcc, bpftrace

# 工具
tmux, screen, glances, dstat, nmon
```

## 三十一、上线 Checklist

- [ ] bash / zsh 配好 alias
- [ ] HISTTIMEFORMAT 配好
- [ ] history 写到统一日志
- [ ] 命令行编辑键熟练
- [ ] 危险命令（rm / dd / chmod）有防线
- [ ] auditd 装上，关键路径监控
- [ ] sudo 日志在收
- [ ] ssh 免密 + config 配置
- [ ] tmux / screen 熟练使用
- [ ] 编辑器 vim / nano 熟练
- [ ] 监控告警命令背得出来（iostat / ss / df）
- [ ] 紧急排错 Runbook 写好
- [ ] 命令速查表存在 ~/.cheatsheet
- [ ] 所有工具版本号记录
- [ ] 包管理源换好
- [ ] 常用 alias 同步到所有机器
- [ ] 关键命令输出格式熟记
- [ ] 知根知底能解释每个命令

文章到这里就结束了。命令这一块，**没别的捷径，就是天天用、用熟**。但更重要的不是"会用"，而是"知道什么时候用、什么时候不用、用了风险是什么"。生产里 `rm -rf` 出事的，十有八九是"知道这命令、不知道后果"。

把这篇存一份到本地或者收藏夹，**日常忘了就翻一下**，比搜索引擎更准。

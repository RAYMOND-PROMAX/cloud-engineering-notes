# Linux 服务器突然无法登录：如何从控制台恢复 SSH

## 1. 问题背景

凌晨发布后，运维从办公网连接服务器一直超时；另一位同事说“昨天刚改过 sshd_config”。直接重启机器可能丢失现场，也不一定能恢复。SSH 登录失败可能发生在 DNS/路由、TCP 22、防火墙、sshd 服务、认证、账号状态、磁盘写满或资源耗尽中的任一层。控制台的价值是提供**不依赖 SSH 的带外通道**，让你看到主机本身是否健康。

本文以具备合法控制台权限的云主机或虚拟机为前提，按症状分层、带外控制台登录、只读取证、最小修复、回归和回滚的顺序写。示例地址与账号是占位。生产上的重启、改防火墙和修账号应遵循本单位变更流程。

## 2. 从客户端先分清故障类型

`Connection timed out` 常提示 TCP 路径未完成；`Connection refused` 说明目标路径上主动拒绝或无监听；`Permission denied (publickey)` 说明网络与 SSH 握手基本走通，问题偏向认证。提示“Host key has changed”则可能是合法重装、DNS 指错，也可能是安全事件，不能简单删除 known_hosts。

```bash
ssh -vvv -o ConnectTimeout=5 ops@server.example.com

nc -vz -w 3 server.example.com 22

getent ahosts server.example.com
```

```bash
ip route get 192.0.2.10
```

客户端日志可能包含用户名、地址与认证方式，分享时脱敏。不同故障层决定下一步；若只有某个办公出口失败，先查 VPN 与安全组，不要马上改服务器 SSH 配置。

## 3. 带外控制台的选择

云厂商网页控制台、VNC/串口控制台、虚拟化平台控制台和物理机 BMC 都属于不依赖正常 SSH 的入口。确认连接的是预期实例、区域和磁盘，保留故障时间和实例 ID。控制台也需登录凭据或受控救援机制；不要因为 SSH 坏了就临时把 root 密码发到群里。

```text
记录：实例 ID、区域、内网/公网 IP、控制台入口、
最后成功登录时间、最近的变更单和故障开始时间。
```

打开控制台后先看机器是否还在正常系统：是否卡在启动、文件系统报错或登录提示。若控制台本身无法登录且系统崩溃，进入云平台救援模式/单用户模式可能需要另行审批与停机窗口；恢复路径要根据发行版和平台文档确认。

## 4. 控制台登陆后先做只读检查

记录时间、主机、启动时长、负载、磁盘块和 inode，再看 sshd 状态与日志。不要第一时间清空日志或修改防火墙，这会破坏证据。若根分区或 `/var` 已满，sshd 可能因日志、PAM 或临时文件写入失败，先定位存储问题。

```bash
hostnamectl

uptime

df -hT / /var /tmp
```

```bash
df -i / /var /tmp

systemctl status sshd --no-pager || systemctl status ssh --no-pager
```

`sshd` 是 RHEL 系发行版常见 unit 名，Ubuntu/Debian 常用 `ssh`；不要把第一条失败当作服务不存在。

## 5. 查看 sshd 配置与日志

`sshd -t` 能检查配置语法，但不能证明所有用户和网络路径都可登录；修改配置前先备份，修改后先测试语法，再 reload。若 sshd 因配置错误启动失败，日志通常有行号或不支持的选项。

```bash
sudo /usr/sbin/sshd -t

sudo journalctl -u sshd -b --no-pager -n 150
```

```bash
sudo journalctl -u ssh -b --no-pager -n 150

sudo cp -a /etc/ssh/sshd_config /etc/ssh/sshd_config.before-recovery
```

如果配置由 `/etc/ssh/sshd_config.d/*.conf` 覆盖，主文件看起来正常也不够。检查实际 include 与有效配置；不同用户、来源地址的 Match 块会改变权限。

```bash
sudo rg -n 'Include|Match|Port|ListenAddress|AllowUsers|DenyUsers|PasswordAuthentication|PubkeyAuthentication' \
  /etc/ssh/sshd_config /etc/ssh/sshd_config.d

sudo /usr/sbin/sshd -T | rg '^(port|listenaddress|passwordauthentication|pubkeyauthentication|permitrootlogin) '
```

`sshd -T` 输出通用有效配置，包含 Match 时应用 `-C` 提供用户、主机和地址上下文后再解释。

## 6. 端口监听与防火墙

若服务正常，确认监听哪个地址和端口。`ListenAddress 127.0.0.1` 只允许本机访问；改端口后客户端、安全组与防火墙须同步。查看规则时先只读；防火墙关闭不是正式修复。

```bash
sudo ss -lntp | rg ':(22|2222)\b'
```

```bash
sudo nft list ruleset | rg -n '22|2222|drop|reject'

sudo iptables -S | rg '22|2222|DROP|REJECT'

sudo firewall-cmd --list-all
```

命令是否存在取决于发行版与网络组件。云主机还要看安全组/NACL 与公网入口；主机 22 监听正常，不代表公网 22 放行。

## 7. 认证失败：密钥与账号状态

`Permission denied` 时检查用户是否存在、是否被锁定、shell 是否允许登录、home 权限、`.ssh` 权限及 `authorized_keys` 内容。OpenSSH 的 StrictModes 会检查归属与权限；不要为了解决问题直接关闭 StrictModes。不要在日志里打印私钥。

```bash
getent passwd ops
```

```bash
sudo passwd -S ops

sudo namei -l /home/ops/.ssh/authorized_keys

sudo ls -ld /home/ops /home/ops/.ssh /home/ops/.ssh/authorized_keys
```

```bash
sudo journalctl -u sshd -b --no-pager | rg -i 'Authentication refused|Failed publickey|Permission denied'
```

如果实际 home 路径不是 `/home/ops`，以 `getent passwd` 输出为准。修权限前先确认是否有 ACL、集中认证和自动配置管理，避免手工改完又被覆盖。

## 8. PAM、密码与集中认证

公钥已被接受但登录仍断开，可能是 PAM 会话、过期账号、LDAP/SSSD、磁盘满或 shell 初始化脚本。日志里的 `Accepted publickey` 只说明认证阶段通过，不等于会话建立成功。检查系统认证服务、用户过期状态与 shell 启动报错。

```bash
sudo chage -l ops

systemctl status sssd --no-pager

sudo journalctl -b --no-pager | rg -i 'pam_sss|pam_unix|session.*failed' | tail -80
```

```bash
sudo -u ops sh -lc 'id; pwd'
```

若组织使用集中认证，修改本地账号不能解决身份后端故障，还可能破坏安全策略。需要区分本地救援账号与业务账号。

## 9. 资源耗尽导致无法登录

磁盘、inode、进程数、内存、文件描述符都可能使 sshd 不能 fork 新会话或写 PAM/日志。控制台保留现状后定位具体资源；不要立即删数据库目录。若系统 OOM，查看内核日志和受害进程，确认是不是 SSH 被杀或整个机器抖动。

```bash
free -h

ps -e --no-headers | wc -l

sudo journalctl -k -b --no-pager | rg -i 'oom|out of memory|fork|I/O error'
```

```bash
sudo lsof +L1 | head -80
```

已删除但仍打开的大日志可能让 `df` 高于 `du`；使用应用支持的日志重开或受控重启，不要盲目删除正在使用的文件描述符。

## 10. 配置修复的安全顺序

先备份受影响文件，修改最小的一项，运行 `sshd -t`，再用 `systemctl reload`（配置支持时）或受控 restart。**保持控制台会话和已有 SSH 会话不退出**，从独立客户端开新连接验证；确认第二条新会话能登录后才关闭救援入口。

```bash
sudo /usr/sbin/sshd -t

sudo systemctl reload sshd || sudo systemctl reload ssh

sudo systemctl is-active sshd || sudo systemctl is-active ssh
```

```bash
ssh -o ConnectTimeout=5 ops@server.example.com 'id; hostname'
```

如果改了端口，记得验证 `ssh -p 2222` 与安全组。不要靠重启整个服务器来验证一行配置。

## 11. 救援模式与离线修复

若系统无法正常启动或账号认证全损坏，需要经平台控制台选择 rescue/emergency 模式，或把系统盘挂到救援实例。确认根分区、加密卷与 LVM 映射，再只修改经过批准的目标配置。`rescue.target` 与 `emergency.target` 的服务、挂载和认证行为不同，不能把进入 shell 的方式写成各发行版通用一步命令。

```bash
lsblk -f

findmnt /

# 只读查看挂载，不在未确认设备前执行 fsck 或写入
mount | head -30
```

离线修复后按平台流程卸载、启动原系统，再从正常网络路径验证 SSH。不可随意改系统盘的所有权或覆盖 `/etc/shadow`，防止扩大故障。

## 12. 虚构案例：改坏 sshd_config

教学示例：一个发布脚本把 `Port` 写成不支持的格式，`sshd -t` 报具体行号，`systemctl status sshd` 显示启动失败。值班通过控制台备份配置、回退出错行、重跑 `sshd -t`、启动服务，确认 22 监听与客户端新会话成功，最后修发布脚本的配置校验。关键证据来自语法测试和日志，不是“重启后好了”。

```bash
sudo /usr/sbin/sshd -t
```

```bash
sudo systemctl restart sshd

sudo ss -lntp | rg ':22\b'
```

重启只在确认语法、维护窗口与控制台可用时执行；如果 `sshd -t` 仍失败，重启会让原有会话之外的登录路径继续不可用。

## 13. 预防下一次失联

保留一个受控带外入口，发布 SSH 配置时在 CI 跑 `sshd -t`；关键主机有配置备份与回滚脚本；从受控探测点检查 TCP 握手和登录能力。磁盘、inode、PAM/SSSD 和安全组变更也纳入告警。不要把 SSH 唯一访问路径建立在没有控制台权限的实例上。

```bash
# 部署前语法门禁，路径根据镜像环境调整
/usr/sbin/sshd -t -f ./sshd_config.candidate
```

```text
发布验收：配置语法通过、服务 Active、端口监听、
新会话登录成功、原始来源网络连通、救援入口仍可用。
```

## 14. 常见问题与阶段小结

**能 ping 却 SSH 超时？** ICMP 与 TCP/22 规则不同。**sshd 是 active 但不能登？** 查监听、入口、防火墙与认证日志。**密钥被拒后应关闭 StrictModes 吗？** 先修归属和权限。**控制台也进不去怎么办？** 用平台授权的救援/离线挂盘方式，别在不确认设备的情况下修改文件系统。

先用客户端错误确定故障层，控制台只读取证，按最小范围修复，然后保持救援通道、从原客户端验证新会话。修复后把具体根因变成发布检查或资源告警。

## 15. 主机启动与网络检查

**主机没启动完成，SSH 排障从启动链开始**

控制台显示文件系统检查、内核 panic 或 emergency shell，而不是正常 login 提示时，SSH 服务尚未启动。先读屏幕提示、`journalctl -b` 与失败 unit；若根文件系统只读或挂载失败，要处理存储与启动配置。此时客户端 TCP 22 超时只是结果，修改 sshd_config 不会让系统启动完成。

```bash
systemctl --failed

journalctl -b -p err --no-pager | tail -100
```

```bash
findmnt /

lsblk -f
```

在应急模式里可用工具和挂载状态与正常系统不同，先确认 `/` 是否只读，以及要修改的文件属于原系统还是救援系统。如果需要对损坏文件系统做修复，应按文件系统和存储平台文档在正确的卸载/离线状态执行，不能直接在已挂载的生产根分区上运行任意 `fsck`。

**sshd 启动失败时的分支**

`systemctl status sshd` 显示 failed，就看 `journalctl -u sshd` 的首个错误。常见原因有配置语法错误、端口被另一个进程占用、主机密钥不可读、运行目录缺失或权限错误。`sshd -t` 可验证配置与部分环境条件；修复必须对准报错，不靠循环重启碰运气。

```bash
sudo /usr/sbin/sshd -t
```

```bash
sudo journalctl -u sshd -b --no-pager -n 100

sudo ss -lntp | rg ':22\b'

sudo systemctl cat sshd
```

在 Ubuntu/Debian 把 unit 改成 `ssh`，使用对应日志。若配置管理每分钟回写错误内容，手工修复会马上失效；暂停或修正错误配置来源才是根治。

**主机监听正常，客户端仍超时**

sshd 监听 `0.0.0.0:22` 只说明本机 IPv4 接口在监听，公网安全组、NACL、VPN、防火墙和回程路由仍可能拦截。用控制台运行 `ss`、`nft`、`ip route`，同时从故障客户端发起连接，并在有权限时短时抓包。若服务器完全收不到 SYN，沿入口链路查；若收到并回复 SYN-ACK，客户端收不到，查回程。

```bash
sudo ss -lntp '( sport = :22 )'
```

```bash
ip -br addr

ip route

sudo tcpdump -ni any 'tcp port 22 and host 192.0.2.20' -c 50
```

云负载均衡或 NAT 场景下服务器看到的源 IP 可能不是客户端公网 IP，采样点和过滤条件应按实际拓扑调整。关闭防火墙不是定位证据，也可能暴露其他端口。

**Permission denied 的分支**

如果客户端收到了 SSH 认证挑战，TCP 路径与服务端进程至少部分正常。先用 `ssh -vvv` 看客户端实际提供的 key，以及是否被服务器接受；再读服务端认证日志。常见失误是本地 key 没被 agent 加载、服务器 `authorized_keys` 中公钥不一致、home 目录归属或权限错误。

```bash
ssh -vvv -i ~/.ssh/id_ed25519 ops@server.example.com
```

```bash
sudo ls -ld /home/ops /home/ops/.ssh /home/ops/.ssh/authorized_keys

sudo namei -l /home/ops/.ssh/authorized_keys

sudo journalctl -u sshd -b --no-pager | rg -i 'publickey|Authentication refused|invalid user'
```

`StrictModes` 默认检查路径归属与权限；仅调低文件权限不能修正错 owner。确认受影响用户的真实 home 路径和目录上 ACL，再按组织标准修复，不要把整个 home chmod 777。

**Accepted publickey 后立即断开**

认证日志有 Accepted，客户端仍看见 `Connection closed`，说明问题可能在会话建立。PAM、账号过期、无效 shell、`.bashrc` 错误、磁盘满、进程数限制、SSSD 后端超时都可能导致。先对照失败时间看 PAM/session 日志，再检查用户属性，避免重新发密钥造成无效工作。

```bash
getent passwd ops
```

```bash
sudo chage -l ops

sudo journalctl -b --no-pager | rg -i 'pam|session opened|session closed' | tail -100

sudo -u ops env -i HOME=/home/ops SHELL=/bin/bash /bin/bash --noprofile --norc -c 'id'
```

最后一条只检验基础账号与 shell，不会加载实际登录脚本；若它成功而正常登录失败，继续检查用户启动脚本与 PAM 配置。

**Host key 变化不能直接清 known_hosts**

客户端提示远端主机指纹变化，先核对实例是否重装、IP 是否复用、DNS 是否指向新主机。通过平台控制台在主机上读取服务端 host key 公钥指纹，与变更记录比对；未经核实直接 `ssh-keygen -R` 会跳过服务器身份验证，存在连接到错误或恶意主机的风险。

```bash
sudo ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
```

```bash
getent ahosts server.example.com

ssh-keygen -F server.example.com
```

确认是合法主机密钥轮换后，再按资产管理流程更新客户端 known_hosts；对大量自动化客户端应规划发布顺序和信任分发。

**SSH 密钥权限的正确边界**

私钥保留在客户端，不复制到服务器；服务器放公钥到指定用户的 authorized_keys。home、`.ssh`、authorized_keys 的归属与权限要满足 OpenSSH 检查和组织策略。修复时不要通过聊天发送生产私钥，也不要为了临时登录开启空密码或 root 全网密码登录。

```bash
sudo install -d -m 700 -o ops -g ops /home/ops/.ssh

sudo chmod 600 /home/ops/.ssh/authorized_keys
```

```bash
sudo chown ops:ops /home/ops/.ssh/authorized_keys
```

这些命令会修改文件；只在已确认用户、路径和配置管理方式后执行。`install -d` 不会自动修复已有 home 目录的所有权限问题，仍需 `namei -l` 核对完整路径。

**用 sshd -T -C 看 Match 规则**

`Match User`、`Match Address`、`AllowUsers`、`DenyGroups` 等会让不同账号和来源行为不同。只运行 `sshd -T` 而不提供连接上下文，可能看不到故障账号的真实策略。`-C` 指定 user、host、addr 等匹配参数后查询有效配置，具体字段格式以安装版本手册核对。

```bash
sudo /usr/sbin/sshd -T -C user=ops,host=server.example.com,addr=192.0.2.20 \
  | rg '^(allowusers|denyusers|pubkeyauthentication|passwordauthentication|permitrootlogin) '

sudo rg -n '^Match |^AllowUsers|^DenyUsers|^AllowGroups|^DenyGroups' \
  /etc/ssh/sshd_config /etc/ssh/sshd_config.d
```

如果问题只影响一个办公出口，Match Address 和安全组源网段都值得检查；如果所有用户都失败，优先服务或系统层。

**控制台救援也要保护证据**

重大故障时先记实例 ID、开机时间、最近配置变更和认证失败时间。若怀疑遭到入侵，保留日志与磁盘快照，并按安全团队流程操作；不要为了恢复登录先把所有认证限制关掉。若是容量或配置失误，保留故障前配置，先最小恢复入口，后续在维护窗口完成长期修复。

```bash
sudo cp -a /etc/ssh /root/ssh-backup-before-recovery

sudo journalctl -b --since '1 hour ago' --no-pager > /root/ssh-incident-journal.txt
```

```bash
sudo sha256sum /etc/ssh/sshd_config
```

备份路径需要足够空间和恰当权限；若根分区已满，先选外部受控存储或小范围导出，不能让备份动作进一步占满磁盘。

**回滚与二次登录验证**

恢复后在旧的控制台会话保持不退出，从原故障客户端建立一个新 SSH 会话，并验证目标用户、权限和业务命令。只看到 TCP 22 连接成功，不代表认证、PAM 和 shell 可用。若服务配置管理将旧错误重新下发，要在控制台观察一段时间，避免刚结案又失联。

```bash
ssh -o BatchMode=yes -o ConnectTimeout=5 ops@server.example.com 'id && hostname'

sudo /usr/sbin/sshd -t && sudo systemctl is-active sshd

sudo journalctl -u sshd --since '10 min ago' --no-pager | tail -50
```

验证账号应与真实业务维护账号一致；不能只用 root 从同机 `ssh localhost` 做验收。

## 16. 控制台取证和配置验证

**先确认控制台类型和权限**

云厂商串口控制台、虚拟化平台 VNC、物理机 BMC/KVM 与救援系统的入口不同，但目标一致：取得一条不依赖故障 SSH 的管理路径。先核对实例 ID、区域、操作系统、控制台登录身份以及值班授权。不要因为 SSH 失联就反复重启；重启可能冲掉内存中的故障证据，并使本来能用控制台恢复的系统进入更坏状态。

```bash
hostnamectl
```

```bash
date -u

who -b
```

进入控制台后记录时间和主机名，确认登录的是目标机器。若根文件系统只读或系统尚未完成启动，应先处理启动/磁盘问题；SSH 服务状态只是下游症状。

**从客户端错误区分分支**

`Connection timed out` 多指网络路径或防火墙；`Connection refused` 表示到达目标但监听端口无服务或被明确拒绝；`Permission denied` 则已抵达认证阶段。主机密钥警告不能忽略，必须核对服务器身份。客户端多开几个 `-v` 的日志可标明停在哪一阶段，但敏感地址和用户名应在分享前脱敏。

```bash
ssh -vvv -o ConnectTimeout=5 ops@server.example.com

nc -vz -w 3 server.example.com 22
```

```bash
getent ahosts server.example.com
```

若域名解析指到旧实例，控制台里改 `sshd_config` 也无用。用目标 IP 与 DNS 结果对照时，注意跳板机、负载均衡和 NAT；先明确客户端实际连接的是哪一台机器。

**控制台里确认网络和 sshd 的监听**

内核地址、路由、监听端口和主机防火墙依次看。`sshd` 可能正常运行，却只监听回环地址或另一张网卡；也可能从 22 改到了其他端口。先读取生效配置，再决定是否修改。发行版服务名可能是 `ssh` 或 `sshd`。

```bash
ip -br addr

ip route

sudo ss -ltnp | rg ':22|sshd'
```

```bash
sudo systemctl status sshd --no-pager || sudo systemctl status ssh --no-pager
```

确认安全组/云防火墙、子网 ACL、主机防火墙和本地路由，按出站到入站的路径逐段查。不要为了抢修永久关闭防火墙；临时规则也应限定源地址和失效时间。

**配置改动前保留副本并验证语法**

常见事故包括 `Include` 文件写错、`Match` 段覆盖全局规则、禁止密钥认证、监听端口和防火墙不一致。先复制配置与引用文件，按实际路径定位变更时间；`sshd -t` 只验证语法，不保证某账户在某来源地址一定能登录。修改前先读服务日志中的明确错误。

```bash
sudo cp -a /etc/ssh/sshd_config /etc/ssh/sshd_config.pre-recovery

sudo rg -n '^(Include|Match|Port|ListenAddress|PubkeyAuthentication|PasswordAuthentication|AllowUsers|DenyUsers)' /etc/ssh/sshd_config /etc/ssh/sshd_config.d

sudo /usr/sbin/sshd -t
```

```bash
sudo journalctl -u sshd -b --no-pager | tail -100
```

如果发行版提供 `ssh.service`，改读对应单元。`sshd -t` 可能因运行目录或 host key 缺失失败，应根据原始错误修复，不要盲目放宽认证配置。

**Match 生效配置的验证**

`sshd -T` 输出全局有效值；加 `-C user=...,host=...,addr=...` 可测试某登录条件下的配置。规则里 `Match Address` 是客户端源地址，跳板机登录时应填跳板机出口 IP；`host` 语义与反向解析相关，按实际规则选取。参数必须对应故障用户和来源，否则结果误导。

```bash
sudo /usr/sbin/sshd -T -C user=ops,host=client.example.com,addr=203.0.113.10 | rg '^(port|listenaddress|pubkeyauthentication|passwordauthentication|authorizedkeysfile|permitrootlogin) '

sudo rg -n -A 20 '^Match ' /etc/ssh/sshd_config /etc/ssh/sshd_config.d
```

若全局允许公钥，特定地址的 `Match` 却关闭公钥，客户端会看到认证失败。修复应改准确的规则块，避免用全局 `PasswordAuthentication yes` 绕过问题。

**公钥认证的文件与账户检查**

认证日志通常能区分密钥未匹配、权限拒绝和账号被锁。核对用户 home、`.ssh` 目录、`authorized_keys` 的所有者与权限，也要看目录上层是否允许 sshd 安全读取；部分环境使用集中身份或自定义 AuthorizedKeysCommand，不能默认只查一个文件。

```bash
getent passwd ops

sudo namei -l /home/ops/.ssh/authorized_keys
```

```bash
sudo stat -c '%U:%G %a %n' /home/ops /home/ops/.ssh /home/ops/.ssh/authorized_keys

sudo journalctl -u sshd -b --no-pager | rg -i 'publickey|bad ownership|authentication failure|account locked'
```

修正权限时应以原账户和发行版策略为准，不要对 home 目录递归 `chmod 777`。共享 home、NFS 和 SELinux 还可能造成 Unix 权限看起来正确而访问被拒绝。

**SELinux、PAM 与账户状态**

`Accepted publickey` 后马上断开，通常已过密钥验证，应看 PAM、登录 shell、home 目录、配额和会话资源。启用 SELinux 的系统还要看 audit 记录与文件标签。账户过期、密码过期和登录限制都可能阻止会话建立；不要把所有认证后失败归因为 sshd 本身。

```bash
sudo faillock --user ops 2>/dev/null || true

sudo chage -l ops
```

```bash
sudo ausearch -m AVC -ts recent 2>/dev/null | tail -40

sudo journalctl -b --no-pager | rg 'pam_sss|pam_unix|session opened|session closed|SELinux'
```

命令可用性取决于发行版。若需要更改账户状态，先确认变更原因与授权，优先恢复既定身份管理链路，不要随意创建高权限临时账户。

**临时恢复要限定范围和回滚时间**

例如防火墙误拦截，可只放行值班出口 IP 到 SSH 端口；如果要更改认证方法，先用第二条独立会话验证，再关闭临时入口。修改防火墙、PAM、SSH 配置都应记录原状态。旧的控制台会话在确认新连接稳定前不要退出。

```bash
sudo firewall-cmd --list-all 2>/dev/null || sudo nft list ruleset

sudo systemctl reload sshd || sudo systemctl reload ssh
```

```bash
ssh -o BatchMode=yes -o ConnectTimeout=5 ops@server.example.com 'id; hostname; uptime'
```

`reload` 之前必须执行语法校验；若服务已退出，需按日志修复后再启动。上述远程验收从另一台允许的管理机器执行，且要使用实际业务采用的跳板路径，而不是仅在本机 `ssh localhost`。

**故障案例：磁盘满导致 SSH 失败**

虚构情形：监控主机活着但无法登录，控制台显示 `/var` inode 为 100%，日志无法写入，PAM 会话失败。此时改端口或换密钥都不能解决。先找实际占用和 inode 生成者，清理经确认可再生的缓存，避免删除系统审计日志，再验证服务。

```bash
df -h / /var /tmp; df -i / /var /tmp

sudo journalctl -u sshd -b --no-pager | tail -80

sudo findmnt -T /var/log -o TARGET,SOURCE,FSTYPE,AVAIL
```

恢复少量空间后观察是否持续增长，针对产生小文件的任务修复保留策略。若根因是文件系统只读或存储故障，要优先处理块设备与文件系统，再重启 sshd。

**交接与长期预防**

完成恢复需要确认新连接成功、原控制台仍可访问、跳板路径可用、认证日志显示正确账户、临时放行规则已撤销。记录客户端错误、控制台时间线、配置差异和回滚动作。持续监控 SSH 服务、磁盘块与 inode、认证失败率及配置变更事件。

```bash
sudo /usr/sbin/sshd -t
```

```bash
sudo ss -ltnp | rg 'sshd|:22'

sudo journalctl -u sshd --since '10 min ago' --no-pager | rg 'Accepted|Failed|error'
```

不要把“控制台里服务 active”作为唯一关闭条件。真实验收是从原授权客户端重新登录、执行一条只读命令并退出；再测试第二次连接，确认配置 reload 后仍然有效。

## 17. 救援环境与认证分支

**控制台不可用时的救援分支**

如果串口控制台无输出、实例卡在引导阶段，先通过云平台状态检查或物理机 BMC 判断电源、内核和存储状态。必要时使用供应商提供的救援系统挂载系统盘，先做快照或备份，再检查配置与日志。救援系统中的 `/etc/ssh/sshd_config` 属于救援环境，目标系统盘通常挂在另一路径；改错根目录会让人以为修复无效。

```bash
lsblk -f

findmnt -o TARGET,SOURCE,FSTYPE,OPTIONS
```

```bash
sudo mount -o ro /dev/mapper/system-root /mnt/target
```

设备名只是示例。首先只读挂载确认系统盘，再按恢复流程决定何时可写。涉及加密卷、LVM、RAID 或文件系统检查时，遵循平台文档，不在未知卷上直接执行修复或强制挂载。

**启动失败不能靠重启 sshd 解决**

控制台若停在 emergency mode、initramfs 或文件系统检查，sshd 可能根本未启动。查看引导消息、根盘挂载、`/etc/fstab`、文件系统状态和最近改动。错误的 UUID 或不存在的网络挂载可能拖慢或阻止进入 multi-user target；先处理启动依赖，再查 SSH。

```bash
systemctl --failed --no-pager

journalctl -xb --no-pager | tail -120

findmnt /; cat /etc/fstab
```

对 `fstab` 的调整要保留副本，确认 UUID 与 `blkid` 对应，重启前运行挂载验证。不要把所有启动失败都归因于文件系统；内核升级、加密密钥和云初始化也可能有关。

**连接超时的逐跳调查**

先确认 DNS 与实例公网/私网 IP，再看客户端能否到达目标端口、目标机接口和路由是否正常、云防火墙和主机防火墙规则是否允许该源地址。控制台上用 `ss` 确认监听；必要时短时抓包判断 SYN 是否抵达，但捕获文件可能含敏感流量，应限时限端口保存。

```bash
sudo ss -ltnp | rg ':(22|2222)\b'
```

```bash
sudo tcpdump -ni eth0 'tcp port 22 and host 203.0.113.10' -c 20

sudo nft list ruleset | rg -n '22|drop|reject'
```

若目标看不到 SYN，重点查上游路径和 DNS；看到 SYN 却无正常 SYN-ACK，查本机防火墙、监听和回程路由；完成握手后认证失败则进入账号分支。不要仅靠 `ping` 通证明 TCP/22 可达。

**Connection refused 的定位**

明确拒绝通常表示目标 IP 可达但指定端口无人监听，或防火墙以 reject 回应。确认 sshd 是否退出、服务端口是否改变、`ListenAddress` 是否绑错地址，以及 IPv4/IPv6 的监听差异。若 sshd 重启循环，`systemctl status` 的瞬间 active 状态会误导，需看重启次数和上一次退出原因。

```bash
sudo systemctl show sshd -p ActiveState -p SubState -p NRestarts -p ExecMainStatus

sudo /usr/sbin/sshd -T | rg '^(port|listenaddress|addressfamily) '
```

```bash
sudo journalctl -u sshd -b --no-pager | tail -120
```

服务单元可能名为 `ssh`，按系统替换。改端口时同步检查云侧、主机侧和客户端 SSH config；恢复后不应留下一个没有审计的临时开放端口。

**认证失败要分清服务器拒绝与客户端没提供密钥**

`Permission denied (publickey)` 可能是客户端没找到私钥、ssh-agent 不工作、服务器不接受该算法或 `authorized_keys` 不匹配。客户端 `-vvv` 会列尝试的身份和服务器允许的方法；服务器日志则说明接受/拒绝原因。不要把私钥内容复制到控制台或工单。核对公钥指纹与目标账户后，再检查文件权限和生效配置。

```bash
ssh -vvv -o IdentitiesOnly=yes -i ~/.ssh/id_ed25519 ops@server.example.com

ssh-keygen -lf ~/.ssh/id_ed25519.pub

sudo journalctl -u sshd --since '15 min ago' --no-pager | rg 'ops|publickey|Failed|Accepted'
```

若用集中认证，用户目录中的 `authorized_keys` 可能不是权威来源，应查看 `AuthorizedKeysCommand`、身份服务健康和缓存状态。修复应恢复原认证路径，避免无期限启用密码登录。

**主机密钥变化的处理**

客户端出现 host identification changed 警告时，不要直接执行 `ssh-keygen -R` 然后接受新密钥。可能是实例重建、IP 复用、运维重装，也可能是中间人攻击。通过可信控制台读取目标的主机公钥指纹，与资产记录和变更单核对；确认合法变化后再更新客户端的 known_hosts。

```bash
sudo ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
```

```bash
ssh-keygen -F server.example.com

ssh-keyscan -t ed25519 server.example.com 2>/dev/null | ssh-keygen -lf -
```

`ssh-keyscan` 只能读取网络上对方声称的公钥，不能单独证明身份。若指纹不一致且无授权变更，停止连接并按安全事件处理。

**配置回滚演练**

恢复前保留原文件及修改记录，优先以最小改动修正具体错误。若最近一次部署引入坏配置，可以回滚该文件，但要考虑系统同时安装的 Include 片段和安全基线。执行 `sshd -t`、针对故障用户运行 `sshd -T -C`，再 reload；始终保持控制台或现有 SSH 会话打开，直到新的独立会话验证成功。

```bash
sudo diff -u /etc/ssh/sshd_config.pre-recovery /etc/ssh/sshd_config | head -100

sudo /usr/sbin/sshd -t && sudo systemctl reload sshd
```

```bash
ssh -o BatchMode=yes ops@server.example.com 'hostname; id'
```

命令中的 `&&` 保证语法检查失败时不 reload。若服务名为 `ssh`，替换单元名。回滚不是把安全选项全部关闭；新连接成功后确认原策略、来源限制和审计仍然有效。

**救援环境编辑目标系统文件**

需要离线修改时，先在目标根目录定位文件，检查软链接是否跨出该根目录，并复制原件。某些系统的 SSH 配置由云初始化或配置管理覆盖；离线编辑成功后首次启动又被覆盖，应先找配置来源。救援系统里运行 `sshd -t` 默认检查救援环境配置，可用 `-f` 指定目标文件，但其 Include、host key 与运行目录仍可能引用救援环境，不能完全代替目标系统启动验证。

```bash
sudo ls -l /mnt/target/etc/ssh/sshd_config /mnt/target/etc/ssh/sshd_config.d

sudo cp -a /mnt/target/etc/ssh/sshd_config /mnt/target/etc/ssh/sshd_config.before-rescue

sudo rg -n 'Include|Match|Port|PubkeyAuthentication' /mnt/target/etc/ssh/sshd_config
```

离线变更前确认文件系统可写、原盘快照可回滚；完成后卸载并正常启动，再用目标系统控制台核验服务。不要在救援环境里直接重启误以为已经重启目标系统。

**恢复后补上监控和管理入口**

一次 SSH 失联如果只能靠值班人员临时找控制台账号，说明恢复路径还不够可靠。演练至少包括管理入口可用、最小权限账户能登录、配置修改前校验、旧会话保持、监控发现监听和端口不可达。针对磁盘满、认证服务故障、证书变化和网络策略变更各保留一个可用分支。

```bash
systemctl is-active sshd || systemctl is-active ssh
```

```bash
sudo /usr/sbin/sshd -t

ssh -o BatchMode=yes -o ConnectTimeout=5 ops@server.example.com true
```

服务 active 只是本机状态；端到端测试要从真实跳板机或管理网络发起。记录恢复耗时、哪一步缺少权限或信息，并把配置变更的语法验证嵌入发布流程。

**一份事件报告的证据顺序**

报告按时间写客户端原始报错、控制台登录、主机启动与磁盘状态、监听与网络、服务日志、认证日志、配置差异、操作和复测结果。不要只写“重启 sshd 后恢复”，否则无法判断真正原因或避免复发。若是网络策略导致，记录变更单和命中规则；若是磁盘耗尽导致，记录增长源与清理策略。

```
| 时刻 UTC | 证据/动作 | 判断 |
| --- | --- | --- |
| 10:00 | 客户端超时 | 网络或监听待查 |
| 10:05 | 控制台 ss 显示监听正常 | 查网络路径 |
| 10:12 | 主机防火墙拒绝源 IP | 命中规则 |
| 10:18 | 授权源恢复登录 | 二次会话通过 |
```

```bash
sudo journalctl -u sshd --since '1 hour ago' --no-pager | tail -100
```

事件关闭时应撤销临时例外、更新资产信息和告警，并证明从原客户端可再次登录。若重启/改动造成业务副作用，也需在同一时间线注明。

## 18. 临时恢复、回滚和验收

**控制台里先做一次只读快照**

故障刚发生时，最有价值的是未经修改的状态：启动时间、失败单元、磁盘块和 inode、IP 与路由、监听、服务日志、认证日志。把这些输出存到受控工单或诊断目录，注意脱敏。重启、清日志、修改配置会改变证据；除非业务恢复需要立即操作，否则先用数分钟把关键数据取齐。不要把控制台截图里的密钥、主机地址和用户信息公开传播。

```bash
uptime -s

systemctl --failed --no-pager

df -h / /var /tmp; df -i / /var /tmp
```

```bash
sudo journalctl -u sshd -b --no-pager | tail -150
```

若机器资源已极度紧张，避免在满盘目录写大诊断文件；先通过控制台输出或远程安全渠道保存有限片段。日志单元名按发行版使用 `ssh` 或 `sshd`。

**防火墙变更如何最小化**

确认安全组或主机防火墙误拦后，只给已授权的管理出口地址、目标端口和有限时间开临时通路。修改前导出规则，操作后从真实跳板机测一次连接，并在故障解除后撤销例外。若是云安全组，则主机上的 `nft` 看不到规则；要在云侧核对优先级、入站方向、关联网卡和变更记录。

```bash
sudo nft list ruleset > /tmp/nft.before-recovery.txt

sudo firewall-cmd --list-all 2>/dev/null || true

nc -vz -w 3 server.example.com 22
```

具体放行命令因平台而异，不在通用文章里给“一键放行所有来源”。还要检查 IPv4/IPv6；一个地址族可用不代表另一个可用，客户端 DNS 可能优先解析到不可达的地址族。

**DNS 与旧 IP 的陷阱**

实例恢复或迁移后 IP 变化，客户端仍连向旧地址；某些 SSH config 的 `HostName` 覆盖了用户输入的域名，或者跳板机内部 DNS 与外部结果不同。比较客户端解析、SSH 配置解析和控制台当前接口地址。若 DNS TTL 尚未到期，故障可能只影响部分客户端；不能在服务器控制台修改 sshd 来修复指向旧主机的记录。

```bash
ssh -G server.example.com | rg '^(hostname|port|proxyjump|user) '
```

```bash
getent ahosts server.example.com

ip -br addr
```

变更 DNS 前确认正确实例身份和业务变更流程。若旧 IP 已被另一台主机使用，客户端主机密钥警告应视为身份校验信号，而不是简单的 known_hosts“脏数据”。

**端口改变后核对完整链路**

把 SSH 端口从 22 调到 2222，必须同步检查 sshd 生效配置、监听、防火墙、云安全组、跳板机和客户端配置。仅改 `Port` 再 reload，却忘记网络放行，会把最后一条管理入口切断。安全的操作是在控制台保留会话，校验语法，短时保留旧端口或预先开通新端口，再从独立客户端测试，最后按安全策略关闭旧入口。

```bash
sudo /usr/sbin/sshd -T | rg '^port '

sudo ss -ltnp | rg ':22|:2222'
```

```bash
ssh -p 2222 -o ConnectTimeout=5 ops@server.example.com 'hostname'
```

如果用了 `Match` 或 Include，按故障用户与来源地址检查生效配置。端口变更不是通用的安全修复，不应当作认证失败的替代方案。

**sshd 启动失败的配置证据**

启动失败时先运行 `sshd -t`，再读服务日志。常见原因包括无效关键字、重复/不合法参数、引用的 HostKey 不存在、权限问题、端口被占用、运行目录异常。不要看到 `status=255` 就推断只有配置错误；日志和 `sshd -t` 的具体提示才可分类。修正后先验证监听，再验证外部连接。

```bash
sudo /usr/sbin/sshd -t

sudo systemctl status sshd -l --no-pager

sudo journalctl -u sshd -b --no-pager | tail -100
```

```bash
sudo ss -ltnp | rg 'sshd|:22'
```

不要覆盖或重建现有主机密钥来“修复”任意启动错误；主机密钥变化会影响客户端信任。确实需要恢复密钥时，遵循备份和身份验证流程。

**Home 目录与公钥权限的另一层**

`authorized_keys` 自身模式正确不保证认证通过：home 上层目录可能可被其他用户写，文件属主不对，SELinux 标签错误，或 NFS root squash 影响访问。`namei -l` 可逐级看路径权限；`sshd -T -C` 可确定生效的 AuthorizedKeysFile。不要在多用户主机上递归改整个 `/home` 权限，修复应针对故障用户和具体文件。

```bash
sudo namei -l /home/ops/.ssh/authorized_keys

sudo /usr/sbin/sshd -T -C user=ops,host=client.example.com,addr=203.0.113.10 | rg 'authorizedkeysfile|strictmodes'

sudo ls -Z /home/ops/.ssh/authorized_keys 2>/dev/null
```

SELinux 标签检查只在启用相关机制的系统适用。若登录密钥来自外部身份源，要同时查看身份服务状态和延迟，不应默默复制公钥到本地绕开管理。

**登录后立刻退出的分支**

客户端显示 `Authenticated to ...` 但会话瞬间结束，应查 PAM session 阶段、默认 shell、home 是否可访问、磁盘和配额、登录脚本以及 systemd 用户会话。通过非交互命令与交互 shell 分别测试，可以缩小范围；若非交互成功而交互退出，检查 `.bashrc`、`.profile` 等文件里的错误命令，但不要直接执行不可信登录脚本。

```bash
getent passwd ops
```

```bash
sudo -u ops test -d /home/ops && echo home_ok

sudo journalctl -b --since '15 min ago' --no-pager | rg 'pam|session|ops|quota'
```

应用账号可能故意使用 `/usr/sbin/nologin`，不能为了“登录成功”改成 shell。先确认该账户是否本来应允许交互登录；特权访问需要符合既有审计流程。

**堡垒机路径与直接登录的差异**

控制台里 sshd 正常，机房内一台管理机直连也成功，但员工仍无法登录，问题可能在堡垒机认证、代理、来源白名单或 DNS。验收必须走原路径：相同用户、相同堡垒机、相同目标地址和端口。若只能直连测试，报告明确“直连已恢复、堡垒机链路待验证”，不能宣称用户登录恢复。

```bash
ssh -J bastion.example.com ops@server.example.com 'hostname; id'

ssh -G server.example.com | rg 'proxyjump|proxycommand|hostname|port'
```

```bash
sudo journalctl -u sshd --since '10 min ago' --no-pager | rg 'Accepted|Failed|Disconnected'
```

如果原链路经网关转换源 IP，`Match Address` 和防火墙规则要按实际源地址判断。联系堡垒机值班方时附请求时间与会话 ID，减少重复猜测。

**一次完整的虚构故障推演**

假设配置管理在 10:00 下发了 `Match Address` 规则，将来自堡垒机的公钥认证关闭。10:03 员工看到 `Permission denied`，控制台确认 sshd 监听正常、磁盘和网络正常；`sshd -T -C user=ops,addr=堡垒机出口` 显示 `pubkeyauthentication no`。保留配置快照，回滚该规则，语法校验并 reload，然后由原堡垒机新建连接成功。旧控制台会话全程保持。

```bash
sudo /usr/sbin/sshd -T -C user=ops,host=bastion.example.com,addr=203.0.113.10 | rg 'pubkeyauthentication|passwordauthentication'

sudo /usr/sbin/sshd -t && sudo systemctl reload sshd

ssh -J bastion.example.com ops@server.example.com 'hostname; id'
```

此案例是教学虚构，实际 `Match` 语义和源 IP 要现场确认。复盘要修复配置发布前的静态校验及针对管理出口的自动连接探针，避免同类错误再次落地。

**为什么恢复后还要二次验证**

一次成功登录可能碰巧沿用旧连接、DNS 缓存或未受影响的来源。关闭旧会话前，从一台独立管理客户端新建连接，退出后再连一次；检查认证日志中的实际用户、来源 IP、密钥类型和时间；同时确认临时开放规则已经撤销，配置管理不会在下一轮又覆盖修复。若采用多台跳板机，应至少覆盖故障所涉及的一条。

```bash
ssh -o ControlMaster=no -o ControlPath=none ops@server.example.com 'hostname; date -u'
```

```bash
sudo journalctl -u sshd --since '5 min ago' --no-pager | rg 'Accepted|Failed'

sudo /usr/sbin/sshd -t
```

验收记录应写明客户端地址、登录方式、目标实例和服务端日志证据。这样“控制台救回 SSH”才是一个可交接的闭环，而不是本地服务状态的一次偶然变绿。

## 19. 交接与长期预防

**“服务器突然无法登录”的五个起点**

第一类是主机未完成启动：控制台会看到 rescue/emergency、文件系统或内核错误；第二类是网络不可达：客户端超时且主机看不到 SYN；第三类是 sshd 未监听：客户端得到拒绝或主机日志有启动失败；第四类是认证失败：握手完成但公钥/PAM/账户状态不通过；第五类是登录成功却会话异常结束。分支依据应是客户端错误、控制台状态、监听与日志的组合，不凭某一个症状猜。

```bash
ssh -vvv -o ConnectTimeout=5 ops@server.example.com

sudo ss -ltnp | rg 'sshd|:22'
```

```bash
sudo journalctl -u sshd -b --no-pager | tail -80
```

每条命令都应带时间和目标机器信息。假如主机有多网卡、NAT 或不同地址族，控制台本机访问成功不能证明原客户端路径恢复。

**什么时候应考虑救援盘**

控制台无法登录、系统盘只读、root 文件系统挂载失败，或配置错误阻止进入正常用户空间时，可能需要离线救援。先有备份/快照，再识别卷、分区、加密和 LVM 层次；只读挂载取证后，才按经批准的恢复步骤写入。救援环境与目标系统有不同的 `/etc`、进程和内核，命令必须指向目标挂载目录。操作完检查文件权限、卸载、正常启动，再在目标系统里运行 `sshd -t` 和端到端连接。

```bash
lsblk -f

sudo mount -o ro /dev/mapper/system-root /mnt/target

sudo ls -l /mnt/target/etc/ssh/sshd_config
```

示例设备名不是通用路径。若底层存储有 I/O 错误，不应反复写盘以验证 SSH；先处理存储与数据保护。

**不能随手放宽的安全选项**

登录失败时将 `PermitRootLogin yes`、密码认证全部打开，或设置 `StrictModes no`，可能把一次可定位的配置错误变成长期暴露。先用 `sshd -T -C` 找具体规则，再核对授权身份、文件权限或身份服务。需要临时入口时，限定源、账号、时间，保留审计和撤销计划。旧管理会话打开到新连接验证结束，不要在 reload 前断开最后一条路径。

```bash
sudo /usr/sbin/sshd -T -C user=ops,host=client.example.com,addr=203.0.113.10 | rg 'permitrootlogin|passwordauthentication|strictmodes|pubkeyauthentication'
```

```bash
sudo /usr/sbin/sshd -t

sudo diff -u /etc/ssh/sshd_config.pre-recovery /etc/ssh/sshd_config
```

若当前控制台不是可信渠道，不要在其中输入新的高权限凭据；应由平台安全团队确认管理入口。恢复的目标是原授权路径正常工作，而非让任何路径都能登录。

**一个端到端验收清单**

语法检查成功、服务 active、端口监听、授权源网络可达、用户公钥认证成功、交互或非交互 shell 可正常运行、退出后第二次新建连接仍成功，才能算 SSH 恢复。若故障涉及堡垒机，验收必须经堡垒机；若涉及某来源地址的 `Match` 规则，应使用该来源验证。确认临时防火墙规则、临时账户或密码认证均已按约定撤销。

```bash
sudo /usr/sbin/sshd -t && sudo systemctl is-active sshd

ssh -o BatchMode=yes -o ControlMaster=no -o ControlPath=none ops@server.example.com 'hostname; id; date -u'
```

```bash
sudo journalctl -u sshd --since '5 min ago' --no-pager | rg 'Accepted|Failed|Disconnected'
```

第一条服务名可能要换 `ssh`；第二条从原管理客户端执行。完成后在工单里记录目标实例 ID、来源、用户、验证时间和日志片段，下一次可据此对照。

**长期机制比临时重启有效**

配置发布流水线加入 `sshd -t` 和代表性 `sshd -T -C` 检查；关键节点在管理网络部署 TCP/22 和真实公钥登录探针；保留独立控制台权限并定期演练。磁盘块、inode、PAM/身份服务、云安全组变更也纳入关联告警。若同一故障反复靠重启解决，说明根因尚未确认，应检查证据链和变更流程，而非把重启写成标准操作。

```bash
sudo /usr/sbin/sshd -t

systemctl list-unit-files | rg '^ssh(d)?\.service'
```

```yaml
runbook_example:
  owner: platform_oncall
  console_access: periodically_verified
  external_login_probe: managed_network
  config_validation: before_reload
```

这些是流程模板而非现成自动化配置。把“谁能进入控制台、谁能改防火墙、谁能恢复身份服务”的职责明确，能显著减少失联后的等待。

**三个常见恢复误区**

控制台里 `ssh localhost` 成功，不能证明外部网络、云安全组和跳板路径正常；`systemctl status` 显示 active，也不能证明监听地址与端口正确；公钥认证成功后即断开，不应继续只盯 `authorized_keys`，而要看 PAM、shell、home 和磁盘。每个误区的共同点是把链路中的一个局部成功当成端到端恢复。

```bash
sudo ss -ltnp | rg 'sshd|:22'

sudo journalctl -u sshd --since '15 min ago' --no-pager | rg 'Accepted|Failed|session'
```

```bash
ssh -J bastion.example.com ops@server.example.com 'hostname; id'
```

验收第三条必须从授权客户端执行，并按实际部署选择是否需要跳板。所有测试结果带来源地址、目标实例和时间，避免将不同机器的成功证据混在一起。

**故障处理中的交接边界**

云平台控制台权限、主机 root 权限、身份服务管理权限与网络策略权限往往分属不同团队。值班人员应先指出当前分支和证据，再向相应责任方提供具体请求：例如“目标机 22 端口监听，抓包没有来自堡垒机的 SYN，请核对云侧安全组”。若日志显示 PAM 拒绝，则联系身份服务负责方，同时保持控制台可用。这样的交接减少重复重启和放宽策略。

恢复后把临时变更逐项撤销并复测。尤其留意配置管理的下一次自动下发、云安全组的临时规则和手工追加的公钥。若某一步无法确认，就在事件报告里标注风险和责任时间，而不是把状态写成完全恢复。

**交接时的验证用语**

恢复通告应包含“从哪一个授权管理源，以哪个账户，通过哪个端口和跳板，在什么时间新建了 SSH 会话；服务端日志显示 Accepted；临时变更已撤销”。如果只是控制台能进入，写“控制台可用、外部 SSH 仍待验证”；如果只能单个来源登录，写清影响范围。准确的状态能防止其他团队误以为业务已经全恢复。

同时保留未解决的风险：比如实例启动虽正常但系统盘仍有 I/O 错误、身份服务偶发超时、云安全组变更尚未回滚。安排跟进责任和时间，避免临时放行成为永久入口。事故演练的目标不是记住一串命令，而是让下一班值守能沿证据找到具体故障段，并在保留管理入口的前提下修复。

## 参考资料

- OpenSSH sshd_config(5): https://man.openbsd.org/sshd_config
- OpenSSH sshd(8): https://man.openbsd.org/sshd
- systemd Debugging: https://wiki.freedesktop.org/www/Software/systemd/Debugging/
- systemctl manual: https://man7.org/linux/man-pages/man1/systemctl.1.html

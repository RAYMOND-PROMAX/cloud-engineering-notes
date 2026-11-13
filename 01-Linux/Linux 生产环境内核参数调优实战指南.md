# Linux 生产环境内核参数调优实战指南

## 一、问题背景与适用场景

Linux 内核通过 `/proc/sys/` 暴露了大量可调参数，涵盖内存管理、网络栈、文件系统、安全策略等核心子系统。绝大多数发行版的默认值是面向"通用桌面或轻量服务器"设计的，放到生产环境——尤其是高并发 Web 服务、数据库、消息队列、大规模容器调度等场景——往往会成为瓶颈。

典型症状包括但不限于：

- Nginx/HAProxy 在流量高峰出现大量 Connection Reset，dmesg 中可见 possible SYN flooding on port 443。
- MySQL/PostgreSQL 在高负载下出现随机数百毫秒延迟尖刺，vmstat 显示 si/so 持续不为 0。
- Redis 执行 BGSAVE 时内存暴涨触发 OOM Killer。
- Elasticsearch 启动失败，日志报 max virtual memory areas vm.max_map_count [65530] is too low。
- 容器宿主机 inotify 资源耗尽，导致 kubelet 文件监听异常。

本文基于以下环境编写，覆盖从参数原理到生产落地的完整闭环：

| 项目 | 版本/说明 |
| --- | --- |
| 发行版 | RHEL 9 / Rocky Linux 9（内核 5.14）、Ubuntu 22.04（内核 5.15）、Ubuntu 24.04（内核 6.8） |
| LTS 内核 | 6.1.191、6.6.161、6.12.115、6.18.60 |
| 适用角色 | 物理机、虚拟机、容器宿主机 |
| 排除场景 | 嵌入式 / 实时内核（PREEMPT_RT）调优不在本文范围 |

> **环境适配提醒**：不同发行版的内核编译选项可能导致某些参数不存在。例如 `net.ipv4.tcp_tw_recycle` 在 4.12+ 内核已被移除；`net.core.default_qdisc` 依赖内核编译时启用了 `CONFIG_NET_SCH_FQ`。修改前务必用 `sysctl -a | grep <参数名>` 确认参数可用。

## 二、核心知识：sysctl 参数管理机制

### 2.1 参数加载顺序

内核参数的持久化配置文件分布在多个位置，按以下顺序加载（后加载的覆盖先加载的）：

```
/usr/lib/sysctl.d/*.conf        ← 发行版/软件包默认值（不要修改）
/run/sysctl.d/*.conf            ← 运行时临时覆盖
/etc/sysctl.d/*.conf            ← 管理员自定义（推荐位置）
/etc/sysctl.conf                ← 传统位置，仍然生效但不推荐
```

同一目录内按文件名的字典序加载，因此命名为 `99-production.conf` 可以确保覆盖 `50-default.conf` 等系统默认配置。

### 2.2 配置管理最佳实践

**原则：不直接编辑 /etc/sysctl.conf**，而是在 `/etc/sysctl.d/` 下按用途拆分独立文件：

```bash
# 按场景拆分，便于管理和回滚
/etc/sysctl.d/
├── 90-memory.conf          # 内存管理参数
├── 91-network.conf         # 网络栈参数
├── 92-filesystem.conf      # 文件系统参数
├── 93-security.conf        # 安全加固参数
└── 99-app-specific.conf    # 特定应用参数（如 ES、Redis）
```

**好处**：

- 回滚时只需删除或移动对应文件，不影响其他配置。
- 配置管理工具（Ansible/Puppet）可以按角色下发不同文件。
- 变更审计清晰，git diff 一目了然。

### 2.3 生效方式

```bash
# 方式一：加载所有 sysctl.d 配置（推荐，等价于重启后的行为）
sudo sysctl --system

# 方式二：仅加载单个文件
sudo sysctl -p /etc/sysctl.d/91-network.conf

# 方式三：临时修改单个参数（重启后失效，适合测试）
sudo sysctl -w net.core.somaxconn=65535

# 验证参数当前值
sysctl net.core.somaxconn
```

`sysctl --system` 的输出会显示每个被加载的文件及其设置的参数，方便确认加载顺序是否正确。

## 三、整体思路与实施框架

内核参数调优遵循以下闭环流程：

```
目标定义 → 环境摸底 → 前置检查 → 基线采集 → 分类实施 → 逐项验证 → 压测对比 → 持久化 → 监控告警 → 维护更新
```

### 3.1 前置检查清单

在修改任何参数之前，务必完成以下检查：

```bash
# 1. 确认内核版本
uname -r
# 预期输出示例：5.14.0-362.24.1.el9_3.x86_64

# 2. 确认发行版
cat /etc/os-release | head -3

# 3. 导出当前全量参数作为基线（用于回滚对比）
sysctl -a > /root/sysctl-baseline-$(date +%Y%m%d%H%M).txt 2>/dev/null

# 4. 检查当前内存状态
free -h
cat /proc/meminfo | grep -E 'MemTotal|MemFree|SwapTotal|SwapFree|HugePages'

# 5. 检查当前网络栈状态
ss -s
cat /proc/net/sockstat

# 6. 检查文件描述符使用情况
cat /proc/sys/fs/file-nr
# 输出格式：已分配  未使用  最大值

# 7. 确认目标参数是否存在
sysctl -a 2>/dev/null | grep -c "net.ipv4.tcp_tw_recycle"
# 输出 0 表示该参数不存在（4.12+ 已移除）
```

### 3.2 基线采集脚本

```bash
#!/bin/bash
# 文件：collect-baseline.sh
# 用途：采集内核参数基线，变更前后各执行一次用于对比
# 依赖：无
# 权限：需要 root 或 sudo

set -euo pipefail

TIMESTAMP=$(date +%Y%m%d_%H%M%S)
OUTDIR="/root/kernel-tuning-baseline"
mkdir -p "${OUTDIR}"

echo "[INFO] 采集时间: ${TIMESTAMP}"

# 全量参数快照
sysctl -a > "${OUTDIR}/sysctl-all-${TIMESTAMP}.txt" 2>/dev/null

# 内存信息
free -m > "${OUTDIR}/free-${TIMESTAMP}.txt"
cat /proc/meminfo > "${OUTDIR}/meminfo-${TIMESTAMP}.txt"

# 网络连接统计
ss -s > "${OUTDIR}/ss-${TIMESTAMP}.txt"
cat /proc/net/sockstat > "${OUTDIR}/sockstat-${TIMESTAMP}.txt"

# CPU 与负载
uptime > "${OUTDIR}/uptime-${TIMESTAMP}.txt"
vmstat 1 5 > "${OUTDIR}/vmstat-${TIMESTAMP}.txt"

# 文件描述符
cat /proc/sys/fs/file-nr > "${OUTDIR}/file-nr-${TIMESTAMP}.txt"
ulimit -a > "${OUTDIR}/ulimit-${TIMESTAMP}.txt"

echo "[INFO] 基线已保存至 ${OUTDIR}/"
ls -la "${OUTDIR}/"
```

## 四、内存管理调优

### 4.1 vm.swappiness —— 控制 Swap 使用倾向

**参数含义**：`vm.swappiness` 控制内核在内存压力下将匿名页（进程私有数据）换出到 swap 分区的积极程度。值域 0-200（老内核 0-100），默认值 60。

- 值越高，越积极使用 swap（倾向释放 RAM 给 page cache）。
- 值越低，越倾向保留匿名页在 RAM 中（牺牲 page cache）。

**为什么默认值在生产环境不合适**：

默认值 60 意味着内核会在还有大量空闲 RAM 时就开始将应用数据换出到磁盘。对于数据库、缓存、Java 应用等，这会导致不可预测的延迟尖刺——当被换出的内存页被再次访问时，需要从磁盘读回（即使是 NVMe SSD，延迟也是 RAM 的 100 倍以上）。

**生产建议**：

| 场景 | 推荐值 | 理由 |
| --- | --- | --- |
| 通用 Web/应用服务器 | 10 | 强烈倾向使用 RAM，仅在内存紧张时少量使用 swap |
| 数据库（MySQL/PostgreSQL） | 1 | 几乎不使用 swap，但保留最后的安全垫 |
| Redis / Memcached | 1 | 纯内存应用，swap 对性能影响灾难性 |
| 容器宿主机（K8s node） | 10 | 平衡多个 Pod 的内存需求 |

**⚠️ 不要设为 0**：在 cgroup v2 和现代内核（4.0+）中，`vm.swappiness=0` 不是"禁用 swap"，而是告诉内核"只在内存极度紧张时才使用 swap"。但这会导致 OOM Killer 在 swap 还有空间时就提前触发，杀掉你的关键进程。如果确实要完全禁用 swap，应该用 `swapoff -a` 并从 `/etc/fstab` 中移除 swap 条目。

```ini
# /etc/sysctl.d/90-memory.conf
# 内存管理 - 通用生产环境基线
vm.swappiness = 10
```

**验证方法**：

```bash
# 确认参数已生效
sysctl vm.swappiness
# 预期输出：vm.swappiness = 10

# 持续观察 swap 使用变化
vmstat 5 10
# 关注 si（swap in）和 so（swap out）列，正常应为 0 或接近 0

# 如果发现 swap 持续增长
free -m
# 对比 used 和 buff/cache，判断是真的内存不足还是参数未生效
```

### 4.2 vm.dirty_ratio / vm.dirty_background_ratio —— 脏页刷写控制

**参数含义**：

- vm.dirty_background_ratio：当脏页占系统可用内存的百分比达到此值时，内核开始后台异步刷写到磁盘。默认值 10。
- vm.dirty_ratio：当脏页占比达到此值时，阻塞写入进程，强制同步刷写。默认值 20。

**生产风险**：默认值在大内存服务器（如 256GB/512GB RAM）上意味着允许积累数十 GB 的脏页，一旦触发同步刷写，所有写 I/O 会被阻塞，导致应用层面出现秒级卡顿。

**生产建议**：

```ini
# /etc/sysctl.d/90-memory.conf（续）
# 减小脏页积累上限，降低 I/O 突发写入的峰值延迟
vm.dirty_background_ratio = 5
vm.dirty_ratio = 15
```

对于数据库服务器，可以进一步使用绝对值参数替代百分比（大内存机器更精确）：

```ini
# 备选：使用字节绝对值（优先级高于 ratio 参数）
# 后台刷写阈值 256MB
vm.dirty_background_bytes = 268435456
# 阻塞刷写阈值 1GB
vm.dirty_bytes = 1073741824
```

**验证**：

```bash
# 查看当前脏页量
grep -E 'Dirty|Writeback' /proc/meminfo
# Dirty:              1234 kB    ← 当前脏页量
# Writeback:             0 kB    ← 正在写回的量

# 使用 dstat 实时观察 I/O 模式变化
dstat -d -D sda 5
```

### 4.3 vm.max_map_count —— 进程内存映射区域上限

**参数含义**：单个进程允许的最大内存映射区域（mmap）数量，默认值 65530。

**需要调大的场景**：

- Elasticsearch / OpenSearch：启动时会创建大量 mmap 映射来管理 Lucene 段文件，默认值会导致启动失败。
- 大型 Java 应用：JVM 的内存管理和 JIT 编译会消耗大量 mmap 区域。
- 数据库（PostgreSQL 等）：大量共享内存段。

```ini
# /etc/sysctl.d/99-app-specific.conf
# Elasticsearch 官方要求至少 262144
vm.max_map_count = 262144
```

**验证**：

```bash
sysctl vm.max_map_count
# 预期输出：vm.max_map_count = 262144

# 查看某个进程当前使用的 mmap 区域数
cat /proc/<PID>/maps | wc -l
```

### 4.4 vm.overcommit_memory —— 内存超售策略

**参数含义**：

| 值 | 行为 |
| --- | --- |
| 0（默认） | 启发式判断，允许适度超售 |
| 1 | 始终允许超售（malloc 永远成功） |
| 2 | 严格模式，总分配不超过 swap + RAM × overcommit_ratio |

**Redis 场景**：Redis 在 `BGSAVE` 时会 `fork()` 子进程，利用 CoW（Copy-on-Write）机制进行持久化。`fork()` 时操作系统需要"承诺"与父进程相同大小的虚拟地址空间。如果 `overcommit_memory=0` 且 Redis 占用了大量内存，`fork()` 可能因为内核认为内存不足而失败，导致持久化中断。

```ini
# /etc/sysctl.d/99-app-specific.conf（续）
# Redis 服务器专用 —— 允许内存超售以支持 fork
vm.overcommit_memory = 1
```

**⚠️ 风险提示**：`overcommit_memory=1` 意味着 `malloc()` 永远不会返回 NULL，如果实际物理内存不足，OOM Killer 会介入。仅在 Redis 等明确需要的场景使用，通用服务器保持默认值 0。

### 4.5 vm.zone_reclaim_mode —— NUMA 架构内存回收

**参数含义**：控制 NUMA 架构下，当本地 node 内存不足时，是优先从本地回收（值 1）还是从远端 node 分配（值 0）。

**为什么要设为 0**：数据库和缓存类应用在 NUMA 架构的多路服务器上，如果设为 1，内核会在本地 node 满时疯狂回收 page cache（甚至频繁换出），而不是从另一个 NUMA node 分配。远端内存访问的延迟（~100ns）远低于磁盘 I/O（~10μs SSD / ~10ms HDD）。

```ini
# /etc/sysctl.d/90-memory.conf（续）
# NUMA 服务器：禁止本地节点激进回收
vm.zone_reclaim_mode = 0
```

**确认 NUMA 拓扑**：

```bash
numactl --hardware
# 如果只有 1 个 node，该参数无实际影响
# 如果有多个 node，务必设为 0
```

## 五、透明大页（THP）与显式大页

### 5.1 为什么要关注大页

标准内存页大小为 4KB。当应用使用大量内存时（如数据库 buffer pool 64GB），页表项数量达到 1600 万+，TLB（Translation Lookaside Buffer）缓存命中率下降，每次 TLB miss 都需要多级页表遍历，带来额外的 CPU 开销。

大页（Hugepages，通常 2MB 或 1GB）将页表项数量减少 512 倍（2MB 页）或 262144 倍（1GB 页），大幅降低 TLB miss。

### 5.2 透明大页（THP）—— 数据库的"隐形杀手"

THP 是内核自动管理大页的机制，无需应用适配。但它有两个严重问题：

**问题一：khugepaged 后台压缩的延迟尖刺**

内核后台线程 `khugepaged` 持续扫描并尝试将连续的 4KB 页合并为 2MB 大页。这个过程需要锁定内存区域，在数据库等频繁分配/释放内存的场景下，会造成随机的数百毫秒延迟。

**问题二：Redis fork() 的 CoW 内存放大**

Redis `BGSAVE` 时 `fork()` 子进程，利用 CoW 共享父进程内存页。在 THP 启用时，父进程修改任意一个字节，内核需要复制整个 2MB 大页（而不是 4KB 标准页），内存放大 512 倍。如果 Redis 占用 10GB 内存且写入频繁，`BGSAVE` 期间可能额外消耗数 GB 到数十 GB 内存，触发 OOM。

**生产建议**：

| 数据库/应用 | THP 设置 | 理由 |
| --- | --- | --- |
| MySQL/MariaDB | never | 避免 InnoDB buffer pool 的延迟尖刺 |
| PostgreSQL | never | 使用显式大页替代 |
| Redis | never（必须） | 避免 fork CoW 内存放大 |
| Oracle（UEK 7+） | madvise | Oracle 代码显式请求时才使用 |
| 通用应用服务器 | madvise 或 never | 按需评估 |

### 5.3 禁用 THP 的操作步骤

**步骤一：检查当前状态**

```bash
cat /sys/kernel/mm/transparent_hugepage/enabled
# 输出示例：[always] madvise never
# 方括号内是当前值，always 表示全局启用

cat /sys/kernel/mm/transparent_hugepage/defrag
# 输出示例：[always] defer defer+madvise madvise never
```

**步骤二：运行时立即禁用**

```bash
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag
```

**步骤三：持久化（方案 A —— 内核引导参数）**

编辑 GRUB 配置：

```bash
# RHEL/Rocky
sudo vi /etc/default/grub
# 在 GRUB_CMDLINE_LINUX 行末尾追加：
# transparent_hugepage=never

# 重新生成 GRUB 配置
sudo grub2-mkconfig -o /boot/grub2/grub.cfg      # BIOS 引导
sudo grub2-mkconfig -o /boot/efi/EFI/redhat/grub.cfg  # UEFI 引导

# Ubuntu/Debian
sudo vi /etc/default/grub
# 同样追加 transparent_hugepage=never
sudo update-grub
```

**步骤三：持久化（方案 B —— systemd 服务）**

如果不方便修改 GRUB（如云服务器），可以创建 systemd 服务：

```ini
# /etc/systemd/system/disable-thp.service
[Unit]
Description=Disable Transparent Huge Pages
DefaultDependencies=no
After=sysinit.target local-fs.target
Before=basic.target

[Service]
Type=oneshot
ExecStart=/bin/sh -c 'echo never > /sys/kernel/mm/transparent_hugepage/enabled && echo never > /sys/kernel/mm/transparent_hugepage/defrag'

[Install]
WantedBy=basic.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable disable-thp.service
sudo systemctl start disable-thp.service
```

**步骤四：验证**

```bash
cat /sys/kernel/mm/transparent_hugepage/enabled
# 预期输出：always madvise [never]

# 重启后再次确认
grep -i hugepage /proc/meminfo
# AnonHugePages:     0 kB   ← THP 已禁用后应逐步降为 0
```

### 5.4 配置显式大页（Explicit Hugepages）

显式大页是手动预分配的、固定的大页内存池，不受 THP 的后台压缩影响，是数据库的首选方案。

**步骤一：确认大页大小**

```bash
grep Hugepagesize /proc/meminfo
# Hugepagesize:       2048 kB
```

**步骤二：计算所需大页数**

以 PostgreSQL `shared_buffers = 32GB` 为例：

```
所需大页数 = 32GB × 1024 MB/GB ÷ 2 MB/页 = 16384 页
# 建议多预留 5-10%
实际配置 = 16384 × 1.05 ≈ 17200 页
```

**步骤三：配置内核参数**

```ini
# /etc/sysctl.d/90-memory.conf（续）
# 显式大页：PostgreSQL shared_buffers=32GB 预留
vm.nr_hugepages = 17200
```

```bash
sudo sysctl -p /etc/sysctl.d/90-memory.conf
```

**步骤四：配置权限**

大页内存默认只有 root 可用。数据库通常以普通用户运行，需要配置 `hugetlb_shm_group`：

```bash
# 查找数据库用户的 GID
id postgres
# uid=26(postgres) gid=26(postgres) ...

# 设置允许使用大页的组
echo 'vm.hugetlb_shm_group = 26' >> /etc/sysctl.d/90-memory.conf
sudo sysctl -p /etc/sysctl.d/90-memory.conf
```

同时调整 limits：

```ini
# /etc/security/limits.d/99-hugepages.conf
postgres  soft  memlock  unlimited
postgres  hard  memlock  unlimited
```

如果 PostgreSQL 通过 systemd 管理，还需要在 service 文件中设置：

```ini
# /etc/systemd/system/postgresql.service.d/hugepages.conf
[Service]
LimitMEMLOCK=infinity
```

```bash
sudo systemctl daemon-reload
sudo systemctl restart postgresql
```

**步骤五：配置数据库使用大页**

PostgreSQL：

```ini
# postgresql.conf
huge_pages = on
# 设为 on（而非 try）可以确保大页不足时 PostgreSQL 启动失败，
# 避免在没有大页的情况下静默降级导致性能问题
```

**步骤六：验证大页使用情况**

```bash
grep HugePages /proc/meminfo
# HugePages_Total:   17200     ← 总预留数
# HugePages_Free:     1016     ← 空闲数（Total - Free = 已使用）
# HugePages_Rsvd:        0     ← 已预留但未实际分配
# HugePages_Surp:        0     ← 超出预分配的临时大页

# 如果 Free 等于 Total，说明没有应用在使用大页，检查应用配置
```

## 六、网络栈调优

### 6.1 连接队列参数

#### net.core.somaxconn —— 全连接队列上限

**参数含义**：`listen()` 系统调用中 `backlog` 参数的上限。当应用层来不及 `accept()` 新连接时，已完成三次握手的连接在此队列中排队。默认值在老内核是 128，新内核（5.4+）提升到了 4096。

**高并发场景的问题**：当队列满时，新的已完成握手的连接会被丢弃，客户端表现为 `Connection Reset` 或超时。

```ini
# /etc/sysctl.d/91-network.conf
# 全连接队列上限
net.core.somaxconn = 65535
```

**重要**：`somaxconn` 是内核层面的上限，应用层面也需要配合调整：

- Nginx：listen 80 backlog=65535;
- Redis：tcp-backlog 65535（redis.conf）
- PostgreSQL：无需特别配置，默认使用 somaxconn

**验证队列溢出**：

```bash
# 查看全连接队列溢出计数
netstat -s | grep overflow
# 或
nstat -az TcpExtListenOverflows
# 如果数值持续增长，说明 somaxconn 或应用的 backlog 不够

# 查看某个监听端口的队列状态
ss -ltn | grep :80
# Recv-Q：当前队列中的连接数
# Send-Q：队列最大长度（= min(backlog, somaxconn)）
```

#### net.ipv4.tcp_max_syn_backlog —— 半连接队列上限

**参数含义**：存放已收到 SYN 但尚未完成三次握手的连接队列大小。默认值通常 128 或 1024。

```ini
# /etc/sysctl.d/91-network.conf（续）
# 半连接队列
net.ipv4.tcp_max_syn_backlog = 16384
```

#### net.core.netdev_max_backlog —— 网卡接收队列

**参数含义**：每个 CPU 核心的网络设备接收队列长度。当网卡接收数据包的速度超过内核处理速度时，数据包在此排队。默认值 1000。

```ini
# /etc/sysctl.d/91-network.conf（续）
# 网卡接收队列
net.core.netdev_max_backlog = 16384
```

### 6.2 TCP 连接回收与复用

#### TIME_WAIT 处理

TCP 连接关闭后进入 `TIME_WAIT` 状态，默认等待 2×MSL（60 秒）。高并发短连接场景（如反向代理）下，大量 `TIME_WAIT` 会耗尽端口资源。

```ini
# /etc/sysctl.d/91-network.conf（续）

# 允许 TIME_WAIT 状态的 socket 被新连接复用（仅对客户端/主动关闭方有效）
net.ipv4.tcp_tw_reuse = 1

# 缩短 FIN-WAIT-2 超时时间（默认 60 秒）
net.ipv4.tcp_fin_timeout = 15
```

**⚠️ 关于 tcp_tw_recycle**：该参数在 4.12+ 内核已被移除。它在 NAT 环境下会导致大量连接失败，绝对不要在老内核上启用。

#### 临时端口范围

```ini
# /etc/sysctl.d/91-network.conf（续）

# 扩大临时端口范围（默认 32768-60999）
# 作为客户端/反向代理时，每个连接占用一个临时端口
net.ipv4.ip_local_port_range = 1024 65535
```

**验证 TIME_WAIT 数量**：

```bash
ss -s
# 输出中 timewait 行显示当前 TIME_WAIT 连接数

# 更详细的统计
ss -tan state time-wait | wc -l
```

### 6.3 SYN 洪泛防护

```ini
# /etc/sysctl.d/91-network.conf（续）

# 启用 SYN Cookie 防护（遭受 SYN 洪泛时不为半连接分配资源）
net.ipv4.tcp_syncookies = 1
```

**判断是否遭受 SYN 洪泛**：

```bash
# 查看 SYN Cookie 触发次数
netstat -s | grep "SYNs to LISTEN"
# 或
nstat -az TcpExtSyncookiesSent TcpExtSyncookiesRecv TcpExtSyncookiesFailed

# dmesg 中的告警
dmesg | grep -i "syn flood"
# 输出示例：TCP: request_sock_TCP: Possible SYN flooding on port 443
```

### 6.4 TCP 拥塞控制 —— BBR

BBR（Bottleneck Bandwidth and Round-trip propagation time）是 Google 开发的拥塞控制算法，相比传统的 CUBIC，在高延迟和丢包网络环境下性能显著提升。

**前置条件**：内核版本 ≥ 4.9，且编译时启用了 `CONFIG_TCP_CONG_BBR` 和 `CONFIG_NET_SCH_FQ`。

```bash
# 检查 BBR 模块是否可用
modprobe tcp_bbr
lsmod | grep bbr
# 如果没有输出，说明内核未编译 BBR 支持

# 查看当前可用的拥塞控制算法
sysctl net.ipv4.tcp_available_congestion_control
# 预期包含 bbr
```

```ini
# /etc/sysctl.d/91-network.conf（续）

# 启用 BBR 拥塞控制
net.core.default_qdisc = fq
net.ipv4.tcp_congestion_control = bbr
```

**验证**：

```bash
sysctl net.ipv4.tcp_congestion_control
# 预期输出：net.ipv4.tcp_congestion_control = bbr

# 确认已生效的连接在使用 BBR
ss -tin | grep bbr | head -5
```

### 6.5 TCP 缓冲区调优

```ini
# /etc/sysctl.d/91-network.conf（续）

# TCP 接收缓冲区 [最小值 默认值 最大值]（字节）
net.ipv4.tcp_rmem = 4096 87380 67108864

# TCP 发送缓冲区 [最小值 默认值 最大值]（字节）
net.ipv4.tcp_wmem = 4096 65536 67108864

# 全局接收/发送缓冲区上限
net.core.rmem_max = 67108864
net.core.wmem_max = 67108864
```

**说明**：上述最大值 64MB 适用于高带宽长距离传输（如跨机房数据同步）。内网低延迟场景通常不需要这么大。TCP 自动调优（`net.ipv4.tcp_moderate_rcvbuf = 1`，默认开启）会根据实际需求动态调整，max 只是设置上限。

## 七、文件系统与描述符

### 7.1 全局文件描述符上限

```ini
# /etc/sysctl.d/92-filesystem.conf

# 系统全局最大文件句柄数（默认通常 ~100万，按需调高）
fs.file-max = 2097152
```

**查看使用情况**：

```bash
cat /proc/sys/fs/file-nr
# 输出格式：已分配  0  最大值
# 示例：5120     0     2097152
# 如果已分配接近最大值，需要调大
```

### 7.2 inotify 限制

容器化环境中，kubelet、容器运行时、日志收集器（Filebeat/Fluentd）、IDE 远程开发等都会注册 inotify watch。默认值 8192 在节点运行大量 Pod 时很容易耗尽。

```ini
# /etc/sysctl.d/92-filesystem.conf（续）

# 单用户最大 inotify watch 数
fs.inotify.max_user_watches = 524288

# 单用户最大 inotify 实例数
fs.inotify.max_user_instances = 8192
```

**查看使用情况**：

```bash
# 方法一：统计所有进程的 inotify watch 总数
find /proc/*/fdinfo -name '*.fdinfo' -exec grep -c inotify {} + 2>/dev/null | awk -F: '{sum+=$2} END {print sum}'

# 方法二（更准确，需要 inotifywait 或内核 tracing）：
# 查看单个进程的 watch 数量
ls -la /proc/<PID>/fd | grep inotify | wc -l
```

### 7.3 用户级 limits 与 systemd LimitNOFILE

`sysctl` 的 `fs.file-max` 是系统全局限制。单个进程的文件描述符限制由 `ulimit`（PAM limits）或 systemd 的 `LimitNOFILE` 控制。两者需要配合：

```ini
# /etc/security/limits.d/99-nofile.conf
# 对通过 PAM 登录的用户生效
*    soft    nofile    1048576
*    hard    nofile    1048576
```

对于 systemd 管理的服务：

```ini
# /etc/systemd/system/nginx.service.d/limits.conf
[Service]
LimitNOFILE=1048576
```

```bash
# 确认 systemd 全局默认上限（如需调整）
# /etc/systemd/system.conf
# DefaultLimitNOFILE=1048576

sudo systemctl daemon-reload
sudo systemctl restart nginx

# 验证进程实际的 limits
cat /proc/$(pidof nginx | awk '{print $1}')/limits | grep "Max open files"
# Max open files    1048576    1048576    files
```

## 八、安全加固参数

### 8.1 网络安全加固

```ini
# /etc/sysctl.d/93-security.conf

# ---- IP 转发控制 ----
# 非路由器/网关/容器宿主机：禁用 IP 转发
# 注意：Docker/K8s 节点必须保持开启
net.ipv4.ip_forward = 0
# IPv6 同理
net.ipv6.conf.all.forwarding = 0

# ---- 源路由防护 ----
# 禁止接受源路由选项的数据包（防止路由欺骗）
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.default.accept_source_route = 0

# ---- ICMP 安全 ----
# 忽略 ICMP 广播请求（防 Smurf 攻击）
net.ipv4.icmp_echo_ignore_broadcasts = 1
# 忽略异常 ICMP 错误消息
net.ipv4.icmp_ignore_bogus_error_responses = 1

# ---- 反向路径过滤 ----
# 启用严格反向路径验证（防 IP 欺骗）
# 注意：多宿主机/非对称路由环境可能需要设为 2（宽松模式）
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1

# ---- ICMP 重定向 ----
# 不接受 ICMP 重定向（防止中间人攻击）
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv4.conf.all.secure_redirects = 0
net.ipv4.conf.default.secure_redirects = 0
# 不发送 ICMP 重定向
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0

# ---- IPv6 安全（如果使用 IPv6）----
net.ipv6.conf.all.accept_redirects = 0
net.ipv6.conf.default.accept_redirects = 0
net.ipv6.conf.all.accept_source_route = 0
net.ipv6.conf.default.accept_source_route = 0
```

**⚠️ Docker/Kubernetes 节点特别注意**：

```ini
# K8s 节点和 Docker 宿主机必须开启 IP 转发！
# 否则 Pod 间通信和 Service 网络完全不通
net.ipv4.ip_forward = 1
# 如果使用 Calico/Cilium 等 CNI 的 IPv6 功能
net.ipv6.conf.all.forwarding = 1

# bridge-nf-call 参数（K8s 网络必需）
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
```

### 8.2 kernel.sysrq —— 魔术键控制

```ini
# /etc/sysctl.d/93-security.conf（续）

# 限制 SysRq 键功能（生产环境不应完全开启）
# 0 = 完全禁用
# 1 = 完全启用（不推荐）
# 推荐值：仅允许同步磁盘(16) + 只读重挂载(32) + 发送 SIGTERM(64) + 发送 SIGKILL(128)
kernel.sysrq = 240
```

### 8.3 ARP 相关参数

在大型二层网络或虚拟化环境中，ARP 表可能溢出导致网络间歇性不通：

```ini
# /etc/sysctl.d/93-security.conf（续）

# ARP 缓存表大小
net.ipv4.neigh.default.gc_thresh1 = 4096    # 低水位（开始 GC）
net.ipv4.neigh.default.gc_thresh2 = 8192    # 中水位（5秒内必须 GC）
net.ipv4.neigh.default.gc_thresh3 = 16384   # 硬上限（超过直接丢弃）
```

## 九、完整生产基线配置模板

以下是综合各节内容的完整配置文件，按场景分类：

### 9.1 通用 Web/应用服务器

```ini
# /etc/sysctl.d/99-production-web.conf
# 适用场景：Nginx/HAProxy 反向代理、Tomcat/Spring Boot 应用服务器
# 适用系统：RHEL 9 / Ubuntu 22.04+ / Rocky 9
# 创建时间：2026-11
# 修改前请先采集基线：sysctl -a > /root/sysctl-baseline.txt

# ========== 内存管理 ==========
vm.swappiness = 10
vm.dirty_background_ratio = 5
vm.dirty_ratio = 15
vm.max_map_count = 262144

# ========== 网络性能 ==========
net.core.somaxconn = 65535
net.core.netdev_max_backlog = 16384
net.ipv4.tcp_max_syn_backlog = 16384
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_fin_timeout = 15
net.ipv4.ip_local_port_range = 1024 65535
net.core.default_qdisc = fq
net.ipv4.tcp_congestion_control = bbr
net.ipv4.tcp_rmem = 4096 87380 67108864
net.ipv4.tcp_wmem = 4096 65536 67108864
net.core.rmem_max = 67108864
net.core.wmem_max = 67108864

# ========== 文件系统 ==========
fs.file-max = 2097152
fs.inotify.max_user_watches = 524288
fs.inotify.max_user_instances = 8192

# ========== 安全加固 ==========
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.default.accept_source_route = 0
net.ipv4.icmp_echo_ignore_broadcasts = 1
net.ipv4.icmp_ignore_bogus_error_responses = 1
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0
kernel.sysrq = 240
```

### 9.2 数据库服务器（PostgreSQL/MySQL）

```ini
# /etc/sysctl.d/99-production-db.conf
# 适用场景：PostgreSQL / MySQL / MariaDB 专用服务器
# 注意事项：
#   1. THP 必须通过 GRUB 或 systemd 服务禁用（见第五节）
#   2. vm.nr_hugepages 根据实际 buffer pool 大小计算
#   3. vm.hugetlb_shm_group 设为数据库运行用户的 GID

# ========== 内存管理 ==========
vm.swappiness = 1
vm.dirty_background_ratio = 3
vm.dirty_ratio = 10
vm.max_map_count = 262144
vm.overcommit_memory = 0
vm.zone_reclaim_mode = 0

# 显式大页（根据 buffer pool 大小计算）
# PostgreSQL shared_buffers=32GB → 32*1024/2*1.05 ≈ 17200
# MySQL innodb_buffer_pool_size=64GB → 64*1024/2*1.05 ≈ 34560
vm.nr_hugepages = 17200
vm.hugetlb_shm_group = 26

# ========== 网络 ==========
net.core.somaxconn = 4096
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_fin_timeout = 15
net.ipv4.ip_local_port_range = 1024 65535

# ========== 文件系统 ==========
fs.file-max = 2097152
fs.aio-max-nr = 1048576

# ========== 安全 ==========
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.default.accept_source_route = 0
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1
kernel.sysrq = 240
```

### 9.3 Redis 服务器

```ini
# /etc/sysctl.d/99-production-redis.conf
# 适用场景：Redis 服务器
# 注意事项：
#   1. THP 必须禁用！（见第五节）
#   2. overcommit_memory=1 允许 fork 成功

vm.swappiness = 1
vm.overcommit_memory = 1
vm.max_map_count = 262144
net.core.somaxconn = 65535
net.ipv4.tcp_max_syn_backlog = 16384
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_tw_reuse = 1
fs.file-max = 2097152
kernel.sysrq = 240
```

### 9.4 Kubernetes 工作节点

```ini
# /etc/sysctl.d/99-production-k8s.conf
# 适用场景：Kubernetes Worker Node
# 注意事项：ip_forward 必须开启

# ========== 必须开启 ==========
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1

# ========== 内存 ==========
vm.swappiness = 10
vm.max_map_count = 262144

# ========== 网络 ==========
net.core.somaxconn = 65535
net.core.netdev_max_backlog = 16384
net.ipv4.tcp_max_syn_backlog = 16384
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_fin_timeout = 15
net.ipv4.ip_local_port_range = 1024 65535
net.core.default_qdisc = fq
net.ipv4.tcp_congestion_control = bbr
net.ipv4.tcp_rmem = 4096 87380 67108864
net.ipv4.tcp_wmem = 4096 65536 67108864
net.core.rmem_max = 67108864
net.core.wmem_max = 67108864

# ========== 文件系统（容器场景消耗巨大）==========
fs.file-max = 2097152
fs.inotify.max_user_watches = 524288
fs.inotify.max_user_instances = 8192

# ========== 安全 ==========
net.ipv4.conf.all.rp_filter = 1
kernel.sysrq = 240
```

## 十、自动化部署脚本

### 10.1 Ansible Playbook 示例

```yaml
# roles/kernel-tuning/tasks/main.yml
---
- name: Deploy sysctl configuration
  ansible.builtin.template:
    src: "{{ item }}.conf.j2"
    dest: "/etc/sysctl.d/{{ item }}.conf"
    owner: root
    group: root
    mode: '0644'
    backup: yes
  loop:
    - 90-memory
    - 91-network
    - 92-filesystem
    - 93-security
  notify: reload sysctl

- name: Disable Transparent Hugepages via GRUB
  ansible.builtin.lineinfile:
    path: /etc/default/grub
    regexp: '^GRUB_CMDLINE_LINUX="(.*)"$'
    line: 'GRUB_CMDLINE_LINUX="\1 transparent_hugepage=never"'
    backrefs: yes
    backup: yes
  when: disable_thp | default(false) | bool
  notify: regenerate grub

- name: Deploy disable-thp systemd service
  ansible.builtin.template:
    src: disable-thp.service.j2
    dest: /etc/systemd/system/disable-thp.service
    owner: root
    group: root
    mode: '0644'
  when: disable_thp | default(false) | bool
  notify:
    - systemd daemon-reload
    - enable disable-thp

# roles/kernel-tuning/handlers/main.yml
- name: reload sysctl
  ansible.builtin.command: sysctl --system

- name: regenerate grub
  ansible.builtin.command: >
    grub2-mkconfig -o /boot/grub2/grub.cfg
  when: ansible_os_family == 'RedHat'

- name: systemd daemon-reload
  ansible.builtin.systemd:
    daemon_reload: yes

- name: enable disable-thp
  ansible.builtin.systemd:
    name: disable-thp
    enabled: yes
    state: started
```

### 10.2 变更验证脚本

```bash
#!/bin/bash
# 文件：verify-kernel-tuning.sh
# 用途：变更后验证关键参数是否生效
# 依赖：bash、sysctl
# 退出码：0 全部通过，非 0 有参数未达预期

set -euo pipefail

PASS=0
FAIL=0
WARN=0

check_param() {
    local param="$1"
    local expected="$2"
    local actual
    actual=$(sysctl -n "${param}" 2>/dev/null || echo "NOT_FOUND")

    if [[ "${actual}" == "${expected}" ]]; then
        echo "[PASS] ${param} = ${actual}"
        ((PASS++))
    elif [[ "${actual}" == "NOT_FOUND" ]]; then
        echo "[WARN] ${param} 不存在（内核版本可能不支持）"
        ((WARN++))
    else
        echo "[FAIL] ${param} = ${actual}（预期: ${expected}）"
        ((FAIL++))
    fi
}

echo "===== 内核参数验证 ====="
echo "时间: $(date)"
echo "内核: $(uname -r)"
echo ""

# 内存参数
check_param "vm.swappiness" "10"
check_param "vm.dirty_background_ratio" "5"
check_param "vm.dirty_ratio" "15"
check_param "vm.max_map_count" "262144"

# 网络参数
check_param "net.core.somaxconn" "65535"
check_param "net.core.netdev_max_backlog" "16384"
check_param "net.ipv4.tcp_max_syn_backlog" "16384"
check_param "net.ipv4.tcp_syncookies" "1"
check_param "net.ipv4.tcp_tw_reuse" "1"
check_param "net.ipv4.tcp_fin_timeout" "15"
check_param "net.ipv4.tcp_congestion_control" "bbr"

# 文件系统
check_param "fs.file-max" "2097152"
check_param "fs.inotify.max_user_watches" "524288"
check_param "fs.inotify.max_user_instances" "8192"

# 安全
check_param "net.ipv4.conf.all.rp_filter" "1"

# THP 检查
THP_STATUS=$(cat /sys/kernel/mm/transparent_hugepage/enabled 2>/dev/null | grep -oP '\[\K[^\]]+')
if [[ "${THP_STATUS}" == "never" ]]; then
    echo "[PASS] THP = [never]"
    ((PASS++))
elif [[ -z "${THP_STATUS}" ]]; then
    echo "[WARN] THP 状态无法读取"
    ((WARN++))
else
    echo "[FAIL] THP = [${THP_STATUS}]（预期: never）"
    ((FAIL++))
fi

echo ""
echo "===== 统计 ====="
echo "通过: ${PASS}  失败: ${FAIL}  警告: ${WARN}"
echo ""

if [[ ${FAIL} -gt 0 ]]; then
    echo "存在未达预期的参数，请检查配置文件和加载顺序。"
    exit 1
fi

exit 0
```

## 十一、监控与告警

### 11.1 关键监控指标

将以下指标纳入 Prometheus + Grafana 监控体系：

| 指标来源 | 指标/命令 | 告警条件 |
| --- | --- | --- |
| node_exporter | node_vmstat_pswpin / node_vmstat_pswpout | 持续 > 0 表示在使用 swap |
| node_exporter | node_sockstat_TCP_tw | TIME_WAIT > 10000 |
| node_exporter | node_nf_conntrack_entries / node_nf_conntrack_entries_limit | 使用率 > 80% |
| node_exporter | node_filefd_allocated / node_filefd_maximum | 使用率 > 70% |
| 自定义 | ss -s 中 TCP overflows | 持续增长 |
| /proc/meminfo | AnonHugePages | THP 应禁用的节点上 > 0 |

### 11.2 Prometheus 告警规则示例

```yaml
# /etc/prometheus/rules/kernel-tuning.yml
groups:
  - name: kernel-tuning
    rules:
      - alert: HighSwapUsage
        expr: node_memory_SwapFree_bytes / node_memory_SwapTotal_bytes < 0.5
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Swap 使用超过 50%"
          description: "节点 {{ $labels.instance }} swap 空闲不足 50%，可能 vm.swappiness 设置不当或内存不足。"

      - alert: TooManyTimeWait
        expr: node_sockstat_TCP_tw > 20000
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "TIME_WAIT 连接过多"
          description: "节点 {{ $labels.instance }} 当前 TIME_WAIT 连接数 {{ $value }}。"

      - alert: FileDescriptorExhaustion
        expr: node_filefd_allocated / node_filefd_maximum > 0.8
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "文件描述符即将耗尽"
          description: "节点 {{ $labels.instance }} 文件描述符使用率超过 80%。"

      - alert: ConntrackTableFull
        expr: node_nf_conntrack_entries / node_nf_conntrack_entries_limit > 0.8
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "conntrack 表即将满"
          description: "节点 {{ $labels.instance }} conntrack 使用率 {{ $value | humanizePercentage }}。"
```

## 十二、排查路径：参数不生效的常见原因

当你确认修改了配置文件但参数没有生效时，按以下路径逐步排查：

### 12.1 配置文件加载顺序覆盖

```bash
# 查看所有 sysctl 配置文件的加载顺序和内容
sysctl --system 2>&1 | grep "Applying"
# 输出示例：
# * Applying /usr/lib/sysctl.d/50-default.conf ...
# * Applying /etc/sysctl.d/99-production.conf ...
```

如果你的配置文件在 `50-default.conf` 之前加载，且 `50-default.conf` 中有同名参数，你的设置会被覆盖。**解决方案**：文件名使用更大的数字前缀。

### 12.2 参数不存在

```bash
sysctl -a 2>/dev/null | grep "net.ipv4.tcp_tw_recycle"
# 空输出 → 参数不存在

# 确认原因
# 方法一：检查内核版本
uname -r

# 方法二：检查内核编译选项
grep CONFIG_TCP_CONG_BBR /boot/config-$(uname -r)
# CONFIG_TCP_CONG_BBR=m  ← 编译为模块，需要 modprobe
# CONFIG_TCP_CONG_BBR=y  ← 内建
# 找不到 ← 内核未编译此功能
```

### 12.3 systemd 服务覆盖了 sysctl

某些 systemd 服务在启动时会通过 `ExecStartPre` 或内置逻辑修改内核参数。例如 Docker 会设置 `net.ipv4.ip_forward=1`。

```bash
# 检查是否有服务在启动时修改了参数
grep -r "ip_forward" /usr/lib/systemd/system/ /etc/systemd/system/ 2>/dev/null
```

### 12.4 容器 sysctl 命名空间

在容器内部，部分 sysctl 参数属于 network namespace，容器看到的是隔离的值，而非宿主机的值。例如 `net.core.somaxconn` 在容器内默认可能是 128。

```bash
# Kubernetes Pod 中设置 sysctl
# 需要在 Pod spec 中声明：
# securityContext:
#   sysctls:
#   - name: net.core.somaxconn
#     value: "65535"
```

## 十三、回滚方案

### 13.1 基于备份文件回滚

如果使用了 `sysctl -a > baseline.txt` 采集了基线，可以通过对比还原：

```bash
# 对比变更前后的差异
diff /root/sysctl-baseline-before.txt /root/sysctl-baseline-after.txt

# 手动还原特定参数
sysctl -w vm.swappiness=60

# 或直接删除自定义配置文件并重新加载
sudo rm /etc/sysctl.d/99-production.conf
sudo sysctl --system
```

### 13.2 回滚操作的风险评估

| 回滚操作 | 风险 | 注意事项 |
| --- | --- | --- |
| 恢复 vm.swappiness 默认值 | 低 | 立即生效，但已在 RAM 中的页不会主动换出 |
| 恢复 somaxconn 默认值 | 中 | 高并发时可能立即出现连接被拒 |
| 减小 vm.nr_hugepages | 高 | 如果数据库正在使用大页，直接减小会失败 |
| 重新启用 THP | 高 | 数据库可能出现延迟尖刺，Redis 可能 OOM |
| 恢复 ip_forward=0 | 极高 | K8s 集群所有 Pod 网络立即中断 |

### 13.3 灰度策略

**生产变更流程建议**：

1. 测试环境验证：完整测试所有参数变更，执行压力测试对比性能。
2. 灰度节点：选择 1-2 台同角色的生产节点先行变更。
3. 观察窗口：至少观察 24 小时，覆盖业务高峰期。
4. 监控对比：对比灰度节点与未变更节点的关键指标（延迟 P99、CPU 利用率、swap 使用）。
5. 全量推送：确认无异常后，通过配置管理工具批量推送。
6. 变更记录：在运维知识库中记录变更内容、时间、操作人和效果。

## 十四、日常维护与版本适配

### 14.1 内核升级时的参数检查

每次内核版本升级后，执行以下检查：

```bash
# 1. 重启后确认所有参数生效
verify-kernel-tuning.sh

# 2. 检查 dmesg 是否有参数相关告警
dmesg | grep -i -E "sysctl|hugepage|bbr"

# 3. 确认新内核是否移除了某些参数
diff <(sysctl -a 2>/dev/null | sort) <(sysctl -a 2>/dev/null | sort) # 对比新旧
```

### 14.2 参数变更的文档管理

建议在每个配置文件顶部包含以下注释信息：

```ini
# /etc/sysctl.d/99-production.conf
#
# 维护信息：
#   创建人：ops-team
#   创建时间：2026-11-06
#   最后修改：2026-11-06
#   适用环境：生产环境 Web 服务器
#   关联工单：JIRA-12345
#   回滚方法：删除此文件后执行 sysctl --system
#
# 变更记录：
#   2026-11-06 初始版本，基于 RHEL 9 / 内核 5.14
```

### 14.3 定期审计脚本

```bash
#!/bin/bash
# 文件：audit-sysctl.sh
# 用途：定期审计内核参数配置，集成到 cron 或运维巡检
# 建议执行频率：每周一次

set -euo pipefail

LOG="/var/log/sysctl-audit-$(date +%Y%m%d).log"
exec > >(tee -a "${LOG}") 2>&1

echo "===== sysctl 配置审计 ====="
echo "时间: $(date)"
echo "主机: $(hostname)"
echo "内核: $(uname -r)"
echo ""

# 检查是否有非标准配置文件
echo "--- /etc/sysctl.d/ 文件列表 ---"
ls -la /etc/sysctl.d/

# 检查 sysctl.conf 是否被直接修改
if [[ -s /etc/sysctl.conf ]]; then
    CUSTOM_LINES=$(grep -v '^#' /etc/sysctl.conf | grep -v '^$' | wc -l)
    if [[ ${CUSTOM_LINES} -gt 0 ]]; then
        echo "[WARN] /etc/sysctl.conf 包含 ${CUSTOM_LINES} 行自定义配置，建议迁移到 /etc/sysctl.d/"
    fi
fi

# 关键参数快照
echo ""
echo "--- 关键参数当前值 ---"
for param in vm.swappiness vm.dirty_ratio vm.max_map_count \
             net.core.somaxconn net.ipv4.tcp_congestion_control \
             fs.file-max fs.inotify.max_user_watches; do
    echo "${param} = $(sysctl -n "${param}" 2>/dev/null || echo 'N/A')"
done

# THP 状态
echo ""
echo "--- THP 状态 ---"
cat /sys/kernel/mm/transparent_hugepage/enabled 2>/dev/null || echo "N/A"

# 大页状态
echo ""
echo "--- 大页使用情况 ---"
grep HugePages /proc/meminfo 2>/dev/null || echo "N/A"

echo ""
echo "审计完成，日志已保存至 ${LOG}"
```

## 十五、总结

内核参数调优不是一次性的操作，而是一个持续演进的过程。核心要点回顾：

1. **配置管理**：使用 `/etc/sysctl.d/` 分文件管理，不直接修改 `/etc/sysctl.conf`。命名用大数字前缀确保优先级。
2. **内存管理**：`vm.swappiness` 降到 10（数据库 1），控制脏页比例避免 I/O 突发，NUMA 架构下禁止本地激进回收。
3. **大页管理**：数据库服务器禁用 THP（设为 `never`），配置显式大页。Redis 服务器必须禁用 THP。PostgreSQL 设置 `huge_pages = on` 使用显式大页。
4. **网络栈**：调大 `somaxconn` 和 SYN backlog，启用 BBR 拥塞控制，合理设置 TIME_WAIT 回收。注意应用层 backlog 需要配合调整。
5. **文件描述符**：`fs.file-max` 与进程级 `ulimit`/`LimitNOFILE` 需要同时调整。
6. **安全加固**：禁用源路由、ICMP 重定向，启用反向路径验证。K8s 节点的 `ip_forward` 必须保持开启。
7. **变更流程**：采集基线 → 测试验证 → 灰度部署 → 全量推送 → 持续监控。保留回滚能力。
8. **版本适配**：关注内核版本差异（参数是否存在、默认值变化），每次内核升级后重新验证。

生产环境的内核调优没有"银弹配置"，本文提供的参数值是经过广泛验证的合理起点，但最优值取决于你的具体工作负载。建议在变更前后进行基准测试，用数据证明效果，而不是盲目套用。

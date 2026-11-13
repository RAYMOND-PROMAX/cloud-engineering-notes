# Linux 系统性能调优实战：从 CPU 到磁盘 IO 的全链路分析

## 1. 问题背景

先看一个生产上非常典型的场景。

某天上午业务高峰，监控告警提示应用响应时间从 80ms 涨到 600ms，运维登上去第一眼看 `top`，load average 是 47（16 核机器），CPU 使用率却有 60% 空闲。这时候很多人的第一反应是"加机器"，但加完机器问题依旧。继续往下看，发现 `%wa`（IO 等待）长期在 35% 以上，`iostat -x` 里数据盘的 `%util` 接近 100%，`w_await` 到了 90ms——瓶颈根本不在 CPU，而在磁盘。最后定位到是一次表结构变更导致 MySQL 的执行计划突变，全表扫描把 Buffer Pool 打穿，随机读把盘打满。

这个案例说明三件事：

1. load 高不等于 CPU 忙。Linux 的 load 统计的是"处于运行态和不可中断睡眠态（D 状态）的任务数"，磁盘 IO 阻塞会直接推高 load，哪怕 CPU 完全空闲。
2. 性能问题必须定位到具体子系统。CPU、内存、磁盘 IO、网络，四个子系统互相影响，凭感觉调参基本等于碰运气。
3. 调优必须有基线、有证据、有验证。没有优化前的数据，优化后的效果无法证明；改了参数不验证，等于埋雷。

本文按 CPU → 内存 → 磁盘 IO → 网络的顺序，把每个环节的观测手段、指标含义、判断逻辑、调优动作、风险与回滚完整过一遍。所有命令基于 CentOS 7/8、Ubuntu 20.04+ 常见环境，个别差异会单独标注。

## 2. 适用场景

适合拿着这篇文章直接干活的场景：

- 业务接口变慢、批量任务变慢，需要判断瓶颈在哪个子系统
- 服务器 load 持续偏高，需要定位到具体进程甚至具体线程
- CPU 使用率异常（用户态高、内核态高、软中断高、steal 高），需要分别处理
- 内存告警、频繁触发 OOM Killer、swap 抖动导致业务卡顿
- 磁盘 IO 延迟高、IOPS 打满，需要确认是应用问题、配置问题还是盘的问题
- 新机器上线前需要做一轮系统性调优（THP、IO 调度器、内核参数）
- 云上实例性能不达标，怀疑宿主机争抢（steal）或云盘限流
- 给监控平台补充性能类告警指标，需要知道每个指标的正常范围和异常含义

不适合的场景：需要深入内核源码级别的性能分析（eBPF 高级用法、内核 bug 定位）不在本文展开，但排查路径会告诉你什么时候该升级到这一层。

## 3. 核心知识点

后面所有命令和判断都建立在下面这些概念上，先对齐认知，再动手。

### 3.1 四大子系统与 USE 方法

性能排查使用最广泛的方法是 Brendan Gregg 提出的 USE 方法，对每个资源检查三个维度：

| 资源 | Utilization 利用率 | Saturation 饱和度 | Errors 错误 |
| --- | --- | --- | --- |
| CPU | %usr/%sys 使用率 | run queue 长度、上下文切换 | steal（虚拟化被抢占） |
| 内存 | used/available 占比 | swap 换入换出、direct reclaim | OOM Kill |
| 磁盘 | %util、带宽占比 | IO 队列深度、await 升高 | IO error、只读挂载 |
| 网络 | 带宽占比 | 队列丢弃（backlog、drop） | retrans、CRC 错误 |

核心思想：**利用率没满不代表没饱和，饱和了不代表已经出问题，必须三个维度一起看**。比如一块 SATA SSD，`%util` 到 100% 可能还很健康（因为它可以并行处理多个请求），而机械盘 `%util` 80% 可能延迟已经爆炸。所以 `iostat` 里 `%util` 对 SSD/NVMe 参考价值有限，必须配合 `await` 一起看。

### 3.2 load average 的正确读法

`uptime` 输出：

```
 14:32:01 up 212 days,  3:41,  2 users,  load average: 47.02, 44.15, 39.87
```

三个数字是 1/5/15 分钟的活跃任务平均值，包含两种状态：

- R（Running/Runnable）：正在运行或在排队等 CPU —— 这是"CPU 型 load"
- D（Uninterruptible Sleep）：不可中断睡眠，绝大多数是在等磁盘 IO —— 这是"IO 型 load"

判断方法（16 核机器）：

```bash
# 分别统计 R 状态和 D 状态的内核调度实体数量
ps -eLo stat | grep -c '^R'
ps -eLo stat | grep -c '^D'
```

- R 多、D 少：CPU 不够或线程过多，往 CPU 方向查
- D 多、R 少：IO 阻塞（磁盘、NFS、某些内核锁），往 IO 方向查
- 三个数字的相对关系：1 分钟 > 15 分钟说明正在恶化；1 分钟 < 15 分钟说明在恢复

参考线（不是绝对标准，以业务基线为准）：load / CPU 核数 < 0.7 正常，0.7~1 需要关注，持续 > 1.5 且伴随延迟上升就该介入了。

### 3.3 CPU 使用率的分解

`top` / `mpstat` 里的每一段含义完全不同，处理方式也完全不同：

| 指标 | 含义 | 高了的常见原因 |
| --- | --- | --- |
| %usr | 用户态 | 应用真的在算：算法、序列化、压缩、死循环 |
| %sys | 内核态 | 系统调用频繁：大量小 IO、内存拷贝、锁竞争、模块问题 |
| %ni | 低优先级用户态 | 被 nice 过的批处理任务在跑 |
| %idle | 空闲 | —— |
| %wa | IO 等待 | 磁盘慢、IO 量大、存储链路问题 |
| %hi | 硬中断 | 中断风暴（罕见，通常是设备问题） |
| %si | 软中断 | 高包量网络（网卡中断集中在单核）、需要 RPS/RSS 调优 |
| %st | steal | 云主机/虚机宿主机超卖、邻居抢占 |

一句话结论：`%usr` 高查应用，`%sys` 高查系统调用和内核，`%wa` 高查磁盘，`%si` 高查网络包量，`%st` 高找云厂商。

### 3.4 内存：free 是骗人的，看 available

```
              total        used        free      shared  buff/cache   available
Mem:        8161180     4213920      302456       12340     3644804     3521456
Swap:             0           0           0
```

- free：完全没被使用的内存。Linux 会把空闲内存拿去做 page cache（缓存磁盘数据），所以 free 小不代表内存紧张，这是最常被误读的一行。
- buff/cache：页缓存 + 缓冲区，内存紧张时可以回收。
- available：内核估算的"不触发 swap 的情况下还能给应用用多少"。判断内存压力还需结合 si/so、内存 PSI 及容器 cgroup 使用量/上限。

内存相关的几个关键行为：

- page cache：读文件的缓存，提高重复读性能，可回收，不是泄漏。
- dirty pages：写文件时先写进内存的脏页，由内核回写线程异步刷盘。脏页积累过多（/proc/meminfo 的 Dirty）会造成突发 IO 抖动。
- direct reclaim（直接回收）：当应用申请内存而内核来不及回收时，应用线程被拉去自己回收内存，表现为业务线程莫名卡几十到几百毫秒，/proc/vmstat 里 allocstall_* 增长即是。
- swap：内存不足时把冷页挪到磁盘，一旦热数据被换出，访问延迟从纳秒级掉到毫秒级。数据库类机器应按工作集、内存余量和延迟目标确定 swap 策略。
- OOM Killer：内存真的耗尽时，内核按 oom_score 杀进程，日志在 dmesg / /var/log/messages，被杀记录 Out of memory: Killed process xxx。

### 3.5 磁盘 IO 的四个核心指标

`iostat -x 1` 输出里，真正用于判断的是这几列（sysstat 12+ 版本列名有变化，已标注）：

| 指标（旧 / 新） | 含义 | 怎么用 |
| --- | --- | --- |
| r/s、w/s | 每秒读/写次数（IOPS） | 和盘的标称 IOPS 对比判断是否打满 |
| rkB/s、wkB/s | 读/写带宽 | 和盘的标称吞吐对比 |
| r_await | 读请求平均延迟（ms），含排队时间 | 最关键指标，见 3.5.1 |
| w_await | 写请求平均延迟 | 同上 |
| aqu-sz（旧 avgqu-sz） | 平均队列深度 | 持续 > 盘并行度说明饱和 |
| %util / %busy | 设备有 IO 在处理的时间占比 | 对机械盘参考意义大，对 SSD/NVMe 会低估 |

#### 3.5.1 await 是"平均延迟含排队"，别被平均值骗了

`await` 是平均完成时间，排队时间算在内。随机读场景下，盘的并行度一旦超过队列，延迟非线性上涨：队列深度 1 时 SATA SSD 延迟 0.1ms，队列深度 32 时可能到 2ms。所以：

- await 高 + aqu-sz 高：盘被打满，要么减少 IO 要么换更快的盘
- await 高 + aqu-sz 低（接近 0）+ 带宽没满：大概率盘本身劣化（SSD 掉速、坏道、存储网络问题），这种情况很危险，赶紧迁移数据

不同介质的延迟参考线（经验值，最终以业务 SLA 为准）：

| 介质 | 正常 await（随机读） | 需要关注 |
| --- | --- | --- |
| NVMe SSD | < 0.2ms | > 1ms |
| SATA SSD | < 0.5ms | > 3ms |
| 云盘（ESSD/云 SSD） | 0.5~2ms（受云厂商限流影响） | 持续 > 5ms |
| 7200 转机械盘 | 8~15ms | > 25ms |

### 3.6 一个必须掌握的公式：吞吐量 = IOPS × 平均 IO 大小

很多"带宽没满但延迟高"的问题，本质是应用把大块顺序读变成了小块随机读。排查时用 `biosnoop`（bcc 工具，若环境没有则用第 7 节替代方法）或简单判断：`rkB/s ÷ r/s` 得到平均读大小，如果是 4KB 级别而业务预期是大文件顺序读，说明访问模式变了（典型原因：索引失效、缓存失效、执行计划突变）。

### 3.7 /proc 关键文件速查

排查到深水区，多数工具的底层就是这些文件。知道去哪看，比多装一个工具更重要：

| 文件 | 内容 | 典型用途 |
| --- | --- | --- |
| /proc/loadavg | load 三值 + 运行/总任务数 | 脚本采集 load |
| /proc/meminfo | 内存全景 | available、Dirty、Committed_AS |
| /proc/<pid>/status | 进程状态、RSS、线程数（Threads 字段） | 线程爆炸排查 |
| /proc/<pid>/io | 进程 IO 读写量 | 区分缓存读写与落盘读写 |
| /proc/<pid>/wchan | 进程当前阻塞的内核函数 | D 状态进程卡在哪 |
| /proc/<pid>/smaps_rollup | 进程内存汇总（Pss/Uss） | 内存归因、多进程共享内存 |
| /proc/<pid>/limits | 进程实际 ulimit | 验证 limits 配置 |
| /proc/pressure/{cpu,memory,io} | PSI 压力指标（内核 4.20+） | 比 load 更精准的饱和信号 |
| /proc/net/softnet_stat | 每核收包处理与积压 | 软中断排查 |
| /proc/interrupts | 每 CPU 每 IRQ 计数 | 中断是否偏科 |
| /proc/vmstat | 全部 VM 计数器 | allocstall、compact_stall、pgscan |

特别提一下 **PSI（Pressure Stall Information）**。传统 load 无法回答"到底是谁在拖累系统"，PSI 直接给出因 CPU/内存/IO 阻塞而损失的时间百分比：

```bash
# some：至少一个任务被阻塞的时间占比；full：所有任务同时被阻塞的占比
cat /proc/pressure/io
some avg10=12.34 avg60=8.21 avg300=5.02 total=987654321
full avg10=3.21 avg60=2.11 avg300=1.05 total=123456789
```

判断逻辑：`io full avg10 > 0` 意味着所有可运行任务在同一时刻都在等 IO，业务延迟必然受影响。CentOS 7（3.10 内核）没有 PSI，用 RHEL 的 tuned 或换观测手段；较新的 Ubuntu 20.04（5.x 内核部分支持）/22.04 默认可用。

## 4. 整体调优思路：先观察，后动手

给一套可以直接抄进工单模板的流程：

```
第一步：确认问题真实存在
  └─ 拉监控：业务响应时间、QPS、错误率，确定开始恶化的时间点 T

第二步：圈定子系统
  └─ 三板斧：uptime / vmstat 1 / iostat -x 1，10 秒内初步判断方向

第三步：定位到进程
  └─ top -H（线程级）、pidstat 1、iotop -oP，找出具体进程/线程

第四步：定位到行为
  └─ perf top 看函数热点、strace 看系统调用、/proc/<pid>/io 看 IO 明细

第五步：制定方案
  └─ 优先应用层/架构层方案（加缓存、改 SQL、削峰），其次系统参数，最后加硬件

第六步：灰度实施 + 验证
  └─ 单机先改 → 对比基线 → 观察一个业务高峰 → 再扩大范围

第七步：保留回滚路径
  └─ 所有参数改动有备份文件、有恢复命令
```

两条铁律：

1. 一次只改一个参数。同时改三个参数，效果好了不知道为什么，效果坏了也不知道回滚哪个。
2. 改之前先想好"怎么证明有效"。指定要观察的指标和目标值，别改完了用"感觉流畅了"来验收。

### 4.1 变更前基线采集脚本（只读，可直接生产执行）

调优或变更前在目标机跑一次，输出落盘存档，事后对比全靠它：

```bash
#!/usr/bin/env bash
# collect_baseline.sh - 采集系统性能基线，纯只读命令，无副作用
set -u
OUT="/tmp/baseline_$(hostname)_$(date +%Y%m%d_%H%M)"
mkdir -p "${OUT}"
exec > >(tee "${OUT}/run.log") 2>&1

echo "== uptime ==";            uptime
echo "== cpu ==";               mpstat -P ALL 1 3
echo "== vmstat ==";            vmstat 1 5
echo "== disk ==";              iostat -x 1 3
echo "== memory ==";            free -h
echo "== meminfo key ==";       grep -E 'MemAvailable|Dirty|SwapCached|CommitLimit|Committed_AS' /proc/meminfo
echo "== top procs ==";         ps -eo pid,pcpu,pmem,rss,comm --sort=-pcpu | head -15
echo "== net dev ==";           sar -n DEV 1 3
echo "== connstat ==";          ss -s
echo "== io sched ==";          for d in /sys/block/*/queue/scheduler; do echo "${d}: $(cat "${d}")"; done
echo "== thp ==";               cat /sys/kernel/mm/transparent_hugepage/enabled 2>/dev/null
echo "== sysctl key ==";        sysctl vm.swappiness vm.dirty_ratio vm.dirty_background_ratio net.core.somaxconn 2>/dev/null
echo "== limits ==";            ulimit -n
echo "== kernel ==";            uname -r
echo "== dmesg tail ==";        dmesg -T | tail -30

echo "baseline saved to ${OUT}"
```

脚本说明：全部为只读观测命令；输出目录在 `/tmp` 且带主机名与时间戳，多机收集不冲突；无删除、无写系统目录动作，普通用户运行部分命令（如 `dmesg`、`iostat` 个别项）需要 root 或相应权限，权限不足时对应段落会为空，不影响其余采集。批量多机分发时外层套一层 `for host in $(cat hosts); do scp collect_baseline.sh "${host}:/tmp/" && ssh "${host}" 'bash /tmp/collect_baseline.sh'; scp "${host}:/tmp/baseline_*.tar.gz" ./ ...`（示例思路），每台机器独立目录、独立日志，失败机器单独重跑，不要 `set -e` 一断全断。

## 5. CPU 调优实战

### 5.1 全局观察：先看整机

**目的**：确认 CPU 压力大小、类型分布、是否均匀。

```bash
# 整体负载 + 排名，按 P 按 CPU 排序，1 秒刷新
top

# 每核分布，重点看有没有单核跑满（单线程瓶颈或中断未打散）
mpstat -P ALL 1 5
```

`mpstat` 预期输出（正常）：各核 `%idle` 分布均匀，无明显单核 100%。

异常形态及判断：

- 单核 100%，其他核空闲：应用是单线程模型（如 Node.js 主线程、Redis 主线程），或网卡中断绑定在单核。加核无用，需要应用多实例或中断打散。
- 所有核 `%usr` 高：应用真实计算量大，考虑扩容或优化代码。
- 所有核 `%sys` 高（> 25%）：系统调用风暴。下一步 perf top 看内核函数，常见的是 copy_user_enhanced_fast_string（大量数据拷贝）、spinlock 竞争、schedule（线程过多疯狂切换）。
- `%st` > 5% 持续存在：云主机宿主机超卖，本地无解，提工单或换实例规格。

### 5.2 上下文切换与运行队列

```bash
# cs 列 = 每秒上下文切换；r 列 = 运行队列长度
vmstat 1 5
```

预期输出：

```
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 2  0      0 302144  12340 3644804    0    0   120  4800 3200 8500 45 12 40  3  0
```

判断逻辑：

- r 持续大于核数：CPU 饱和，请求在排队，延迟必然上升。
- cs 数万属于正常（取决于核数和线程数）；达到几十万且 %sys 同时高，说明锁竞争激烈或线程池配置不合理（比如把线程池开到几千）。
- 定位是哪个进程在刷切换：

```bash
# 每秒每个进程的非自愿切换（cswch/s 自愿、nvcswch/s 非自愿）
pidstat -w 1 5
```

非自愿切换高 = CPU 不够分；自愿切换高 = 线程在等锁/等 IO。

### 5.3 定位到线程和函数

```bash
# 线程级 CPU
top -H -p $(pgrep -d, java | head)   # 示例：观察 java 进程所有线程
# 更清晰的方式：
pidstat -t -p <pid> 1 5

# 函数热点（perf 需要内核对应版本的 debug 信息才能看清符号）
perf top -p <pid>
perf record -F 99 -g -p <pid> -- sleep 30 && perf report
```

`perf top` 输出里如果是业务函数 → 交给开发优化；如果大量时间在内核 `native_queued_spin_lock_slowpath` → 锁竞争，检查高并发下的内核锁（常见于大量小文件、大量进程创建、`/proc` 高频读取）。

### 5.4 CPU 相关调优动作

**动作一：调整进程优先级（低风险）**

```bash
# 查看 nice 值
ps -o pid,ni,comm -p <pid>
# 降低批处理任务优先级，避免和业务抢 CPU
renice +10 -p <pid>
# 启动时直接指定
nice -n 10 ./backup_job.sh
```

风险提醒：`renice` 负值（提权）会让业务进程抢占所有其他进程，生产不建议随意给核心业务设负 nice——一旦它跑飞，SSH 都会卡死。用正 nice 压制批处理是安全的，用负 nice 提权是危险的。

**动作二：CPU 亲和性绑核（中风险）**

适用场景：软中断集中单核、NUMA 跨节点访存、实时性要求高的进程。

```bash
# 查看网卡中断分布
cat /proc/interrupts | grep eth0
# 将某进程绑定到 2-7 核
taskset -cp 2-7 <pid>
```

风险：绑核范围设置不当会造成绑定的核 100% 而系统其他部分无法使用这些核。必须先在 `mpstat -P ALL 1` 里确认目标核有余量。回滚：`taskset -cp 0-$(($(nproc)-1)) <pid>` 恢复全部核。

**动作三：cgroup v2 资源限制（推荐做法）**

给批处理/容器设上限，防止饿死业务：

```bash
# 示例思路：用 systemd 给某个服务限 2 核
systemctl set-property backup.service CPUQuota=200%
# 验证
systemctl show backup.service -p CPUQuotaPerSecUSec
```

**动作四：软中断（%si）高的处理**

```bash
# 看 softnet_stat，第 2 列非零说明该 CPU 出现过收包积压
cat /proc/net/softnet_stat
# 检查网卡队列数与多队列开启情况
ethtool -l eth0
ethtool -L eth0 combined 8    # 高风险操作：执行瞬间可能有短暂丢包，需在变更窗口做
```

### 5.5 CPU 频率与 governor：性能"莫名"波动的隐藏原因

同样的利用率，降频运行的机器延迟完全不同。云上一般不用管这层，自建机房和老内核机器要检查：

```bash
# 查看可用调节器与当前值（需要 cpupower 工具，属于 kernel-tools/sysstat 配套包）
cpupower frequency-info
# 逐核实时频率，负载来了频率应立刻拉满
cpupower frequency-monitor -i 1
```

| governor | 行为 | 适用 |
| --- | --- | --- |
| performance | 恒定最高频（含 turbo 策略允许范围内） | 延迟敏感型：数据库、交易、网关 |
| powersave | 内核自主调频（intel_pstate 驱动下实际也会睿频） | 新平台默认，一般无需改 |
| ondemand | 旧 acpi-cpufreq 驱动按需升频 | 老平台，延迟敏感场景不推荐 |

```bash
# 临时设置（示例：切 performance）
cpupower frequency-set -g performance
```

判断逻辑：如果 `cpupower frequency-monitor` 显示高峰期有核停留在低频（如主频 2.4G 的机器跑在 1.2G），而利用率并不低，先切 `performance` 再看延迟变化。风险提醒：切 performance 增加功耗与发热，老机器注意散热余量；回滚即 `cpupower frequency-set -g <原值>`。持久化用 `cpupower frequency-set` 配合 systemd 服务或 tuned profile（tuned 的各 profile 本身带 governor 设置，优先走 tuned 统一管理）。

### 5.6 NUMA：跨内存节点访存的隐性损耗

多路 CPU 服务器上，进程在 A 节点 CPU 上跑却访问 B 节点内存，延迟高 30%~100%。Redis、MySQL、大内存 Java 服务对此敏感。

```bash
# 拓扑与内存分布（numactl 属于 numactl 包）
numactl --hardware
# 某进程的内存分布在哪个节点
numastat -p <pid>
# 整机各节点统计，注意 numa_miss/foreign 是否持续增长
numastat
```

处理手段：

```bash
# 绑到节点 0 的 CPU 与内存（示例思路，配合 cpuset）
numactl --cpunodebind=0 --membind=0 redis-server /etc/redis.conf
# 禁止跨节点分配（严格本地，可能因单节点内存不足分配失败，评估后再用）
numactl --preferred=0 ...
```

风险提醒：`--membind` 严格绑定后，该内存节点耗尽会直接分配失败（表现为 OOM 而整机内存充足），必须预留水位；`--interleave=all` 适合大共享内存场景但平均延迟不一定更优。NUMA 设置错误的回滚是去掉 numactl 包装重启进程，属于需要重启窗口的操作。

### 5.7 线程与进程规模：被忽视的"系统开销放大器"

线程数爆炸本身就是一种性能病：调度成本上升、上下文切换爆炸、内核栈内存浪费。

```bash
# 整机线程总数，数万即异常
ps -eLf | wc -l
# 单进程线程数排序
for p in $(pgrep -f 'java|nginx|mysqld'); do echo "$(ls /proc/$p/task 2>/dev/null | wc -l) $p $(cat /proc/$p/comm)"; done | sort -rn | head
# 整机进程数对比 pid_max
ps -e --no-headers | wc -l; cat /proc/sys/kernel/pid_max
```

判断逻辑：线程总数与核数差两个数量级以内一般没问题；出现"16 核机器 3 万线程"这类情况，`vmstat` 的 `cs` 几乎一定同步爆炸。处理顺序：应用线程池收敛（根本解法）→ cgroup 限制子进程数量 → 不建议直接调大 `kernel.pid_max` 掩盖问题。

### 5.8 CPU 案例闭环

**现象**：16 核机器，凌晨批处理跑批从 20 分钟延长到 90 分钟。**初步判断**：`vmstat 1` 显示 `r` 稳定在 40+，`%usr` 95%，CPU 型饱和。**命令检查**：`pidstat -u 1` 发现跑批脚本拉起了 400 个并发子进程。**关键指标**：进程数远超核数，cs 飙到 30 万/秒。**根因定位**：跑批脚本 `xargs -P 0`（无限并发），白天机器空闲没暴露，业务量上来后与在线任务互踩。**修复方案**：`xargs -P 8` 限制并发，并 `nice -n 10` 启动。**验证结果**：跑批 25 分钟完成，在线接口 P99 无波动。**回滚预案**：修改的是脚本参数，直接 git revert 脚本即可。**复盘总结**：所有批处理脚本模板强制 `-P` 参数显式声明，纳入 code review 检查项。

## 6. 内存调优实战

### 6.1 快速评估：内存到底紧不紧张

```bash
# 看 available 列，不看 free 列
free -h
# 持续观察换页：si/so 持续非零 = 正在用 swap，性能问题基本坐实
vmstat 1 5
# 缺页速率：majflt/s 非零 = 在从磁盘读页（缺文件缓存或被换出后又访问）
pidstat -r 1 5
```

判断逻辑（按严重度排序）：

1. vmstat 的 si/so 持续非零 → 已在 swap，最高优先级处理。
2. available 接近 0 → 即将 OOM。
3. sar -B 1 5 里 pgscand/s（kswapd 扫描速率）持续很高、pgstalt/s（直接回收停顿）非零 → 内存回收风暴，业务延迟毛刺的常见根因。

### 6.2 OOM 排查

```bash
# 查 OOM 记录
dmesg -T | grep -i -E 'out of memory|oom'
journalctl -k | grep -i oom      # systemd 系统
# 查看某进程被 OOM kill 的历史（需要内核开启对应配置）
grep -i oom /var/log/messages
```

`dmesg` 里会打印每个进程的 RSS 打分表，重点看被杀进程的 `oom_score_adj`。业务进程应显式设置保护：

```bash
# 核心进程设置 -1000（完全免杀）要非常谨慎，等于该进程永远优先让别人牺牲
echo -800 > /proc/<pid>/oom_score_adj
# 持久化：systemd 服务单元里
# [Service]
# OOMScoreAdjust=-800
```

风险提醒：把数据库主进程设成 -1000 后，内存真正耗尽时被杀的是 SSH、监控 agent，机器会变成"数据库活着但连不上"的僵尸状态。更安全的做法是通过限制内存（cgroup `MemoryMax`）+ 应用自身配置（如 MySQL `innodb_buffer_pool_size`）从源头避免 OOM。

### 6.3 swappiness 与 swap 策略

```bash
# 当前值（默认 60）
cat /proc/sys/vm/swappiness
# 临时生效：降低内核主动换页倾向
sysctl -w vm.swappiness=1
# 持久化
echo 'vm.swappiness = 1' > /etc/sysctl.d/99-perf.conf
```

说明：`swappiness=1`（不是 0，0 在旧内核会导致极端行为）让内核尽量回收 page cache 而不是换出匿名页。数据库、缓存、JVM 类机器推荐。完全关闭 swap（`swapoff -a`）是高风险操作：一旦内存超卖，行为从"变慢"变成"直接 OOM 杀进程"，必须有内存监控告警配套。**swapoff -a 属于修改系统级行为，执行前确认：①监控有内存告警；②业务有内存上限约束；③回滚命令是 swapon -a。**

### 6.4 脏页回写参数：治"周期性卡顿"

现象特征：业务每间隔一段时间规律性卡一下，`iostat` 里 `w_await` 规律性尖刺。这是脏页积累到阈值后集中回写导致的。

```bash
# 当前值
sysctl vm.dirty_ratio vm.dirty_background_ratio vm.dirty_expire_centisecs
```

| 参数 | 默认 | 含义 |
| --- | --- | --- |
| vm.dirty_background_ratio | 10 | 脏页占内存比例达到此值，后台回写线程开始刷 |
| vm.dirty_ratio | 20 | 达到此值，写进程被阻塞同步回写（卡顿的直接原因） |
| vm.dirty_expire_centisecs | 3000 | 脏页最长存活 30s，超期由回写线程处理 |
| vm.dirty_writeback_centisecs | 500 | 回写线程唤醒周期 5s |

大内存机器（如 256G）用比例会算出天文数字的脏页上限，建议改用绝对值：

```ini
# 持久化示例：写入 /etc/sysctl.d/99-dirty.conf 再 sysctl --system 加载
vm.dirty_background_bytes = 268435456     # 256MB 开始后台回写
vm.dirty_bytes = 1073741824               # 1GB 才阻塞写
```

`dirty_background_bytes` 与 `dirty_background_ratio` 互斥，`dirty_bytes` 与 `dirty_ratio` 互斥；使用字节阈值时检查其他文件是否设置了对应比例阈值。

风险提醒：参数调小会让 IO 更频繁（平滑但持续），对 IO 已经接近饱和的盘是雪上加霜。调参前先看 `iostat -x` 确认盘有余量。验证方法：调整前后各采集 30 分钟 `iostat -x 1`，对比 `w_await` 的 P99（用 awk 对日志统计分位，见第 9 节）。

### 6.5 透明大页（THP）

THP 的内存整理可能使部分延迟敏感负载出现尖刺；是否关闭应依应用版本建议及实测决定。

```bash
# 当前模式：always=madvise=never
cat /sys/kernel/mm/transparent_hugepage/enabled
# 临时关闭 THP（madvise 只是改为应用显式请求时启用）
echo never > /sys/kernel/mm/transparent_hugepage/enabled
echo never > /sys/kernel/mm/transparent_hugepage/defrag
```

持久化（两种常见方式）：

```bash
# 方式一：grub 内核参数（需重启生效，变更窗口执行）
# GRUB_CMDLINE_LINUX 追加 transparent_hugepage=never
grub2-mkconfig -o /boot/grub2/grub.cfg
# 方式二：tuned 配置文件或 rc.local（视发行版规范而定）
```

风险与回滚：修改 grub 属于高风险操作，错误配置会导致无法开机——改之前备份 `/etc/default/grub`，改完先在测试机验证；回滚即恢复备份文件重新 `grub2-mkconfig`。重启操作在高峰期禁止执行。

### 6.6 内存泄漏识别与归因

"内存缓慢上涨然后 OOM"是最常见的一类慢性病，处理流程：

```bash
# 第一步：确认是应用内存还是缓存/内核内存
# 关注 used 与 buff/cache 的趋势差（sar -r 历史记录）
sar -r -f /var/log/sa/sa23 | tail -30
# available 持续下降且 buff/cache 同步下降 → 应用真实占用在涨

# 第二步：进程级归因。RSS 会重复计算共享内存，多实例场景看 Pss/Uss
cat /proc/<pid>/smaps_rollup
# 关键字段：Rss（物理占用）、Pss（按比例分摊共享）、Private_Dirty（私有脏页，最接近"该进程独有的真实开销"）

# 第三步：持续跟踪单个进程
# 示例思路：每 60 秒记录一次指定进程的 RssAnon，跑一天看斜率
while true; do
  ts=$(date '+%F %T')
  rss=$(awk '/^RssAnon/{print $2}' "/proc/${PID_TO_WATCH}/status")
  echo "${ts} ${rss}" >> /var/log/mem_watch.log
  sleep 60
done
```

判断逻辑：

- RssAnon（匿名页）持续上涨：应用堆/原生内存泄漏的典型特征。
- Private_Dirty 上涨但业务无感知且增速与流量无关：怀疑 glibc malloc 碎片或 mmap 泄漏，MALLOC_ARENA_MAX 与 jemalloc 是常用验证手段（验证型变更，需重启进程生效）。
- slab 占用巨大（/proc/meminfo 的 SReclaimable/SUnreclaim）：内核侧问题，slabtop -o | head 看哪类对象最多，dentry 巨量可安全回收，其他对象暴涨要查内核模块/驱动。

安全回收 dentry/inode 缓存（低风险，应急可用）：

```bash
sync && echo 2 > /proc/sys/vm/drop_caches
```

说明：`drop_caches` 只丢干净缓存，不丢数据，但会造成接下来的读全部打到磁盘——IO 型业务高峰期禁止执行，`echo 3`（含 dentry+inode+page）影响更大。

### 6.7 overcommit 与 Committed_AS

Java/数据库这类"申请多、用得少"的负载，默认启发式 overcommit（`vm.overcommit_memory=0`）通常够用。需要关注的场景：

```bash
# 内核"承诺出去"的总量 vs 物理+swap，Committed_AS 远超 CommitLimit 时风险高
grep -E 'CommitLimit|Committed_AS' /proc/meminfo
```

- Redis 做 BGSAVE fork 时，写时复制（COW）需要最坏情况下"再准备一份内存"，官方建议 vm.overcommit_memory=1，否则 fork 可能失败（日志有 Can't allocate memory for the background job）。
- 改成 1（总是允许）的代价：内存承诺失去闸门，耗尽时直接 OOM。适合"应用自己有内存上限、且配了 cgroup/告警"的受控场景，不适合混乱的混部机器。
- 改成 2（严格不超卖）会让 Java 大堆进程申请直接失败，除非对容量有精确账本，否则别用。

## 7. 磁盘 IO 调优实战

### 7.1 第一层：找到瓶颈设备

```bash
# 每秒每设备明细
iostat -x 1 5
```

拿到输出后的判断顺序：

1. 哪块设备的 await/r_await/w_await 超过了第 3.5 节的参考线？
2. 该设备是 r/s 高还是 w/s 高？平均 IO 大小（rkB/s ÷ r/s）是多少？
3. %util 是否接近 100%？aqu-sz 是否持续 > 10？
4. 是单盘慢还是 RAID/云盘整体慢？（RAID 卡 battery 失效导致 write-back 降级为 write-through，是"写入突然慢十倍"的经典原因，检查 storcli64 /c0 show all 或对应厂商工具。）

云主机注意：云盘有 IOPS/吞吐限流（如 ESSD PL1 有上限），打满限流后延迟升高是正常保护行为，看云监控的"突发/限流"指标确认，本地 `iostat` 看不出是被限流还是盘慢。

### 7.2 第二层：找到 IO 大户

```bash
# 每进程读写速率（iotop 需要 root）
iotop -oP -d 1
# 没有 iotop 时用 pidstat（sysstat 自带）
pidstat -d 1 5
# 某进程的 IO 明细
cat /proc/<pid>/io
```

`/proc/<pid>/io` 字段解读：

- read_bytes / write_bytes：真实落盘的字节数（走 page cache 未落盘的读不计数）
- rchar / wchar：应用视角读写的字节数（含缓存命中）
- wchar 巨大而 write_bytes 小 → 写都命中缓存，正常；两者接近 → 直写或缓存已失效

典型元凶类型：数据库刷盘/全表扫描、备份任务、日志集中切分（logrotate 压缩）、监控 agent 全量采集、异常进程写死循环日志。

### 7.3 第三层：区分是"量大"还是"盘慢"

**目的**：同样的 `await`，一种情况是应用打出海啸般 IOPS（盘正常，应用有病），一种是低 IOPS 下高延迟（盘有病）。

方法：对比盘的标称能力 + 用独立压测工具验证。

```bash
# 简单同步延迟测试（示例思路，测完删除测试文件）
ioping -c 10 /data
# 吞吐测试参见下文 fio
```

如果 `iostat` 显示该盘 `r/s` 只有 200，`await` 却有 50ms，而 `ioping` 也复现 50ms+ 延迟 → 盘或存储链路真的慢了，机械盘查坏道：

```bash
smartctl -a /dev/sda | grep -E 'Reallocated|Pending|Uncorrect'
```

任何非零的 Reallocated/Pending sector 都要启动换盘流程。SSD 看 `wear_leveling` / `percentage_used`（NVMe：`smartctl -a /dev/nvme0n1 | grep -E 'Percentage Used|Available Spare'`）。

### 7.4 IO 调度器选择

```bash
# 查看当前调度器（方括号为当前）
cat /sys/block/sda/queue/scheduler
# 临时切换（立即生效，重启失效）
echo mq-deadline > /sys/block/sda/queue/scheduler
```

| 调度器 | 适用 |
| --- | --- |
| none | NVMe 盘（盘内并行度高，内核排序反而多此一举） |
| mq-deadline | SATA SSD、常规服务器默认推荐 |
| bfq | 桌面、需要公平性和低交互延迟的场景 |
| kyber | 低延迟 NVMe 上想保留一点调控时可选 |

数据库机械盘/RAID 场景保留 `deadline/mq-deadline`；`read_ahead_kb` 对顺序读为主的负载可以适当加大（默认 128，顺序读大文件场景可试 256~512）：

```bash
cat /sys/block/sda/queue/read_ahead_kb
echo 256 > /sys/block/sda/queue/read_ahead_kb
```

风险提醒：改调度器对在线 IO 影响是瞬时的，个别内核版本在 IO 高峰期切换会短暂卡 IO，仍建议低峰操作；持久化用 udev 规则，不要只写 rc.local（时序可能晚于设备使用）。回滚：写回原值即可。

### 7.5 文件系统挂载选项

```bash
# 当前挂载参数
mount | grep ' /data '
```

`noatime` 可减少访问时间更新，但现代文件系统通常默认 `relatime`；先评估业务是否依赖 atime，再比较实际收益。

修改方式：编辑 `/etc/fstab` 对应行加 `noatime`，然后 `mount -o remount /data`（remount 通常安全，但生产上仍算修改挂载配置，低峰执行）。风险提醒：未加 `nofail` 的必需挂载若开机找不到设备可能阻塞启动；加上 `nofail` 可以继续启动，但依赖挂载的业务仍可能误写根盘；`remount` 改 `ro` 方向会立刻引发业务写失败，方向一定要确认清楚。

### 7.6 应急：IO 被打满时的临时控制

紧急恢复业务的优先级高于查根因。可以用 cgroup 临时压制元凶进程：

```bash
# systemd 下临时限制某服务磁盘写入速率（需指定块设备，且依赖 cgroup IO 控制器）
systemctl set-property backup.service 'IOWriteBandwidthMax=/dev/sda 20M'
```

若确认元凶是可牺牲的进程，`kill` 前先确认：`ps -fp <pid>` 看清命令行，避免杀错 PID 相同的业务进程；优先 `kill -TERM` 观察 30 秒再 `-KILL`。`kill -9` 对正在写文件的数据库进程可能造成恢复时间变长，非极端情况不要用。

### 7.7 用 fio 做标准化压测（换盘、调参前后对比的标准动作）

`ioping` 只能测延迟，验证盘的 IOPS/吞吐能力要用 fio。**压测属于消耗型操作：会产生真实 IO 流量、会写测试文件，生产盘压测前必须确认容量余量和业务高峰窗口。**

```bash
yum install -y fio   # 或 apt install fio

# 4K 随机读 IOPS（模拟数据库随机读）
fio --name=randread --ioengine=libaio --direct=1 --rw=randread \
    --bs=4k --iodepth=32 --numjobs=1 --size=1G --runtime=60 \
    --time_based --group_reporting --filename=/data/fio_randread_test.tmp

# 1M 顺序写吞吐（模拟备份/日志写入）
fio --name=seqwrite --ioengine=libaio --direct=1 --rw=write \
    --bs=1M --iodepth=8 --numjobs=1 --size=2G --runtime=60 \
    --time_based --group_reporting --filename=/data/fio_seqwrite_test.tmp

# 4K 随机写延迟分布（关注 lat 的 clat 分位）
fio --name=randlat --ioengine=libaio --direct=1 --rw=randwrite \
    --bs=4k --iodepth=1 --numjobs=1 --size=512M --runtime=30 \
    --time_based --group_reporting --filename=/data/fio_randlat_test.tmp
```

先在独立测试目录按目标容量预置随机读测试文件；fio 对不存在的读文件可能自动初始化，初始化写入会污染读测量。确认专用文件路径与容量（本例最多约 3.5GiB），读写测试分开执行。参数说明：`--direct=1` 绕过 page cache 测真实盘能力（对比"带缓存"与"裸盘"差异时才是故意的例外）；`--iodepth` 是队列深度，模拟并发；`--numjobs` 是并发线程数。

结果判读：

- IOPS= 与 BW=：与盘的标称值/云盘规格对比，达到 80% 以上算正常发挥。
- clat percentiles：看 99.00th 和 99.99th 而不是平均 lat，云盘尾延迟毛刺在平均值里完全看不见。
- 随机写吞吐明显低于标称：先跑 30 分钟以上再看——SSD 有 SLC 缓存，缓存打满后进入"稳态"性能，短测全部失真。评估盘能力必须以稳态数据为准。

压测收尾（数据清理，低风险但仍需确认路径）：

```bash
rm -f /data/fio_randread_test.tmp /data/fio_seqwrite_test.tmp /data/fio_randlat_test.tmp
```

风险提醒：`filename` 指向已有业务文件会破坏数据，必须核对测试专用路径；清理前再次确认文件名。

### 7.8 文件系统层检查

块层正常但业务仍慢，问题常在文件系统层：

```bash
# inode 耗尽：df 看不出来，必须 -i
df -i
# 大量小文件目录：ls 卡住、find 慢，用 du 前先 df -i
# 挂载点异常变只读（ext4 元数据错误保护）
mount | grep 'ro,'
dmesg | grep -iE 'ext4|xfs' | tail
```

判断逻辑：

- df -i 100% 而 df -h 有空间：小文件打爆 inode（典型：session 文件、海量小图、未清理的临时文件）。解决是清文件而不是扩盘。
- dmesg 出现 EXT4-fs error / XFS (sdb1): Metadata I/O Error 且盘变 ro：文件系统进入保护模式，不要直接重启了事，先 dmesg 取证、确认备份完好，卸载后 fsck（ext4）或按 XFS 流程处理——对仍被占用的文件系统执行 fsck 会直接摧毁数据，这是本篇最危险的操作之一：执行前必须 umount，卸载不掉必须排查占用进程而不是 umount -f 强拆。
- dentry 缓存巨大（slab 中 dentry 项）通常是海量小文件访问的痕迹，本身可回收，但反映出应用在读很多小文件——回到 3.6 节的"访问模式突变"思路。

## 8. 网络概览（性能链路最后一环）

```bash
# 网卡带宽与错误
sar -n DEV 1 5
# 错误/丢弃
sar -n EDEV 1 5
# 连接状态统计
ss -s
# 重传
netstat -s | grep -i retrans
```

判断要点：`rxpck/s` 高但 `rxkB/s` 很小 → 小包风暴，查 `%si` 和 conntrack 表；`rxdrop/s`/`rxerr/s` 非零 → 网卡环形缓冲不足（`ethtool -G eth0 rx 4096`，需评估内存）或物理链路问题；`nf_conntrack_count` 接近 `nf_conntrack_max` → 新建连接被丢，`dmesg` 会有 `nf_conntrack: table full` 日志（属于内核参数修改，见第 10 节）。

网络层的两个高频深查手段：

```bash
# 监听队列溢出：Recv-Q 持续接近 Send-Q（上限）说明 accept 队列溢出
ss -lnt | head
# 溢出的硬证据：netstat -s 里 "times the listen queue of a socket overflowed" 持续增长
netstat -s | grep -i -E 'overflow|listen'
```

判断逻辑：队列溢出时调 `net.core.somaxconn` 和应用 backlog（Nginx 的 `listen ... backlog=8192`）要同时改，取两者较小值生效——只改系统参数不改应用监听参数是经典无效操作，验证方式是上面的 overflow 计数停止增长。

`tcp_tw_reuse`、`tcp_fin_timeout`、TIME_WAIT 相关参数属于高频误调区：TIME_WAIT 本身是正常状态、消耗的是内存而非端口（只要不是单机对单一目的端口发起数万级新建连接），盲目调参收益小、风险真实（`tcp_tw_reuse` 需要两端时间戳配合，误用于某些 NAT 场景曾出过连接错乱事故）。端口耗尽的真实解法是加源端口段、加客户端 IP 或改长连接，优先做这些。

## 9. 常用命令

速查表，按排查顺序排列：

```bash
# ---------- 整机快速三板斧 ----------
uptime                          # load 三值，判断趋势
vmstat 1 5                      # r/b 队列、si/so 换页、cs 切换、us/sy/wa
iostat -x 1 5                   # 每设备 IOPS/延迟/util
free -h                         # available 为准
dmesg -T | tail -50             # 内核最近事件（OOM、IO error、网卡 up/down）

# ---------- CPU ----------
mpstat -P ALL 1 5               # 每核分布
pidstat -u 1 5                  # 每进程 CPU
pidstat -w 1 5                  # 每进程上下文切换
top -H                          # 线程级
perf top -p <pid>               # 函数热点
mpstat -I SUM 1 5               # 中断速率
cat /proc/interrupts            # 中断分布

# ---------- 内存 ----------
free -h
cat /proc/meminfo | grep -E 'MemAvailable|Dirty|SwapCached|Committed_AS'
sar -B 1 5                      # pgscand/s 扫描、pgstalt/s 直接回收
pidstat -r 1 5                  # 每进程缺页（majflt 最关键）
smem -rs swap                   # 按 swap 排序看进程（需安装 smem）

# ---------- 磁盘 ----------
iostat -x 1
iostat -x -y 1                  # 忽略首行自启动以来的历史均值，只看实时段
iotop -oP
pidstat -d 1
blktrace 场景用 blktrace+btrace（深入，慎用，本身有开销）
smartctl -a /dev/sdX            # 盘健康度
lsscsi                          # 设备清单
df -Th; df -i                   # 空间与 inode 都要看
mount

# ---------- 网络 ----------
sar -n DEV 1; sar -n EDEV 1; sar -n TCP 1
ss -s; ss -ant | awk '{print $1}' | sort | uniq -c
netstat -s | grep -iE 'retrans|listen'
cat /proc/net/softnet_stat
ethtool -S eth0 | grep -iE 'drop|err|discard'

# ---------- 历史回溯（没实时监控时救命用） ----------
sar -u -f /var/log/sa/saXX      # CPU 历史
sar -r -f /var/log/sa/saXX      # 内存历史
sar -d -f /var/log/sa/saXX      # 磁盘历史
# saXX 里 XX 是日期数字，如 sa23 表示 23 号
```

一个小分析脚本：从 `iostat -x` 的表头动态定位 `w_await` 列，按采样窗口内每秒的平均延迟计算分位数。它不是每次 IO 的 P99：

```bash
#!/usr/bin/env bash
# 采集 60 秒 iostat，统计指定设备的 await 分布
DEV="${1:-sda}"
iostat -x -y 1 60 | awk -v d="$DEV" '
  $1 == "Device" || $1 == "Device:" {
    col=0; for (i=1; i<=NF; i++) if ($i=="w_await") col=i
    next
  }
  col && $1==d {print $col}
' \
  | sort -n \
  | awk '{a[NR]=$1} END {
      if (NR==0) {print "no sample"; exit 1}
      p50=int((NR-1)*0.5)+1; p95=int((NR-1)*0.95)+1
      printf "samples=%d  p50=%.2f  p95=%.2f  max=%.2f\n",
             NR, a[p50], a[p95], a[NR]
    }'
```

脚本说明：无变量拼接路径、无删除动作、只读采集，可安全在生产执行。若要扩展为批量多机版本，必须加执行日志（`exec >> /var/log/io_probe.log 2>&1`）和结果汇总。

### 9.1 观测工具选型对照

| 层次 | 首选工具 | 备用 | 说明 |
| --- | --- | --- | --- |
| 快照概览 | top / vmstat / iostat | dstat（已不活跃，建议用 pcp-dstat） | 人人都会，但要看对列 |
| 历史回溯 | sar（sysstat） | node_exporter + Prometheus | 没装 = 事后失明 |
| 进程归因 | pidstat | top -H、htop | pidstat -d/-r/-w 三件套 |
| 函数热点 | perf top / perf record | bpftrace | 需要匹配的内核与符号表 |
| 内核行为 | bcc/bpftrace 工具集 | systemtap（老环境） | 见 9.2 |
| 盘健康 | smartctl、storcli | 云监控 | 硬件层证据 |
| 压测验证 | fio | ioping、hdparm -t（仅读带宽粗测） | 对比必须有同口径 |

### 9.2 eBPF / bcc 进阶观测工具

内核 4.x/5.x 环境（CentOS 7 的 3.10 大部分 bcc 工具不可用，需升级内核或使用 RedHat 支持子集）：

```bash
# 安装：yum install bcc-tools 或 apt install bpfcc-tools（包名视发行版）
biosnoop                 # 逐条 IO 事件：进程、设备、延迟（含排队）
biolatency               # IO 延迟直方图，替代"平均值陷阱"
opensnoop                # 谁在频繁 open（排查海量小文件读打开）
offcputime -p <pid> 30   # 线程被调度出去后在等什么（非 CPU 延迟）
runqlat                  # 进程等待 CPU 的直方图，CPU 饱和标准证据
tcpretrans               # 实时 TCP 重传及来源进程
```

使用原则：这些工具观察开销小（通常 < 3%），但 `offcputime`/`perf record` 带栈采集开销会到 10% 量级，高峰期使用要评估。bcc 工具输出的是采样事件流，需要跑至少 30 秒才有统计意义。

## 10. 配置示例

### 10.1 按负载验证的 sysctl 配置示例

文件：`/etc/sysctl.d/99-perf-tuning.conf`（不要直接大改 `/etc/sysctl.conf`，独立文件便于回滚——删掉即恢复）。

```ini
# ---------- 内存 ----------
# 数据库/缓存类机器降低换页倾向
vm.swappiness = 1
# 大内存机器控制脏页，避免集中回写抖动
vm.dirty_background_ratio = 5
vm.dirty_ratio = 10
# 允许内存超 commit 的保守策略：0=启发式（默认），生产数据库保持默认或 1 之前先确认监控到位
vm.overcommit_memory = 0

# ---------- IO ----------
# 脏页达到可回写年龄的阈值（单位百分之一秒），3000 对应 30 秒；与 5 秒的 `vm.dirty_writeback_centisecs` 唤醒周期不同
vm.dirty_expire_centisecs = 3000

# ---------- 网络 ----------
# 半连接队列，DDoS 或高峰新建连接失败时调（配合 ss -s 观察）
net.ipv4.tcp_max_syn_backlog = 8192
net.core.somaxconn = 8192
# conntrack 表容量（防火墙/NAT 开启时生效）
net.netfilter.nf_conntrack_max = 262144
```

生效方式：`sysctl --system`（加载所有 /etc/sysctl.d 文件）或 `sysctl -p /etc/sysctl.d/99-perf-tuning.conf`。验证方式：`sysctl vm.swappiness net.core.somaxconn` 逐项确认输出与预期一致。

### 10.2 tuned 统一管理（推荐）

`tuned` 是 RHEL/CentOS 系推荐的性能配置管理工具，好处是 profile 可切换、可回滚：

```bash
yum install -y tuned   # 或 apt install tuned
systemctl enable --now tuned
tuned-adm list
tuned-adm active
# 数据库场景常用 throughput-performance / virtual-guest 等
tuned-adm profile throughput-performance
# 观察切换前后对比
tuned-adm verify
```

回滚方式：`tuned-adm profile <原profile>` 一条命令回退，这是它优于手改散参数的原因。

### 10.3 IO 调度器 udev 持久化

文件：`/etc/udev/rules.d/60-iosched.rules`

```
# 所有 SSD（rotational=0）用 mq-deadline；NVMe 用 none
ACTION=="add|change", KERNEL=="sd[a-z]", ATTR{queue/rotational}=="0", ATTR{queue/scheduler}="mq-deadline"
ACTION=="add|change", KERNEL=="nvme[0-9]n[0-9]", ATTR{queue/scheduler}="none"
```

生效方式：udev 规则在设备下次加入时生效；对新挂载盘立即生效可配合 `udevadm trigger`。验证：`cat /sys/block/nvme0n1/queue/scheduler` 看方括号项。风险提醒：规则匹配条件写错会影响所有匹配设备（比如通配符把系统盘也改了），生产用前在测试机 `udevadm test` 演练。

### 10.4 ulimits 示例

文件：`/etc/security/limits.d/20-nofile.conf`

```
*    soft nofile 65535
*    hard nofile 65535
root soft nofile 65535
root hard nofile 65535
```

生效说明：对通过 PAM 登录的会话生效；systemd 托管的服务需要在 unit 里显式设 `LimitNOFILE=65535`（这是最常见的踩坑点：改了 limits.conf 发现服务没生效）。验证：`cat /proc/<pid>/limits` 看进程实际值。

## 11. 日志与指标观察方法

### 11.1 无监控时：sar 历史回溯

`sar` 由 sysstat 提供，默认每小时（或每分钟，视 `sysstat` 配置）采样存盘，是事后取证的最后一道防线：

```bash
# 确认采集是否开启（否则出事后什么都查不到，这是上线检查项）
systemctl status sysstat
cat /etc/sysconfig/sysstat    # RHEL: HISTORY=SADAY
# 回溯某天 CPU：sa23 = 23 号
sar -u -f /var/log/sa/sa23
# 回溯 IO：找 await 尖刺出现的时间点，和变更记录对时间线
sar -d -f /var/log/sa/sa23 | sort -k6 -n | tail -20
```

调高采样频率（默认 10 分钟粒度太粗，建议改为 1 分钟）：修改 `/etc/sysconfig/sysstat` 的 `SA_INTERVAL=1`，RHEL8/CentOS8+ 及 Debian 系用 systemd timer：`systemctl edit sysstat-collect.timer` 设 `OnCalendar=*:0/1`。注意对 `/var` 空间的评估：1 分钟粒度每天约几十 MB，属可接受范围。

### 11.2 有监控时：Prometheus 关键指标

以下指标名为 node_exporter 常见口径，**不同版本 exporter 指标名和采集开关可能略有差异，以实际部署版本为准**（如磁盘相关指标需在启动参数开启 `--collector.diskstats`，部分指标需配合 `--collector=...` 调整）。

| 关注点 | 指标 | 异常判断 |
| --- | --- | --- |
| CPU 饱和 | node_load1 / count(node_cpu_seconds_total{mode="idle"}) by (instance) | 持续 > 1 关注，结合 us/wa 结构 |
| IO 等待 | rate(node_cpu_seconds_total{mode="iowait"}[5m]) | > 0.2 排查 |
| 内存告急 | node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes | < 0.1 告警 |
| swap 活跃 | rate(node_vmstat_pswpin[5m]) | 持续 > 0 异常 |
| 直接回收 | rate(node_vmstat_allocstall_normal[5m]) | > 0 即延迟毛刺嫌疑 |
| 磁盘延迟 | rate(node_disk_read_time_seconds_total[1m]) / rate(node_disk_reads_completed_total[1m]) | 超过介质参考线 |
| 队列深度 | node_disk_io_time_weighted_seconds_total 的增量 | 持续增长代表堆积 |
| conntrack | node_nf_conntrack_entries / node_nf_conntrack_entries_limit | > 0.8 告警 |

**阈值必须用自家业务基线校准**，上表数值是常见起点而非绝对标准：OLTP 数据库机器 `iowait 0.2` 可能是灾难，日志服务器同样的值可能毫无问题。上线新告警前先用一周的 P99 数据画出分布。

常用 PromQL 直接抄（同样以实际 exporter 版本指标名为准，改前先 `curl http://<node>:9100/metrics | grep node_disk_read` 核对）：

```promql
# 单机 load / 核数（饱和度）
node_load1 / count(node_cpu_seconds_total{mode="idle"}) by (instance)

# IO 等待占比（5 分钟窗口）
rate(node_cpu_seconds_total{mode="iowait"}[5m])

# 每设备读延迟均值（ms），按设备聚合
1000 * rate(node_disk_read_time_seconds_total[1m])
       / rate(node_disk_reads_completed_total[1m])

# 内存可用率
node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes

# swap 换入速率（页/秒）
rate(node_vmstat_pswpin[5m])

# conntrack 使用率
node_nf_conntrack_entries / node_nf_conntrack_entries_limit

# 磁盘使用率与 inode 使用率（两个都要告警）
1 - node_filesystem_avail_bytes{fstype!~"tmpfs|overlay"} / node_filesystem_size_bytes{fstype!~"tmpfs|overlay"}
1 - node_filesystem_files_free{fstype!~"tmpfs|overlay"} / node_filesystem_files{fstype!~"tmpfs|overlay"}
```

告警规则设计建议：单指标异常只算"关注"，持续 N 分钟 + 组合条件才升级。例如"IO 等待高"应写成 `iowait > 0.2 AND (读延迟 > 阈值 OR load/核数 > 1)` 持续 5 分钟，避免备份窗口期天天误报。告警通知里把 `instance`、`device` 标签带上，值班人不用二次查询。

### 11.3 观察节奏

- 变更前：记录"变更前基线截图/数据"（uptime、iostat、业务 P99），保存为文件。
- 变更中：盯被改动的子系统 + 业务错误率。
- 变更后 24 小时内：覆盖至少一个业务高峰，重点看 P99 而不是平均值。

## 12. 排查路径（决策树）

从"业务变慢"出发的完整决策树，可直接打印贴在工位：

```
业务变慢
│
├─ 1. uptime：load 相对核数如何？
│   ├─ load 高 + R 状态多 ──────────────► CPU 型，走分支 A
│   ├─ load 高 + D 状态多 ──────────────► IO 阻塞，走分支 B
│   └─ load 正常但仍慢 ─────────────────► 应用内部/下游/网络，走分支 C
│
├─ 分支 A：CPU
│   ├─ mpstat：哪段高？
│   │   ├─ %usr 高 → pidstat 找进程 → perf top → 函数热点
│   │   │            ├─ 业务函数：应用优化/扩容
│   │   │            └─ 内核函数：系统调用风暴/锁竞争
│   │   ├─ %sys 高 → 同上，重点 strace -c 统计系统调用
│   │   ├─ %si 高  → softnet_stat/ethtool 中断与包量
│   │   └─ %st 高  → 云厂商/宿主机问题
│   └─ cs 高？→ pidstat -w 定位切换大户 → 线程池/锁
│
├─ 分支 B：IO
│   ├─ iostat -x：哪块盘？await 超参考线？
│   │   ├─ IOPS/带宽已打满 → iotop 找大户
│   │   │     ├─ 备份/日志类 → 错峰或限速
│   │   │     └─ 业务进程 → 访问模式变了（缓存/索引失效）
│   │   └─ 量不高但延迟高 → 盘劣化/RAID 降级/云盘限流
│   │         └─ smartctl / storcli / 云监控验证
│   └─ D 状态进程卡在哪？→ cat /proc/<pid>/wchan 或 /stack（需权限）
│
└─ 分支 C：非资源瓶颈
    ├─ sar -n DEV/TCP：网络层？
    ├─ 下游依赖慢：DB 慢日志、Redis 延迟、第三方 API
    ├─ 锁/队列：应用自身线程池排队（jstack/线程 dump）
    └─ GC：JVM GC 日志、内存回收风暴（sar -B）
```

每个分支的结论都必须有证据：说"是磁盘问题"，要能拿出 `iostat` 里该设备 `await` 超标 + `iotop` 进程归属 + 时间线与业务曲线吻合三件证据。单一指标不下结论。

### 12.1 常见误判清单

一线排查里反复出现的坑，逐条对照自查：

1. 把 buff/cache 当泄漏。free 里 buff/cache 大而 available 充足，是正常现象；应结合进程 RSS、cgroup 水位及 reclaim/PSI 进一步定位。
2. 用 `free` 判断内存水位。同时看 available、si/so、PSI 和 cgroup 限额。
3. SSD 上看 `%util` 下结论。%util 只说明"设备没闲过"，SSD/NVMe 可以并行处理，%util 100% 可能延迟仍然健康；必须看 await 和队列深度。
4. 只看平均值。延迟用 P99/P999，await 和监控面板的平均值会把 1% 请求的 500ms 毛刺平均没。
5. `iostat` 第一行当实时值。iostat -x 1 的第一段输出是自开机以来的累计均值，必须看第二段起（或加 -y 省略历史段）。
6. load 高就重启。D 状态堆积时重启只是掩盖，还可能把未落盘数据搅进更复杂的恢复局面；先查 D 进程卡在哪个内核函数。
7. 短测 SSD 下结论。SLC 缓存没耗尽前，全盘测速软件的数据都偏乐观 3~10 倍；评估稳态跑 30 分钟以上（见 7.7）。
8. 同时改多个参数。效果无法归因，回滚无法定位；一次一个，或每次改动有独立可回滚的配置文件。
9. 拿网上脚本直接跑 `sysctl`。内核版本、业务模型、发行版差异都很大，逐项理解再落地（tcp_tw_recycle 这类已被移除的参数还会直接报错）。
10. 高峰期跑 fio/压测。压测流量与业务流量叠加，压测本身变成故障源。
11. 忽略时钟不一致。跨机做时间线关联前不校时，根因分析方向整个跑偏。
12. 只看机器不看依赖。机器指标全绿但业务慢，瓶颈在下游（DB、Redis、第三方接口）的概率远大于"机器还有隐藏问题"。

### 12.2 一页速查卡

值班时可以直接抄在纸上的一页表：症状 → 确认命令 → 判断 → 下一步。

| 症状 | 确认命令 | 判断标准 | 下一步动作 |
| --- | --- | --- | --- |
| load 高 | ps -eLo stat \| grep -c '^D' | D 多 → IO 型 | iostat -x 1 找慢盘 |
| load 高 | mpstat -P ALL 1 | 单核满 → 单线程/中断 | 应用多实例 / ethtool -l 查队列 |
| CPU %usr 高 | pidstat -u 1 → perf top | 业务函数热点 | 交应用侧，给火焰图 |
| CPU %sys 高 | perf top / strace -c -p <pid> | 系统调用分布 | 按调用类型定位（IO/内存/锁） |
| %si 高 | cat /proc/net/softnet_stat | 第 2/3 列增长 | 开网卡多队列、irqbalance |
| %st 高 | vmstat 1 st 列 | 持续 > 5% | 找云厂商/换规格 |
| 内存告警 | free -h | available 低 | smem/smaps_rollup 归因进程 |
| 业务卡顿 | vmstat 1 si/so 列 | 持续非零 | 调 swappiness / 扩内存 / 关 swap（评估后） |
| 延迟毛刺 | /proc/meminfo Dirty | 周期性尖峰 | 脏页参数 + 日志异步化 |
| OOM | dmesg -T \| grep -i oom | 看打分表 | 限制元凶内存而非调 oom_adj |
| 磁盘慢 | iostat -x 1 | await 超参考线 | 先分"量大"还是"盘慢"（7.3） |
| 磁盘慢 | iostat -x 1 | 低 IOPS 高延迟 | smartctl / 云监控限流 |
| IO 大户 | iotop -oP | 进程归属 | 限速/错峰/修访问模式 |
| 写入突然慢 10 倍 | RAID 卡工具查 BBU | write-back 降级 | 换电池/临时调回策略需评估掉电风险 |
| 连不上/超时 | ss -s、dmesg \| grep conntrack | 表满/半连接满 | 调 conntrack/backlog 参数 |
| 空间告警 | df -h; df -i | inode 也算空间 | 清小文件，不是删大文件 |
| 压测对比 | 同参数 fio 两次 | 分位数对比 | 差异超 20% 深挖 |
| 一切正常但慢 | sar -n DEV 1、下游慢日志 | 机器全绿 | 转应用/依赖排查 |

## 13. 高风险操作清单与风险提醒

本文涉及的操作按风险分级：

**低风险（可日常执行）**

- nice/renice 正值压制批处理；只读观测命令（top/iostat/sar/perf top，perf record 本身有 < 5% 开销）
- tuned-adm profile 切换（一条命令可回退）

**中风险（低峰窗口 + 备份后执行）**

- sysctl -w 修改内核参数：错误值可能导致无法新建连接、内存分配异常。执行前备份原值，准备回滚命令。
- 切换 IO 调度器 / 改 read_ahead_kb：对在线存储瞬时可能抖动。
- swapoff -a：见 6.3 节三条件检查，且注意它是危险方向开关——关掉后内存耗尽的后果从变慢变成杀进程。
- systemctl set-property 资源限制：配额设得过小会把服务限死。
- ethtool -G/-L 改环形缓冲/队列：执行瞬间可能短暂丢包。

**高风险（变更审批 + 回滚方案 + 测试机演练）**

- 修改 grub 参数（THP、CPU 隔离等）：改错可能无法开机。操作前备份 /etc/default/grub 并确认带外（VNC/IPMI）可进 rescue。
- 重启服务/重启机器清缓存类操作：生产高峰期禁止。
- mount -o remount 改挂载选项：方向写反（ro）直接故障。
- 批量脚本多机下发内核参数：必须先单机灰度 + 监控观察期。

通用原则：**改系统参数永远有"原值记录"和"恢复命令"两张纸**。

### 13.1 性能变更工单模板（可直接抄）

```
【变更标题】MySQL 主库脏页回写参数调整（vm.dirty_background_bytes）
【变更原因】业务高峰 w_await P99 90ms，Dirty 峰值 3.2G，周期性回写风暴（附 iostat/sar 截图与时间点）
【影响范围】db-prod-01/02/03 三台，角色：订单库主从；涉及业务：订单下单、支付回调
【变更前基线】collect_baseline.sh 输出存档路径：/bak/baseline/db-prod-01_20260920_1400/
【变更内容】新增 /etc/sysctl.d/99-dirty.conf，内容见附件；不修改既有文件
【实施步骤】1) db-prod-03 从库灰度 → 2) sysctl --system 加载 → 3) 验证命令清单（见下）→ 4) 观察 24h 覆盖一个高峰 → 5) 同法推 01/02（主库放最后，备库切换窗口执行）
【验证命令】sysctl vm.dirty_background_bytes；iostat -x 1 60 对比 w_await；业务 P99 监控面板链接
【回滚方案】rm /etc/sysctl.d/99-dirty.conf && sysctl --system（原值为默认值，删除文件即恢复）
【执行窗口】2026-09-22 14:00-16:00（低峰），值班人：张三；复核人：李四
【审批】DBA：____  运维负责人：____
```

要点：验证命令要写到"复制就能跑"的程度；回滚方案独立于实施步骤存在（回滚是出事时凭肌肉记忆执行的，不能依赖现场翻文档）；主库变更永远放在从库验证之后。

## 14. 验证方式

一次调优的完整验收由三层组成：

**第一层：参数生效验证（分钟级）**

```bash
sysctl vm.swappiness            # 确认值已加载
cat /sys/block/sda/queue/scheduler
systemctl show tuned -p ActiveState
cat /proc/<pid>/limits          # ulimit 类
```

**第二层：系统指标验证（小时级）**

对比变更前后同类命令的输出，必须同口径、同负载时段：

```bash
# 变更前、变更后各跑一次，保存对比
iostat -x 1 60 > /tmp/io_before.txt
# （变更后）
iostat -x 1 60 > /tmp/io_after.txt
diff <(awk '/sda/{print $NF}' /tmp/io_before.txt) \
     <(awk '/sda/{print $NF}' /tmp/io_after.txt)   # 仅示例思路，实际按列名核对后写对比脚本
```

**第三层：业务指标验证（覆盖一个完整高峰）**

- 接口 P99/P95 延迟是否达到目标值
- 错误率、超时率无上升
- 吞吐无下降（性能类改动有时会牺牲吞吐）
- 脏页参数改动：观察 w_await 尖刺频率是否消失

验收标准要在动手前写进工单，例如：“MySQL 实例所在盘 w_await P99 从 90ms 降至 15ms 以下，业务下单接口 P99 从 600ms 降至 150ms 以下，错误率不高于基线”。

## 15. 回滚方案

按改动类型给出标准回滚路径：

| 改动类型 | 回滚方式 | 生效速度 |
| --- | --- | --- |
| sysctl -w 临时值 | sysctl -w 参数=原值 | 即时 |
| /etc/sysctl.d/99-*.conf | 删除该文件后 sysctl --system（重启后彻底恢复） | 即时+重启 |
| tuned profile | tuned-adm profile <原profile> | 秒级 |
| IO 调度器 | echo 原调度器 > /sys/block/X/queue/scheduler | 即时 |
| THP 临时值 | echo always > /sys/kernel/mm/transparent_hugepage/enabled | 即时 |
| grub 参数 | 恢复 /etc/default/grub 备份 → grub2-mkconfig → 重启窗口 | 需重启 |
| 挂载选项 | 恢复 fstab 备份 → mount -o remount 或下次挂载 | 视选项 |
| cgroup/ulimit | 恢复原 systemctl set-property 值/原 limits 文件 | 下次启动服务 |
| 绑核 | taskset 恢复全核范围 | 即时 |

回滚预案写在变更记录里，和变更本身同等重要。如果一次变更涉及多台机器，回滚要支持"一键执行"——即回滚本身也做成脚本，而不是让值班同事现场手敲。

## 16. 生产环境注意事项

1. 上线前基线采集：新机器上线前把 mpstat -P ALL 1、iostat -x 1、sysctl -a 输出存档，日后对比有据可依。
2. 先应用后系统：80% 的性能问题根因在应用层（烂 SQL、缓存击穿、线程池乱配）。只调内核参数通常是止痛药，不治病。
3. 灰度节奏：参数类变更按 单机 → 10% → 全量 推进，每阶段观察至少覆盖一个业务高峰。
4. 监控先行：没有 sar 或 node_exporter 的机器等于裸奔，先补采集再谈优化——否则你永远在"猜"。
5. 别信网上的通用调优脚本：网上流传的 sysctl 优化三十项 多数照抄自陈旧场景（如随意开 tcp_tw_recycle——该参数在 4.12 内核已被移除，NAT 环境下会引发丢包，新版本根本不存在），逐项理解再落地。
6. 云环境先查限流：本地看不到云盘的 IOPS 限流计数，遇到 IO 慢先对云监控的"读写 IOPS/吞吐/BPS 触顶"指标再查本地。
7. 时钟与时间线：多机对比日志前先 chronyc sources 确认时钟一致，跨机时间线对不齐会让根因分析全盘皆错。
8. 文档化：每次参数变更登记"改了什么、为什么、谁批的、怎么回滚"，三个月后的你会感谢现在的你。

### 16.1 新机器上线性能基线检查单

上线前逐项打勾，避免带着隐患进生产——本文多个案例的根因（THP 漂移、调度器未设、无采集）都属于这张表：

| 检查项 | 命令 | 期望值（按业务角色调整） |
| --- | --- | --- |
| THP 设置 | cat /sys/kernel/mm/transparent_hugepage/enabled | DB/缓存类为 never 或 madvise |
| IO 调度器 | cat /sys/block/*/queue/scheduler | NVMe=none，SATA SSD=mq-deadline |
| swappiness | sysctl vm.swappiness | DB/缓存类 ≤ 10 |
| swap 配置 | swapon --show | 与角色策略一致（关或有告警配套） |
| 文件描述符 | cat /proc/1/limits + 服务进程 limits | 服务实际值 ≥ 65535 |
| sysstat 采集 | systemctl is-active sysstat | active，采样 ≤ 1 分钟 |
| node_exporter | curl -s localhost:9100/metrics \| head -3 | 有数据、版本已知 |
| 时钟同步 | chronyc sources | 偏差 < 50ms |
| CPU governor | cpupower frequency-info | 延迟敏感机 performance |
| irqbalance | systemctl is-active irqbalance | 按规范开启或已手动绑核 |
| 盘健康 | smartctl -a /dev/sda \| head -15 | SMART 状态 PASSED，新盘 0 重映射 |
| 挂载参数 | mount \| grep data | 与 fstab 一致（防漂移） |
| 内核参数基线 | sysctl -a > /tmp/sysctl_baseline.txt | 已归档，供 diff |
| conntrack 上限 | sysctl net.netfilter.nf_conntrack_max | 网关/NAT 机显式设置过 |

这张表的价值在"一致性"：集群所有同角色机器的期望值应当完全一致，任何一台的偏差都是未来"随机单点故障"的候选根因。把表里的命令做成 Ansible 校验任务或巡检脚本，每天跑一次比靠人记靠谱。

## 17. 五个完整复盘案例

### 案例一：load 150，CPU 却是 idle 的

- 现象：监控 load1 148（8 核），%usr 8%，接口偶发超时。
- 初步判断：ps -eLo stat | grep -c '^D' = 142，D 状态堆积，IO 型 load。
- 命令检查：iostat -x 显示 sdb w_await 210ms；ps aux | awk '$8~/^D/' 全是 java 线程。
- 关键指标：D 状态进程的 /proc/<pid>/wchan 均指向文件系统锁；mount 显示该盘是 NFSv3 挂载。
- 根因定位：NFS 服务端存储抖动，客户端进程全部卡在不可中断 IO，推高 load。
- 修复方案：应急把该挂载点上的临时目录切本地盘（fstab 改路径 + 应用配置灰度切换）；推动 NFS 服务端排查存储阵列。
- 验证结果：切走 10 分钟后 load 回落至 6，wchan 无 NFS 等待。
- 回滚预案：配置回切原挂载路径，NFS 服务端问题修复后择机回迁。
- 复盘：load 报警必须区分 R/D 构成，NFS 类挂载必须有独立的延迟告警（对挂载点做 stat 超时检测）。

### 案例二：规律性每 5 秒卡一下

- 现象：订单接口 P99 每隔固定几分钟出现毛刺。
- 初步判断：毛刺有周期性，像定时任务或内核周期行为。
- 命令检查：毛刺时刻 iostat 出现 200MB/s 突发写；sar -d -f 回溯同样规律。
- 关键指标：/proc/meminfo 的 Dirty 峰值 3.2GB（机器 128G，dirty_ratio=10 即 12.8GB 前理论不会阻塞——但 dirty_expire_centisecs 过期集中回写同样会造成突发）。
- 根因定位：应用顺序写大日志 + 过期脏页集中回写叠加，形成突发刷盘风暴。
- 修复方案：vm.dirty_background_bytes=256M + vm.dirty_bytes=1G 让后台持续小步回写；同时日志切异步并限制大小。
- 验证结果：次周高峰 w_await 毛刺从 85ms 降至 6ms，接口 P99 恢复。
- 回滚预案：删除 /etc/sysctl.d/99-dirty.conf + sysctl --system。
- 复盘：周期性毛刺优先怀疑定时任务与内核周期回写；Dirty 曲线要纳入监控。

### 案例三：JVM 频繁 Full GC 的幕后黑手是 THP

- 现象：Java 服务每周随机一台出现长 GC，间隔无规律。
- 初步判断：GC 日志显示 STW 时间异常，但堆使用正常。
- 命令检查：问题机 cat /sys/kernel/mm/transparent_hugepage/enabled 为 always，同集群其他机器是 never（装机模板漂移）。
- 关键指标：/proc/vmstat 的 compact_stall 在 GC 时间点突增。
- 根因定位：THP 的内存碎片整理在 JVM 分配大段内存时同步触发 compaction，STW 被拉长。
- 修复方案：按 6.5 节统一关闭 THP，并把该项纳入上线检查基线（Ansible 校验任务）。
- 验证结果：修复后三个月无同类 GC。
- 回滚预案：恢复 madvise 模式即可，无需重启。
- 复盘：集群机器配置漂移是"随机单点故障"的第一嫌疑人，核心参数要有集群一致性巡检。

### 案例四：网关机器 %si 打满单核

- 现象：Nginx 网关 8 核机器，top 里某一行 %CPU 300+，其中大半是 si；业务 QPS 无明显增长但 P99 抖动。
- 初步判断：软中断集中在单核（top 按 1 展开看每行 CPU，si 集中在 CPU0）。
- 命令检查：sar -n DEV 1 显示 rxpck/s 120 万但 rxkB/s 很小——小包量暴涨；cat /proc/net/softnet_stat 第二列（ backlog 轮次）持续增长；cat /proc/interrupts | grep eth0 确认中断全落在 CPU0。
- 关键指标：单核 si > 90%，time_squeeze（第 3 列）增长。
- 根因定位：上游新上的服务用短连接高频调用，网卡多队列未开启，单核处理全部收包。
- 修复方案：ethtool -l eth0 确认支持 8 队列后，变更窗口内 ethtool -L eth0 combined 8；同时确认 irqbalance 开启（systemctl status irqbalance），未开启的环境手动核对 /proc/irq/<n>/smp_affinity 分发。中期推动上游改长连接。
- 验证结果：每核 si 降至 15% 以下，softnet_stat 不再增长，P99 恢复。
- 回滚预案：ethtool -L eth0 combined 1 即回原状态；执行瞬间有微秒级收发中断重建，可能丢极少量包，所以放在低峰窗口。
- 复盘：小包量与带宽是两个独立维度，监控里只有带宽没有 pps 是盲区，补充 rxpck/s 指标。

### 案例五：云盘 IOPS 触顶限流

- 现象：某业务每天 02:00~02:30 批量任务变慢，iostat 显示云盘 w_await 从 2ms 升至 40ms，但 %util 只有 60%。
- 初步判断：量没打满、延迟却高，本地看不出硬件原因，怀疑云侧。
- 命令检查：任务期间云监控显示该盘 IOPS 曲线正好压在规格上限（如 4K 随机写 1 万 IOPS 触顶），云厂商控制台有"触发性能上限"事件记录。
- 关键指标：盘 IOPS = 规格上限值（平顶曲线），与 iostat 的 w/s 对得上。
- 根因定位：云盘按规格限流，本地一切正常是"被限流后的表象"。
- 修复方案：短期把批量任务的并发度调低（把 IOPS 压到规格的 80% 以内换延迟稳定）；中期提升盘规格或拆盘分摊。
- 验证结果：并发调低后 w_await 回到 3ms，任务总时长反而缩短 18%（限流排队消失，效率更高）。
- 回滚预案：脚本参数回退，git revert。
- 复盘：云主机 IO 慢的排查清单必须加一步"看云监控限流"；iostat 看不到"被限流"这个事实本身。低并发快速失败优于高并发排队，是限流环境的通用策略。

## 18. 常见问题 FAQ

**Q1：线上能不能直接 echo 3 > /proc/sys/vm/drop_caches “释放内存”？**不能当成常规操作。它不影响数据正确性（只丢干净缓存），但会把后续读全部打到磁盘，IO 型负载下制造一次小型故障。它唯一合理的用途是压测前清环境，或确认"高内存是缓存造成"的验证动作，低峰执行并提前评估 IO 余量。

**Q2：JVM 机器要不要关 swap？**建议"低使用"而不是"一刀切关"。`vm.swappiness=1` 让内核基本不主动换页；完全 `swapoff` 后内存耗尽的后果是 OOM 直接杀进程。如果必须关，先确认：内存监控有分级告警、应用有内存上限（-Xmx/cgroup）、且团队接受"没有缓冲垫"的行为。

**Q3：数据库服务器 load 长期 2~3（32 核机器），要不要处理？**先看构成。`ps -eLo stat | grep -c '^D'` 接近 0、`iostat` 正常、业务 P99 正常，这个 load 完全不用动。load 的绝对值没有意义，相对核数的水位加业务延迟才有意义。

**Q4：iostat 的 %util 到 100% 是不是就该加盘了？**先回答三个问题：介质是机械盘还是 SSD？`await` 超没超该介质的参考线？队列深度是否持续高位？机械盘 + 高 await + 高队列 → 加盘/换 SSD；NVMe `%util` 100% 但 await 0.1ms → 盘没问题，去看是不是应用并发本来就该限流。

**Q5：怎么证明一次调优"有效"而不是"正好那阵子业务量小"？**同口径对比：同一天同类高峰时段、同样采集命令、同样时长；更严谨的做法是灰度——同角色机器一半改一半不改（A/B），两组同时段对比。平均值不行的话就比分位数。没有对照组时，至少用 sar 历史数据对时间线。

**Q6：sar 和 node_exporter 只留一个选哪个？**选"都留"，如果被迫二选一选 node_exporter（有告警能力）。但要清楚各自边界：Prometheus 默认 15~30s 采样 + 保留期有限，`sar` 是本机 1 分钟粒度存盘，事后回溯一两周前的某天 14:23 发生了什么，sar 更可靠。告警靠 Prometheus，取证靠 sar，两者不冲突。

**Q8：pidstat、iotop、sar 都不存在，机器很老怎么办？**前两个属于 sysstat 包（CentOS：`yum install sysstat`；Ubuntu：`apt install sysstat`），内网离线环境用发行版 ISO 做本地源安装。临时顶替方案：`top`、`cat /proc/<pid>/io`、`iostat` 的替代品是直接读 `/proc/diskstats`（两次采样做差除以间隔即每秒速率）、`vmstat` 属于 procps 一般自带。再老的系统，`/proc` 文件系统永远是最可靠的兜底数据源。

**Q9：容器/K8s 节点上做系统调优有什么额外注意？**节点级 sysctl 对容器内进程同样生效（内核参数是全局的），但 `ulimit`、cgroup 限额要在 Pod 的 resources 与容器运行时层面配；节点上 `swapoff` 是 kubelet 的默认要求之一。另外容器里看到的 CPU 数可能与节点不同（CPU limits 生效时），`top` 数据要拿节点视角复核。

## 19. 总结

把全文压缩成一套可执行的肌肉记忆：

1. 三板斧定位方向：uptime（load 构成）、vmstat 1（CPU/内存/切换）、iostat -x 1（IO）。
2. 分系统下钻：CPU 看 us/sy/wa/si/st 五种分解；内存只看 available 和 si/so/allocstall；IO 看 await 与 IOPS 的组合；网络看错误丢弃和 conntrack。
3. 结论必须有三件套：指标异常 + 进程归属 + 时间线吻合，缺一不可。
4. 调优顺序：应用层 → 架构层 → 系统参数 → 硬件扩容；系统参数里优先 tuned/文件化的可回滚方式。
5. 变更纪律：一次一个参数、先备份原值、单机灰度、覆盖高峰验证、回滚命令写在工单里。
6. 两个必做项：所有机器部署 sar 或 node_exporter（事后取证的生命线）；集群核心内核参数做一致性巡检（防配置漂移）。

性能工作没有玄学，只有"基线 → 观测 → 证据 → 假设 → 最小改动 → 验证"的循环。把这套循环跑熟，绝大多数性能问题在三板斧阶段就已经有了方向。

最后给一组可以直接放进每日巡检的"五连命令"，每条 5 秒、总共半分钟，能覆盖本文全部四个子系统的最差情况预警：

```bash
uptime                                   # load 趋势与核数的比值
vmstat 1 3 | tail -1                     # si/so（换页）、cs（切换）、wa（IO 等待）一屏看全
iostat -x 1 3 | awk 'NR>2 && $NF+0>0'    # 有活动的设备；结合 awk 核对 await 列（列位以表头为准）
free -h | awk 'NR==2{print "available:",$7}'
dmesg -T | tail -5                       # 最近内核事件：OOM/IO error/链路抖动
```

跑完这五步，把负载、si/so、wa、available 趋势和内核错误与本机历史基线及业务延迟对照；不存在适合所有主机的统一 load/核数或 wa 正常阈值。

调优的终点不是"参数调得越多越好"，而是**建立属于你这套环境的性能基线，并让每一次偏离都能被解释**。基线在，系统就永远不会"莫名其妙"；这比任何单条参数值都值钱。

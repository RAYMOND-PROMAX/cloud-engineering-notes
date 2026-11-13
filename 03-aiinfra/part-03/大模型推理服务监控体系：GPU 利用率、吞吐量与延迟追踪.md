# 大模型推理服务监控体系：GPU 利用率、吞吐量与延迟追踪

## 一、问题背景：为什么 nvidia-smi 看着 98% 服务还是慢

先把一个真实场景摆出来。某公司 8 卡 H100 节点跑 vLLM，对外提供 Qwen 系列模型的 OpenAI 兼容接口，网关侧监控看到 P99 首字延迟从 800ms 涨到 6 秒，用户投诉"打字机效果变成了一卡一卡"。运维第一反应是登机器敲 `nvidia-smi`：

```bash
$ nvidia-smi
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 570.124.06             Driver Version: 570.124.06   CUDA Version: 12.8      |
|-----------------------------------------+------------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id          Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |           Memory-Usage | GPU-Util  Compute M. |
|                                         |                      |               MIG M. |
|=========================================+========================+======================|
|   0  NVIDIA H100 80GB HBM3          On  |   00000000:19:00.0 Off |                    0 |
| N/A   61C    P0            412W / 700W |   78321MiB / 81559MiB |    100%      Default |
|                                         |                        |             Disabled |
+-----------------------------------------+------------------------+----------------------+
```

GPU-Util 100%，显存 78GB/80GB，看着"跑得很满"。于是结论变成了"GPU 打满了，加卡吧"。

这个结论大概率是错的。`GPU-Util` 在 nvidia-smi 里的定义是"在过去采样周期内，至少有一个 kernel 在 GPU 上执行的**时间占比**"，它不反映这个 kernel 干了多少有效计算。一个每 10 微秒启动一次、每次只做几个字节 memcpy 的空转循环，同样能把 GPU-Util 顶到 100%。对推理服务来说，最常见的高占用低效率状态有三种：

第一种，**KV Cache 打满导致的调度空转**。vLLM 的 block manager 分不到新的 block，调度器只能先抢占（preemption）部分请求、把它们的 KV 释放掉重新排队，然后同一批请求反复 recompute。GPU 一直在忙，但忙的是重复的 prefill，而不是有效的 decode。表现出来就是 `GPU-Util` 100%、`vllm:num_requests_waiting` 持续上涨、`e2e_request_latency_seconds` 恶化，但 `generation_tokens_total` 的增长速率反而下降。

第二种，**小 batch 上的内存带宽瓶颈**。decode 阶段每生成一个 token 都要把权重从 HBM 读一遍，batch 小的时候算术强度极低，GPU 的时间全花在等内存上。这时候 `DCGM_FI_PROF_PIPE_TENSOR_ACTIVE`（张量核心活跃度）可能只有 5%~15%，而 `DCGM_FI_DEV_GPU_UTIL` 是 90%+。两者的巨大差距就是"访存受限"的典型指纹。

第三种，**GPU 与 CPU 之间的同步热点**。采样、logits 后处理、streaming 输出回调里有同步操作，GPU 每个 step 都要等 CPU。表现是 GPU-Util 忽高忽低、利用率的频谱图上出现规律性锯齿。

所以"GPU 利用率"这个词在推理服务里必须拆成至少三个指标来看：**时间维度的占用率**（GPU_UTIL）、**计算单元的活跃度**（PIPE_TENSOR_ACTIVE / SM_ACTIVE）、**显存带宽的饱和度**（DRAM_ACTIVE）。只盯第一个，等于只看汽车的发动机转速表判断油耗。

这篇文章要解决的是：把 GPU 层、引擎层、请求层三层指标串成一条能直接回答"慢在哪"的观测链，给出可用的采集配置、PromQL、告警阈值推导方法，以及踩坑后会遇到的坑。

## 二、适用场景与环境假设

### 2.1 覆盖范围

- 单机多卡张量并行（TP）部署的在线推理服务，vLLM V1 引擎；
- Kubernetes 部署（GPU Operator 管理驱动与 exporter）与裸机 systemd 部署两种形态；
- 需要为推理服务建立 SLO（首字延迟、单 token 延迟、可用性），并把 SLO 拆到可观测指标上；
- 需要容量规划：给定 QPS 和输入输出长度分布，估算需要多少卡；
- 需要故障定位：慢请求到底是排队慢、prefill 慢、decode 慢，还是被抢占重算。

### 2.2 不覆盖的部分

- 训练任务的监控（gradient 爆炸、loss 曲线、NCCL 通信），虽然 DCGM 指标通用，但关注点完全不同；
- 多机多卡 PD 分离架构的跨节点链路监控，只在相关处提一句；
- 算子级 profiling（nsys、ncu），那是性能优化的下一层，本篇只在定位到"compute bound"之后给出入口。

### 2.3 环境与版本

| 组件 | 版本 | 说明 |
| --- | --- | --- |
| vLLM | v0.32.x / v0.33.0 | V1 引擎为默认；v0.33.0 于 2026-11-02 发布 |
| Python | 3.11 ~ 3.14 | v0.33.0 的 PyPI 元数据要求 >=3.11,<3.15 |
| CUDA | 12.8 / 12.9 | 对应驱动 R570 及以上 |
| GPU | H100 80GB / A100 80GB / L40S 48GB | 示例以 H100 为主 |
| DCGM Exporter | 4.x（GPU Operator 25.x 自带） | 指标前缀 DCGM_FI_* |
| Prometheus | 3.x（示例用 3.5+） | PromQL 与 2.x 兼容，但 --enable-feature 有变化 |
| Grafana | 11.x / 12.x | 面板 JSON 参考，不绑定版本 |
| Kubernetes | 1.30 ~ 1.33 | ServiceMonitor CRD 由 Prometheus Operator 提供 |

### 2.4 为什么要标版本

vLLM 的指标名称在小版本之间会变。V0 引擎时代用 `vllm:gpu_cache_usage_perc`，V1 引擎文档里写作 `vllm:kv_cache_usage_perc`；`num_requests_swapped` 在 V1 里因为 swap 抢占模式被移除而消失；v0.32.0 起新增了 `vllm:admission_rejections_total`、`vllm:watermark_requests`、`vllm:prompt_tokens_cached_by_source_total`、`vllm:num_requests_kv_fetch_by_stage` 四个指标，并支持 `--custom-histogram-buckets` 自定义直方图分桶。

对策很简单：**任何 dashboard 上线前，先 curl 一次真实的 /metrics，把指标名导出来对比**。这件事放在第 4.2 节给出具体做法。照抄网上的 dashboard JSON，十有八九面板是空的。

## 三、核心知识：三层指标与它们的关系

### 3.1 三层结构

推理服务的观测可以分成三层，每层回答不同的问题：

```text
┌──────────────────────────────────────────────────────────────┐
│  请求层（用户视角）                                           │
│  TTFT / TPOT / E2E latency / 成功率 / 输入输出 token 分布      │
│  回答：用户感受到了什么？                                     │
└──────────────────────────┬───────────────────────────────────┘
                           │ 请求层指标的恶化，由下面两层解释
┌──────────────────────────▼───────────────────────────────────┐
│  引擎层（调度视角）                                           │
│  running/waiting 队列 / KV cache 使用率 / 抢占 / 前缀缓存命中  │
│  token 吞吐（prompt、generation）                             │
│  回答：引擎在做什么？为什么排队？                             │
└──────────────────────────┬───────────────────────────────────┘
                           │ 引擎层的瓶颈，最终落到硬件上
┌──────────────────────────▼───────────────────────────────────┐
│  GPU 层（硬件视角）                                           │
│  GPU_UTIL / TENSOR_ACTIVE / SM_ACTIVE / DRAM_ACTIVE / 显存     │
│  温度 / 功耗 / 频率 / ECC / XID                                │
│  回答：硬件是忙还是闲？忙在算什么？                            │
└──────────────────────────────────────────────────────────────┘
```

排查的**顺序必须是自上而下**：先从请求层确认"确实慢了"，再从引擎层判断"慢在排队还是慢在执行"，最后用 GPU 层确认"执行阶段是被算力还是被带宽卡住"。反过来先看 GPU 利用率，容易被"虚假繁忙"带偏。

### 3.2 延迟指标的准确定义

这几个指标的定义必须精确，否则告警阈值没法定。

**TTFT（Time To First Token）**：从请求到达服务端到第一个输出 token 被生成的时间。它包括排队时间（queue time）+ prefill 时间 + 第一次采样时间。vLLM 的 `vllm:time_to_first_token_seconds` 是一个 Histogram，从请求进入调度器开始计时。注意：如果网关前面还有排队，这段不计入。

**TPOT（Time Per Output Token）**：vLLM 中 `vllm:request_time_per_output_token_seconds` 的定义是 `(e2e_latency - ttft) / (output_tokens - 1)`。分母减 1 是因为第一个 token 的时间已经算在 TTFT 里了。输出只有 1 个 token 的请求，该指标记为 0。这个定义在做容量估算时很关键：用户感知的"打字速度"就是 TPOT 的倒数。

**ITL（Inter-Token Latency）**：`vllm:inter_token_latency_seconds`，相邻两个流式 token 之间的间隔，每输出一个 token 记一个样本。TPOT 是"平均值"，ITL 是"逐次值"。用户抱怨"卡顿"通常对应 ITL 的 P99 抖动，而 TPOT 的均值可能看着正常。两者要一起看。

**E2E（端到端延迟）**：`vllm:e2e_request_latency_seconds`，请求进入调度器到最后一个 token 生成完毕。

这四者的关系可以这样验证：

```text
E2E ≈ Queue + Prefill + (OutputTokens - 1) × TPOT
```

其中 `Queue + Prefill` 约等于 TTFT。当 E2E 的恶化无法用这个等式解释时，说明有采样外的开销（比如输出被前端阻塞、HTTP 层慢客户端拖累），需要看 `http_request_duration_seconds` 做交叉验证。

### 3.3 吞吐量指标与"两种吞吐"

**请求吞吐（QPS/RPS）**：`rate(vllm:request_success_total[1m])`，按 `finished_reason` 区分 `stop`（正常结束）、`length`（撞到 max_tokens）、`abort`（客户端断开）。只有 `stop` 和 `length` 才代表服务真正完成了工作。

**Token 吞吐**：输入侧 `rate(vllm:prompt_tokens_total[1m])`，输出侧 `rate(vllm:generation_tokens_total[1m])`。这两个必须分开算，因为输入和输出的成本模型完全不同——prefill 是计算密集的，decode 是访存密集的。把两者相加成一个"总 token 吞吐"会掩盖问题：一个长 prompt 短回答的负载（RAG、摘要）和一个短 prompt 长回答的负载（创作、对话），总 token 吞吐相同，但 GPU 压力差好几倍。

**有效吞吐 vs 名义吞吐**：名义吞吐是上面算出来的数字；有效吞吐要乘上成功率，并且要考虑被 `abort` 的请求浪费的算力。一个 30% 请求被客户端中途取消的服务，名义吞吐再高，单位成本也是虚的。

### 3.4 GPU 指标的真实含义

| 指标 | 数据来源 | 含义 | 推理场景下的正常范围 |
| --- | --- | --- | --- |
| DCGM_FI_DEV_GPU_UTIL | DCGM / NVML | 采样周期内 GPU 有 kernel 在执行的时间占比 | decode 阶段常 80%~100%，但这个数本身没诊断价值 |
| DCGM_FI_PROF_PIPE_TENSOR_ACTIVE | DCGM（需 profiling 权限） | 张量核心 pipe 活跃周期占比 | prefill 阶段 30%~60%，decode 阶段常低于 20% |
| DCGM_FI_PROF_SM_ACTIVE | DCGM | 至少一个 warp 在 SM 上驻留的时间占比 | 通常接近 GPU_UTIL，差值大说明是调度开销 |
| DCGM_FI_PROF_DRAM_ACTIVE | DCGM | 显存控制器活跃周期占比 | decode 阶段 > 50% 说明访存受限 |
| DCGM_FI_DEV_FB_USED | DCGM / NVML | 已用显存（MiB） | 应稳定在 gpu_memory_utilization × 总显存 附近 |
| DCGM_FI_DEV_POWER_USAGE | DCGM | 功耗（W） | H100 PCIe 350W / SXM 700W 为上限 |
| DCGM_FI_DEV_SM_CLOCK | DCGM | SM 频率（MHz） | 与负载和功耗墙相关 |
| DCGM_FI_DEV_XID_ERRORS | DCGM | 最近的 XID 错误码 | 正常为 0 |

`DCGM_FI_PROF_*` 系列（profiling 指标）比 `DCGM_FI_DEV_*`（设备指标）采样开销高，且需要容器具备相应权限。GPU Operator 默认采集的部分 `PROF` 指标由 `dcgm-exporter` 的 `--collectors` 参数控制，不同版本默认值不同，必须先确认。

一句经验：**判断一个推理服务健不健康，先看 generation_tokens_total 的增长速率与 GPU 数量的比值（每卡每秒出多少个 token），再看 TTFT/TPOT 的分位数，最后才看 GPU 利用率。** 利用率高但每卡 token 吞吐低，就是浪费。

### 3.5 五个容易搞混的"使用率"

| 名称 | 指标 | 分子 | 高点说明什么 |
| --- | --- | --- | --- |
| 设备利用率 | DCGM_FI_DEV_GPU_UTIL | 有 kernel 在跑的时间 | 卡没闲着，仅此而已 |
| SM 活跃度 | DCGM_FI_PROF_SM_ACTIVE | 至少一个 warp 驻留的周期 | warp 调度层面基本饱和 |
| 张量核活跃度 | DCGM_FI_PROF_PIPE_TENSOR_ACTIVE | 张量类指令在执行的周期 | 真正在做矩阵乘 |
| 显存带宽活跃度 | DCGM_FI_PROF_DRAM_ACTIVE | 显存控制器忙的周期 | 被数据搬运卡住 |
| KV 池占用 | vllm:kv_cache_usage_perc | 已分配 block 数 | 调度器可用上下文容量见底 |

这五个数一起看，才能给出"这卡在干什么"的结论。单独任何一个都会误导。按判断矩阵来：

| GPU_UTIL | TENSOR_ACTIVE | DRAM_ACTIVE | KV 使用率 | 判断 |
| --- | --- | --- | --- | --- |
| 高 | 高 | 中 | 中 | 健康的 prefill 为主负载 |
| 高 | 低 | 高 | 中低 | decode 访存受限，batch 偏小，正常但可优化 |
| 高 | 低 | 中 | 接近 1.0 | KV 打满引发抢占重算，健康度差 |
| 低 | 低 | 低 | 低 | 真没流量，或调度器空转、CPU 侧卡住 |
| 低 | 高 | 高 | 中 | 采样或 GPU-CPU 同步导致等待，短时突发正常 |

值班时按这个矩阵判断，比背指标定义有效得多。

## 四、整体思路：从采集到告警的链路

### 4.1 链路设计

```text
┌─────────────┐   /metrics (9400)   ┌────────────┐
│ dcgm-exporter│────────────────────▶│            │
└─────────────┘                     │            │
┌─────────────┐   /metrics (8000)   │ Prometheus │──▶ Grafana
│ vllm server │────────────────────▶│  (scrape)  │
└─────────────┘                     │            │──▶ Alertmanager
┌─────────────┐   /metrics (9100)   │            │
│node-exporter│────────────────────▶│            │
└─────────────┘                     └────────────┘
┌─────────────┐   /metrics (8080)
│ vllm-router │───────────────────────────────────┘
└─────────────┘
```

四个采集点各有分工：dcgm-exporter 看硬件，vllm server 看引擎，node-exporter 看宿主机（网卡、CPU、文件系统，尤其是 NFS/对象存储挂载点的延迟），如果前面有 router 则 router 看路由层的队列。

### 4.2 第一件事：先把真实指标导出来

不要凭记忆写 PromQL。在正式搭监控之前，先手动抓一次指标清单：

```bash
# 1) 确认 vllm 服务已开 metrics（默认开启，显式写出来更清楚）
#    启动参数里不要加 --disable-log-stats；vLLM 默认暴露 /metrics
curl -s http://10.0.10.21:8000/metrics | grep '^# HELP' | awk '{print $3}' | sort
```

预期输出（节选，vLLM 0.32+）：

```text
http_request_duration_seconds
http_request_size_bytes
http_response_size_bytes
process_cpu_seconds_total
process_open_fds
process_resident_memory_bytes
python_gc_objects_collected_total
vllm:e2e_request_latency_seconds
vllm:generation_tokens_total
vllm:inter_token_latency_seconds
vllm:kv_cache_usage_perc
vllm:num_requests_running
vllm:num_requests_waiting
vllm:prefix_cache_hits
vllm:prefix_cache_queries
vllm:prompt_tokens_total
vllm:request_decode_time_seconds
vllm:request_prefill_time_seconds
vllm:request_prompt_tokens
vllm:request_queue_time_seconds
vllm:request_success_total
vllm:request_time_per_output_token_seconds
vllm:time_to_first_token_seconds
```

如果看到的是 `vllm:avg_prompt_throughput_toks_per_s` 或 `vllm:num_requests_swapped`，说明你跑的是较老的版本（V0 引擎或未迁移的代码路径），后面的面板需要调整。如果 `vllm:` 前缀的指标一条都没有，只有 `http_*`，那说明你在启动时用了 `--disable-log-stats` 之类的参数，或者前面挂了会过滤的代理。

同样先导 dcgm-exporter 的清单：

```bash
curl -s http://10.0.10.21:9400/metrics | grep -E '^DCGM_FI_(DEV_GPU_UTIL|DEV_FB_USED|PROF_PIPE_TENSOR_ACTIVE|PROF_DRAM_ACTIVE)' | head -20
```

看标签里有没有 `modelName`、`UUID`、`Hostname`、`gpu`、`device`。后三个用来对齐指标，`modelName` 用来在 Grafana 里按卡型区分阈值。

```bash
# 2) 裸机部署时，直接确认 exporter 进程与端口
sudo systemctl status nvidia-dcgm-exporter --no-pager
ss -lntp | grep 9400
nvidia-smi --query-gpu=index,uuid,name,memory.used,memory.total,utilization.gpu \
  --format=csv,noheader
```

第三条命令的输出要和 `/metrics` 里的数值对得上。如果 `nvidia-smi` 显示 78GB，而 `DCGM_FI_DEV_FB_USED` 只有 40GB，说明 exporter 采的可能不是同一个 GPU 实例（MIG 场景常见），或者指标是旧的缓存值。

### 4.3 采集频率与开销

- vLLM 的 /metrics 生成成本很低（内存中读计数器），15s 间隔没问题。GPU 数量多、模型多时，Prometheus 的系列数会线性增长，注意 scrape_samples_post_metric_relabeling。
- DCGM Exporter 的 PROF 指标采样间隔由 --collect-interval（默认 30000ms，单位毫秒）控制，不要和 Prometheus 的 scrape_interval 混为一谈。把 scrape 调到 5s 而 exporter 30s 才更新一次，只会造成 6 个完全相同的样本，浪费 TSDB。
- 每个 /metrics 请求都会触发一次指标序列化，长文本模型配大 max_num_seqs 时，vllm: 直方图的桶多，单次响应可能几百 KB。如果 exporter 和被监控服务在同一 GPU 节点上，注意 CPU 抢占对 prefill 的抖动影响。

## 五、实战步骤一：部署 DCGM Exporter

### 5.1 目的与前置检查

目的：拿到 GPU 的硬件层指标，且标签能与 Pod/节点对齐，便于在 Grafana 里按服务和卡型聚合。

前置检查：

```bash
# 驱动与 DCGM 版本（GPU Operator 环境下在宿主上执行）
nvidia-smi --query-gpu=driver_version --format=csv,noheader | sort -u
dcgmi --version 2>/dev/null || echo "dcgmi not installed on host"

# 确认 profiling 指标可用（需要 DCGM 与驱动支持）
dcgmi discovery -l 2>/dev/null | head
```

如果宿主机没装 dcgmi，在容器里执行也行：

```bash
docker run --rm --gpus all nvcr.io/nvidia/k8s/dcgm-exporter:4.4.2-4.6.0-ubuntu22.04 dcgmi discovery -l
```

预期看到每个 GPU 的 entity ID。如果报 `Error: Failed to initialize DCGM`，常见原因是容器没有拿到 `SYS_ADMIN` 或者 `/dev/nvidia*` 没有映射。

### 5.2 裸机 systemd 部署

```bash
# 1) 安装 DCGM（以 Ubuntu 22.04 + CUDA 12.8 仓库为例）
sudo apt-get update
sudo apt-get install -y datacenter-gpu-manager-4-cuda12
sudo systemctl enable --now nvidia-dcgm

# 2) 下载 dcgm-exporter（版本号按实际发布调整，勿用 latest 上生产）
DCGM_EXPORTER_VERSION=4.4.2-4.6.0
wget -q "https://github.com/NVIDIA/dcgm-exporter/archive/refs/tags/${DCGM_EXPORTER_VERSION}.tar.gz" -O /tmp/dcgm-exporter.tar.gz
mkdir -p /opt/dcgm-exporter && tar -xzf /tmp/dcgm-exporter.tar.gz -C /opt/dcgm-exporter --strip-components=1

# 3) 自定义指标白名单（只采必要的，降低开销）
sudo tee /etc/dcgm-exporter/metrics.csv > /dev/null <<'EOF'
# 格式：DCGM 字段名, 是否导出为 prometheus 指标（1=是）
DCGM_FI_DEV_GPU_UTIL,          gauge, GPU utilization (in %).
DCGM_FI_DEV_MEM_COPY_UTIL,     gauge, Memory utilization (in %).
DCGM_FI_DEV_FB_USED,           gauge, Frame buffer used (in MiB).
DCGM_FI_DEV_FB_FREE,           gauge, Frame buffer free (in MiB).
DCGM_FI_DEV_POWER_USAGE,       gauge, Power draw (in W).
DCGM_FI_DEV_GPU_TEMP,          gauge, GPU temperature (in C).
DCGM_FI_DEV_SM_CLOCK,          gauge, SM clock (in MHz).
DCGM_FI_DEV_XID_ERRORS,        gauge, Value of the last XID error encountered.
DCGM_FI_PROF_SM_ACTIVE,        gauge, The ratio of cycles an SM has at least 1 warp assigned.
DCGM_FI_PROF_PIPE_TENSOR_ACTIVE, gauge, The ratio of cycles the tensor (HMMA) pipe is active.
DCGM_FI_PROF_DRAM_ACTIVE,      gauge, The ratio of cycles the device memory interface is active.
EOF
```

这个 metrics.csv 的格式是 `字段名, 类型, 描述`，不是四列。如果从网上抄了三列的版本（老版本还有 `fb_used` 别名列），exporter 会启动失败并在日志里打 `unable to parse csv`。不同大版本的 dcgm-exporter 这个文件格式有差异，**务必对照所下载 tag 里的 etc/default-counters.csv**。

```bash
# 4) systemd unit
sudo tee /etc/systemd/system/dcgm-exporter.service > /dev/null <<'EOF'
[Unit]
Description=NVIDIA DCGM Exporter
After=nvidia-dcgm.service network-online.target
Wants=nvidia-dcgm.service

[Service]
Type=simple
ExecStart=/opt/dcgm-exporter/dcgm-exporter \
  -f /etc/dcgm-exporter/metrics.csv \
  -c 30000 \
  -d 1000 \
  --web.listen-address=:9400
Restart=always
RestartSec=5
User=root

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now dcgm-exporter
sudo systemctl status dcgm-exporter --no-pager -l
```

参数说明：

- -c 30000：采集间隔 30 秒（毫秒）。
- -d 1000：DCGM 内部统计窗口 1 秒，用于计算利用率。
- --web.listen-address：监听地址。默认 :9400，注意不要监听 0.0.0.0 暴露到公网。
- -f：指标配置文件，限制白名单能降低 30%~50% 的序列化开销（profil 指标尤其贵）。

生效验证：

```bash
curl -s http://127.0.0.1:9400/metrics | grep -c '^DCGM_FI_'
# 预期：与 metrics.csv 里启用的字段数 × GPU 数大致相等
curl -s http://127.0.0.1:9400/metrics | grep 'DCGM_FI_PROF_PIPE_TENSOR_ACTIVE{' | head -2
```

异常表现与判断：

| 现象 | 判断 | 下一步 |
| --- | --- | --- |
| /metrics 返回空或只有注释 | 白名单文件格式错误或字段名不匹配 | 对照 tag 内 default-counters.csv 核对字段名 |
| DCGM_FI_PROF_* 缺失 | 容器/进程权限不足，profiling 被禁用 | 用 root 运行，确认 --collectors 未排除；K8s 下需 SYS_ADMIN |
| 指标里所有 GPU 的 gpu 标签都是 0 | 只采到一张卡 | 检查 NVIDIA_VISIBLE_DEVICES 是否被限制 |
| DCGM_FI_DEV_FB_USED 与 nvidia-smi 差很多 | MIG 模式下按实例统计 | 用 gpu + UUID 标签区分 MIG instance |

### 5.3 Kubernetes 部署（GPU Operator）

GPU Operator 从 23.x 起默认通过 `nvidia-dcgm-exporter` DaemonSet 暴露 9400 端口。如果集群里已装，不要重复部署。

```bash
kubectl get ds -n gpu-operator nvidia-dcgm-exporter -o wide
kubectl get svc -n gpu-operator nvidia-dcgm-exporter
```

如果要用自定义白名单，通过 ConfigMap 挂载并 patch DaemonSet：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: dcgm-exporter-metrics
  namespace: gpu-operator
data:
  metrics.csv: |
    DCGM_FI_DEV_GPU_UTIL, gauge, GPU utilization (in %).
    DCGM_FI_DEV_FB_USED, gauge, Frame buffer used (in MiB).
    DCGM_FI_DEV_POWER_USAGE, gauge, Power draw (in W).
    DCGM_FI_PROF_PIPE_TENSOR_ACTIVE, gauge, Tensor pipe active ratio.
    DCGM_FI_PROF_DRAM_ACTIVE, gauge, DRAM active ratio.
```

```bash
kubectl apply -f dcgm-metrics-cm.yaml
kubectl -n gpu-operator patch ds nvidia-dcgm-exporter --type=json -p='[
  {"op":"add","path":"/spec/template/spec/volumes/-",
   "value":{"name":"custom-metrics","configMap":{"name":"dcgm-exporter-metrics"}}},
  {"op":"add","path":"/spec/template/spec/containers/0/volumeMounts/-",
   "value":{"name":"custom-metrics","mountPath":"/etc/dcgm-exporter/custom"}},
  {"op":"replace","path":"/spec/template/spec/containers/0/args",
   "value":["-f","/etc/dcgm-exporter/custom/metrics.csv","--web.listen-address=:9400"]}
]'
kubectl -n gpu-operator rollout status ds/nvidia-dcgm-exporter --timeout=180s
```

风险提示：`--type=json` 的 `add` 操作如果目标路径已存在会报错（例如 `volumes` 已经挂过同名卷），先 `kubectl -n gpu-operator get ds nvidia-dcgm-exporter -o yaml > backup-ds.yaml` 备份，失败时 `kubectl replace -f backup-ds.yaml` 回滚。GPU Operator 升级会重建这个 DaemonSet，自定义 patch 会丢失，要在 Operator 的 `ClusterPolicy` 里用 `dcgmExporter.config.name` 指定 ConfigMap，而不是直接 patch。

### 5.4 Prometheus 采集配置

裸机用 static_configs：

```yaml
# /etc/prometheus/prometheus.yml
scrape_configs:
  - job_name: dcgm-exporter
    scrape_interval: 15s
    scrape_timeout: 10s
    static_configs:
      - targets: ['10.0.10.21:9400', '10.0.10.22:9400']
        labels:
          cluster: prod-gpu
    relabel_configs:
      - source_labels: [__address__]
        regex: '([^:]+):.*'
        target_label: instance_ip
      - source_labels: [__address__]
        target_label: instance
```

K8s 用 ServiceMonitor（Prometheus Operator）：

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: nvidia-dcgm-exporter
  namespace: gpu-operator
  labels:
    release: kube-prometheus-stack
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: nvidia-dcgm-exporter
  namespaceSelector:
    matchNames: [gpu-operator]
  endpoints:
    - port: gpu-metrics
      interval: 15s
      path: /metrics
```

`selector.matchLabels` 必须和 Service 上的标签完全对齐，否则 ServiceMonitor 静默不生效（Prometheus 的 targets 页面看不到任何 endpoint，但也没有错误日志）。上线后去 Prometheus UI 的 `Status → Targets` 确认 endpoint 数量等于 GPU 节点数。

## 六、实战步骤二：vLLM 指标暴露与标签规范

### 6.1 启动参数确认

vLLM 默认在 API server 的同一端口暴露 `/metrics`。可以显式指定，避免被误以为关掉了：

```bash
vllm serve /models/Qwen3-32B \
  --served-model-name qwen3-32b \
  --tensor-parallel-size 4 \
  --max-model-len 32768 \
  --max-num-seqs 256 \
  --gpu-memory-utilization 0.90 \
  --enable-prefix-caching \
  --port 8000
```

端口相关参数别和 `--port`、`--host` 搞混。`--host 0.0.0.0` 会让 `/metrics` 和推理接口一起暴露在内网，如果节点有公网 IP，务必用安全组/防火墙限制 8000 端口只允许 Prometheus 和网关访问：

```bash
sudo nft add rule inet filter input tcp dport 8000 ip saddr { 10.0.10.0/24, 10.0.20.0/24 } accept
sudo nft add rule inet filter input tcp dport 8000 drop
```

第一条规则的白名单必须包含所有需要访问的网段（包括 Prometheus、网关、调试用的跳板机）。改防火墙前用 `nft list ruleset > /root/nft.bak` 备份，并在带外控制台或第二会话中执行，避免把自己锁在门外。

### 6.2 多副本部署的标签设计

一个模型部署 3 个副本，每个副本的 `/metrics` 里 `model_name` 标签相同。要区分副本，必须靠 Prometheus 的目标标签（`instance`、`pod`）。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vllm-qwen3-32b
  namespace: inference
spec:
  replicas: 3
  selector:
    matchLabels:
      app: vllm-qwen3-32b
  template:
    metadata:
      labels:
        app: vllm-qwen3-32b
        model: qwen3-32b
        team: ai-platform
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8000"
        prometheus.io/path: "/metrics"
    spec:
      containers:
        - name: vllm
          image: vllm/vllm-openai:v0.32.0
          args:
            - --model=/models/Qwen3-32B
            - --served-model-name=qwen3-32b
            - --tensor-parallel-size=4
            - --max-model-len=32768
            - --enable-prefix-caching
            - --port=8000
          ports:
            - name: http
              containerPort: 8000
          resources:
            limits:
              nvidia.com/gpu: 4
          readinessProbe:
            httpGet:
              path: /health
              port: 8000
            periodSeconds: 10
            failureThreshold: 3
```

注意这里 `resources.limits` 只写了 GPU，没写 CPU/memory。生产上必须写，否则节点内存耗尽时会触发 OOM Killer，而 OOM 掉的可能是同一节点上的另一个推理副本。给出参考值：32B 模型 TP=4、`max-num-seqs=256` 时，API server 进程 RSS 约 8~15GB，加上 tokenizer 缓存和 CUDA context，建议 `memory: 32Gi`（limit），`cpu: 16`。

用 `prometheus.io/scrape` 注解的方式依赖 Prometheus 的 kubernetes-pods 抓取配置，这种方式无法控制 sampling 参数也无法做 relabel，生产上更推荐用 PodMonitor（需要 8000 端口有个命名 port，上面已加 `name: http`）：

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: vllm-inference
  namespace: inference
  labels:
    release: kube-prometheus-stack
spec:
  jobLabel: model
  selector:
    matchLabels:
      app: vllm-qwen3-32b
  podMetricsEndpoints:
    - port: http
      path: /metrics
      interval: 15s
      relabelings:
        - source_labels: [__meta_kubernetes_pod_name]
          target_label: pod
        - source_labels: [__meta_kubernetes_node_name]
          target_label: node
```

`jobLabel: model` 会把 Pod 上的 `model: qwen3-32b` 标签变成 `job` 标签。这样在 Grafana 里 `job="qwen3-32b"` 就能直接选中一个模型的所有副本。

### 6.3 自定义直方图桶

TTFT 和 TPOT 用固定的桶，会在关键区间产生较大误差。比如默认桶在 0.1 到 1 秒之间的跨度很大，而如果你的 SLO 是 TTFT ≤ 500ms，这个区间的分位数就会不准。

vLLM 0.32+ 提供了 `--custom-histogram-buckets`，可以按需设置：

```bash
vllm serve /models/Qwen3-32B \
  --served-model-name qwen3-32b \
  --tensor-parallel-size 4 \
  --custom-histogram-buckets '{"vllm:time_to_first_token_seconds": [0.05,0.1,0.2,0.3,0.5,0.75,1.0,2.0,5.0,10.0], "vllm:request_time_per_output_token_seconds": [0.005,0.01,0.02,0.03,0.05,0.08,0.12,0.2,0.5]}'
```

不同版本这个参数的接收格式（JSON 字符串 vs 文件路径）可能不一致，上线前用 `vllm serve --help | grep -A3 custom-histogram` 确认。如果版本不支持，就退回到默认桶，并在 PromQL 里用 `histogram_quantile` 时对误差做说明，不要硬报一个"P99 精确到毫秒"的结果。

### 6.4 一个必须知道的坑：多进程模式下 process_* 指标消失

vLLM 的 API server 可以开多进程（`--api-server-count`，用于提高 HTTP 层吞吐）。开启后，`process_resident_memory_bytes`、`process_cpu_seconds_total` 这类来自 Python `prometheus_client` 默认注册器的指标不会再暴露。如果你的告警里有"vllm 进程内存超过 X"的规则，开了多进程后会静默失效——规则不触发不等于没问题，而是没有数据。

对策：用容器的 `container_memory_working_set_bytes`（来自 cAdvisor）替代，或者单独部署一个 sidecar 采集。这条规则一定要写进告警自检清单。

## 七、实战步骤三：Prometheus 告警规则

### 7.1 请求层：SLO 型告警

```yaml
# /etc/prometheus/rules/vllm-slo.yml
groups:
  - name: vllm-request-slo
    interval: 30s
    rules:
      - record: vllm:ttft_p95:5m
        expr: |
          histogram_quantile(0.95,
            sum by (job, le) (
              rate(vllm:time_to_first_token_seconds_bucket[5m])
            )
          )

      - record: vllm:ttft_p99:5m
        expr: |
          histogram_quantile(0.99,
            sum by (job, le) (
              rate(vllm:time_to_first_token_seconds_bucket[5m])
            )
          )

      - record: vllm:tpot_p95:5m
        expr: |
          histogram_quantile(0.95,
            sum by (job, le) (
              rate(vllm:request_time_per_output_token_seconds_bucket[5m])
            )
          )

      - record: vllm:request_success:rate5m
        expr: |
          sum by (job, finished_reason) (
            rate(vllm:request_success_total[5m])
          )

      - alert: VllmHighTTFT
        expr: vllm:ttft_p95:5m > 1.5
        for: 10m
        labels:
          severity: warning
          team: ai-platform
        annotations:
          summary: "{{ $labels.job }} 首字延迟 P95 超阈值"
          description: |
            {{ $labels.job }} 的 TTFT P95 = {{ $value | humanizeDuration }}，持续 10 分钟超过 1.5s。
            检查：1) vllm:num_requests_waiting 是否同时上涨；2) vllm:kv_cache_usage_perc 是否 > 0.95；
            3) DCGM_FI_PROF_PIPE_TENSOR_ACTIVE 是否偏低（访存受限）。

      - alert: VllmHighTPOT
        expr: vllm:tpot_p95:5m > 0.15
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "{{ $labels.job }} 单 token 延迟 P95 超阈值"
          description: "TPOT P95 = {{ $value | humanizeDuration }}，用户感知为明显卡顿。"
```

阈值不要抄。1.5s 的 TTFT 阈值只适合 32B 以下、TP 在 4 卡以内、`max-model-len` ≤ 32K 的配置。阈值推导方法在 7.4 节。

### 7.2 引擎层：排队与 KV Cache

```yaml
  - name: vllm-engine-saturation
    interval: 30s
    rules:
      - alert: VllmQueueBacklog
        expr: |
          max by (job) (vllm:num_requests_waiting) > 20
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "{{ $labels.job }} 排队请求数持续偏高"
          description: "waiting={{ $value }}，running 队列也接近上限时说明调度已饱和。"

      - alert: VllmKVCacheNearlyFull
        expr: |
          max by (job) (vllm:kv_cache_usage_perc) > 0.95
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "{{ $labels.job }} KV Cache 使用率 > 95%"
          description: |
            KV Cache 接近打满会触发抢占重算，表现为 TTFT 与 TPOT 同时恶化。
            查看 vllm:num_preemptions_total 的增长速率确认。

      - alert: VllmPreemptionSpike
        expr: |
          sum by (job) (rate(vllm:num_preemptions_total[5m])) > 0.2
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "{{ $labels.job }} 抢占速率异常"
          description: "每秒抢占 {{ $value | humanize }} 次，KV Cache 不足或 max-num-seqs 配置过大。"

      - alert: VllmAbortRatioHigh
        expr: |
          sum by (job) (rate(vllm:request_success_total{finished_reason="abort"}[5m]))
          /
          clamp_min(sum by (job) (rate(vllm:request_success_total[5m])), 0.001)
          > 0.15
        for: 15m
        labels:
          severity: info
        annotations:
          summary: "{{ $labels.job }} 请求中止比例 > 15%"
          description: "大量 abort 多为客户端超时断开，检查网关 read timeout 与前端体验。"
```

`vllm:num_preemptions_total` 在不同版本里是否存在需要先确认（`curl .../metrics | grep preempt`）。如果不存在，就改用 `kv_cache_usage_perc` + `num_requests_running` 的锯齿特征间接判断：running 数周期性掉到 0 再爬升，是很强的抢占信号。

### 7.3 GPU 层：区分"忙"和"有效忙"

```yaml
  - name: gpu-health
    interval: 30s
    rules:
      - record: gpu:tensor_active:p50
        expr: |
          quantile(0.5,
            DCGM_FI_PROF_PIPE_TENSOR_ACTIVE{gpu!=""}
          )

      - alert: GpuUtilHighButTensorLow
        expr: |
          (avg by (instance, gpu) (DCGM_FI_DEV_GPU_UTIL) > 85)
          and
          (avg by (instance, gpu) (DCGM_FI_PROF_PIPE_TENSOR_ACTIVE) < 0.20)
        for: 15m
        labels:
          severity: info
        annotations:
          summary: "{{ $labels.instance }} GPU{{ $labels.gpu }} 高占用低算力"
          description: |
            GPU_UTIL 高但张量核心活跃度低，典型为 decode 访存受限或 KV 抢占空转。
            结合 vllm:num_preemptions_total、DCGM_FI_PROF_DRAM_ACTIVE 一起判断。

      - alert: GpuXidError
        expr: |
          DCGM_FI_DEV_XID_ERRORS > 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "{{ $labels.instance }} GPU{{ $labels.gpu }} 出现 XID {{ $value }}"
          description: "XID 13/31/43/45/48/63/64/79/94/95 通常需要下线该卡并保留 dmesg。"

      - alert: GpuEccDoubleBitError
        expr: |
          increase(DCGM_FI_DEV_ECC_DBE_VOL_TOTAL[10m]) > 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "{{ $labels.instance }} GPU{{ $labels.gpu }} 出现 ECC 双比特错误"

      - alert: GpuTemperatureHigh
        expr: |
          DCGM_FI_DEV_GPU_TEMP > 85
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "{{ $labels.instance }} GPU{{ $labels.gpu }} 温度 {{ $value }}℃"
```

这里要明确一条纪律：**不要用"功耗接近上限"去推断降频。** 功耗顶到 TDP 只是说明卡在满载，不代表频率被压制；真正的降频原因要读位图指标。DCGM 里对应字段常见的有三组，不同 exporter 版本导出的名字不一样：

- DCGM_FI_DEV_CLOCK_THROTTLE_REASONS：位图，第 0 位是 GPU idle，第 3 位是 HW thermal slowdown，第 4 位是 HW power brake，第 5 位是 SW thermal，第 6 位是 HW slowdown；
- DCGM_FI_DEV_THERMAL_VIOLATION：累计的温度违规时间（微秒），单调递增；
- DCGM_FI_DEV_POWER_VIOLATION：累计的功耗违规时间（微秒），单调递增。

先确认环境里有哪几个：

```bash
curl -s http://10.0.10.21:9400/metrics | grep -iE 'throttle|violation' | awk -F'{' '{print $1}' | sort -u
```

再按实际情况写规则。用温度违规时间更直观，因为它是单调计数器，能做 `rate`：

```yaml
      - alert: GpuThermalViolation
        expr: |
          rate(DCGM_FI_DEV_THERMAL_VIOLATION[5m]) > 0.05
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "{{ $labels.instance }} GPU{{ $labels.gpu }} 因温度触发降频"
          description: "5 分钟内平均每秒有 {{ $value | humanize }} 秒处于温度违规状态。"

      - alert: GpuHwSlowdown
        expr: |
          (DCGM_FI_DEV_CLOCK_THROTTLE_REASONS % 16) >= 8
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "{{ $labels.instance }} GPU{{ $labels.gpu }} 硬件降频位被置位"
          description: |
            位图 bit3（HW thermal）/ bit4（HW power brake）/ bit6（HW slowdown）任一置位。
            先查散热风道与进风温度，再查是否有机箱内其他卡超温连带。
```

`DCGM_FI_DEV_CLOCK_THROTTLE_REASONS` 在不同驱动版本下返回的是位图或分字段形式，`% 16 >= 8` 这种写法只在返回位图时成立。稳妥做法是先在目标机上 `curl` 出实际值，用一个已知降频的卡核对数值，再定表达式。**规则的前提是数据形状已经确认，这一点在多版本混布的集群里尤其重要。**

这种"先查后写"的纪律要贯穿所有告警规则。规则上线后立刻做一次"制造异常"的验证：手工把 `max-num-seqs` 调大、灌压测流量，确认对应的告警确实能触发（用 `promtool test rules` 做单元测试也可以）。

### 7.4 阈值怎么推导，而不是怎么抄

以 TTFT 为例，分四步：

**第一步，确定 SLO**。和产品侧约定：TTFT P95 ≤ 1.2s，TPOT P95 ≤ 80ms（约每秒 12.5 个汉字）。这是用户可感知的基线，不是拍脑袋。

**第二步，压测得到基线**。用固定的输入输出长度（例如 input=1024、output=256），逐级加压到设计 QPS，记录每一级的 TTFT 分位数：

```bash
# 用 vLLM 自带 benchmark 客户端（在独立机器上跑，避免和被压服务抢 CPU）
vllm bench serve \
  --host 10.0.10.21 --port 8000 \
  --model qwen3-32b \
  --dataset-name random \
  --random-input-len 1024 \
  --random-output-len 256 \
  --num-prompts 500 \
  --request-rate 8 \
  --percentile-metrics ttft,tpot,e2el \
  --metric-percentiles 50,95,99
```

预期输出形如：

```text
============ Serving Benchmark Result ============
Successful requests:                     500
Benchmark duration (s):                  62.51
Request throughput (req/s):              8.00
Output token throughput (tok/s):         2047.12
---------------Time to First Token----------------
Mean TTFT (ms):                          318.42
Median TTFT (ms):                        291.07
P95 TTFT (ms):                           612.88
P99 TTFT (ms):                           903.15
-----Time per Output Token (excl. 1st token)------
Mean TPOT (ms):                          22.14
P95 TPOT (ms):                           31.02
==================================================
```

**第三步，留安全余量**。告警阈值取"压测 P95 的 1.5~2 倍"，并且不超过 SLO 的 1.25 倍。上例中压测 P95 = 613ms，SLO = 1200ms，取 `max(613 × 1.8, 1200 × 1.15) ≈ 1380ms`。所以之前写的 1.5s 阈值基本合理。

**第四步，区分"慢"和"故障"**。同一个指标要两条规则：`> SLO×1.15` 持续 10 分钟报 warning（服务劣化），`> SLO×3` 持续 2 分钟报 critical（可能已经不可用）。只设一条规则会导致告警要么太吵要么太晚。

### 7.5 记录规则：避免面板把 Prometheus 打爆

Grafana 面板里直接写 `histogram_quantile` 对 30 天区间做聚合，会让 Prometheus 一次加载几百万个 bucket 样本，查询超时或 OOM。做法是把常用聚合写成 recording rule，面板只读结果：

```yaml
      - record: vllm:job:ttft_bucket_rate5m
        expr: |
          sum by (job, le) (rate(vllm:time_to_first_token_seconds_bucket[5m]))

      - record: vllm:job:tokens_per_sec
        expr: |
          sum by (job) (rate(vllm:generation_tokens_total[2m]))
```

注意 recording rule 的 label 必须包含 `le`，否则 `histogram_quantile` 无法工作。这是一个高频错误：`sum without (instance, pod)` 会把 `le` 也去掉，得到的是每 5 分钟总请求数，不是分位数。

## 八、实战步骤四：Grafana 面板设计

### 8.1 面板分区原则

一个好的推理服务面板不是"把指标都放上去"，而是按排查路径组织。我习惯分成四行：

第一行，**SLO 概览**（只放 4 个数字）：当前 QPS、TTFT P95、TPOT P95、错误/中止率。放在最上面，一屏看到"服务好不好"。

第二行，**三层趋势联动**（同一时间轴）：上排是请求层延迟曲线，中排是引擎层队列与 KV 使用率，下排是 GPU 层的 UTIL/TENSOR_ACTIVE。三条曲线共用 x 轴，恶化时刻对得上，根因一眼可见。

第三行，**吞吐与成本**：输入/输出 token 速率、每卡每秒 token 数、前缀缓存命中率。这一行是给容量规划和成本优化用的。

第四行，**硬件健康**：温度、功耗、ECC、XID、显存趋势。平时没人看，出事时救命。

### 8.2 关键面板 PromQL

**每卡每秒输出 token 数**（成本核心指标）：

```
sum by (job) (rate(vllm:generation_tokens_total[2m]))
/
count by (job) (DCGM_FI_DEV_GPU_UTIL)
```

分母用 GPU 数需要能对齐 `job` 标签。如果 DCGM 和 vLLM 的 `job` 标签体系不同（一个按模型、一个按节点），就用固定常量除，或建一张"节点—模型"映射的 `label_join` 规则。更稳妥的写法是用 GPU 的 `UUID` 和 vLLM 的 `pod` 通过 `node` 标签关联，但那需要给两侧都打上 `node` 标签（第 6.2 节的 relabel 已经做了）。

**前缀缓存命中率**：

```
sum by (job) (rate(vllm:prefix_cache_hits[5m]))
/
clamp_min(sum by (job) (rate(vllm:prefix_cache_queries[5m])), 0.001)
```

含义需要注意：`vllm:prefix_cache_queries` 和 `vllm:prefix_cache_hits` 都是按 token 数计的 Counter，不是请求数。命中率 40% 意味着 40% 的 prompt token 从缓存中复用，直接体现为 prefill 开销下降和 TTFT 改善。RAG 场景（固定 system prompt + 固定文档片段）通常能到 50%~70%；纯对话场景低于 10% 是正常的，不是故障。

**排队时间占比**：

```
sum by (job) (rate(vllm:request_queue_time_seconds_sum[5m]))
/
clamp_min(sum by (job) (rate(vllm:request_queue_time_seconds_count[5m])), 0.001)
```

这个值除以平均 E2E 延迟，就是"用户等待中由排队贡献的比例"。超过 30% 说明问题在容量或调度，不在模型本身——加卡比改参数有效。

**输入输出 token 比**（判断负载形态）：

```
sum by (job) (rate(vllm:prompt_tokens_total[5m]))
/
clamp_min(sum by (job) (rate(vllm:generation_tokens_total[5m])), 0.001)
```

这个比值在 8:1 以上，说明是"长输入短输出"型负载（RAG、分类、抽取），优化重点在 prefill 和前缀缓存；在 0.5:1 以下说明是"长输出"型（创作），优化重点在 decode 的 KV Cache 与 batch size。

### 8.3 面板上要标注的"误导性指标"

Grafana 里所有设备类指标旁边都加一条 text panel 说明，避免值班同学误判：

- GPU_UTIL 100% 不代表"算力用满了"，只代表"kernel 一直在跑"。
- DCGM_FI_DEV_FB_USED 接近总显存是正常状态（vLLM 预分配 gpu_memory_utilization 比例的显存做 KV 池），不是"显存泄漏"。
- num_requests_waiting = 0 有可能是"没有流量"，要一起看 QPS。
- request_success_total{finished_reason="length"} 大量出现，说明 max_tokens 或 max-model-len 设小了，用户回答被截断。

## 九、日志与指标的对账

### 9.1 vLLM 自己的统计日志

vLLM 默认每 10 秒（`--log-stats-interval` 可调）打一行引擎统计：

```text
INFO 11-10 14:22:31 [metrics.py:521] Avg prompt throughput: 4123.5 tokens/s,
Avg generation throughput: 1876.2 tokens/s, Running: 42 reqs, Waiting: 3 reqs,
GPU KV cache usage: 91.3%, Prefix cache hit rate: 38.7%
```

这一行信息密度很高，是现场排查的第一手材料。要能和 Prometheus 的数据对上：

```bash
# 从 vLLM 日志里抽统计行（假设日志走 journald 或容器 stdout）
kubectl logs -n inference deploy/vllm-qwen3-32b --tail=200 \
  | grep 'Avg prompt throughput' | tail -5
```

如果日志显示 `GPU KV cache usage: 91.3%`，而面板上的 `vllm:kv_cache_usage_perc` 是 0.63，说明面板取的时间窗口或样本不对（例如面板用了 `max_over_time` 而日志是瞬时值），先去 `query` 页面直接查瞬时值确认。**两套数据不一致时，先怀疑采集和查询，不要急着改服务。**

`Running: 42 reqs` 与 `vllm:num_requests_running` 的关系也一样，要能对上。如果对不上且差距稳定，可能是 `--api-server-count` 大于 1，指标由某个子进程聚合，日志由另一个统计，这种非一致是已知现象，需在文档里注明。

### 9.2 把慢请求的 trace 落下来

指标只能告诉你"慢了"，要定位到具体请求，需要请求级日志。给 vLLM 加一个 `--enable-request-id-headers` 之类的自定义头透传（不同版本参数名不同，也可能需要自己封装），然后在 Nginx 侧记录上游耗时：

```
log_format vllm_upstream '$remote_addr - $http_x_request_id - "$request" '
                         '$status $body_bytes_sent '
                         'rt=$request_time uct=$upstream_connect_time '
                         'uht=$upstream_header_time urt=$upstream_response_time';
access_log /var/log/nginx/vllm_access.log vllm_upstream;

location /v1/ {
    proxy_pass http://vllm_upstream;
    proxy_buffering off;            # SSE 流式必须关闭缓冲
    proxy_cache off;
    proxy_read_timeout 600s;        # 长输出场景要放大
    proxy_http_version 1.1;
    proxy_set_header Connection "";
    proxy_set_header X-Request-Id $request_id;
    proxy_set_header Host $host;
}
```

`uht`（upstream header time）对 SSE 流式接口来说约等于 TTFT，`urt` 约等于 E2E。这两个字段和 vLLM 的直方图对账，能发现"vLLM 内部很快但网关侧很慢"的情况——通常原因是 `proxy_buffering on` 让 token 被攒着一起发，或者网关到客户端链路慢。

配置生效方式与验证：

```bash
sudo nginx -t && sudo nginx -s reload
# 验证：发一个流式请求，看 token 到达是否连续
curl -N -s -X POST http://gw.example.com/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{"model":"qwen3-32b","stream":true,"messages":[{"role":"user","content":"用 50 个字介绍 PagedAttention"}]}' \
  | while IFS= read -r line; do echo "$(date +%H:%M:%S.%3N) $line"; done
```

预期：每行时间戳间隔均匀（几十毫秒）。如果出现"长时间无输出然后一次刷出一大段"，就是缓冲没关，需要检查是否有 CDN 或云负载均衡在中间层做了缓冲。

### 9.3 一次慢请求的多层对账样例

现场记录（已脱敏，模型 32B，TP=4，H100）：

| 层 | 指标/日志 | 观测值 | 结论 |
| --- | --- | --- | --- |
| 网关 | urt P99 | 11.2s | 客户端确实慢 |
| vLLM 请求层 | e2e_request_latency_seconds P99 | 10.8s | 慢在 vLLM 内部 |
| vLLM 请求层 | time_to_first_token_seconds P99 | 6.4s | 大头在首字 |
| vLLM 引擎层 | request_queue_time_seconds 均值 | 5.9s | 排队占了 5.9s |
| vLLM 引擎层 | num_requests_waiting | 68（持续） | 队列积压 |
| vLLM 引擎层 | kv_cache_usage_perc | 0.98 | KV 打满 |
| vLLM 引擎层 | num_preemptions_total 增速 | 3.1/s | 频繁抢占重算 |
| GPU 层 | DCGM_FI_DEV_GPU_UTIL | 100% | 表面满负荷 |
| GPU 层 | DCGM_FI_PROF_PIPE_TENSOR_ACTIVE | 0.07 | 算力实际闲置 |

结论：KV Cache 不足 → 抢占 → 重算 prefill → GPU 忙但有效算力低 → 首字延迟暴涨。这不是"加卡"能根治的问题（加卡确实缓解，但成本高），首选是降低 `max-model-len`、降低 `max-num-seqs`、启用/调优前缀缓存、或对长上下文请求单独分流到长上下文实例池。

这个对账表就是故障复盘模板的核心部分。凡是"根因"，后面必须跟着至少两列不同来源的证据。

## 十、排查路径：从症状到根因

### 10.1 决策树

```text
用户投诉慢
│
├─ 请求层：TTFT P95 还是 TPOT P95 超标？
│   │
│   ├─ TTFT 高，TPOT 正常
│   │   ├─ queue_time 占比 > 30% → 排队问题
│   │   │   ├─ waiting 队列长 & kv_cache < 0.9 → 到达率超过处理能力，扩容或限流
│   │   │   └─ waiting 队列长 & kv_cache > 0.95 → KV 不足，降 max-model-len / max-num-seqs
│   │   ├─ queue_time 占比 < 10% → prefill 本身慢
│   │   │   ├─ request_prefill_time 高 & prompt_tokens 分布尾巴长 → 长 prompt 拖累，开 chunked prefill
│   │   │   └─ prefill 时间高但 prompt 不长 → TENSOR_ACTIVE 低？查降频、查前缀缓存是否失效
│   │   └─ 两者都不高但 TTFT 仍高 → 采样/后处理开销，查 model_execute_time
│   │
│   ├─ TPOT 高，TTFT 正常
│   │   ├─ running 数小但 TPOT 高 → batch 不足，decode 带宽利用率低（正常，无法优化到无限）
│   │   ├─ running 数大 & TPOT 高 → 算力打满，单卡 token 速率见顶，扩容
│   │   ├─ running 数周期性锯齿 & preemptions 涨 → 抢占重算
│   │   └─ ITL P99 高但 TPOT 均值正常 → 尾延迟抖动，查 CPU 抢占、查日志刷盘
│   │
│   └─ TTFT/TPOT 都正常，但 E2E 高
│       ├─ output_tokens 分布尾巴长 → 正常现象，用户请求的是长回答
│       └─ SSE 传输慢 → 网关缓冲、客户端带宽、CDN
│
└─ GPU 层：有 XID / ECC / 降频 / 温度异常？
    ├─ 有 → 先处理硬件，硬件问题会污染所有上层指标
    └─ 无 → 回到引擎层
```

### 10.2 五个高频现场案例

**案例一：GPU 利用率 100%，但每卡 token 吞吐只有 300 tok/s（H100 60B 级模型正常应在 2000+）**

初判：不是算力问题，是访存或同步问题。

检查：

```bash
# 1) 看 tensor pipe 活跃度
curl -s http://10.0.10.21:9400/metrics | grep 'DCGM_FI_PROF_PIPE_TENSOR_ACTIVE{'
# 2) 看 DRAM 活跃度
curl -s http://10.0.10.21:9400/metrics | grep 'DCGM_FI_PROF_DRAM_ACTIVE{'
# 3) 看 batch 大小（running 数）
curl -s http://10.0.10.21:8000/metrics | grep -E 'num_requests_running|num_requests_waiting'
```

如果 TENSOR_ACTIVE = 0.04、DRAM_ACTIVE = 0.88、running = 2，那么根因是 **batch 太小，decode 完全被显存带宽卡住**。这类情况常见于：QPS 很低但模型很大、或者客户端的请求被网关串行化。

一个容易忽略的原因：网关侧的 `keepalive` 和连接池配置不当，导致请求被排队在 Nginx 层而不是 vLLM 层。用 `ss -tn state established '( dport = :8000 )' | wc -l` 看并发连接数，如果只有个位数而 QPS 应该有几十，说明连接复用把并发吃掉了。

**案例二：TTFT 突然从 600ms 涨到 4s，KV Cache 使用率反而下降**

初判：不是容量问题。KV 使用率下降说明并发变少了，那 TTFT 变高只能是 prefilling 变慢或排队机制变了。

检查步骤：

```bash
# 1) prefill 时间分布
curl -s http://10.0.10.21:8000/metrics | grep 'request_prefill_time_seconds_sum\|request_prefill_time_seconds_count'
# 2) 前缀缓存命中率变化
curl -s http://10.0.10.21:8000/metrics | grep 'prefix_cache'
# 3) 最近的配置变更
kubectl -n inference rollout history deploy/vllm-qwen3-32b
kubectl -n inference describe deploy/vllm-qwen3-32b | grep -A2 'Last Update'
# 4) CPU 侧是否被抢占（prefill 在 CPU 上做 tokenize 和调度）
sar -u 1 5
```

真实原因往往是**有人改了 system prompt 或者请求格式**，导致前缀缓存的命中点前移，命中率从 60% 掉到 5%。`prefix_cache_hits` 的曲线会明显下台阶，和 TTFT 恶化的时间点完全重合。这种问题的"修复"不在运维侧，而在业务侧——把可变的用户信息从 system prompt 移到 messages 末尾。

**案例三：P99 延迟稳定在某个固定值（比如 60s），P50 正常**

初判：固定的 P99 通常是超时值，不是真实延迟。60s 可能是网关、客户端或者 vLLM 的某个 timeout 参数。

检查：

```bash
# 找超时配置
grep -rn 'timeout' /etc/nginx/conf.d/*.conf | grep -i upstream
kubectl -n inference get deploy vllm-qwen3-32b -o yaml | grep -i timeout
# 看 abort 请求的比例
curl -s http://10.0.10.21:8000/metrics | grep 'request_success_total'
```

`request_success_total{finished_reason="abort"}` 占 15%，就印证了这一点：请求在 60s 时被客户端或网关断开，vLLM 记为 abort，但 latency 直方图记录的是 60s。**修复方向是确认到底是哪一层超时**，而不是优化模型。如果是 `max_tokens` 设得过大导致单请求时长超过网关的 `proxy_read_timeout`，那就该同时调整两个参数，而不是只放大超时——放大超时会让排队更严重。

**案例四：显存 OOM，服务重启，日志 CUDA out of memory**

初判：不是"显存泄漏"，而是并发或长度超过了预分配。

```bash
# 看 OOM 前的关键日志
kubectl logs -n inference vllm-qwen3-32b-xxxx --previous | tail -60 | grep -i -E 'oom|out of memory|maximum|seq'
```

典型日志：

```text
torch.OutOfMemoryError: CUDA out of memory. Tried to allocate 2.18 GiB.
GPU 0 has a total capacity of 79.11 GiB of which 1.92 GiB is free.
Of the allocated memory 76.31 GiB is allocated by PyTorch,
and 156.42 MiB is reserved by PyTorch but unallocated.
```

vLLM 预分配 KV 池后，理论上不会 OOM；出现 OOM 通常是：`--max-model-len` 大于模型实际支持的上下文、`--gpu-memory-utilization` 设得太高（0.95 以上没有余量应对 activation 峰值）、或者多进程 API server + 大 batch 的组合让 activation 峰值超出预估。

判断依据：看 `vllm:kv_cache_usage_perc` 是否在 OOM 前达到 1.0，以及 `request_prompt_tokens` 的 P99 是否远超 `--max-model-len / 2`。修复用保守配置：`--gpu-memory-utilization 0.88`，并加 `--max-num-seqs` 上限。

**案例五：只有某一个副本慢，其他正常**

初判：副本间差异一定是环境差异或硬件差异。

```bash
# 逐副本抓指标
for p in $(kubectl -n inference get pod -l app=vllm-qwen3-32b -o name); do
  echo "== $p"
  kubectl -n inference exec "$p" -- curl -s http://127.0.0.1:8000/metrics \
    | grep -E '^vllm:(time_to_first_token_seconds_sum|time_to_first_token_seconds_count)'
done
# 对应节点的 GPU 健康
kubectl get pod -l app=vllm-qwen3-32b -o wide
nvidia-smi --query-gpu=index,serial,clocks_throttle_reasons.active,power.draw,temperature.gpu \
  --format=csv
```

检查顺序：GPU 是否降频（`clocks_throttle_reasons.active` 非 0x0）→ NVLink/PCIe 是否降速（`nvidia-smi -q | grep -A3 'Link Width'`）→ 节点上是否有其他进程抢 GPU → 该副本所在的 GPU 是否曾经有 XID 记录（`dmesg -T | grep -i xid`）。

真实现场里，"一个副本慢 3 倍"最常见的原因是这张卡的 NVLink 降速或 HBM 显存有 ECC 错误被降级使用。这类问题不会自愈，必须下线换卡。

## 十一、风险与生产注意事项

### 11.1 监控本身的成本

- 指标基数爆炸。每个 vLLM 副本的直方图有约 20 个指标 × 15~30 个桶 = 400+ 序列。30 个副本就是 1.2 万序列，再加 DCGM 的 PROF 指标（每卡 10+ 个），Prometheus 内存占用轻松上 GB。对策：控制 --custom-histogram-buckets 的长度（不超过 12 个桶），用 metric_relabel_configs 丢弃不需要的 python_* 指标。

```yaml
    metric_relabel_configs:
      - source_labels: [__name__]
        regex: 'python_gc_.*|process_open_fds'
        action: drop
```

- **跨节点抓取延迟影响数据准确性**。Prometheus 抓取的目标在 GPU 节点上，如果网络抖动，抓取超时会导致 `up == 0`，误报。对策：`scrape_timeout` 不要超过 `scrape_interval` 的 60%，并对 `up == 0` 单独做短时间窗（2 分钟）抑制再转告警。
- **告警风暴的抑制**。GPU 节点宕机会同时触发：`up==0`、`VllmQueueBacklog`、`VllmHighTTFT`、`GpuXidError`。必须建立依赖抑制（inhibit_rules）：节点级 critical 抑制该节点上所有 Pod 级 warning。

```yaml
inhibit_rules:
  - source_matchers: [alertname="GpuNodeDown"]
    target_matchers: [severity="warning"]
    equal: [instance]
```

### 11.2 变更管理

监控系统的变更本身也要走流程：

- 新增/修改告警规则：先在 staging Prometheus 上跑 promtool check rules 和 promtool test rules 单元测试，再灰度到 10% 的消息通道，观察 24 小时误报率。
- 调整抓取间隔：会改变 TSDB 增长率，需要评估存储容量。H100 单节点 + vLLM 的典型序列数是 3000~6000，15s 间隔约 2~4 KB/s，一天约 200~350MB 磁盘。把间隔从 15s 改成 5s，磁盘消耗变 3 倍。
- Grafana 面板变更：用版本化 JSON 存在 Git 里，禁止在线手改。手改的面板在升级 Grafana 或迁移环境时一定会丢。

### 11.3 权限与安全

- /metrics 端点没有鉴权。在 Kubernetes 里用 NetworkPolicy 限制只能被 Prometheus 所在 namespace 访问：

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-prometheus-to-vllm-metrics
  namespace: inference
spec:
  podSelector:
    matchLabels:
      app: vllm-qwen3-32b
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: monitoring
      ports:
        - protocol: TCP
          port: 8000
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: ingress-nginx
      ports:
        - protocol: TCP
          port: 8000
  policyTypes: [Ingress]
```

这条策略会让除 monitoring 和 ingress-nginx 之外的所有入站流量被拒绝。**应用前先确认没有其他组件依赖直连**（例如压测工具在 `bench` namespace 里），否则压测时会静默连不通。用 `kubectl -n inference get networkpolicy` 确认没有冲突策略。

- Grafana 的面板要给"查看"权限，不给"编辑"权限；数据源要开 --metrics 只读模式。
- DCGM 的 profiling 指标包含 GPU 卡序列号（serial 标签在部分版本中），不要在对外分享的报表里保留。

## 十二、总结

把这篇的核心结论压缩成几条可以贴墙上的规则：

**第一条，GPU 利用率不是健康指标，是解释指标。** 判断服务好坏永远从请求层开始，GPU 指标只用来解释请求层为什么变差。看到 100% 先问"忙的是有效计算吗"，用 `PROF_PIPE_TENSOR_ACTIVE` / `PROF_DRAM_ACTIVE` 区分算力受限和访存受限。

**第二条，延迟必须拆开看。** TTFT = 排队 + prefill，E2E = TTFT + (N-1)×TPOT。任何一个"慢"都能被这三个量分解，分解完就知道问题是容量、调度还是硬件。

**第三条，吞吐要分输入输出。** 只报一个"总 token 吞吐"是在掩盖负载形态。输入输出比决定了优化方向：比值高就优化前缀缓存和 prefill，比值低就优化 KV Cache 和 batch。

**第四条，指标名先验证再用。** vLLM 和 DCGM 的指标集在版本之间会变。任何 dashboard 和告警上线前，`curl /metrics` 导出真实清单，比对一次。

**第五条，阈值靠压测推导，不靠抄。** SLO → 压测基线 → 留余量 → 分 warning/critical 两档。没有压测数据的阈值就是猜测，会在关键时刻失灵。

**第六条，硬件问题优先处理。** XID、ECC、降频会污染所有上层指标。监控链路里一定要有独立的硬件健康层，并且它的告警优先级高于业务指标。

最后说一句实践体会：推理服务的监控和传统 Web 服务最大的区别在于"资源是硬的、时间是长的、批处理是共享的"。传统服务单个请求独享一个进程、耗时毫秒级，指标之间基本独立；推理服务里所有请求共享同一批 GPU 和同一个 KV 池，一个长 prompt 会拖慢同批所有人的 TPOT，一个打满的 KV 池会让新请求排队几十秒。这种"互相影响"的特性决定了必须把三层指标放在同一张图上、同一条时间轴上看——单看任何一层，都会得出错误结论。

## 十三、附录：可复用的巡检脚本与 runbook

### 13.1 一键采集现场快照

值班时不希望手忙脚乱地一条条敲命令，把关键信息一次性落盘，事后复盘也有据可查。脚本只读，不做任何变更：

```bash
#!/usr/bin/env bash
# vllm-snapshot.sh —— 采集 vLLM 推理服务现场信息（只读）
# 用法: ./vllm-snapshot.sh <vllm_endpoint> <dcgm_endpoint> <outdir>
set -Eeuo pipefail

VLLM_EP="${1:?usage: $0 <vllm_host:port> <dcgm_host:port> <outdir>}"
DCGM_EP="${2:?dcgm endpoint required}"
OUT="${3:-./snapshot-$(date +%Y%m%d-%H%M%S)}"

mkdir -p "${OUT}"
exec > >(tee -a "${OUT}/run.log") 2>&1

log() { printf '[%s] %s\n' "$(date '+%F %T')" "$*"; }

need() {
  command -v "$1" >/dev/null 2>&1 || { log "缺少依赖: $1"; exit 1; }
}
need curl

log "开始采集 vllm=${VLLM_EP} dcgm=${DCGM_EP}"
log "输出目录: ${OUT}"

# 1) 原始指标（保留完整原文，便于事后核对）
if curl -fsS --max-time 10 "http://${VLLM_EP}/metrics" -o "${OUT}/vllm.metrics"; then
  log "vLLM /metrics 采集成功: $(wc -l < "${OUT}/vllm.metrics") 行"
else
  log "vLLM /metrics 采集失败（端口/鉴权/服务未起？）"
fi

if curl -fsS --max-time 10 "http://${DCGM_EP}/metrics" -o "${OUT}/dcgm.metrics"; then
  log "DCGM /metrics 采集成功: $(wc -l < "${OUT}/dcgm.metrics") 行"
else
  log "DCGM /metrics 采集失败"
fi

# 2) 关键标量指标摘要
if [[ -s "${OUT}/vllm.metrics" ]]; then
  grep -E '^vllm:(num_requests_running|num_requests_waiting|kv_cache_usage_perc|gpu_cache_usage_perc)' \
    "${OUT}/vllm.metrics" > "${OUT}/vllm.key_gauges.txt" || true
  grep -E '^vllm:request_success_total' \
    "${OUT}/vllm.metrics" > "${OUT}/vllm.finished_reasons.txt" || true
fi

# 3) 延迟直方图的累计和与计数（用于离线算均值，无需等 Prometheus）
if [[ -s "${OUT}/vllm.metrics" ]]; then
  grep -E '^vllm:(time_to_first_token_seconds|request_time_per_output_token_seconds|e2e_request_latency_seconds)_(sum|count)' \
    "${OUT}/vllm.metrics" > "${OUT}/vllm.latency_sums.txt" || true
fi

# 4) 硬件直读（如本机有 GPU）
if command -v nvidia-smi >/dev/null 2>&1; then
  nvidia-smi --query-gpu=index,uuid,name,memory.used,memory.total,utilization.gpu,\
utilization.memory,temperature.gpu,power.draw,clocks.sm,clocks_throttle_reasons.active \
    --format=csv > "${OUT}/nvidia-smi.csv" 2>/dev/null || log "nvidia-smi 查询失败"
  nvidia-smi --query-compute-apps=pid,process_name,used_memory \
    --format=csv > "${OUT}/nvidia-smi-procs.csv" 2>/dev/null || true
fi

log "采集完成，共 $(find "${OUT}" -type f | wc -l) 个文件"
log "建议同时保存: kubectl -n inference describe pod / journalctl -u vllm-service --since '-30 min'"
```

使用说明：

- 第一、二个参数必须显式给出，脚本没有默认值，避免误指向生产环境的其他服务。
- set -Eeuo pipefail 保证出错即停；但对每条采集命令又用了 || log 兜底，因为现场往往只有部分端点可达，全部失败也要把目录建出来。
- exec > >(tee ...) 用的是进程替换，bash 需要 4.0+；如果目标机是 /bin/sh（dash），必须显式用 bash script.sh 调用。

预期输出节选：

```text
[2026-11-10 14:41:02] 开始采集 vllm=10.0.10.21:8000 dcgm=10.0.10.21:9400
[2026-11-10 14:41:02] 输出目录: ./snapshot-20261110-144101
[2026-11-10 14:41:02] vLLM /metrics 采集成功: 1468 行
[2026-11-10 14:41:03] DCGM /metrics 采集成功: 312 行
[2026-11-10 14:41:03] 采集完成，共 7 个文件
```

如果 vLLM 那行是"采集失败"，先 `curl -v` 看是连接被拒（服务没起）还是 403/404（路径或鉴权问题）。DCGM 失败但 vLLM 成功，说明是 exporter 单独挂了，此时业务指标仍可信，硬件指标需要补采。

### 13.2 runbook：TTFT 恶化时的标准动作

按顺序执行，每一步都记录结果，不要跳步：

| 步 | 动作 | 命令/查询 | 判断依据 | 下一步 |
| --- | --- | --- | --- | --- |
| 1 | 确认是否真实劣化 | vllm:ttft_p95:5m 曲线 | 与前 7 天同时段对比，涨幅 > 30% | 涨了进第 2 步，没涨查客户端 |
| 2 | 看排队 | vllm:num_requests_waiting | > 20 且持续 5 分钟 | 进第 3 步 |
| 3 | 看 KV | vllm:kv_cache_usage_perc | > 0.95 | 进第 4 步（KV 不足分支） |
| 4a | KV 不足 | 查 request_prompt_tokens 分布 | P99 是否接近 max-model-len | 是→降 max-model-len 或分流长请求 |
| 4b | KV 正常但排队 | 对比 QPS 与历史容量上限 | QPS 是否超过压测得出的拐点 | 是→扩容/限流 |
| 5 | 看 prefill | rate(request_prefill_time_seconds_sum)/count | 与前值对比 | 变高进第 6 步 |
| 6 | 看前缀缓存 | prefix_cache_hits / queries | 是否下台阶 | 是→查 prompt 结构变更 |
| 7 | 看硬件 | TENSOR_ACTIVE、throttle reasons、XID | 是否降频/报错 | 是→下线该卡 |

每一步的"下一步"都是确定的分支，不允许出现"再观察观察"这种含糊结论——观察是可以的，但要写明观察什么指标、观察多久、什么条件下重新进入流程。

### 13.3 压测脚本：把基线数据固化下来

基线只有在环境稳定时才有意义，所以压测要在固定的配置、固定的数据长度下做，并把结果存档：

```bash
#!/usr/bin/env bash
# baseline.sh —— 分级加压，输出每级的 TTFT/TPOT/吞吐
set -Eeuo pipefail

HOST="${HOST:-10.0.10.21}"
PORT="${PORT:-8000}"
MODEL="${MODEL:-qwen3-32b}"
IN_LEN="${IN_LEN:-1024}"
OUT_LEN="${OUT_LEN:-256}"
RATES="${RATES:-2 4 8 12 16 24}"
RESULT_DIR="${RESULT_DIR:-./baseline-$(date +%Y%m%d-%H%M%S)}"

mkdir -p "${RESULT_DIR}"

for r in ${RATES}; do
  echo "=== request-rate=${r} req/s ==="
  vllm bench serve \
    --host "${HOST}" --port "${PORT}" --model "${MODEL}" \
    --dataset-name random \
    --random-input-len "${IN_LEN}" \
    --random-output-len "${OUT_LEN}" \
    --num-prompts $(( r * 30 )) \
    --request-rate "${r}" \
    --percentile-metrics ttft,tpot,e2el \
    --metric-percentiles 50,95,99 \
    --save-result \
    --result-dir "${RESULT_DIR}" \
    --result-filename "rate-${r}.json" 2>&1 | tee "${RESULT_DIR}/rate-${r}.log"
  sleep 20   # 每级之间留冷却时间，避免上一级的排队影响下一级
done

echo "结果保存在 ${RESULT_DIR}"
```

几个容易踩坑的点：

- --num-prompts 要足够大，至少是 request_rate × 30，否则压测在后半段流量就断崖了，算出来的吞吐虚高。
- 每级之间 sleep 20 是必须的。上一级产生的长请求可能还在生成，残留的 KV 会污染下一级的起点。
- --request-rate 是"目标请求速率"，客户端会用泊松过程发请求。如果服务端处理不过来，实际发出的速率会低于设定值，这一点在对比不同配置时要注意区分。
- 压测机不能和被压服务在同一台机器上，否则 CPU 抢占会直接体现在 TTFT 上。

预期看到的现象：随着 `request-rate` 上升，P50 TTFT 缓慢上升，超过某个拐点后 P95 陡增。这个拐点就是当前配置的容量上限。把每一级的 `Output token throughput` 除以 GPU 数，就得到"每卡 token 速率"，这是横向对比不同部署方案最公平的指标。

### 13.4 与容量规划的结合

有了压测基线，容量估算就有依据了。假设压测得出：单张 H100 在 TTFT P95 ≤ 1.2s 的前提下，能稳定贡献 14 QPS（输入 1024、输出 256 的混合负载）。业务侧给出的峰值是 40 QPS，要求 N+1 冗余。计算分两步：

```text
第一步：满足峰值所需的最小副本数
  ceil(40 / 14) = ceil(2.86) = 3

第二步：N+1 冗余，即任意一个副本故障后剩余副本仍能承载全部流量
  3 + 1 = 4
```

结论是 4 卡/4 副本，而不是"3 卡再打个折"。这里容易犯的错是反向算：先算冗余再算容量，或者用"平均值 × 2"拍一个数。

三个必须注明的假设：

- 单卡 QPS 贡献是特定配置下的值。换了模型、换了 max-model-len、换了 TP 大小，这个数就作废，要重新压测。
- 14 QPS 是在 TTFT P95 达标前提下取的。如果业务能接受 3s 的 TTFT，同一张卡可能能跑 30 QPS，成本能降一半。SLO 和成本是一对强耦合的变量，要一起定。
- 输入输出长度按压测设定。RAG 场景的 prompt 常常 8K 以上，同样的卡可能只能贡献 2~3 QPS。用错基线做出来的容量规划，上线一周就会被打回。

同时要留一个"突发余量"：大模型服务的流量有明显的时段性和突发性（营销活动、批处理任务同时启动），在压测基线上按峰值 × 1.3 做规划，比按平均值打两倍冗余更省钱，也更能覆盖真实的突发。留余量的方式优先选"多留一个副本"而不是"把每卡的 max-num-seqs 调大"——后者会让单副本故障时的雪崩效应放大。

## 十四、常见反对意见与回应

写到这里，有必要回应几个在评审时经常被提出的质疑，避免方案落地时反复拉扯。

**“我们的 GPU 利用率已经 95% 了，还要监控什么？”** 利用率高恰恰是最需要细分指标的状态。95% 的利用率可能是健康的批量推理，也可能是 KV 打满后的抢占空转，两者每卡 token 吞吐能差 5 倍。没有 `PROF_PIPE_TENSOR_ACTIVE` 和 `num_preemptions_total`，你无法区分。

**“Prometheus 太重量级，我们用云厂商的监控。”** 云监控通常只有 GPU 利用率和显存，缺 vLLM 引擎层指标（排队、KV、前缀缓存）。而推理服务 80% 的故障定位都发生在引擎层。可以不自建 Prometheus，但引擎层指标必须采集，至少要接 vLLM 的 `/metrics`。

**“告警太多，值班同学不看了。”** 这是告警设计的失败，不是监控的失败。三个手段：一是把 warning 全部降级到工单/日报，只有 critical 发即时消息；二是建立抑制规则，节点级故障抑制所有上层告警；三是每条告警的 annotation 里必须写清"下一步查什么"，让值班同学有动作可执行，而不是只能转给二线。

**“为什么不用 vLLM 自带的 dashboard？”** 可以直接用，但要知道它假设的指标名和版本。用之前先按 4.2 节 curl 一遍，确认指标都在；再按 7.4 节把阈值改成自己的基线。社区 dashboard 的价值是省去排版工作，不是省去理解指标的工作。

**“能不能用一个综合健康分代替这么多指标？”** 综合分会掩盖具体问题。一个健康分 85 分，你仍然不知道该降 max-model-len 还是该扩容。健康分适合给非技术管理层做汇报，技术决策必须回到分项指标。

## 十五、上线检查清单

方案落地前逐项打勾，避免遗漏：

- [ ] vLLM 版本、DCGM Exporter 版本、Prometheus 版本已记录在案，且与实际运行版本一致
- [ ] 已 curl /metrics 导出真实指标清单，dashboard 里引用的每个指标名都存在
- [ ] 直方图桶已按 SLO 调整（或不调整并注明误差范围）
- [ ] gpu_memory_utilization、max-model-len、max-num-seqs 三者的乘积关系已验证不会 OOM
- [ ] 每个告警规则都有对应 annotation 说明排查动作
- [ ] 告警已做"制造异常"验证，确认能触发
- [ ] 抑制规则覆盖节点级故障场景
- [ ] /metrics 端口已通过 NetworkPolicy 或防火墙限制来源
- [ ] 压测基线已存档，阈值推导过程有记录
- [ ] 面板 JSON 已入 Git，禁止在线手改
- [ ] 快照脚本在测试环境跑通，路径与依赖已确认
- [ ] runbook 已同步给值班同学，且至少演练过一次

这套体系跑起来之后，日常运维的工作量会明显下降——因为大多数问题在变成故障之前，就会以"TTFT P95 缓慢上升"或"前缀缓存命中率下滑"的形式出现在面板上。监控的价值不在于出事后画图，而在于让异常在变成事故之前被看见。

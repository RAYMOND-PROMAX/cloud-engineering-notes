# 大模型推理服务监控：首 Token 延迟、生成速度与请求排队

## 开篇：一个只在推理服务上出现的问题

某天上午，业务方反馈"AI 助手响应变慢了"，但监控大盘上的 5xx 错误率是 0，CPU、内存、GPU 利用率看起来都很健康。Nginx 侧的 P99 延迟涨了 4 倍，vLLM 的 `num_requests_running` 却一直贴着上限。

这就是大模型推理服务监控和传统 Web 服务监控最大的差别：传统服务慢，通常是某个下游慢或者线程池打满；推理服务慢，可能是"请求本身在排队"，而排队这件事在传统的服务级指标里根本看不见。

推理服务的一条请求，会经历这样一段路：

1. 网关和负载均衡转发过来；
2. API Server 收到请求，做鉴权、解析、分词（tokenize）；
3. 请求进入引擎的等待队列，等待调度器分配 KV Cache 块和批次预算；
4. Prefill 阶段：把 prompt 的 token 全部算一遍 KV，写入 KV Cache；
5. 吐出第一个 token（用户感知到"开始回答了"）；
6. Decode 阶段：一个 token 一个 token 地往下生成，直到 EOS 或达到 max_tokens；
7. 请求结束，释放 KV Cache 块。

用户抱怨的"慢"，可能是第 3 步（排队）、第 4 步（prefill 慢）、第 6 步（生成慢），也可能是第 2 步和第 1 步（网关、tokenizer）。这四种原因在传统 APM 的一个"HTTP 延迟"数字里被完全混在一起。

所以这篇东西要解决的核心问题是：**如何把"首 Token 延迟（TTFT）"、“生成速度（TPOT / ITL）”、"请求排队"这三件事拆开看，并且能把每一件事的变化对应到具体的处置动作。**

下面所有指标名、命令、配置都以 **vLLM V1 引擎 + Prometheus + Grafana** 为主线来写，同时给出 SGLang 的对照。文中涉及版本差异的地方我会单独标出来，落地前请务必用第 5.4 节给出的自检脚本核对一遍你实际跑的版本。

## 一、问题背景

### 1.1 为什么推理服务的监控不能照搬 Web 服务的做法

传统服务的 SLO 通常写成"请求成功率 ≥ 99.9%"、“P99 延迟 ≤ 300ms”。这套东西放到推理服务上有三个硬伤。

**第一，延迟是多段的，且用户对每一段的容忍度不同。**

一个生成 500 token 的回答，端到端延迟可能是 12 秒。如果只看端到端延迟，你根本不知道用户是在"等首字"还是在"看字慢慢出来"。对聊天类产品，用户等 3 秒看不到第一个字就会关页面；但如果首字在 0.8 秒就出来了，后面慢一点是可以接受的。所以 TTFT 和 TPOT 必须单独定 SLO。

**第二，服务端的"资源利用率高"不等于"服务正常"。**

一个把 batch 塞满、`num_requests_running` 一直等于 `max_num_seqs` 的推理实例，GPU 利用率 95%，看起来完美。但实际上它没有任何余量：新请求只能进等待队列，用户感知到的 TTFT 会随着流量线性劣化。推理服务的"健康"状态是低延迟 + 有余量，而不是高利用率。

**第三，排队是推理服务特有的现象，传统指标体系里没有对应物。**

Web 服务的排队会体现在线程池队列、连接池等待上，通常还能从框架的指标里看到。推理引擎有自己的调度器，有自己的批次预算和 KV Cache 容量约束，队列深度、排队时长、KV Cache 使用率、抢占次数，这些指标只存在于引擎内部。

### 1.2 三个核心延迟指标的定义

先把定义钉死，后面的 PromQL 和排查逻辑都基于这三个定义。

| 指标 | 全称 | 含义 | 用户感知 |
| --- | --- | --- | --- |
| TTFT | Time To First Token | 从请求到达 API Server 到吐出第一个 token 的时间 | 强感知，决定"是不是卡住了" |
| TPOT | Time Per Output Token | 每个输出 token 的平均耗时，(结束时间 - 首 token 时间) / (输出 token 数 - 1) | 中等感知，决定"流畅不流畅" |
| ITL | Inter-Token Latency | 相邻两个 token 之间的实际间隔 | 强感知，出现"卡顿停顿"就是它 |

几个必须知道的细节：

- TPOT 和 ITL 不是一回事。 TPOT 是一个请求的平均值，ITL 是逐 token 的瞬时值。一个请求的 ITL 分布可能极度不均匀（前面快、后面慢），但 TPOT 看起来很正常。判断"用户有没有感受到卡顿"要看 ITL 的高分位，判断"整体生成速度"要看 TPOT。vLLM 把两者分成两个指标：vllm:inter_token_latency_seconds 是 token 加权的 ITL，vllm:request_time_per_output_token_seconds 是每请求等权的 TPOT。
- TTFT 在 vLLM 里是从 tokenization 开始算的。 vLLM 的设计文档明确写了 TTFT 是相对于前端的 arrival_time 计算的，而 arrival_time 从开始分词起算。这意味着 TTFT 包含了分词耗时，但不包含 HTTP 层的排队（请求在 Nginx/Envoy 排队的那部分不在 TTFT 里）。跨网关对比 TTFT 时要注意这个口径差异。
- 端到端延迟 ≈ TTFT + (输出 token 数 - 1) × TPOT。 这条公式在定位"到底是哪一段慢"时非常有用，第十章会反复用。

### 1.3 排队的三种典型成因

推理引擎的等待队列不是"只有一个原因"。至少要区分这三类，它们的处置手段完全不同：

**成因 A：批次槽位不够。**`--max-num-seqs` 限制了同时运行的序列数。超出部分进等待队列。表现是 `num_requests_running` 贴着上限，`num_requests_waiting` 持续增长，但 KV Cache 使用率可能并不高。

**成因 B：token 预算被长请求吃掉。**`--max-num-batched-tokens` 限制了单个调度步内能处理的 token 总数。如果此时进来一个 32K 的超长 prompt，chunked prefill 会把它切成很多块，分多个调度步慢慢算，这期间其他短请求的 token 预算被挤占，TTFT 出现毛刺。**这是生产环境里最容易被误判为"网络抖动"的问题**，因为短请求的 TTFT 是间歇性尖刺，长请求反而正常。

**成因 C：KV Cache 不够。**`vllm:kv_cache_usage_perc` 逼近 1 时，新请求分不到块，只能等；已经运行的请求可能被**抢占**（V1 引擎的抢占是"重算"策略：释放该请求的 KV 块把它踢回等待队列，等有空间了再从头 prefill 一遍）。抢占的代价是：被抢请求的 TTFT 被推后，已经生成的 token 前面的 prefill 白做了，decode 阶段的 TPOT 出现尖刺。

还有一个"假排队"必须排除：**API Server 侧的 CPU 饱和**。tokenizer 是纯 CPU 的，如果单进程 API Server 的 CPU 已经打满，请求会在分词阶段就卡住——此时引擎里的 `num_requests_waiting` 可能是 0，但 TTFT 一样飙高。这种情况下 `num_requests_waiting` 干净得令人困惑。

## 二、适用场景与边界

### 2.1 适用场景

这套东西适用于：

- 自建 vLLM / SGLang 推理集群，用 K8s 部署，用 Prometheus + Grafana 监控；
- 对外提供 OpenAI 兼容接口的推理网关后面挂了一组推理副本；
- 已经接了 Ingress / Envoy AI Gateway / Gateway API Inference Extension，需要在网关层和引擎层做指标对齐；
- 需要给推理服务定 SLO、做容量规划、或者在故障时快速定位是"引擎慢"还是"流量分布不均"。

### 2.2 不适用 / 需要注意边界的情况

- PD 分离部署（Prefill / Decode 分离）：vLLM 的 P/D 分离模式、sglang 的 disaggregation 模式下，TTFT 里包含了 P→D 的 KV 传输时间，指标口径和单机不同。sglang 文档明确说 PD prefill worker 不计入 request_time_per_output_token_seconds。跨实例对比前要确认部署模式一致。
- 多进程 API Server（`--api-server-count > 1`）：此时 vLLM 走 prometheus_client 的多进程模式，counter 会在进程间聚合，但 python_* 和 process_* 系列指标不可用（vLLM 设计文档里明确写了这一点）。排查 CPU/内存问题时不能依赖这些指标，要用 node exporter 或 cAdvisor。
- 不是 vLLM/SGLang 的引擎：TGI 有自己的指标名（tgi_request_success、tgi_request_queue_time 等），本文的 PromQL 不能直接套用。
- 纯离线批量推理：LLM.generate() 这种批量接口没有在线请求级指标，只有批处理级别的吞吐，业务含义完全不同。

### 2.3 贯穿全文的环境假设

- vLLM：本文以 V1 引擎为基准（vLLm ≥ 0.8 默认 V1；当前 PyPI 最新为 0.32.x 系列的版本，指标名以 /metrics 实际输出为准）。
- Kubernetes：1.29+，Prometheus Operator（Prometheus CR + ServiceMonitor + PrometheusRule）。
- 采集：Prometheus 抓取 vLLM 的 /metrics（默认 HTTP 端口 8000），Grafana 作为看板与告警接收端。
- 假设模型是自托管的开源权重（vLLM / SGLang），不涉及托管厂商的专有指标。

## 三、核心知识

### 3.1 一次请求在引擎内的状态机

理解排队的关键，是搞清楚请求在 vLLM 里的状态迁移。V1 引擎的调度器维护三个队列：`waiting`（等待）、`running`（运行中）、以及被抢占后回到 `waiting` 的请求。

```text
        到达 API Server
              |
              v
     [ tokenize ]  <- arrival_time 从这里开始
              |
              v
        WAITING  <-------------------------------+
              |                                    |
     拿到批次槽位 + KV 块                          | 抢占（释放 KV，重算）
              |                                    |
              v                                    |
     [ PREFILL chunk ]  <- 可被切成多个调度步         |
              |                                    |
              v                                    |
        首个 token  -> TTFT 结束                    |
              |                                    |
              v                                    |
        [ DECODE ]  x N  ->  TPOT / ITL            |
              |                                    |
              v                                    |
          FINISHED  -------------------------------+
```

对应到 vLLM 的指标：

| 阶段 | 对应指标 | 类型 |
| --- | --- | --- |
| 分词 + 入队 | 包含在 TTFT 里，没有独立指标 | — |
| 等待调度 | vllm:request_queue_time_seconds | Histogram |
| Prefill | vllm:request_prefill_time_seconds | Histogram |
| 首 token | vllm:time_to_first_token_seconds | Histogram |
| Decode | vllm:request_decode_time_seconds | Histogram |
| 首 token → 末 token | vllm:request_inference_time_seconds | Histogram |
| 全流程 | vllm:e2e_request_latency_seconds | Histogram |
| token 间间隔 | vllm:inter_token_latency_seconds | Histogram |
| 每请求 TPOT | vllm:request_time_per_output_token_seconds | Histogram |

**关键理解：排队时长和 prefill 时长是独立的两个指标，可以相加构成 TTFT 的主体。** 排查 TTFT 时第一刀永远是"看这两个指标谁涨了"，而不是去看总延迟。

### 3.2 KV Cache 是并发上限的真正约束

这是推理服务容量规划里最容易搞错的地方。

很多人以为并发上限由 `--max-num-seqs` 决定。实际上：

- --max-num-seqs 是批次槽位上限，即"同时最多调度多少条序列"；
- 真正的物理约束是 KV Cache 块总数。KV Cache 块的总量由 --gpu-memory-utilization 扣掉权重和激活占用之后的剩余显存决定；
- 每个请求占用的块数 = ceil((prompt_len + output_len) / block_size)，其中 block_size 默认 16（以实际启动日志为准）。

所以：**并发数 ≈ KV Cache 总块数 / 单请求平均占用块数**，而不是 `--max-num-seqs`。

一个具体例子。假设 A100 80G 上跑一个 GQA 架构的 8B 模型（32 层、8 个 KV head、head_dim 128、bf16 权重）：

```text
每 token 的 KV 占用 = 2(K和V) × 32(层) × 8(KV head) × 128(head_dim) × 2(bf16 字节)
                     = 131072 字节 = 128 KiB / token

显存预算：80GiB × 0.9(--gpu-memory-utilization) = 72GiB
减去权重（8B × 2 字节 ≈ 16GiB）、激活、CUDA graph 与框架开销（约 4~6GiB）
KV Cache 可用 ≈ 50GiB = 51,200,000 KiB

KV Cache 总容量 ≈ 51,200,000 KiB ÷ 128 KiB/token ≈ 400,000 token
按 block_size = 16 分块 ≈ 25,000 个块

若平均请求占用 2000 token（1000 prompt + 1000 输出）→ 125 个块
物理并发上限 ≈ 25,000 ÷ 125 ≈ 200 条请求
```

而 `--max-num-seqs` 的常见默认值是 256 或 1024（随版本变化，用 `vllm serve --help` 确认）。**注意这里 200 < 256**：也就是说，队列长度是被 KV Cache 卡住的，而不是被批次槽位卡住的。假设监控里看到 `num_requests_waiting` 稳定在几十，而 `num_requests_running` 远没到 `max-num-seqs`，那瓶颈就在 KV Cache 侧——调大 `--max-num-seqs` 不会有任何帮助，只会让更多请求进来抢本来就紧张的显存。

> 如果换成一个 MHA 架构的模型（比如 32 层、32 个 KV head），每 token 的 KV 占用会变成 4 倍（约 512 KiB/token），物理并发上限直接掉到四分之一。**换模型 = 换容量曲线**，不要拿旧模型的容量结论套新模型。

这条推论直接决定了排查顺序：

- kv_cache_usage_perc 高 + num_requests_waiting 高 + 抢占在涨 → KV Cache 不足，需要扩显存/扩副本，或者缩短上下文。
- kv_cache_usage_perc 低 + num_requests_waiting 高 → 不是 KV 的问题，查批次槽位、token 预算、长 prefill 挤占。
- kv_cache_usage_perc 低 + num_requests_waiting 为 0 + TTFT 仍然高 → 问题不在引擎调度，去查 tokenizer / 网关 / 客户端。

### 3.3 Prefill 与 Decode 的资源画像完全不同

- Prefill 是计算密集（compute bound）：prompt 的所有 token 一次性过一遍矩阵乘，算力打满，Attention 呈平方复杂度，长 prompt 尤其贵。
- Decode 是访存密集（memory bound）：每步只算一个 token，但要把全部权重和 KV Cache 从显存读一遍，算力利用率很低，靠"多个请求共享一次权重读取"（即 batching）来提升吞吐。

这个差异带来两个实际影响：

1. 单个超长 prompt 会阻塞整个 batch。 即使开了 chunked prefill，长 prompt 也会占用多个调度步的 token 预算，同批次里的短请求被迫等待。这解释了"TTFT 偶尔尖刺但整体正常"的现象。
2. Prefill/Decode 分离（PD disaggregation）就是为了解决这个矛盾：prefill 节点负责算 prompt，decode 节点负责逐 token 生成，两类资源独立扩缩。代价是引入 KV 传输延迟，TTFT 口径会变。

### 3.4 Prefix Cache：TTFT 的第一个优化杠杆

启用前缀缓存（vLLM V1 默认开启）后，多次请求共享的相同前缀可以直接复用 KV，跳过这部分 prefill。

- vllm:prefix_cache_queries：本次查询的前缀 token 总数；
- vllm:prefix_cache_hits：其中命中缓存的 token 总数；
- 命中率 = hits / queries（token 级）。

命中率高 → prefill 成本被摊薄 → TTFT 下降；命中率低 → 每次请求都要完整 prefill → TTFT 直接等于 prefill 时长。

对 RAG、Few-shot、多轮对话这类"前缀高度重复"的业务，命中率能从 0 拉到 0.6 以上，TTFT 通常能降一半以上。反过来，如果一个业务场景下命中率长期接近 0，**说明前缀缓存没起作用**——要么请求前缀确实不重复（比如每次重新组装 system prompt 且顺序不定），要么多副本部署导致请求被路由到没有缓存的副本上（这需要路由亲和来解决，见第十章场景 E）。

## 四、监控体系总览：三层结构

不要一上来就配 Prometheus。先把"谁回答什么问题"定清楚。

```text
┌─────────────────────────────────────────────────────────────┐
│ 第 3 层 业务 SLO 层                                            │
│  TTFT p95 ≤ 1.2s / TPOT p95 ≤ 80ms / 成功率 ≥ 99.5%          │
│  指标来源：引擎 histogram + 网关 5xx + 客户端侧抽样              │
│  消费方：产品、业务方、SLO 周报、错误预算                        │
└───────────────────────────┬─────────────────────────────────┘
                            │  recording rule 聚合
┌───────────────────────────▼─────────────────────────────────┐
│ 第 2 层 引擎状态层                                            │
│  排队深度 / 排队时长 / KV Cache / 抢占 / 吞吐 / 批大小          │
│  指标来源：vllm:* 与 sglang:* /metrics                        │
│  消费方：值班工程师、容量规划、扩缩容决策                        │
└───────────────────────────┬─────────────────────────────────┘
                            │  服务发现 + 抓取
┌───────────────────────────▼─────────────────────────────────┐
│ 第 1 层 基础设施层                                             │
│  GPU 利用率 / 显存 / 时钟节流 / 温度 / 节点网络 / CPU            │
│  指标来源：DCGM exporter / node exporter / kube-state-metrics   │
│  消费方：硬件与平台团队                                          │
└─────────────────────────────────────────────────────────────┘
```

三层之间的关系是**自上而下收敛、自下而上排除**：

- 业务反馈"慢" → 在第 3 层确认是 TTFT 还是 TPOT 退化；
- 打开第 2 层，看排队时长 / prefill 时长 / KV Cache / 抢占，找到引擎内部的原因；
- 如果第 2 层一切正常，就下沉到第 1 层和客户端/网关侧；
- 如果第 1 层也正常，问题在流量分布（比如请求打到了冷副本）或者客户端。

**最常见的错误是跳过第 2 层直接看 GPU 利用率。** GPU 利用率 95% 既可能是"健康的满载"，也可能是"一个超长 prefill 把整个 batch 拖死"，两种状态的处置方式完全相反。

### 4.1 采集与展示的三条原则

**原则一：延迟用 histogram，不用 gauge 存当前值。**引擎给你的是 histogram，这是好事——histogram 可以算任意分位数，也能算"超过某个阈值的请求占比"。前提是桶边界要覆盖你的 SLO 区间（见第 7.7 节）。

**原则二：队列深度和 KV Cache 使用率必须和延迟指标画在同一张图上。**单看 TTFT 曲线无法判断原因；把 `num_requests_waiting` 叠在 TTFT 下面，才能看出"延迟涨是因为排队"。

**原则三：延迟指标要按实例（instance）画，不要只画全局聚合。**全局 p95 可能掩盖"一个副本全坏"的情况。按实例画，再叠加 `max`/`min`，一眼就能看出负载不均。

## 五、vLLM 指标详解与版本核对

### 5.1 抓取入口与基本形态

vLLM 的 Prometheus 指标通过 OpenAI 兼容 API Server 的 `/metrics` 端点暴露，默认 HTTP 端口 8000：

```bash
# 本地快速验证端点是否可访问
curl -s http://127.0.0.1:8000/metrics | head -n 20

# 只看 TTFT 相关的直方图桶，确认桶边界
curl -s http://127.0.0.1:8000/metrics | grep '^vllm:time_to_first_token_seconds_bucket'
```

输出形态大致如下：

```text
# HELP vllm:time_to_first_token_seconds Histogram of time to first token in seconds.
# TYPE vllm:time_to_first_token_seconds histogram
vllm:time_to_first_token_seconds_bucket{le="0.3",model_name="Qwen/Qwen3-8B"} 12.0
vllm:time_to_first_token_seconds_bucket{le="0.5",model_name="Qwen/Qwen3-8B"} 41.0
vllm:time_to_first_token_seconds_bucket{le="1.0",model_name="Qwen/Qwen3-8B"} 96.0
vllm:time_to_first_token_seconds_bucket{le="2.0",model_name="Qwen/Qwen3-8B"} 110.0
vllm:time_to_first_token_seconds_bucket{le="+Inf",model_name="Qwen/Qwen3-8B"} 110.0
vllm:time_to_first_token_seconds_sum{model_name="Qwen/Qwen3-8B"} 84.31
vllm:time_to_first_token_seconds_count{model_name="Qwen/Qwen3-8B"} 110.0
```

注意：**counter 类型指标在 Prometheus 的文本暴露格式里会自动加上 _total 后缀**。文档里的表格写 `vllm:generation_tokens`，但在 `/metrics` 里你看到的是 `vllm:generation_tokens_total`，在 PromQL 里也应该写 `vllm:generation_tokens_total`。这是新手最常踩的坑之一——写 `vllm:generation_tokens` 的 `rate()` 会返回空。

### 5.2 监控三件套：TTFT、生成速度、排队

| 关注点 | 指标 | 类型 | 说明 |
| --- | --- | --- | --- |
| 首 token 延迟 | vllm:time_to_first_token_seconds | Histogram | 含 tokenize + 排队 + prefill |
| 生成速度 | vllm:request_time_per_output_token_seconds | Histogram | 每请求等权 TPOT |
| 生成速度 | vllm:inter_token_latency_seconds | Histogram | token 加权 ITL，看卡顿 |
| 排队深度 | vllm:num_requests_waiting | Gauge | 等待队列长度 |
| 排队深度 | vllm:num_requests_running | Gauge | 正在执行的批内请求数 |
| 排队原因 | vllm:num_requests_waiting_by_reason | Gauge | capacity / deferred |
| 排队时长 | vllm:request_queue_time_seconds | Histogram | 入队到首次被调度 |
| KV Cache | vllm:kv_cache_usage_perc | Gauge | 0~1，1 表示 100% |
| 抢占 | vllm:num_preemptions | Counter | 累计抢占次数（V1 为重算式抢占） |
| 抢占 | vllm:request_num_preemptions | Histogram | 每请求被抢占次数 |
| 前缀缓存 | vllm:prefix_cache_hits / vllm:prefix_cache_queries | Counter | token 级命中/查询 |
| 已缓存 prompt | vllm:prompt_tokens_cached | Counter | 复用的 prompt token 数 |
| 吞吐 | vllm:generation_tokens | Counter | 生成 token 总数 |
| 吞吐 | vllm:prompt_tokens | Counter | prefill token 总数 |
| 准入拒绝 | vllm:admission_rejections | Counter | 被准入限制拒绝的次数（按 reason） |
| 请求结果 | vllm:request_success | Counter | 带 finished_reason 标签：stop/length/abort |
| 批次效率 | vllm:iteration_tokens_total | Histogram | 每个调度步产出的 token 数 |
| 实例配置 | vllm:cache_config_info | Gauge | 值为 1，标签里带启动参数 |
| 睡眠模式 | vllm:engine_sleep_state | Gauge | 0 睡眠 / 1 清醒 |

`vllm:cache_config_info` 这个指标非常有用，它把实例的关键启动参数变成了 Prometheus 标签：

```text
# HELP vllm:cache_config_info Information of the LLMEngine CacheConfig
# TYPE vllm:cache_config_info gauge
vllm:cache_config_info{block_size="16",cache_dtype="auto",cpu_offload_gb="0",
  enable_prefix_caching="True",gpu_memory_utilization="0.9",
  max_num_batched_tokens="8192",max_num_seqs="1024",max_model_len="32768",...} 1.0
```

排查问题时可以直接拿它确认"这个副本的启动参数是不是和我以为的一致"。灰度期间尤其有用——新旧两批副本参数不同，指标却长得一模一样，很容易误判。

### 5.3 SGLang 指标对照

SGLang 需要显式打开 `--enable-metrics`，默认端口 30000：

```bash
python -m sglang.launch_server \
  --model-path <model_path> \
  --port 30000 \
  --enable-metrics \
  --enable-mfu-metrics
```

| 关注点 | vLLM | SGLang |
| --- | --- | --- |
| 首 token 延迟 | vllm:time_to_first_token_seconds | sglang:time_to_first_token_seconds |
| 端到端延迟 | vllm:e2e_request_latency_seconds | sglang:e2e_request_latency_seconds |
| 每请求 TPOT | vllm:request_time_per_output_token_seconds | sglang:request_time_per_output_token_seconds |
| token 级 ITL | vllm:inter_token_latency_seconds | sglang:inter_token_latency_seconds |
| 运行中请求 | vllm:num_requests_running | sglang:num_running_reqs |
| 排队深度 | vllm:num_requests_waiting | sglang:num_queue_reqs |
| token 吞吐 | vllm:generation_tokens（rate） | sglang:gen_throughput（gauge，token/s） |
| KV / token 使用 | vllm:kv_cache_usage_perc | sglang:token_usage（gauge） |
| 前缀缓存命中率 | 由 hits/queries 计算 | sglang:cache_hit_rate（gauge） |
| MFU 估算 | vllm:estimated_flops_per_gpu_total 等 | sglang:estimated_flops_per_gpu_total 等 |
| 服务函数耗时 | — | sglang:func_latency_seconds{name="generate_request"} |

两点需要注意：

1. SGLang 的 sglang:gen_throughput 是当前瞬时吞吐的 gauge，不是 counter，算历史平均要用它的 rate 之外的方式；而 vLLM 侧要自己用 rate(vllm:generation_tokens_total[5m]) 算。这两个在写统一看板时容易对不上。
2. SGLang 的 sglang:cache_hit_rate 是 gauge，语义是"最近若干次查询的命中率"，具体窗口以 SGLang 实现为准，不等价于 vLLM 的 token 级比率。做跨引擎对比时不要直接拿这两个数相减。

### 5.4 启动前自检：确认你的版本到底有哪些指标

指标名会随版本变化。vLLM 官方有明确的废弃策略：某个版本标记废弃，下个版本隐藏（可用 `--show-hidden-metrics-for-version=X.Y` 临时打开），再下个版本删除。**不要凭记忆写 PromQL，写完先用脚本核对。**

下面这个脚本会检查你依赖的指标是否存在，并把缺失项单独列出来：

```bash
#!/usr/bin/env bash
# check_vllm_metrics.sh —— 核对目标实例暴露的指标是否覆盖监控依赖
# 用法：
#   VLLM_ENDPOINT=http://127.0.0.1:8000/metrics ./check_vllm_metrics.sh
# 退出码：0=全部存在；2=存在缺失项（缺失清单打印到 stdout）
set -o pipefail

ENDPOINT="${VLLM_ENDPOINT:-http://127.0.0.1:8000/metrics}"

if [[ -z "${ENDPOINT}" ]]; then
echo"ERROR: VLLM_ENDPOINT 未设置" >&2
exit 1
fi

METRICS="$(curl -sS --max-time 10 --fail "${ENDPOINT}" 2>/dev/null)"
if [[ -z "${METRICS}" ]]; then
echo"ERROR: 无法从 ${ENDPOINT} 拉取指标，检查端点/网络/端口" >&2
exit 1
fi

REQUIRED=(
"vllm:time_to_first_token_seconds"
"vllm:request_time_per_output_token_seconds"
"vllm:inter_token_latency_seconds"
"vllm:e2e_request_latency_seconds"
"vllm:request_queue_time_seconds"
"vllm:request_prefill_time_seconds"
"vllm:num_requests_running"
"vllm:num_requests_waiting"
"vllm:kv_cache_usage_perc"
"vllm:num_preemptions"
"vllm:prefix_cache_hits"
"vllm:prefix_cache_queries"
"vllm:generation_tokens"
"vllm:request_success"
)

missing=0
for m in"${REQUIRED[@]}"; do
# counter 在文本格式下会带 _total 后缀，两种都算命中
if grep -qE "^# (HELP|TYPE) ${m}( |$)" <<<"${METRICS}" \
     || grep -qE "^${m}(_total)?[{]?" <<<"${METRICS}"; then
    printf'OK   %s\n'"${m}"
else
    printf'MISS %s\n'"${m}"
    missing=$((missing + 1))
fi
done

echo"----"
if (( missing > 0 )); then
echo"缺失 ${missing} 项。请执行 vllm serve --help / 查看官方 metrics 文档确认本版本命名。"
echo"若为旧版本名称差异（例如 kv_cache_usage_perc 在旧版为 gpu_cache_usage_perc），"
echo"在 Prometheus 侧用 recording rule 做别名兼容，而不是改采集端。"
exit 2
fi
echo"全部指标齐备。"
exit 0
```

几个用法上的说明：

- 脚本默认只读，不做任何变更，可以放在 CI 或者发布流水线里跑；
- curl --max-time 10 防止端点假死时把脚本挂住；
- set -o pipefail 保证管道里 curl 失败时不会因为 grep 返回非零而误判；
- 缺失指标的兼容策略写在 PromQL 侧（recording rule 里做 or 兜底），不要改 vLLM 的暴露格式——引擎不是你自己的代码。

### 5.5 建议打开的引擎侧参数

有三个参数会直接影响可观测性，建议在生产启动参数里明确写死：

```bash
vllm serve Qwen/Qwen3-8B \
  --host 0.0.0.0 \
  --port 8000 \
  --enable-prefix-caching \
  --enable-mfu-metrics \
  --custom-histogram-buckets '{"time_to_first_token":[0.1,0.2,0.3,0.5,0.8,1.2,2.0,3.0,5.0],"inter_token_latency":[0.01,0.02,0.03,0.05,0.08,0.12,0.2,0.4]}' \
  --log-config-file /etc/vllm/logging.json
```

- --enable-mfu-metrics 打开 vllm:estimated_flops_per_gpu_total / estimated_read_bytes_per_gpu_total / estimated_write_bytes_per_gpu_total，可以推导 MFU 和显存带宽利用率，判断"GPU 到底是被算力限制还是被带宽限制"。注意官方文档明确说这些是模型化估算值，不是硬件计数器。
- --custom-histogram-buckets 调整桶边界，让直方图的分位数在你的 SLO 区间里有足够分辨率。这个参数会覆盖对应指标族的默认桶，只影响你列出的族，未列出的族保持默认。桶边界必须为正、有限、严格递增，否则启动直接报错。
- --log-config-file 让日志级别和格式可控，排查时需要能临时打开 debug。

> **风险提示**：`--custom-histogram-buckets` 需要重启引擎才生效，属于生产变更项；桶越多，Prometheus 的时间序列数量和存储成本线性上涨。桶列表建议不超过 8~10 个边界。

自定义桶的族名（family key）与对应直方图的关系：

| 族名 | 覆盖的直方图 |
| --- | --- |
| request_latency | e2e_request_latency_seconds、request_queue_time_seconds、request_inference_time_seconds、request_prefill_time_seconds、request_decode_time_seconds |
| time_to_first_token | time_to_first_token_seconds |
| inter_token_latency | inter_token_latency_seconds、request_time_per_output_token_seconds |
| iteration_tokens | iteration_tokens_total |
| request_params_n | request_params_n |
| request_num_preemptions | request_num_preemptions |
| request_tokens | request_prompt_tokens、request_generation_tokens、request_max_num_generation_tokens、request_params_max_tokens、request_prefill_kv_computed_tokens |
| kv_cache_residency | kv_block_lifetime_seconds、kv_block_idle_before_evict_seconds、kv_block_reuse_gap_seconds |

`kv_cache_residency` 族需要额外开启 `--kv-cache-metrics` 才能拿到 KV 块的驻留时长、驱逐前空闲时长、复用间隔；采样频率由 `--kv-cache-metrics-sample` 控制（采样开启时开销极小）。这三个指标对"prefix cache 到底有没有在复用"是直接证据。

另外两点：

- 前缀缓存在 V1 引擎默认开启。如果你的服务没生效，先确认启动参数里没有显式关掉它（关闭开关的名称随版本变化，用 vllm serve --help 确认），再去看命中率指标；
- --enable-prefix-caching 之外，跨副本共享前缀缓存需要额外的 KV Connector（如 NIXL / HiSparse），那属于另一套部署形态，指标名也不同（见 vLLM 官方 metrics 文档中的 NIXL / HiSparse Connector 指标小节）。

## 六、采集落地

### 6.1 Docker 部署时的最小可用形态

先在单机把采集链路走通，再上 K8s。最小形态是三个容器：vLLM、Prometheus、Grafana。

```yaml
# docker-compose.yml（最小验证用，非生产配置）
# 镜像版本通过 .env 固定，${VAR:?msg} 表示未设置时直接报错退出
services:
vllm:
    image:${VLLM_IMAGE:-vllm/vllm-openai:latest}
    command:>
      --model Qwen/Qwen3-8B
      --host 0.0.0.0
      --port 8000
      --enable-prefix-caching
      --enable-mfu-metrics
    ports:
      -"8000:8000"
    deploy:
      resources:
        reservations:
          devices:
            -driver:nvidia
              count:1
              capabilities: [gpu]
    environment:
      HF_HOME:/data/hf
    volumes:
      -/data/hf:/data/hf

prometheus:
    image:${PROMETHEUS_IMAGE:?请在.env中固定Prometheus镜像tag}
    volumes:
      -./prometheus.yml:/etc/prometheus/prometheus.yml:ro
      -./prom-data:/prometheus
    ports:
      -"9090:9090"
    command:
      ---config.file=/etc/prometheus/prometheus.yml
      ---storage.tsdb.path=/prometheus
      ---storage.tsdb.retention.time=15d

grafana:
    image:${GRAFANA_IMAGE:?请在.env中固定Grafana镜像tag}
    volumes:
      -./grafana/provisioning:/etc/grafana/provisioning:ro
      -./grafana/dashboards:/var/lib/grafana/dashboards:ro
    ports:
      -"3000:3000"
    environment:
      GF_SECURITY_ADMIN_PASSWORD:${GRAFANA_ADMIN_PASSWORD:?必须在.env中设置}
```

> **镜像版本必须固定。**`latest` 在生产里是不可接受的：一次上游构建就可能把引擎升级到不兼容的版本，而且回滚时你拿不到旧镜像。正确做法是把验证过的 tag 推到内网镜像仓库，`${VAR:?msg}` 语法保证忘记设置时 compose 直接失败而不是静默拉取最新。**凭证处理**：`GRAFANA_ADMIN_PASSWORD` 用环境变量注入，`.env` 文件加入 `.gitignore`，生产环境改用密钥管理系统注入。不要把密码写进 compose 文件或镜像层。

对应的抓取配置：

```yaml
# prometheus.yml
global:
scrape_interval:15s
scrape_timeout:10s
evaluation_interval:15s
external_labels:
    cluster:prod-shanghai
    env:prod

scrape_configs:
-job_name:vllm
    metrics_path:/metrics
    scheme:http
    static_configs:
      -targets:
          -"vllm:8000"
        labels:
          model:"Qwen/Qwen3-8B"
```

注意 `scrape_timeout` 必须小于 `scrape_interval`，否则 Prometheus 会报 “scrape timeout larger than scrape interval”，抓取会不稳定。

启动与验证：

```bash
# 1. 拉起服务
docker compose up -d

# 2. 确认 Prometheus 目标状态为 UP
curl -s http://127.0.0.1:9090/api/v1/targets | python -m json.tool | head -n 40

# 3. 直接查一个即时表达式，确认指标已被采集
curl -sG http://127.0.0.1:9090/api/v1/query \
  --data-urlencode 'query=vllm:num_requests_waiting'

# 4. 确认 Grafana 数据源可用
curl -s http://127.0.0.1:3000/api/health
```

第 3 步如果返回 `{"status":"success","data":{"result":[]}}`，说明指标名写错了或者端点没抓到。返回空数组和返回错误值要能区分开——前者是命名问题，后者是采集问题。

### 6.2 K8s + Prometheus Operator：ServiceMonitor

生产环境推荐 Operator 方式，因为目标发现、标签管理、告警规则都变成声明式的。

先说一个最容易踩的坑：**ServiceMonitor 必须放在 Prometheus CR 选中的命名空间里。** 默认 kube-prometheus-stack 的 `serviceMonitorSelector` 只匹配 release 所在命名空间（且带 `release=<helm-release-name>` 标签），`serviceMonitorNamespaceSelector` 为空。你把 ServiceMonitor 扔到 `vllm` 命名空间里，Prometheus 根本看不到，Target 列表里也不会出现。

正确做法（kube-prometheus-stack 场景，release 名 `prometheus`，位于 `monitoring` 命名空间）：

```yaml
# monitoring/vllm-servicemonitor.yaml
apiVersion:monitoring.coreos.com/v1
kind:ServiceMonitor
metadata:
name:vllm
namespace:monitoring          # 与 Prometheus CR 同一命名空间
labels:
    release:prometheus          # 匹配 helm release 标签，否则不会被选中
spec:
namespaceSelector:
    matchNames:
      -llm                     # vLLM 服务所在命名空间
selector:
    matchLabels:
      app.kubernetes.io/name:vllm
endpoints:
    -port:http                 # 必须与 Service 中暴露的端口 name 一致
      path:/metrics
      interval:15s
      scrapeTimeout:10s
      honorLabels:true
      jobLabel:app.kubernetes.io/name   # job 标签取 Service 上该标签的值（这里即 vllm）
      relabelings:
        -sourceLabels: [__meta_kubernetes_pod_name]
          targetLabel:pod
        -sourceLabels: [__meta_kubernetes_pod_node_name]
          targetLabel:node
      podTargetLabels:
        -app.kubernetes.io/instance     # 把 Pod 标签复制成指标标签
```

> `jobLabel` 的作用：没有它时 `job` 标签由 Operator 按 Service 命名规则生成，各版本行为不完全一致；显式指定后 `job` 标签稳定等于 `vllm`，第九章的 `up{job="vllm"}` 告警才能可靠匹配。`jobLabel` 引用的是 **Service** 上的标签，所以 Service 必须有 `app.kubernetes.io/name: vllm`。`podTargetLabels` 与 `targetLabels` 的区别：前者从 **Pod** 复制标签，后者从 **Service** 复制。两者都会把标签附加到抓取上来的所有指标上，是做多副本下钻的关键。**不要用 metricRelabelings 删掉 model_name。** 它是引擎自带的模型维度标签，是看板分组和告警路由的依据；真要控制基数，应该限制"一个实例只服务一个模型"，而不是在采集侧把维度删掉。

Service 必须有一个 **name 为 http** 的端口，否则 `port: http` 匹配不到：

```yaml
# llm/vllm-service.yaml
apiVersion:v1
kind:Service
metadata:
name:vllm
namespace:llm
labels:
    app.kubernetes.io/name:vllm
spec:
type:ClusterIP
selector:
    app.kubernetes.io/name:vllm
ports:
    -name:http
      port:8000
      targetPort:8000
      protocol:TCP
```

> `honorLabels: true` 的作用：允许被采集对象自带的标签覆盖目标标签。vLLM 的每个指标都带 `model_name`，不开启的话在某些 relabel 组合下会被改写。如果你的 relabel 规则里没有用 `model_name` 作为目标标签，其实不需要开——按需设置。

**安全要求：metrics 端点不应该对整个集群开放。** vLLM 的 `/metrics` 和 API 走同一个端口，通常没有独立的鉴权。正确的做法是用 NetworkPolicy 限制只有监控命名空间的 Prometheus 能访问：

```yaml
# llm/allow-metrics-scraping.yaml
apiVersion:networking.k8s.io/v1
kind:NetworkPolicy
metadata:
name:vllm-allow-scrape
namespace:llm
spec:
podSelector:
    matchLabels:
      app.kubernetes.io/name:vllm
policyTypes: [Ingress]
ingress:
    -from:
        -namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name:monitoring
      ports:
        -protocol:TCP
          port:8000
```

> **注意**：这条策略会把该命名空间内**所有**到 8000 端口的入站流量都挡掉（包括业务侧调用）。如果你有业务网关也要访问 8000，需要额外加一条 `from` 规则。NetworkPolicy 的 `podSelector` + `ports` 是叠加生效的（AND 关系），`from` 列表项之间是 OR 关系。

### 6.3 非 Operator 场景：用 Kubernetes 服务发现

如果没有 Prometheus Operator，用原生服务发现也可以：

```yaml
scrape_configs:
  -job_name:vllm
    metrics_path:/metrics
    scrape_interval:15s
    kubernetes_sd_configs:
      -role:pod
        namespaces:
          names: [llm]
    relabel_configs:
      -source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action:keep
        regex:"true"
      -source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape_port]
        action:replace
        target_label:__address__
        regex:(.+)
        replacement:$1
      -source_labels: [__meta_kubernetes_pod_name]
        target_label:pod
      -source_labels: [__meta_kubernetes_pod_node_name]
        target_label:node
      -source_labels: [__meta_kubernetes_pod_label_app_kubernetes_io_name]
        target_label:app
```

对应 Pod 上的注解：

```yaml
metadata:
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/scrape_port: "8000"
    prometheus.io/path: "/metrics"
```

### 6.4 GPU 侧指标的接入

只看 vLLM 指标是不够的。TTFT 恶化有时是 GPU 降频、有时是显存带宽被抢占，必须有硬件层证据。

DCGM Exporter 是标准选择，它以 DaemonSet 形式跑在每个 GPU 节点上，暴露 9400 端口。

> **镜像版本必须与节点上的 DCGM 驱动兼容。** dcgm-exporter 的 tag 形如 `<exporter 版本>-<支持的 DCGM 版本>-<基础镜像>`，随便挑一个会在启动时拿不到 GPU 字段。部署前先在目标节点上确认 DCGM 版本：`nvidia-smi -q | grep -i "DCGM"`，或者 `cat /usr/local/dcgm/dcgm_version.txt`（版本不同路径略有差异），再选对应的 exporter tag。下面的 YAML 里的 `<VERSION>` 是占位符，不替换直接 apply 会失败——这是故意的。

```yaml
# monitoring/dcgm-exporter.yaml（要点片段）
apiVersion:apps/v1
kind:DaemonSet
metadata:
name:dcgm-exporter
namespace:monitoring
labels:
    app.kubernetes.io/name:dcgm-exporter
spec:
selector:
    matchLabels:
      app.kubernetes.io/name:dcgm-exporter
template:
    metadata:
      labels:
        app.kubernetes.io/name:dcgm-exporter
      annotations:
        prometheus.io/scrape:"true"
        prometheus.io/port:"9400"
    spec:
      serviceAccountName:dcgm-exporter
      containers:
        -name:exporter
          image:nvcr.io/nvidia/k8s/dcgm-exporter:<VERSION>   # 占位符，必须替换为与节点 DCGM 驱动兼容的 tag
          args:
            ---kubernetes
            ---dcgm-root=/var/run/dcgm
            -# 如需 profiling 类指标（tensor 活跃度）再打开
              # - --enable-profile
          env:
            -name:DCGM_EXPORTER_LISTEN
              value:":9400"
            -name:DCGM_EXPORTER_KUBERNETES
              value:"true"
          resources:
            limits:
              nvidia.com/gpu:1
          securityContext:
            capabilities:
              add: [SYS_ADMIN]
            privileged:true
          volumeMounts:
            -name:dcgm
              mountPath:/var/run/dcgm
      volumes:
        -name:dcgm
          hostPath:
            path:/var/run/dcgm
```

DCGM 的指标名在不同版本里略有差异，**不要凭记忆写**，先在节点上列一遍：

```bash
# 列出 exporter 实际暴露的指标
kubectl -n monitoring port-forward svc/dcgm-exporter 9400:9400 &
curl -s http://127.0.0.1:9400/metrics | grep -E '^DCGM_FI' | cut -d'{' -f1 | sort -u
```

常用的几类（名字以实际输出为准）：

- DCGM_FI_DEV_GPU_UTIL：GPU 利用率百分比；
- DCGM_FI_DEV_MEM_COPY_UTIL：显存拷贝利用率；
- DCGM_FI_DEV_FB_USED / FB_FREE / FB_TOTAL：显存；
- DCGM_FI_DEV_SM_CLOCK_THROTTLE_REASONS（带 reason 标签，值为 0/1）、DCGM_FI_DEV_THERMAL_THROTTLE_REASONS、DCGM_FI_DEV_POWER_THROTTLE_REASONS：各类节流原因；
- DCGM_FI_PROF_PIPE_TENSOR_ACTIVE：Tensor Core 活跃度（需要 profiling 指标开启）；
- DCGM_FI_PROF_GR_ENGINE_ACTIVE：图形/计算引擎活跃度。

**节流指标的正确用法**：`DCGM_FI_DEV_SM_CLOCK_THROTTLE_REASONS` 带一个 `reason` 标签，值为 0/1，指标值大于 0 表示在采样窗口内发生过该原因的限频。查询"当前是否被限频"：

```promql
max by (instance, reason) (DCGM_FI_DEV_SM_CLOCK_THROTTLE_REASONS) > 0
```

判定时要注意两点：

1. reason 的编码在不同 DCGM 版本里不一致（有的版本用数字序号，有的用语义字符串），必须先 `curl .../metrics | grep THROTTLE` 看清实际有哪些 reason 值再下结论，否则你会把"限频原因 1"误读成"某个具体原因"；
2. 该指标是采样窗口内的统计，不等于"此刻正在限频"。要确认是持续性降频，还得配合当前 SM 频率与最大 SM 频率的比值来看（指标名以实际输出为准，常见为 DCGM_FI_DEV_SM_CLOCK 与 DCGM_FI_DEV_MAX_SM_CLOCK）：

```promql
DCGM_FI_DEV_SM_CLOCK / DCGM_FI_DEV_MAX_SM_CLOCK
```

这个比值持续显著小于 1，说明频率被压住了。用之前先 `curl .../metrics | grep -E 'SM_CLOCK'` 确认名字。

### 6.5 采集侧的三个坑

**坑一：抓取间隔决定分位数精度。** 抓取间隔 15s 意味着任何在 15 秒内发生并恢复的延迟毛刺（尖刺）都不会被记录。延迟指标的抓取间隔建议 5s~15s，但不要低于 5s——vLLM 的 `/metrics` 是同步渲染的，抓太频繁本身会占用 API Server 的 CPU 和 GIL 锁。

**坑二：/metrics 端点的渲染开销。** 指标数量多（尤其开了自定义桶、KV cache 采样）时，`/metrics` 的响应体可以到几百 KB，抓取本身会占用 CPU。可以在压测时对比开关监控前后的吞吐差异。

**坑三：Pod 重启导致 counter 归零。**`rate()` 和 `increase()` 能正确处理 counter 重置，但 `sum(vllm:generation_tokens_total)` 这种写法在重启后会突然变小。做累计统计一定要用 `increase()`。

## 七、PromQL 实战

### 7.1 首 Token 延迟（TTFT）分位数

```promql
# P95 TTFT，按实例维度
histogram_quantile(
  0.95,
  sum by (le, instance) (
    rate(vllm:time_to_first_token_seconds_bucket[5m])
  )
)

# P50 / P95 / P99 一次算全
histogram_quantile(0.99,
  sum by (le, instance) (rate(vllm:time_to_first_token_seconds_bucket[5m]))
)
```

工程上有几个细节：

- `[5m]` 窗口不是随便写的。 窗口太短（比如 [1m]）在低 QPS 下样本不足，分位数抖动大；太长（比如 [1h]）会让故障响应变慢，SLO 告警失去意义。常规做法是快速窗口 5m 用于展示与快速告警，长窗口 1h 用于慢速告警和趋势。
- 必须保留 `instance` 分组。 直接 sum by (le) 会把所有副本混在一起算分位数，"一个坏副本"这种情况会被平均掉。
- 如果要多副本看全局分位数，用 sum by (le) (rate(...)) 合并所有实例再算——但这是"所有请求混在一起"的分位数，和"每个实例各自的分位数再取平均"不是一回事，两者可以差很多。

### 7.2 生成速度：TPOT 与 ITL

```promql
# 每请求等权的 TPOT P95
histogram_quantile(
  0.95,
  sum by (le, instance) (
    rate(vllm:request_time_per_output_token_seconds_bucket[5m])
  )
)

# 平均 TPOT（mean ≈ sum/count），用于换算 tok/s
sum by (instance) (rate(vllm:request_time_per_output_token_seconds_sum[5m]))
  /
sum by (instance) (rate(vllm:request_time_per_output_token_seconds_count[5m]))

# token 加权的 ITL P95（用来看卡顿）
histogram_quantile(
  0.95,
  sum by (le, instance) (
    rate(vllm:inter_token_latency_seconds_bucket[5m])
  )
)

# 单请求解码速度的倒数关系：TPOT 越小 tok/s 越大（这里是集群平均值口径）
1 / (
  sum(rate(vllm:request_time_per_output_token_seconds_sum[5m]))
  /
  sum(rate(vllm:request_time_per_output_token_seconds_count[5m]))
)
```

**这里有一个非常容易搞错的点**：P95 TPOT 的倒数**不等于** P95 tok/s。正确的关系是"每请求解码速度的 P5 ≈ 1 / P95 TPOT"，但如果你想看"整个集群的 tok/s"，应该直接用吞吐指标（下一节），不要用分位数做算术。

另外注意上面最后一个表达式算的是**集群整体的平均 TPOT 倒数**，它不等于"平均 tok/s"：不同请求的输出长度不同，TPOT 是按请求算的，再取倒数和按 token 加权平均是两回事。要看集群真实吞吐，请用 `sum(rate(vllm:generation_tokens_total[5m]))`。

### 7.3 吞吐

```promql
# 生成 token 吞吐（tok/s），集群总和
sum(rate(vllm:generation_tokens_total[5m]))

# prompt token 处理吞吐（tok/s）
sum(rate(vllm:prompt_tokens_total[5m]))

# 每实例生成吞吐
sum by (instance) (rate(vllm:generation_tokens_total[5m]))

# 请求完成速率（QPS），用 e2e 直方图的 count 更可靠
sum(rate(vllm:e2e_request_latency_seconds_count[5m]))

# 按结束原因拆分（stop=自然结束，length=撞 max_tokens，abort=被中断）
sum by (finished_reason) (rate(vllm:request_success_total[5m]))
```

`finished_reason="length"` 的占比是个容易被忽略但很有价值的业务指标：这个比例突然升高，通常意味着上游改了 `max_tokens` 参数，或者模型开始输出复读/超长内容。它不是性能问题，但它会让总输出 token 数暴涨，进而把 KV Cache 和队列全部打满——**这正是"业务变更伪装成性能故障"的经典案例**。

```promql
# 撞上 max_tokens 的比例
sum(rate(vllm:request_success_total{finished_reason="length"}[5m]))
  /
sum(rate(vllm:request_success_total[5m]))
```

### 7.4 排队：队列深度与排队时长

```promql
# 等待队列深度
vllm:num_requests_waiting

# 正在执行的请求数
vllm:num_requests_running

# 按原因拆分等待队列（capacity=等待调度容量，deferred=被暂缓）
vllm:num_requests_waiting_by_reason

# 排队时长 P95（判断"排了多久"）
histogram_quantile(
  0.95,
  sum by (le, instance) (
    rate(vllm:request_queue_time_seconds_bucket[5m])
  )
)

# 排队时长的均值（比分位数更抗噪，适合做容量基线）
sum by (instance) (rate(vllm:request_queue_time_seconds_sum[5m]))
  /
sum by (instance) (rate(vllm:request_queue_time_seconds_count[5m]))
```

**队列深度是一个瞬时量，非常容易被采样时机误导。** 队列在两个请求到达之间会瞬间清空，15 秒一次的抓取很容易错过尖峰。判断"是否真的在排队"要同时看**排队时长分位数**——排队时长是每个请求的完整经历，不会因为采样时机而丢失。

### 7.5 KV Cache 与抢占

```promql
# KV Cache 使用率
vllm:kv_cache_usage_perc

# 抢占速率（次/秒）
sum(rate(vllm:num_preemptions_total[5m]))

# 每请求被抢占次数分布 P99
histogram_quantile(
  0.99,
  sum by (le, instance) (
    rate(vllm:request_num_preemptions_bucket[5m])
  )
)

# 前缀缓存 token 级命中率
sum(rate(vllm:prefix_cache_hits_total[5m]))
  /
sum(rate(vllm:prefix_cache_queries_total[5m]))
```

抢占是**重算式**的：被抢占的请求回到等待队列，重新调度时要从头 prefill。这意味着：

- 抢占次数上升 → 有一批请求的 TTFT 会异常变长（重新排队 + 重新 prefill）；
- 抢占消耗的是 CPU 和 GPU 的重复计算 → 集群有效吞吐下降；
- 抢占是 KV Cache 不足的直接证据，比 kv_cache_usage_perc 更明确。

### 7.6 用 Little 定律估算在途并发

```promql
# 平均在途请求数（集群口径）
# rate(sum[5m]) 的物理含义是"请求·秒/秒"= 平均并发
sum(rate(vllm:e2e_request_latency_seconds_sum[5m]))

# 平均端到端延迟（秒）
sum(rate(vllm:e2e_request_latency_seconds_sum[5m]))
  /
sum(rate(vllm:e2e_request_latency_seconds_count[5m]))
```

这两个数字配合起来做容量规划非常有用：

```text
所需并发 = 目标 QPS × 平均端到端延迟
所需副本数 = ceil(所需并发 / 单副本实测承载并发) × 冗余系数
```

注意 `sum(rate(vllm:e2e_request_latency_seconds_sum[5m]))` 得到的平均并发，和 `vllm:num_requests_running` 的瞬时值应该基本吻合（前者是引擎视角，后者是瞬时快照）。如果长期不一致，说明有请求在引擎外等待（网关排队）或者直方图统计口径有偏差。

### 7.7 桶边界决定分位数精度

这是一个非常实际、也非常容易被忽略的问题。

直方图分位数的精度**完全取决于桶边界**。假设默认桶是 `0.1, 0.2, 0.4, 0.8, 1.6, 3.2, 6.4, +Inf`，那么你算出来的 P95 TTFT 只可能是"某个桶边界"或其插值，真实值可能是 0.9s 也可能是 1.5s。**如果你的 SLO 是"TTFT ≤ 1s"，而桶边界里没有落在 1s 附近，那这个 P95 数字对定 SLO 毫无意义。**

vLLM 官方文档里明确提到，请求阶段相关直方图的默认最小边界是 0.3s 量级，默认桶是按"典型 serving 负载"调的。因此：

- 如果你的 SLO 在亚秒级，必须用 --custom-histogram-buckets 把桶边界打到 SLO 附近；
- 自定义桶之后，所有基于桶的查询都要重新检查，尤其是"超过阈值 S 的请求占比"这类查询——如果阈值不在桶边界上，Prometheus 无法精确计算。

安全的做法是把 SLO 阈值本身作为一个桶边界写进去：

```bash
--custom-histogram-buckets '{"time_to_first_token":[0.1,0.2,0.5,1.0,2.0,4.0],"request_latency":[0.5,1.0,2.0,5.0,10.0,30.0,60.0]}'
```

这里的 `1.0` 就是 SLO 阈值，第 9.3 节的"坏请求占比"查询就精确了。

另外还有一个隐蔽的问题：**桶边界是启动参数，改桶要重启引擎。** 生产环境改桶要走变更流程（第 13 章）。如果不能立刻重启，可以先用"超过某桶边界"的粗略上界做临时判断，并在工单里注明口径。

### 7.8 错误率与 API 层指标

vLLM 通过 `prometheus_fastapi_instrumentator` 暴露 HTTP 层指标：

```promql
# 5xx 比例
sum(rate(http_requests_total{status=~"5.."}[5m]))
  /
sum(rate(http_requests_total[5m]))

# 按 handler 和状态码拆分
sum by (handler, status) (rate(http_requests_total[5m]))
```

> **重要口径说明**：`http_request_duration_seconds` 记录的是 HTTP handler 的处理时长。对**流式**请求，handler 在开始推流后就返回了，所以这个指标**不等于**用户感知的端到端延迟；对**非流式**请求，它接近端到端延迟。跨流式/非流式混用这个指标做 SLO 会得到错误的结论。**流式场景请一律用 vllm:e2e_request_latency_seconds。**

同样地，`http_requests_total` 里统计的是**完成 HTTP 响应**的请求数，被客户端超时中断的请求可能算作已完成但内容不全，这类"半截回答"在引擎侧会体现为 `finished_reason="abort"`。

```promql
# 被中断的请求比例
sum(rate(vllm:request_success_total{finished_reason="abort"}[5m]))
  /
sum(rate(vllm:request_success_total[5m]))
```

### 7.9 完整的排障查询包

把上面这些组合起来，做成一段可以直接贴进 Prometheus 查询页的表达式，用于回答"现在到底卡在哪"：

```promql
# 单实例全景：TTFT / 排队 / 运行中 / KV / 抢占速率
(
  histogram_quantile(0.95, sum by (le, instance) (rate(vllm:time_to_first_token_seconds_bucket[5m])))
  or
  histogram_quantile(0.95, sum by (le, instance) (rate(vllm:request_queue_time_seconds_bucket[5m])))
  or
  histogram_quantile(0.95, sum by (le, instance) (rate(vllm:request_prefill_time_seconds_bucket[5m])))
)
and on (instance) vllm:num_requests_waiting
```

实际排查时更实用的做法是把下面几组放进同一个 Grafana 面板组：

| Panel | PromQL（简写） | 回答什么 |
| --- | --- | --- |
| TTFT P95 | histogram_quantile(0.95, sum by (le,instance) (rate(vllm:time_to_first_token_seconds_bucket[5m]))) | 用户等待首字多久 |
| 排队时长 P95 | 同上，换 request_queue_time_seconds | 排队占 TTFT 多少 |
| Prefill 时长 P95 | 同上，换 request_prefill_time_seconds | prefill 占 TTFT 多少 |
| 队列深度 | vllm:num_requests_waiting | 有多少请求在等 |
| KV 使用率 | vllm:kv_cache_usage_perc | 显存侧还有多少余量 |
| 抢占速率 | sum(rate(vllm:num_preemptions_total[5m])) | 有没有在重算 |
| 生成吞吐 | sum(rate(vllm:generation_tokens_total[5m])) | 有效产出 |
| TPOT P95 | histogram_quantile(0.95, sum by (le,instance) (rate(vllm:request_time_per_output_token_seconds_bucket[5m]))) | 生成是否流畅 |

**这 8 个面板必须放在同一张图上、下面对齐时间轴。** 单独看任何一个都会得出片面的结论。

## 八、Grafana 面板搭建

### 8.1 面板布局建议

按"业务 → 引擎 → 资源"三段自上而下：

```text
┌───────────────────────────────────────────────────────────┐
│ Row 1: SLO 总览（给所有人看）                                │
│  [TTFT P95] [TPOT P95] [E2E P95] [成功率] [QPS] [生���吞吐]   │
├───────────────────────────────────────────────────────────┤
│ Row 2: 排队（最常出问题的一段）                              │
│  [队列深度 running/waiting] [排队时长 P50/P95/P99]           │
│  [TTFT vs 排队时长 叠加图]                                    │
├───────────────────────────────────────────────────────────┤
│ Row 3: 资源水位                                             │
│  [KV Cache 使用率] [批大小 iteration_tokens] [抢占速率]       │
│  [prefix cache 命中率]                                      │
├───────────────────────────────────────────────────────────┤
│ Row 4: 实例分布（找不均衡）                                  │
│  [每实例 TTFT P95] [每实例 QPS] [每实例 KV 使用率]            │
├───────────────────────────────────────────────────────────┤
│ Row 5: 硬件（排除环境因素）                                  │
│  [GPU 利用率] [Tensor 活跃度] [SM 时钟节流] [显存] [节点网络]  │
└───────────────────────────────────────────────────────────┘
```

### 8.2 关键 Panel 的配置要点

**Panel: TTFT P95 + 排队时长 P95 叠加**

- TTFT 用实线，排队时长用虚线，两者共用 Y 轴（单位都是秒）；
- 当 TTFT 上涨而排队时长同步上涨 → 排队问题；
- 当 TTFT 上涨但排队时长不变 → prefill 或上游问题；
- 当排队时长上涨而 TTFT 不变 → 说明在排队的都是长请求（长 prompt 的 prefill 预算挤占），短请求没受影响。

**Panel: 队列深度分状态堆叠**

```promql
# 堆叠图：running 与 waiting
sum by (instance) (vllm:num_requests_running) or vector(0)
```

Grafana 侧用两个 query + `Stacked` 显示方式。用 `or vector(0)` 的技巧是：当某个实例在重启期间不上报该指标时，Prometheus 不会返回空 series，图上不会出现断裂。

**Panel: KV Cache 使用率带阈值线**

- 查询：vllm:kv_cache_usage_perc
- 单位：percentunit（0~1 → 0%~100%）
- 阈值线：0.9 (warning) / 0.98 (critical)
- 建议对所有实例取 max，同时用一条单独的 min 线，可以发现"某个副本 KV 一直很空"（通常是路由不均）。

**Panel: 每实例对比（热力图或分位线图）**

```promql
histogram_quantile(0.95, sum by (le, instance) (rate(vllm:time_to_first_token_seconds_bucket[5m])))
```

Legend 设为 `{{instance}}`，并且**把 instance 维度放到表格而不是只看聚合值**。找出负载不均最快的办法是：

```promql
# 实例间 TTFT P95 的极差
max(histogram_quantile(0.95, sum by (le,instance) (rate(vllm:time_to_first_token_seconds_bucket[5m]))))
-
min(histogram_quantile(0.95, sum by (le,instance) (rate(vllm:time_to_first_token_seconds_bucket[5m]))))
```

这个极差持续大于 2 倍，基本可以判定路由有问题。

### 8.3 数据源与变量

- 数据源类型选 Prometheus，别用兼容模式，兼容模式在 histogram 上行为不一致；
- 建议加两个模板变量：
- 多集群部署时加 cluster 变量：label_values(vllm:num_requests_running, cluster)。

### 8.4 用官方 dashboard 作为起点

vLLM 仓库提供了 Prometheus + Grafana 的参考实现（`examples/observability/prometheus_grafana/`），SGLang 仓库提供了 `examples/monitoring/grafana/dashboards/json/sglang-dashboard.json`。这两个可以直接导入作为基线，然后按本文的三段式布局改造。

**但不要直接信官方 dashboard 里的 PromQL。** 官方示例里出现过一些在当前版本已移除或改名的指标，例如 V1 引擎已经去掉的 `vllm:num_requests_swapped`（swap 式抢占已不存在）和 `vllm:cpu_cache_usage_perc`，以及改名前的 `vllm:gpu_cache_usage_perc`。导入后逐个面板执行一次查询，确认不出 `No data` 或空结果，再上线。

## 九、告警设计

### 9.1 告警分层原则

告警必须分层，否则值班会被噪音淹没：

| 层级 | 面向 | 触发条件 | 通知方式 |
| --- | --- | --- | --- |
| P0 立刻处理 | 服务不可用 | 实例全部 down、5xx 突增 | 电话/IM 强提醒 |
| P1 影响 SLO | 业务受损 | TTFT/TPOT 错误预算快速消耗 | IM + 工单 |
| P2 容量预警 | 未来会出问题 | KV Cache 高位、队列持续增长、抢占上升 | 每日摘要 |
| P3 观察项 | 无需立即动作 | prefix cache 命中率低、单副本不均 | 周报 |

**P2 类绝对不要发强提醒。** 一个"KV Cache 使用率 85%"的告警每天响几十次，值班就会开始无视它，最后连 P0 也一起无视。

### 9.2 立即可用的告警规则

先写 recording rule，把重复的分位数计算抽出来：

```yaml
# monitoring/vllm-recording-rules.yaml
apiVersion:monitoring.coreos.com/v1
kind:PrometheusRule
metadata:
name:vllm-recording-rules
namespace:monitoring
labels:
    release:prometheus
spec:
groups:
    -name:vllm.slo
      rules:
        # ---- TTFT ----
        -record:job:vllm:ttft_p95:rate5m
          expr:|
            histogram_quantile(
              0.95,
              sum by (le, instance) (rate(vllm:time_to_first_token_seconds_bucket[5m]))
            )
        -record:job:vllm:ttft_p99:rate5m
          expr:|
            histogram_quantile(
              0.99,
              sum by (le, instance) (rate(vllm:time_to_first_token_seconds_bucket[5m]))
            )
        # ---- TPOT ----
        -record:job:vllm:tpot_p95:rate5m
          expr:|
            histogram_quantile(
              0.95,
              sum by (le, instance) (rate(vllm:request_time_per_output_token_seconds_bucket[5m]))
            )
        # ---- 排队 ----
        -record:job:vllm:queue_p95:rate5m
          expr:|
            histogram_quantile(
              0.95,
              sum by (le, instance) (rate(vllm:request_queue_time_seconds_bucket[5m]))
            )
        # ---- 吞吐与并发 ----
        -record:job:vllm:gen_throughput:rate5m
          expr:sum(rate(vllm:generation_tokens_total[5m]))
        -record:job:vllm:req_rate:rate5m
          expr:sum(rate(vllm:e2e_request_latency_seconds_count[5m]))
        -record:job:vllm:avg_concurrency:rate5m
          expr:sum(rate(vllm:e2e_request_latency_seconds_sum[5m]))
        # ---- 错误率 ----
        -record:job:vllm:http_5xx_ratio:rate5m
          expr:|
            sum(rate(http_requests_total{status=~"5.."}[5m]))
              /
            clamp_min(sum(rate(http_requests_total[5m])), 0.001)
        # ---- KV 与抢占 ----
        -record:job:vllm:kv_cache_usage:max
          expr:max(vllm:kv_cache_usage_perc)
        -record:job:vllm:kv_cache_usage:min
          expr:min(vllm:kv_cache_usage_perc)
        -record:job:vllm:preempt_rate:rate5m
          expr:sum(rate(vllm:num_preemptions_total[5m]))
        # ---- 前缀缓存 ----
        -record:job:vllm:prefix_cache_hit_ratio:rate5m
          expr: |
            sum(rate(vllm:prefix_cache_hits_total[5m]))
              /
            clamp_min(sum(rate(vllm:prefix_cache_queries_total[5m])), 0.001)
```

`clamp_min(..., 0.001)` 是必要的：没有流量时 `0/0` 会得到 NaN，NaN 参与比较会污染告警判断。

对应的告警规则：

```yaml
# monitoring/vllm-alerts.yaml
apiVersion:monitoring.coreos.com/v1
kind:PrometheusRule
metadata:
name:vllm-alerts
namespace:monitoring
labels:
    release:prometheus
spec:
groups:
    -name:vllm.availability
      rules:
        -alert:VLLMInstanceDown
          expr:up{job="vllm"}==0
          for:2m
          labels:
            severity:critical
          annotations:
            summary:"vLLM 实例 {{ $labels.instance }} 抓取失败"
            description:"实例已 {{ $value | humanizeDuration }} 无响应，请检查 Pod 状态与 /metrics 端点。"

        -alert:VLLMHighErrorRate
          expr:job:vllm:http_5xx_ratio:rate5m>0.01
          for:5m
          labels:
            severity:critical
          annotations:
            summary:"vLLM 5xx 比例 {{ $value | humanizePercentage }}"

        -alert:VLLMAllInstancesDown
          expr:count(up{job="vllm"}==1)==0
          for:1m
          labels:
            severity:critical
          annotations:
            summary:"所有 vLLM 实例均不可达"

    -name:vllm.latency
      rules:
        -alert:VLLMTTFTHigh
          expr:job:vllm:ttft_p95:rate5m>1.2
          for:10m
          labels:
            severity:warning
          annotations:
            summary:"{{ $labels.instance }} TTFT P95 = {{ $value }}s，超过 SLO 阈值 1.2s"

        -alert:VLLMTTPOTHigh
          expr:job:vllm:tpot_p95:rate5m>0.08
          for:10m
          labels:
            severity:warning
          annotations:
            summary:"{{ $labels.instance }} TPOT P95 = {{ $value }}s，生成速度劣化"

        -alert:VLLMQueueTimeHigh
          expr:job:vllm:queue_p95:rate5m>0.5
          for:10m
          labels:
            severity:warning
          annotations:
            summary:"{{ $labels.instance }} 排队时长 P95 = {{ $value }}s"

    -name:vllm.capacity
      rules:
        -alert:VLLMKVCacheNearFull
          expr:job:vllm:kv_cache_usage:max>0.95
          for:10m
          labels:
            severity:warning
          annotations:
            summary:"KV Cache 使用率 {{ $value | humanizePercentage }}，已接近上限"

        -alert:VLLMPreemptionSustained
          expr:job:vllm:preempt_rate:rate5m>0
          for:15m
          labels:
            severity:warning
          annotations:
            summary:"检测到持续的请求抢占（重算），速率 {{ $value }} 次/秒"

        -alert:VLLMAdmissionRejected
          expr:sum(rate(vllm:admission_rejections_total[5m]))>0
          for:5m
          labels:
            severity:warning
          annotations:
            summary:"请求被准入控制拒绝，检查上游限流与 max-num-seqs 配置"

    -name:vllm.distribution
      rules:
        -alert:VLLMInstanceImbalance
          expr:|
            max(histogram_quantile(0.95, sum by (le,instance) (rate(vllm:time_to_first_token_seconds_bucket[5m]))))
              -
            min(histogram_quantile(0.95, sum by (le,instance) (rate(vllm:time_to_first_token_seconds_bucket[5m]))))
            > 1.0
          for:15m
          labels:
            severity:warning
          annotations:
            summary:"实例间 TTFT P95 极差过大，疑似路由或副本健康度不均"
```

> `job:vllm:kv_cache_usage:max` 这种带多个冒号的 recording rule 名字是合法的，Prometheus 允许冒号出现在 recording rule 名称中（官方推荐用冒号做层级分隔）。但要注意：**在 Prometheus 里冒号是 recording rule 保留的分隔符**，用 `:` 连接的部分在部分工具（如 Grafana 变量、旧版 Thanos）里解析行为不一致，如果你的生态链有兼容性问题，可以改用下划线。

### 9.3 正确的 SLO 错误预算燃烧率告警

上面的 `> 1.2` 是"阈值告警"，简单直接但有两个问题：一是阈值拍脑袋定；二是对"短时间严重劣化"和"长时间轻微劣化"的区分度不足。

严谨的做法是定义"坏请求"——TTFT 超过 SLO 阈值的请求——然后监控**错误预算消耗速度**。

```yaml
        # ---- 坏请求占比（TTFT > 1s 的请求占比）----
        - record: job:vllm:ttft_bad_ratio:rate5m
          expr: |
            (
              sum(rate(vllm:time_to_first_token_seconds_count[5m]))
              -
              sum(rate(vllm:time_to_first_token_seconds_bucket{le="1"}[5m]))
            )
            /
            clamp_min(sum(rate(vllm:time_to_first_token_seconds_count[5m])), 0.001)

        - record: job:vllm:ttft_bad_ratio:rate1h
          expr: |
            (
              sum(rate(vllm:time_to_first_token_seconds_count[1h]))
              -
              sum(rate(vllm:time_to_first_token_seconds_bucket{le="1"}[1h]))
            )
            /
            clamp_min(sum(rate(vllm:time_to_first_token_seconds_count[1h])), 0.001)
```

> **前置条件**：这里用了桶边界 `le="1"`。如果你的引擎没有把 1.0 作为桶边界，这个表达式会恒为 0（`le="1"` 匹配不到任何桶，减法结果等于总数 → 比值为 1，实际上算出来是 100% 坏请求，也是错的）。**用之前必须先 curl /metrics | grep '_bucket{le=' 确认边界存在**，或者用 `--custom-histogram-buckets` 把它加进去（第 7.7 节）。

以"99% 请求 TTFT ≤ 1s"为 SLO（错误预算 1%）为例，多窗口多燃烧率告警：

```yaml
    - name:vllm.slo.burnrate
      rules:
        # 快速燃烧：5 分钟窗口 14.4 倍 + 1 小时窗口 14.4 倍
        -alert:VLLMTTFTBudgetBurnFast
          expr:|
            (
              job:vllm:ttft_bad_ratio:rate5m > 14.4 * 0.01
              and
              job:vllm:ttft_bad_ratio:rate1h > 14.4 * 0.01
            )
          for:2m
          labels:
            severity:critical
          annotations:
            summary:"TTFT SLO 错误预算快速消耗（14.4 倍燃烧率）"

        # 慢速燃烧：1 小时窗口 6 倍 + 6 小时窗口 6 倍
        -alert:VLLMTTFTBudgetBurnSlow
          expr:|
            (
              job:vllm:ttft_bad_ratio:rate5m > 6 * 0.01
              and
              job:vllm:ttft_bad_ratio:rate1h > 6 * 0.01
            )
          for:30m
          labels:
            severity:warning
          annotations:
            summary:"TTFT SLO 错误预算持续消耗（6 倍燃烧率）"
```

`14.4` 这个数字不是随便来的：99% SLO 下错误预算是 1%，14.4 倍燃烧率意味着 1 小时的 1% 预算会在 1/14.4 ≈ 4.2 分钟内耗尽。窗口组合（快窗口 + 慢窗口同时超阈值）是用来过滤"毛刺"的：只有短窗口和长窗口同时超，才认为是真的持续劣化。

**这套告警必须配合错误预算看板**，否则值班只知道"预算在烧"，不知道"还剩多少"：

```promql
# 30 天滚动窗口的错误预算剩余比例
1 - (
  job:vllm:ttft_bad_ratio:rate1h
  / 0.01
)
```

### 9.4 阈值必须来自业务基线

**不要抄上面的阈值。** 那些数字是举例。正确的做法是：

1. 选一个业务低峰时段，跑一次基线压测（第 11 章），拿到真实的 TTFT/TPOT 分布；
2. 观察一到两周生产数据，取 P95 的 1.2~1.5 倍作为初始告警阈值；
3. 用业务方能感知的数字反推：比如"超过 3 秒用户会关页面"，那 TTFT SLO 就是 2.5s，P99 阈值设在 3s；
4. 每季度复核一次阈值。

GPU 侧的阈值同理：不同型号的卡温度墙、功耗墙不同，A100 和 4090 的降频阈值不一样。节流告警用"节流原因非零"作为条件，具体是否可接受由平台团队判断。

## 十、故障排查闭环

下面六个场景按"现象 → 初判 → 检查 → 指标 → 根因 → 修复 → 验证 → 回滚 → 复盘"的完整链路写。每个场景都给出可执行的命令和判断依据。

### 场景 A：TTFT P95 突增，QPS 没变

**现象**：业务反馈"AI 回答变慢"，Grafana 上 `vllm:time_to_first_token_seconds` 的 P95 从 0.6s 涨到 3.2s。QPS、5xx 错误率正常。

**初判**：先确认是"所有实例都慢"还是"个别实例慢"。

```promql
# 看每个实例的 TTFT P95
histogram_quantile(0.95, sum by (le, instance) (rate(vllm:time_to_first_token_seconds_bucket[5m])))
```

**检查**：

```bash
# 1. 引擎侧：队列和 prefill 谁在涨？
curl -s http://127.0.0.1:8000/metrics | grep -E '^vllm:(num_requests_waiting|num_requests_running|kv_cache_usage_perc)'
```

**指标**：关键的一刀是把 TTFT 拆成排队 + prefill：

```promql
# 排队时长 P95
histogram_quantile(0.95, sum by (le,instance) (rate(vllm:request_queue_time_seconds_bucket[5m])))
# prefill 时长 P95
histogram_quantile(0.95, sum by (le,instance) (rate(vllm:request_prefill_time_seconds_bucket[5m])))
```

**判断依据**（这是本场景的核心分岔）：

| 排队时长 | Prefill 时长 | 结论 |
| --- | --- | --- |
| ↑↑ | 基本不变 | 调度容量不足，转 A-1 |
| 基本不变 | ↑↑ | prefill 变慢，转 A-2 |
| 都 ↑ | 都 ↑ | 系统级资源问题，转 A-3 |
| 都基本不变 | 都基本不变 | 问题不在引擎，转 A-4 |

**A-1 调度容量不足的细分**

```promql
vllm:num_requests_waiting
vllm:num_requests_waiting_by_reason
vllm:kv_cache_usage_perc
```

- num_requests_waiting 高 + kv_cache_usage_perc 接近 1 → KV Cache 不足，新请求拿不到块。根因通常是上下文太长或输出长度暴涨。
- num_requests_waiting 高 + kv_cache_usage_perc 低（比如 0.3）+ num_requests_running 贴着 --max-num-seqs → 批次槽位不足。
- num_requests_waiting 高 + kv_cache_usage_perc 低 + num_requests_running 不高 → token 预算被长 prefill 挤占。

最后这种情况的证据是 prompt 长度的分布：

```promql
histogram_quantile(0.95, sum by (le) (rate(vllm:request_prompt_tokens_bucket[5m])))
```

如果 P95 prompt 长度突然从 2K 涨到 30K，就是这个原因。

**根因定位到"token 预算被挤占"时**，看引擎日志：

```bash
# vLLM 日志里调度相关的信息（在 K8s 里）
kubectl -n llm logs deploy/vllm --since=30m | grep -iE 'prefill|budget|chunk|waiting' | tail -n 50
```

**修复**（按影响从小到大）：

1. 业务侧限流：对长 prompt 请求降优先级或加排队上限（在网关层）；
2. 调整调度参数：调大 --max-num-batched-tokens（提升吞吐但会增加单步延迟），或调小 --long-prefill-token-threshold（让更多请求被划入"长 prefill"走独立调度步，避免它们挤占短请求的 token 预算）；
3. 调整 --max-num-seqs：降低它可以减少 KV 占用（因为同时在跑的少），是 KV 紧张时的常见缓解手段；
4. 扩容副本。

> **--long-prefill-token-threshold 的语义**：V1 调度器会按 prompt 长度把请求分成"普通请求"和"长 prefill 请求"，后者被安排到独立的调度步里处理，避免它的 prefill 计算挤进其他请求的 decode 步里。调小这个阈值 = 更多请求走隔离路径 = 短请求的 TTFT 更稳，但总体调度步数变多、吞吐略降。该参数的默认值随版本和 `max_num_batched_tokens` 联动，动手前先 `vllm serve --help` 看清楚当前值。

> **参数调整的取舍必须写清楚**：调大 `--max-num-batched-tokens`：吞吐上升，但单次调度步时间变长，**TTFT 的平均值会变差**（每个请求要等更长的批次才能凑齐）。这是"用 TTFT 换吞吐"的典型操作，动手前要跟业务确认能不能接受。调大 `--max-num-seqs`：并发上升，KV 占用上升，可能触发抢占，反而让 TPOT 变差。调小 `--max-num-seqs`：并发下降，KV 压力缓解，TTFT 改善，但吞吐下降，QPS 下降。这些参数**都需要重启引擎生效**，属于生产变更，必须走灰度（先改一个副本，观察 30 分钟，再推全量）。

**验证**：重启后 10 分钟，确认：

```promql
job:vllm:ttft_p95:rate5m
job:vllm:queue_p95:rate5m
vllm:kv_cache_usage_perc
job:vllm:preempt_rate:rate5m
```

TTFT 回落、队列深度下降、KV 使用率从 95% 降到 70% 以下，且吞吐没有明显下降。

**回滚**：

```bash
# 方式一：Helm 部署的集群，直接回退到上一个 revision
helm history deploy/vllm --namespace llm            # 先确认 revision 编号
helm rollback deploy/vllm <revision> --namespace llm --wait --timeout 10m

# 方式二：原生 Deployment，用 kubectl 撤销上一次的 spec 变更
kubectl -n llm rollout history deployment/vllm      # 查看变更历史
kubectl -n llm rollout undo deployment/vllm
kubectl -n llm rollout status deployment/vllm --timeout=10m

# 方式三：参数通过 ConfigMap 挂载时，回滚 ConfigMap 后触发滚动更新
kubectl -n llm apply -f vllm-args-configmap.yaml    # 用旧版本配置覆盖
kubectl -n llm rollout restart deployment/vllm
kubectl -n llm rollout status deployment/vllm --timeout=10m
```

> 三种回滚方式的前提是**变更前有可回退的版本**：Helm 要有历史 revision，Deployment 要有 `rollout history` 记录，ConfigMap 要有版本化备份。**如果参数是直接改 Deployment 的 args 且没开 --record，rollout undo 会回退到"上一个" spec 而不是你想要的版本**——所以生产变更前先 `kubectl -n llm get deploy vllm -o yaml > vllm-deploy-backup-$(date +%F-%H%M).yaml` 存一份。

**复盘**：把这次的输入长度分布、输出长度分布、队列深度曲线归档，作为下一次容量规划的输入。

**A-2 prefill 变慢**

可能原因：输入变长（见 A-1）、前缀缓存失效、GPU 降频、chunked prefill 被关闭或预算太小。

```bash
# GPU 状态
nvidia-smi --query-gpu=index,name,clocks.sm,clocks.max.sm,temperature.gpu,power.draw,utilization.gpu,memory.used,memory.total --format=csv

# 持续采样看降频
nvidia-smi dmon -s pucvmet -d 1 -c 30
```

```promql
# 节流是否发生（reason 编码以实际输出为准）
max by (instance, reason) (DCGM_FI_DEV_SM_CLOCK_THROTTLE_REASONS) > 0
```

```promql
# 前缀缓存命中率
job:vllm:prefix_cache_hit_ratio:rate5m
```

如果 GPU 正常而前缀缓存命中率从 0.6 掉到 0.05，那就是缓存失效。常见原因：**多副本没有做前缀亲和路由**，同一个会话的前缀被路由到不同副本，每个副本都要重新 prefill。

**修复**：上 KV cache 感知的路由（Gateway API Inference Extension 的 EPP、llm-d router，或 vLLM Router 这类方案），让相同前缀的请求落到同一副本。

**A-3 系统级资源问题**

检查节点是否被别的负载抢占：

```bash
# 节点 GPU 情况
kubectl get nodes -o wide | grep -i gpu
# 是否有其他 Pod 也在用同一块卡
nvidia-smi
kubectl describe node <node> | grep -A5 "Allocated resources"
```

**A-4 引擎没问题，问题在上游**

此时引擎的所有指标都正常，但客户端观测到的 TTFT 涨了。检查网关：

```bash
# 网关侧的排队指标（Envoy 示例）
curl -s http://127.0.0.1:9901/stats | grep -E 'upstream_rq_(pending|total|overflow)'
```

同时检查 API Server 自身的 CPU：

```promql
# 单进程的 CPU 使用率
rate(process_cpu_seconds_total[5m])
```

> 用 `--api-server-count > 1` 时，`process_cpu_seconds_total` 不可用（多进程模式不支持），需要用 cAdvisor（`container_cpu_usage_seconds_total`）或 node exporter。**这是第 2.2 节提到的边界情况在排查时最常踩的坑。**

还有一个常见情况：**客户端自己在排队**。用 `vllm bench serve` 的 `--percentile-metrics` 里包含 `client_queue_time` 这个指标，专测客户端侧排队：

```bash
vllm bench serve --backend vllm \
  --host 10.0.0.10 --port 8000 \
  --dataset-name random --random-input-len 1024 --random-output-len 256 \
  --num-prompts 200 --max-concurrency 20 \
  --percentile-metrics ttft,tpot,itl,e2el,client_queue_time \
  --metric-percentiles 50,95,99
```

输出里 `Client queue time` 明显大于 0，说明请求在客户端连接池里就排队了，跟服务端无关。

### 场景 B：TTFT 正常但生成速度慢（TPOT 劣化）

**现象**：首字快，但生成过程"一顿一顿"。

**初判**：TPOT P95 上升，ITL P99 上升，而 TTFT 正常。

**检查与指标**：

```promql
job:vllm:tpot_p95:rate5m
vllm:kv_cache_usage_perc
job:vllm:preempt_rate:rate5m
vllm:num_requests_running
```

**根因一：抢占。** KV 使用率高 + 抢占速率 > 0 + ITL 出现尖刺。这是 V1 引擎重算式抢占的直接后果。

```bash
# 日志里找抢占
kubectl -n llm logs deploy/vllm --since=30m | grep -i preempt | tail -n 30
```

**修复**：缓解顺序是——降低 `--max-num-seqs`（减少同时占用的序列）、限制 `max_tokens`（业务侧）、缩短上下文（业务侧）、扩容。**注意方向：抢占场景下调大 --max-num-seqs 是反作用，会让情况更糟。**

**根因二：长 prefill 挤占 decode。** 观察 prefill 时长和 decode 时长的比值：

```promql
histogram_quantile(0.95, sum by (le,instance) (rate(vllm:request_prefill_time_seconds_bucket[5m])))
histogram_quantile(0.95, sum by (le,instance) (rate(vllm:request_decode_time_seconds_bucket[5m])))
```

prefill 时长明显大于 decode 时长，说明调度步被 prefill 主导，decode 请求在等预算。

**根因三：GPU 降频或被抢占。** 见 A-3。

**根因四：批过大导致单步变慢。** 批次里请求太多，每步计算量变大，所有请求的 TPOT 同步劣化：

```promql
# 平均每个调度步的 token 数
sum by (instance) (rate(vllm:iteration_tokens_total_sum[5m]))
  /
sum by (instance) (rate(vllm:iteration_tokens_total_count[5m]))
```

**验证**：调整后重跑压测（第 11 章），对比 TPOT P95 与吞吐，确认没有"降延迟换吞吐"的副作用。

**回滚**：参数类变更同场景 A。

### 场景 C：排队高但 KV Cache 使用率低

这是最反直觉的场景，因为直觉是"排队 = 资源不够"。但资源不一定是 KV。

**排查顺序**：

```bash
# 1. 确认批次槽位是不是顶满了
curl -s http://127.0.0.1:8000/metrics | grep -E '^vllm:num_requests_(running|waiting)'
```

对比 `num_requests_running` 与启动参数里的 `--max-num-seqs`（可以从 `vllm:cache_config_info` 拿到）：

```promql
vllm:num_requests_running
# 启动参数里的值
vllm:cache_config_info
```

如果 `running` 恒等于 `max_num_seqs`，就是槽位限制。

```promql
# 2. 确认是不是长 prefill 挤占
histogram_quantile(0.99, sum by (le) (rate(vllm:request_prompt_tokens_bucket[5m])))
```

```promql
# 3. 确认不是 API Server CPU 饱和（tokenizer 瓶颈）
sum by (pod) (rate(container_cpu_usage_seconds_total{container="vllm"}[5m]))
```

**修复**：

- 槽位限制 → 调大 --max-num-seqs（前提是 KV 还有余量）；
- 长 prefill → 调大 --max-num-batched-tokens，或者业务侧限制最大上下文；
- tokenizer 瓶颈 → 调大 --api-server-count（多进程 API Server），或者客户端侧预分词（部分框架支持传入 token IDs）。

### 场景 D：请求被拒绝 / 大量 429、503

**指标**：

```promql
sum by (reason) (rate(vllm:admission_rejections_total[5m]))
sum by (handler, status) (rate(http_requests_total[5m]))
sum by (finished_reason) (rate(vllm:request_success_total[5m]))
```

**判断**：

- admission_rejections 上升 → 引擎侧的准入限制生效（队列上限类限制）；
- HTTP 429 → 上游网关限流，不是引擎的问题；
- HTTP 503 → 引擎副本不够（Service 没有可用后端）或 Pod 未就绪；
- finished_reason="abort" 上升 → 客户端超时断开或网关主动中断，说明真实延迟已经超过了上游超时配置。

**修复**：区分是"真拒绝"还是"限流保护"：

- 真拒绝（容量不足）→ 扩容；
- 限流保护（保护系统）→ 调高阈值，但要先确认容量够，否则会把限流变成雪崩。

**风险**：直接调高网关限流阈值而不扩容，会让请求全部堆到引擎，TTFT 和 TPOT 同时崩坏，触发级联失败。**任何"提高上限"的改动都要先看容量水位。**

### 场景 E：副本之间表现不一致

**现象**：全局 P95 看起来还行，但用户反馈"有时候特别慢"。

**指标**：

```promql
# 每实例 TTFT P95
histogram_quantile(0.95, sum by (le, instance) (rate(vllm:time_to_first_token_seconds_bucket[5m])))
# 每实例 QPS
sum by (instance) (rate(vllm:e2e_request_latency_seconds_count[5m]))
# 每实例 KV 使用率
vllm:kv_cache_usage_perc
```

**判断依据**：如果 QPS 分布均匀但 TTFT 不均匀 → 不是流量问题，是**副本本身的差异**：

- GPU 型号不同（混插了 A100 和 4090）→ 性能天然不同；
- 副本启动参数不同（灰度期间）→ 对比 vllm:cache_config_info 的标签；
- 某个副本所在节点有问题（网络、磁盘、CPU 争抢）；
- 某个副本的 KV cache 冷（前缀缓存没建立起来）。

```bash
# 对比不同副本的启动配置
kubectl -n llm get pods -l app.kubernetes.io/name=vllm -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[0].args}{"\n"}{end}'
```

```bash
# 看节点 GPU 型号
kubectl -n llm get pods -l app.kubernetes.io/name=vllm \
  -o custom-columns=POD:.metadata.name,NODE:.spec.nodeName,GPU:.status.allocatable.nvidia\\.com/gpu
```

```bash
# 节点级排查
kubectl -n llm describe node <node> | grep -A10 "Allocated resources"
kubectl -n llm get events --field-selector involvedObject.kind=Node --sort-by=.lastTimestamp | tail -n 20
```

**修复**：统一镜像与参数；GPU 型号分池部署并用节点亲和隔离；加健康检查与自动摘除；上 KV 感知的会话亲和路由。

**验证**：重启/调整后 30 分钟，看实例间 TTFT 极差是否收敛到 1.5 倍以内。

### 场景 F（附加）：Pod 重启风暴

**现象**：推理 Pod 反复重启，TTFT 出现规律性尖刺。

**检查**：

```bash
kubectl -n llm get pods -l app.kubernetes.io/name=vllm
kubectl -n llm describe pod <pod> | tail -n 30   # 看 Last State / Exit Code
kubectl -n llm logs <previous-pod> --previous | tail -n 100
```

**判断依据**：

| 退出码/信号 | 常见原因 |
| --- | --- |
| Exit Code 137 | 被 OOM Killer 杀掉（内存超限） |
| Exit Code 1，日志有 CUDA OOM | 显存不足，--gpu-memory-utilization 太高或 KV Cache 计算错误 |
| Exit Code 139 | 段错误，通常是模型加载或显存映射问题 |
| 无退出码，Node 变 NotReady | 节点故障或网络插件异常 |

**注意**：`--gpu-memory-utilization` 的计算是"按当前可用显存乘以比例"，如果节点上还有别的进程占显存，vLLM 启动时会算错，导致运行期 OOM。排查时先 `nvidia-smi` 看卡上还有什么。

**修复**：调整 `--gpu-memory-utilization`、限制 `--max-model-len`、加 `preStop` 优雅退出（给在途请求留时间）。

```yaml
# 在 vLLM Deployment 上加优雅退出
spec:
  template:
    spec:
      terminationGracePeriodSeconds: 120
      containers:
        - name: vllm
          lifecycle:
            preStop:
              exec:
                command: ["sh", "-c", "sleep 15"]
```

> `preStop` 的 `sleep` 是为了让 Endpoint 摘除先传播出去，再终止进程，避免"Pod 已经不再接流量但进程还在被请求打"的窗口。`terminationGracePeriodSeconds` 必须大于 `preStop` 的 sleep 时长加上最长请求的处理时间，否则请求会被硬杀（表现为 `finished_reason="abort"`）。

## 十一、基准验证与容量规划

### 11.1 压测是唯一可靠的基线来源

监控数据告诉你"现在多快"，压测告诉你"最多能多快"。没有压测基线，扩容决策就是拍脑袋。

```bash
# 基线压测：先定并发台阶，再看拐点
vllm bench serve --backend vllm \
  --host 10.0.0.10 --port 8000 \
  --model Qwen/Qwen3-8B \
  --dataset-name random \
  --random-input-len 1024 --random-output-len 256 \
  --num-prompts 300 \
  --max-concurrency 16 \
  --num-warmups 20 \
  --percentile-metrics ttft,tpot,itl,e2el \
  --metric-percentiles 50,95,99 \
  --save-result --result-dir ./bench --result-filename baseline
```

关键参数说明：

- --max-concurrency：并发上限，模拟"同时在途的请求数"。做容量扫描时要逐级调整（8/16/32/64/128）；
- --request-rate：请求到达速率，设成具体值（如 10）模拟稳态流量，设成 inf（默认）是"一瞬间全部打进去"，测的是极限吞吐；
- --burstiness：突发因子，默认 1.0 是泊松分布；模拟"整点批量任务"时可以调小到 0.3；
- --num-warmups：预热请求数。一定要设，否则前几十个请求的冷启动（权重换入、CUDA graph 初始化、KV Cache 预热）会污染分位数；
- --goodput ttft:1000 tpot:100：直接算"同时满足 TTFT ≤ 1s 且 TPOT ≤ 100ms 的请求比例"。这是最接近业务真实感知的指标；
- --probe-request-rate：发送少量单 token 探测请求，绕过主负载的并发限制，用来测"主负载把无关请求拖慢了多少"。这是探测队头阻塞（head-of-line blocking）的利器。

输出结果长这样（字段名以实际版本为准）：

```text
============ Serving Benchmark Result ============
Successful requests:                     300
Benchmark duration (s.):                 42.18
Total input tokens:                     307200
Total generated tokens:                 76800
Request throughput (req/s):             7.11
Output token throughput (tok/s):        1820.9
Total token throughput (tok/s):         9104.5
Mean TTFT (ms):                         412.7
Median TTFT (ms):                       358.1
P99 TTFT (ms):                          1284.3
Mean TPOT (ms):                         52.4
Median TPOT (ms):                       49.8
P99 TPOT (ms):                          96.2
Mean ITL (ms):                          50.1
Median ITL (ms):                        47.6
P99 ITL (ms):                           88.7
Mean E2EL (ms):                         1704.2
Median E2EL (ms):                       1598.7
P99 E2EL (ms):                          3110.5
==============================================
```

> **版本差异**：`--random-input` / `--random-output` 是旧版本的参数名，新版本改为 `--random-input-len` / `--random-output-len`（并提供 `--input-len` / `--output-len` 作为通用别名）。较新版本里 `vllm bench serve` 有独立的 Rust 实现（vllm-project/vllm-bench），输出字段名与 Python 实现有细微差别。`vllm bench serve --help` 是唯一权威来源，本节给出的输出样例来自 Python 实现，仅用于说明各字段的含义。

### 11.2 找拐点

容量扫描的做法：逐级提高 `--max-concurrency`，记录每档的 TTFT P95、TPOT P95 和输出吞吐，画成三条曲线。

```text
并发   TTFT P95   TPOT P95   输出吞吐(tok/s)
8      0.35s      0.045s     1850
16     0.42s      0.052s     1820   <- 吞吐已经不再涨
32     1.85s      0.091s     1790   <- TTFT 开始劣化
64     6.20s      0.180s     1650   <- 明显过载
```

**拐点判读**：

- 吞吐不再增长 = 已经饱和；
- 出现拐点后 TTFT 迅速恶化 = KV Cache 或批次预算成为瓶颈（吞吐不变但延迟暴涨，说明工作被"排队"了）；
- 吞吐和延迟同时恶化 = 已经在丢请求或者触发大量抢占了。

**单副本可用并发 = 拐点处的并发数。** 规划时留 30% 余量：

```text
单副本安全并发 = 拐点并发 × 0.7
所需副本数 = ceil(峰值 QPS × 平均 E2E / 单副本安全并发)
```

### 11.3 压测脚本：带 dry-run 和二次确认

生产环境跑压测要有保护，下面这个脚本封装了检查、确认和日志记录：

```bash
#!/usr/bin/env bash
# llm_loadtest.sh —— 受控的推理服务压测脚本
# 特性：dry-run、并发确认、结果归档、退出码检查
set -euo pipefail

usage() {
cat <<'EOF'
用法：
  llm_loadtest.sh --host <ip> --port <port> --model <name> \
      [--concurrency 16] [--prompts 200] [--input-len 1024] [--output-len 256] \
      [--goodput ttft:1000] [--dry-run] [--confirm]

环境变量：
  VLLM_API_KEY   若服务开启鉴权则必须提供（不落盘，仅存在于进程环境）
EOF
}

HOST=""; PORT="8000"; MODEL=""; CONCURRENCY="16"; PROMPTS="200"
INPUT_LEN="1024"; OUTPUT_LEN="256"; GOODPUT=""; DRY_RUN="0"; CONFIRM="0"

while [[ $# -gt 0 ]]; do
case"$1"in
    --host)       HOST="$2"; shift 2 ;;
    --port)       PORT="$2"; shift 2 ;;
    --model)      MODEL="$2"; shift 2 ;;
    --concurrency) CONCURRENCY="$2"; shift 2 ;;
    --prompts)    PROMPTS="$2"; shift 2 ;;
    --input-len)  INPUT_LEN="$2"; shift 2 ;;
    --output-len) OUTPUT_LEN="$2"; shift 2 ;;
    --goodput)    GOODPUT="$2"; shift 2 ;;
    --dry-run)    DRY_RUN="1"; shift ;;
    --confirm)    CONFIRM="1"; shift ;;
    -h|--help)    usage; exit 0 ;;
    *) echo"未知参数: $1" >&2; usage >&2; exit 1 ;;
esac
done

if [[ -z "${HOST}" || -z "${MODEL}" ]]; then
echo"ERROR: --host 与 --model 为必填项" >&2
  usage >&2
exit 1
fi

# 数值合法性校验，避免把非数字传给压测工具导致行为不可预期
for pair in"CONCURRENCY:${CONCURRENCY}""PROMPTS:${PROMPTS}" \
            "INPUT_LEN:${INPUT_LEN}""OUTPUT_LEN:${OUTPUT_LEN}"; do
  name="${pair%%:*}"; val="${pair##*:}"
if ! [[ "${val}" =~ ^[0-9]+$ ]] || (( val <= 0 )); then
    echo"ERROR: ${name} 必须是正整数，当前值: ${val}" >&2
    exit 1
fi
done

TS="$(date +%Y%m%d-%H%M%S)"
OUT_DIR="./bench-results/${TS}"
LOG_FILE="${OUT_DIR}/run.log"

AUTH_ARGS=()
if [[ -n "${VLLM_API_KEY:-}" ]]; then
  AUTH_ARGS=(--header "Authorization=Bearer ${VLLM_API_KEY}")
fi

CMD=(vllm bench serve --backend vllm
     --host "${HOST}" --port "${PORT}" --model "${MODEL}"
     --dataset-name random
     --random-input-len "${INPUT_LEN}" --random-output-len "${OUTPUT_LEN}"
     --num-prompts "${PROMPTS}" --max-concurrency "${CONCURRENCY}"
     --num-warmups 20
     --percentile-metrics ttft,tpot,itl,e2el
     --metric-percentiles 50,95,99
     --save-result --result-dir "${OUT_DIR}")

if [[ -n "${GOODPUT}" ]]; then
  CMD+=(--goodput "${GOODPUT}")
fi
if (( ${#AUTH_ARGS[@]} > 0 )); then
  CMD+=("${AUTH_ARGS[@]}")
fi

echo"将执行压测："
printf'  %q '"${CMD[@]}"; echo

if (( DRY_RUN == 1 )); then
echo"DRY-RUN：仅打印命令，不发起请求。"
exit 0
fi

if (( CONFIRM != 1 )); then
echo"提示：压测会向生产服务注入 ${PROMPTS} 个请求。"
read -r -p "确认执行请加 --confirm 参数。输入 yes 继续: " ans
  [[ "${ans}" == "yes" ]] || { echo"已取消。"; exit 1; }
fi

mkdir -p "${OUT_DIR}"
{
echo"=== 开始时间: $(date -Is) ==="
echo"目标: ${HOST}:${PORT} 模型: ${MODEL}"
echo"并发: ${CONCURRENCY} 请求数: ${PROMPTS} 输入: ${INPUT_LEN} 输出: ${OUTPUT_LEN}"
} >> "${LOG_FILE}"

set +e
"${CMD[@]}" 2>&1 | tee -a "${LOG_FILE}"
RC=${PIPESTATUS[0]}
set -e

{
echo"=== 结束时间: $(date -Is) ==="
echo"退出码: ${RC}"
} >> "${LOG_FILE}"

if (( RC != 0 )); then
echo"ERROR: 压测失败，退出码 ${RC}，详见 ${LOG_FILE}" >&2
exit"${RC}"
fi

echo"压测完成，结果目录：${OUT_DIR}"
exit 0
```

这个脚本里几个值得学的点：

- set -euo pipefail：-u 防止未定义变量，-o pipefail 让 tee 管道的失败能被捕获；
- 数值校验用正则 ^[0-9]+$，避免把 16; rm -rf / 这种内容传进命令行；
- 凭证只从环境变量读，不落盘、不进命令行历史（--header 的值会出现在进程列表里，所以更稳妥的做法是让压测工具读环境变量，具体以工具支持的参数为准）；
- PIPESTATUS[0] 取管道里第一个命令的真实退出码；
- dry-run 与 --confirm 双保险；
- 结果按时间戳归档，保留原始 JSON，便于后续和监控数据对齐分析。

### 11.4 用监控数据交叉验证压测结论

压测结束后，对比压测结果和监控指标是否一致：

```promql
# 压测期间的实际生成吞吐应该接近压测报告的 Output token throughput
sum(rate(vllm:generation_tokens_total[5m]))
```

如果压测报告说 1800 tok/s，但监控只看到 400 tok/s，说明有请求在压测客户端侧就失败了（连接错误、超时），压测结果不可信。

## 十二、常见陷阱清单

### 12.1 指标名与后缀

- counter 在文本格式下自动带 `_total`。文档写 vllm:generation_tokens，/metrics 里是 vllm:generation_tokens_total，PromQL 里必须写后者。
- KV Cache 指标名有版本差异。新版本是 vllm:kv_cache_usage_perc，旧版本（0.9.x 及以前）是 vllm:gpu_cache_usage_perc。用第 5.4 节的脚本核对，不要凭印象。
- 官方 Grafana dashboard 里可能有过期指标名，导入后必须逐面板验证。

### 12.2 统计口径

- TTFT 从 tokenization 开始，不含网关排队。 跨层对比延迟时要说清口径。
- 流式请求的 `http_request_duration_seconds` 不等于端到端延迟。 流式场景一律用 vllm:e2e_request_latency_seconds。
- TPOT 与 ITL 的加权方式不同。inter_token_latency_seconds 是 token 加权，request_time_per_output_token_seconds 是请求等权。做"用户感知卡顿"用前者，做"平均生成速度"用后者。
- 分位数不能跨桶算术。1 / P95(TPOT) 不等于 P95 的 tok/s。

### 12.3 采样与精度

- 抓取间隔决定毛刺可见性。 15 秒间隔抓不到秒级尖刺。
- 桶边界决定分位数精度。 亚秒级 SLO 必须自定义桶。
- 低 QPS 下分位数抖动。 5 分钟内请求数少于几十个时，P95 基本没有意义。这时候改用均值，或者拉长窗口。

### 12.4 多实例与重启

- counter 在 Pod 重启后归零，跨重启的累计用 increase()。
- 分位数必须按 instance 分组，否则坏副本被平均掉。
- `--api-server-count > 1` 时 `python_*` / `process_*` 指标不可用。
- `vllm:num_requests_waiting` 是瞬时值，容易被采样时机误导，配合排队时长分位数一起看。

### 12.5 别把 vLLM 指标当业务真相

引擎指标反映的是"引擎处理得怎么样"，不反映"回答对不对"。内容质量、需要靠业务侧评估（离线评测集 + 在线人工抽检）。推理服务的监控到引擎层就该收口，再往上是内容质量监控，那是另一套体系。

## 十三、变更风险与回滚

监控体系本身也是生产系统，改配置一样要评估风险。

| 变更项 | 影响范围 | 前置检查 | 风险 | 回滚 |
| --- | --- | --- | --- | --- |
| 新增 ServiceMonitor | 监控命名空间 + 目标命名空间的流量 | 确认 Prometheus 的 serviceMonitorSelector / serviceMonitorNamespaceSelector 能选中 | 抓取风暴拖垮 Prometheus | kubectl delete servicemonitor |
| 修改直方图桶 | 需要重启所有推理 Pod | 确认新桶覆盖 SLO 阈值；评估时间序列数量增长 | 重启期间容量下降；Prometheus 存储成本上升 | helm rollback + 重启 |
| 修改 vllm:cache_config_info 之类的 label | 全部查询 | 检查依赖这些标签的 recording rule | 面板空白、告警失效 | 回滚 PrometheusRule |
| 调整告警阈值 | 值班噪音 | 用历史数据回算触发频次 | 漏报或告警风暴 | 恢复上一版 PrometheusRule |
| 扩容副本 | 集群 GPU 资源 | 确认节点有空闲 GPU；确认镜像和参数一致 | 调度不上、抢占其他任务 | kubectl scale --replicas=N |
| 改 --max-num-seqs 等调度参数 | 全部副本 | 先在预发环境压测对比 | TTFT/吞吐此消彼长 | helm rollback |

### 13.1 Prometheus 配置变更的标准流程

```bash
# 1. 本地校验语法
promtool check config prometheus.yml
promtool check rules vllm-alerts.yaml

# 2. 用历史数据做单元测试（可选但强烈建议）
promtool test rules tests/vllm-alerts.test.yaml

# 3. 先在预发环境验证
# 4. 灰度：先改一个副本的 ServiceMonitor 或先加载规则但不启用通知
# 5. 观察 30 分钟，再全量
```

单元测试文件示例：

```yaml
# tests/vllm-alerts.test.yaml
rule_files:
-vllm-recording-rules.yaml
-vllm-alerts.yaml

evaluation_interval:1m

tests:
-interval:1m
    input_series:
      # 60 次/分 → rate = 1.0/s
      -series:'vllm:time_to_first_token_seconds_count{instance="a"}'
        values:'0+60x60'
      # 其中 30 次/分 落在 le="1" 桶 → rate = 0.5/s
      -series:'vllm:time_to_first_token_seconds_bucket{le="1",instance="a"}'
        values:'0+30x60'
      -series:'vllm:time_to_first_token_seconds_bucket{le="+Inf",instance="a"}'
        values:'0+60x60'
    promql_expr_test:
      -expr:job:vllm:ttft_bad_ratio:rate5m
        eval_time:10m
        exp_samples:
          # 该 recording rule 没有 by(...) 聚合，因此结果不带 instance 标签
          -labels:'{}'
            value:0.5
```

这个测试的含义是：10 分钟内共 60 个请求，其中 30 个的 TTFT 超过 1 秒，坏请求比例应为 30/60 = 0.5。

> `exp_samples` 里的 `labels` 字段是**结果的标签集**，不是带指标名的完整标识。如果你的 recording rule 带了 `by (instance)`，期望标签就要写成 `{instance="a"}`；写成 `job:vllm:ttft_bad_ratio:rate5m{instance="a"}` 是错的，promtool 会因为标签不匹配而报测试失败。

### 13.2 生产变更的权限与窗口

- PrometheusRule、ServiceMonitor 的修改权限应与推理集群的部署权限分离，监控配置变更走独立审批；
- 告警规则的变更避开业务高峰，或者先用 severity: none 的形式加载，观察触发频次后再启用通知；
- 所有变更在工单里记录：变更前后配置、变更时间、执行人、验证方式、回滚命令；
- 推理引擎的参数变更属于有损变更（重启会中断在途请求），必须：

```bash
# 逐副本滚动更新的观察点
kubectl -n llm get pods -l app.kubernetes.io/name=vllm -w
kubectl -n llm rollout status deployment/vllm --timeout=15m
```

### 13.3 灰度与验证方式

监控变更的灰度方式是"先看数据，再开通知"：

1. 加载新的 PrometheusRule，但把 severity 改成 none，让它产生告警但不通知；
2. 跑 24 小时，看有多少触发、触发时长、通知对象对不对；
3. 调整阈值和抑制规则；
4. 改成正式 severity。

## 十四、生产注意事项

### 14.1 指标端点的安全

- vLLM 的 /metrics 与 API 同端口，不要把 8000 端口暴露到公网；
- 用 NetworkPolicy 限制只有监控命名空间能访问；
- 如果必须跨集群采集，走 mTLS 的服务网格或 VPN，不要裸暴露；
- Prometheus 里如果需要 basic auth，用 authorization.credentials_file 指向 Secret 挂载的文件：

```yaml
scrape_configs:
  - job_name: vllm
    scheme: https
    tls_config:
      ca_file: /etc/prometheus/secrets/ca.crt
    authorization:
      type: Bearer
      credentials_file: /etc/prometheus/secrets/token
```

### 14.2 指标基数控制

推理服务的指标基数主要来自：

- model_name：每个模型一个值。一个实例只服务一个模型（vLLM 的多模型服务会有额外开销），不要在同一个 Prometheus 里混入几十个模型的 label 值；
- 自定义桶：桶边界数量 × 直方图数量 × 实例数；
- lora_requests_info：它把多个 LoRA 计数编码成逗号分隔的字符串放在标签值里，基数随 LoRA 数量增长，注意评估；
- HTTP 指标的 handler 标签：路径模板数量，可控。

一个粗略的估算方式：

```text
时间序列数 ≈ Σ(每个直方图的分位数族数) × (桶数 + 2) × 实例数
```

以 8 个直方图、10 个桶、10 个副本估算，约 8 × 12 × 10 = 960 条，再加 gauge/counter，量级在 1500 左右。这对 Prometheus 来说很轻松，但**不要在同一个实例上开 30 个自定义桶族**，那会变成几千条。

### 14.3 值班手册要点

值班交接时必须交代清楚的三件事：

1. 当前 SLO 状态：TTFT/TPOT 的 P95 与 P99，错误预算还剩多少；
2. 容量水位：KV Cache 使用率、队列深度常态值、副本数；
3. 已知异常：正在灰度的参数、最近的变更、已知会误触发的告警。

### 14.4 定期复盘

建议每季度做一次专项复盘，内容包括：

- 期间的 P0/P1 事件，逐条核对当时的告警是否及时、准确、有没有噪音；
- SLO 阈值与实际业务感知的偏差；
- 容量水位与实际峰值的偏差，是否需要提前扩容；
- 指标盲区：这几次故障里，有哪些信息是监控没覆盖到的，补上指标。

## 十五、总结

推理服务的监控和传统服务有本质区别，因为它的"慢"有多个独立的来源，必须拆开看。

**三条主线：**

1. 首 Token 延迟（TTFT） = tokenize + 排队 + prefill。排查时先用 request_queue_time_seconds 和 request_prefill_time_seconds 做二分，再往上下游追。
2. 生成速度（TPOT / ITL）。TPOT 看整体生成速率，ITL 看瞬时卡顿。抢占（vllm:num_preemptions）是 TPOT 劣化的第一嫌疑，KV Cache 不足是抢占的根因。
3. 请求排队。队列深度是瞬时值，必须配合排队时长的分位数一起看。排队的成因分三类：批次槽位、token 预算被长 prefill 挤占、KV Cache 不足，三者的处置完全不同。

**贯穿全文的一条判断链：**

```text
TTFT 异常
  ├─ 排队时长也涨 → 调度容量问题
  │    ├─ KV 使用率高 + 抢占涨 → KV Cache 不足（降 max-num-seqs / 限 max_tokens / 扩容）
  │    ├─ KV 低 + running 顶满 max-num-seqs → 批次槽位不足（调 max-num-seqs）
  │    └─ KV 低 + running 不高 + prompt 变长 → 长 prefill 挤占（调 max-num-batched-tokens / 限上下文）
  ├─ Prefill 时长涨 → 输入变长 / 前缀缓存失效 / GPU 降频
  ├─ 引擎全正常 → 网关排队 / tokenizer CPU / 客户端排队
  └─ 只有个别实例慢 → 路由不均 / 副本配置不一致 / 节点故障
```

**几条硬规矩：**

- 指标名以你实际版本的 /metrics 为准，落地前跑一遍自检脚本；
- 亚秒级 SLO 必须自定义直方图桶，并且桶边界要包含 SLO 阈值；
- 告警阈值来自压测基线和生产数据，不要抄别人的数字；
- 引擎参数变更都要重启，先灰度一个副本；
- 监控配置本身也要走变更流程，配 promtool check 和单元测试。

**最后一句**：推理服务的监控做得好不好，判断标准不是"指标多不多"，而是**故障发生时，你能不能在五分钟内，从大盘上看出问题在引擎内部还是外部，并说出下一步该做什么**。如果做不到，就从"TTFT、排队时长、KV Cache、抢占速率、生成吞吐"这五个面板开始补。

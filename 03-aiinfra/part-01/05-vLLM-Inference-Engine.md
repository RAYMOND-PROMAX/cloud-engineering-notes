# 05 · vLLM 推理引擎

> **本篇目标：**从请求生命周期理解 vLLM 如何管理推理任务、KV Cache、批次与 GPU 计算；理解关键参数为什么必须组合调节。

**范围与版本说明：**以下讨论以现代 vLLM 在线推理架构及 V1 引擎常见概念为主要背景。内部类名、默认参数、调度算法细节和 `/metrics` 指标会随版本变化，必须以所用版本的文档、`vllm serve --help`、日志与实测端点为准。

## 1. vLLM 解决什么问题

大语言模型的自回归生成不是一次性输出全文：每个请求可能有不同的 Prompt 长度、生成长度和到达时间。若固定批次、预先为每个请求分配最大上下文显存，就可能浪费 GPU 与显存。

vLLM 主要从两条路径提高资源使用效率：

- **Continuous Batching**：以迭代级别组织正在生成和新到达的请求，不必等完整批次全部结束再处理下一批。
- **PagedAttention / Paged KV Cache 管理**：把 KV Cache 按块管理，减少连续内存分配和预留最大长度带来的浪费。

注意：这些机制提升的是资源利用效率，不保证每一种输入分布、每一档 GPU 上都具有相同的吞吐收益。

## 2. 推理请求的逻辑链路

```text
Client / API 调用
    ↓
OpenAI-compatible API Server（HTTP、请求校验、Tokenization 等）
    ↓
Engine / Scheduler（请求状态、token budget、KV 资源）
    ↙           ↘
KV Cache Manager  Model Executor
    ↘           ↙
 CUDA Kernel / GPU 执行
    ↓
输出 token → 流式或非流式响应
```

此图是**职责图**，不是严格的一线程一节点拓扑。不同 vLLM 版本会拆分 API Server、Engine Core 和执行进程；GPU 执行和缓存管理在内部紧密配合。

## 3. Scheduler 如何处理请求

请求大致经历：

```text
到达 → WAITING → RUNNING（Prefill/Decode）→ FINISHED
                 ↑              |
                 └─ 资源压力下可能被抢占、重新排队
```

Scheduler 每个调度步都需要回答三个问题：

1. 哪些请求可以获得本次执行机会？
2. 本次可以处理多少 token（批次预算）？
3. 本次是否有足够的 KV Cache Block 供请求继续执行？

如果并发槽位不足、token budget 不足或 KV Block 紧张，都可能造成等待，但对应的调参方向不同。

## 4. Continuous Batching 与 Chunked Prefill

**Continuous Batching** 意味着请求可以在不同调度步进入或退出运行批次。它区别于“攒够一整个静态 Batch，直到所有请求结束才释放”的处理方式。

**Chunked Prefill** 允许较长 Prompt 的 Prefill 分成多个调度块，和其他请求的 Decode/Prefill 协调资源。

二者影响的是**调度和执行安排**。长输入依然消耗计算与 KV 空间；启用了 Chunked Prefill 并不代表 TTFT 一定变小，效果取决于负载分布与调度参数。

## 5. PagedAttention 与 KV Block

直观理解：模型需要为请求历史 token 存 K/V。如果按“请求最大上下文长度”连续预留空间，会留下大量空洞。Paged KV 管理将缓存划分为固定大小的 Block，通过映射关系把逻辑 token 序列关联到可用物理块。

```text
请求 A: [Block 3] → [Block 8] → [Block 21]
请求 B: [Block 5] → [Block 17]
               ↓
       GPU KV Block 池按需分配与回收
```

这张图只表达逻辑映射概念。不同版本可能存在共享前缀、引用计数、不同 KV 数据格式以及更复杂的块管理。

**粗略容量关系：**

```text
可驻留 KV token 上限 ≈ 可用 KV Block 数 × 每 Block 的 token 容量
并发容量 ≈ 可驻留 KV token 上限 / 单请求实际驻留 token 数
```

这只是显存侧估算；并发还受 `max_num_seqs`、执行负载、输入/输出分布、调度 token budget 等条件限制。

## 6. Prefix Caching 的工作方式

当多个请求拥有相同且可复用的 token 前缀时，引擎可能复用已经计算过的部分 KV 块，从而降低重复 Prefill 计算。

前缀缓存的收益依赖于：

- 前缀是否**完全匹配**，包括 token 序列及相关计算上下文；
- 路由是否把相似请求送到有可用缓存的实例；
- 缓存是否已经被回收或替换；
- 多租户安全策略是否允许复用（如适用版本支持 cache salt 等隔离措施）。

**不应为缓存命中率而绕过权限隔离。** 跨租户共享服务时，缓存安全属于独立设计目标。

## 7. 三个重要参数必须一起理解

| 参数 | 主要限制 | 太大可能造成 | 太小可能造成 |
| --- | --- | --- | --- |
| `--gpu-memory-utilization` | 推理实例计划使用的 GPU 显存比例预算 | 给其他进程余量不足、运行时 OOM 风险 | KV 空间偏少、并发受限 |
| `--max-model-len` | 单请求允许的模型序列长度 | 长请求占用更多可用缓存、容量规划保守 | 业务长上下文无法承载 |
| `--max-num-seqs` | 同时调度的序列数上限 | 竞争有限 KV 块、抖动和抢占 | GPU 资源可能用不满、等待增加 |

此外要关注 `--max-num-batched-tokens`（每步 token 预算）、模型精度与量化、Tensor Parallel 数、Prefix Cache 及实际模型配置。

**正确的调参顺序不是先把数值调大，而是：**

```text
确定业务输入/输出 token 分布
       ↓
选定模型、精度和 GPU 资源
       ↓
读取权重和运行时显存 / KV Block 数
       ↓
设置上下文与并发边界
       ↓
同时观察 TTFT、TPOT、吞吐、Waiting、抢占
       ↓
一次只变更一组有明确因果假设的参数
```

## 8. TP、DP、PP 是什么

| 类型 | 主要思想 | 关注点 |
| --- | --- | --- |
| Tensor Parallel（TP） | 将模型层内张量计算分布到多卡 | 跨卡通信、显存分摊、算子支持 |
| Data Parallel（DP） | 多份模型副本处理不同数据或请求 | 吞吐扩展、负载分配、每副本显存 |
| Pipeline Parallel（PP） | 不同层/阶段分布在不同设备 | 阶段均衡、流水线气泡、通信 |

并行度越高不一定越快。TP 受高速通信链路影响明显；DP 则通常需要每份副本都能装得下模型。训练和推理场景中的实现细节也不同。

## 9. 用什么证据判断内部压力

| 现象 | 首先观察 | 可能方向 |
| --- | --- | --- |
| TTFT 升高 | Waiting、Queue Time、Prefill Time | 排队或 Prefill 压力 |
| TPOT/ITL 变差 | Decode 时间、GPU 利用率与带宽、抢占 | Decode 竞争或设备层限制 |
| KV Cache 使用率高 | KV Block 容量、抢占计数 | 缓存容量压力 |
| Waiting 高而 KV 不高 | 活跃序列数、token budget、长 Prefill | 调度槽位/预算限制 |
| 模型启动失败 | 权重、显存、驱动、模型配置 | 启动阶段资源或兼容性问题 |

**版本陷阱：** vLLM V0/V1 的内部模型、参数和指标不完全一致，诸如早期 swap、swapped requests、cache usage 指标不能跨版本直接套用。

## 10. 自测问题

- PagedAttention 为什么不是“模型准确率优化”？
- `max_num_seqs` 已调大但 Waiting 还在增加，应该如何排查？
- Prefix Cache 命中率高，但 TTFT 不下降，可能还有哪些限制？
- TP=2 是否天然优于两个独立副本？要用什么数据回答？

版本默认参数值、启动日志格式、吞吐提升百分比和具体配置有效性；本文不引用来源文章中的收益数字作为本地实测。

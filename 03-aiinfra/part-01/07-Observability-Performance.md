# 07 · 可观测性与性能分析

> **本篇目标：**建立从用户体验到推理引擎、GPU 基础设施的观测链路，并学会通过证据区分排队、Prefill、Decode 与资源限制。

## 1. 为什么 GPU 利用率不足以代表服务质量

一台 GPU 可以达到很高的利用率，同时新请求不断进入等待队列，导致用户首字等待变长。传统 HTTP 的请求数、5xx 和 CPU 使用率无法独立解释这种现象。

推理服务需要至少三层观测：

```text
用户/服务层：成功率、端到端延迟、TTFT、业务 SLO
               ↑
引擎层：Queue、Prefill、Decode、Running、Waiting、KV Cache、Preemption
               ↑
基础设施层：GPU、CPU、内存、网络、磁盘、K8s Pod/Node
```

遇到异常时，应把三个层次按时间线对齐，而不是看到一条 GPU 曲线就直接调整参数。

## 2. 关键指标与统计口径

| 指标 | 描述 | 常见误读 |
| --- | --- | --- |
| TTFT | 首 token 时间 | 不同网关/引擎起点可能不同 |
| TPOT | 输出 token 平均生成间隔 | 不能反映所有逐 token 卡顿 |
| ITL | 相邻 token 的间隔分布 | 用平均 ITL 可能掩盖极端尾延迟 |
| E2E Latency | 全请求耗时 | 输入/输出长度变化会影响比较 |
| Queue Time | 在引擎等待调度的时间 | 不能代表网关前的等待 |
| Prefill Time | 初始上下文计算耗时 | 不一定包含前置 Tokenization |
| Decode Time | 输出生成阶段耗时 | 受 token 数和 batch 条件影响 |
| Output Tokens/s | 输出 token 吞吐 | 与 RPS 并非同一种吞吐单位 |
| KV Cache Usage | 缓存使用程度 | 高使用率不必然等于发生故障 |
| Preemption | 请求因资源压力被抢占 | 需要结合引擎版本与回退策略解释 |

**在报表里务必声明：时间从哪里开始、到哪里结束、以请求还是 token 为统计单位。**

## 3. 三类常见的排队原因

### A. 同时运行序列槽位达到限制

例如 Running 已到 `max_num_seqs` 对应边界，Waiting 增多，但 KV Cache 使用率未必很高。此时首先看调度槽位、流量分布和服务容量。

### B. Token Budget 被长 Prompt 占用

超长 Prompt 的 Prefill 需要较多计算和调度 token 预算。其他请求可能出现间歇性的 TTFT 尖刺。应观察请求长度分布、Prefill 延迟和 `max_num_batched_tokens` 等参数。

### C. KV Cache 容量紧张

Waiting 增长、KV 使用率高、抢占增加同时出现，说明缓存容量可能限制并发。可检查上下文长度、模型大小、KV 精度、显存预算与副本容量。

**还有第四种易误判场景**：网关或 Tokenizer 先排队，引擎 Waiting 指标仍可能很低。这时应把客户端、入口层与引擎层延迟拆开。

## 4. Metrics、Logs、Traces 各看什么

| 证据 | 擅长回答的问题 | 注意事项 |
| --- | --- | --- |
| Metrics | 什么时候变慢、影响多大、和什么指标关联 | 需管理标签基数与统计窗口 |
| Logs | 哪一个请求/进程出了什么错误 | 控制敏感信息，保持 Request ID |
| Traces | 请求跨网关、路由与应用各消耗多久 | GPU 内核级细节不一定由应用 Trace 覆盖 |
| GPU 遥测 | 显存、功率、设备健康、时钟与部分计算行为 | 设备级统计不等于单请求指标 |

在 Kubernetes 中，常用 Prometheus + Grafana 收集展示应用指标，DCGM Exporter 提供 NVIDIA GPU 遥测，Node Exporter / kube-state-metrics 补充系统和编排状态。

## 5. vLLM 指标必须进行版本核对

vLLM 的 `/metrics` 会随版本变更。例如有些版本使用 `kv_cache_usage_perc`，旧版可能出现 `gpu_cache_usage_perc`。某些 V0 时代的 swapped 请求指标在 V1 下不再具有相同意义。

阅读指标教程时，**先查现场真实暴露的指标名称，而不是先复制整套 PromQL**：

```bash
# 假设在允许的网络位置可以访问推理实例的 metrics 端点
curl -fsS http://<vllm-host>:8000/metrics | grep -E 'vllm:|vllm_'
```

上面的 Host 是占位符，不能原样执行。若指标出于安全原因只在内网暴露，应通过经授权的管理路径查询。

Prometheus Histogram 的 P95 一般按 `histogram_quantile(0.95, sum by (le, ...)(rate(<metric>_bucket[5m])))` 计算，但**具体 `<metric>` 名称、模型标签以及是否具有 bucket 必须现场核对**。不同系统/采样口径之间不能直接对比。

## 6. 排障决策矩阵

| 观测组合 | 优先假设 | 下一步证据 |
| --- | --- | --- |
| TTFT↑、Queue Time↑、Waiting↑ | 排队容量不足 | Running 上限、请求长度、扩容边界 |
| TTFT↑、Queue Time→、Prefill Time↑ | Prefill 计算压力 | Prompt 长度、Prefix Cache 命中率 |
| TPOT↑、ITL P99↑、GPU 忙 | Decode 受资源约束 | Batch、设备利用、KV、通信 |
| Waiting↑、KV Usage↑、Preemption↑ | KV 容量压力 | 缓存预算、上下文分布、序列数 |
| Waiting→、KV Usage→、E2E↑ | 引擎外部延迟 | 网关、分词、网络、下游 |
| Pod Pending↑、引擎指标缺失 | K8s/设备资源问题 | Events、Allocatable、Device Plugin |

箭头表示趋势变化，不代表单独满足某行就能确认根因；需要控制输入长度和并发等变量交叉验证。

## 7. 基线与压力测试的关系

为能比较不同配置，至少固定以下条件：

- 模型版本、量化格式、推理引擎版本；
- GPU 型号、设备数、拓扑和并行方式；
- 输入/输出 token 长度分布、是否流式；
- 请求到达模型（固定并发还是固定 RPS）、预热策略；
- TTFT/TPOT/吞吐/错误率的统计窗口与分位数。

性能验证应同时观察**吞吐、P95/P99 延迟、Waiting、KV、显存和错误率**。只报告“最快 1 个请求”不构成系统定容结论。

## 8. 告警不能只有阈值

设计告警时需定义：触发条件、持续时间、影响级别、降噪分组、责任方、处置入口和恢复条件。

例如 TTFT P95 超过阈值时，应考虑是否属于低流量样本、是否伴随 Queue Time 和 Waiting 的增加，以及是否只是某一个副本异常。阈值必须根据业务 SLO 和正常流量基线制定，不能套用参考文章里的示例值。

## 9. 自测问题

- 为什么 GPU 利用率 95%，TTFT 仍可能不断升高？
- TTFT 和 TPOT 哪个更适合发现首字等待，哪个更适合发现生成慢？
- Waiting 很高而 KV Usage 很低，说明应排查什么？
- 为什么监控系统升级后需要重新核对指标名称和标签？


# Part I · AI Infrastructure 技术教学

> 先学系统原理、架构和运行机制；综合部署、压测、故障演练放在 Part II 实战部分。

这一部分围绕通用 AI 基础设施知识组织。每篇文档都尽可能明确技术职责、运行链路、观察方法、容易混淆的概念和进一步学习的问题。

## 章节目录

| 编号 | 教学章节 | 主要内容 |
| --- | --- | --- |
| 01 | [GPU 与计算环境](./01-GPU-Computing-Environment.md) | GPU、显存、Driver/CUDA、设备与容器链路 |
| 02 | [Kubernetes GPU 资源管理](./02-Kubernetes-GPU-Resources.md) | GPU Operator、Device Plugin、扩展资源、Pod 调度与配额 |
| 03 | [GPU 共享与集群调度](./03-GPU-Sharing-Scheduling.md) | Time Slicing、MPS、MIG、Run:ai、Volcano、Gang Scheduling |
| 04 | [大模型推理基础](./04-LLM-Inference-Fundamentals.md) | Token、Prefill、Decode、KV Cache、延迟与吞吐 |
| 05 | [vLLM 推理引擎](./05-vLLM-Inference-Engine.md) | PagedAttention、Continuous Batching、Scheduler、关键参数 |
| 06 | [模型服务与弹性](./06-Model-Serving-Elasticity.md) | API、网关、K8s Deployment、就绪、伸缩、灰度与回滚 |
| 07 | [可观测性与性能分析](./07-Observability-Performance.md) | TTFT、TPOT、ITL、Queue、KV、Prometheus 与 GPU 监控 |
| 08 | [训练与微调基础](./08-Training-Finetuning-Fundamentals.md) | SFT、LoRA、分布式训练、Checkpoint（待增加训练专题来源） |
| 09 | [安全与可靠性](./09-Security-Reliability.md) | 多租户、安全边界、变更、可靠性与故障恢复（待补充来源） |

## 推荐阅读路径

先读 **01 → 02 → 03** 理解算力与调度底座，再读 **04 → 05 → 06 → 07** 建立完整的在线推理系统认知。**08** 和 **09** 是相对独立但必要的横向主题；目前两篇只提供基础框架，后续继续扩充。

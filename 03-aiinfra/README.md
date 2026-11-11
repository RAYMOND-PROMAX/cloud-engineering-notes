# 第三章：AI Infrastructure

> 专注于算力，坚守安全，追求稳定。

本章记录我对 **GPU 算力基础设施、大模型训练与推理、云原生调度及生产运维** 的学习和工程实践。

AI Infrastructure 不只是安装驱动、部署 Kubernetes，再用一条命令启动 vLLM。真正的工程问题是：**如何把算力变成可交付、可观察、可维护，并能持续服务业务的模型能力。**

## 1. 学习目标

以企业私有化 GPU 大模型平台为主线，逐步建立以下能力：

- **算力基础设施**：理解 GPU 硬件、驱动、CUDA、容器运行时与 Kubernetes 的依赖关系。
- **资源管理**：规划 GPU 资源池、工作负载隔离、训练与推理资源分配及弹性调度。
- **推理服务**：理解从 API 入口到 vLLM Scheduler、KV Cache、Executor 和 GPU 的完整请求路径。
- **训练工程**：掌握数据准备、SFT、LoRA/QLoRA、Checkpoint、评测及任务恢复的实际流程。
- **可观测性与排障**：关联请求指标、GPU 指标、系统日志和部署状态，完成故障定位与验证。
- **安全与稳定性**：关注访问控制、数据边界、版本管理、压力测试、灰度发布及回滚。
- **业务验证**：用真实场景、质量指标和运行成本判断平台是否具有交付价值。

## 2. 工程实践背景

本章以参与建设的企业私有化 GPU 平台为主要参考场景，并在个人实验环境中拆解和验证关键机制。生产项目经验与本地实验结果分别记录，不混用数据。

| 层次         | 参考配置或技术                                         | 关注点                                  |
| ------------ | ------------------------------------------------------ | --------------------------------------- |
| GPU 计算节点 | 4 × Dell R760xa；每台 4 × NVIDIA L40S 48GB             | 16 GPU，合计 768GB 显存；硬件与资源规划 |
| 控制节点     | 3 × Dell R660                                          | Kubernetes 控制平面与平台基础服务       |
| 基础软件     | Linux、Docker / Container Runtime、NVIDIA Driver、CUDA | 驱动与运行时兼容性                      |
| 资源编排     | Kubernetes、NVIDIA GPU Operator                        | GPU 发现、分配、隔离、任务调度          |
| 推理服务     | HAProxy、Envoy、Inference Router、vLLM                 | 入口、模型路由、请求排队、生成与返回    |
| 模型         | Qwen2.5-32B、14B、BGE-M3、Reranker 等                  | 模型资源需求与服务部署                  |
| 训练与评测   | PyTorch、LoRA / QLoRA、SFT、Checkpoint                 | 训练任务、续训、评估与版本管理          |
| 可观测性     | Prometheus、Grafana、DCGM Exporter                     | 服务指标、GPU 状态、告警与故障证据      |

> 注意：上表是学习案例的参考环境，不代表所有组件、功能或优化方案均已在同一生产版本中实现。具体文章应分别注明「生产实践」「实验验证」或「方案研究」。

## 3. 平台架构与主要链路

### 3.1 推理链路（Inference Serving）

```text
Client
  -> HAProxy
  -> Envoy
  -> Inference Router
  -> vLLM API Server
  -> Scheduler / KV Cache Manager
  -> Model Executor
  -> CUDA / NCCL / GPU
  -> Streaming Response
```

重点研究：请求如何进入、在哪里排队、连续批处理如何调度、KV Cache 如何管理、跨 GPU 通信在何时发生，以及如何观察 TTFT、TPOT、吞吐量和错误率。

### 3.2 训练链路（Model Training）

```text
Data Preparation
  -> Dataset Validation
  -> SFT / LoRA / QLoRA
  -> Checkpoint
  -> Evaluation
  -> Model / Adapter Artifact
```

重点研究：训练数据质量、显存占用、训练状态、Checkpoint 保存与恢复、评测集以及训练任务对在线推理资源的影响。

### 3.3 发布链路（Model Delivery）

```text
Model Artifact
  -> Compatibility Check
  -> Deployment
  -> API / Functional Test
  -> Load Test
  -> Canary / Release
  -> Monitoring / Rollback
```

重点研究：模型和环境版本、服务可用性、并发能力、回滚条件，以及「能启动」与「能交付」之间的差距。

## 4. 学习与实践目录规划

后续文章按下列主题组织，目录会随实验逐步建立：

| 目录                       | 研究主题         | 典型实践                                               |
| -------------------------- | ---------------- | ------------------------------------------------------ |
| `01-GPU-Fundamentals/`     | GPU 硬件与运行时 | L40S、显存、PCIe、NUMA、NVIDIA Driver、CUDA、NCCL      |
| `02-Kubernetes-GPU/`       | GPU 集群与调度   | GPU Operator、Device Plugin、配额、工作负载与资源隔离  |
| `03-LLM-Inference/`        | 大模型推理       | vLLM 部署、TP、Continuous Batching、KV Cache、流式响应 |
| `04-LLM-Training/`         | 微调训练         | SFT、LoRA、QLoRA、Checkpoint、训练恢复                 |
| `05-Observability/`        | 监控与性能分析   | DCGM、Prometheus、Grafana、TTFT、TPOT、吞吐、显存      |
| `06-Reliability-Security/` | 稳定性与安全     | 故障演练、访问控制、容量规划、回滚与应急预案           |
| `07-Experiments/`          | 实验与验证       | 基线、压测、参数对照、异常注入、结果复盘               |
| `08-Runbooks/`             | 运维知识沉淀     | 部署 SOP、巡检、故障 RCA、恢复验证                     |

这些路径是**计划中的知识目录**，不是当前已完成内容的声明。

## 5. 统一分析方法

对一个完整项目，按六个维度进行研究：

1. **Operations**：如何部署、配置、启动、维护和恢复？
2. **Mechanisms**：请求、任务与资源在系统内部如何流动？
3. **Engineering**：关键参数、容量、安全边界与异常处理如何控制？
4. **Statistical Observation**：延迟、吞吐、错误率和资源利用率如何变化？
5. **Mechanism-level Observation**：队列、缓存、调度、通信等机制是否符合预期？
6. **Validation**：功能、稳定性和业务效果是否达到目标？

对每一个生命周期节点，则记录统一的八项要素：

| 要素                   | 核心问题                                  |
| ---------------------- | ----------------------------------------- |
| Stage / Node           | 当前处于哪个阶段或组件？                  |
| Input                  | 输入数据、请求或任务是什么？              |
| Resource               | 消耗哪些 CPU、内存、GPU、网络和存储资源？ |
| Core Mechanism         | 内部怎样处理？                            |
| Key Control Parameters | 可以调整哪些关键变量？                    |
| Local Metrics          | 怎样判断本节点是否正常？                  |
| Global Metrics         | 对端到端性能和可靠性有什么影响？          |
| Output                 | 输出是什么，下游依赖什么？                |

## 6. 重点实验方向

逐步完成以下可复现的实验，而不是仅停留在安装成功：

- **环境验证**：驱动、CUDA Runtime、容器 GPU 可见性与 Operator 状态检查。
- **模型启动**：记录模型版本、精度、TP、最大上下文长度、GPU 显存占用和启动日志。
- **推理基线**：区分冷启动和预热后的 TTFT、TPOT、吞吐量及错误率。
- **并发与队列**：改变请求并发、输入/输出长度，观察排队、批处理和 KV Cache 压力。
- **资源与调度**：验证在线推理、离线训练及批任务之间的资源隔离和调度边界。
- **故障实验**：模拟模型加载失败、GPU OOM、Pod 重启、服务超时和依赖不可用，记录排障路径。
- **发布验证**：部署新版本，完成 API 验证、压力测试、灰度与回滚演练。

每一次实验都尽量保留环境版本、命令、配置、原始数据和结论；没有测量的数据，不写成优化成果。

## 7. 实验记录模板

```markdown
# 实验名称

## 目标与假设
- 要验证什么？为什么？

## 环境与版本
- 硬件、驱动、CUDA、Kubernetes、vLLM、模型版本

## 基线
- 并发、输入/输出 Token、TTFT、TPOT、吞吐、GPU 使用情况

## 操作与变量
- 命令、配置、一次只改变的关键参数

## 观测证据
- 指标、日志、事件、截图、原始测试数据

## 分析与结论
- 结果是否支持假设？原因与边界是什么？

## 回滚与复盘
- 如何恢复？下次如何更快定位？
```

## 8. 本章的工程原则

**先明确需求，再选择架构。** 不因为某项技术流行，就默认它适合当前规模和场景。

**先建立基线，再谈性能优化。** 将「现象」「推测」「证据」和「结论」分开记录。

**优先稳定与安全，再追求极限吞吐。** 生产平台除了跑得快，还需要能观测、能恢复、能交接。

**先完成小闭环，再逐步扩展。** 从一套可验证的模型服务开始，再扩展训练、资源调度和复杂业务应用。

**沉淀可复用的方法，而不是只记录命令。** 每一次部署、故障和实验最终都应形成 SOP、Runbook 或 RCA。

---

> 从 GPU 资源到模型服务，从一次部署到长期稳定运行。
>
> **专注于算力，坚守安全，追求稳定。**
# Part II · AI Infrastructure 工程实战（Hands-on Labs）

> **按工程目标逐步完成：环境可用 → 单机模型服务 → GPU 集群 → Kubernetes 推理 → 可观测与压测 → 共享调度 → 故障恢复 → 基础训练。**

这是 [Part I · 技术教学](../Part-I-Fundamentals/README.md) 的实践部分。**全部实验采用通用环境和公开示例，不使用个人或企业项目案例**。命令与 YAML 是拟复现的实验草案，不包含虚构的运行截图、故障经历或性能数据。

## 实验目录

| Lab | 名称 | 前置依赖 | 本篇交付的证据 |
| --- | --- | --- | --- |
| 01 | [GPU 环境搭建与基础验收](./Lab-01-GPU-Environment-Acceptance.md) | Linux、NVIDIA GPU、Docker | Driver / Toolkit / 容器 GPU 可见性 |
| 02 | [单机 vLLM 推理服务](./Lab-02-Single-Node-vLLM.md) | Lab 01 | 健康检查、模型 API、日志、Metrics |
| 03 | [Kubernetes GPU 调度环境](./Lab-03-Kubernetes-GPU-Baseline.md) | 已有隔离 K8s 集群、GPU Worker | Operator/Device Plugin、调度、配额与事件 |
| 04 | [Kubernetes 部署 vLLM](./Lab-04-vLLM-on-Kubernetes.md) | Lab 03 | Deployment、Service、Probe、变更/回退 |
| 05 | [推理监控与压力测试](./Lab-05-Observability-and-Load-Testing.md) | Lab 02 或 04 | Client TTFT/E2E、引擎 Queue、并发基线 |
| 06 | [GPU 共享与调度边界](./Lab-06-GPU-Sharing-and-Scheduling.md) | Lab 03、专用 GPU Worker | Time Slicing、资源名、Node Allocatable、回退 |
| 07 | [故障注入与恢复](./Lab-07-Fault-Injection-and-Recovery.md) | 独立实验 K8s；部分步骤需 Lab 04 | OOMKilled、Pending、启动异常、回归验收 |
| 08 | [LoRA 微调与 Checkpoint](./Lab-08-LoRA-and-Checkpoint.md) | Lab 01、Python/PyTorch GPU 环境 | 合成数据 SFT、Adapter、恢复训练、资源观察 |

## 推荐操作路线

```text
GPU 主机 ── Lab 01 ── Lab 02 ────────────────── Lab 05
      └─── K8s 集群 ── Lab 03 ── Lab 04 ──┬── Lab 07
                                   └──────┴── Lab 06（另一个独立实验窗口）
      └─── Python / PyTorch ──────────────── Lab 08
```

- **请不要并行运行所有 Lab。** 一张 GPU 上同时部署 vLLM、训练和多个共享 Pod，会导致资源竞争，使观察结果不可靠。
- **实验资源需要隔离。** `ai-labs`、`fault-labs`、`gpu-share-labs` 等名称可自定义，避免与既有 Namespace 冲突。
- **不同实验使用了不同权限级别。** Lab 03 / 06 修改 GPU 集群组件；Lab 07 故障注入；这些仅适用于独立实验环境。
- **版本不是永久有效常量。** 例子中固定了一些镜像/包的参考 tag，但必须核对 GPU Driver、CUDA、K8s、GPU Operator、vLLM、TRL 的兼容关系，并以实际实验记录更新。

## 实验报告模板

每篇文章按照统一闭环进行：

1. 实验目标及验收条件
2. 环境与版本快照
3. 操作步骤和配置理由
4. 指标、日志、Events 等证据
5. 假设、异常与排查过程
6. 验收与对照实验
7. 清理、回退及复盘

建议每个实验都保存一份独立记录：`date`、`git rev`、`nvidia-smi`、`kubectl version`、`helm list`、`docker image inspect` / `pip freeze`、配置文件、原始指标、异常日志和验证结论。

## 版本及事实声明

- **状态：v0.1 / 待实验复核。** 文档在当前对话环境中完成编写与静态检查；没有 NVIDIA GPU 或可用 Kubernetes 集群供实际运行，因此不存在已执行的压测、训练、故障恢复结果。
- 来源包括此前提供的 7 篇去重技术文章，以及 NVIDIA、Kubernetes、vLLM、Hugging Face 的官方文档；出现分歧时，应优先按**对应版本**的官方文档与实际观察记录判断。
- Part II 是通用公开教学材料，不是对任何个人项目的描述，也不声称已经做过全部工程实验。

## 计划补充（不影响本版实验入门）

后续可以单独增加：Kueue / Volcano 批量调度、KServe/KEDA 弹性、推理网关、多副本发布、训练 QLoRA / DDP / DeepSpeed、GPU/网络通信拓扑、对象存储与模型缓存、评测与安全测试。当前版本不会把它们塞入没有验证条件的“万能一键部署”。

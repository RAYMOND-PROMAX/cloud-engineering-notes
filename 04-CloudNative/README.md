# 第四章：云原生（Cloud Native）

> 从容器到编排、从声明式交付到可观测：理解云原生系统如何构建、运行与演进。

云原生不是一组工具的集合，而是一套围绕**不可变基础设施、声明式 API 与自动化控制循环**形成的工程方法。容器解决了"运行环境一致"，Kubernetes 解决了"调度与自愈"，而真正决定系统能否长期稳定运行的，是资源模型、发布流程、可观测性与权限边界。

本章记录本人在容器、Kubernetes、Service Mesh、GitOps 与云原生可观测性方向的原理梳理、工程实践与故障分析。

重点关注技术背后的运行机制：一个 Pod 为什么被调度到某台节点、一次发布为什么回滚、一次网络抖动的证据链在哪里。

## 学习目标

通过本章的学习与实践，希望能够：

- 理解容器与镜像的分层机制、命名空间与 cgroup 的资源隔离边界。
- 掌握 Kubernetes 的核心对象模型与声明式控制循环。
- 理解 Pod 调度、探针、扩缩容与滚动发布的实际行为。
- 理解云原生网络模型（CNI、Service、Ingress、NetworkPolicy）与常见故障点。
- 理解有状态服务在云原生环境下的存储与数据一致性约束。
- 使用 Metrics / Logs / Traces / Events 定位问题，并区分"平台问题"与"应用问题"。
- 建立从变更、观测到回滚的闭环交付习惯。

## 研究方向（Research Areas）

| 技术方向 | 主要内容 |
| --- | --- |
| 容器与镜像 | Namespace、Cgroup、镜像分层、构建优化与运行时（containerd） |
| Kubernetes 核心 | 对象模型、Controller、Scheduler、Admission、CRD 与 Operator |
| 工作负载与弹性 | Deployment / StatefulSet / DaemonSet、探针、HPA / VPA / KEDA |
| 云原生网络 | CNI、Service / kube-proxy / IPVS、Ingress、NetworkPolicy、DNS |
| 存储与有状态服务 | PV / PVC / StorageClass、CSI、有状态迁移与备份恢复 |
| 交付与变更 | GitOps、CI/CD、Helm / Kustomize、灰度发布与回滚 |
| 可观测性 | Metrics、Logs、Traces、Events、告警治理与成本 |
| 安全与多租户 | RBAC、ServiceAccount、镜像供应链、网络隔离与资源配额 |

## 技术文章目录（Technical Articles）

本部分收录云原生方向的原理分析与工程实践文章，围绕具体问题说明运行机制、方案取舍与验证方式。

*技术文章持续整理中。*

## 文档原则

- **机制优先**：说明"系统为什么这样表现"，而不只记录命令与 YAML。
- **结果可验证**：给出观察指标、日志或事件作为判断依据。
- **区分环境**：标注 Kubernetes 版本、容器运行时、CNI/CSI 实现与云厂商差异。
- **保留边界**：写清方案不适用的场景与失效条件。

## 当前进度

目前处于知识框架搭建阶段，优先整理容器运行时、Kubernetes 对象模型、调度与网络、交付流程与可观测性。

> 与 AI Infrastructure 的分工：本章讲通用的云原生底座；GPU 调度、推理服务与模型工程化属于第三章。

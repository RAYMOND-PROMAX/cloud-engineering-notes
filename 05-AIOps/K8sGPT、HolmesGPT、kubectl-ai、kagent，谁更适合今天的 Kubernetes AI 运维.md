# K8sGPT、HolmesGPT、kubectl-ai、kagent，谁更适合今天的 Kubernetes AI 运维？

过去两三年，Kubernetes 与大模型结合得越来越紧。

到了 2026 年，方向又进一步演进到了 Agent、MCP、多 Agent 协同和自动化运维。在这个过程中，**K8sGPT**、**HolmesGPT**、**kubectl-ai** 和 **kagent** 是四个非常有代表性的项目。

但它们其实并不是完全同类产品。如果用一句话概括它们各自的位置：

K8sGPT —— 诊断器

HolmesGPT —— 事故调查员

kubectl-ai —— AI 版 kubectl

kagent —— Kubernetes 上的 Agent 平台

先分清角色，再谈选型

理解这一点，比简单比较谁的 Star 更多更重要。

## 没有过时，但定位已经变了

### K8sGPT

K8sGPT 是这几类工具里最早获得广泛关注的项目之一，目前依然保持维护，也是 CNCF Sandbox 项目。

它最核心的设计思想其实很简单：

也就是说，它并不是完全让大模型自由发挥。Pod、Deployment、Service、PVC、Ingress、NetworkPolicy 等资源都有相应的 Analyzer，先通过确定性规则发现问题，再让 LLM 把问题翻译成人能看懂的语言。

这套设计最大的优点是：**简单、稳定、风险低**。

例如出现 CrashLoopBackOff、ImagePullBackOff、PVC 异常、调度失败时，K8sGPT 很适合快速给出解释和排查建议。

它的安装和使用成本也很低，非常适合个人开发者、小团队，或者作为第一层 Kubernetes 健康检查工具。

但它的局限也很明显。真实生产事故往往不是某一个 Kubernetes Object 单独出错，而可能是：

这种跨 Kubernetes、日志、指标、数据库、发布系统的复杂因果链，仅靠 Analyzer 很难还原。

所以 K8sGPT 并没有过时。更准确地说，它正在从早期的 AI for Kubernetes 明星项目，逐渐回归到一个很合理的位置。

轻量级 Kubernetes AI Scanner

## 更像真正的 AI SRE

### HolmesGPT

如果说 K8sGPT 更关注哪里出了问题，那么 HolmesGPT 更关注的是：为什么会出问题？这是两者最大的区别。

HolmesGPT 的设计更接近一个调查型 Agent。它可以根据问题不断调用不同工具。

它能够接入 Kubernetes、Prometheus、Loki、Grafana、Tempo、Elasticsearch、Datadog、ArgoCD、AWS、Azure、GCP、数据库等多个系统。

这意味着 HolmesGPT 面对的已经不只是 Kubernetes Object，而是整个生产环境。

例如你问：

PROMPT 为什么 payment-service 下午两点开始延迟突然升高？

它可以进一步检查：

有没有新版本发布？

Pod 有没有重启？

CPU 和 Memory 是否异常？

Prometheus 中延迟什么时候开始升高？

Loki 有没有 timeout？

数据库连接池是否耗尽？

最近有没有 ArgoCD 部署？

这才比较接近真正意义上的 **Root Cause Analysis，根因分析**。

因此，如果企业已经建设了 Prometheus、Loki、Grafana、Tempo、ArgoCD 等可观测性体系，HolmesGPT 的价值会明显高于 K8sGPT。

但代价也存在。Agent 需要多轮调用工具和模型，意味着Token 成本、响应时间以及系统复杂度都会提高。而且它的效果高度依赖数据质量。

！真正的前提 🕳

如果日志不完整、指标没接好、Trace 缺失，再强的 Agent 也很难找到真正根因。

## 最像 Kubernetes Copilot

### kubectl-ai

kubectl-ai 走的是另外一条路线。它并不试图成为一套完整的 SRE 平台，而是解决一个非常直接的问题：能不能少写一点 kubectl？

传统操作可能是：

kubectl get pods -A

kubectl describe pod xxx

kubectl logs xxx

kubectl get events

用了 kubectl-ai 以后，可以直接问：

PROMPT 为什么 nginx Pod 起不来？

PROMPT 帮我看看 default namespace 哪些 Pod 有问题。

Agent 再自己调用 kubectl 或 shell 完成操作。所以它更像：

kubectl + Copilot

对于开发人员、平台工程师以及日常 Kubernetes 运维来说，这种体验其实非常自然。

它的优势也很明确：

上手快

交互简单

模型支持广

可以接入 MCP

但 kubectl-ai 最大的问题也是它最强的地方：它离真实执行动作太近了。

一旦允许 Agent 执行 kubectl、patch、scale、delete 等操作，就必须认真设计 RBAC、ServiceAccount、审批和认证机制。

生产环境千万不能把LLM 会不会询问确认当作真正的安全机制。真正的安全边界永远应该是 **Kubernetes RBAC、IAM 和权限治理**。

## 它已经不是简单的排障工具

### kagent

四个项目中，kagent 是最容易被放错位置的一个。

很多人会直接比较：K8sGPT vs HolmesGPT vs kubectl-ai vs kagent。但严格来说，kagent 已经不是单纯的 Kubernetes Troubleshooting Tool。

它更像：

kagent 的真实身份

Kubernetes-native AI Agent Platform

它关注的是如何在 Kubernetes 中定义、运行和管理 Agent。

未来你完全可以在一套 kagent 环境里运行：

Kubernetes Agent

Observability Agent

Security Agent

Cost Agent

Network Agent

再通过一个 Planner Agent 协调多个 Agent。

也就是说，Kubernetes 以前管理的是 Pod、Deployment、Service、Job。而 kagent 希望继续管理 **Agent、Tool、MCP Server、Session、Agent Runtime**。

它天然拥抱 MCP、多 Agent、OpenTelemetry、GitOps 等能力。

因此，如果团队想建设的是一套企业内部 Agentic Ops 平台，kagent 的想象空间最大。

但问题也非常明显：复杂。如果你只是想查一个 Pod 为什么 CrashLoopBackOff，那么部署一整套 Agent 平台显然没有必要。

kagent 更适合平台团队，而不是普通开发者拿来临时排障。

## 四个工具应该怎么选？

如果把四个项目放在一起，可以简单理解为：

| 工具 | 一句话定位 | 最适合的场景 |
| --- | --- | --- |
| K8sGPT | 诊断器 | Kubernetes 快速扫描、轻量诊断 |
| HolmesGPT | 事故调查员 | 复杂生产事故、自动 RCA |
| kubectl-ai | AI 版 kubectl | 日常 Kubernetes 操作 |
| kagent | Agent 平台 | 企业级 Agent 平台建设 |

如果你只是想快速检查集群状态，K8sGPT 依然值得用。

如果已经有比较成熟的 Observability 体系，并且希望让 AI 真正参与事故调查，HolmesGPT 更值得重点关注。

如果主要目标是提高工程师使用 kubectl 的效率，kubectl-ai 的体验最直接。

如果你的目标是建设下一代 AI SRE、Agentic Ops 或企业内部 Agent 平台，那么真正值得深入研究的是 kagent。

///

LAST

K8sGPT 过时了吗？

CONCLUSION · 分层组合

答案是：**没有过时，但时代正在往前走**。

第一阶段

让 AI 解释 K8s

第二阶段

让 AI 调查 K8s

第三阶段

Agent 成为基础设施

K8sGPT → HolmesGPT → kagent

所以 Kubernetes AI 的变化，本质上可以概括成一句话：

从**让 AI 解释 Kubernetes**，逐渐走向**让 AI 调查 Kubernetes**，最终走向**让 Agent 成为云原生基础设施的一部分**。

因此，如果今天重新做一个 Kubernetes AI 项目，我不会简单地问：K8sGPT、HolmesGPT、kubectl-ai、kagent 到底谁最好？

✦ 更值得问的问题

哪些事情应该交给规则，哪些交给 Agent，哪些 Agent 又应该通过统一的平台治理？

未来真正成熟的 AI SRE 架构，大概率也不会是四选一，而是分层组合：

K8sGPT 可以负责前面的轻量诊断，HolmesGPT 负责调查，MCP 负责连接工具，而 kagent 负责管理 Agent。

从这个角度看，Kubernetes AI 的主战场，已经开始从 AI 帮你解释错误，转向：

**Observability** + **Agent** + **MCP** + **权限治理** + **自动化闭环**

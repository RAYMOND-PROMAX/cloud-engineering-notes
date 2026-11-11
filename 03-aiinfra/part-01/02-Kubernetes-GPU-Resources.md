# 02 · Kubernetes GPU 资源管理

> **本篇目标：**解释一张 GPU 如何变成 Kubernetes 可申请的资源，以及为什么“设备空闲但 Pod Pending”并不矛盾。

**适用范围：**Linux + NVIDIA GPU + Kubernetes 原生 Device Plugin 扩展资源模型。DRA 属于不同资源分配路径，本篇仅介绍概念，不提供未经版本核实的 DRA 清单。

## 1. Kubernetes 为什么不能直接调度 GPU

CPU、内存是 Kubernetes 的基础资源；GPU 通常通过 **Device Plugin** 注册为扩展资源（Extended Resource）。Kubernetes 原生调度器基于资源申请量与节点可分配量作出绑定决策，不直接依据 `nvidia-smi` 的实时 GPU 利用率调度。

```text
NVIDIA 驱动识别物理 GPU
        ↓
NVIDIA Device Plugin 枚举、上报设备
        ↓ 通过 kubelet Device Plugin API
kubelet 更新 Node Capacity / Allocatable
        ↓
kube-scheduler 按 Pod 请求与调度约束选节点
        ↓
kubelet 调用设备分配接口并启动容器
        ↓
容器运行时暴露指定 GPU 设备
```

上面是逻辑链路，不意味着 GPU Operator 和 Device Plugin 是同一个组件。

## 2. GPU Operator 与 Device Plugin 分工

| 组件 | 主要责任 | 不负责什么 |
| --- | --- | --- |
| GPU Driver | 操作系统与 GPU 驱动交互 | 不进行 K8s Pod 调度 |
| NVIDIA Container Toolkit | 容器访问 GPU 的运行时支持 | 不管理队列和租户公平性 |
| GPU Device Plugin | 发现、注册、分配设备资源 | 不决定哪个 Pod 优先排队 |
| GPU Operator | 通过 Operator 模式管理驱动、插件、工具等组件生命周期 | 本身不是推理请求调度器 |
| kube-scheduler | 根据资源、亲和性、污点等选择节点 | 不负责 vLLM 内部请求调度 |
| kubelet | 维护节点状态、执行容器启动与设备分配 | 不负责业务层的模型路由 |

GPU Operator 的具体部署能力取决于 ClusterPolicy、驱动是否由宿主机预装、容器运行时与插件版本。不要把所有安装策略写成一套固定命令。

## 3. 扩展资源的语义

常见资源名是 `nvidia.com/gpu`，典型声明形式：

```yaml
resources:
  limits:
    nvidia.com/gpu: 1
```

关键规则：

- **整数资源**：原生扩展资源不直接申请 `0.5` 张 GPU；可采用设备插件提供的共享或切分资源来满足不同需求。
- **GPU request 与 limit 应一致**：只写 limit 时 Kubernetes 可将其视作 request；若同时指定，两者须一致。
- **调度按账本**：已被 Pod 申请的 GPU，即使实际计算利用率只有 5%，也通常被算作占用。
- **Capacity 与 Allocatable 不同**：调度看可分配量和已分配申请，不仅看物理卡数。

只有在明确配置了共享/切分方式时，“一张物理 GPU 对应多个可调度资源单位”才有可能成立。

## 4. Pod 为什么会 Pending

常见原因并非只有 GPU 不够：

| 现象或事件 | 可能原因 | 先查哪里 |
| --- | --- | --- |
| Insufficient nvidia.com/gpu | GPU 资源申请超出可分配余量 | `kubectl describe node`、Pod requests |
| Untolerated taint | GPU 节点带有污点但 Pod 无容忍 | Node Taints、Pod Tolerations |
| Node affinity mismatch | 节点标签与亲和条件不匹配 | Node Labels、Pod Affinity |
| Exceeded quota | 命名空间资源配额不足 | ResourceQuota、Events |
| GPU 字段消失 | 插件、kubelet 或驱动异常 | Device Plugin Pod、节点状态 |
| Pod Running 但 CUDA 不可用 | 设备注入/Runtime/镜像兼容问题 | 容器日志与设备可见性 |

**排查逻辑：先控制面账本、再节点插件、最后设备和容器内部。**

### 只读排查入口

```bash
kubectl describe pod <pod-name> -n <namespace>
kubectl describe node <node-name>
kubectl get nodes -o custom-columns=NAME:.metadata.name,GPU:.status.allocatable.nvidia\.com/gpu
kubectl get resourcequota -A
kubectl get events -A --sort-by=.lastTimestamp
```

注意：命令中的资源字段转义和输出列可能依 shell/客户端环境有所不同，实际使用时可退回 `kubectl get node -o yaml` 检查。

## 5. 三种不同的资源控制

1. **节点级资源**：`Allocatable - 已申请` 决定调度余量。
2. **命名空间级配额**：`ResourceQuota` 限制租户合计可以申请多少 GPU。
3. **工作负载级限制**：单个 Pod 的 GPU 申请、PriorityClass、节点亲和与拓扑约束决定放置和优先级。

ResourceQuota 不会自动提供强隔离；Namespace 不等于 GPU 显存隔离。多租户还需权限、网络、存储和共享策略共同设计。

## 6. Placement：选择在哪个节点运行

- `nodeSelector`：按标签进行简单筛选。
- `nodeAffinity`：表达更复杂的硬约束/软偏好。
- `taints/tolerations`：控制哪些工作负载允许进入某类节点。
- `PriorityClass`：定义工作负载优先级；是否实际抢占取决于策略与调度条件。
- **拓扑感知**：多卡任务需关注同节点 GPU 数、NUMA、PCIe/NVLink 关系；默认整数 GPU 资源声明不等于完整拓扑保证。

例如多卡 Tensor Parallel 往往要求一个 Pod 同时申请本节点的多张 GPU；如果跨节点扩展则涉及更复杂的网络与分布式执行机制。

## 7. DRA 是什么

Dynamic Resource Allocation（DRA）为设备属性化选择与分配提供更丰富的 API，试图超越传统的“只用一个整数资源名表示设备”模式。但其资源对象、API 版本和具体驱动支持与 Kubernetes 版本强相关。**本教学只介绍设计动机；具体 ResourceClaim 模板要在选定版本后重新编写。**

## 8. 自测问题

- GPU 物理空闲，为什么 Pod 依然 Pending？
- GPU Operator、Device Plugin、kube-scheduler 各负责什么？
- 为什么设置 ResourceQuota 不能阻止不同租户在共享 GPU 上互相影响？
- 什么情况下需要拓扑感知，而不是单纯申请 `nvidia.com/gpu: 2`？


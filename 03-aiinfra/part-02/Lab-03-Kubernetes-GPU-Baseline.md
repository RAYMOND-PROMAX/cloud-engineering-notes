# Lab 03 · 建立 Kubernetes GPU 调度实验环境

> **实验目标：**在隔离的 Kubernetes 集群中，通过 NVIDIA GPU Operator / Device Plugin 暴露 GPU 扩展资源，提交 GPU Pod，观察 `Capacity / Allocatable / Requests`、调度事件及命名空间配额。

## 1. 实验边界

本实验**以已经安装并连通的 Kubernetes 集群为起点**，完成 GPU 软件栈与资源调度的建设。Kubernetes 控制面、CNI、CRI 的从零安装依赖发行版和版本，按 [kubeadm 官方安装指南](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/) 单独完成；不把未经验证的 kubeadm、CNI、Containerd 混合脚本写成一键方案。

硬件至少有一台 NVIDIA GPU Worker；需要 Helm、`kubectl` 和有权限安装 Cluster-scoped Operator 的账号。**只在专用测试集群操作**；GPU Operator 的组件升级、Driver 重装与节点回滚可能影响整机容器。

操作前记录：

```bash
kubectl version --client
kubectl get nodes -o wide
kubectl get pods -A
kubectl get storageclass
helm version
# Linux GPU Worker：
nvidia-smi -L
```

## 2. 选择唯一的设备管理路径

**方案 A：使用 GPU Operator（本实验采用）**。由 Operator 根据配置管理 Driver、Toolkit、Device Plugin、DCGM 等组件。

**方案 B：手动 Driver + Toolkit + NVIDIA Device Plugin**。适合已有完整运维体系的集群，但不能和方案 A 对同一组件并行部署，避免重复管理、升级冲突和资源注册异常。

安装前检查 `kubectl get clusterpolicy`、NVIDIA DaemonSet 是否已经存在，不要重复安装：

```bash
kubectl get clusterpolicy 2>/dev/null || true
kubectl get ds -A | grep -i nvidia || true
```

## 3. 安装 GPU Operator（示例）

以下以 Driver **已在 GPU Worker 上安装**的环境为例；如果打算由 Operator 管驱动，需要根据受支持的平台配置 `driver.enabled=true`，不能和宿主机现有驱动管理冲突。

```bash
helm repo add nvidia https://helm.ngc.nvidia.com/nvidia
helm repo update
helm search repo nvidia/gpu-operator --versions | head -15
```

选定与 Kubernetes/OS/Driver/Containerd 兼容的 Chart 版本，记录到实验报告。以下展示一个可查证的固定参考版本 `v26.7.0`；**部署前再次核对 GPU Operator Support Matrix**。

```bash
helm install gpu-operator nvidia/gpu-operator \
  --namespace gpu-operator --create-namespace \
  --version v26.7.0 \
  --set driver.enabled=false \
  --wait --timeout 20m

kubectl -n gpu-operator get pods -o wide
kubectl get clusterpolicy
```

如宿主已手工安装 Container Toolkit，也须按文档明确谁负责 Runtime 配置，必要时设置 `toolkit.enabled=false`；安装不是“越多组件越好”。

## 4. GPU 扩展资源的注册与验证

```bash
kubectl get nodes -o wide
kubectl describe node <gpu-node-name>  # 手动替换节点名
kubectl -n gpu-operator get pods | grep -E 'device-plugin|validator|dcgm' || true
```

在 `describe node` 中核对 `nvidia.com/gpu` 的 Capacity/Allocatable。`nvidia-smi` 的低 GPU 利用率不代表能调度更多 Pod：**Kubernetes 按声明的扩展资源配额调度**。

## 5. 创建独立命名空间、配额与验证 Pod

```bash
kubectl create namespace ai-labs
cat > gpu-quota.yaml <<'YAML'
apiVersion: v1
kind: ResourceQuota
metadata:
  name: gpu-quota
  namespace: ai-labs
spec:
  hard:
    requests.nvidia.com/gpu: "1"
    limits.cpu: "8"
    limits.memory: 32Gi
    pods: "5"
YAML
kubectl apply -f gpu-quota.yaml
```

提交一个请求 1 张 GPU 的短任务：

```bash
cat > gpu-check.yaml <<'YAML'
apiVersion: v1
kind: Pod
metadata:
  name: gpu-check
  namespace: ai-labs
spec:
  restartPolicy: Never
  containers:
    - name: check
      image: nvidia/cuda:12.4.1-base-ubuntu22.04
      command: ["/bin/bash", "-lc", "nvidia-smi && sleep 15"]
      resources:
        requests:
          cpu: 100m
          memory: 256Mi
        limits:
          cpu: "1"
          memory: 1Gi
          nvidia.com/gpu: "1"
YAML
kubectl apply -f gpu-check.yaml
kubectl get pod -n ai-labs -o wide
kubectl logs -n ai-labs gpu-check
kubectl describe pod -n ai-labs gpu-check
```

如果宿主驱动不支持示例 CUDA 12.4 镜像，**先改镜像为已验证兼容的 tag**，不应因为启动失败就认定 Device Plugin 损坏。

## 6. 调度边界实验（无须制造故障）

在唯一一张 GPU 的实验节点上，`gpu-check` 运行结束后 Pod 仍占用资源账本；可以先 `kubectl delete pod ...` 再提交其他 GPU Pod。为观察资源竞争，可在第一个 GPU Pod 运行期间尝试提交第二个 Pod（需要确认命名空间 GPU quota 是否允许；否则先会被 admission 拒绝）。区分两种情况：

- **ResourceQuota 拒绝**：Pod 可能无法创建；看 API 服务器报错和 namespace quota。
- **GPU 不足导致 Pending**：Pod 创建成功但调度不上；看 `kubectl describe pod` Events 与 Node Allocatable。

两种现象不同，不能统称为“GPU 调度失败”。

## 7. 故障排查矩阵

| 现象 | 证据 | 排查方向 |
| --- | --- | --- |
| Node 没有 GPU 字段 | `describe node`、Operator/Plugin Pod | 设备插件、Driver、Toolkit |
| GPU Pod Pending | Pod Events、Node allocatable | 请求资源、亲和性、污点/容忍 |
| Pod 启动但 GPU 不可见 | `kubectl logs`、容器 Runtime | CDI / Runtime / 设备挂载 |
| Operator Pod CrashLoop | `kubectl logs -n gpu-operator` | Driver/Toolkit 冲突、镜像网络 |
| 资源超过配额 | namespace quota、API 错误 | `requests.nvidia.com/gpu` 限制 |

## 8. 验收与清理

- [ ] `nvidia.com/gpu` 出现在 GPU Node 的可分配资源。
- [ ] GPU Pod 能输出设备信息。
- [ ] 分别理解 quota admission 和 scheduler Pending。
- [ ] 保存节点、事件、Pod 日志以及 Operator Chart 版本。

```bash
kubectl delete namespace ai-labs
# GPU Operator 是集群组件，不应因为结束单个实验就默认卸载。
# 仅在该独立实验集群确实不再需要时，按官方流程移除。
```


# Lab 06 · GPU 共享、资源配额与调度边界实验

> **实验目标：**比较原生整卡设备分配与 NVIDIA Time Slicing 的调度现象，理解“GPU 资源副本数增加”与“物理显存/计算能力增加”完全不同。

## 1. 安全边界与适用范围

**本实验会修改 GPU Operator 的 Device Plugin 配置，可能触发 GPU 节点插件重启并影响 GPU Pod。只能在无其他工作负载的隔离实验集群执行。**

- 前提：Lab 03；使用 NVIDIA GPU Operator 路径，配备独占 GPU Worker；有 `clusterpolicy` 管理权限。
- 本实验只演示 **Time Slicing**。MIG 必须有支持的硬件；MPS 也有独立的配置、隔离与兼容约束，不能当作等价方案。
- 不假设所有 GPU 支持 MIG；如果使用不支持 MIG 的 GPU，**不要运行 `nvidia-smi mig` 的修改命令**。
- 本实验不用运行耗尽显存的压力脚本；可以观察多个低负载 GPU 容器被安排到同一张物理卡上。

## 2. 保存修改前基线

```bash
kubectl get nodes -o wide
kubectl get clusterpolicy -o yaml > clusterpolicy-before-lab06.yaml
kubectl describe node <dedicated-gpu-node> > gpu-node-before-lab06.txt
kubectl get pods -A -o wide | grep <dedicated-gpu-node> || true
```

将目标 GPU Worker 单独标记为实验角色：

```bash
kubectl label node <dedicated-gpu-node> gpu-lab-role=time-slice --overwrite
```

## 3. 通过 Operator 为单节点配置 Time Slicing

保存为 `time-slice-config.yaml`：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ai-lab-time-slicing
  namespace: gpu-operator
data:
  ts-2: |-
    version: v1
    flags:
      migStrategy: none
    sharing:
      timeSlicing:
        renameByDefault: true
        failRequestsGreaterThanOne: true
        resources:
          - name: nvidia.com/gpu
            replicas: 2
```

`renameByDefault: true` 时，共享资源名称会是 **`nvidia.com/gpu.shared`**，帮助实验识别共享 Pod。`replicas: 2` 表示每张物理 GPU 对外发布两个可调度份额，**不是把显存、显卡性能翻倍，也不保证显存隔离**。

```bash
kubectl apply -f time-slice-config.yaml

# 刻意不设置 default，避免给整个集群默认启用共享
kubectl patch clusterpolicies.nvidia.com/cluster-policy \
  -n gpu-operator --type merge \
  -p '{"spec":{"devicePlugin":{"config":{"name":"ai-lab-time-slicing"}}}}'

# 只标记目标独占 GPU Worker
kubectl label node <dedicated-gpu-node> \
  nvidia.com/device-plugin.config=ts-2 --overwrite

kubectl -n gpu-operator get pods -o wide
kubectl describe node <dedicated-gpu-node>
```

预期可观察到共享资源容量与节点标签变化。若原 Operator 版本对配置字段有差异，先回到本版本 NVIDIA Device Plugin / GPU Operator 文档，不要继续执行后续任务。

## 4. 提交两个独立共享 Pod

新建专用 Namespace，避免 Lab 03 的资源配额干扰：

```bash
kubectl create namespace gpu-share-labs
```

保存为 `shared-gpu-pods.yaml`：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: shared-a
  namespace: gpu-share-labs
spec:
  restartPolicy: Never
  nodeSelector:
    gpu-lab-role: time-slice
  containers:
    - name: cuda
      image: nvidia/cuda:12.4.1-base-ubuntu22.04
      command: ["/bin/bash", "-lc", "nvidia-smi -L; sleep 120"]
      resources:
        limits:
          nvidia.com/gpu.shared: "1"
---
apiVersion: v1
kind: Pod
metadata:
  name: shared-b
  namespace: gpu-share-labs
spec:
  restartPolicy: Never
  nodeSelector:
    gpu-lab-role: time-slice
  containers:
    - name: cuda
      image: nvidia/cuda:12.4.1-base-ubuntu22.04
      command: ["/bin/bash", "-lc", "nvidia-smi -L; sleep 120"]
      resources:
        limits:
          nvidia.com/gpu.shared: "1"
```

```bash
kubectl apply -f shared-gpu-pods.yaml
kubectl -n gpu-share-labs get pods -o wide
kubectl -n gpu-share-labs logs shared-a
kubectl -n gpu-share-labs logs shared-b
kubectl describe node <dedicated-gpu-node>
```

如果独占节点只有一张 GPU，两个 Pod 可能同时显示同一物理 GPU 的 UUID（实际输出要记录）；即使如此，也**不能据此证明容器获得了独立显存、性能隔离或租户安全隔离**。这个实验主要证明的是设备插件如何改变 K8s 调度资源模型。

## 5. 进一步观察优先级与配额（选做）

- 在该 Namespace 新建 `ResourceQuota`，限制 `requests.nvidia.com/gpu.shared: "2"`，然后尝试提交第三个 Pod；记录是 quota admission 拒绝还是 scheduler Pending。
- 在没有 GPU 共享的节点上请求两个整卡资源单位，观察 Pending，与 Time Slicing 对比。
- 如希望引入 Volcano/Kueue 或 Run:ai 的队列公平性、Gang Scheduling、抢占策略，应作为**后续独立实验**；普通 K8s `PriorityClass` 不等于获得完整的训练 Gang Scheduling。

## 6. 回退（重要）

1. 先停止本实验所有共享 GPU Pod：

   ```bash
   kubectl delete namespace gpu-share-labs
   ```

2. 取消目标节点的共享配置标签：

   ```bash
   kubectl label node <dedicated-gpu-node> nvidia.com/device-plugin.config-
   kubectl label node <dedicated-gpu-node> gpu-lab-role-
   ```

3. 对照 `clusterpolicy-before-lab06.yaml` 恢复 **原来的** `spec.devicePlugin.config`（不建议对已有其他变更的 ClusterPolicy 直接整对象覆盖），让 Operator 重新加载配置。
4. 确认设备插件恢复、Node Allocatable 回到基线，再删除 `gpu-operator/ai-lab-time-slicing` ConfigMap。

## 7. 观察记录与验收

| 项目 | 整卡模式 | Time Slicing 模式 |
| --- | --- | --- |
| Resource name | `nvidia.com/gpu` | `nvidia.com/gpu.shared`（本例） |
| Node 可分配资源 | 待测 | 待测 |
| 同一物理 GPU 运行的 Pod 数 | 待测 | 待测 |
| GPU UUID | 待测 | 待测 |
| 显存与性能隔离是否成立 | 不由本实验结论决定 | **不能仅由共享资源数推断** |

- [ ] 能解释 `Capacity` 与真实 GPU 数量为何不同。
- [ ] 明确 Time Slicing 不是 MIG，不提供相同隔离语义。
- [ ] 掌握 Operator 修改对 GPU Pod 的影响和恢复方法。
- [ ] 配额拒绝、Pending 和设备分配失败能分开判断。


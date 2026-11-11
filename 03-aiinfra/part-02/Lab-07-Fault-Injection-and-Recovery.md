# Lab 07 · GPU / Kubernetes / 推理服务故障注入与恢复

> **实验目标：**在隔离的实验命名空间中，分别观察容器内存 OOM、无法调度、应用启动失败与恢复。围绕 **现象 → 证据 → 判断 → 恢复 → 复测** 形成可复用的故障记录。

## 1. 安全约束与实验说明

- **只能在专用实验集群执行**，禁止对生产 GPU、共享调度器、系统命名空间或其他团队 Pod 进行故障注入。
- 本实验所有注入均为**限定到单个 Namespace / Pod / Deployment**，不修改节点驱动，不执行 `kill -9` 系统进程，不破坏 PV 或 GPU 固件。
- 每一次实验前记录正常状态和当前配置；实验结束后检查服务恢复。若现有业务不允许中断，跳过 Lab 07-C。
- Lab 07-C 需要 Lab 04 的 `ai-labs/vllm-lab`，会短暂停止该**实验**推理服务。

建立 Namespace，保存证据目录：

```bash
kubectl create namespace fault-labs --dry-run=client -o yaml | kubectl apply -f -
mkdir -p evidence-lab07
kubectl get nodes -o wide > evidence-lab07/nodes-before.txt
kubectl get events -A --sort-by=.lastTimestamp > evidence-lab07/events-before.txt
```

## 2. 实验 A：容器触发内存 OOMKilled（非 GPU OOM）

### 故障预期

一个 Pod 只有 64Mi 容器内存上限，进程尝试分配约 256Mi 的 Python 字节数组，可能被 Linux Cgroup OOM Killer 结束；其状态通常会呈现 `OOMKilled`。

保存为 `oom-lab.yaml`：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: oom-lab
  namespace: fault-labs
spec:
  restartPolicy: Never
  containers:
    - name: allocate
      image: python:3.11-slim
      command: ["python", "-c"]
      args:
        - "import time; x=bytearray(256*1024*1024); print(len(x)); time.sleep(10)"
      resources:
        requests:
          cpu: 50m
          memory: 32Mi
        limits:
          cpu: 200m
          memory: 64Mi
```

```bash
kubectl apply -f oom-lab.yaml
kubectl -n fault-labs get pod oom-lab -w
kubectl -n fault-labs describe pod oom-lab
kubectl -n fault-labs get pod oom-lab -o jsonpath='{.status.containerStatuses[0].state.terminated.reason}{"\n"}'
```

**排障要点：**`OOMKilled` 属于**容器 Cgroup 内存**问题，不能直接判断为 CUDA GPU 显存 OOM。GPU OOM 经常体现在应用错误、PyTorch/CUDA 日志中，不保证呈现 Kubernetes 的 `OOMKilled` 状态。

如果未出现预期 OOM，保留事实和实际退出原因，检查是否先触发其他系统限制，而不是随意扩大内存申请直到影响节点。

## 3. 实验 B：Pod 因资源不足而 Pending

保存为 `pending-lab.yaml`：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pending-lab
  namespace: fault-labs
spec:
  restartPolicy: Never
  containers:
    - name: sleep
      image: busybox:1.36
      command: ["sh", "-c", "sleep 300"]
      resources:
        requests:
          cpu: "100000"
          memory: 128Mi
```

```bash
kubectl apply -f pending-lab.yaml
kubectl -n fault-labs get pod pending-lab
kubectl -n fault-labs describe pod pending-lab | tail -60
kubectl -n fault-labs get events --sort-by=.lastTimestamp | tail -20
```

实验预期：Pod 能创建但**不可能满足 CPU requests**，Scheduler 事件可能出现 `Insufficient cpu`。如果 API 准入策略或配额提前拒绝，则记录为 **Admission / Quota 问题**，而不是 Pending。

尝试比较：`Insufficient cpu` 与 Lab 03 的 `Insufficient nvidia.com/gpu` 在机制上类似，都是调度账本不足，但资源类型不同。

恢复：

```bash
kubectl -n fault-labs delete -f pending-lab.yaml
```

## 4. 实验 C：刻意让 vLLM 启动参数非法，再回滚

**前提：**Lab 04 的 `ai-labs/vllm-lab` 原本可正常响应；先保存工作负载配置。

```bash
kubectl -n ai-labs get deploy vllm-lab -o yaml > evidence-lab07/vllm-before.yaml
kubectl -n ai-labs rollout history deployment/vllm-lab
kubectl -n ai-labs get deploy,pods,svc -o wide
```

在 Lab 04 的 Deployment 示例中，`args[7]` 是 `--max-model-len` 的取值（原为 `2048`）。**先检查实际 YAML**，确认索引含义一致；不一致则不要运行下面的 patch。

```bash
kubectl -n ai-labs get deployment vllm-lab \
  -o jsonpath='{.spec.template.spec.containers[0].args}'

# 仅当 args[7] 确实是 max-model-len 的值时再执行：
kubectl -n ai-labs patch deployment vllm-lab --type=json \
  -p='[{"op":"replace","path":"/spec/template/spec/containers/0/args/7","value":"invalid-integer"}]'

kubectl -n ai-labs get pods -w
```

该 Deployment 在新副本启动时可能由于参数类型错误退出，`rollout status` 不能完成。关键证据：

```bash
kubectl -n ai-labs get pods -o wide
kubectl -n ai-labs describe deployment vllm-lab
kubectl -n ai-labs logs deployment/vllm-lab --tail=100 || true
kubectl -n ai-labs get events --sort-by=.lastTimestamp | tail -30
```

**恢复优先：**

```bash
# 在确认没有其他人并发修改实验 Deployment 的前提下：
kubectl -n ai-labs rollout undo deployment/vllm-lab
kubectl -n ai-labs rollout status deployment/vllm-lab --timeout=20m
```

或者从 Lab 04 原始 `vllm-lab.yaml`（不是简单 `kubectl apply` 一整份带 status 的 `kubectl get -o yaml` 备份）重新部署，确保参数恢复。然后重新运行 Lab 04 的 `/health` 与 `/v1/chat/completions` 两个验收请求。

**恢复标准是功能恢复，不是命令执行返回 0。**

## 5. 分层诊断表

| 现象 | 第一证据 | 不应直接做出的结论 |
| --- | --- | --- |
| Pod `OOMKilled` | 容器状态 + 内存 limit | GPU 显存故障 |
| Pod `Pending` | Scheduler Event + Allocatable | 驱动坏了 |
| `ImagePullBackOff` | Pod Event + registry 日志 | 模型不兼容 |
| vLLM CrashLoop | 启动日志、参数、模型路径、GPU | K8s 调度器故障 |
| `Readiness=False` | `/health`、模型加载进度、探针阈值 | 一定要重建节点 |
| GPU 被容器看到但 CUDA 失败 | 应用 CUDA 错误 + 宿主驱动 | Device Plugin 一定没工作 |

## 6. 复盘模板与清理

```text
故障编号：LAB07-____
时间与环境：____
正常基线：____
注入动作：____
用户可见现象：____
关键 Events / Logs / Metrics：____
候选原因与排除过程：____
根因判定：____
恢复步骤：____
功能回归检查：____
剩余风险与可改进措施：____
```

```bash
kubectl delete namespace fault-labs
# 只删除实验 Namespace；保留 evidence-lab07 作为本地证据。
# ai-labs 中 vLLM 应恢复到实验前的正常配置。
```

- [ ] 至少记录两个不同层次的异常证据。
- [ ] 能区分 OOMKilled、GPU OOM、Pending、应用启动错误。
- [ ] 每一次恢复都有实际 API/Pod 检查结果。
- [ ] 没有使用未经授权的生产故障注入。

 

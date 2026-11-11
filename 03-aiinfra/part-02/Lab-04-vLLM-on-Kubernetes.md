# Lab 04 · 在 Kubernetes 部署 vLLM 推理服务

> **实验目标：**在 Lab 03 的 GPU 集群上部署一个**单副本** vLLM 服务，通过 Deployment、Service、GPU Resource Limit、共享内存和健康探针完成从 Pod 到 API 的功能闭环。

## 1. 环境与技术边界

- 已完成 Lab 03：GPU Worker 可分配至少 1 张 GPU，能创建 GPU Pod。
- 一开始只部署 **1 个副本 / 1 GPU**，避免多副本同时争抢 GPU；不在无额外 GPU 资源时要求“零停机滚动更新”。
- 使用 Lab 02 相同模型 `Qwen/Qwen2.5-0.5B-Instruct` 和已固定参考镜像 `vllm/vllm-openai:v0.26.0`；仍须核验与 GPU/Driver 的兼容性。
- 实验使用 `emptyDir` 下载缓存，Pod 重建会丢失缓存；**生产环境应设计 PVC、对象存储或预热模型方案**。不要把此实验部署当成生产级高可用模板。
- 对外只用 `ClusterIP`；通过 `kubectl port-forward` 本地访问。

## 2. 创建 Namespace

```bash
kubectl create namespace ai-labs --dry-run=client -o yaml | kubectl apply -f -
kubectl get nodes -o wide
kubectl describe node <gpu-node-name>
```

在此前 Lab 03 若设了 `requests.nvidia.com/gpu=1` 的 `ResourceQuota`，确认没有其他 GPU Pod 占用，避免部署被配额拒绝。

## 3. 编写可读的 Deployment 与 Service

将以下保存为 `vllm-lab.yaml`：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vllm-lab
  namespace: ai-labs
spec:
  replicas: 1
  # 单 GPU 节点没有多余 GPU 时，升级需先释放旧 Pod；会有短暂停机。
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: vllm-lab
  template:
    metadata:
      labels:
        app: vllm-lab
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/path: /metrics
        prometheus.io/port: "8000"
    spec:
      terminationGracePeriodSeconds: 90
      containers:
        - name: vllm
          image: vllm/vllm-openai:v0.26.0
          imagePullPolicy: IfNotPresent
          args:
            - --model
            - Qwen/Qwen2.5-0.5B-Instruct
            - --served-model-name
            - lab-llm
            - --gpu-memory-utilization
            - "0.75"
            - --max-model-len
            - "2048"
            - --max-num-seqs
            - "8"
          env:
            - name: HF_HOME
              value: /var/cache/huggingface
          ports:
            - name: http
              containerPort: 8000
          resources:
            requests:
              cpu: "2"
              memory: 4Gi
              nvidia.com/gpu: "1"
            limits:
              cpu: "4"
              memory: 16Gi
              nvidia.com/gpu: "1"
          volumeMounts:
            - name: hf-cache
              mountPath: /var/cache/huggingface
            - name: shm
              mountPath: /dev/shm
          startupProbe:
            httpGet:
              path: /health
              port: http
            periodSeconds: 10
            failureThreshold: 120
          readinessProbe:
            httpGet:
              path: /health
              port: http
            periodSeconds: 10
            failureThreshold: 3
          livenessProbe:
            httpGet:
              path: /health
              port: http
            periodSeconds: 30
            failureThreshold: 3
      volumes:
        - name: hf-cache
          emptyDir: {}
        - name: shm
          emptyDir:
            medium: Memory
            sizeLimit: 2Gi
---
apiVersion: v1
kind: Service
metadata:
  name: vllm-lab
  namespace: ai-labs
spec:
  type: ClusterIP
  selector:
    app: vllm-lab
  ports:
    - name: http
      port: 8000
      targetPort: http
```

**重要的两点：**

1. `gpu-memory-utilization=0.75` 并不等于 Kubernetes 的显存限制；Kubernetes 原生 `nvidia.com/gpu: 1` 声明的是设备资源，不是显存配额。
2. `startupProbe` 允许模型首次下载/加载时间，避免启动过程中被误重启。但过长的失败窗口可能掩盖真实失败，应结合事件和日志人工判断。

## 4. 部署与检查

```bash
kubectl apply -f vllm-lab.yaml
kubectl -n ai-labs get deploy,pods,svc -o wide
kubectl -n ai-labs rollout status deployment/vllm-lab --timeout=20m
kubectl -n ai-labs logs deployment/vllm-lab --tail=100
kubectl -n ai-labs get events --sort-by=.lastTimestamp | tail -25
```

对模型下载缓存首次为空的实验，启动时间与网络有很大关系。若等候超时，先定位下载与模型加载日志，不要盲目提高探针阈值。

## 5. API 与 Kubernetes Service 验收

在单独终端执行：

```bash
kubectl -n ai-labs port-forward svc/vllm-lab 8000:8000
```

本机另一终端：

```bash
curl -i http://127.0.0.1:8000/health
curl -sS http://127.0.0.1:8000/v1/models | python3 -m json.tool
curl -sS http://127.0.0.1:8000/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{"model":"lab-llm","messages":[{"role":"user","content":"什么是 Kubernetes GPU 扩展资源？"}],"max_tokens":64}' \
  | python3 -m json.tool
```

检查 EndpointSlice 是否出现已经 ready 的后端：

```bash
kubectl -n ai-labs get endpointslices -l kubernetes.io/service-name=vllm-lab
kubectl -n ai-labs describe deployment vllm-lab
```

`Pod Running`、`Readiness=True`、Service Endpoint 已注册是不同的验证层次，不能只看 `kubectl get pods`。

## 6. 观察、变更与回退

把 `max-num-seqs` 从 `8` 改为 `4`，作为一个可控配置变更案例；变更前记录原值、镜像、当前 Pod UID 和启动时间，再运行：

```bash
kubectl -n ai-labs rollout history deployment/vllm-lab
# 更改 vllm-lab.yaml 中的 max-num-seqs 为 4，再执行：
kubectl apply -f vllm-lab.yaml
kubectl -n ai-labs rollout status deployment/vllm-lab --timeout=20m
```

只有当你实际更改了 YAML 中的 Pod 模板字段时，重新 `apply` 才会触发新 Revision。旧副本需要先释放 GPU，因此 `Recreate` 策略有短暂停机，这正是实验要理解的容量边界。

回退：恢复 YAML 中原先数值重新 `apply`；或在明确 Revision 变化、检查过 Revision 内容后使用 `kubectl rollout undo`。不要把“回滚命令完成”误当成 API 已恢复，仍需重复接口验收。

## 7. 常见异常矩阵

| 状态 | 主要证据 | 可能原因 |
| --- | --- | --- |
| Pod Pending | Pod Events | GPU 不足 / Taint / Quota |
| ImagePullBackOff | Pod Events | 镜像拉取失败、认证或网络 |
| CrashLoopBackOff | `kubectl logs --previous` | CUDA、模型、OOM、启动错误 |
| Readiness 长期失败 | `/health`、Pod Events、日志 | 模型未加载、监听地址、探针窗口 |
| Service 不通 | EndpointSlice、port-forward | Selector、端口、NetworkPolicy |
| 变更后停机 | Deployment Strategy、GPU Allocatable | 单 GPU 无升级冗余 |

## 8. 验收与清理

- [ ] Deployment 保持期望副本数，Pod Ready。
- [ ] Service Endpoint 与 API 验证通过。
- [ ] GPU 请求数与节点资源核对一致。
- [ ] 执行一次修改与恢复，记录停机和模型加载现象。
- [ ] 明确何时需要 PVC、额外 GPU 节点、PDB、Ingress/Auth 和观测系统。

```bash
kubectl -n ai-labs delete -f vllm-lab.yaml
# 如果此 Namespace 仍有其他 Lab 资源，不删除 Namespace。
```


# Lab 02 · 单机部署 vLLM 推理服务

> **实验目标：**在 Lab 01 已验收的 GPU 容器环境中部署一台 vLLM 服务，完成模型加载、健康检查、Chat Completions、日志观察与配置变更对比。

## 1. 环境与约束

- 必需：Lab 01 通过；实验 GPU 可独占；Docker 能通过 `--gpus` 使用 NVIDIA 设备；可访问 Docker Hub 和 Hugging Face，或事先准备合规模型缓存。
- 示例模型：`Qwen/Qwen2.5-0.5B-Instruct`，作为小模型练习使用；请自行检查模型许可证、网络访问和 GPU 架构兼容。
- 示例镜像：`vllm/vllm-openai:v0.26.0`。此处是**固定参考版本，不声称已在你的机器验证**；如果下载失败或与 GPU/Driver 不兼容，先通过官方发行说明选定另一**具体 tag**，不要直接改用不可追溯的 `latest`。
- 建议：预留至少约 12–16 GiB GPU 显存进行实验；实际最低要求受镜像、架构、上下文长度、并发和内核能力影响。
- 本实验只监听回环地址，不向公网暴露推理接口。

## 2. 记录环境与下载镜像

```bash
export VLLM_IMAGE='vllm/vllm-openai:v0.26.0'
export MODEL_ID='Qwen/Qwen2.5-0.5B-Instruct'

nvidia-smi --query-gpu=name,memory.total,memory.free --format=csv
docker pull "$VLLM_IMAGE"
docker image inspect "$VLLM_IMAGE" --format '{{.Id}}'
```

记录镜像 digest、GPU 型号、Driver、输入的模型 Revision（用于正式复现可进一步固定到模型 commit）。如果使用 HF Token，不要写在可公开的 Shell 历史、YAML 或 README 中。

## 3. 启动推理容器

```bash
mkdir -p "$HOME/.cache/huggingface"

docker run -d --name ai-lab-vllm \
  --gpus '"device=0"' \
  --shm-size=4g \
  -p 127.0.0.1:8000:8000 \
  -v "$HOME/.cache/huggingface:/root/.cache/huggingface" \
  "$VLLM_IMAGE" \
  --model "$MODEL_ID" \
  --served-model-name lab-llm \
  --dtype auto \
  --gpu-memory-utilization 0.75 \
  --max-model-len 2048 \
  --max-num-seqs 8
```

说明：`--gpus '"device=0"'` 是在 Docker 中选择 GPU 0 的常见 Linux Shell 写法；若使用不同 Shell 需调整引号。`gpu-memory-utilization` 是引擎总显存预算系数，**不是固定留给 KV Cache 的比例**。

使用 `docker logs` 查看模型下载、显存预算、KV Cache、服务启动与潜在错误：

```bash
docker logs -f ai-lab-vllm
# 另开终端：
docker ps --filter name=ai-lab-vllm
```

## 4. 健康与接口验收

```bash
curl -i http://127.0.0.1:8000/health
curl -sS http://127.0.0.1:8000/v1/models | python3 -m json.tool

curl -sS http://127.0.0.1:8000/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{
    "model": "lab-llm",
    "messages": [{"role":"user","content":"用一句话解释 GPU Device Plugin 的作用。"}],
    "temperature": 0,
    "max_tokens": 48
  }' | python3 -m json.tool
```

`/health` 正常不代表模型回答一定正确；验收时分清 HTTP 服务可用、模型成功生成、输出质量三个层次。

### 验证流式输出

```bash
curl -N http://127.0.0.1:8000/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{
    "model":"lab-llm",
    "messages":[{"role":"user","content":"介绍什么是 KV Cache"}],
    "stream":true,
    "max_tokens":64
  }'
```

流式输出是否一块一个 Token，取决于协议实现与服务端分块策略，**不能把每次 TCP/HTTP chunk 直接当作一个 token**。

## 5. 观察机制和关键参数

```bash
nvidia-smi --query-gpu=utilization.gpu,memory.used,memory.total --format=csv
curl -fsS http://127.0.0.1:8000/metrics | grep -E 'vllm[:_]' | head -40

docker stats --no-stream ai-lab-vllm
```

依次尝试**每次只改一个变量**：

1. 记录基线：`max-model-len=2048`、`max-num-seqs=8`、`gpu-memory-utilization=0.75`。
2. 停止并删除实验容器（模型缓存保留）。
3. 仅把 `max-model-len` 改成 `4096`，其余不变；重新启动。
4. 对比启动是否成功、日志中的 KV Cache 容量、显存使用、响应和是否发生资源不足。
5. 恢复基线，再独立调整 `max-num-seqs`，不要同时改多个参数。

在单次请求下，不一定能观察出显著差异。真正的并发性能比较放在 Lab 05，不能从单次延迟推导集群吞吐。

## 6. 常见异常与排查顺序

| 异常 | 先检查 | 可能的下一步 |
| --- | --- | --- |
| 容器退出 | `docker logs`、`docker inspect` | Driver/镜像 CUDA/GPU 架构兼容 |
| 模型下载失败 | 网络、许可、缓存目录 | 镜像代理、Token/缓存权限 |
| `CUDA out of memory` | `nvidia-smi`、日志、已有进程 | 清理**自己的**进程、缩小上下文/显存预算 |
| `/health` 不通 | 进程启动情况、端口绑定 | 模型加载是否完成 |
| API 400 | JSON、model 名称、Token/上下文长度 | `GET /v1/models` 和具体错误体 |
| 吞吐低、TTFT 高 | 不只看 GPU Util | 进入 Lab 05 检查排队、Prefill、Decode |

## 7. 验收与清理

- [ ] 固定的镜像与模型名称已记录。
- [ ] `/health`、`/v1/models` 与 Chat API 成功。
- [ ] 抓取过一次 `/metrics` 和日志，记录真实指标名称。
- [ ] 完成至少一次**单变量**参数对照，并记录观察结果（允许“未观察到显著变化”）。
- [ ] 明确数据没有被写成虚构的性能提升数字。

```bash
docker stop ai-lab-vllm
docker rm ai-lab-vllm
# 不自动删除 Hugging Face 缓存，避免误删其他模型。
```


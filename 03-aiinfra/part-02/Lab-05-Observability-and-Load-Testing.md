# Lab 05 · 推理服务监控、压测与基线分析

> **实验目标：**从用户侧和引擎侧同时观察推理服务，绘制简单的 **并发 → TTFT / E2E / Queue / Throughput** 对照表。**不预设收益百分比、不拿 GPU 利用率代替服务质量。**

## 1. 前提与指标口径

- 已完成 Lab 02 或 Lab 04，vLLM 可通过 `http://127.0.0.1:8000` 访问。下面以**本机端口转发**为例；若运行在 K8s 中，先启动：`kubectl -n ai-labs port-forward svc/vllm-lab 8000:8000`。
- 测试流量仅打到专用实验服务，**不要对生产 API 做未经授权的并发压测**。
- 记录：GPU、Driver、vLLM tag、模型、Prompt 规模、生成长度、并发、实验轮次、冷/热启动状态。

| 层级 | 指标 | 定义与注意 |
| --- | --- | --- |
| 客户端 | Client TTFT | 发起 HTTP 请求到**首个非空生成内容**的耗时，包括网络和请求前处理 |
| 客户端 | E2E | 客户端发起请求至流结束的总耗时 |
| 客户端 | 成功率 | 正确获得响应的请求 / 发起请求 |
| 客户端 | Goodput / Tokens/s | 仅在拿到可信 token usage 时统计；不能靠 HTTP Chunk 数推断 |
| 引擎 | `num_requests_waiting` | 当时排队数量；不代表进入网关前的排队 |
| 引擎 | Queue/TTFT/ITL 直方图 | 可用于延迟分布与 P95；名字随版本变化 |
| 基础设施 | GPU Util / FB Used | GPU 设备利用率、显存；不能直接等价于请求延迟 |

## 2. 先验证真实 Metrics

```bash
curl -sS http://127.0.0.1:8000/metrics > vllm-metrics.txt
# 获取真正存在的指标名，而不是复制旧版 Grafana JSON：
grep '^# HELP vllm' vllm-metrics.txt | head -60

grep -E '^vllm:num_requests_(waiting|running)' vllm-metrics.txt | head
```

新版本通常公开 `vllm:num_requests_waiting`、`vllm:time_to_first_token_seconds`、`vllm:request_queue_time_seconds` 等，但实际服务可能因为版本、配置、请求尚未发生而缺失部分时间序列。**不应为了让查询有值而编造监控数据。**

## 3. 运行一个小型客户端并发实验

安装脚本依赖：

```bash
python3 -m venv .venv
. .venv/bin/activate
python -m pip install 'httpx>=0.27,<1'
```

保存为 `bench_stream.py`。它只测**客户端可见**的 TTFT/E2E；不声称得到引擎内部 Prefill/Decode 时间。

```python
import asyncio
import json
import os
import sys
import time
from statistics import median

import httpx

BASE = os.getenv("VLLM_URL", "http://127.0.0.1:8000")
MODEL = os.getenv("MODEL_NAME", "lab-llm")
CONCURRENCY = int(sys.argv[1]) if len(sys.argv) > 1 else 1
TOTAL = int(sys.argv[2]) if len(sys.argv) > 2 else 8

async def request(client, n, gate):
    async with gate:
        start = time.perf_counter()
        first_content = None
        ok = False
        tokens = None
        try:
            async with client.stream(
                "POST", f"{BASE}/v1/chat/completions",
                json={
                    "model": MODEL,
                    "messages": [{"role": "user", "content":
                        "用三句话解释 Kubernetes GPU 资源如何被调度。"}],
                    "max_tokens": 64,
                    "temperature": 0,
                    "stream": True,
                    "stream_options": {"include_usage": True},
                },
                timeout=180,
            ) as response:
                response.raise_for_status()
                async for line in response.aiter_lines():
                    if not line.startswith("data: "):
                        continue
                    body = line[6:].strip()
                    if body == "[DONE]":
                        ok = True
                        break
                    item = json.loads(body)
                    delta = (item.get("choices") or [{}])[0].get("delta", {})
                    if delta.get("content") and first_content is None:
                        first_content = time.perf_counter() - start
                    usage = item.get("usage")
                    if usage and usage.get("completion_tokens") is not None:
                        tokens = usage["completion_tokens"]
            return {"ok": ok, "ttft": first_content,
                    "e2e": time.perf_counter() - start, "tokens": tokens}
        except (httpx.HTTPError, ValueError, json.JSONDecodeError) as exc:
            return {"ok": False, "ttft": None,
                    "e2e": time.perf_counter() - start,
                    "tokens": None, "error": str(exc)}

async def main():
    gate = asyncio.Semaphore(CONCURRENCY)
    async with httpx.AsyncClient() as client:
        start = time.perf_counter()
        rows = await asyncio.gather(*(request(client, i, gate) for i in range(TOTAL)))
        elapsed = time.perf_counter() - start
    good = [r for r in rows if r["ok"]]
    ttft = sorted(r["ttft"] for r in good if r["ttft"] is not None)
    e2e = sorted(r["e2e"] for r in good)
    tokens = [r["tokens"] for r in good]
    def p95(a):
        return a[max(0, (len(a) * 95 + 99) // 100 - 1)] if a else None
    print(json.dumps({
        "concurrency": CONCURRENCY, "total": TOTAL,
        "success": len(good), "duration_s": round(elapsed, 3),
        "ttft_median_s": round(median(ttft), 3) if ttft else None,
        "ttft_p95_nearest_rank_s": round(p95(ttft), 3) if ttft else None,
        "e2e_p95_nearest_rank_s": round(p95(e2e), 3) if e2e else None,
        "completed_requests_per_s": round(len(good) / elapsed, 3),
        "output_tokens_per_s": round(sum(tokens) / elapsed, 3)
          if tokens and all(t is not None for t in tokens) else None,
        "errors": [r.get("error") for r in rows if not r["ok"]],
    }, indent=2, ensure_ascii=False))

asyncio.run(main())
```

注意：`p95` 用的是 nearest-rank 样本分位数，小样本 P95 不稳定，示例仅用于训练统计口径。`stream_options.include_usage` 可能随推理 API 实现变化；若没有最后的 usage，脚本会将 Tokens/s 标记为 `null`。脚本不是专业压测框架，尤其不是“每秒固定到达 RPS”的开环压测。

### 先预热再测量

```bash
python bench_stream.py 1 3  # 预热，仅作为观察
python bench_stream.py 1 12
python bench_stream.py 2 12
python bench_stream.py 4 12
```

保持相同输入、输出设置，避免把不同 Prompt/输出长度混在一起比较。若模型容量不足，减少并发；观察排队和错误是实验内容，不要靠无限提高并发“压坏”为止。

## 4. Prometheus 最小化采集（可选，但推荐）

如果实验集群**已有 Prometheus**，先检查其 targets，再添加针对 `vllm-lab.ai-labs.svc.cluster.local:8000/metrics` 的抓取任务；不要重复安装一套。

对于暂未安装的**独立实验集群**，可以把以下配置加入已有 Prometheus 的 `scrape_configs`；具体热更新方式由你的安装方法决定：

```yaml
scrape_configs:
  - job_name: vllm-lab
    scrape_interval: 15s
    metrics_path: /metrics
    static_configs:
      - targets:
          - vllm-lab.ai-labs.svc.cluster.local:8000
```

这里的 DNS 只能在能够解析 Kubernetes Service DNS 的采集环境里直接访问。**仅在 Pod annotations 中写 `prometheus.io/scrape=true` 并不保证 Prometheus 自动发现。**

确认可抓取后，在 Prometheus 查询：

```promql
# 基于引擎暴露的排队 Gauge（先核实真实指标名）
sum(vllm:num_requests_waiting)

# 服务端 TTFT P95：要先确认 bucket 存在
histogram_quantile(
  0.95,
  sum by (le) (rate(vllm:time_to_first_token_seconds_bucket[5m]))
)

# 引擎层请求排队时间 P95
histogram_quantile(
  0.95,
  sum by (le) (rate(vllm:request_queue_time_seconds_bucket[5m]))
)
```

**提示：**5 分钟窗口样本很少时，P95 没有统计意义；同时混合多个模型、引擎的分位数也可能掩盖差异，必要时按模型/实例标签分组。

若使用 DCGM Exporter，可再补充 GPU FB Used、Utilization、功耗与温度曲线；不要用 `gpu_util>70%` 自动判定服务性能正常。

## 5. 实验记录表（填写真实结果）

| 并发 | 总请求 | 成功数 | Client TTFT P95 | Client E2E P95 | 请求完成数/s | 引擎 Waiting / Queue | GPU FB Used |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | 12 | 待测 | 待测 | 待测 | 待测 | 待测 | 待测 |
| 2 | 12 | 待测 | 待测 | 待测 | 待测 | 待测 | 待测 |
| 4 | 12 | 待测 | 待测 | 待测 | 待测 | 待测 | 待测 |

## 6. 分析步骤与验收

1. 对齐客户端、引擎与 GPU 的时间窗口。
2. 如果 TTFT 上升，同时引擎 Waiting 和 Queue Time 上升，先怀疑排队；若 Waiting 没升，检查 Tokenization/Prefill/网关等其它阶段。
3. 如果 E2E 升高但 TTFT 变化不大，优先检查输出 token 数、Decode/ITL，而不是直接扩 GPU。
4. 判断错误率、P95 与吞吐是否出现拐点；**实际容量必须在某个事先定义的 SLO 下衡量，不能只有“最大 QPS”。**

- [ ] 有完整的测试环境版本信息。
- [ ] 有至少 3 档并发的输出与服务日志。
- [ ] 区分客户端 TTFT、引擎 TTFT 与排队时间。
- [ ] 标记了未采到的指标，而不是用零补齐。
- [ ] 能基于证据提出下一步实验，而不是仅凭显卡利用率下结论。

- 

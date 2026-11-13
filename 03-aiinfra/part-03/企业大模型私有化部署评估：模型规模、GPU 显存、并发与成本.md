# 企业大模型私有化部署评估：模型规模、GPU 显存、并发与成本



## 一、问题背景

做私有化大模型项目，最常见的翻车方式不是模型效果不好，而是资源规划做错了。典型场景有这么几种：

第一种，只算了权重显存。7B 模型 FP16 权重 14GB，有人直接买个 24GB 的卡就上，跑起来确实能加载，然后发现并发一上来就 OOM，或者上下文一开到 8K 就崩。原因很简单，权重要 14GB，KV Cache 要几 GB 到几十 GB，框架本身还要占 1-2GB，24GB 卡根本装不下。

第二种，只看参数量不看落盘体积。同样是 7B，BF16 落盘约 15GB，INT4 量化后约 4-5GB。有人拿 7B 的显存预算去规划 72B 的项目，或者反过来把量化后的体积当成原始体积，采购单下错了，硬件到了才发现装不下。

第三种，把"能跑"当成"能扛"。单并发测试延迟 30ms/token，看着很好。上线之后 50 个用户同时用，TTFT 从 300ms 涨到 8 秒，用户直接投诉。这是因为单序列延迟和批量吞吐是两个完全不同的指标，压测方式不对就会得出错误结论。

第四种，只算 GPU 钱不算周边。8 卡整机采购 100 万，然后发现机柜供电不够、空调制冷不够、UPS 容量不够、机房空间不够，改造费用又是一大笔。或者忽略了折旧周期，三年折旧和五年折旧算出来的单价差 40%。

第五种，选错了精度。金融、医疗场景对数值敏感，直接上 INT4 量化，结果模型输出质量下降明显，业务方不认。反过来，一个内部知识问答系统用 FP16 全精度跑 72B，成本高得离谱，其实 INT8 完全够用。

这篇文章不是讲大模型原理的，是把显存、并发、成本这三笔账算清楚。算完之后你能回答几个具体问题：

- 我要跑 Qwen2.5-32B，32K 上下文，支持 20 并发，需要几张什么卡？
- 我预算 80 万，能撑起多大规模、多少并发、什么延迟水平？
- 我现有 2 台 8 卡 A800 机器，能上哪些模型、能扛多少 QPS？
- 私有化和调用公有云 API 相比，盈亏平衡点在多少 token/月？

这几个问题的答案，都可以用下面这套方法算出来，而且算出来的数字和实际部署的偏差能控制在合理范围内。

## 二、适用场景

### 2.1 必须先做评估的场景

下面这些情况，不做评估直接采购，基本都会出问题：

- 一次性采购金额超过 20 万，硬件选错了没法退换或者退换成本很高
- 模型规模超过 14B，单卡放不下，需要做张量并行切分，拓扑错了性能损失一半
- 业务要求上下文长度超过 8K，KV Cache 会变成显存的主要消耗而不是次要消耗
- 预期并发超过 10，需要区分单序列延迟和批量吞吐两套指标
- 有明确 SLA，比如首 token 延迟 < 1s、输出速度 > 20 token/s
- 需要过等保、行业合规、数据不出域，意味着不能用公有云 API 兜底
- 预算有限但业务方期望很高，需要用数据说服业务方调整期望

### 2.2 可以简化评估的场景

- 纯 PoC 验证，只有 1-2 个人用，不考虑并发，24GB 消费卡 + INT4 量化足够
- 只是内部测试模型效果，不接业务流量，跑起来看一眼输出就关掉
- 已有的 GPU 资源闲置，只是想把闲置算力用起来，不涉及新增采购

### 2.3 评估需要收集的输入

评估开始之前，这些信息必须问清楚，缺一个都会导致结论偏差：

| 输入项 | 具体内容 | 为什么必须问 |
| --- | --- | --- |
| 业务场景 | 问答 / RAG / 代码补全 / 文档摘要 / Agent 工具调用 | 决定输入输出长度比例。RAG 输入长输出短，代码补全输入输出都短但请求密集 |
| 输入长度 | 平均 prompt token 数、P95 prompt token 数 | 决定 prefill 开销和 KV Cache 占用 |
| 输出长度 | 平均 completion token 数、P95 | 决定 decode 总时长和 KV Cache 峰值 |
| 并发 | 同时在线的请求数、峰值 QPS | 决定批大小，直接决定 KV Cache 总量 |
| 延迟要求 | TTFT 上限、TPOT 上限、端到端上限 | 决定批大小上限，延迟和吞吐是矛盾的 |
| 质量要求 | 是否允许量化、允许到哪一档 | 决定显存和成本，是最省钱的杠杆 |
| 可用性 | 是否需要双活、冗余、是否允许单点 | 决定 GPU 数量要乘冗余系数 |
| 数据合规 | 数据能不能出域、是否需要审计日志 | 决定能不能混合公有云 |
| 预算 | 一次性采购预算、年度运维预算 | 决定选什么档次的卡 |
| 机房条件 | 机柜剩余空间、单机柜供电上限、制冷能力 | 决定能不能上 8 卡整机 |
| 现有资源 | 已有 GPU 型号数量、是否空闲 | 决定是复用还是新增 |

这里面最关键的是"延迟要求"和"并发"这两项。很多项目方给不出准确的数字，只会说"要快"“要支持很多人”。这种情况下的处理方式：先按经验值给一个基线假设（比如 TTFT < 2s、TPOT < 50ms、并发 20），并明确标注这是假设值，等压测数据出来之后再反向修正，把假设和实测的差距写进评估报告，让业务方签字确认。

## 三、核心知识点

### 3.1 模型规模到底指什么

说"7B 模型"，7B 指的是参数总量约 70 亿。但做显存计算时，光看这个数字不够，需要从模型的 `config.json` 里读出来下面这些字段：

- `num_hidden_layers`：层数。Llama-3.1-8B 是 32 层，Qwen2.5-72B 是 80 层
- `hidden_size`：隐藏层维度。Llama-3.1-8B 是 4096，Qwen2.5-72B 是 8192
- `num_attention_heads`：注意力头数
- `num_key_value_heads`：KV 头数。这个字段只在 GQA（Grouped Query Attention）模型里出现，等于 `num_attention_heads` 时就是标准 MHA
- `head_dim`：单头维度。很多模型 `head_dim = hidden_size / num_attention_heads`，但也有例外，比如 Qwen2.5 系列显式配置 `head_dim: 128`
- `vocab_size`：词表大小，影响 embedding 层参数
- `max_position_embeddings`：训练时的最大位置长度，决定了理论上能支持多长上下文
- `torch_dtype`：权重精度

其中 `num_key_value_heads` 决定了 KV Cache 的大小，这是显存测算里最容易算错的地方。

从 MHA 到 GQA 的演进，直接动机就是省 KV Cache：

- MHA（Multi-Head Attention）：每个 KV 头对应一个 Q 头，`num_key_value_heads = num_attention_heads`。KV Cache 最大，质量最好
- MQA（Multi-Query Attention）：所有 Q 头共享一组 KV，`num_key_value_heads = 1`。KV Cache 最小，但质量下降明显
- GQA（Grouped Query Attention）：折中方案，把 Q 头分组，每组共享一组 KV。Llama-3.1-8B 是 32 个 Q 头、8 个 KV 头，相当于是 4 组

以 Llama-2-7B（MHA）和 Llama-3.1-8B（GQA）对比：前者 32 层 32 个 KV 头 head\_dim 128，后者 32 层 8 个 KV 头 head\_dim 128。同样 4096 上下文，前者的 KV Cache 是后者的 4 倍。MHA 在 4096 上下文下单序列就要 2GB，GQA 只要 512MB。这就是为什么现在新模型基本都是 GQA。

还有一类特殊结构：DeepSeek-V2/V3 用的 MLA（Multi-head Latent Attention），把 KV 压缩到一个低秩潜向量再缓存，KV Cache 比 GQA 还要小一个数量级。这种模型不能套用下面的公式，必须看官方 `config.json` 里的 `kv_lora_rank` 和 `qk_rope_head_dim` 字段。下文的所有计算都以标准 GQA/MHA 模型为准，MLA 模型单独说明。

### 3.2 精度与量化

权重显存 = 参数量 × 每参数字节数。不同精度的字节数：

| 精度 | 每参数字节 | 7B 权重 | 32B 权重 | 72B 权重 | 说明 |
| --- | --- | --- | --- | --- | --- |
| FP32 | 4 | 28 GB | 128 GB | 288 GB | 推理几乎不用，训练场景才涉及 |
| FP16 / BF16 | 2 | 14 GB | 64 GB | 144 GB | 推理默认精度，效果基准 |
| FP8 | 1 | 7 GB | 32 GB | 72 GB | 需要 H100/H800 或 Ada 架构（4090/L40S/L20）支持，质量损失很小 |
| INT8 | 1 | 7 GB | 32 GB | 72 GB | 通用支持，质量损失小 |
| INT4（AWQ/GPTQ） | 0.5 | 3.5 GB | 16 GB | 36 GB | 质量损失可感知，简单问答任务通常可接受 |

几点必须注意：

**第一，量化不是只有权重。** AWQ、GPTQ 这类权重量化方案，激活值仍然是 FP16 计算的，所以 KV Cache 仍然是 FP16，激活开销也不变。量化只省了权重那部分。有人以为 INT4 之后显存全减到 1/4，这是错的。

**第二，实际显存比理论值大。** 上面算的是理论值。实际部署时：

- Embedding 层和 lm\_head 层有时会被量化方案特殊处理，可能保持 FP16，多占 1-2GB
- 量化元数据（scales、zeros）会额外占 0.1-0.5GB
- 如果开了 LoRA，还要加上 LoRA 权重

所以规划时，算出来的权重值建议再乘 1.1-1.15 的安全系数。

**第三，量化是效果和成本的权衡，必须让业务方验收。** 不要技术团队自己决定上 INT4 就上线了。正确做法是：准备 50-100 条真实业务问题，同一模型跑 FP16 和 INT4 两版，把输出并排给业务方看，让业务方判断能不能接受。这个过程要留书面记录，避免上线之后效果不好扯皮。

**第四，不同量化方案的选择。** AWQ 通常比 GPTQ 在同等比特下质量略好，而且推理速度更快；GPTQ 生态更成熟、模型覆盖面更广。INT8 基本没有质量争议，能上 INT8 就优先 INT8，显存实在不够再考虑 INT4。

### 3.3 显存组成

推理时 GPU 上真正占用的显存，包含下面几块：

```
总显存 = 权重 + KV Cache + 激活值 + 框架开销 + CUDA 上下文 + 通信缓冲区
```

逐项说明：

**权重**：模型参数本身，计算方式见 3.2 节。这部分是固定的，模型加载完就占住。

**KV Cache**：最容易被低估的部分，也是本文重点。计算方式见 3.4 节。它和上下文长度、并发数成正比。vLLM 这类引擎会预先分配一块 KV Cache 池，池子大小由 `--gpu-memory-utilization` 和 `--max-model-len` 共同决定。

**激活值**：中间计算结果。Prefill 阶段由于要处理整段 prompt，激活值会比较大，和批大小、序列长度强相关。Decode 阶段激活值很小。经验值：prefill 时激活可占 1-4GB（视批大小和序列长度），decode 时几百 MB。开启 FlashAttention 之后激活开销会明显下降。

**框架开销**：Python 进程、PyTorch、CUDA 库、模型权重加载缓冲等。经验值 1-2GB。加载模型时会有短暂的显存峰值（权重从 CPU 拷到 GPU 的过程中，可能同时存在 CPU 和 GPU 两份），如果显存卡得很紧，加载阶段反而比运行时更容易 OOM。

**CUDA 上下文**：每个进程初始化 CUDA 大约占 300-600MB。如果一张卡上跑多个进程（比如两个推理服务实例共享一张卡），每个进程都要单独吃这份开销。

**通信缓冲区**：张量并行时 NCCL 需要通信缓冲。TP=2 时每卡约 0.5-1GB，TP 越大缓冲区越大。这部分和 NVLink 是否可用有关，走 PCIe 的通信需要更大的缓冲。

**显存碎片**：运行一段时间后会产生碎片，vLLM 用 PagedAttention 管理 KV Cache 很大程度上缓解了这个问题，但权重和激活部分仍然可能有碎片。计划时留 5-10% 余量。

### 3.4 KV Cache 计算

这是本文最核心的公式。对于标准 MHA/GQA 模型，单个序列的 KV Cache 字节数：

```
KV_bytes_per_seq = 2 × num_layers × num_kv_heads × head_dim × seq_len × bytes_per_element
```

公式里每一项的含义：

- `2`：K 和 V 两份，这是固定系数
- `num_layers`：Transformer 层数
- `num_kv_heads`：KV 头数（不是 Q 头数）
- `head_dim`：单头维度
- `seq_len`：序列长度，包含输入 prompt 和已生成的输出
- `bytes_per_element`：KV 的精度字节数。FP16/BF16 是 2，FP8 是 1

注意一个容易混淆的点：`num_kv_heads × head_dim` 对 GQA 模型来说不等于 `hidden_size`。比如 Llama-3.1-8B，`hidden_size = 4096`，但 `num_kv_heads × head_dim = 8 × 128 = 1024`，只有 hidden\_size 的四分之一。如果误用 hidden\_size 去算，结果会大 4 倍。

批量场景：

```
KV_total = KV_bytes_per_seq × batch_size
```

下面用真实模型参数算一遍，FP16 精度，`bytes_per_element = 2`。

| 模型 | 层数 | KV 头数 | head\_dim | 每 token 每序列 KV | 2048 上下文 | 8192 上下文 | 32768 上下文 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Llama-3.1-8B | 32 | 8 | 128 | 128 KiB | 256 MiB | 1 GiB | 4 GiB |
| Qwen2.5-7B | 28 | 4 | 128 | 56 KiB | 112 MiB | 448 MiB | 1.75 GiB |
| Qwen2.5-14B | 48 | 8 | 128 | 192 KiB | 384 MiB | 1.5 GiB | 6 GiB |
| Qwen2.5-32B | 64 | 8 | 128 | 256 KiB | 512 MiB | 2 GiB | 8 GiB |
| Qwen2.5-72B | 80 | 8 | 128 | 320 KiB | 640 MiB | 2.5 GiB | 10 GiB |
| Llama-3.1-70B | 80 | 8 | 128 | 320 KiB | 640 MiB | 2.5 GiB | 10 GiB |

验算一下 Qwen2.5-7B：`2 × 28 × 4 × 128 × 2 = 57344 字节 = 56 KiB`。8192 上下文：`56 KiB × 8192 = 458752 KiB = 448 MiB`。对得上。

看这张表要有一个直觉：**32K 上下文下，72B 模型单序列 KV Cache 就要 10GB**。如果并发 20，就是 200GB——比权重（144GB）还多。这就是为什么长上下文场景下 KV Cache 会变成主要矛盾，也是为什么很多团队"能启动但不能长上下文"。

批量显存需求（并发 × 上下文）：

| 模型 | 并发 8 / 8K | 并发 20 / 8K | 并发 20 / 32K | 并发 50 / 4K |
| --- | --- | --- | --- | --- |
| Qwen2.5-7B | 3.5 GiB | 8.75 GiB | 35 GiB | 11 GiB |
| Qwen2.5-14B | 12 GiB | 30 GiB | 120 GiB | 37.5 GiB |
| Qwen2.5-32B | 16 GiB | 40 GiB | 160 GiB | 50 GiB |
| Qwen2.5-72B | 20 GiB | 50 GiB | 200 GiB | 62.5 GiB |

结论很直接：**32B 以上模型想跑 32K 上下文 + 20 并发，单机 8 卡 80GB 都已经很紧张**。这种情况必须考虑几种手段：降低并发、缩短上下文、量化 KV Cache、或者用分页 + 前缀缓存提高复用率。

关于 MLA 模型（DeepSeek 系），KV Cache 不能套上面的公式。MLA 缓存的是压缩后的潜向量（`c_kv`，维度 `kv_lora_rank`，通常 512）加上一个解耦的 RoPE 部分（`qk_rope_head_dim`，通常 64）。每 token 每序列的 KV Cache 约等于 `num_layers × (kv_lora_rank + qk_rope_head_dim) × bytes`。以 61 层、512+64 为例：`61 × 576 × 2 = 70272 字节 ≈ 68.6 KiB`。看起来和 GQA 差不多，但 MLA 的优势在于它是在 671B 参数的模型上做到的——如果 DeepSeek-V3 用标准 GQA（比如 128 个 KV 头），KV Cache 会是这个数字的十倍以上。所以 MLA 的价值是"在超大模型上保持小 KV Cache"。实际算的时候一定以官方 `config.json` 为准，不要照搬。

### 3.5 并发模型与批处理

搞清楚了显存，还要搞清楚并发是怎么实现的。这直接决定了你算出来的显存能不能兑现成实际吞吐。

**静态批处理（Static Batching）**：把一批请求凑齐，一起跑完再返回。问题很明显：一个请求输出 10 个 token，另一个输出 500 个 token，短的那个要等长的那个跑完，GPU 空转。这种方式吞吐低、尾延迟高，现在的推理引擎基本不用了。

**连续批处理（Continuous Batching）**：也叫 iteration-level scheduling。每生成一个 token 就重新调度一次，某个序列生成完了立刻腾出位置给新请求。这是 vLLM、TGI、TensorRT-LLM 现在的默认方式，能显著提高 GPU 利用率。

连续批处理下，并发请求数和实际批大小不是一回事。假设 100 个并发请求，由于序列长度不同，实际每个 step 参与的序列数可能是 30-60 个。所以压测时不能只看"连接数"，要看引擎暴露的实际运行序列数指标（vLLM 里是 `vllm:num_requests_running`）。

**PagedAttention**：vLLM 的核心优化。传统实现里 KV Cache 要按最大长度预分配连续显存，浪费严重（实际用了 1000 token 也要占 8192 token 的位置）。PagedAttention 把 KV Cache 切成固定大小的 block，按需分配，显存利用率从 20-40% 提高到 90% 以上。而且 block 可以共享，同一段 prompt 的多个请求能复用同一份 KV。

**前缀缓存（Prefix Caching）**：如果很多请求有相同的前缀（比如同一个 system prompt，或者同一个文档做 RAG），开启前缀缓存后这部分 KV 只算一次。vLLM 里通过 `--enable-prefix-caching` 开启（新版本可能默认开启，以实际版本文档为准）。在 RAG 场景里这个优化收益很大，能把 prefill 开销降一半以上。

**分块预填充（Chunked Prefill）**：把长 prompt 的 prefill 拆成几块，和 decode 交错执行。好处是长 prompt 不会阻塞已经在跑的短请求，能明显改善尾延迟。vLLM 里通过 `--enable-chunked-prefill` 开启。

这几个机制的组合效果，直接决定了"算出来的显存能支撑多少有效并发"。这也是为什么显存测算只是第一步，必须用压测验证。

### 3.6 关键性能指标

评估大模型服务，必须区分下面这几组指标。混在一起谈就是外行。

**TTFT（Time To First Token，首 token 延迟）**：从请求发出到收到第一个 token 的时间。主要由 prefill 阶段决定。长 prompt 的 TTFT 会明显变长，因为 prefill 要对整个输入做一次前向。用户体验上，TTFT 决定了"点了发送之后多久看到字开始出来"。

**TPOT（Time Per Output Token）或 ITL（Inter-Token Latency）**：生成阶段每个 token 的平均耗时。主要由 decode 阶段和显存带宽决定。注意 TPOT 通常定义为首 token 之后的时间除以（输出 token 数 - 1），不同工具定义可能略有差异。

**端到端延迟（E2E Latency）**：`TTFT + TPOT × (输出 token 数 - 1)`。用户实际感知的总时长。

**吞吐（Throughput）**：单位时间处理的 token 数或请求数。分两种：

- 输出吞吐：生成 token/s，衡量 decode 效率
- 总吞吐：输入 token + 输出 token / 秒，衡量整体处理能力

**QPS / RPS**：每秒请求数，和请求长度强相关，脱离长度谈 QPS 没有意义。

**并发数（Concurrency）**：同时处于处理中的请求数。

这几个指标是互相矛盾的，核心矛盾是**延迟和吞吐**：

- 批大小越大，GPU 利用率越高，总吞吐越高
- 但批大小越大，每个请求分到的算力越少，单请求延迟越高

这是根本性的权衡，没有免费的午餐。所以评估时必须明确：**业务要的是低延迟（比如对话场景，TTFT < 1s）还是高吞吐（比如批量文档摘要，不在乎单个请求多久）**。

典型参考区间（具体数值随模型、硬件、长度变化很大，仅供量级参考，必须以自己环境的压测为准）：

| 场景 | TTFT | TPOT | 端到端 |
| --- | --- | --- | --- |
| 实时对话，体验优秀 | < 500 ms | < 30 ms | — |
| 实时对话，可接受 | < 1.5 s | < 60 ms | — |
| 内部知识问答，可接受 | < 3 s | < 100 ms | — |
| 离线批处理 | 不敏感 | 不敏感 | 单批 < 30 min |

**单序列解码速度的理论上限**，由显存带宽决定：

```
单序列 decode 速度上限 ≈ 显存带宽 / 权重字节数
```

因为 decode 阶段每个 token 都要把全部权重从显存读一遍。以 A100 80GB（HBM2e 带宽约 2039 GB/s，实际可达 1600 GB/s 左右）跑 7B FP16（14GB）：

```
1600 / 14 ≈ 114 token/s
```

这是理论上限。实际由于 kernel 效率、kernel launch 开销等因素，单序列能跑到 60-90 token/s 就已经不错了。这个公式很有用，它能让你快速判断一个配置合不合理——如果有人告诉你 4090 跑 72B FP16 能到 50 token/s，直接可以用这个公式否定：72B FP16 是 144GB，24GB 显存根本装不下，更不用谈速度。

### 3.7 GPU 型号与互联

选卡不能只看显存，还要看带宽、互联能力、精度支持。

| 型号 | 显存 | 显存带宽 | 互联 | FP8 支持 | 典型定位 |
| --- | --- | --- | --- | --- | --- |
| A100 80GB SXM | 80 GB HBM2e | 约 2039 GB/s | NVLink 600 GB/s | 否 | 数据中心推理/训练，仍是主力 |
| A100 40GB PCIe | 40 GB HBM2e | 约 1555 GB/s | NVLink 桥 | 否 | 存量设备 |
| A800 80GB | 80 GB HBM2e | 约 2039 GB/s | NVLink 400 GB/s | 否 | 国内合规版本 |
| H800 / H20 系列 | 80-96 GB | 3.35 TB/s 量级 | 高速互联 | 是 | 新采购主力，规格以厂商文档为准 |
| L20 48GB | 48 GB GDDR6 | 约 864 GB/s | 无 NVLink | 是 | 性价比推理卡 |
| L40S 48GB | 48 GB GDDR6 | 约 864 GB/s | 无 NVLink | 是 | 推理 + 图形混合 |
| A40 48GB | 48 GB GDDR6 | 约 696 GB/s | 无 NVLink | 否 | 存量推理卡 |
| RTX 4090 24GB | 24 GB GDDR6X | 约 1008 GB/s | 无 NVLink | 是 | PoC、小模型单机 |
| RTX 3090 24GB | 24 GB GDDR6X | 约 936 GB/s | NVLink 桥（部分型号） | 否 | 存量消费卡 |
| T4 16GB | 16 GB GDDR6 | 约 320 GB/s | 无 | 否 | 极轻量、边缘 |
| V100 32GB | 32 GB HBM2 | 约 900 GB/s | NVLink 300 GB/s | 否 | 存量老卡 |

上表规格为量级参考，采购时以厂商最新规格书为准，不同批次、不同封装（SXM/PCIe/NVL）差异明显。

几个选型判断：

**第一，能不能多卡并行取决于互联。** 72B FP16 是 144GB，单卡装不下，必须切分。切分后每层都要做 all-reduce 通信，通信走 NVLink 还是走 PCIe 差一个数量级。PCIe 4.0 x16 双向带宽约 64 GB/s，NVLink 是 400-600 GB/s。走 PCIe 做 TP=4 跑 72B，延迟会明显变差，甚至可能因为通信成为瓶颈，加卡反而变慢。

**第二，消费卡能不能上生产。** RTX 4090 单卡 24GB、带宽 1008 GB/s，跑 7B-14B 单卡推理性价比很高。但有几个坑：

- 无 NVLink，多卡并行只能走 PCIe，TP > 2 性能损失大
- 被动散热，普通服务器装 4 张以上散热跟不上，需要专门改装的机箱或风扇墙
- 驱动和 CUDA 生态对企业级特性（如 MIG、vGPU）支持有限
- 部分型号有出口限制或保修限制，采购合规要考虑

结论：4090 适合 PoC、小规模内部服务、单卡小模型；不适合要求高可用、需要多卡并行的生产核心系统。

**第三，FP8 值不值得。** 支持 FP8 的卡（H 系列、Ada 架构）能把权重压到 FP16 的一半，且质量损失很小，吞吐还能提升。如果新采购且预算允许，优先选支持 FP8 的卡，相当于免费拿到一倍显存效率。

**第四，MIG 的取舍。** A100/H100 支持 MIG（Multi-Instance GPU），能把一张卡切成多个实例，各自独立。好处是隔离性好，适合多租户。坏处是切分后单个实例的显存和带宽都被限制，跑大模型通常不合适，更适合小模型多实例场景。

### 3.8 成本构成

私有化大模型的成本不只是买卡的钱。完整的成本清单：

**一次性投入（CAPEX）**：

- GPU 服务器整机：8 卡 A800 整机约在百万量级，具体以当期采购价为准
- 网络设备：多机部署需要高速交换机（IB 或 RoCE），单台 10-30 万
- 存储：模型仓库、日志、知识库向量数据，全闪存阵列或分布式存储
- 机柜与配套：机柜、PDU、线缆
- 机房改造：供电扩容、制冷改造、消防，这部分经常被漏算

**持续性投入（OPEX）**：

- 电费：单张 400W 卡，8 卡整机加 CPU/主板/风扇约 5-6 kW，PUE 1.4 则实际耗电约 7-8.4 kW。按 0.8 元/度、24 小时运行算：`7.5 kW × 24 h × 365 天 × 0.8 元 ≈ 5.3 万元/年`
- 制冷费用（如果单独计费）
- 带宽费用
- 运维人力：至少 0.5-1 人
- 软件许可：部分推理引擎、监控平台有商业版本
- 备件与保修：过保后的维修成本
- 折旧：一般按 3-5 年折

**隐性成本（最容易被忽略）**：

- 模型迭代带来的重复调优成本（新模型出来要重新适配、重新压测）
- 效果不达标导致的返工（量化选错了要重跑，上下文不够要加卡）
- 空闲浪费：为了应对峰值买的卡，平时利用率可能只有 20-30%
- 学习成本：团队要熟悉推理引擎、显存调优、压测方法

**成本核算的正确方式**：算单位 token 成本，而不是算总投入。

```
单位成本(元/千 token) = 年度总成本 / 年度处理 token 数 × 1000

年度总成本 = 硬件折旧 + 电费 + 网络 + 人力 + 软件许可 + 运维
```

举例：一台 8 卡 A800 整机，假设采购 120 万，3 年折旧，则年折旧 40 万；电费 5.3 万/年；人力分摊 15 万/年；其他（网络、软件、备件）10 万/年。年度总成本约 70 万。

这台机器跑 Qwen2.5-32B INT8，压测实测输出吞吐 2500 token/s（这是批量场景的合理量级，实际值必须压测）。假设日均有效运行 12 小时（另一半时间低负载或空闲），年处理输出 token：

```
2500 × 3600 × 12 × 365 = 3.942e10 token = 394 亿 token
```

换算成千 token：`3.942e10 / 1000 = 3.942e7` 千token。

单位成本：

```
70 万元 / 3.942e7 千token = 0.0178 元/千 token
```

**这里最容易犯的错误是把"亿"和"千"的换算搞混**，算出来差 10 倍。记住：1 亿 token = 1e8 token = 1e5 千token，所以 394 亿 token = 3.94e7 千token。

这个数字和公有云 API 报价对比，就能算出盈亏平衡点。注意要多算一个系数：实际业务很难保持满负载，如果平均利用率只有 30%，成本就是上面的 3.3 倍（约 0.059 元/千 token）。

**关于空闲资源的处理**：私有化部署最大的成本陷阱是"为峰值买单，为均值付钱"。8 卡机器白天跑业务，晚上闲置。可以做的事：把离线批处理任务（文档摘要、数据标注预生成）放到夜间跑，提升整体利用率。这能显著改善单位成本。评估报告里应该给出"按 100% 利用率"和"按实际预估利用率"两个成本数字，后者才是真实成本。

### 3.9 显存、并发、成本三者的关系

把上面的内容串起来，形成一张关系图：

```
业务需求（上下文长度、并发数、延迟要求）
        │
        ▼
模型选择（参数量、结构 GQA/MHA/MLA）
        │
        ▼
精度选择（FP16 / FP8 / INT8 / INT4）
        │
        ├──▶ 权重显存 = 参数量 × 字节数
        │
        ▼
KV Cache 需求 = 2 × 层数 × KV头数 × head_dim × 上下文 × 并发 × 字节数
        │
        ▼
单卡总需求 = 权重 + KV Cache + 激活 + 框架开销 + 余量
        │
        ▼
GPU 数量 = ceil(单卡需求 / 单卡可用显存)
        │
        ▼
成本 = GPU 数量 × 单价 + 配套 + 运维
```

关键洞察有三条：

**第一条，KV Cache 是弹性的，权重是刚性的。** 权重不随业务变化，KV Cache 随上下文和并发线性增长。所以优化显存的第一优先级是控制上下文长度和并发上限，不是换更大的卡。

**第二条，量化对权重有效，对 KV Cache 部分有效。** 权重量化省的是刚性部分；KV Cache 量化（FP8 KV Cache）是另一个独立开关，很多引擎支持，能省一半 KV Cache，但会带来一定的质量损失，要单独评估。

**第三条，成本对利用率极度敏感。** 同样一台机器，利用率 30% 和 80% 的单位成本差了 2.7 倍。所以评估报告里必须包含利用率假设，并且给出提升利用率的方案。

## 四、整体评估思路

整个评估分六步，每一步都有明确的产出物。不要跳步，跳步的结果就是算出来的数字自己都不信。

**第一步：需求盘点。** 把 2.3 节表格里的输入项全部收集齐。产出物是《业务需求确认表》，需要业务方签字。

**第二步：模型选型。** 根据需求确定候选模型（2-3 个），确定精度方案（1-2 个）。产出物是《模型选型对比表》，包含参数、结构、显存、质量评估结论。

**第三步：显存测算。** 按 3.2-3.4 节的公式，算出每种组合下的单卡显存需求，反推 GPU 数量和拓扑。产出物是《显存测算表》，必须列出每个数字的计算过程，不要只给结果。

**第四步：部署与压测。** 在小规模环境（哪怕单卡）上实际部署，跑压测，拿到真实的 TTFT、TPOT、吞吐数据。**这一步是整个评估里最重要的一步**，因为前三步都是理论值，压测才能给出真实数字。产出物是《压测报告》，包含不同并发梯度下的指标曲线。

**第五步：成本核算。** 按 3.8 节的方法算 CAPEX 和 OPEX，算单位 token 成本，和公有云 API 对比算盈亏平衡点。产出物是《成本核算表》。

**第六步：出评估报告。** 汇总前五步，给出明确结论：推荐配置、预期性能、预期成本、风险点、回滚和降级方案。产出物是《私有化部署评估报告》。

一个实务建议：**第三步和第四步在时间上重叠**。不要等理论测算完全做完再去压测，可以先用一台机器或者一张卡把模型跑起来，边压测边修正理论模型。这样能更早发现需求里的问题（比如业务方说的"20 并发"实际含义是"20 个用户，每人每秒可能发多个请求"）。

## 五、实战步骤

### Step 1：盘点业务需求

**目的**：把模糊的业务期望转化成可计算的参数。

**操作**：和业务方一起过一遍下面的问题清单。不要发邮件让业务方自己填，要当面过，因为业务方对 token 这个概念通常没有直觉。

问题清单：

```
1. 主要使用场景是什么？（问答 / 摘要 / 代码 / 检索增强 / 工具调用）
2. 一次请求的输入大概多少字？（给例子：一段 500 字的文档约 700-800 token）
3. 一次请求的输出大概多少字？
4. 有多少人会用？同时在线多少人？峰值多少人在同一分钟内提问？
5. 用户能接受的等待时间是多少？（首字延迟 / 完整输出延迟）
6. 输出质量要求多高？能不能接受量化模型的输出？
7. 有没有数据不能出公司的要求？
8. 预算是多少？什么时候要用？
9. 机房还有空间和电吗？
```

**关键判断**：

- 如果业务方说不出并发数，用"人数 × 活跃系数"估算。经验值：企业内网工具，活跃系数 5-15%。100 人的部门，峰值并发约 5-15。
- 如果业务方说不出输出长度要求，先用 500 token 做假设，标注为待确认。
- 输入长度要区分平均和 P95。RAG 场景下，P95 可能是平均值的 3 倍以上，规划必须按 P95 算。

**token 换算经验值**：

| 内容类型 | 字符数 | 约合 token（中文） | 约合 token（英文） |
| --- | --- | --- | --- |
| 一句话 | 30 | 30-45 | 8-12 |
| 一段话 | 200 | 200-300 | 50-70 |
| 一篇短文 | 1000 | 1000-1500 | 250-350 |
| 一份文档 | 5000 | 5000-7500 | 1200-1800 |

中文大致是 1 字符 ≈ 1-1.5 token（不同 tokenizer 差异较大，Qwen 系对中文压缩率较好，Llama 系对中文较差，同样中文文本 token 数可能多 30-50%）。这是评估时必须注意的：**同样一段中文，用 Qwen 的 tokenizer 和 Llama 的 tokenizer 算出来的 token 数不一样**，直接影响显存和成本。必须在目标模型上用它的 tokenizer 实测，不要用经验值糊弄。

**预期输出**：填好的需求表。下面是一个填好的示例：

| 输入项 | 值 | 备注 |
| --- | --- | --- |
| 场景 | 内部制度问答 RAG | 有固定 system prompt + 检索到的 3-5 段文档 |
| 输入长度 平均 | 2000 token | system 200 + 检索文档 1500 + 问题 100 + 历史 200 |
| 输入长度 P95 | 6000 token | 检索命中长文档时 |
| 输出长度 平均 | 300 token |  |
| 输出长度 P95 | 800 token |  |
| 并发 | 峰值 15 | 500 人使用，活跃系数 3% |
| TTFT 要求 | < 2 s | 业务方确认 |
| TPOT 要求 | < 80 ms | 业务方确认（约 12 token/s，用户可接受） |
| 质量要求 | 可用 INT8 | 业务方已看对比，确认可接受 |
| 合规 | 数据不出域 | 必须私有化 |
| 预算 | 60 万 |  |

### Step 2：模型选型

**目的**：确定候选模型和精度方案。

**操作**：

先根据场景筛模型。经验规则：

- 通用问答、指令跟随：Qwen2.5 系列、Llama-3.1 系列都是成熟选择
- 中文场景优先选中文语料占比高的模型
- 长文档理解：优先选原生支持 32K 以上上下文的模型，或者确认外推方案可靠
- 代码场景：选代码专项模型，通用模型在代码任务上差距明显
- 需要工具调用（Function Calling）：必须选明确支持且格式稳定的模型，不要自己想当然

然后确定参数量档位。这里有个实务判断：

| 参数量 | 显存（INT8） | 单卡可行性 | 适合场景 |
| --- | --- | --- | --- |
| 1.5B - 3B | 2-3 GB | 任意显卡 | 分类、抽取、简单意图识别 |
| 7B - 8B | 7-8 GB | 单张 24GB 卡 | 通用问答、摘要，质量够用 |
| 14B | 14-16 GB | 单张 24GB 卡（紧）/ 48GB 卡 | 质量明显提升，性价比档 |
| 32B | 32-35 GB | 48GB 卡或 2 张 24GB | 复杂推理、长文档 |
| 72B | 72-80 GB | 80GB 卡单卡或 2 卡 | 高质量场景 |
| 100B+ | 需要多卡 | 8 卡起步 | 追求效果，成本高 |

**关键判断**：不要盲目追大。很多场景 7B/14B 微调或 RAG 之后的效果，比裸的 72B 更好。评估阶段应该把 2-3 个档位都试一下，用真实业务问题对比效果，用数据决定，不要用"参数越大越好"的直觉决定。

**精度决策树**：

```
显存够跑 FP16？
  ├── 是 → 用 FP16/BF16，质量无损，最简单
  └── 否 → 显存够跑 INT8？
        ├── 是 → 用 INT8，质量损失很小
        └── 否 → 业务方接受 INT4？
              ├── 是 → 用 AWQ/GPTQ INT4
              └── 否 → 要么加卡，要么换小模型
```

**预期输出**：候选模型对比表。

| 候选 | 参数 | 层数 | KV头 | 上下文 | 精度 | 权重显存 | 质量评价 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| A | Qwen2.5-14B | 48 | 8 | 32K | INT8 | 约 15 GB | 业务测试通过，答案准确 |
| B | Qwen2.5-32B | 64 | 8 | 32K | INT8 | 约 33 GB | 质量更好，需要 48GB 卡 |
| C | Qwen2.5-32B | 64 | 8 | 32K | INT4 | 约 17 GB | 质量下降可感知 |

**异常表现与处理**：

- 如果用目标模型跑业务问题，发现明显答非所问，先用原始 prompt 手工测几次，排除是 prompt 模板问题而不是模型能力问题
- 如果量化后效果崩了，先检查是不是 tokenizer 或 chat template 配错了，这两个错误造成的效果下降比量化严重得多

### Step 3：显存测算

**目的**：算出每种配置下的显存需求，确定 GPU 数量和拓扑。

**操作**：用下面的脚本自动算，不要手算，手算容易漏项。

先说明脚本的依赖：Python 3.8+，只需标准库，无需安装第三方包。

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""
大模型推理显存测算工具
用法:
    python3 calc_vram.py --config config.json
或直接修改下面的 MODELS 列表运行
仅使用标准库，无需额外依赖
"""

import json
import math
import argparse
import sys

# 常见模型参数（以官方 config.json 为准，此处为示例值）
# 字段: 层数, 隐藏维度, Q头数, KV头数, 单头维度, 参数总量(B)
MODELS = {
    "qwen2.5-7b":   dict(layers=28, hidden=3584, heads=28, kv_heads=4, head_dim=128, params=7.6),
    "qwen2.5-14b":  dict(layers=48, hidden=5120, heads=40, kv_heads=8, head_dim=128, params=14.7),
    "qwen2.5-32b":  dict(layers=64, hidden=5120, heads=40, kv_heads=8, head_dim=128, params=32.5),
    "qwen2.5-72b":  dict(layers=80, hidden=8192, heads=64, kv_heads=8, head_dim=128, params=72.7),
    "llama3.1-8b":  dict(layers=32, hidden=4096, heads=32, kv_heads=8, head_dim=128, params=8.0),
    "llama3.1-70b": dict(layers=80, hidden=8192, heads=64, kv_heads=8, head_dim=128, params=70.6),
}

# 每参数字节数
DTYPE_BYTES = {
    "fp32": 4.0,
    "fp16": 2.0,
    "bf16": 2.0,
    "fp8": 1.0,
    "int8": 1.0,
    "int4": 0.5,
}

GiB = 1024 ** 3
MiB = 1024 ** 2


def kv_bytes_per_token(model, kv_dtype="fp16"):
    """单序列每 token 的 KV Cache 字节数"""
    b = DTYPE_BYTES[kv_dtype]
    # 2 表示 K 和 V 两份
    return 2 * model["layers"] * model["kv_heads"] * model["head_dim"] * b


def calc_vram(model_name, weight_dtype="fp16", kv_dtype="fp16",
              context=8192, concurrency=1, gpu_count=1,
              tensor_parallel=1,
              overhead_gb=2.0, activation_gb=1.0, safety=1.10):
    """
    返回测算结果的字典
    gpu_count: 实际使用的 GPU 数量
    tensor_parallel: 张量并行度，权重按 TP 切分
    """
    if model_name not in MODELS:
        raise ValueError("未知模型: %s，请先添加到 MODELS" % model_name)
    model = MODELS[model_name]

    if weight_dtype not in DTYPE_BYTES:
        raise ValueError("未知权重精度: %s" % weight_dtype)
    if kv_dtype not in DTYPE_BYTES:
        raise ValueError("未知 KV 精度: %s" % kv_dtype)
    if tensor_parallel < 1:
        raise ValueError("tensor_parallel 必须 >= 1")
    if concurrency < 1:
        raise ValueError("concurrency 必须 >= 1")

    # 1. 权重显存
    #    注意：params 是"十亿参数"，× 1e9 得到参数量，再 × 每参数字节 = 字节数
    param_count = model["params"] * 1e9
    weight_bytes = param_count * DTYPE_BYTES[weight_dtype] * safety
    weight_per_gpu = weight_bytes / tensor_parallel

    # 2. KV Cache：TP 会切分 KV 头，所以单卡 KV 也要除以 TP
    kv_per_token = kv_bytes_per_token(model, kv_dtype)
    kv_total = kv_per_token * context * concurrency
    kv_per_gpu = kv_total / tensor_parallel

    # 3. 固定开销
    fixed = overhead_gb * GiB
    act = activation_gb * GiB

    total_per_gpu = weight_per_gpu + kv_per_gpu + fixed + act

    return {
        "model": model_name,
        "weight_dtype": weight_dtype,
        "kv_dtype": kv_dtype,
        "context": context,
        "concurrency": concurrency,
        "tensor_parallel": tensor_parallel,
        # 全部统一用 GiB 口径输出，避免 GB/GiB 混淆
        "weight_total_gib": weight_bytes / GiB,
        "weight_per_gpu_gib": weight_per_gpu / GiB,
        "kv_per_token_kib": kv_per_token / 1024,
        "kv_total_gib": kv_total / GiB,
        "kv_per_gpu_gib": kv_per_gpu / GiB,
        "overhead_gib": overhead_gb,
        "activation_gib": activation_gb,
        "total_per_gpu_gib": total_per_gpu / GiB,
    }


def check_fit(result, gpu_mem_gib, reserve_gib=4.0):
    """
    判断是否装得下。
    reserve_gib: 建议保留的显存余量（碎片、CUDA 上下文波动）
    """
    usable = gpu_mem_gib - reserve_gib
    return result["total_per_gpu_gib"] <= usable, usable


def fmt_result(r, gpu_mem_gib=80, reserve_gib=4.0):
    ok, usable = check_fit(r, gpu_mem_gib, reserve_gib)
    lines = []
    lines.append("=" * 66)
    lines.append("模型: %s | 权重精度: %s | KV精度: %s" %
                 (r["model"], r["weight_dtype"], r["kv_dtype"]))
    lines.append("上下文: %d | 并发: %d | 张量并行: %d" %
                 (r["context"], r["concurrency"], r["tensor_parallel"]))
    lines.append("-" * 66)
    lines.append("权重总量        : %8.2f GiB" % r["weight_total_gib"])
    lines.append("权重/卡         : %8.2f GiB" % r["weight_per_gpu_gib"])
    lines.append("KV/序列/token   : %8.2f KiB" % r["kv_per_token_kib"])
    lines.append("KV 总量         : %8.2f GiB" % r["kv_total_gib"])
    lines.append("KV/卡           : %8.2f GiB" % r["kv_per_gpu_gib"])
    lines.append("框架开销/卡     : %8.2f GiB" % r["overhead_gib"])
    lines.append("激活估计/卡     : %8.2f GiB" % r["activation_gib"])
    lines.append("-" * 66)
    lines.append("单卡总需求      : %8.2f GiB" % r["total_per_gpu_gib"])
    lines.append("单卡可用(扣预留): %8.2f GiB   [卡显存 %d GiB - 预留 %.1f GiB]"
                 % (usable, gpu_mem_gib, reserve_gib))
    lines.append("结论            : %s" % ("装得下" if ok else "装不下，需要调整"))
    lines.append("=" * 66)
    return "\n".join(lines), ok


def main():
    parser = argparse.ArgumentParser(description="大模型推理显存测算")
    parser.add_argument("--model", default="qwen2.5-14b",
                        choices=sorted(MODELS.keys()))
    parser.add_argument("--weight-dtype", default="int8",
                        choices=sorted(DTYPE_BYTES.keys()))
    parser.add_argument("--kv-dtype", default="fp16",
                        choices=sorted(DTYPE_BYTES.keys()))
    parser.add_argument("--context", type=int, default=8192)
    parser.add_argument("--concurrency", type=int, default=15)
    parser.add_argument("--tp", type=int, default=1, help="张量并行度")
    parser.add_argument("--gpu-mem", type=float, default=80.0, help="单卡显存 GiB")
    parser.add_argument("--reserve", type=float, default=4.0, help="预留显存 GiB")
    parser.add_argument("--sweep", action="store_true",
                        help="扫描并发梯度，找出单卡上限")
    args = parser.parse_args()

    if args.sweep:
        print("扫描模式：模型=%s 权重=%s 上下文=%d 卡显存=%.0f GiB" %
              (args.model, args.weight_dtype, args.context, args.gpu_mem))
        print("%-8s %-12s %-12s %s" % ("并发", "单卡需求GiB", "可用GiB", "结论"))
        for c in [1, 2, 5, 10, 15, 20, 30, 50, 80, 100, 150, 200]:
            r = calc_vram(args.model, args.weight_dtype, args.kv_dtype,
                          args.context, c, tensor_parallel=args.tp)
            ok, usable = check_fit(r, args.gpu_mem, args.reserve)
            print("%-8d %-12.2f %-12.2f %s" %
                  (c, r["total_per_gpu_gib"], usable, "OK" if ok else "超限"))
        return 0

    r = calc_vram(args.model, args.weight_dtype, args.kv_dtype,
                  args.context, args.concurrency, tensor_parallel=args.tp)
    text, ok = fmt_result(r, args.gpu_mem, args.reserve)
    print(text)
    return 0 if ok else 2


if __name__ == "__main__":
    sys.exit(main())
```

**运行方式与预期输出**：

```bash
# 单点测算：Qwen2.5-14B INT8，8K 上下文，15 并发，单卡 48GB
python3 calc_vram.py --model qwen2.5-14b --weight-dtype int8 \
    --context 8192 --concurrency 15 --gpu-mem 48 --tp 1
```

预期输出：

```
==================================================================
模型: qwen2.5-14b | 权重精度: int8 | KV精度: fp16
上下文: 8192 | 并发: 15 | 张量并行: 1
------------------------------------------------------------------
权重总量        :    15.06 GiB
权重/卡         :    15.06 GiB
KV/序列/token   :   192.00 KiB
KV 总量         :    22.50 GiB
KV/卡           :    22.50 GiB
框架开销/卡     :     2.00 GiB
激活估计/卡     :     1.00 GiB
------------------------------------------------------------------
单卡总需求      :    40.56 GiB
单卡可用(扣预留):    44.00 GiB   [卡显存 48 GiB - 预留 4.0 GiB]
结论            : 装得下
==================================================================
```

**所有数值统一用 GiB 口径**（1 GiB = 1024³ 字节）。权重那行 15.06 GiB 的计算过程是：`14.7e9 参数 × 1 字节/参数 × 1.10 安全系数 = 16.17e9 字节 = 15.06 GiB`。

**关于单位口径的实务提醒**：厂商标注的显存（比如"80GB"卡）通常是 GB 还是 GiB 并不统一。`nvidia-smi` 显示的 `memory.total` 是 MiB 口径。**评估报告里必须写明用的是什么口径**，否则 80GB 的卡算出来"只剩 79 GiB 可用"会让人困惑。为避免歧义，本文一律用 GiB，并在脚本里统一换算。

**扫描模式**（找单卡并发上限）：

```bash
python3 calc_vram.py --model qwen2.5-14b --weight-dtype int8 \
    --context 8192 --gpu-mem 48 --sweep
```

预期输出：

```
扫描模式：模型=qwen2.5-14b 权重=int8 上下文=8192 卡显存=48 GiB
并发     单卡需求GiB   可用GiB      结论
1        19.56        44.00        OK
2        21.06        44.00        OK
5        25.56        44.00        OK
10       33.06        44.00        OK
15       40.56        44.00        OK
20       48.06        44.00        超限
30       63.06        44.00        超限
```

**验算方式**：固定部分是 `权重 15.06 + 框架开销 2 + 激活 1 = 18.06 GiB`。KV 部分每并发增加 `192 KiB × 8192 / 1GiB = 1.5 GiB`。所以并发 N 时的单卡需求 = `18.06 + 1.5 × N`。

- 并发 1：18.06 + 1.5 = 19.56
- 并发 15：18.06 + 22.5 = 40.56
- 并发 20：18.06 + 30 = 48.06（超过可用的 44 GiB）

**判断逻辑**：从这张表能直接读出来，48GB 卡跑 Qwen2.5-14B INT8、8K 上下文，理论并发上限在 15-20 之间（约 17 左右）。**这个数字是理论上限，实际能跑多少要以压测为准，通常会比理论值低**，因为实际序列长度分布不均、显存有碎片、框架还需要额外空间。

**异常表现与处理**：

- 如果算出来的并发上限远低于业务需求，先看 KV 那行是不是主要矛盾。如果是，优先考虑开 KV Cache 量化（FP8）或者降低上下文
- 如果权重那行就快占满了，说明模型太大或者精度选错了，考虑降精度或换小模型
- 如果理论上"装得下"但实际启动就 OOM，检查 `--max-model-len` 是不是设得远大于实际需要，vLLM 会按这个值预分配 KV Cache 池

**下一步动作**：拿着测算结果去确认 GPU 采购清单，或者反过来确认业务需求能不能调整。

### Step 4：GPU 选型与拓扑设计

**目的**：确定买什么卡、几张、单机还是多机、并行度多少。

**操作**：

先确定单卡显存档位：

```
单卡需求（Step 3 结果）
  ├── < 20 GiB  → 24GB 卡（4090 / A10）
  ├── 20-44 GiB → 48GB 卡（L20 / L40S / A40）
  ├── 44-76 GiB → 80GB 卡（A100 / A800 / H 系列）
  └── > 76 GiB  → 必须多卡并行
```

然后确定并行策略：

| 情况 | 方案 | 说明 |
| --- | --- | --- |
| 单卡装得下 | TP=1，单实例 | 最简单，无通信开销，性能最好 |
| 单卡装不下，同机多卡装得下 | TP=N，单实例跨卡 | 需要 NVLink 或高速 PCIe，通信开销随 N 增长 |
| 多机才装得下 | TP + PP，或者多机 TP | 需要 IB/RoCE 高速网络，复杂度大幅上升 |
| 单卡装得下但要更高吞吐 | 多实例，每卡一个实例 | 各实例独立，负载均衡分发，隔离性好 |

**关于 TP 的实务经验**：

- TP=1 到 TP=2 通常收益明显（能装下更大的模型），延迟增加不大
- TP=2 到 TP=4，吞吐能提升但延迟开始上升，需要 NVLink
- TP=4 以上，如果没有 NVLink，性能可能不升反降
- TP 度通常不要超过单机的卡数（8），跨机 TP 对网络要求极高

**关于高可用**：如果业务要求不能单点，需要冗余。三种方式：

1. **主备**：一套跑着，一套待命。成本翻倍，但切换简单
2. **双活 + 负载均衡**：两套都跑，前面挂负载均衡。成本翻倍，但资源利用率高
3. **降级方案**：单套 + 可切换的公有云 API 兜底。成本最低，但需要数据可以出域

对多数企业内网工具，**方案 3 或者"单套 + 快速重建"** 是更现实的选择。评估报告里要把这三种方案的成本和风险都列出来，让业务方选。

**预期输出**：硬件配置清单。

```
示例配置 A（成本优先）：
  1 台 GPU 服务器，2 张 L20 48GB
  用途：Qwen2.5-14B INT8，8K 上下文，并发 15
  预估性能：TTFT < 1.5s，TPOT < 60ms
  预算量级：单卡数万元（以当期采购价为准）

示例配置 B（性能优先）：
  1 台 GPU 服务器，8 张 A800 80GB
  用途：Qwen2.5-72B INT8，32K 上下文，并发 30
  预估性能：待压测
  预算量级：整机百万级
```

### Step 5：部署与压测

**目的**：用真实数据替换理论测算。

**操作**：

先在能拿到的任何硬件上把模型跑起来。哪怕是单张 4090 跑量化版小模型，也能验证部署流程、验证 tokenizer 和 chat template 是否正确。

关于推理引擎的选择，主流选项：

| 引擎 | 优势 | 适用 |
| --- | --- | --- |
| vLLM | 生态成熟，PagedAttention，支持模型多，OpenAI 兼容 API | 通用首选 |
| TGI (Text Generation Inference) | HuggingFace 官方，部署简单，生产验证充分 | 追求稳定 |
| TensorRT-LLM | NVIDIA 官方，极致性能，需编译 | 追求极限性能，有 GPU 编译能力 |
| SGLang | RadixAttention 前缀缓存强，结构化输出好 | 多轮对话、RAG 复用多 |
| llama.cpp | CPU/混合推理，量化格式多 | 无 GPU 或边缘场景 |

下文以 vLLM 为例，因为它的 OpenAI 兼容接口最通用，压测工具也最完善。

**Step 5.1 环境准备检查**

```bash
# 检查 GPU 是否可见
nvidia-smi -L

# 预期输出
# GPU 0: NVIDIA A800 80GB PCIe (UUID: GPU-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx)
# GPU 1: NVIDIA A800 80GB PCIe (UUID: GPU-yyyyyyyy-...)

# 检查驱动和 CUDA 版本
nvidia-smi | head -4

# 预期输出（示例，版本以实际为准）
# | NVIDIA-SMI 535.xx    Driver Version: 535.xx    CUDA Version: 12.2  |

# 检查拓扑（多卡并行必须看这个）
nvidia-smi topo -m

# 关键看 GPU0-GPU1 交叉点：
#   NV12 表示 NVLink 12 链路（最好）
#   PIX  表示同一 PCIe 交换芯片下
#   PHB  表示同一 CPU 下的不同 PCIe 主机桥
#   SYS  表示跨 NUMA 节点，走 QPI/UPI，最慢

# 检查 Python 与 PyTorch
python3 -c "import torch; print(torch.__version__, torch.cuda.device_count())"

# 预期输出
# 2.x.x 2
```

**异常表现与处理**：

- `nvidia-smi -L` 报 “No devices were found”：驱动没装好或者被内核模块屏蔽了，检查 `lsmod | grep nvidia`
- 卡能看到但 `torch.cuda.device_count()` 返回 0：容器里没挂 GPU，检查启动参数有没有 `--gpus all` 或者 `NVIDIA_VISIBLE_DEVICES`
- `nvidia-smi topo -m` 显示 SYS：说明卡跨了 NUMA，TP 性能会差，建议绑核或者调整 PCIe 插槽

**Step 5.2 启动服务**

以 Qwen2.5-14B INT8、单卡 48GB 为例：

```bash
# 注意：--gpu-memory-utilization 是显存占用比例，不是字节数
# 0.90 表示允许引擎使用 GPU 显存的 90% 作为 KV Cache 池 + 权重
vllm serve /data/models/Qwen2.5-14B-Instruct-AWQ \
  --served-model-name qwen14b \
  --host 0.0.0.0 \
  --port 8000 \
  --quantization awq \
  --dtype float16 \
  --max-model-len 8192 \
  --gpu-memory-utilization 0.90 \
  --max-num-seqs 32 \
  --enable-prefix-caching \
  --enable-chunked-prefill \
  --trust-remote-code \
  2>&1 | tee /var/log/vllm/qwen14b.log
```

参数说明：

| 参数 | 作用 | 调优建议 |
| --- | --- | --- |
| `--max-model-len` | 最大序列长度，决定 KV Cache 池的分配粒度 | 按 P95 输入 + P95 输出 + 余量设置，不要照搬模型的 32K 上限 |
| `--gpu-memory-utilization` | 引擎可用的 GPU 显存比例 | 默认 0.9。显存紧张时降到 0.85-0.88，留更多给 CUDA 上下文 |
| `--max-num-seqs` | 单批次最大并发序列数 | 控制瞬时并发上限。值太大容易 OOM，值太小浪费吞吐 |
| `--enable-prefix-caching` | 开启前缀缓存 | RAG 场景必开 |
| `--enable-chunked-prefill` | 分块预填充 | 长 prompt 场景建议开，改善尾延迟 |
| `--tensor-parallel-size` | 张量并行度 | 单卡不设，多卡设为卡数 |
| `--quantization` | 量化方法 | 按实际模型格式填 awq / gptq / fp8 等 |

**版本差异提醒**：vLLM 的参数名和默认值在不同版本间有变化。比如 `--enable-chunked-prefill` 在某些版本中默认开启；`--quantization` 的取值集合也随版本增加。部署前必须查对应版本文档，本文的参数名以 0.5-0.6 系列为参考。

**启动成功的标志**：日志里会出现类似下面几行，关键看 KV Cache 分配了多少：

```
INFO ... Initializing an LLM engine with config: ...
INFO ... model weights take 15.02GiB; non_torch_memory takes 0.15GiB;
          PyTorch activation peak memory takes 1.31GiB;
          the rest of the memory reserved for KV Cache is 24.5GiB.
INFO ... # GPU blocks: 4685, # CPU blocks: 3276
INFO ... Maximum concurrency for 8192 tokens per request: 36.60x
```

最后那行 `Maximum concurrency for 8192 tokens per request` 就是引擎自己算出来的理论上限，可以直接和 Step 3 的手算结果对照。**如果这两个数字差很多，说明手算的 `overhead_gb` 或 `activation_gb` 假设偏了，要修正。**

**Step 5.3 功能验证**

```bash
# 探活
curl -s http://127.0.0.1:8000/health -o /dev/null -w "%{http_code}\n"
# 预期：200

# 列出模型
curl -s http://127.0.0.1:8000/v1/models | python3 -m json.tool

# 单次推理
curl -s http://127.0.0.1:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "qwen14b",
    "messages": [{"role": "user", "content": "用三句话说明什么是 TCP 三次握手"}],
    "max_tokens": 256,
    "temperature": 0.2
  }' | python3 -m json.tool
```

返回里重点看 `usage` 字段：

```
"usage": {
  "prompt_tokens": 34,
  "completion_tokens": 187,
  "total_tokens": 221
}
```

**这一步是校准 token 数量的关键**。拿几条真实业务数据（前面填的需求表里说的"2000 token 输入"），实际发一遍，看 `prompt_tokens` 是多少。经验值经常偏差 30% 以上，尤其是中文内容。

**Step 5.4 压测**

压测要分两个维度做，不能只做一种：

**维度一：固定并发，看延迟。** 目标是把并发梯度拉满，找到延迟拐点。

```bash
# vLLM 自带压测工具（版本不同命令可能不同，老版本为 benchmarks/benchmark_serving.py）
vllm bench serve \
  --backend openai-chat \
  --model qwen14b \
  --base-url http://127.0.0.1:8000 \
  --dataset-name random \
  --random-input-len 2000 \
  --random-output-len 300 \
  --num-prompts 200 \
  --max-concurrency 15 \
  --request-rate 2 \
  --save-result \
  --result-dir /data/bench/result-14b-c15
```

参数含义：

| 参数 | 说明 |
| --- | --- |
| `--random-input-len` | 随机数据集的输入长度，要按实测的业务平均长度设 |
| `--random-output-len` | 输出长度，同上 |
| `--num-prompts` | 总请求数，建议至少是并发数的 10 倍 |
| `--max-concurrency` | 最大并发 |
| `--request-rate` | 每秒请求数。设为 `inf` 表示不限速，能打多快打多快；设为具体值模拟真实到达率 |

**关于数据集**：`random` 数据集用的是随机 token，和真实数据的长度分布不同，测出来的绝对数字会偏乐观。更准确的做法是用真实业务日志构造数据集（vLLM 支持 `--dataset-name custom` 加 `--dataset-path` 指定 jsonl 文件，每行一个请求）。评估阶段建议两种都跑：random 快速摸上限，custom 验证真实场景。

**压测梯度**：并发从 1 开始，按 1 / 2 / 4 / 8 / 15 / 20 / 30 递增，每个梯度跑一轮，记录指标。下面是记录表的模板：

| 并发 | TTFT 平均 | TTFT P99 | TPOT 平均 | TPOT P99 | 输出吞吐 token/s | 请求吞吐 req/s | GPU 显存 | 结论 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- |
| 1 |  |  |  |  |  |  |  | 基线 |
| 2 |  |  |  |  |  |  |  |  |
| 4 |  |  |  |  |  |  |  |  |
| 8 |  |  |  |  |  |  |  |  |
| 15 |  |  |  |  |  |  |  |  |
| 20 |  |  |  |  |  |  |  |  |
| 30 |  |  |  |  |  |  |  | 是否 OOM |

**怎么读这张表**：

- 并发增加，TTFT 和 TPOT 会上升，这是正常的
- 输出吞吐会先上升，到某个点后趋于平缓，这个点就是**吞吐拐点**，通常是最佳工作点
- 如果某个梯度出现 OOM 或者请求失败，那就是硬上限
- **找到"延迟刚好满足 SLA"的那个并发值，这就是推荐的最大并发**。超过这个值延迟会破线

**维度二：固定速率，看稳定性。** 用真实到达率跑长时间（至少 30 分钟），看有没有内存泄漏、有没有延迟漂移。

```bash
# request-rate 设为业务实际到达率，跑 30 分钟
vllm bench serve \
  --backend openai-chat \
  --model qwen14b \
  --base-url http://127.0.0.1:8000 \
  --dataset-name random \
  --random-input-len 2000 \
  --random-output-len 300 \
  --num-prompts 5400 \
  --max-concurrency 20 \
  --request-rate 3 \
  --result-dir /data/bench/result-14b-stab
```

5400 个请求按 3 req/s 算，约 30 分钟。**长时间压测期间，另开一个终端持续采样显存和利用率**：

```bash
# 每 5 秒采样一次，输出到文件，持续 1 小时
nvidia-smi --query-gpu=timestamp,index,utilization.gpu,memory.used,memory.total,power.draw \
  --format=csv,noheader,nounits \
  -l 5 | tee /data/bench/gpu-during-bench.csv
```

**预期输出**（片段）：

```
2026/01/15 10:00:05, 0, 87, 43210, 49140, 298.4
2026/01/15 10:00:10, 0, 91, 43580, 49140, 301.2
```

**怎么判断**：显存 `memory.used` 应该在启动后很快达到平台期，然后长时间稳定。如果看到显存**持续缓慢上涨**，说明有泄漏或者 KV Cache 没有正确回收，这是严重问题，必须查。

**Step 5.5 结果分析**

把压测数据整理成结论。要回答的问题：

1. 目标并发（15）下，TTFT 和 TPOT 是否满足 SLA？不满足的话，最大能支持多少并发？
2. 单卡/单机的输出吞吐是多少 token/s？用这个算单位成本
3. 有没有出现 OOM？在哪个并发梯度？
4. 长时间运行稳定性如何？有没有延迟漂移？
5. 和 Step 3 的理论测算差多少？差异来自哪里？

**典型偏差来源**：

- 输入 token 数实测比估算多：中文 tokenizer 压缩率低于预期
- 激活值实际占用比假设的大：长 prompt 的 prefill 峰值
- 实际序列长度分布不均：短请求等待长请求，有效并发低于名义并发
- 网络和后端框架开销：走 HTTP 和网关有额外延迟

### Step 6：成本核算

**目的**：算出单位成本，给出和公有云 API 的对比。

**操作**：用下面的脚本算。所有输入参数都要用实际采购价和实测吞吐，不要用估值。

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""
大模型私有化部署成本核算
仅依赖标准库
"""

import argparse
import sys


def annual_cost(capex, depreciation_years, power_kw, pue, price_per_kwh,
                hours_per_year=8760, labor=0.0, network=0.0,
                software=0.0, spares=0.0):
    """计算年度总成本（元）"""
    depreciation = capex / depreciation_years
    electricity = power_kw * pue * hours_per_year * price_per_kwh
    total = depreciation + electricity + labor + network + software + spares
    return {
        "depreciation": depreciation,
        "electricity": electricity,
        "labor": labor,
        "network": network,
        "software": software,
        "spares": spares,
        "total": total,
    }


def unit_cost(total_annual, tokens_per_second, daily_active_hours,
              utilization=1.0, days_per_year=365):
    """
    计算单位 token 成本（元/千 token）
    注意：utilization 必须参与计算，否则会严重低估单价
    """
    effective_tps = tokens_per_second * utilization
    annual_tokens = effective_tps * 3600 * daily_active_hours * days_per_year
    if annual_tokens <= 0:
        raise ValueError("年度 token 数为 0，检查 throughput / utilization / active hours")
    return total_annual / annual_tokens * 1000, annual_tokens


def cloud_cost_same_volume(annual_tokens, cloud_price_per_1k):
    """同样 token 量走公有云需要多少钱"""
    return annual_tokens / 1000 * cloud_price_per_1k


def main():
    p = argparse.ArgumentParser(description="私有化大模型成本核算")
    p.add_argument("--capex", type=float, default=1200000, help="一次性投入，元")
    p.add_argument("--years", type=float, default=3, help="折旧年限")
    p.add_argument("--power-kw", type=float, default=6.0, help="整机功耗 kW")
    p.add_argument("--pue", type=float, default=1.4, help="机房 PUE")
    p.add_argument("--kwh-price", type=float, default=0.8, help="电价 元/kWh")
    p.add_argument("--labor", type=float, default=150000, help="年人力成本，元")
    p.add_argument("--network", type=float, default=30000, help="年网络成本，元")
    p.add_argument("--software", type=float, default=20000, help="年软件许可，元")
    p.add_argument("--spares", type=float, default=50000, help="年备件维保，元")
    p.add_argument("--throughput", type=float, default=2500,
                   help="实测输出吞吐 token/s")
    p.add_argument("--active-hours", type=float, default=12,
                   help="日均有效运行小时")
    p.add_argument("--cloud-price", type=float, default=0.008,
                   help="公有云参考价 元/千 token（输出）")
    p.add_argument("--utilization", type=float, default=1.0,
                   help="综合利用率系数，0-1，用于折算真实成本")
    args = p.parse_args()

    c = annual_cost(args.capex, args.years, args.power_kw, args.pue,
                    args.kwh_price, labor=args.labor, network=args.network,
                    software=args.software, spares=args.spares)

    print("=" * 60)
    print("年度成本构成")
    print("-" * 60)
    for k, name in [("depreciation", "硬件折旧"), ("electricity", "电费"),
                    ("labor", "人力"), ("network", "网络"),
                    ("software", "软件许可"), ("spares", "备件维保")]:
        print("%-12s: %14.2f 元" % (name, c[k]))
    print("-" * 60)
    print("%-12s: %14.2f 元" % ("年度合计", c["total"]))

    uc, annual_tokens = unit_cost(c["total"], args.throughput,
                                  args.active_hours, args.utilization)
    print()
    print("年度处理输出 token : %.2f 亿" % (annual_tokens / 1e8))
    print("单位成本           : %.6f 元/千 token" % uc)
    print("  （已按利用率 %.0f%% 折算，满负载理论值 %.6f 元/千 token）" %
          (args.utilization * 100,
           uc * args.utilization))
    print()
    cloud = cloud_cost_same_volume(annual_tokens, args.cloud_price)
    print("同等 token 量年费用对比")
    print("  自建            : %.2f 元/年（固定，与用量无关）" % c["total"])
    print("  公有云          : %.2f 元/年（按量，%.4f 元/千 token）"
          % (cloud, args.cloud_price))
    print("  结论            : %s" %
          ("自建更省，年省 %.2f 元" % (cloud - c["total"]) if cloud > c["total"]
           else "公有云更省，年省 %.2f 元" % (c["total"] - cloud)))
    if args.cloud_price > 0:
        be = c["total"] / args.cloud_price * 1000
        print("  盈亏平衡点      : 年 %.2f 亿 token（低于此量公有云更划算）"
              % (be / 1e8))
        print("  当前用量占比    : %.1f%%" % (annual_tokens / be * 100))
    print("=" * 60)
    print()
    print("提醒：以上成本只含推理服务本身，不含上游应用、向量库、")
    print("      网关、运维平台、模型迭代调优等成本，做整体预算时要加。")
    return 0


if __name__ == "__main__":
    sys.exit(main())
```

**运行与预期输出**：

```bash
python3 cost_calc.py --capex 1200000 --throughput 2500 --active-hours 12 \
    --utilization 0.5 --cloud-price 0.008
```

预期输出：

```
============================================================
年度成本构成
------------------------------------------------------------
硬件折旧        :      400000.00 元
电费            :       58867.20 元
人力            :      150000.00 元
网络            :       30000.00 元
软件许可        :       20000.00 元
备件维保        :       50000.00 元
------------------------------------------------------------
年度合计        :      708867.20 元

年度处理输出 token : 197.10 亿
单位成本           : 0.035965 元/千 token
  （已按利用率 50% 折算，满负载理论值 0.017982 元/千 token）

同等 token 量年费用对比
  自建            : 708867.20 元/年（固定，与用量无关）
  公有云          : 157680.00 元/年（按量，0.0080 元/千 token）
  结论            : 公有云更省，年省 551187.20 元
  盈亏平衡点      : 年 886.08 亿 token（低于此量公有云更划算）
  当前用量占比    : 22.2%
============================================================

提醒：以上成本只含推理服务本身，不含上游应用、向量库、
      网关、运维平台、模型迭代调优等成本，做整体预算时要加。
```

**电费那行怎么算出来的**：`6.0 kW × 1.4 (PUE) × 8760 h × 0.8 元/kWh = 58867.20 元`。

**单位成本那行怎么算出来的**（这一步最容易算错，务必自己验算）：

```
年度输出 token = 2500 token/s × 50% × 3600 s × 12 h × 365 天
              = 1250 × 3600 × 12 × 365
              = 1.971e10 token
              = 197.1 亿 token

单位成本 = 708867.20 元 / (1.971e10 / 1000) 千token
        = 708867.20 / 1.971e7
        = 0.035965 元/千 token
```

**换算关系要记清楚**：1 亿 token = 1e8 token = 1e5 千token。所以"年度 197.1 亿 token"是 1.971e7 千token，不是 1.971e8。

**怎么读这个结果**：

按 50% 利用率、日均有效 12 小时算，这台机器一年处理约 197 亿 token，单位成本 **0.036 元/千 token**，而公有云的参考价是 0.008 元/千 token。**自建的单位成本是公有云的 4.5 倍。**

关键原因是**这台机器一年只处理 197 亿 token，远低于盈亏平衡点**。盈亏平衡点的算法：

```
盈亏平衡 token 量 = 年度成本 / 公有云单价 × 1000
                 = 708867.20 / 0.008 × 1000
                 = 8.861e10 token
                 = 886 亿 token/年
```

当前用量 197 亿，只到盈亏平衡点的 22%。**在这个用量下，公有云明显更省（年省 55 万）。**

要让它变得划算，有三条路：

1. **提高用量**：把更多业务迁到这套服务上，年用量要涨到 4.5 倍以上
2. **提高利用率**：夜间跑离线批处理，把 `--active-hours` 从 12 提到 20 以上，同时 `--utilization` 提到 0.8。这样年处理量能到 2500 × 0.8 × 3600 × 20 × 365 = 5.26e10 = 526 亿 token，单位成本降到 0.0135 元/千 token，接近但仍高于公有云
3. **降低成本**：换个更便宜的硬件方案。比如用单张 L20 替代多卡 A800 整机，CAPEX 从 120 万降到 10 万级，年度成本降到 20 万以内，盈亏平衡点降到 250 亿 token 左右

第三条路的效果最直接。**这也说明一个常见的规划错误：为了"以后可能的大规模使用"提前买 8 卡整机，结果长期只用 20% 的负载，单位成本高得离谱。** 更合理的做法是先买少量卡验证业务需求，用量上来再扩。

**必须说清楚的一点**：很多企业上私有化是因为合规，不是因为便宜。**评估报告里要把这两件事分开说**：合规驱动 vs 成本驱动。如果是合规驱动，成本高也要做，但必须把成本说清楚，让决策层知道代价是多少，而不是含糊地讲"长期看更省钱"。

**还要注意一件事**：上面的对比用的是输出 token 单价。公有云的输入 token 单价通常远低于输出 token（大概是 1/4 到 1/3），如果业务输入很长（RAG 场景输入可能是输出的 5-10 倍），对比时要把输入输出分开算。混合单价公式：

```
混合单价(元/千token) = (输入token数 × 输入单价 + 输出token数 × 输出单价) / 总token数
```

**自建侧也要做同样的拆分**：上面的 2500 token/s 是输出吞吐，输入 token 的处理成本（prefill）已经包含在同一台机器里了。所以严格来说，自建的单位成本应该按"输入+输出总 token"分摊，这样单位成本会更低一些。评估时两种口径都要算，报告里说明用的是哪种。RAG 场景下按总 token 分摊，自建的单位成本大约能降到 0.008-0.01 元/千 token，和公有云基本持平——**这正好说明口径选择对结论影响很大，必须在报告里写清楚**。

## 六、常用命令速查

这一节把评估和部署过程中高频使用的命令集中起来，每条都标注用途和关键输出怎么读。命令按场景分组，方便现场查。

### 6.1 硬件与驱动检查

```bash
# 列出所有 GPU 及型号（最常用）
nvidia-smi -L
# 输出示例：
# GPU 0: NVIDIA A800 80GB PCIe (UUID: GPU-1234abcd-...)
# GPU 1: NVIDIA A800 80GB PCIe (UUID: GPU-5678efgh-...)
# 用途：确认卡的数量、型号、是否被其他进程占用导致不可见

# 完整状态（驱动版本、CUDA 版本、每卡利用率/显存/温度/功耗/进程）
nvidia-smi
# 关键看的几行：
#   顶部 Driver Version / CUDA Version —— 决定能跑什么版本的推理框架
#   中部 Fan/Temp/Perf/Pwr —— 功耗接近 TDP 说明卡在满载
#   中部 Memory-Usage —— 当前显存占用，已加载模型时这里是权重+KV
#   底部 Processes —— 有没有别人在用这张卡，评估前务必确认

# 持续监控，每 2 秒刷新
nvidia-smi -l 2

# 只取关键字段，便于抓取和记录（评估压测时最有用）
nvidia-smi --query-gpu=timestamp,index,name,utilization.gpu,utilization.memory,memory.used,memory.total,temperature.gpu,power.draw,clocks.sm \
  --format=csv,noheader,nounits -l 5

# 字段含义速查：
#   utilization.gpu     SM 占用率，% 。推理 decode 阶段通常不高（30-60%），prefill 阶段高
#   utilization.memory  显存控制器带宽占用率，% 。decode 阶段这个指标高说明是带宽瓶颈
#   memory.used         已用显存 MiB
#   temperature.gpu     核心温度 ℃ ，长期 > 80 要注意散热
#   power.draw          实时功耗 W
#   clocks.sm           SM 当前频率 MHz，掉频说明撞功耗墙或温度墙

# 查看 GPU 拓扑（多卡并行前必看）
nvidia-smi topo -m
# 交叉点含义：
#   NV#    NVLink，数字是链路数，如 NV12 表示 12 条链路，带宽最高
#   PIX    同一 PCIe 交换机下
#   PXB    跨多个 PCIe 交换机
#   PHB    同一 CPU 下的不同 PCIe 主机桥
#   SYS    跨 NUMA 节点（走 UPI/QPI），最慢
# 判断：做 TP 并行的几张卡之间，最好都是 NV# 或 PIX，出现 SYS 就要重新规划

# 查看 PCIe 链路速率和宽度
nvidia-smi --query-gpu=index,pcie.link.gen.current,pcie.link.width.current \
  --format=csv
# 理想值：gen 4 / x16。如果显示 x8 或 gen 1，可能是插槽带宽不足或卡没插好

# 是否有 ECC 错误（长期运行的卡要关注）
nvidia-smi --query-gpu=index,ecc.errors.uncorrected.volatile.total,ecc.errors.corrected.volatile.total \
  --format=csv
# uncorrected 非 0 是很严重的问题，会导致静默数据错误，需要报修换卡

# 查看 MIG 配置（如果是 A100/H100 且启用了 MIG）
nvidia-smi mig -lgi
```

### 6.2 系统资源检查

```bash
# CPU 型号、核数、NUMA 布局（决定能否绑核优化）
lscpu | grep -E "Model name|^CPU\(s\)|NUMA|Socket|Core"

# NUMA 节点与 GPU 亲和性（多卡时重要）
lspci -tv | grep -i nvidia
# 或者
nvidia-smi topo -m   # 结合上面看

# 内存总量与可用
free -h
# 关键：可用内存要能放下模型权重的一份 CPU 拷贝（加载时临时需要）

# 磁盘空间（模型文件动辄几十 GB）
df -h /data
# 经验：留出至少 2 倍模型大小的空间，因为要放原始权重 + 可能还有转换后的格式

# 磁盘 IO 性能（模型加载速度取决于这个）
# 注意：fio 会写磁盘，只在测试盘上用，不要在数据盘上直接跑
fio --name=readtest --filename=/data/testfile --size=2G --rw=read \
    --bs=1M --direct=1 --numjobs=1 --runtime=30 --group_reporting
# 关键看 bw（带宽）。全闪存应该在 GB/s 量级，机械盘只有百 MB/s 量级

# 网络（多机部署时看）
ip -br addr
# 查看网卡速率
ethtool eth0 | grep -E "Speed|Duplex"
# 查看是否有 RDMA 网卡（IB 或 RoCE）
ibstat 2>/dev/null || echo "no IB device"
# 查看 RoCE 是否启用
ls /sys/class/infiniband/ 2>/dev/null

# 检查是否有其他推理进程在占卡
nvidia-smi --query-compute-apps=pid,process_name,used_memory --format=csv

# 供电与散热（8 卡整机必查）
# 查看电源相关传感器（需要 ipmitool，且要 BMC 权限）
ipmitool sensor list 2>/dev/null | grep -iE "power|temp" | head -20
```

### 6.3 推理引擎与服务检查

```bash
# 探活
curl -sS -o /dev/null -w "health=%{http_code} time=%{time_total}s\n" \
  http://127.0.0.1:8000/health

# 列出已加载模型
curl -s http://127.0.0.1:8000/v1/models | python3 -m json.tool

# 单次推理，看 usage 校准 token 数
curl -s http://127.0.0.1:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "qwen14b",
    "messages": [{"role":"user","content":"测试"}],
    "max_tokens": 16
  }' | python3 -c "
import sys, json
d = json.load(sys.stdin)
print('prompt_tokens    :', d['usage']['prompt_tokens'])
print('completion_tokens:', d['usage']['completion_tokens'])
"

# 流式推理，实测 TTFT（首 token 延迟）
# 用 curl 的 -w 无法拿到首字节时间，用一个小脚本更准
python3 - <<'PY'
import json, time, urllib.request

url = "http://127.0.0.1:8000/v1/chat/completions"
payload = {
    "model": "qwen14b",
    "messages": [{"role": "user", "content": "用一句话解释什么是负载均衡"}],
    "max_tokens": 128,
    "stream": True,
}
req = urllib.request.Request(
    url,
    data=json.dumps(payload).encode("utf-8"),
    headers={"Content-Type": "application/json"},
)
t0 = time.time()
ttft = None
n = 0
with urllib.request.urlopen(req, timeout=60) as resp:
    for raw in resp:
        line = raw.decode("utf-8").strip()
        if not line or not line.startswith("data:"):
            continue
        body = line[5:].strip()
        if body == "[DONE]":
            break
        try:
            chunk = json.loads(body)
        except json.JSONDecodeError:
            continue
        delta = chunk.get("choices", [{}])[0].get("delta", {})
        if delta.get("content"):
            if ttft is None:
                ttft = time.time() - t0
            n += 1
t_end = time.time() - t0
print("TTFT      : %.3f s" % (ttft if ttft else -1))
print("总耗时    : %.3f s" % t_end)
print("输出 token: %d" % n)
if ttft and n > 1:
    print("TPOT      : %.1f ms" % ((t_end - ttft) / (n - 1) * 1000))
PY

# 查看推理引擎的 Prometheus 指标（vLLM 默认在 /metrics）
curl -s http://127.0.0.1:8000/metrics | grep -E "^vllm:" | head -40

# 关键指标名（vLLM 0.5+，不同版本命名可能变化，以实际 /metrics 输出为准）：
#   vllm:num_requests_running        正在处理的请求数
#   vllm:num_requests_waiting        排队等待的请求数
#   vllm:num_requests_swapped        被换出的请求数
#   vllm:gpu_cache_usage_perc        KV Cache 使用率（0-1），最重要的指标之一
#   vllm:cpu_cache_usage_perc        CPU 侧 KV 使用率
#   vllm:time_to_first_token_seconds TTFT 直方图
#   vllm:time_per_output_token_seconds TPOT 直方图
#   vllm:e2e_request_latency_seconds 端到端延迟直方图
#   vllm:prompt_tokens_total         累计处理输入 token
#   vllm:generation_tokens_total     累计生成输出 token
#   vllm:request_success_total       成功请求数
#   vllm:prefix_cache_hits_total     前缀缓存命中数（如果启用了）
```

**这里有一条实战经验**：`vllm:gpu_cache_usage_perc` 是最该盯的指标。它长期高于 0.9，说明 KV Cache 快满了，请求会开始排队；长期低于 0.3，说明显存配得过于保守，可以调大 `--max-num-seqs` 或 `--max-model-len` 提升吞吐。**评估阶段就要把这条曲线采集下来，它是判断配置是否合理的直接证据。**

### 6.4 压测相关命令

```bash
# vLLM 新版自带压测（v0.6+ 命令形式）
vllm bench serve \
  --backend openai-chat \
  --model qwen14b \
  --base-url http://127.0.0.1:8000 \
  --dataset-name random \
  --random-input-len 2000 \
  --random-output-len 300 \
  --num-prompts 200 \
  --max-concurrency 15 \
  --save-result \
  --result-dir /data/bench/run-c15

# 老版本路径形式（v0.5 及以前）
# python3 benchmarks/benchmark_serving.py \
#   --backend openai-chat --model qwen14b \
#   --base-url http://127.0.0.1:8000 \
#   --dataset-name random --random-input-len 2000 --random-output-len 300 \
#   --num-prompts 200 --max-concurrency 15

# 用真实业务数据压测（推荐，比 random 准）
# 准备 jsonl，每行一个请求，字段按引擎要求
cat > /data/bench/custom.jsonl <<'EOF'
{"prompt": "公司年假怎么申请？", "completion": ""}
{"prompt": "报销流程是什么？", "completion": ""}
EOF
# 注意：不同版本的 custom 数据集字段名不同，
# 有的要求 conversation/prompt，有的要求 prompt+completion，
# 必须查对应版本源码里的 dataset 定义，不要照搬。

# 用 hey / wrk 打 HTTP 层（测网关和连接数上限，测不出 token 级指标）
# hey 需要单独安装
hey -n 1000 -c 20 -m POST \
  -H "Content-Type: application/json" \
  -d '{"model":"qwen14b","messages":[{"role":"user","content":"hi"}],"max_tokens":32}' \
  http://127.0.0.1:8000/v1/chat/completions
# 用途：快速确认服务在 20 并发下会不会 5xx，不做精细指标分析
```

### 6.5 模型文件检查

```bash
# 查看模型目录结构和体积
du -sh /data/models/Qwen2.5-14B-Instruct-AWQ
ls -lh /data/models/Qwen2.5-14B-Instruct-AWQ/

# 读 config.json 关键字段（显存测算的数据源）
python3 -c "
import json
c = json.load(open('/data/models/Qwen2.5-14B-Instruct-AWQ/config.json'))
keys = ['num_hidden_layers','hidden_size','num_attention_heads',
        'num_key_value_heads','head_dim','vocab_size',
        'max_position_embeddings','torch_dtype','model_type',
        'quantization_config']
for k in keys:
    print('%-24s: %s' % (k, c.get(k, '<not set>')))
"
# 预期输出示例：
# num_hidden_layers       : 48
# hidden_size             : 5120
# num_attention_heads     : 40
# num_key_value_heads     : 8
# head_dim                : 128
# vocab_size              : 152064
# max_position_embeddings : 32768
# torch_dtype             : bfloat16
# model_type              : qwen2
# quantization_config     : {'quant_method': 'awq', 'bits': 4, ...}

# 核对分片数量与索引文件是否一致（下载不全会导致加载失败）
python3 -c "
import json, os
p = '/data/models/Qwen2.5-14B-Instruct-AWQ'
idx = os.path.join(p, 'model.safetensors.index.json')
if os.path.exists(idx):
    d = json.load(open(idx))
    files = sorted(set(d['weight_map'].values()))
    print('索引中声明的分片数:', len(files))
    for f in files:
        full = os.path.join(p, f)
        print('  %-40s %s' % (f, 'OK' if os.path.exists(full) else '缺失'))
else:
    print('单文件模型，无需检查索引')
"

# 校验文件完整性（如果下载时提供了 checksum）
# sha256sum -c checksums.txt
```

**这里有个坑要提醒**：模型下载不完整是启动失败的头号原因。`config.json` 在、`tokenizer.json` 在、但某个 `model-00003-of-00007.safetensors` 缺失，加载时才会报错。评估阶段下载完一定要跑一次上面的索引检查。

## 七、配置示例

### 7.1 vLLM 启动配置（单卡场景）

目录约定：

```
/data/models/          模型仓库
/opt/llm/scripts/      启动脚本
/var/log/vllm/         日志
/data/bench/           压测结果
```

启动脚本 `/opt/llm/scripts/start-vllm-14b.sh`：

```bash
#!/usr/bin/env bash
# 大模型推理服务启动脚本 - Qwen2.5-14B INT8 单卡
# 用法: ./start-vllm-14b.sh [-n|--dry-run]
set -euo pipefail

# ---------- 可配置项，通过环境变量覆盖 ----------
MODEL_PATH="${MODEL_PATH:-/data/models/Qwen2.5-14B-Instruct-AWQ}"
SERVED_NAME="${SERVED_NAME:-qwen14b}"
LISTEN_HOST="${LISTEN_HOST:-0.0.0.0}"
LISTEN_PORT="${LISTEN_PORT:-8000}"
MAX_MODEL_LEN="${MAX_MODEL_LEN:-8192}"
MAX_NUM_SEQS="${MAX_NUM_SEQS:-32}"
GPU_MEM_UTIL="${GPU_MEM_UTIL:-0.90}"
TP_SIZE="${TP_SIZE:-1}"
QUANT="${QUANT:-awq}"
LOG_DIR="${LOG_DIR:-/var/log/vllm}"
LOG_FILE="${LOG_FILE:-${LOG_DIR}/qwen14b.log}"
PID_FILE="${PID_FILE:-/run/vllm-qwen14b.pid}"
# API Key 从环境变量读取，绝不写进脚本
API_KEY="${VLLM_API_KEY:-}"

DRY_RUN=0
for arg in "$@"; do
  case "$arg" in
    -n|--dry-run) DRY_RUN=1 ;;
    -h|--help)
      echo "用法: $0 [-n|--dry-run]"
      exit 0 ;;
    *)
      echo "未知参数: $arg" >&2
      exit 1 ;;
  esac
done

# ---------- 前置检查 ----------
check_prereq() {
  echo "[检查] 模型目录..."
  if [ ! -d "$MODEL_PATH" ]; then
    echo "  错误: 模型目录不存在: $MODEL_PATH" >&2
    return 1
  fi
  if [ ! -f "${MODEL_PATH}/config.json" ]; then
    echo "  错误: 缺少 config.json" >&2
    return 1
  fi
  echo "  OK"

  echo "[检查] GPU 可用性..."
  if ! command -v nvidia-smi >/dev/null 2>&1; then
    echo "  错误: 未找到 nvidia-smi" >&2
    return 1
  fi
  local gpu_count
  gpu_count=$(nvidia-smi -L 2>/dev/null | wc -l)
  echo "  检测到 GPU 数量: $gpu_count"
  if [ "$gpu_count" -lt "$TP_SIZE" ]; then
    echo "  错误: GPU 数量($gpu_count) < TP_SIZE($TP_SIZE)" >&2
    return 1
  fi

  echo "[检查] 端口占用..."
  if command -v ss >/dev/null 2>&1; then
    if ss -ltn "( sport = :${LISTEN_PORT} )" 2>/dev/null | grep -q ":${LISTEN_PORT}"; then
      echo "  错误: 端口 ${LISTEN_PORT} 已被占用" >&2
      return 1
    fi
  fi
  echo "  OK"

  echo "[检查] 日志目录..."
  if [ ! -d "$LOG_DIR" ]; then
    echo "  日志目录不存在，即将创建: $LOG_DIR"
    if [ "$DRY_RUN" -eq 0 ]; then
      mkdir -p "$LOG_DIR"
    fi
  fi
  echo "  OK"

  echo "[检查] 磁盘剩余空间..."
  local avail_gb
  avail_gb=$(df -BG "$MODEL_PATH" | awk 'NR==2 {gsub(/G/,"",$4); print $4}')
  echo "  模型所在分区剩余: ${avail_gb} GiB"
  if [ "${avail_gb:-0}" -lt 20 ]; then
    echo "  警告: 剩余空间不足 20 GiB，日志和 KV 换出可能失败" >&2
  fi

  return 0
}

# ---------- 组装启动命令 ----------
build_cmd() {
  CMD=(
    vllm serve "$MODEL_PATH"
    --served-model-name "$SERVED_NAME"
    --host "$LISTEN_HOST"
    --port "$LISTEN_PORT"
    --quantization "$QUANT"
    --dtype float16
    --max-model-len "$MAX_MODEL_LEN"
    --gpu-memory-utilization "$GPU_MEM_UTIL"
    --max-num-seqs "$MAX_NUM_SEQS"
    --enable-prefix-caching
    --enable-chunked-prefill
    --trust-remote-code
  )
  if [ "$TP_SIZE" -gt 1 ]; then
    CMD+=( --tensor-parallel-size "$TP_SIZE" )
  fi
  if [ -n "$API_KEY" ]; then
    # 注意：不同版本鉴权参数名可能不同，新版为 --api-key
    CMD+=( --api-key "$API_KEY" )
  fi
}

show_cmd() {
  printf '即将执行:'
  printf ' %q' "${CMD[@]}"
  printf '\n'
}

# ---------- 主流程 ----------
main() {
  check_prereq || exit 1
  build_cmd

  if [ "$DRY_RUN" -eq 1 ]; then
    echo "--- DRY RUN 模式，只打印不执行 ---"
    show_cmd
    exit 0
  fi

  if [ -f "$PID_FILE" ] && kill -0 "$(cat "$PID_FILE")" 2>/dev/null; then
    echo "错误: 服务已在运行，PID=$(cat "$PID_FILE")" >&2
    echo "如需重启，先执行: kill \$(cat $PID_FILE)" >&2
    exit 1
  fi

  show_cmd
  echo "日志输出: $LOG_FILE"

  # 前台启动并写日志；生产环境建议用 systemd 托管（见 7.2）
  nohup "${CMD[@]}" >> "$LOG_FILE" 2>&1 &
  echo $! > "$PID_FILE"
  echo "已启动，PID=$(cat "$PID_FILE")"
  echo "等待服务就绪..."

  local i
  for i in $(seq 1 120); do
    if curl -sf "http://127.0.0.1:${LISTEN_PORT}/health" >/dev/null 2>&1; then
      echo "服务就绪，耗时约 ${i} 秒"
      return 0
    fi
    sleep 1
  done

  echo "错误: 120 秒内服务未就绪，请查看 $LOG_FILE" >&2
  return 1
}

main "$@"
```

**关于密钥的处理**：脚本里 `VLLM_API_KEY` 从环境变量读，不写死在脚本里。生产环境下环境变量本身也可能被 `/proc/<pid>/environ` 读到，更稳妥的方式是用 systemd 的 `EnvironmentFile`，文件权限设为 `600` 且属主是服务账号，或者用 Vault、KMS 这类密钥管理系统注入。

**dry-run 用法**：

```bash
# 先演练，确认命令组装正确、检查项通过
bash /opt/llm/scripts/start-vllm-14b.sh --dry-run
```

预期输出：

```
[检查] 模型目录...
  OK
[检查] GPU 可用性...
  检测到 GPU 数量: 8
[检查] 端口占用...
  OK
[检查] 日志目录...
  OK
[检查] 磁盘剩余空间...
  模型所在分区剩余: 1250 GiB
--- DRY RUN 模式，只打印不执行 ---
即将执行: vllm serve /data/models/Qwen2.5-14B-Instruct-AWQ --served-model-name qwen14b --host 0.0.0.0 --port 8000 --quantization awq --dtype float16 --max-model-len 8192 --gpu-memory-utilization 0.90 --max-num-seqs 32 --enable-prefix-caching --enable-chunked-prefill --trust-remote-code
```

### 7.2 systemd 托管配置

生产环境不要用 `nohup`，用 systemd 托管，好处是能自动重启、能限资源、能统一管日志。

`/etc/systemd/system/vllm-qwen14b.service`：

```bash
[Unit]
Description=vLLM Inference Server (Qwen2.5-14B)
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=llm
Group=llm
WorkingDirectory=/opt/llm
# API Key 放在权限 600 的文件里，通过 EnvironmentFile 注入
EnvironmentFile=/etc/vllm/qwen14b.env
Environment=CUDA_VISIBLE_DEVICES=0
ExecStart=/usr/local/bin/vllm serve /data/models/Qwen2.5-14B-Instruct-AWQ \
    --served-model-name qwen14b \
    --host 0.0.0.0 \
    --port 8000 \
    --quantization awq \
    --dtype float16 \
    --max-model-len 8192 \
    --gpu-memory-utilization 0.90 \
    --max-num-seqs 32 \
    --enable-prefix-caching \
    --enable-chunked-prefill \
    --trust-remote-code \
    --api-key ${VLLM_API_KEY}

Restart=on-failure
RestartSec=10
# 启动超时给足，72B 模型加载可能要几分钟
TimeoutStartSec=600
# 停止时先发 SIGTERM，给引擎时间清理
KillSignal=SIGTERM
TimeoutStopSec=120

# 资源限制
LimitNOFILE=655360
LimitMEMLOCK=infinity

StandardOutput=append:/var/log/vllm/qwen14b.log
StandardError=append:/var/log/vllm/qwen14b.err

[Install]
WantedBy=multi-user.target
```

`/etc/vllm/qwen14b.env`（权限必须 600）：

```
VLLM_API_KEY=sk-替换为真实密钥
```

设置权限：

```bash
sudo install -m 600 -o root -g root /dev/null /etc/vllm/qwen14b.env
sudo tee /etc/vllm/qwen14b.env >/dev/null <<'EOF'
VLLM_API_KEY=sk-替换为真实密钥
EOF
sudo chmod 600 /etc/vllm/qwen14b.env
sudo chown root:root /etc/vllm/qwen14b.env
ls -l /etc/vllm/qwen14b.env
# 预期：-rw------- 1 root root ... /etc/vllm/qwen14b.env
```

**生效方式**：

```bash
sudo systemctl daemon-reload
sudo systemctl enable vllm-qwen14b
sudo systemctl start vllm-qwen14b
sudo systemctl status vllm-qwen14b --no-pager
```

**验证生效**：

```bash
# 服务状态是 active (running)
sudo systemctl is-active vllm-qwen14b
# 预期：active

# 端口在监听
ss -ltnp | grep 8000

# 探活
curl -sf http://127.0.0.1:8000/health && echo " OK"
```

### 7.3 Nginx 反向代理配置

生产环境不会让业务直接连推理引擎端口，中间要加网关做鉴权、限流、负载均衡、日志。

`/etc/nginx/conf.d/llm-gateway.conf`：

```bash
upstream llm_backend {
    # 多个推理实例时在这里列出；单实例时保留一行
    server 127.0.0.1:8000 max_fails=2 fail_timeout=10s;
    # server 127.0.0.1:8001 max_fails=2 fail_timeout=10s;

    # 长连接复用，降低握手开销
    keepalive 32;
    keepalive_requests 1000;
    keepalive_timeout 60s;
}

# 限流：按客户端 IP，每秒 5 个请求，突发允许 10 个
# 注意：LLM 请求耗时长，用 limit_req 主要防刷，不是防过载
limit_req_zone $binary_remote_addr zone=llm_req:10m rate=5r/s;

# 并发连接限制：单 IP 最多 10 个并发请求
limit_conn_zone $binary_remote_addr zone=llm_conn:10m;

server {
    listen 8443 ssl;
    server_name llm.internal.example.com;

    ssl_certificate     /etc/nginx/ssl/llm.crt;
    ssl_certificate_key /etc/nginx/ssl/llm.key;
    ssl_protocols       TLSv1.2 TLSv1.3;
    ssl_ciphers         HIGH:!aNULL:!MD5;
    ssl_session_cache   shared:SSL:10m;
    ssl_session_timeout 10m;

    # 大模型请求体可能很大（长 prompt 场景）
    client_max_body_size 20m;
    client_body_timeout  120s;

    # 关键：LLM 请求耗时长，超时必须放宽
    # 注意 proxy_read_timeout 是"两次读之间"的间隔，不是总时长
    proxy_connect_timeout 10s;
    proxy_send_timeout    600s;
    proxy_read_timeout    600s;
    # HTTP 1.1 长连接必须显式指定
    proxy_http_version 1.1;
    proxy_set_header Connection "";
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;

    # 流式输出必须关闭缓冲，否则用户要等全部生成完才看到内容
    # 这两个是流式 LLM 服务最容易踩的坑
    proxy_buffering off;
    proxy_request_buffering off;
    # 如果用了 gzip，对 SSE 流式响应要关掉压缩
    gzip off;

    access_log /var/log/nginx/llm_access.log main;
    error_log  /var/log/nginx/llm_error.log warn;

    location /v1/ {
        limit_req zone=llm_req burst=10 nodelay;
        limit_conn llm_conn 10;

        # 去掉客户端直接带过来的鉴权头，统一由网关鉴权
        proxy_set_header Authorization $http_authorization;

        proxy_pass http://llm_backend;
    }

    location /health {
        # 健康检查不鉴权、不限流
        access_log off;
        proxy_pass http://llm_backend;
    }

    # 指标端口不对业务网开放，只允许监控服务器
    location /metrics {
        allow 10.0.0.0/8;
        deny all;
        proxy_pass http://llm_backend;
    }
}
```

**生效方式**：

```bash
# 先检查语法，语法错误时 reload 会失败但旧配置继续生效，不会中断服务
sudo nginx -t
# 预期：nginx: configuration file /etc/nginx/nginx.conf test is successful

# 平滑重载，不断连接
sudo systemctl reload nginx
# 或者：sudo nginx -s reload
```

**验证生效**：

```bash
# 走网关调用，确认能通
curl -s -o /dev/null -w "%{http_code}\n" \
  -k https://llm.internal.example.com/health
# 预期：200

# 确认流式输出没有被缓冲（这是最容易出问题的地方）
curl -N -s -k https://llm.internal.example.com/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"qwen14b","messages":[{"role":"user","content":"数到20"}],"stream":true,"max_tokens":64}' \
  | head -5
# 预期：能立刻看到逐块输出的 data: 行，而不是等几秒后一次性刷出来
# 如果卡住不动然后一次性输出，说明 proxy_buffering 没关掉

# 验证限流生效
for i in $(seq 1 20); do
  curl -s -o /dev/null -w "%{http_code} " -k \
    https://llm.internal.example.com/v1/chat/completions \
    -H "Content-Type: application/json" \
    -d '{"model":"qwen14b","messages":[{"role":"user","content":"hi"}],"max_tokens":4}'
done; echo
# 预期：出现 503（限流拒绝），说明 limit_req 生效
```

### 7.4 模型显存测算配置

评估阶段需要的模型参数表，存成 JSON 便于脚本读取：

`/opt/llm/conf/models.json`：

```json
{
  "qwen2.5-7b-instruct": {
    "layers": 28,
    "hidden_size": 3584,
    "num_attention_heads": 28,
    "num_key_value_heads": 4,
    "head_dim": 128,
    "vocab_size": 152064,
    "max_position_embeddings": 32768,
    "params_billion": 7.6,
    "note": "GQA，KV 头较少，长上下文友好"
  },
  "qwen2.5-14b-instruct": {
    "layers": 48,
    "hidden_size": 5120,
    "num_attention_heads": 40,
    "num_key_value_heads": 8,
    "head_dim": 128,
    "vocab_size": 152064,
    "max_position_embeddings": 32768,
    "params_billion": 14.7
  },
  "qwen2.5-32b-instruct": {
    "layers": 64,
    "hidden_size": 5120,
    "num_attention_heads": 40,
    "num_key_value_heads": 8,
    "head_dim": 128,
    "vocab_size": 152064,
    "max_position_embeddings": 32768,
    "params_billion": 32.5
  },
  "qwen2.5-72b-instruct": {
    "layers": 80,
    "hidden_size": 8192,
    "num_attention_heads": 64,
    "num_key_value_heads": 8,
    "head_dim": 128,
    "vocab_size": 152064,
    "max_position_embeddings": 32768,
    "params_billion": 72.7
  },
  "llama-3.1-8b-instruct": {
    "layers": 32,
    "hidden_size": 4096,
    "num_attention_heads": 32,
    "num_key_value_heads": 8,
    "head_dim": 128,
    "vocab_size": 128256,
    "max_position_embeddings": 131072,
    "params_billion": 8.0
  },
  "llama-3.1-70b-instruct": {
    "layers": 80,
    "hidden_size": 8192,
    "num_attention_heads": 64,
    "num_key_value_heads": 8,
    "head_dim": 128,
    "vocab_size": 128256,
    "max_position_embeddings": 131072,
    "params_billion": 70.6
  }
}
```

**重要提醒**：上面的值是常见公开配置，**实际使用前必须用 6.5 节的命令从下载到的模型的 `config.json` 里读出来核对**。同一系列不同版本（比如 Qwen2.5 和 Qwen2，或者 instruct 和 base）字段值可能不同。写死的参数表只能用于初步估算，不能作为采购依据。

## 八、日志与指标观察方法

### 8.1 启动日志怎么读

启动日志是判断配置合不合理的第一手材料。关注这几行：

```bash
# 查看启动日志的关键行
grep -E "Initializing|memory|blocks|concurrency|Maximum|Error|error|OOM|out of memory" \
  /var/log/vllm/qwen14b.log | tail -40
```

**关键行 1：显存分配明细**

```
INFO 01-15 10:23:41 model_runner.py:1051] Loading model weights took 15.02 GB
INFO 01-15 10:23:45 worker.py:219] Memory profiling with 1.0 GiB per GPU
INFO 01-15 10:23:47 worker.py:280] Available KV cache memory: 24.51 GiB
```

怎么读：

- `Loading model weights took` 这一行是权重占用的实测值。**和 Step 3 手算的对比**，如果差超过 20%，说明精度判断或参数假设有问题
- `Available KV cache memory` 是引擎刨除权重、激活、框架开销后剩下的 KV Cache 空间。这个数字直接决定能支持多少并发

**关键行 2：KV Cache 块数和并发上限**

```
INFO 01-15 10:23:48 gpu_worker.py:139] # GPU blocks: 4685, # CPU blocks: 3276
INFO 01-15 10:23:48 gpu_worker.py:146] Maximum concurrency for 8192 tokens per request: 36.60x
```

怎么读：

- `# GPU blocks` 是 KV Cache 被切成多少块（vLLM 默认每块 16 个 token）
- `Maximum concurrency for 8192 tokens per request` 是在 8192 长度下理论上能并行的请求数
- **这个数字和 Step 3 手算结果对比**。手算 15，引擎报告 36，说明手算的并发约束过保守（可能 `--max-num-seqs` 限制了实际值）；手算 20，引擎报告 8，说明手算的 overhead 或 activation 假设太乐观

**关键行 3：异常信号**

```bash
# 查启动失败原因
grep -iE "error|traceback|failed|not found|out of memory|CUDA" \
  /var/log/vllm/qwen14b.log | tail -30
```

常见启动失败及对应处理：

| 报错关键词 | 原因 | 处理 |
| --- | --- | --- |
| `CUDA out of memory` | 显存不够 | 降 `--gpu-memory-utilization`、降 `--max-model-len`、降 `--max-num-seqs`，或降量化精度 |
| `No such file or directory` + safetensors | 模型文件缺失 | 按 6.5 节检查索引文件声明的分片是否齐全 |
| `unsupported quantization` | 量化方法与模型不匹配 | 检查 `config.json` 里的 `quantization_config.quant_method`，把 `--quantization` 改成对应值 |
| `maximum context length` | 请求超过 `--max-model-len` | 调大 `--max-model-len`（注意显存），或者在前端截断输入 |
| `tokenizer` 相关报错 | tokenizer 文件缺失或不兼容 | 确认 `tokenizer.json`、`tokenizer_config.json` 都在 |
| `NCCL` 相关报错 + 多卡 | 通信库问题 | 检查 `nvidia-smi topo -m`，确认 `NCCL_P2P_DISABLE` 等环境变量是否需要调整 |
| `Address already in use` | 端口被占 | `ss -ltnp |

### 8.2 运行日志怎么读

运行期日志主要用于发现慢请求、异常请求。

```bash
# 实时跟踪
tail -f /var/log/vllm/qwen14b.log

# 找出慢请求（日志里带耗时信息时）
grep -E "prompt throughput|generation throughput" /var/log/vllm/qwen14b.log | tail -20
# 输出形如：
#   Avg prompt throughput: 4210.3 tokens/s, Avg generation throughput: 852.1 tokens/s,
#   Running: 15 reqs, Swapped: 0 reqs, Pending: 0 reqs, GPU KV cache usage: 78.2%,
#   CPU KV cache usage: 0.0%
```

**这一行信息量很大，要会读**：

| 字段 | 含义 | 正常范围 | 异常表现与含义 |
| --- | --- | --- | --- |
| Avg prompt throughput | 输入处理速度 token/s | 数千量级 | 骤降说明 prefill 遇到瓶颈 |
| Avg generation throughput | 输出生成速度 token/s | 数百到数千 | 骤降说明显存带宽不够或有争抢 |
| Running | 正在处理的请求数 | 接近但不超过 `--max-num-seqs` | 长期等于上限说明该调大 |
| Swapped | 被换出到 CPU 的请求数 | 0 | 非 0 说明显存不够，KV 被换出，延迟会飙升 |
| Pending | 排队等待的请求数 | 0 或很小 | 持续 >0 说明过载，需要扩容或限流 |
| GPU KV cache usage | KV Cache 使用率 | 60%-90% | >95% 说明快满；<30% 说明浪费 |
| CPU KV cache usage | CPU 侧 KV 使用率 | 0 | 非 0 通常与 Swapped 同时出现 |

**Swapped 非 0 是最需要警惕的信号**。它意味着显存装不下所有活跃请求的 KV，引擎把一部分挪到了 CPU 内存。每次生成 token 都要从 CPU 把 KV 读回 GPU，延迟会从几十毫秒级涨到秒级。看到这个，说明并发上限到了，要么降并发要么加卡。

### 8.3 Prometheus 指标采集

要在评估期间持续采集指标，需要接入 Prometheus。**下面所有指标名以实际 exporter 输出为准**，不同 vLLM 版本命名有变化，接入前先 `curl /metrics | grep vllm` 核对一遍。

Prometheus 配置 `prometheus.yml` 片段：

```yaml
scrape_configs:
  - job_name: 'vllm'
    scrape_interval: 15s
    scrape_timeout: 10s
    metrics_path: /metrics
    static_configs:
      - targets:
          - '10.0.1.21:8000'
        labels:
          env: 'prod'
          service: 'llm-inference'
          model: 'qwen14b'

  - job_name: 'gpu-dcgm'
    scrape_interval: 15s
    static_configs:
      - targets:
          - '10.0.1.21:9400'
        labels:
          env: 'prod'
          service: 'llm-inference'
```

GPU 指标需要单独部署 DCGM Exporter（NVIDIA 官方容器），它暴露 `DCGM_FI_DEV_*` 系列指标。

**评估期必须记录的核心指标**：

| 指标 | 类型 | 用途 |
| --- | --- | --- |
| `vllm:num_requests_running` | Gauge | 实际并发，和压测设定的并发对照 |
| `vllm:num_requests_waiting` | Gauge | 排队深度，直接反映过载程度 |
| `vllm:gpu_cache_usage_perc` | Gauge | KV Cache 水位，判断显存配置是否合理 |
| `vllm:time_to_first_token_seconds_bucket` | Histogram | 算 TTFT 分位数 |
| `vllm:time_per_output_token_seconds_bucket` | Histogram | 算 TPOT 分位数 |
| `vllm:e2e_request_latency_seconds_bucket` | Histogram | 算端到端分位数 |
| `vllm:request_success_total` | Counter | 成功率 |
| `vllm:prompt_tokens_total` | Counter | 累计输入量，用于成本核算 |
| `vllm:generation_tokens_total` | Counter | 累计输出量，用于成本核算 |
| `DCGM_FI_DEV_GPU_UTIL` | Gauge | GPU 计算利用率 |
| `DCGM_FI_DEV_MEM_COPY_UTIL` | Gauge | 显存带宽利用率 |
| `DCGM_FI_DEV_FB_USED` | Gauge | 已用显存 MiB |
| `DCGM_FI_DEV_FB_FREE` | Gauge | 剩余显存 MiB |
| `DCGM_FI_DEV_GPU_TEMP` | Gauge | 温度 |
| `DCGM_FI_DEV_POWER_USAGE` | Gauge | 功耗 W |
| `DCGM_FI_DEV_XID_ERRORS` | Gauge | XID 错误，非 0 要立刻查 |

**怎么用这些指标做判断**：

**判断一：配置是否合理。** 跑一轮目标并发的压测，看 `gpu_cache_usage_perc` 的稳定值：

- 稳定在 70%-90%：配置合理
- 稳定在 95% 以上且 `num_requests_waiting` 持续 > 0：显存配小了，或者并发设大了
- 稳定在 30% 以下：显存配大了，可以调高 `--max-num-seqs` 提升吞吐，或者换更小的卡省钱

**判断二：瓶颈在 prefill 还是 decode。**

- `DCGM_FI_DEV_MEM_COPY_UTIL` 高（>70%）而 `DCGM_FI_DEV_GPU_UTIL` 不高：decode 阶段显存带宽瓶颈，这是正常的大模型推理特征
- 两个都高：prefill 和 decode 混合负载，算力瓶颈
- `DCGM_FI_DEV_GPU_UTIL` 低（<30%）而延迟很高：可能是 `num_requests_waiting` 高导致排队，或者长 prompt 阻塞了短请求，考虑开 chunked prefill

**判断三：是否有硬件问题。**

- `DCGM_FI_DEV_XID_ERRORS` 非 0：查 NVIDIA XID 错误码表对应原因，常见的是 XID 13/31/43/48 等，多数需要报修
- `DCGM_FI_DEV_GPU_TEMP` 长期 > 85℃：散热不足，会触发降频，影响性能
- `DCGM_FI_DEV_POWER_USAGE` 明显低于 TDP 且利用率低：可能被功耗墙限制，或者卡没有真正在跑

### 8.4 关键查询（PromQL）

```bash
# 当前并发请求数
vllm:num_requests_running

# 排队请求数（这个值持续 > 0 就要考虑扩容）
vllm:num_requests_waiting

# KV Cache 使用率（百分比）
vllm:gpu_cache_usage_perc * 100

# TTFT P50 / P95 / P99（用 histogram_quantile 算）
histogram_quantile(0.50, sum by (le) (rate(vllm:time_to_first_token_seconds_bucket[5m])))
histogram_quantile(0.95, sum by (le) (rate(vllm:time_to_first_token_seconds_bucket[5m])))
histogram_quantile(0.99, sum by (le) (rate(vllm:time_to_first_token_seconds_bucket[5m])))

# TPOT P95（毫秒）
histogram_quantile(0.95, sum by (le) (rate(vllm:time_per_output_token_seconds_bucket[5m]))) * 1000

# 输出 token 生成速率（token/s）
rate(vllm:generation_tokens_total[5m])

# 请求成功率（5 分钟窗口）
rate(vllm:request_success_total[5m])

# GPU 显存使用率
DCGM_FI_DEV_FB_USED / (DCGM_FI_DEV_FB_USED + DCGM_FI_DEV_FB_FREE) * 100

# GPU 温度
DCGM_FI_DEV_GPU_TEMP

# 30 天累计输出 token（用于成本核算）
increase(vllm:generation_tokens_total[30d])
```

**告警规则示例**（阈值必须结合业务基线调，下面是起点值，不是标准值）：

```bash
groups:
  - name: llm-inference
    rules:
      - alert: LLMHighQueueDepth
        # 排队请求持续超过 5 个，说明服务过载
        expr: vllm:num_requests_waiting > 5
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "推理服务排队积压"
          description: "{{ $labels.instance }} 排队请求数 {{ $value }}，持续 5 分钟"

      - alert: LLMKVCacheSaturated
        # KV Cache 使用率超过 95%
        expr: vllm:gpu_cache_usage_perc > 0.95
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "KV Cache 接近饱和"

      - alert: LLMTTFTSlow
        # TTFT P95 超过 5 秒
        expr: >
          histogram_quantile(0.95,
            sum by (le) (rate(vllm:time_to_first_token_seconds_bucket[5m]))) > 5
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "首 token 延迟偏高"

      - alert: LLMServiceDown
        expr: up{job="vllm"} == 0
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "推理服务不可达"

      - alert: GPUHighTemperature
        expr: DCGM_FI_DEV_GPU_TEMP > 85
        for: 15m
        labels:
          severity: warning
        annotations:
          summary: "GPU 温度过高，可能触发降频"

      - alert: GPUXidError
        expr: increase(DCGM_FI_DEV_XID_ERRORS[10m]) > 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "GPU 出现 XID 错误，需立即排查"
```

**告警阈值设定提醒**：上面的数值是根据经验给的建议起点。真实阈值必须结合三项调整：一是业务可接受的延迟基线（不同业务对 TTFT 的容忍度差好几倍）；二是当前硬件的实际能力（48GB 卡和 80GB 卡的 KV 水位特性不同）；三是流量峰谷规律（业务高峰期排队多属于正常，半夜排队多就是异常）。**建议做法是：先按建议值配上去观察两周，把实际分布画出来，再根据 P95 和 P99 的位置调整阈值，让告警既不太吵也不太钝。**

## 九、排查路径

评估和部署阶段会遇到的问题，按下面的路径走。

### 9.1 服务起不来

```
服务起不来
  │
  ├─ 1. 看进程状态
  │     systemctl status vllm-qwen14b --no-pager
  │     ├─ 状态是 activating 卡住 → 模型还在加载，大模型几分钟正常，看日志有无进展
  │     ├─ 状态是 failed → 看日志最后 50 行
  │     └─ 状态是 inactive 且立刻退出 → 启动命令有问题，dry-run 验证命令
  │
  ├─ 2. 看日志关键错误
  │     grep -iE "error|traceback|not found|out of memory|no module" \
  │       /var/log/vllm/qwen14b.log | tail -30
  │     │
  │     ├─ ImportError / ModuleNotFoundError → Python 环境问题，确认 venv 和依赖
  │     ├─ CUDA out of memory →
  │     │     ├─ 确认没有别的进程占卡：nvidia-smi --query-compute-apps
  │     │     ├─ 降 --gpu-memory-utilization 到 0.85
  │     │     ├─ 降 --max-model-len
  │     │     └─ 还不够就降精度或换更大的卡
  │     ├─ 文件不存在 → 按 6.5 节核对模型文件完整性
  │     └─ 端口被占 → ss -ltnp | grep 8000 找到占用者处理
  │
  ├─ 3. 确认 GPU 可见
  │     ├─ nvidia-smi 能出结果？不能 → 驱动或容器 GPU 透传问题
  │     ├─ systemd 服务里 CUDA_VISIBLE_DEVICES 设对了吗
  │     └─ 容器场景：--gpus all 或 NVIDIA_VISIBLE_DEVICES 设了吗
  │
  └─ 4. 确认模型文件
        检查 config.json / tokenizer.json / 所有 safetensors 分片
```

### 9.2 能启动但请求失败

```
请求返回错误
  │
  ├─ HTTP 400 Bad Request
  │     ├─ 请求体 JSON 格式错 → 用 jq 校验一下请求体
  │     ├─ model 字段和 --served-model-name 不一致 → 调 /v1/models 看实际名称
  │     └─ max_tokens 超限 → 检查是否超 --max-model-len 减去输入长度
  │
  ├─ HTTP 404 Not Found
  │     └─ 路径错了。OpenAI 兼容接口路径是 /v1/chat/completions
  │        注意有些引擎的探活路径不是 /health，查对应版本文档
  │
  ├─ HTTP 401 / 403
  │     └─ 鉴权问题。检查 --api-key 是否设了，请求头 Authorization: Bearer xxx
  │        走 Nginx 时确认有没有把 Authorization 头传下去
  │
  ├─ HTTP 500 Internal Server Error
  │     ├─ 看服务日志的 traceback
  │     ├─ 常见：显存不足触发 OOM → 降并发或降 max-model-len
  │     └─ 常见：prompt 里有特殊字符导致 tokenizer 异常 → 复现并上报
  │
  └─ HTTP 503 Service Unavailable
        ├─ 走 Nginx → 大概率是 limit_req / limit_conn 触发，看 nginx error log
        └─ 直连引擎 → 引擎过载保护，看 num_requests_waiting
```

### 9.3 能返回但很慢

这是评估阶段最常见的问题，也是最需要系统排查的。

```
响应慢
  │
  ├─ 第一步：区分是 TTFT 慢还是 TPOT 慢（这两个原因完全不同）
  │     用 6.3 节的流式脚本分别测
  │     │
  │     ├─ TTFT 慢（首字要等很久），TPOT 正常
  │     │     │
  │     │     ├─ 原因 A：输入太长
  │     │     │    检查 usage.prompt_tokens，如果远大于预期
  │     │     │    → 检查 prompt 模板是不是拼接了过多内容
  │     │     │    → RAG 场景检查检索返回的文档数量
  │     │     │
  │     │     ├─ 原因 B：排队
  │     │     │    看 vllm:num_requests_waiting，如果 > 0
  │     │     │    → 并发超过承载能力，降并发或加资源
  │     │     │
  │     │     ├─ 原因 C：长请求阻塞短请求
  │     │     │    如果开了 chunked prefill 还是这样，检查是否真生效
  │     │     │    → 确认启动参数里有 --enable-chunked-prefill
  │     │     │
  │     │     └─ 原因 D：前缀缓存没命中
  │     │          检查 vllm:prefix_cache_hits_total 是否增长
  │     │          RAG 场景下把 system prompt 和检索内容顺序固定，能提升命中率
  │     │
  │     └─ TPOT 慢（生成很慢），TTFT 正常
  │           │
  │           ├─ 原因 A：批太大
  │           │    批大小越大，每个请求分到的算力越少
  │           │    → 减少并发能提升单请求速度（吞吐和延迟的权衡）
  │           │
  │           ├─ 原因 B：显存带宽瓶颈
  │           │    看 DCGM_FI_DEV_MEM_COPY_UTIL，如果 > 80%
  │           │    → 这是 decode 阶段的固有瓶颈，只能换带宽更高的卡
  │           │
  │           ├─ 原因 C：KV Cache 被换出到 CPU
  │           │    看日志里 Swapped 是不是非 0
  │           │    → 降并发、降 max-model-len，或加卡
  │           │
  │           ├─ 原因 D：GPU 降频
  │           │    看 nvidia-smi 的 clocks.sm 和 temperature.gpu
  │           │    → 温度 > 85℃ 或撞功耗墙，改善散热或放宽功耗限制
  │           │
  │           └─ 原因 E：张量并行通信开销
  │                多卡时看 nvidia-smi topo -m，检查卡间是不是 SYS
  │                → 如果有 SYS，调整卡的位置或减少 TP 度
  │
  ├─ 第二步：确认不是网络和网关的问题
  │     在服务器本机 curl 直连引擎测一次
  │     再通过 Nginx 测一次
  │     对比两次的耗时差，差值就是网络+网关的开销
  │     如果差很多，检查 proxy_buffering 是否关闭（流式场景这是必查项）
  │
  └─ 第三步：确认真实序列长度分布
        统计 request 的输入输出 token 分布
        如果 P95 是平均值的 5 倍以上，说明负载极不均匀
        → 用 P95 而不是平均值做容量规划
```

### 9.4 结果质量异常

```
模型输出质量问题
  │
  ├─ 1. 排除量化因素
  │     同一个问题，跑 FP16 版本和量化版本对比
  │     如果 FP16 好很多 → 量化损失超出预期，考虑升精度
  │     如果两版都差 → 不是量化问题，继续下面
  │
  ├─ 2. 检查 chat template
  │     这是最容易被忽略的问题
  │     用 tokenizer 的 apply_chat_template 生成 prompt，人工看一眼格式对不对
  │     各模型模板差异很大，套错了效果会明显下降
  │     检查 tokenizer_config.json 里的 chat_template 字段
  │
  ├─ 3. 检查 stop token
  │     如果输出停不下来或者过早截断
  │     → 检查 generation_config.json 里的 eos_token_id
  │     → 确认 --stop 参数没有误设
  │
  ├─ 4. 检查采样参数
  │     temperature 太高导致胡说，太低导致重复
  │     问答类建议 temperature 0.1-0.3，创意类 0.7-1.0
  │     top_p 一般 0.8-0.95
  │
  ├─ 5. 检查上下文截断
  │     如果输入超了 --max-model-len，有的配置会静默截断
  │     截断掉的内容可能正是关键信息
  │     → 看日志有没有截断警告，确认 usage.prompt_tokens 是否符合预期
  │
  └─ 6. 最后才考虑模型本身能力不足
        用同样的 prompt 在官方 API 或在线 Demo 上试
        如果官方也答不好 → 是模型能力边界，考虑换更大的模型
        如果官方答得好而本地答不好 → 回到第 2 步查模板
```

### 9.5 显存相关专项排查

```bash
# 场景：启动成功但跑一段时间 OOM

# 第一步：确认显存是被谁占了
nvidia-smi --query-compute-apps=pid,process_name,used_memory --format=csv

# 第二步：看显存增长曲线（判断是泄漏还是正常的 KV 增长）
nvidia-smi --query-gpu=timestamp,memory.used --format=csv,noheader,nounits -l 10 \
  | tee /tmp/memwatch.txt
# 判断：
#   启动后快速到平台期然后稳定 → 正常
#   持续缓慢上涨不收敛 → 疑似泄漏，需要上报

# 第三步：确认 KV Cache 水位
curl -s http://127.0.0.1:8000/metrics | grep -E "gpu_cache_usage|num_requests"

# 第四步：如果 KV 水位不高但显存高，检查激活
# 用长 prompt 测，看 prefill 时显存峰值
# 如果 prefill 峰值很高，降低单次请求的最大输入长度

# 第五步：兜底手段
#   a. 降 --gpu-memory-utilization 到 0.85（给激活留更多空间）
#   b. 降 --max-model-len
#   c. 降 --max-num-seqs
#   d. 开 KV Cache 量化（如果引擎和硬件支持）
```

## 十、风险提醒

这一节把评估和部署过程中的高风险点集中列出，每个都说明影响范围、执行前检查和应对方式。

### 10.1 需要显式确认的高风险操作

**风险 1：修改生产推理服务的启动参数**

```bash
# 危险动作示例
sudo systemctl restart vllm-qwen14b
# 或者改了 /etc/systemd/system/vllm-qwen14b.service 后 reload + restart
```

- **影响范围**：重启期间所有推理请求失败，服务不可用。大模型加载慢，72B 模型从重启到可用可能要 3-10 分钟
- **执行前检查**：

1. 确认无正在进行的批量任务（查 `vllm:num_requests_running`）
2. 通知使用方，确认可以接受停机窗口
3. 备份当前配置文件：`sudo cp -a /etc/systemd/system/vllm-qwen14b.service /etc/vllm/backup/$(date +%F-%H%M)-vllm.service`
4. 确认新参数的显存需求已通过 dry-run 或独立设备验证

- **替代方案**：多实例部署时做滚动重启，先停一个、起来验证、再停下一个，业务不中断
- **回滚方式**：恢复备份的配置文件，`daemon-reload` + `restart`

**风险 2：修改内核参数**

```bash
# 危险动作示例（大模型服务常见需要调优的内核参数）
sudo sysctl -w net.core.somaxconn=65535
sudo sysctl -w net.ipv4.tcp_max_syn_backlog=65535
sudo sysctl -w vm.swappiness=10
```

- **影响范围**：内核参数是全机生效的，如果机器上还跑着其他服务，可能影响它们
- **执行前检查**：

1. 记录当前值：`sysctl net.core.somaxconn` 等，或 `sysctl -a > /tmp/sysctl.before`
2. 确认改动只影响目标项
3. 评估是否影响同机上其他服务

- **风险点**：`vm.swappiness` 调低会让系统更倾向不用 swap，内存紧张时可能触发 OOM Killer 杀掉推理进程。大模型服务本身内存占用高，这个参数要谨慎
- **替代方案**：优先用 systemd 的资源限制，而不是全局内核参数
- **回滚方式**：`sudo sysctl -w key=old_value`，或者从备份恢复

**风险 3：清理日志**

```bash
# 危险动作示例
sudo find /var/log/vllm -name "*.log" -mtime +30 -delete
# 或者
sudo truncate -s 0 /var/log/vllm/qwen14b.log
```

- **影响范围**：日志是排查问题的唯一证据。删了之后历史问题无法追溯
- **执行前检查**：

1. 确认日志已经归档到其他地方（或者确实不需要了）
2. 确认没有正在写日志的进程会被影响（`truncate` 对已打开的文件句柄是安全的，`rm` 后进程会继续写但空间不释放）
3. 用 `du -sh` 确认清理量，避免误算

- **替代方案**：用 logrotate 做自动轮转，保留 N 天，而不是手工删
- **特别注意**：`truncate` 正在写入的日志文件是安全的（inode 不变），但 `rm` 掉正在写的文件，进程会继续持有句柄，磁盘空间不会释放，`df` 看不到但 `du` 能看到，容易误判

**风险 4：删除或用新模型覆盖旧模型文件**

```bash
# 危险动作示例
rm -rf /data/models/Qwen2.5-14B-old
# 或者下载新版本覆盖同名目录
```

- **影响范围**：模型文件动辄几十 GB，重新下载耗时长（取决于带宽），期间服务无法重建
- **执行前检查**：

1. 确认该模型没有任何服务在用：`grep MODEL_PATH /etc/systemd/system/*.service`
2. 确认有备份或者可以重新下载（记录模型来源和版本）
3. 用 `du -sh` 确认删除量

- **替代方案**：重命名保留一段时间，而不是直接删；用软链接指向当前版本，切换时只改软链接
- **回滚方式**：如果只是改了软链接，改回来即可

**风险 5：批量操作多台 GPU 服务器**

```bash
# 危险动作示例
for h in gpu01 gpu02 gpu03 gpu04; do
  ssh "$h" "sudo systemctl restart vllm-qwen14b"
done
```

- **影响范围**：如果这些节点都在提供同一服务，全部重启会导致服务完全不可用
- **执行前检查**：

1. 先在一台机器上验证（灰度）
2. 加 dry-run 开关打印将要执行的命令
3. 确认每台机器的机器名没写错
4. 通知相关人员

- **必须遵守**：批量脚本要有二次确认、要有执行日志、要支持逐台粒度
- **回滚方式**：准备好反向脚本，但重启类操作通常只能重新配置再重启

**风险 6：修改数据库或中间件参数（如果推理服务依赖）**

推理服务通常依赖 Redis（缓存）、PostgreSQL/MySQL（业务数据）、向量库（RAG）。修改这些组件的参数属于独立的高风险操作，需要单独评估。**不要在一次变更里同时改推理服务和数据库，出问题无法定位是哪个改动导致的。**

### 10.2 评估阶段的方法论风险

**风险 7：用随机数据集的压测结果做容量规划**

随机 token 和真实文本的 token 分布完全不同。random 数据集生成的是均匀随机 token，真实文本有大量重复和规律，影响：

- 前缀缓存的命中率（random 数据几乎不命中，真实 RAG 场景命中率很高）
- 实际吞吐（缓存命中能大幅提升）
- 实际显存（KV Cache 因为共享而占用更少）

**结论**：random 压测可以摸性能上限，但**容量规划必须用真实数据集的压测结果**。如果只能用 random，规划时要在结果上打折（经验上打 6-7 折），并在报告里明确标注这个折扣的依据。

**风险 8：忽略 P95 只算平均值**

前面反复强调过。用平均值算出来的并发上限，在真实负载下会导致 5% 的请求严重超时。**规划必须按 P95，告警必须按 P95/P99。**

**风险 9：把理论上限当成可承诺值**

Step 3 算出来的并发上限、引擎报告的 `Maximum concurrency`，都是特定条件下的理论上限。实际要留出余量（建议按理论值的 70%-80% 做承诺），因为：

- 真实序列长度分布不均
- 显存有碎片
- 引擎需要一些余量应对突发
- 上游网关和网络有额外开销

**风险 10：不区分"峰值"和"常态"**

按峰值配置硬件，会让平时资源大量闲置，单位成本飙升。按常态配置，峰值时会降级。**正确做法是分层**：常态用 70% 的容量，峰值时通过限流+排队+降级（比如降低 max\_tokens 上限、关闭非核心功能）来承受，把峰值当作可容忍的降级而不是必须无损承载。这个策略要在评估报告里写清楚，让业务方认可。

### 10.3 合规与安全风险

**风险 11：模型和数据合规**

- 模型许可证：商用前确认模型 License 允许商用（有些模型不允许，有些有额外限制）
- 训练数据合规：确认模型训练数据来源合规，避免版权和隐私风险
- 数据不出域：私有化的核心价值。但要注意，如果配置了外部遥测上报（部分工具默认开启），数据可能仍会外传，必须显式关闭
- 输出内容审计：面向员工的服务通常需要保留审计日志

**审计日志的留存**：如果业务要求可追溯，需要把请求和响应都记录。但**注意不要记录敏感信息**。常见做法是记录请求 ID、时间、token 数、模型名、状态码，不记录 prompt 和 completion 原文；如果必须记录原文，要加密存储并控制访问权限。

**风险 12：API Key 管理**

```bash
# 反面示例：密钥写在脚本里
# VLLM_API_KEY=sk-abc123...  # 错误：会被 git 记录、会被 cat 看到

# 反面示例：密钥作为命令行参数
# vllm serve ... --api-key sk-abc123...
# 错误：ps aux 和 /proc/<pid>/cmdline 都能看到

# 正确做法
# 1. 放在权限 600 的文件里，通过 EnvironmentFile 注入
# 2. 或者用密钥管理系统（Vault / KMS / 云厂商密钥服务）
# 3. 服务账号只给读权限，不给写
```

**风险 13：网络暴露**

推理服务不要直接暴露到公网。哪怕是内网，也要：

- 绑定内网 IP，不要 `0.0.0.0`（除非前面有网关）
- 网关做鉴权
- 防火墙/安全组限制来源网段
- 指标端口只对监控网开放

```bash
# 检查是否有意外的公网监听
ss -ltnp | grep -vE "127.0.0.1|::1"
# 逐条确认每个监听端口的来源限制

# 检查防火墙规则
sudo iptables -L -n --line-numbers 2>/dev/null | head -40
# 或
sudo firewall-cmd --list-all 2>/dev/null
```

## 十一、验证方式

评估结论和部署结果都必须验证。验证要分层，从功能到性能到稳定性。

### 11.1 功能验证清单

```bash
# --- 1. 服务可达性 ---
curl -sf http://127.0.0.1:8000/health && echo "PASS: 探活" || echo "FAIL: 探活"

# --- 2. 模型列表正确 ---
curl -s http://127.0.0.1:8000/v1/models \
  | python3 -c "
import sys, json
d = json.load(sys.stdin)
ids = [m['id'] for m in d.get('data', [])]
print('PASS: 模型列表', ids) if 'qwen14b' in ids else print('FAIL: 模型名不符', ids)
"

# --- 3. 基本推理可用 ---
resp=$(curl -s http://127.0.0.1:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"qwen14b","messages":[{"role":"user","content":"1+1等于几？只回答数字"}],"max_tokens":16,"temperature":0}')
echo "$resp" | python3 -c "
import sys, json
d = json.load(sys.stdin)
c = d['choices'][0]['message']['content']
print('PASS: 推理返回 =', repr(c)) if c.strip() else print('FAIL: 内容为空')
"

# --- 4. 流式可用 ---
timeout 30 curl -N -s http://127.0.0.1:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"qwen14b","messages":[{"role":"user","content":"数到5"}],"stream":true,"max_tokens":32}' \
  | grep -c "^data:" | xargs -I{} echo "PASS: 流式返回 {} 个数据块"

# --- 5. 长上下文可用（关键，容易出问题）---
# 构造一段长文本，确认不报错且能正常回答
python3 - <<'PY'
import json, urllib.request
long_text = "这是一段测试文本。" * 500   # 约 3000+ 字
payload = {
    "model": "qwen14b",
    "messages": [{"role": "user", "content": long_text + "\n\n上面这段话重复了多少次？"}],
    "max_tokens": 64,
}
req = urllib.request.Request(
    "http://127.0.0.1:8000/v1/chat/completions",
    data=json.dumps(payload).encode(),
    headers={"Content-Type": "application/json"},
)
d = json.load(urllib.request.urlopen(req, timeout=120))
print("PASS: 长上下文 prompt_tokens =", d["usage"]["prompt_tokens"])
print("输出:", d["choices"][0]["message"]["content"][:100])
PY

# --- 6. 超长上下文应被正确拒绝（不能静默截断）---
python3 - <<'PY'
import json, urllib.request, urllib.error
huge = "测试" * 20000   # 远超 max-model-len
payload = {
    "model": "qwen14b",
    "messages": [{"role": "user", "content": huge}],
    "max_tokens": 16,
}
req = urllib.request.Request(
    "http://127.0.0.1:8000/v1/chat/completions",
    data=json.dumps(payload).encode(),
    headers={"Content-Type": "application/json"},
)
try:
    d = json.load(urllib.request.urlopen(req, timeout=120))
    print("注意: 未报错，返回了结果，确认是否发生了静默截断:", d["usage"])
except urllib.error.HTTPError as e:
    print("PASS: 正确拒绝超长请求, HTTP", e.code)
PY

# --- 7. 错误模型名应被拒绝 ---
curl -s -o /dev/null -w "错误模型名返回码: %{http_code}\n" \
  http://127.0.0.1:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"not-exist","messages":[{"role":"user","content":"hi"}],"max_tokens":4}'
# 预期：400 或 404

# --- 8. 鉴权验证（如果启用了）---
curl -s -o /dev/null -w "无 Key 返回码: %{http_code}\n" \
  http://127.0.0.1:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"qwen14b","messages":[{"role":"user","content":"hi"}],"max_tokens":4}'
# 预期：401
```

### 11.2 性能验证

```bash
# 在评估报告的推荐并发下跑一轮，产出基准数据
vllm bench serve \
  --backend openai-chat \
  --model qwen14b \
  --base-url http://127.0.0.1:8000 \
  --dataset-name random \
  --random-input-len 2000 \
  --random-output-len 300 \
  --num-prompts 300 \
  --max-concurrency 15 \
  --save-result --result-dir /data/bench/verify-c15
```

**验收判据表**（按评估阶段和业务方约定的基线填）：

| 指标 | 目标值 | 实测值 | 判定 |
| --- | --- | --- | --- |
| TTFT P95 | < 2.0 s |  |  |
| TPOT P95 | < 80 ms |  |  |
| 输出吞吐 | ≥ 800 token/s |  |  |
| 请求成功率 | 100% |  |  |
| GPU KV 水位 | < 90% |  |  |
| 队列深度 | 0 |  |  |

**判定逻辑**：任何一项不达标，都要先定位原因再决定是调参还是调整验收目标。**不要把"调低验收标准"当作解决问题的方式**，那只是把问题推到上线后。

### 11.3 稳定性验证

```bash
# 持续 30 分钟的中等负载，看有没有延迟漂移和内存泄漏
vllm bench serve \
  --backend openai-chat \
  --model qwen14b \
  --base-url http://127.0.0.1:8000 \
  --dataset-name random \
  --random-input-len 2000 \
  --random-output-len 300 \
  --num-prompts 5400 \
  --max-concurrency 20 \
  --request-rate 3 \
  --save-result --result-dir /data/bench/verify-stability &

BENCH_PID=$!

# 同期采样 GPU 和显存
nvidia-smi --query-gpu=timestamp,utilization.gpu,memory.used,power.draw,temperature.gpu \
  --format=csv,noheader,nounits -l 5 > /data/bench/verify-gpu.csv &
SMI_PID=$!

wait $BENCH_PID
kill $SMI_PID 2>/dev/null || true

# 分析显存趋势：前后半段的均值应该接近
python3 - <<'PY'
import csv

rows = []
with open("/data/bench/verify-gpu.csv") as f:
    for r in csv.reader(f):
        if len(r) < 5:
            continue
        try:
            rows.append(float(r[2]))   # memory.used
        except ValueError:
            continue

if not rows:
    print("没有采到数据")
else:
    n = len(rows)
    head = sum(rows[:n // 4]) / (n // 4)
    tail = sum(rows[-(n // 4):]) / (n // 4)
    print("前半段显存均值: %.0f MiB" % head)
    print("后半段显存均值: %.0f MiB" % tail)
    growth = (tail - head) / head * 100
    print("增长: %.2f%%" % growth)
    if growth > 10:
        print("告警: 显存增长超过 10%%，疑似泄漏，需要进一步排查")
    else:
        print("通过: 显存稳定")
PY
```

**通过判据**：

- 显存前后半段差异 < 10%
- 延迟数据没有随时间明显上升的趋势
- 请求成功率 100%
- 期间没有 `Swapped` 非 0

### 11.4 配置生效验证

```bash
# --- 验证 systemd 配置生效 ---
# 看实际使用的环境变量和启动命令
systemctl show vllm-qwen14b -p Environment -p ExecStart | tr ' ' '\n' | grep -v '^$'
# 对比期望值

# --- 验证 Nginx 配置生效 ---
sudo nginx -T 2>/dev/null | grep -A 20 "location /v1/"
# 确认渲染后的配置符合预期

# 验证流式不被缓冲（最关键的验证项）
curl -N -s -k https://llm.internal.example.com/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"qwen14b","messages":[{"role":"user","content":"从1数到30，每个数字单独一行"}],"stream":true,"max_tokens":200}' \
  | while IFS= read -r line; do
      printf '%s %s\n' "$(date +%H:%M:%S.%3N)" "$line"
      case "$line" in *"[DONE]"*) break ;; esac
    done | head -8
# 判据：每行的时间戳应该是递增的、有时间间隔的
#      如果前几行时间戳完全相同，说明被缓冲了

# --- 验证限流生效 ---
for i in $(seq 1 15); do
  code=$(curl -s -o /dev/null -w "%{http_code}" -k \
    https://llm.internal.example.com/v1/chat/completions \
    -H "Content-Type: application/json" \
    -d '{"model":"qwen14b","messages":[{"role":"user","content":"hi"}],"max_tokens":4}')
  printf "%s " "$code"
done; echo
# 判据：应该出现 503，说明 limit_req 生效

# --- 验证监控采集生效 ---
# 在 Prometheus 里查
# up{job="vllm"} 应该为 1
# vllm:generation_tokens_total 应该持续增长
```

## 十二、回滚方案

回滚方案必须在变更**之前**准备好，不能出问题再想。这一节给出各类变更的回滚方法。

### 12.1 配置变更回滚

**变更前必做**：

```bash
# 建立统一备份目录
BACKUP_DIR="/etc/vllm/backup/$(date +%Y%m%d-%H%M%S)"
sudo mkdir -p "$BACKUP_DIR"
sudo cp -a /etc/systemd/system/vllm-qwen14b.service "$BACKUP_DIR/"
sudo cp -a /etc/vllm/qwen14b.env "$BACKUP_DIR/"
sudo cp -a /etc/nginx/conf.d/llm-gateway.conf "$BACKUP_DIR/"
sudo bash -c "systemctl show vllm-qwen14b > '$BACKUP_DIR/vllm-service-state.txt'"
sudo bash -c "sysctl -a > '$BACKUP_DIR/sysctl.before.txt' 2>/dev/null"
echo "备份到: $BACKUP_DIR"
ls -la "$BACKUP_DIR"
```

**回滚操作**：

```bash
# 回滚推理服务配置
BACKUP_DIR="/etc/vllm/backup/20260115-100000"   # 换成实际目录
sudo cp -a "$BACKUP_DIR/vllm-qwen14b.service" /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl restart vllm-qwen14b
sudo systemctl status vllm-qwen14b --no-pager

# 回滚 Nginx 配置
sudo cp -a "$BACKUP_DIR/llm-gateway.conf" /etc/nginx/conf.d/
sudo nginx -t && sudo systemctl reload nginx

# 回滚内核参数（逐项恢复）
sudo sysctl -w net.core.somaxconn=4096
sudo sysctl -w vm.swappiness=60
```

**回滚耗时预期**：

| 变更类型 | 回滚耗时 | 主要耗时在哪 |
| --- | --- | --- |
| Nginx 配置 | 秒级 | reload 不断连接 |
| 推理服务小参数 | 3-10 分钟 | 模型重新加载 |
| 推理服务换模型 | 10-30 分钟 | 大模型加载 + 权重从磁盘读 |
| 内核参数 | 秒级 | 立即生效 |
| 换卡/换机 | 小时级 | 硬件更换 |

**关键结论**：模型相关的变更回滚慢，所以**新模型必须先在同型号的测试机上验证通过再上生产**。

### 12.2 模型版本回滚

用软链接管理模型版本，切换和回滚都是一条命令：

```bash
# 目录结构
# /data/models/
#   ├── Qwen2.5-14B-Instruct-AWQ-v1/    实际目录
#   ├── Qwen2.5-14B-Instruct-AWQ-v2/    实际目录
#   └── current -> Qwen2.5-14B-Instruct-AWQ-v1    软链接

# 服务配置里指向软链接，而不是具体版本目录
# MODEL_PATH=/data/models/current

# 切换版本
sudo -u llm ln -sfn /data/models/Qwen2.5-14B-Instruct-AWQ-v2 /data/models/current.tmp
# 用 ln -sfn 到临时名再 mv，保证原子性
sudo mv -Tf /data/models/current.tmp /data/models/current
ls -l /data/models/current

# 重启服务加载新版本
sudo systemctl restart vllm-qwen14b

# 回滚只需把软链接指回去
sudo -u llm ln -sfn /data/models/Qwen2.5-14B-Instruct-AWQ-v1 /data/models/current.tmp
sudo mv -Tf /data/models/current.tmp /data/models/current
sudo systemctl restart vllm-qwen14b
```

**用 `mv -Tf` 而不是直接 `ln -sfn` 覆盖**：直接覆盖软链接在某些情况下不是原子的，服务可能短暂看到不存在或指向错误的链接。`ln -sfn` 到临时名再 `mv -Tf` 覆盖是原子的，这是运维常用技巧。

**注意**：模型目录只增不减。回滚需要旧版本还在，所以清理旧模型要等新版本稳定运行至少一个完整的业务周期（建议 2-4 周）之后再做。清理前务必确认没有服务引用。

### 12.3 服务实例回滚（多实例场景）

```bash
# 假设有两个实例：8000 和 8001，通过 Nginx upstream 做负载均衡
# 变更时逐个操作，保证至少一个实例在线

# 检查当前两个实例状态
for p in 8000 8001; do
  printf "port %s: " "$p"
  curl -sf "http://127.0.0.1:${p}/health" >/dev/null && echo "UP" || echo "DOWN"
done

# 1. 从 Nginx upstream 摘除 8001（编辑 upstream，注释掉该行）
# 2. reload nginx
sudo nginx -t && sudo systemctl reload nginx
# 3. 确认流量已不流向 8001
sleep 5
curl -s http://127.0.0.1:8001/metrics | grep -E "^vllm:num_requests_running"
# 预期：接近 0
# 4. 变更 8001
sudo systemctl restart vllm-8001
# 5. 验证 8001
curl -sf http://127.0.0.1:8001/health && echo "8001 OK"
# 6. 挂回 upstream，reload nginx
# 7. 对 8000 重复 1-6

# 回滚：把两个实例都切回旧配置，同样逐台做
```

**关键点**：摘除实例前必须确认该实例上**没有正在处理的请求**。硬切会导致进行中的流式响应中断，用户看到截断。等待时间建议 10-30 秒，看 `num_requests_running` 归零。

### 12.4 数据回滚

推理服务本身通常无状态，但相关数据需要备份：

| 数据类型 | 备份方式 | 恢复方式 |
| --- | --- | --- |
| 评估数据集 | 版本控制或对象存储 | 直接取回 |
| 压测结果 | `--save-result` 输出到独立目录 | 直接取回 |
| 审计日志 | 按天归档，保留期按合规要求 | 从归档取回 |
| 配置 | 变更前备份到带时间戳目录 | 复制回原位 |
| 模型文件 | 保留旧版本目录，不删 | 切换软链接 |
| 向量库数据（RAG 场景） | 向量库自身的备份机制 | 按向量库文档恢复 |

**向量库数据要特别提醒**：如果推理服务配合 RAG，向量库是有状态的。重建向量库需要重新做 embedding，如果文档量大（几十万条），重建可能要几小时到几天。**向量库必须纳入备份范围，并且定期验证备份可恢复。** 只备份不验证等于没备份。

### 12.5 回滚决策树

```
变更后发现异常
  │
  ├─ 问题严重（服务不可用 / 结果明显错误）
  │     └─ 立即回滚，不要尝试现场修复
  │        回滚步骤：恢复配置 → 重启 → 验证 → 通知
  │
  ├─ 问题中等（部分请求慢 / 部分功能不好用）
  │     ├─ 影响面可控？→ 可以先观察，同时准备回滚
  │     └─ 影响面在扩大？→ 回滚
  │
  └─ 问题轻微（指标好看了一点点 / 影响很小）
        └─ 不回滚，但要记录，下次变更时一起处理

判断"是否回滚"的红线：
  - 服务不可用超过 2 分钟
  - 错误率超过 1%
  - 数据出现不一致
  - 有数据丢失风险
  触及任何一条，立即回滚，不要犹豫
```

## 十三、生产环境注意事项

### 13.1 容量规划

**不要按峰值配硬件。** 前面算过，按峰值配置会让单位成本飙升。推荐的分层策略：

| 层级 | 覆盖比例 | 配置方式 | 超限时的处理 |
| --- | --- | --- | --- |
| 常态容量 | 覆盖日常 70% 的需求 | 按 P50 业务量配置 | — |
| 峰值容量 | 覆盖 25% 的峰值 | 通过提高批大小利用剩余显存 | 队列+延迟上升 |
| 溢出容量 | 覆盖 5% 的极端峰值 | 限流 + 排队 + 降级 | 拒绝或降级响应 |

**降级手段**（按优先级）：

1. 收紧单请求 `max_tokens` 上限
2. 关闭非核心功能（比如自动摘要、延伸提问）
3. 缩短 `--max-model-len`，拒绝超长请求
4. 对非关键业务降级到小模型
5. 最后才拒绝请求

**评估报告里要把这套降级策略写出来**，并和业务方确认可接受。

### 13.2 变更管理

- **变更窗口**：模型相关的变更（换模型、改量化）建议放在低峰期，且要预留足够的回滚时间。改一个参数到验证完成，至少要 30 分钟
- **灰度**：多实例时逐台变更；单实例时先在测试机上验证
- **变更记录**：记录变更内容、时间、操作人、验证结果、是否回滚
- **一次只改一件事**：不要在一次变更里同时改模型、改推理引擎版本、改 Nginx 配置。出问题无法定位

### 13.3 监控告警

- **必须监控**：服务存活、TTFT/TPOT 分位数、KV 水位、队列深度、GPU 温度、GPU 显存、XID 错误
- **告警分级**：服务不可用（Critical，立即处理）> 队列积压/延迟超线（Warning，15 分钟内响应）> KV 水位高/温度高（Info，当天处理）
- **告警要有可操作性**：每条告警的注释里写清楚"看到这条告警先做什么"。比如"KV 水位 > 95%“的告警，注释里应该写"检查 num\_requests\_running 和 num\_requests\_waiting，如果并发正常则考虑扩容或降低 max-model-len”
- **避免告警风暴**：队列积压通常会同时触发延迟告警和 KV 告警，用 Alertmanager 的 inhibition 做抑制，只发最上游的那条

### 13.4 日常维护

```bash
# 每日检查清单（建议做成脚本定时跑）
cat > /opt/llm/scripts/daily-check.sh <<'SCRIPT'
#!/usr/bin/env bash
# 大模型推理服务日常巡检
set -uo pipefail

ENDPOINT="${ENDPOINT:-http://127.0.0.1:8000}"
REPORT="/var/log/llm-check/$(date +%F).log"
mkdir -p "$(dirname "$REPORT")"

{
  echo "=== 巡检时间: $(date '+%F %T') ==="

  echo "--- 1. 服务状态 ---"
  systemctl is-active vllm-qwen14b 2>/dev/null || echo "不是 systemd 管理"

  echo "--- 2. 探活 ---"
  code=$(curl -s -o /dev/null -w "%{http_code}" --max-time 10 "${ENDPOINT}/health" || echo "000")
  echo "HTTP ${code}"

  echo "--- 3. 关键指标 ---"
  curl -s --max-time 10 "${ENDPOINT}/metrics" 2>/dev/null | grep -E \
    "^vllm:(num_requests_running|num_requests_waiting|gpu_cache_usage_perc|num_requests_swapped)" \
    || echo "指标获取失败"

  echo "--- 4. GPU 状态 ---"
  nvidia-smi --query-gpu=index,name,utilization.gpu,memory.used,memory.total,temperature.gpu,power.draw \
    --format=csv,noheader 2>/dev/null || echo "nvidia-smi 失败"

  echo "--- 5. GPU 进程 ---"
  nvidia-smi --query-compute-apps=pid,process_name,used_memory --format=csv,noheader 2>/dev/null

  echo "--- 6. 磁盘空间 ---"
  df -h /data /var/log 2>/dev/null | grep -vE "^Filesystem" || true

  echo "--- 7. 日志错误统计（近 24h）---"
  if [ -f /var/log/vllm/qwen14b.log ]; then
    # 注意：大日志文件用 tail 限制范围，不要直接 grep 整个文件
    tail -n 20000 /var/log/vllm/qwen14b.log \
      | grep -ciE "error|exception|traceback" \
      | xargs -I{} echo "错误行数: {}"
  fi

  echo "=== 巡检结束 ==="
  echo
} 2>&1 | tee -a "$REPORT"
SCRIPT
chmod +x /opt/llm/scripts/daily-check.sh

# 用 cron 每天 8 点跑
# 0 8 * * * /opt/llm/scripts/daily-check.sh >/dev/null 2>&1
```

**关于日志检查的注意点**：`grep` 整个大日志文件会很慢，也很吃资源。用 `tail -n 20000` 限制范围是常见做法。更好的做法是配合 logrotate 做轮转，检查时只看当天的文件。

**logrotate 配置**`/etc/logrotate.d/vllm`：

```
/var/log/vllm/*.log {
    daily
    rotate 30
    missingok
    notifempty
    compress
    delaycompress
    copytruncate
    create 0644 llm llm
}
```

`copytruncate` 的作用：先复制再清空原文件，避免进程持有旧句柄导致新日志写到旧文件。代价是复制瞬间的日志可能丢失，对大模型服务这种日志量不大的场景完全可以接受。如果日志量大到不能接受丢几行，则改用 `create` 配合服务 reload。

### 13.5 安全加固

```bash
# --- 服务账号最小权限 ---
# 不要用 root 跑推理服务
sudo useradd -r -s /sbin/nologin -d /opt/llm llm 2>/dev/null || true
sudo chown -R llm:llm /opt/llm /var/log/vllm
# 模型目录只给读权限
sudo chmod -R o+rX /data/models
sudo chmod 750 /data/models

# --- 文件权限检查 ---
# 确认没有全局可写的配置文件
find /etc/vllm -type f -perm -o+w -ls 2>/dev/null
# 预期：无输出

# 确认密钥文件权限正确
stat -c "%a %U:%G %n" /etc/vllm/*.env

# --- 监听检查 ---
# 确认没有意外监听在公网
ss -ltnp | awk 'NR==1 || $4 !~ /^(127\.|\[::1\])/'

# --- 防火墙 ---
# 只放通业务网段
sudo iptables -A INPUT -p tcp --dport 8000 -s 10.0.0.0/8 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 8000 -j DROP
# 注意：iptables 规则变更会立即生效且不持久化，重启后丢失
# 生产环境要用 firewalld 或保存规则到配置文件
# 修改前先确认不会把自己锁在外面（保留 SSH 端口和已有会话）

# --- 关闭不必要的外部上报 ---
# 部分工具默认开启遥测，数据合规要求下必须关闭
# 具体方式因工具而异，查对应文档
```

**修改防火墙的风险提醒**：修改防火墙规则是高风险操作，错误的规则可能立即切断所有连接，包括你的 SSH 会话。执行前必须：

1. 保留 SSH 端口（通常是 22）的放通规则，且要放在 DROP 规则之前
2. 用 `iptables -L -n --line-numbers` 先确认现有规则
3. 用 `iptables-save > /tmp/iptables.before` 备份
4. 如果是远程操作，用 `at` 或 `sleep && 恢复` 做自动兜底：

   ```bash
   # 危险操作兜底：5 分钟后自动恢复原规则，如果确认没问题就取消这个任务
   echo "iptables-restore < /tmp/iptables.before" | at now + 5 minutes
   # 确认操作成功后再删除这个 at 任务
   atq
   atrm <job_id>
   ```
5. 或者用 `firewall-cmd --permanent` + `--reload`，reload 失败时旧规则仍然生效，比直接改 iptables 安全

### 13.6 成本持续优化

评估不是一次性的，上线后要持续跟踪实际成本，并和评估预测对比。

```bash
# 从 Prometheus 拉实际 token 量（30 天）
curl -s -G 'http://prometheus:9090/api/v1/query' \
  --data-urlencode 'query=increase(vllm:generation_tokens_total[30d])' \
  | python3 -c "
import sys, json
d = json.load(sys.stdin)
for r in d['data']['result']:
    inst = r['metric'].get('instance', '?')
    val = float(r['value'][1])
    print('%-24s 30天输出 token: %.2f 亿' % (inst, val / 1e8))
"
```

**对比清单**（每月做一次）：

| 项目 | 评估预测 | 实际 | 偏差 | 分析 |
| --- | --- | --- | --- | --- |
| 日均输出 token |  |  |  |  |
| 平均并发 |  |  |  |  |
| TTFT P95 |  |  |  |  |
| GPU 利用率 |  |  |  |  |
| 单位 token 成本 |  |  |  |  |

**偏差大时要做的事**：

- 实际用量远低于预测：说明业务没铺开，需要推动内部推广，否则成本摊不薄
- 实际用量远高于预测：说明需求旺盛，考虑扩容，并重新评估是否需要更大规模的部署
- 显存利用率长期低于 50%：说明配置过保守，可以降卡或者提升并发设置
- GPU 利用率长期低于 30%：说明服务闲置，考虑合并到更少的机器上，或者承接更多业务

**能降本的具体动作**（按投入产出排序）：

1. **把离线批处理放到夜间**：不改硬件，纯调度优化，效果最直接
2. **调整 `--max-num-seqs` 提升吞吐**：如果 KV 水位长期偏低，调大能提升吞吐，摊薄成本
3. **开前缀缓存**：RAG 场景能大幅降低 prefill 开销
4. **缩短实际使用的上下文**：如果业务实际只用 4K，把 `--max-model-len` 从 32K 降到 8K，能显著提升可承载并发
5. **合并小模型服务**：多个小模型服务合并到一张卡上（用多实例或者 MIG）
6. **换更合适的卡**：如果一直跑不满 80GB 卡的显存，换 48GB 卡能省钱

### 13.7 常见误区

**误区一：参数越大效果越好。** 7B 微调 + 好的 RAG，效果经常超过裸跑的 72B。先优化数据和 prompt，再考虑加参数。

**误区二：量化一定不行。** INT8 的质量损失在多数任务上不可感知。INT4 需要验证但不等于不能用。先测再否。

**误区三：显存够就能跑。** 显存只是一条线。带宽、互联、CPU、内存、网络都可能成为瓶颈。8 卡装下了模型但跑得比 2 卡还慢的情况确实存在，原因就是通信。

**误区四：压测跑过就行。** 压测环境的数据长度分布、并发模式必须和真实业务接近，否则压测结果没有参考价值。

**误区五：私有化一定比云便宜。** 前面算过，用量的盈亏平衡点通常是年几百亿 token。低于这个量，公有云更省。私有化的价值主要在合规和数据控制。

**误区六：上线就完事了。** 模型迭代快，半年可能就有更适合的新模型。要预留 15%-20% 的精力做模型更新和调优。

## 十四、总结

这篇把私有化大模型部署评估拆成了可计算的步骤，核心是四笔账。

**第一笔账：显存。** 公式是 `总显存 = 权重 + KV Cache + 激活 + 框架开销 + 通信缓冲 + 余量`。权重刚性，KV Cache 弹性。最容易算错的是 KV Cache，因为它和 `num_key_value_heads`、`head_dim`、上下文长度、并发数四个量都相关。GQA 模型下 `num_kv_heads × head_dim` 不等于 `hidden_size`，用错了会差好几倍。32K 上下文下 72B 模型单序列 KV 就要 10GB，这是长上下文场景的最大约束。

**第二笔账：并发。** 理论上限由显存决定，实际上限由压测决定，可用上限还要再打折。单序列 decode 速度的理论上限是 `显存带宽 / 权重字节数`，这个公式能快速判断一个配置是否合理。延迟和吞吐是矛盾的，必须先明确业务要哪个。

**第三笔账：成本。** 算单位 token 成本，不要算总投入。年度总成本 = 折旧 + 电费 + 人力 + 网络 + 软件 + 备件。关键是 `utilization` 这一项——利用率 50% 和 80% 的单位成本差 1.6 倍。盈亏平衡点通常是年几百亿 token，低于这个量公有云更经济。

**第四笔账：风险。** 理论上限不能当承诺值（打 7-8 折）；random 数据集压测不能直接用于容量规划（打 6-7 折）；不用平均值规划（用 P95）；不按峰值配硬件（分层+降级）；回滚方案必须在变更前准备好。

**评估的六步流程**：需求盘点 → 模型选型 → 显存测算 → 部署压测 → 成本核算 → 出报告。其中**第三步的手算和第五步的实测必须对照**，差异超过 20% 就说明假设有问题，要回头修正。引擎启动日志里的 `Available KV cache memory` 和 `Maximum concurrency` 是最直接的对照数据，不要浪费。

**最该记住的三条**：

1. **先压测再决定，不要先决定再压测。** 评估的大部分工作量应该花在实测上，理论测算只是用来缩小试验范围和解释结果。
2. **把假设写进报告并让业务方确认。** 并发数、长度、延迟基线这些参数，如果业务方给不出，就用假设值并明确标注。上线后按实测数据修正，把修正过程留档。
3. **回滚比上线重要。** 模型加载慢，回滚成本高，所以每次变更前必须先确认回滚路径可行、回滚时间在可接受范围内。

评估做完，产出应该是一份能签字的报告，里面每一页的数字都能追溯到具体来源：模型参数来自 `config.json`，显存数字来自测算脚本和引擎日志，性能数字来自压测记录，成本数字来自采购报价和实际账单。**能被追溯的结论才是能交付的结论。**

# 01 · GPU 与计算环境

> **本篇目标：**理解 GPU 计算资源从物理硬件到容器应用的完整路径，以及 AI 工作负载为什么受算力、显存和通信共同约束。

**范围：**通用 NVIDIA/Linux 环境；具体 GPU 型号、驱动和 CUDA 组合需查对应官方兼容矩阵。本篇属于技术教学，不等同于安装验收记录。

## 1. GPU 在 AI Infra 中的位置

CPU 擅长复杂控制流和通用任务，GPU 将大量运算单元组织起来执行高并行度计算。在大模型场景中，GPU 承担矩阵运算、注意力计算等主要负载；CPU 仍负责请求接入、分词、调度、数据准备、I/O 和系统管理。

一条最简化的运行链路是：

```text
推理/训练应用（PyTorch、vLLM 等）
      ↓ 调用框架与算子
CUDA Runtime / CUDA Libraries
      ↓ 通过 NVIDIA 驱动访问设备
GPU（计算单元 + 显存 + 片上缓存）
      ↕ PCIe / NVLink / 网络（视设备与拓扑而定）
CPU / 系统内存 / 其他 GPU / 存储
```

请区分：**GPU 能被系统识别**、**容器能访问 GPU**、**框架能执行 CUDA 算子**、**模型能稳定承载工作负载**，是四层不同的成功条件。

## 2. GPU 的主要资源

| 资源 | 作用 | 常见约束 | 观察入口 |
| --- | --- | --- | --- |
| 计算单元（SM、Tensor Core） | 执行矩阵等并行计算 | 计算吞吐、精度、占用率 | GPU 利用率、时钟、实际任务吞吐 |
| 显存（VRAM/HBM/GDDR） | 存放权重、KV Cache、激活和工作缓冲 | 容量、显存带宽 | `nvidia-smi`、框架内存统计 |
| GPU ↔ CPU 通路 | 模型加载、Host/Device 数据传输 | PCIe 代际、通道宽度、NUMA | `nvidia-smi topo -m`、`lspci` |
| GPU ↔ GPU 通路 | Tensor Parallel、跨卡通信 | 是否有 NVLink、跨 NUMA、通信方式 | `nvidia-smi topo -m`、NCCL 日志 |
| 存储 / 网络 | 加载模型、数据集、Checkpoint | 吞吐和尾延迟 | `iostat`、网络统计、应用日志 |

**利用率不等于吞吐。** GPU 利用率高只能表示设备在采样区间内较忙，不能直接证明请求完成得快；显存占用高也不等于显存带宽已成为瓶颈。

## 3. 计算受限与访存受限

- **Compute-bound（计算受限）**：主要时间花在算术运算，常见于较大批次的矩阵乘等计算。
- **Memory-bound（访存受限）**：主要时间花在读取权重、KV Cache 或搬运数据，低批次的自回归 Decode 往往更容易受此限制。
- **Communication-bound（通信受限）**：多卡计算需要跨卡同步或交换中间结果，拓扑与通信开销可能抵消并行收益。

这三种瓶颈有时同时存在。识别瓶颈需要工作负载基线，不能仅凭某一张监控图推断。

## 4. Driver、CUDA Toolkit、Runtime 与容器的区别

| 组件 | 职责 | 常见误解 |
| --- | --- | --- |
| NVIDIA Driver | 让操作系统和应用能使用 GPU 设备 | `nvidia-smi` 显示的 CUDA Version 不是系统已安装 Toolkit 的版本 |
| CUDA Toolkit | 编译器、开发工具、库等 | 使用预编译镜像时，宿主机不一定需要完整 Toolkit |
| CUDA Runtime / Libraries | 应用执行 GPU 计算所需运行库 | 运行时和驱动仍需满足兼容要求 |
| NVIDIA Container Toolkit | 把 GPU 设备和相关运行时能力交给容器 | 安装 Docker 本身不会自动提供 GPU 访问 |
| PyTorch / vLLM | 提供张量运算和推理引擎 | 框架版本与驱动支持的 CUDA 能力需要共同核对 |

**兼容性检查的原则**：从目标框架/镜像及 GPU 型号出发，反查所需驱动下限和支持矩阵；不要为了与镜像“数字一致”随意更换宿主机 CUDA 组件。

## 5. 必要的观测命令

以下是只读检查示例，不代表验收结果：

```bash
nvidia-smi
nvidia-smi -L
nvidia-smi --query-gpu=index,name,memory.total,memory.used,utilization.gpu,temperature.gpu,power.draw --format=csv
nvidia-smi topo -m
nvidia-smi --query-compute-apps=pid,process_name,used_memory --format=csv
```

判断顺序建议是：设备是否存在 → 驱动是否正常 → 显存是否被占用 → 时钟与温度是否异常 → 多卡拓扑是否符合预期 → 框架内是否可见。

观察结果需要结合上下文。例如 `nvidia-smi` 显示空闲，只能说明该采样时刻的设备状态，无法证明 Kubernetes 的调度资源账本也空闲。

## 6. 显存预算的基本模型

推理服务显存大致可拆为：

```text
总需求 ≈ 模型权重 + KV Cache + 激活/临时缓冲 + CUDA/框架开销 + 通信缓冲
```

仅估算**模型权重理论大小**时，可使用：

```text
权重字节数 ≈ 参数个数 × 每参数字节数
BF16/FP16 ≈ 2 字节/参数；FP8 ≈ 1 字节/参数
```

这是粗略估算，不包括量化元数据、KV Cache、加载时临时峰值等。不能据此直接断言某张卡一定可以完整承载模型。

## 7. 易混淆的问题

1. **“装了 CUDA 就能使用 GPU”**：仍需驱动、设备可见性及运行库兼容。
2. **“单机有四张卡，多卡性能就会线性增长”**：跨卡通信、算子并行方式和内存访问会影响加速比。
3. **“GPU 利用率高说明服务健康”**：队列、TTFT、TPOT 和失败率可能已经恶化。
4. **“所有 NVIDIA GPU 都支持 MIG/NVLink”**：能力依型号不同，应查询官方产品规格。

## 8. 自测问题

- 为什么宿主机能执行 `nvidia-smi`，容器中仍可能看不到 GPU？
- 为什么相同模型在两种 GPU 上，显存足够但推理吞吐差异很大？
- 为什么启用 Tensor Parallel 后，必须检查 GPU 拓扑？
- 为什么判断“模型能否加载”与“模型可承载多少并发”是两个问题？


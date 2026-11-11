# Lab 08 · 单 GPU LoRA 微调与 Checkpoint 恢复入门

> **实验目标：**使用一个公开的小参数量模型和**人工构造的教学数据**跑通 SFT + LoRA 的最小训练过程，观察显存、Loss、Checkpoint，并验证训练可以从上次的 Checkpoint 继续。

## 1. 为什么本实验独立于 vLLM

vLLM 是**推理引擎**；LoRA/SFT 训练通过 PyTorch、Transformers、TRL、PEFT 等工具完成。推理服务中的 GPU Request Batching 与训练中的梯度累积不是一个概念。本实验不依赖生产数据，也不需要 Kubernetes 或大规模分布式训练。

**资料范围：**此前提供的文章主要覆盖 GPU 调度、推理与服务，训练部分细节较少；本篇最小步骤借鉴 Hugging Face 官方 TRL/PEFT/Transformers 文档，**不把用户上传文章中没有的训练结论归因于原文**。

## 2. 前置条件和版本记录

- 一台 Linux + NVIDIA GPU 主机，已完成 Lab 01；建议独占 GPU、至少 12–16 GiB 显存起步（实际能否跑通取决于型号、CUDA、PyTorch 版本和配置）。
- Python 3.10+；可以访问 Hugging Face 的公开模型，或本地已缓存相应文件。
- 模型：`Qwen/Qwen2.5-0.5B-Instruct`。训练用例纯教学，不讨论效果提升和业务泛化能力。
- TRL 示例按 `trl==0.26.2` 的 `SFTTrainer`/`SFTConfig` 设计；其他依赖需要根据环境选择兼容版本并通过 `pip freeze` 固定。
- 训练实验之前先停止 Lab 02 的 vLLM 容器，避免与训练进程争抢 GPU。

```bash
nvidia-smi
python3 -m venv .venv-train
. .venv-train/bin/activate
python -m pip install --upgrade pip
```

先按 [PyTorch 官方安装选择器](https://pytorch.org/get-started/locally/) 安装与主机驱动兼容的 GPU PyTorch（**不要直接复制别人的 CUDA wheel index URL**）。随后：

```bash
python -m pip install 'trl==0.26.2' 'transformers>=4.57,<5' 'peft>=0.17,<1' 'datasets>=3,<5' 'accelerate>=1,<2'
python -m pip check
python -c 'import torch, trl, transformers, peft; print("CUDA:",torch.cuda.is_available()); print("torch",torch.__version__,"trl",trl.__version__,"transformers",transformers.__version__,"peft",peft.__version__)'
python -m pip freeze > requirements-resolved.txt
```

如果 `torch.cuda.is_available()` 为 False，先回到 Lab 01 排查驱动/运行时；不要仅靠 `pip install cuda` 修复。

## 3. 准备最小教学数据

本实验不读取企业文件、工单、真实用户对话或未授权数据。数据通过 Python 在内存中构造，采用 `prompt` + `completion` 形式；训练过程的 Loss 不等于“模型已经理解了这些概念”。

将以下保存为 `train_lora.py`：

```python
import argparse
import os
from pathlib import Path

import torch
from datasets import Dataset
from peft import LoraConfig
from transformers.trainer_utils import get_last_checkpoint
from trl import SFTConfig, SFTTrainer

parser = argparse.ArgumentParser()
parser.add_argument("--max_steps", type=int, default=4)
parser.add_argument("--resume", action="store_true")
args = parser.parse_args()

model_id = "Qwen/Qwen2.5-0.5B-Instruct"
output_dir = "./outputs/ai-infra-lora-lab"

# 完全人工构造的简短示例；只为验证训练流程
examples = [
    {"prompt": "问题：CPU 的主要作用是什么？\n答案：", "completion": "CPU 执行程序指令并协调计算任务。"},
    {"prompt": "问题：GPU 更擅长处理哪类计算？\n答案：", "completion": "GPU 适合大量并行的数据计算。"},
    {"prompt": "问题：什么是 Kubernetes Pod？\n答案：", "completion": "Pod 是 Kubernetes 中调度的基本运行单位。"},
    {"prompt": "问题：什么是显存？\n答案：", "completion": "显存用于保存 GPU 计算所需的数据、权重和中间状态。"},
    {"prompt": "问题：什么是 Checkpoint？\n答案：", "completion": "Checkpoint 保存训练状态，供后续恢复或检查。"},
    {"prompt": "问题：为什么需要记录实验版本？\n答案：", "completion": "版本影响兼容性、行为和结果的可复现性。"},
    {"prompt": "问题：什么是批处理？\n答案：", "completion": "批处理把多个计算任务组织起来共同处理。"},
    {"prompt": "问题：为什么监控不能只看 GPU 利用率？\n答案：", "completion": "请求延迟还会受到排队、输入长度与应用处理等因素影响。"},
]
dataset = Dataset.from_list(examples * 4)

config = SFTConfig(
    output_dir=output_dir,
    max_steps=args.max_steps,
    per_device_train_batch_size=1,
    gradient_accumulation_steps=2,
    learning_rate=2e-4,
    max_length=256,
    logging_steps=1,
    save_strategy="steps",
    save_steps=2,
    save_total_limit=3,
    save_only_model=False,
    report_to="none",
    fp16=not torch.cuda.is_bf16_supported(),
    bf16=torch.cuda.is_bf16_supported(),
    gradient_checkpointing=True,
    completion_only_loss=True,
    seed=42,
)

trainer = SFTTrainer(
    model=model_id,
    args=config,
    train_dataset=dataset,
    peft_config=LoraConfig(
        r=8,
        lora_alpha=16,
        lora_dropout=0.05,
        target_modules=["q_proj", "v_proj"],
        task_type="CAUSAL_LM",
    ),
)

checkpoint = get_last_checkpoint(output_dir) if args.resume and Path(output_dir).exists() else None
if args.resume and not checkpoint:
    raise RuntimeError("未找到可恢复的 checkpoint，请先运行第一阶段训练")
print("resume_from:", checkpoint)
trainer.train(resume_from_checkpoint=checkpoint)
trainer.save_model(os.path.join(output_dir, "adapter-final"))
print("max_allocated_GiB:", round(torch.cuda.max_memory_allocated() / 2**30, 3))
```

**说明：**`prompt` + `completion` 的损失掩码和 `completion_only_loss=True` 的语义应与 TRL 实际版本一致；数据很少，Loss 很可能不稳定，本例只验证工程流程。LoRA 配置中 `target_modules` 的可用名称依模型结构而变。

## 4. 两段式训练与恢复

第一次训练 4 个优化步骤，按每 2 步保存 Checkpoint：

```bash
python train_lora.py --max_steps 4 2>&1 | tee train-stage1.log
find outputs/ai-infra-lora-lab -maxdepth 2 -type f | head -60
```

正常情况下会产生 `checkpoint-2`、`checkpoint-4` 等目录（保留数量受配置影响），包含 LoRA 权重及用于恢复的训练器状态。

然后把目标训练步数提升到 8 步，**从同一个输出目录恢复**：

```bash
python train_lora.py --max_steps 8 --resume 2>&1 | tee train-stage2.log
find outputs/ai-infra-lora-lab -maxdepth 2 -type d | sort
```

`resume_from_checkpoint` 与“只加载 Adapter 权重”不同：严格续训应尽量恢复**模型、优化器、学习率调度器、随机数状态和步数**。如果 Checkpoint 中缺少这些状态，应明确记录为“从权重重新开始训练”，而不是断点续训。

## 5. 运行观测与验证

训练期间另开终端：

```bash
nvidia-smi --query-gpu=memory.used,utilization.gpu,power.draw --format=csv -l 1
# 结束观察时 Ctrl+C
```

核对 `train-stage1.log` 与 `train-stage2.log` 中的 `global_step` / Loss、内存峰值；建议保存：

| 实验项目 | 第一阶段 | 续训阶段 |
| --- | --- | --- |
| 最大步数 | 4 | 8 |
| 实际结束 global_step | 待测 | 待测 |
| 最近 Checkpoint | 待测 | 待测 |
| 最大显存峰值 | 待测 | 待测 |
| 是否读取旧 Checkpoint | 不适用 | 待验证 |
| 是否发生 OOM/异常 | 待测 | 待测 |

如遇 CUDA OOM，先降低 `max_length`、batch size 或换用更小模型、合适的量化训练方案；不要从本实验直接推出“多卡 DDP / DeepSpeed 也能正常运行”。

## 6. 常见问题与可选扩展

| 问题 | 优先检查 |
| --- | --- |
| PyTorch 不识别 GPU | `torch.cuda.is_available`、Driver、PyTorch CUDA wheel |
| Model / Tokenizer 加载失败 | 模型许可、网络、HF 缓存、Transformers 版本 |
| `target_modules` 无效 | 模型模块名称是否存在，LoRA 是否支持当前结构 |
| Checkpoint 恢复失败 | 目录是否完整，TRL/Transformers/PEFT 版本是否变化 |
| Loss 下降但回答无明显改善 | 数据规模、评测题集、任务是否足够有代表性 |
| 显存不足 | 显存余量、数据长度、batch、精度、优化器 |

**后续扩展（尚未包含在本实验）：**QLoRA 4-bit、模型评测、Adapter 合并、DDP/DeepSpeed、训练任务弹性调度。这些应在获得更多专题资料及明确版本后独立成 Lab，不把未经验证的复杂功能硬塞进最小示例。

## 7. 验收、清理与来源

- [ ] 数据全部为合成教学数据。
- [ ] 第一阶段训练有完整日志及 Checkpoint。
- [ ] 第二阶段日志能证明从旧 Checkpoint 继续至新的 global step。
- [ ] 保存 `requirements-resolved.txt`、GPU/Driver、模型 revision、训练参数。
- [ ] 不以短样本 Loss 变化推断实际模型能力提升。

清理前先备份所需 Checkpoint；之后只删除本实验的 `outputs/ai-infra-lora-lab/`，不要批量清理用户其他模型缓存。

- [Part I 08 · 训练与微调基础](../Part-I-Fundamentals/08-Training-Finetuning-Fundamentals.md)
- Hugging Face：[TRL SFTTrainer](https://huggingface.co/docs/trl/sft_trainer) · [Transformers Checkpoint 恢复](https://huggingface.co/docs/transformers/en/main_classes/trainer) · [PEFT](https://huggingface.co/docs/peft/)
- 这是对原有推理资料的**官方文档扩展**；模型训练代码为拟执行示例，不表示已在特定 GPU 上通过验证。

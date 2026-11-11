# Lab 01 · GPU 环境搭建与基础验收

> **实验目标：**在一台独占的实验主机上，打通 **物理 GPU → Linux 驱动 → NVIDIA Container Toolkit → Docker 容器**，形成可复核的环境验收记录。本实验不安装 Kubernetes，也不进行模型推理。

## 1. 范围、前提与验收标准

- 环境：Linux x86_64、NVIDIA GPU、管理员可用 `sudo`、已经安装 Docker Engine；推荐使用专用实验机。
- 权限：更改驱动或容器运行时前，确认设备没有承载其他用户工作负载；先准备回退窗口。
- 版本：**以实机记录为准**，不要直接套用文章里的驱动、CUDA 版本。`nvidia-smi` 中的 CUDA Version 表示驱动支持的 CUDA 上限，**不等于本机安装的 CUDA Toolkit 版本**。
- 出口条件：宿主机可以枚举 GPU；容器可以枚举同一设备；能解释驱动、Toolkit、容器镜像三者的关系；保存检查证据。

> **资料性质：**以下是可操作的实验步骤与**预期观察**，不是已执行的测试报告。包含系统配置修改的步骤，仅在实验机运行。

## 2. 环境记录（先填再操作）

| 条目 | 实际值（实验时填写） |
| --- | --- |
| 日期、实验人员 | 待填写 |
| 操作系统 / Kernel | 待填写 |
| GPU 型号、数量、显存 | 待填写 |
| Driver 版本 | 待填写 |
| Docker Engine / NVIDIA Toolkit | 待填写 |
| CUDA 测试镜像 tag | 待填写 |
| 是否独占实验机 | 待确认 |

```bash
uname -a
cat /etc/os-release
lspci | grep -i nvidia || true
nvidia-smi -L
nvidia-smi --query-gpu=index,name,uuid,memory.total,driver_version --format=csv
docker version
docker info --format '{{.ServerVersion}}'
```

`nvidia-smi` 不存在时，先核对 `lspci` 是否能发现 GPU，检查驱动安装状态和内核模块；不要为了“修复”直接执行未知驱动脚本。

## 3. 宿主机层检查

```bash
# 驱动与设备状态
nvidia-smi
nvidia-smi -q -d PERFORMANCE,POWER,TEMPERATURE,MEMORY | head -100

# 多卡拓扑；单卡设备也可执行
nvidia-smi topo -m

# 当前计算进程：不要在共享环境随意终止
nvidia-smi --query-compute-apps=pid,process_name,used_gpu_memory --format=csv

# 检查 Toolkit（可没有，容器使用的是镜像中的 CUDA 用户态库）
command -v nvcc && nvcc --version || echo 'No host CUDA Toolkit; not necessarily an error'
```

**判定：**`nvidia-smi` 应能够列出预期设备、显存总量和驱动版本。设备数量不符时，优先核对 PCIe、驱动日志、容器可见性设置，而非先调整 vLLM 参数。

## 4. 配置 Docker 访问 GPU

### 4.1 确认 Toolkit 是否已安装

```bash
command -v nvidia-ctk || echo 'Install NVIDIA Container Toolkit first'
# 已有 apt 软件源时：
dpkg -l | grep -E 'nvidia-container-toolkit|nvidia-container-runtime' || true
```

如果没有 `nvidia-ctk`，先按 [NVIDIA 官方 Container Toolkit 安装说明](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html) 配置对应发行版的软件源并安装。**不要复制第三方过期的 GPG key 或任意 apt 源**。

### 4.2 设置 Docker Runtime

```bash
# 查看 Docker 当前配置（如有自定义 daemon.json，先备份）
sudo cp -a /etc/docker/daemon.json /etc/docker/daemon.json.lab01.bak 2>/dev/null || true

sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker

docker info | grep -i -A3 runtimes
```

`systemctl restart docker` 可能中断宿主机上运行的容器，所以只对实验机操作。修改前如果已有现有配置，需要人工检查合并结果，而不是直接覆盖。

### 4.3 验证容器能够访问 GPU

```bash
# 这是示例 CUDA 镜像；必须与驱动兼容，不适配时换成经过确认的 tag。
docker run --rm --gpus all \
  nvidia/cuda:12.4.1-base-ubuntu22.04 \
  nvidia-smi
```

期望在容器输出中看到 GPU 列表。**宿主机能看到 GPU 而容器看不到 GPU**，通常优先检查 NVIDIA Container Toolkit、Docker Runtime 和设备暴露，不应重新安装整套 CUDA Toolkit。

## 5. 观察与问题定位

| 异常 | 首个检查点 | 下一层验证 |
| --- | --- | --- |
| `nvidia-smi: command not found` | 驱动/命令是否安装 | `lspci`、`lsmod`、系统日志 |
| `Failed to initialize NVML` | 驱动模块与用户态库 | 内核日志、驱动版本与重启状态 |
| `could not select device driver ... gpu` | Docker Runtime | `nvidia-ctk`、`docker info` |
| 容器内设备数量不对 | `--gpus`、设备可见性 | `NVIDIA_VISIBLE_DEVICES`、runtime 配置 |
| CUDA 初始化失败 | 镜像 CUDA 与宿主驱动兼容性 | 设备计算能力、驱动支持矩阵 |

只读日志入口：

```bash
journalctl -u docker -n 100 --no-pager
journalctl -k -n 100 --no-pager | grep -i -E 'nvidia|nvrm|xid' || true
```

不要看到 GPU Xid 就直接下结论“显卡损坏”；需要结合错误编号、复现负载、驱动日志和硬件检查。

## 6. 验收记录与退出

- [ ] GPU 数量、型号、Driver 与计划一致。
- [ ] `nvidia-smi` 运行正常，输出留存。
- [ ] Docker GPU 容器可以成功枚举设备。
- [ ] 解释了 `Driver / CUDA Toolkit / CUDA Runtime / Container Toolkit` 的边界。
- [ ] 保存版本、异常和修复记录。

**回退：**如更改运行时造成问题，在核对原配置有效的前提下恢复 `/etc/docker/daemon.json.lab01.bak` 并重启 Docker；如原文件不存在，不可直接使用复制命令恢复，需人工核对生成的配置。驱动回退需要按发行版与维护窗口单独处理。


# Slurm 任务调度使用指南

目前实验室有两台通过 Slurm 管理的计算服务器：

| 节点名称 | 类型 | 分区 | 可调度资源 | 登录入口 |
| -------- | ---- | ---- | ---------- | -------- |
| amax-a100 | GPU 服务器 | `normal` | 224 CPU / 512GiB 内存 / 2 × NVIDIA A100 80GB | SSH 至 amax-a100 的 Slurm Client |
| trx50-ai-top | CPU 服务器 | `normal` | 112 CPU / 128GiB 内存 | `ssh stort@trx50-ai-top-slurm` |

为了保证系统稳定性，节点对用户目录有严格限制，并强制通过 Slurm 调度系统管理资源。

### amax-a100 配置信息

```
ClusterName=NCCLab_HPC
SlurmctldHost=amax-a100

NodeName=amax-a100 CPUs=256 Boards=1 SocketsPerBoard=2 CoresPerSocket=64 ThreadsPerCore=2 RealMemory=515656 \
CPUSpecList=0-31 MemSpecLimit=32768 \
Gres=gpu:nvidia_a100_80gb_pcie:2 \
State=UNKNOWN

PartitionName=normal Nodes=amax-a100 Default=YES MaxTime=infinite State=UP

# Common Limitations
MaxNodes=UNLIMITED
MinNodes=0
MaxTime=UNLIMITED
MaxCPUsPerNode=UNLIMITED
MaxCPUsPerSocket=UNLIMITED
DefMemPerNode=UNLIMITED
MaxMemPerNode=UNLIMITED
TRES=cpu=224,mem=515656M,node=1,billing=224
```

### trx50-ai-top 配置信息

```
ClusterName=ncclab_hpc
SlurmctldHost=trx50-ai-top

NodeName=trx50-ai-top CPUs=128 Boards=1 SocketsPerBoard=1 CoresPerSocket=64 ThreadsPerCore=2 RealMemory=128071 \
CPUSpecList=0-15 MemSpecLimit=8192 \
State=UNKNOWN

PartitionName=normal Nodes=trx50-ai-top Default=YES MaxTime=infinite State=UP

# Common Limitations
MaxNodes=UNLIMITED
MinNodes=0
MaxTime=UNLIMITED
MaxCPUsPerNode=UNLIMITED
MaxCPUsPerSocket=UNLIMITED
DefMemPerNode=UNLIMITED
MaxMemPerNode=UNLIMITED
TRES=cpu=112,mem=128071M,node=1,billing=112
```

### 注意事项

- **存储限制**：`/home` 目录默认配额仅为 **10GB**。严禁在此存放数据集、实验结果或大型 Python 环境。
- **NAS 挂载**：实验室 NAS 已自动挂载至该节点。请将您的代码、Apptainer 容器镜像（`.sif`）及实验数据全部存放在 NAS 对应目录下。
- **GPU 任务运行模式**：`amax-a100` 节点**无法直接通过终端调用显卡**，必须通过 `sbatch` 或 `srun` 提交任务。
- **环境推荐**：推荐使用 **Apptainer (Singularity)** 容器技术。将所有依赖打包在镜像内，直接在 NAS 或本地数据盘上运行，无需在宿主机安装库。

### GPU 交互式调试 (amax-a100)

如果您想进入容器内部检查环境：

```bash
# 申请显卡并进入交互模式
srun --partition=normal --gres=gpu:1 --pty bash

# 在计算节点上运行容器
apptainer shell --nv /mnt/nas/your_name/pytorch_env.sif
```

### GPU 任务提交示例 (amax-a100)

代码示例：

```python
import torch
import sys

print(f"Python Version: {sys.version}")
print(f"PyTorch Version: {torch.__version__}")

cuda_available = torch.cuda.is_available()
print(f"CUDA Available: {cuda_available}")

if cuda_available:
    print(f"GPU Device Name: {torch.cuda.get_device_name(0)}")
    print(f"GPU Count: {torch.cuda.device_count()}")

    x = torch.rand(1000, 1000).cuda()
    y = torch.rand(1000, 1000).cuda()
    z = torch.matmul(x, y)

    print("Matrix multiplication on GPU successful!")
else:
    print("Error: CUDA not detected. Check --nv flag and driver status.")
    sys.exit(1)
```

编写一个作业脚本（例如 `train.sh`），并将其存放在特定目录中：

```bash
#!/bin/bash
#SBATCH --job-name=Torch_Test     # 任务名称
#SBATCH --partition=normal        # 分区名称
#SBATCH --gres=gpu:1              # 申请 1 块 A100 显卡
#SBATCH --cpus-per-task=8         # 申请 8 个 CPU 核心
#SBATCH --mem=32G                 # 申请 32GB 内存
#SBATCH --output=%j_test.out      # 标准输出日志
#SBATCH --error=%j_test.err       # 错误输出日志

# 定义镜像路径
CONTAINER_IMAGE="/home/container/pytorch_25.11.sif"

echo "Starting job at: $(date)"
echo "Running on node: $SLURM_NODELIST"

# 使用 Apptainer 运行测试
# 加入 --nv 参数才可以调用显卡
apptainer exec --nv $CONTAINER_IMAGE python check_gpu.py

echo "Job finished at: $(date)"
```

提交作业：

```bash
sbatch train.sh
```

任务完成后可以在 `{}.out` 中看到输出结果。

### CPU 交互式调试 (trx50-ai-top)

登录 Slurm Gateway 后，可以使用 `srun` 申请 CPU 与内存资源：

```bash
srun --partition=normal --cpus-per-task=8 --mem=16G --pty bash
```

如果使用 Apptainer 容器，不需要添加 `--nv`：

```bash
apptainer shell /mnt/data0/your_name/cpu_env.sif
```

### CPU 任务提交示例 (trx50-ai-top)

编写一个作业脚本（例如 `cpu_job.sh`）：

```bash
#!/bin/bash
#SBATCH --job-name=CPU_Test
#SBATCH --partition=normal
#SBATCH --cpus-per-task=16
#SBATCH --mem=32G
#SBATCH --output=%j_cpu.out
#SBATCH --error=%j_cpu.err

echo "Starting job at: $(date)"
echo "Running on node: $SLURM_NODELIST"
echo "Allocated CPUs: $SLURM_CPUS_PER_TASK"

python your_cpu_script.py

echo "Job finished at: $(date)"
```

提交作业：

```bash
sbatch cpu_job.sh
```

### 常用命令

| **命令**                     | **用途**                     |
| ---------------------------- | ---------------------------- |
| `squeue`                     | 查看当前队列中所有作业的状态 |
| `sinfo`                      | 查看节点状态和分区信息       |
| `scancel <job_id>`           | 取消指定的作业               |
| `scontrol show job <job_id>` | 查看作业的详细配置信息       |


# Slurm 任务调度使用指南

`amax-a100` 节点采用计算与存储分离的架构。为了保证系统稳定性，该节点对用户目录有严格限制，并强制通过 Slurm 调度系统管理资源。

### 配置信息

```
PartitionName=normal
   AllowGroups=ALL AllowAccounts=ALL AllowQos=ALL
   AllocNodes=ALL Default=YES QoS=N/A
   DefaultTime=NONE DisableRootJobs=NO ExclusiveUser=NO ExclusiveTopo=NO GraceTime=0 Hidden=NO
   MaxNodes=UNLIMITED MaxTime=UNLIMITED MinNodes=0 LLN=NO MaxCPUsPerNode=UNLIMITED MaxCPUsPerSocket=UNLIMITED
   Nodes=amax-a100
   PriorityJobFactor=1 PriorityTier=1 RootOnly=NO ReqResv=NO OverSubscribe=NO
   OverTimeLimit=NONE PreemptMode=OFF
   State=UP TotalCPUs=256 TotalNodes=1 SelectTypeParameters=NONE
   JobDefaults=(null)
   DefMemPerNode=UNLIMITED MaxMemPerNode=UNLIMITED
   TRES=cpu=224,mem=515656M,node=1,billing=224

NodeName=amax-a100 Arch=x86_64 CoresPerSocket=64
   CPUAlloc=0 CPUEfctv=224 CPUTot=256 CPULoad=0.02
   AvailableFeatures=(null)
   ActiveFeatures=(null)
   Gres=gpu:nvidia_a100_80gb_pcie:2
   NodeAddr=amax-a100 NodeHostName=amax-a100 Version=24.11.3
   OS=Linux 6.12.0-160000.9-default #1 SMP PREEMPT_DYNAMIC Fri Jan 16 09:29:05 UTC 2026 (9badd3c)
   RealMemory=515656 AllocMem=0 FreeMem=453775 Sockets=2 Boards=1
   CoreSpecCount=16 CPUSpecList=0-31 MemSpecLimit=32768
   State=IDLE ThreadsPerCore=2 TmpDisk=0 Weight=1 Owner=N/A MCS_label=N/A
   Partitions=normal
   BootTime=2026-02-25T17:35:51 SlurmdStartTime=2026-02-26T16:01:35
   LastBusyTime=2026-03-12T19:53:46 ResumeAfterTime=None
   CfgTRES=cpu=224,mem=515656M,billing=224
   AllocTRES=
   CurrentWatts=0 AveWatts=0
```

### 注意事项

- **存储限制**：`/home` 目录配额仅为 **10GB**。严禁在此存放数据集、实验结果或大型 Python 环境。
- **NAS 挂载**：实验室 NAS 已自动挂载至该节点。请将您的代码、Apptainer 容器镜像（`.sif`）及实验数据全部存放在 NAS 对应目录下。
- **运行模式**：该节点**无法直接通过终端调用显卡**，必须通过 `sbatch` 或 `srun` 提交任务。
- **环境推荐**：推荐使用 **Apptainer (Singularity)** 容器技术。将所有依赖打包在镜像内，直接在 NAS 上运行，无需在宿主机安装库。

### 交互式调试 (Debug)

如果您想进入容器内部检查环境：

```bash
# 申请显卡并进入交互模式
srun --partition=normal --gres=gpu:1 --pty bash

# 在计算节点上运行容器
apptainer shell --nv /mnt/nas/your_name/pytorch_env.sif
```

### 任务提交示例 (sbatch)

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

### 常用命令

| **命令**                     | **用途**                     |
| ---------------------------- | ---------------------------- |
| `squeue`                     | 查看当前队列中所有作业的状态 |
| `sinfo`                      | 查看节点状态和分区信息       |
| `scancel <job_id>`           | 取消指定的作业               |
| `scontrol show job <job_id>` | 查看作业的详细配置信息       |


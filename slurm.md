# Slurm 任务调度使用指南

`amax-a100` 节点采用计算与存储分离的架构。为了保证系统稳定性，该节点对用户目录有严格限制，并强制通过 Slurm 调度系统管理资源。

### 注意事项

- **存储限制**：`/home` 目录配额仅为 **10GB**。严禁在此存放数据集、实验结果或大型 Python 环境。
- **NAS 挂载**：实验室 NAS 已自动挂载至该节点。请将您的代码、Apptainer 容器镜像（`.sif`）及实验数据全部存放在 NAS 对应目录下。
- **运行模式**：该节点**无法直接通过终端调用显卡**，必须通过 `sbatch` 或 `srun` 提交任务。
- **环境推荐**：推荐使用 **Apptainer (Singularity)** 容器技术。将所有依赖打包在镜像内，直接在 NAS 上运行，无需在宿主机安装库。

### 交互式调试 (Debug)

如果您想进入容器内部检查环境：

```bash
# 申请显卡并进入交互模式
srun --partition=a100 --gres=gpu:1 --pty bash

# 在计算节点上运行容器
apptainer shell --nv /mnt/nas/your_name/pytorch_env.sif
```

### 任务提交示例 (sbatch)

编写一个作业脚本（例如 `train.sh`），并将其存放在您的 NAS 目录中：

```bash
#!/bin/bash
#SBATCH --job-name=NCC_Train      # 任务名称
#SBATCH --partition=a100          # 分区名称
#SBATCH --gres=gpu:1              # 申请 1 块 A100 显卡
#SBATCH --cpus-per-task=16        # 申请 CPU 核心数
#SBATCH --mem=64G                 # 申请内存大小
#SBATCH --output=%j.out           # 标准输出日志 (%j 为作业号)
#SBATCH --error=%j.err            # 错误日志

# 1. 进入您的 NAS 工作目录
cd /mnt/dataset/your_name/project_dir

# 2. 使用 Apptainer 运行容器内的指令
# 假设您的镜像文件为 my_env.sif
apptainer exec --nv my_env.sif python train.py --epochs 100
```

提交作业：

```bash
sbatch train.sh
```

### 常用命令

| **命令**                     | **用途**                     |
| ---------------------------- | ---------------------------- |
| `squeue`                     | 查看当前队列中所有作业的状态 |
| `sinfo`                      | 查看节点状态和分区信息       |
| `scancel <job_id>`           | 取消指定的作业               |
| `scontrol show job <job_id>` | 查看作业的详细配置信息       |


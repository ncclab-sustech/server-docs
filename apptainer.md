# Apptainer 使用教程

## Apptainer 定义文件 (pytorch.def)

创建一个名为 `pytorch.def` 的文件。我们推荐基于 NVIDIA 官方的 PyTorch 容器进行构建，因为它已经针对 GPU 驱动和 CUDA 进行了深度优化。

```
Bootstrap: docker
From: nvcr.io/nvidia/pytorch:24.01-py3  # 使用 NVIDIA 官方镜像作为基础

%post
    # 更新系统并安装必要的 Linux 工具
    apt-get update && apt-get install -y \
        git \
        vim \
        wget \
        tmux && \
        apt-get clean && \
        rm -rf /var/lib/apt/lists/*

    # 安装额外的 Python 库
    # 注意：基础镜像已自带 PyTorch，此处只需安装项目特定依赖
    pip install --no-cache-dir \
        matplotlib \
        scikit-learn \
        pandas \
        mne

%environment
    # 设置容器运行时的环境变量
    export LC_ALL=C
    export PYTHONUNBUFFERED=1

%runscript
    # 定义容器默认执行的命令
    exec python "$@"

```

## 构建镜像

在有 **root 权限** 的 Linux 环境下执行以下命令进行构建：

```bash
apptainer build pytorch_env.sif pytorch.def
```

构建完成后，将生成的 `pytorch_env.sif` 文件通过发送至实验室的 **NAS 目录** 中。

## 通过 Slurm 构建镜像

Slurm Gateway 可能运行在 Docker/LXC 等容器化环境中，因此即使 gateway 上有 `apptainer` 命令，也不一定允许直接运行或构建镜像。常见报错如下：

```text
ERROR  : Failed to create user namespace: not allowed to create user namespace
```

这是合理的安全限制。推荐做法是：**在 gateway 上编写定义文件和提交脚本，通过 Slurm 把构建任务提交到计算节点执行**。

### 准备构建目录

不要把 Apptainer 缓存、临时目录或生成的 `.sif` 镜像放在 `/home` 中。请放在 NAS 或本地数据盘上。

CPU 节点示例：

```bash
mkdir -p /mnt/data0/$USER/apptainer-build
cd /mnt/data0/$USER/apptainer-build
```

GPU 节点示例：

```bash
mkdir -p /mnt/dataset0/$USER/apptainer-build
cd /mnt/dataset0/$USER/apptainer-build
```

将 `pytorch.def` 放在该目录下。

### 编写 Slurm 构建脚本

创建 `build_apptainer.sh`：

```bash
#!/bin/bash
#SBATCH --job-name=Build_Apptainer
#SBATCH --partition=normal
#SBATCH --cpus-per-task=4
#SBATCH --mem=16G
#SBATCH --output=%j_build.out
#SBATCH --error=%j_build.err

set -euo pipefail

cd "$SLURM_SUBMIT_DIR"

export APPTAINER_CACHEDIR="$SLURM_SUBMIT_DIR/.apptainer-cache"
export APPTAINER_TMPDIR="$SLURM_SUBMIT_DIR/.apptainer-tmp"

mkdir -p "$APPTAINER_CACHEDIR" "$APPTAINER_TMPDIR"

apptainer build --fakeroot pytorch_env.sif pytorch.def
```

说明：

- `--fakeroot` 用于在没有真实 root 权限时构建需要安装系统包的镜像。
- 构建镜像通常不需要 GPU，因此不需要写 `#SBATCH --gres=gpu:1`。
- `APPTAINER_CACHEDIR` 和 `APPTAINER_TMPDIR` 必须放在空间充足的目录中。
- 如果计算节点没有配置 fakeroot，构建会失败，需要联系管理员启用 fakeroot、user namespace 或 `apptainer-suid`。

### 提交构建任务

```bash
sbatch build_apptainer.sh
```

查看任务状态：

```bash
squeue -u $USER
```

查看构建日志：

```bash
tail -f <job_id>_build.out
tail -f <job_id>_build.err
```

构建成功后，当前目录会生成：

```text
pytorch_env.sif
```

### 测试镜像

CPU 镜像测试：

```bash
srun --partition=normal --cpus-per-task=4 --mem=8G --pty bash
apptainer exec pytorch_env.sif python --version
```

GPU 镜像测试：

```bash
srun --partition=normal --gres=gpu:1 --cpus-per-task=4 --mem=16G --pty bash
apptainer exec --nv pytorch_env.sif python -c "import torch; print(torch.cuda.is_available())"
```

注意：`--nv` 只在运行 GPU 程序时需要，构建镜像时一般不需要。


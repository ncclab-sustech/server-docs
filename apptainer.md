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
        tmux

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

------

## 构建镜像

在有 **root 权限** 的 Linux 环境下执行以下命令进行构建：

```
apptainer build pytorch_env.sif pytorch.def
```

构建完成后，将生成的 `pytorch_env.sif` 文件通过发送至实验室的 **NAS 目录** 中。


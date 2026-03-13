# NCC Lab 服务器资源使用手册

### 节点资源信息

| 节点名称  | CPU                       | 内存                    | GPU                    | 储存                           |
| --------- | ------------------------- | ----------------------- | ---------------------- | ------------------------------ |
| nccserv0  | AMD EPYC 7T83 64-Core     | 512GiB (64GB DDR4 * 8)  | 8 × RTX 3090 (24G)     | 500GB SSD + 2TB SSD            |
| nccserv1  | AMD EPYC 7T83 64-Core     | 512GiB (64GB DDR4 * 8)  | 5 × RTX 4090 (24G)     | 2TB SSD + 7.68TB SSD (3.84T*2) |
| amax-a100 | 2 × AMD EPYC 7Y83 64-Core | 512GiB (32GB DDR4 * 16) | 2 × NVIDIA A100 (80GB) | 512GB SSD + 3.84TB SSD         |
| amax-l40  | 2 × AMD EPYC 9554 64-Core | 512GiB (64GB DDR5 * 8)  | 8 × NVIDIA L40 (48G)   | 3.84TB SSD + 16TB HDD          |

| **节点名称** | **系统**                                                   | **SSH 地址** | **端口** |
| ------------ | ---------------------------------------------------------- | ------------ | -------- |
| nccserv0     | Ubuntu 22.04                                               | 10.20.38.38  | 1022     |
| nccserv1     | Ubuntu 22.04                                               | 10.20.37.212 | 1022     |
| amax-a100    | openSUSE Leap 16.0 (Slurm Host) / Debian 13 (Slurm Client) | 10.20.37.38  | 1022     |
| amax-l40     | Ubuntu 20.04                                               | 10.20.97.33  | 1022     |

注意：

- 服务器只接受密钥连接，推荐使用 `ed25519`: `ssh-keygen -t ed25519`

### NAS资源信息

| NAS 名称 | 远程地址 (SAMBA / SMB) | 总容量  | 服务器挂载路径      |
| -------- | ---------------------- | ------- | ------------------- |
| dataset0 | //10.20.37.22/dataset0 | 143 TiB | /mnt/dataset0 (NFS) |
| dataset1 | //10.16.57.94/dataset1 | 100 TiB | /mnt/dataset1       |
| dataset2 | //10.16.57.94/dataset2 | 53 TiB  | /mnt/dataset2       |
| dataset3 | //10.16.93.90/dataset3 | 158 TiB | /mnt/dataset3       |
| dataset4 | //10.20.33.82/dataset4 | 419 TiB | /mnt/dataset4 (NFS) |

注：

- `dataset0` / `dataset4` 与四台服务器通过 **10G** 网络连接
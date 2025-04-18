# NCCL 带宽测试指南

## 1. nccl-tests 编译安装
### 环境要求
- CUDA Toolkit (>=11.0)
- NCCL 库 (建议使用与 CUDA 版本匹配的 NCCL)

```bash
# 设置环境变量
export CUDA_HOME=/usr/local/cuda  # CUDA 安装路径
export LD_LIBRARY_PATH=$CUDA_HOME/lib64:$LD_LIBRARY_PATH

# 下载源码
git clone https://github.com/NVIDIA/nccl-tests.git
cd nccl-tests

# 编译安装
make MPI=1 NCCL_HOME=/path/to/nccl  # 指定 NCCL 安装路径
sudo make install
```

## 2. 单机多卡带宽测试
### 测试命令
```bash
# 8 GPU 测试
NCCL_DEBUG=INFO ./build/all_reduce_perf -b 8G -e 8G -f 2 -g 8
```

### 输出关键参数分析
```log
# 典型输出片段
nvidia-pci-0100   Send bytes: 8589934592 -> 8.000 GB
nvidia-pci-0100   Out of Place bandwidth : 201.43 GB/s  # 实际带宽
nvidia-pci-0100   Algo 0: trees         # 算法类型
nvidia-pci-0100   Size : 8      (0.000 MB)  # 数据块大小
```

## 3. 多机测试 SSH 免密配置
### 配置流程
```bash
# 生成密钥对（所有节点执行）
ssh-keygen -t rsa -P "" -f ~/.ssh/id_rsa

# 主节点分发公钥
ssh-copy-id -i ~/.ssh/id_rsa.pub user@node2
ssh-copy-id -i ~/.ssh/id_rsa.pub user@node3

# 验证免密登录
ssh node2 hostname
```

## 4. 结果解析脚本
### Python 解析脚本
```python
import re

pattern = r"Algo (\d+).*?bw\s*:\s*([\d.]+)\s*GB/s.*?Size\s*:\s*(\d+)"
results = []

with open("nccl.log") as f:
    for line in f:
        match = re.search(pattern, line)
        if match:
            algo, bw, size = match.groups()
            results.append((int(algo), float(bw), int(size)))

# 生成表格
print("| Algo | Bandwidth (GB/s) | Size |")
print("|------|------------------|------|")
for algo, bw, size in sorted(results, key=lambda x: x[2]):
    print(f"| {algo} | {bw:.2f} | {size} |")
```

## 附：IB 网卡绑定验证
### ibdev2netdev 验证方法
```bash
# 安装工具（如未自带）
sudo apt install infiniband-diags

# 查看绑定状态
ibdev2netdev -v

# 期望输出示例
mlx5_0 port 1 ==> enp1s0f0 (Up)  # 正确绑定物理网卡
mlx5_1 port 2 ==> enp1s0f1 (Up)

# 异常情况处理
# 若显示 "N/A" 或错误网卡，需重新绑定：
sudo ibdev2netdev -c
```

## 注意事项
1. 多机测试需保证所有节点时钟同步
2. 建议关闭 NUMA 平衡：`echo 0 > /proc/sys/kernel/numa_balancing`
3. IB 网络需验证链路状态：`ibstatus`
4. 测试前执行预热：`nvidia-smi -pm 1`

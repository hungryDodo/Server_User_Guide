# 实验室服务器快速入门指南

## 1. SSH连接服务器
```bash
# 基础连接命令（替换username和server_ip）
ssh -p 3222 username@server_ip -i ~/.ssh/your_private_key

# 保持连接防止超时（每60秒发送心跳包）
ssh -o ServerAliveInterval=60 -p 3222 username@server_ip -i ~/.ssh/your_private_key

# 首次连接跳过指纹验证
ssh -o StrictHostKeyChecking=no -p 3222 username@server_ip -i ~/.ssh/your_private_key
```
**注意**：私钥文件权限需设置为600：`chmod 600 ~/.ssh/your_private_key`

---

## 2. Conda环境配置流程
```bash
# 安装Miniconda
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
bash Miniconda3-latest-Linux-x86_64.sh -b -p $HOME/miniconda
source ~/miniconda/bin/activate

# 创建Python3.8环境
conda create -n lab_env python=3.8 -y
conda activate lab_env

# 安装PyTorch全家桶（CUDA 11.6版本）
conda install pytorch torchvision torchaudio pytorch-cuda=11.6 -c pytorch -c nvidia -y

# 导出环境配置
conda env export > environment.yml
```

---

## 3. GPU设备检测脚本
创建`check_gpu.py`：
```python
import torch

print(f"CUDA Available: {torch.cuda.is_available()}")
print(f"GPU数量: {torch.cuda.device_count()}")
print(f"当前设备: {torch.cuda.current_device()}")
print(f"设备名称: {torch.cuda.get_device_name(0)}")

# 显存信息查询
if torch.cuda.is_available():
    print(f"显存总量: {torch.cuda.get_device_properties(0).total_memory/1024**3:.2f} GB")
```

执行命令：
```bash
python check_gpu.py
```

---

## 4. 多卡训练示例
创建`train.py`：
```python
import torch
import torch.nn as nn
import torch.distributed as dist
from torch.nn.parallel import DistributedDataParallel as DDP

def main():
    # 初始化进程组
    dist.init_process_group(backend='nccl')
    local_rank = int(os.environ["LOCAL_RANK"])
    torch.cuda.set_device(local_rank)

    # 创建模型
    model = nn.Linear(10, 10).cuda()
    ddp_model = DDP(model, device_ids=[local_rank])

    # 测试数据
    inputs = torch.randn(20, 10).cuda()
    outputs = ddp_model(inputs)
    print(f"Rank {local_rank}: Output shape {outputs.shape}")

    # 清理进程组
    dist.destroy_process_group()

if __name__ == "__main__":
    main()
```

启动命令：
```bash
# 使用2个GPU
torchrun --nproc_per_node=2 --nnodes=1 --node_rank=0 --master_addr=127.0.0.1 --master_port=29500 train.py

# nohup后台运行模板
nohup torchrun --nproc_per_node=2 train.py > train.log 2>&1 & echo $! > train.pid
```

**操作说明**：
- 实时查看日志：`tail -f train.log`
- 终止进程：`kill -9 $(cat train.pid)`
- 建议设置不同的`master_port`（29500-29599）防止端口冲突

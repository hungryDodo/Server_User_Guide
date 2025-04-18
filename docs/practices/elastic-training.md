# TorchElastic 弹性训练测试文档

## 1. TorchElastic 安装步骤 (v1.13.1)

```bash
# 指定CUDA 11.7版本
pip install torch==1.13.1+cu117 torchvision==0.14.1+cu117 --extra-index-url https://download.pytorch.org/whl/cu117
pip install torchelastic==0.2.2

# 验证安装
python -c "import torch.distributed.elastic as elastic; print(elastic.__version__)"
```

CUDA版本匹配建议表：
| TorchElastic 版本 | PyTorch 版本 | CUDA 版本 | 支持架构 |
|-------------------|--------------|-----------|----------|
| 0.2.x             | 1.13.x       | 11.7-11.8 | Ampere+  |
| 0.1.x             | 1.12.x       | 11.3-11.6 | Turing   |

## 2. 节点动态扩展配置 (etcd)

启动etcd服务器：
```bash
etcd --enable-v2=true \
     --listen-client-urls http://0.0.0.0:2379 \
     --advertise-client-urls http://${HOST_IP}:2379 \
     --data-dir /var/lib/etcd
```

关键参数说明：
- `initial-cluster-state new`：新集群初始化模式
- `heartbeat-interval 500`：500ms心跳检测
- `election-timeout 2500`：2.5秒选举超时

## 3. 故障注入测试脚本

auto_fault_injection.sh：
```bash
#!/bin/bash
# 随机终止训练进程
while true; do
    PID=$(ps -ef | grep "train_script.py" | grep -v grep | awk '{print $2}' | shuf -n1)
    if [ ! -z "$PID" ]; then
        echo "$(date) Killing PID: $PID" >> fault_log.txt
        kill -9 $PID
    fi
    sleep $((RANDOM % 1800 + 600))  # 10-30分钟随机间隔
done
```

## 4. Checkpoint 完整性验证

```python
import hashlib

def verify_checkpoint(checkpoint_path):
    with open(checkpoint_path, "rb") as f:
        md5 = hashlib.md5(f.read()).hexdigest()
    
    # 对比上次保存的校验和
    with open("checkpoints.md5", "r") as f:
        prev_md5 = f.read().strip()
    
    assert md5 == prev_md5, f"Checkpoint corrupted! Current: {md5}, Expected: {prev_md5}"
    print("Checkpoint verification passed")
```

操作流程：
```bash
# 生成校验和
md5sum checkpoint.pt > checkpoints.md5

# 恢复时验证
if md5sum -c checkpoints.md5; then
    echo "Checkpoint valid"
else
    echo "Checkpoint invalid" >&2
    exit 1
fi
```

## 弹性参数配置建议表

| 场景                  | min_replicas | max_replicas | 建议值组合 |
|-----------------------|--------------|--------------|------------|
| 单机多卡开发环境      | 1            | 4            | (1,4)      |
| 小规模测试集群        | 2            | 8            | (2,8)      |
| 生产环境弹性调度      | 4            | 32           | (4,32)     |
| 高可用关键任务        | 总节点数/2   | 总节点数     | (N/2, N)   |

最佳实践建议：
1. 开发环境设置max_replicas不超过物理节点数的200%
2. 生产环境建议预留20%资源缓冲空间
3. 当使用Spot实例时建议设置min_replicas=ceil(总需求的70%)
4. 关键任务系统建议min_replicas >= 3以保证容错性

# 容错训练恢复指南

## 1. 自动保存点配置
### 频率策略
```yaml
# PyTorch 示例配置
checkpoint_config = {
    'interval': 30,      # 每30分钟保存
    'step_interval': 1000, # 每1000步保存
    'keep_last': 5,      # 保留最近5个检查点
    'save_optimizer': True
}

# TensorFlow 示例配置
model_checkpoint = tf.keras.callbacks.ModelCheckpoint(
    filepath='checkpoints/epoch-{epoch:04d}',
    save_weights_only=False,
    save_freq='epoch'  # 或指定steps=500
)
```

### 存储路径设置
```bash
# 分布式存储路径示例
export CHECKPOINT_PATH=/nfs_shared/checkpoints/${USER}/${SLURM_JOB_ID}

# 本地缓存+异步上传方案
local_dir=/tmp/checkpoints
rsync -avz ${local_dir}/* ${CHECKPOINT_PATH} &
```

## 2. 断点续训代码示例
### PyTorch 优化器状态加载
```python
def load_checkpoint(model, optimizer, path):
    checkpoint = torch.load(path)
    model.load_state_dict(checkpoint['model_state'])
    optimizer.load_state_dict(checkpoint['optimizer_state'])
    epoch = checkpoint['epoch'] + 1
    return epoch

# 恢复训练示例
if os.path.exists('checkpoint.pt'):
    start_epoch = load_checkpoint(model, optimizer, 'checkpoint.pt')
else:
    start_epoch = 0
```

## 3. 硬件故障模拟测试
### GPU设备屏蔽命令
```bash
# 模拟单个GPU故障
sudo nvidia-smi --gpu-reset -i 1

# 使用cgroup限制GPU内存
sudo cgcreate -g memory:gpufault
sudo cgset -g memory:gpufault memory.limit_in_bytes=1G
sudo cgexec -g memory:gpufault python train.py

# 屏蔽特定GPU设备
export CUDA_VISIBLE_DEVICES=0,2,3  # 隐藏GPU 1
```

## 4. 日志追踪方案
### ELK 配置参数示例
```yaml
# Filebeat 配置 (filebeat.yml)
filebeat.inputs:
- type: log
  paths:
    - /var/log/training/*.log
  fields:
    job_id: ${SLURM_JOB_ID}
    node: ${HOSTNAME}

output.logstash:
  hosts: ["logstash:5044"]

# Logstash 过滤规则 (trainlog.conf)
filter {
  grok {
    match => { "message" => "%{TIMESTAMP_ISO8601:timestamp} %{LOGLEVEL:level} \[%{DATA:module}\] %{GREEDYDATA:msg}" }
  }
  mutate {
    add_field => { 
      "node_id" => "%{[fields][node]}"
      "job_id" => "%{[fields][job_id]}" 
    }
  }
}
```

## MPI作业重新排程脚本模板
```bash
#!/bin/bash
# mpi_restart.sh

ORIGINAL_NODES=4
FAILED_NODE="node03"
MAX_RETRY=3

# 检查失败节点状态
if ping -c 1 $FAILED_NODE >/dev/null; then
    echo "Node $FAILED_NODE alive but not responding"
else
    echo "Node $FAILED_NODE is down"
fi

# 重新调度作业
scontrol update jobid=$SLURM_JOB_ID \
    MinNodes=$((ORIGINAL_NODES-1)) \
    ExcNodeList=$FAILED_NODE

# 恢复训练命令
mpirun -np $((ORIGINAL_NODES-1)) \
    -hostfile <(scontrol show hostnames $SLURM_NODELIST | grep -v $FAILED_NODE) \
    python train.py --resume checkpoint_latest.pt
```

## 附录：最佳实践建议
1. 混合保存策略：每小时完整检查点 + 每15分钟差异保存
2. 存储验证：每次保存后执行checksum校验
3. 故障注入测试：定期随机kill训练进程验证恢复能力
4. 日志分级：ERROR级别日志必须包含节点ID和时间戳
5. 监控集成：Prometheus + Grafana实时显示检查点状态

> 注意：测试硬件故障命令需在隔离环境执行，避免影响生产系统。建议使用Docker容器或专用测试集群进行故障模拟。
# LLaMA-7B 多卡训练指南

## 1. 环境准备

### 基础依赖
```bash
# 安装PyTorch(>=1.12)
conda install pytorch==1.13.1 torchvision==0.14.1 torchaudio==0.13.1 -c pytorch

# 安装Apex混合精度库
git clone https://github.com/NVIDIA/apex
cd apex && pip install -v --no-cache-dir \
--global-option="--cpp_ext" \
--global-option="--cuda_ext" \
--global-option="--deprecated_fused_adam" ./

# 安装指定版本Transformers
pip install transformers==4.28.0
pip install git+https://github.com/huggingface/transformers

# 安装DeepSpeed
pip install deepspeed==0.9.2
```

### 环境验证
```bash
nvidia-smi  # 确认GPU驱动正常
gpustat -i  # 查看多卡状态
```

## 2. DeepSpeed配置文件（ds_config.json）
```json
{
  "train_batch_size": "auto",
  "gradient_accumulation_steps": "auto",
  "optimizer": {
    "type": "AdamW",
    "params": {
      "lr": 2e-5,
      "weight_decay": 0.0
    }
  },
  "scheduler": {
    "type": "WarmupDecayLR",
    "params": {
      "warmup_min_lr": 0,
      "warmup_max_lr": 2e-5,
      "warmup_num_steps": 1000
    }
  },
  "zero_optimization": {
    "stage": 3,
    "offload_optimizer": {
      "device": "cpu",
      "pin_memory": true
    },
    "offload_param": {
      "device": "cpu", 
      "pin_memory": true
    },
    "contiguous_gradients": true,
    "overlap_comm": true
  },
  "fp16": {
    "enabled": true,
    "loss_scale": 0,
    "loss_scale_window": 1000,
    "initial_scale_power": 16,
    "hysteresis": 2,
    "min_loss_scale": 1
  },
  "bf16": {
    "enabled": false
  },
  "gradient_clipping": 1.0,
  "steps_per_print": 50
}
```

## 3. 启动命令模板
```bash
# 单机多卡启动（4 GPU）
CUDA_VISIBLE_DEVICES=0,1,2,3 \
MASTER_ADDR=localhost \
deepspeed --num_gpus=4 \
--module training.run_clm \
--model_name_or_path /path/to/llama-7b \
--dataset_name wikitext \
--per_device_train_batch_size 2 \
--gradient_accumulation_steps 8 \
--deepspeed ds_config.json \
--output_dir ./output
```

## 4. 训练监控方法

### 实时GPU监控
```bash
watch -n 1 gpustat --color  # 每秒刷新GPU状态

# 输出示例：
[0] NVIDIA A100-SXM4-40GB | 78°C, 96% | 32510 / 40960 MB | user/pid(43321) python(32500M)
```

### 日志解析脚本
```python
# parse_logs.py
import re

with open("training.log") as f:
    for line in f:
        if "loss" in line:
            step = re.search(r"step (\d+)", line).group(1)
            loss = re.search(r"loss: (\d+\.\d+)", line).group(1)
            print(f"Step {step}: Loss={loss}")
```

## FP16/BF16混合精度对比

| 特性               | FP16                          | BF16                          |
|--------------------|-------------------------------|-------------------------------|
| 内存占用           | 较高（动态损失缩放）          | 较低（自动扩展数值范围）      |
| 训练稳定性         | 需要动态损失缩放              | 原生稳定，适合大模型          |
| 硬件要求           | 所有CUDA GPU                  | Ampere架构+（A100/3090+）     |
| 典型batch_size     | 2-4（A100 40GB）              | 4-8（A100 40GB）              |
| 常见问题           | 梯度溢出/下溢                 | CUDA内核不兼容问题            |
| OOM解决方案        | 1. 启用梯度检查点<br>2. 增大offload比例<br>3. 减小batch_size | 1. 升级CUDA到11.0+<br>2. 禁用自定义CUDA内核 |

### OOM错误处理流程
1. 检查`nvidia-smi`确认显存占用
2. 逐步启用以下配置（按优先级）：
   ```json
   "gradient_checkpointing": true,
   "zero_optimization.reduce_bucket_size": 5e8,
   "zero_optimization.stage3_max_live_parameters": 1e9
   ```
3. 最终手段：添加`--fp16_full_eval`评估时降精度

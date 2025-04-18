# GPU吞吐量测试指南

---

## 1. 测试脚本代码
```python
import torch
import time

def test_throughput(precision, compute_type="matmul", size=4096):
    # 精度设置
    dtype = torch.float16 if precision == "fp16" else \
            torch.float32 if precision == "fp32" else \
            torch.float32  # TF32使用float32容器
    torch.backends.cuda.matmul.allow_tf32 = (precision == "tf32")
    torch.backends.cudnn.allow_tf32 = (precision == "tf32")

    # 创建测试数据
    x = torch.randn(size, size, device="cuda", dtype=dtype)
    y = torch.randn(size, size, device="cuda", dtype=dtype)
    
    # 预热
    for _ in range(10):
        if compute_type == "matmul":
            _ = torch.matmul(x, y)
        else:  # conv
            conv = torch.nn.Conv2d(3, 64, kernel_size=3).cuda().to(dtype)
            _ = conv(torch.randn(1,3,224,224,device="cuda",dtype=dtype))
    
    # 正式测试
    torch.cuda.synchronize()
    start = time.time()
    for _ in range(100):
        if compute_type == "matmul":
            _ = torch.matmul(x, y)
        else:
            conv = torch.nn.Conv2d(3, 64, kernel_size=3).cuda().to(dtype)
            _ = conv(torch.randn(1,3,224,224,device="cuda",dtype=dtype))
    torch.cuda.synchronize()
    elapsed = time.time() - start
    
    # 计算吞吐量
    operations = 2 * size**3 * 100  # matmul FLOPs
    if compute_type == "conv":
        operations = 100 * 3*64*3*3*224*224*2  # conv FLOPs
    tflops = (operations / elapsed) / 1e12
    return tflops
```

---

## 2. 多卡NCCL测试命令
```bash
# AllReduce测试
python -m torch.distributed.run --nproc_per_node=8 all_reduce_test.py \
    --precision fp16 --message-size 256M

# AllToAll测试
python -m torch.distributed.run --nproc_per_node=8 all_to_all_test.py \
    --precision tf32 --iterations 1000
```

测试脚本核心代码：
```python
import torch.distributed as dist

def test_nccl(precision, operation):
    dist.init_process_group(backend='nccl')
    tensor = torch.randn(2**28, device='cuda',  # 256MB
                        dtype=torch.float16 if precision == "fp16" else torch.float32)
    
    timings = []
    for _ in range(100):
        start = torch.cuda.Event(enable_timing=True)
        end = torch.cuda.Event(enable_timing=True)
        start.record()
        if operation == "all_reduce":
            dist.all_reduce(tensor)
        elif operation == "all_to_all":
            output = [torch.empty_like(tensor) for _ in range(dist.get_world_size())]
            dist.all_to_all(output, tensor)
        end.record()
        torch.cuda.synchronize()
        timings.append(start.elapsed_time(end))
    
    bandwidth = (2 * tensor.element_size() * tensor.nelement() * 1e3) / \
               (sum(timings) * 1e9)  # GB/s
    return bandwidth
```

---

## 3. 功率监测方法
```bash
# 开始监测（每1秒采样一次）
nvidia-smi dmon -d 1 -s pucvmet -o DT -f power.log

# 参数说明：
# -d 1      : 采样间隔1秒
# -s pucvmet: 监测电源(p)、使用率(u)、时钟(c)、温度(v)、显存(m)、ECC错误(e)、吞吐量(t)
# -o DT     : 显示时间戳（格式：YYYY/MM/DD HH:MM:SS）
# -f        : 输出到文件

# 典型输出示例：
# # gpu   pwr  temp    sm   mem   enc   dec  mclk  pclk
# Idx     W     C     %     %     %     %   MHz   MHz
#    0   135    56    97    34     0     0  7159  1980
```

---

## 4. 结果分析模板
```python
import pandas as pd
import matplotlib.pyplot as plt

# 生成CSV报告
data = {
    "Test Type": ["matmul", "conv", "all_reduce", ...],
    "Precision": ["fp16", "fp32", "tf32", ...],
    "Throughput (TFLOPS)": [120, 45, 85, ...],
    "Bandwidth (GB/s)": [..., ...],
    "Avg Power (W)": [250, 300, 280, ...]
}
df = pd.DataFrame(data)
df.to_csv("gpu_benchmark.csv", index=False)

# 绘制折线图
plt.figure(figsize=(12,6))
for precision in ["fp16", "fp32", "tf32"]:
    subset = df[df["Precision"] == precision]
    plt.plot(subset["Test Type"], subset["Throughput (TFLOPS)"], 
             marker='o', label=precision)

plt.title("Compute Throughput Comparison")
plt.ylabel("TFLOPS")
plt.xticks(rotation=45)
plt.grid(True)
plt.legend()
plt.tight_layout()
plt.savefig("throughput_comparison.png")
```

---

## 完整测试流程
1. **环境准备**
```bash
export NVIDIA_TF32_OVERRIDE=1  # 强制启用TF32
pip install torch==2.1.0+cu118 --extra-index-url https://download.pytorch.org/whl/cu118
```

2. **执行测试矩阵**
| 测试类型   | 精度选项       | 执行命令                      |
|------------|----------------|------------------------------|
| 矩阵计算   | fp16/fp32/tf32 | `python matmul_test.py --precision fp16` |
| 卷积计算   | fp16/fp32/tf32 | `python conv_test.py --size 1024` |
| NCCL AllReduce | fp16/fp32 | `nccl-tests/build/all_reduce_perf -b 256M -e 1G -f 2 -g 8` |

3. **结果对比要点**
- 计算密集型任务：比较TFLOPS数值
- 通信密集型任务：比较带宽和延迟
- 能效分析：每瓦特性能（TFLOPS/W）
```python
df["Perf per Watt"] = df["Throughput (TFLOPS)"] / df["Avg Power (W)"]
```

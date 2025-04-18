# 实时推理压力测试指南

## 1. Triton Server 配置指南

### 模型仓库结构示例
```bash
model_repository/
├── resnet50
│   ├── 1
│   │   └── model.plan
│   └── config.pbtxt
├── bert-base
│   ├── 1
│   │   └── model.onnx
│   └── config.pbtxt
└── ensemble_model
    ├── 1
    └── config.pbtxt

# 典型config.pbtxt配置
platform: "tensorrt_plan"
max_batch_size: 128
input [
  {
    name: "input"
    data_type: TYPE_FP32
    dims: [ 3, 224, 224 ]
  }
]
output [
  {
    name: "output"
    data_type: TYPE_FP32
    dims: [ 1000 ]
  }
]
```

### 启动命令
```bash
tritonserver --model-repository=/path/to/model_repository \
             --metrics-config=prometheus,address=0.0.0.0:8002 \
             --allow-metrics=true \
             --log-verbose=1
```

## 2. Vegeta 压力测试脚本

### 测试步骤
```bash
# 安装Vegeta
go install github.com/tsenart/vegeta/v12/cmd/vegeta@latest

# 创建测试目标文件（targets.txt）
POST http://localhost:8000/v2/models/resnet50/infer
Content-Type: application/json
@payload.json

# 执行压测（持续60秒，100 RPS）
vegeta attack -rate=100 -duration=60s -targets=targets.txt | vegeta report

# 生成可视化报告
vegeta attack -rate=100 -duration=60s -targets=targets.txt | \
tee results.bin | \
vegeta plot > plot.html
```

## 3. 延迟分布直方图生成

### Matplotlib 模板
```python
import matplotlib.pyplot as plt
import pandas as pd
import json

with open('vegeta_results.json') as f:
    data = json.load(f)

latencies = [d['latency']/1e6 for d in data]  # 转换为毫秒

plt.figure(figsize=(12, 6))
plt.hist(latencies, bins=50, alpha=0.75, color='steelblue')
plt.title('Request Latency Distribution')
plt.xlabel('Latency (ms)')
plt.ylabel('Request Count')
plt.grid(True, linestyle='--', alpha=0.6)
plt.savefig('latency_distribution.png', dpi=300, bbox_inches='tight')
```

## 4. 熔断机制测试方案

### 测试场景设计
1. 部署熔断规则（示例istio虚拟服务配置）：
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
spec:
  trafficPolicy:
    connectionPool:
      tcp: 
        maxConnections: 100
      http:
        http1MaxPendingRequests: 50
        maxRequestsPerConnection: 10
    outlierDetection:
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 60s
```

### 测试流程
1. 使用vegeta发起超量请求（超过服务容量50%）
2. 监控熔断触发状态：
```promql
# Prometheus查询语句
sum(rate(triton_inference_request_failure_total{reason="UNAVAILABLE"}[1m])) 
by (model)
```

### 关键监控指标
| 指标名称                            | 类型    | 说明                     |
|-----------------------------------|---------|--------------------------|
| triton_inference_request_duration_seconds | Summary | 请求处理耗时分布          |
| process_cpu_seconds_total         | Counter | CPU使用量                |
| triton_request_queue_duration_us  | Summary | 请求队列等待时间          |

## Prometheus 监控配置
```yaml
# prometheus.yml
scrape_configs:
  • job_name: 'triton'
    static_configs:
      ◦ targets: ['triton-host:8002']
    metrics_path: '/metrics'
    
  • job_name: 'application'
    static_configs:
      ◦ targets: ['app-server:9090']
```

# 附录：监控看板建议指标
```promql
# 实时吞吐量
sum(rate(triton_inference_request_success_total[1m])) by (model)

# P99延迟
histogram_quantile(0.99, 
  sum by (le, model) (
    rate(triton_inference_request_duration_seconds_bucket[5m])
  )
)
```
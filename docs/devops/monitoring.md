# 服务器监控方案文档

## 1. Prometheus部署步骤

### 1.1 安装Prometheus
```shell
# 下载二进制包
wget https://github.com/prometheus/prometheus/releases/download/v2.47.2/prometheus-2.47.2.linux-amd64.tar.gz
tar xvfz prometheus-*.tar.gz
mv prometheus-* /opt/prometheus

# 创建systemd服务
cat <<EOF > /etc/systemd/system/prometheus.service
[Unit]
Description=Prometheus Server

[Service]
ExecStart=/opt/prometheus/prometheus \
  --config.file=/opt/prometheus/prometheus.yml \
  --storage.tsdb.path=/prometheus-data \
  --web.enable-lifecycle

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable --now prometheus
```

### 1.2 安装node_exporter
```shell
wget https://github.com/prometheus/node_exporter/releases/download/v1.6.1/node_exporter-1.6.1.linux-amd64.tar.gz
tar xvfz node_exporter-*.tar.gz
mv node_exporter-*/node_exporter /usr/local/bin/

# 创建systemd服务
cat <<EOF > /etc/systemd/system/node_exporter.service
[Unit]
Description=Node Exporter

[Service]
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable --now node_exporter
```

## 2. Grafana看板导入（ID 11074）

1. 访问Grafana页面（默认http://<grafana-server>:3000）
2. 左侧菜单选择"+" > Import
3. 在"Import via grafana.com"输入框填写11074
4. 选择Prometheus数据源
5. 推荐配置：
   - 仪表盘名称：Node Exporter Full
   - 数据源选择：Prometheus（默认）
   - 时间间隔：设置`$interval`变量为`1m`
6. 点击"Import"完成导入

## 3. GPU监控方案（DCGM Exporter）

```shell
# 运行DCGM Exporter容器
docker run -d \
  --gpus all \
  -p 9400:9400 \
  -v /run/prometheus:/run/prometheus \
  -v /sys/kernel/debug/nvidia-uvm:/sys/kernel/debug/nvidia-uvm:ro \
  -v /proc/driver/nvidia/gpus:/proc/driver/nvidia/gpus:ro \
  --device /dev/nvidia-uvm \
  --device /dev/nvidia-uvm-tools \
  --device /dev/nvidiactl \
  --device /dev/nvidia0 \
  nvcr.io/nvidia/k8s/dcgm-exporter:3.3.4-3.1.5-ubuntu22.04
```

Prometheus配置追加：
```yaml
• job_name: 'dcgm-exporter'
  static_configs:
  • targets: ['gpu-host:9400']
```

## 4. 报警规则设置

### 4.1 Prometheus报警规则
```yaml
# alert_rules.yml
groups:
• name: system-alerts
  rules:
  • alert: HighMemoryUsage
    expr: (node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) / node_memory_MemTotal_bytes * 100 > 90
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "High memory usage on {{ $labels.instance }}"
      description: "Memory usage is above 90% for 5 minutes. Current value: {{ $value }}%"
```

### 4.2 Alertmanager配置
```yaml
route:
  group_by: ['alertname']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 1h
  receiver: 'email-alerts'

receivers:
• name: 'email-alerts'
  email_configs:
  • to: 'admin@example.com'
    from: 'alertmanager@example.com'
    smarthost: 'smtp.example.com:587'
    auth_username: 'alertmanager'
    auth_password: 'password'
```

## 5. VictoriaMetrics长期存储配置

### 5.1 安装VictoriaMetrics
```shell
docker run -d -p 8428:8428 \
  -v /victoria-metrics-data:/victoria-metrics-data \
  victoriametrics/victoria-metrics:latest
```

### 5.2 Prometheus远程写入配置
```yaml
# prometheus.yml
remote_write:
  • url: http://victoriametrics:8428/api/v1/write
    queue_config:
      max_samples_per_send: 10000
      capacity: 20000
```

### 5.3 数据保留策略
```shell
# 启动参数（追加到docker run命令）
-retentionPeriod=180d  # 保留180天
-storageDataPath=/victoria-metrics-data
```

# 验证配置
- Prometheus状态页：http://<prometheus-server>:9090
- VictoriaMetrics查询接口：http://<victoria-server>:8428
- Grafana数据源配置需同时包含Prometheus和VictoriaMetrics

# Docker 环境配置指南

## 1. 存储驱动选择建议
### overlay2 存储驱动配置
```bash
# 修改 /etc/docker/daemon.json（需创建）
{
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true",
    "overlay2.size=20G"  # 设置基础设备大小
  ]
}

# 重启服务生效
sudo systemctl restart docker

# 验证配置
docker info | grep -i storage
```

## 2. GPU容器支持配置
### NVIDIA Container Runtime 安装
```bash
# 添加软件源
distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
curl -s -L https://nvidia.github.io/nvidia-container-runtime/$distribution/nvidia-container-runtime.repo | \
  sudo tee /etc/yum.repos.d/nvidia-container-runtime.repo

# 安装组件（适用于 CentOS/RHEL）
sudo yum install -y nvidia-container-runtime

# 配置 Docker（编辑 /etc/docker/daemon.json）
{
  "runtimes": {
    "nvidia": {
      "path": "/usr/bin/nvidia-container-runtime",
      "runtimeArgs": []
    }
  },
  "default-runtime": "nvidia"
}

# 重启服务并验证
sudo systemctl restart docker
docker run --rm --gpus all nvidia/cuda:11.0-base nvidia-smi
```

## 3. 私有仓库HTTPS配置
### 使用 Registry 镜像搭建
```bash
# 生成自签名证书
mkdir -p certs
openssl req -newkey rsa:4096 -nodes -sha256 -keyout certs/domain.key \
  -x509 -days 365 -out certs/domain.crt \
  -subj "/CN=myregistry.example.com" -addext "subjectAltName=DNS:myregistry.example.com"

# 启动 Registry 容器
docker run -d \
  -p 5000:5000 \
  -v $(pwd)/certs:/certs \
  -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt \
  -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key \
  --name registry \
  registry:2

# 客户端配置（/etc/docker/daemon.json）
{
  "insecure-registries": ["myregistry.example.com:5000"]
}
```

## 4. 容器资源限制参数
### 常用限制指令
```bash
# CPU限制（最多使用1.5个CPU核心）
docker run -it --cpus="1.5" ubuntu

# 内存限制（硬限制2GB）
docker run -it --memory="2g" --memory-swap="2g" ubuntu

# GPU设备分配（指定2块GPU）
docker run -it --gpus '"device=0,1"' nvidia/cuda:11.0-base

# 组合限制示例
docker run -d \
  --cpus="2" \
  --memory="4g" \
  --gpus all \
  my-gpu-app
```

## 多阶段构建示例
### Go 应用 Dockerfile
```dockerfile
# 构建阶段
FROM golang:1.19 AS builder
WORKDIR /app
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -o myapp .

# 运行阶段
FROM alpine:3.16
RUN apk add --no-cache ca-certificates
COPY --from=builder /app/myapp /usr/local/bin/
EXPOSE 8080
CMD ["myapp"]
```

### 构建优化技巧
1. 使用 `--mount=type=cache` 缓存包管理目录
2. 分离开发依赖与运行时依赖
3. 使用 `.dockerignore` 排除非必要文件
4. 选择体积小的基础镜像（如 alpine/distroless）

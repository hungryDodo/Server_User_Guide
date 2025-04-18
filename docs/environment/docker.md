## GPU容器支持

### 必要组件安装
```bash
distribution=$(. /etc/os-release;echo $ID$VERSION_ID)
curl -s -L https://nvidia.github.io/nvidia-docker/gpgkey | sudo apt-key add -
curl -s -L https://nvidia.github.io/nvidia-docker/$distribution/nvidia-docker.list | sudo tee /etc/apt/sources.list.d/nvidia-docker.list
```

### 容器运行示例
```bash
docker run --gpus all -it nvidia/cuda:11.8.0-base-ubuntu20.04
```

支持GPU数量限制：
```bash
# 仅使用前4张卡
docker run --gpus '"device=0,1,2,3"' ...
```
## GPU驱动安装指南

### 推荐版本
- 驱动版本: 525.85.12
- CUDA版本: 11.8

### 安装步骤
```bash
# 禁用nouveau驱动
echo "blacklist nouveau" | sudo tee /etc/modprobe.d/blacklist-nvidia-nouveau.conf

# 安装驱动
wget https://us.download.nvidia.com/.../NVIDIA-Linux-x86_64-525.85.12.run
sudo sh NVIDIA-Linux-x86_64-525.85.12.run --silent
```

安装验证：
```bash
nvidia-smi
```
预期输出示例：
![nvidia-smi截图](images/nvidia-smi.png)
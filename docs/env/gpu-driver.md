# NVIDIA GPU 驱动安装与配置指南

## 1. 驱动版本与CUDA版本对应关系矩阵

| CUDA版本  | 最低驱动版本 | 推荐驱动版本 | 支持架构                   |
|-----------|--------------|--------------|----------------------------|
| CUDA 12.x | 535.86.05    | 545.23.08    | Ampere, Ada Lovelace, Hopper |
| CUDA 11.8 | 450.80.02    | 470.223.02   | Turing, Ampere              |
| CUDA 11.0 | 450.36.06    | 450.51.05    | Volta, Turing               |
| CUDA 10.2 | 440.33        | 440.118.02   | Pascal, Volta               |

> 注：完整矩阵参考[NVIDIA官方文档](https://docs.nvidia.com/cuda/cuda-toolkit-release-notes/index.html)

## 2. 旧驱动卸载流程

### 2.1 标准卸载方法（推荐）
```bash
sudo nvidia-uninstall  # 使用官方卸载脚本
sudo /usr/bin/nvidia-uninstall  # 备用路径
```

### 2.2 强制卸载（当标准方法失效时）
```bash
sudo /bin/nvidia-installer --uninstall  # 使用安装程序反向卸载
sudo apt purge nvidia-*  # 适用于Debian/Ubuntu
sudo yum remove nvidia-*  # 适用于RHEL/CentOS
```

### 2.3 深度清理
```bash
sudo rm -rf /usr/lib/xorg/modules/drivers/nvidia_drv.so
sudo rm -rf /etc/X11/xorg.conf
```

## 3. DKMS自动重建配置

### 3.1 安装前准备
```bash
sudo apt install dkms build-essential linux-headers-$(uname -r)  # Debian/Ubuntu
sudo yum install dkms kernel-devel-$(uname -r)  # RHEL/CentOS
```

### 3.2 驱动安装时启用DKMS
```bash
sudo ./NVIDIA-Linux-x86_64-535.86.05.run --dkms -s --no-drm
```

### 3.3 手动注册DKMS模块
```bash
sudo dkms install -m nvidia -v 535.86.05
sudo dkms status  # 验证注册状态
```

## 4. 持久化模式系统服务

### 4.1 创建systemd服务文件
```ini
# /etc/systemd/system/nvidia-persistenced.service
[Unit]
Description=NVIDIA Persistence Daemon

[Service]
Type=forking
ExecStart=/usr/bin/nvidia-persistenced --user root
ExecStopPost=/usr/bin/nvidia-smi -pm 0

[Install]
WantedBy=multi-user.target
```

### 4.2 服务管理命令
```bash
sudo systemctl enable nvidia-persistenced
sudo systemctl start nvidia-persistenced
nvidia-smi -pm 1  # 立即生效
```

## 5. PCIe带宽验证方法

### 5.1 使用内置测试工具
```bash
sudo apt install nvidia-cuda-toolkit  # 安装测试套件
/usr/local/cuda/extras/demo_suite/p2pBandwidthLatencyTest
```

### 5.2 自定义测试脚本
```bash
# test_pcie
#!/bin/bash
DEVICE=$(nvidia-smi --query-gpu=gpu_bus_id --format=csv,noheader | head -n1)
sudo apt install pciutils
lspci -vvv -s $DEVICE | grep LnkSta
nvidia-smi topo -mpci
```

### 5.3 预期输出示例
```
P2P Connectivity: OK
Bandwidth: 24.5 GB/s (PCIe Gen3 x16)
Latency: 1.28 μs
```

## 附录：内核兼容性检查
```bash
sudo dmesg | grep NVRM
journalctl -k | grep NVIDIA
```

> 注意：安装完成后建议执行`nvidia-bug-report.sh`生成完整诊断报告

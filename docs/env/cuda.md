# CUDA Toolkit 安装与配置指南

## 1. 官方仓库配置

```bash
# 添加NVIDIA CUDA仓库（以Ubuntu 22.04为例）
wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2204/x86_64/cuda-ubuntu2204.pin
sudo mv cuda-ubuntu2204.pin /etc/apt/preferences.d/cuda-repository-pin-600
sudo apt-key adv --fetch-keys https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2204/x86_64/3bf863cc.pub
sudo add-apt-repository "deb https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2204/x86_64/ /"

# 安装网络仓库版CUDA
sudo apt-get update
sudo apt-get -y install cuda-toolkit-12-2  # 指定版本号安装
```

## 2. 多版本管理方法

```bash
# 安装不同版本CUDA（示例：CUDA 11.8和12.2）
sudo apt install cuda-11-8 cuda-toolkit-11-8
sudo apt install cuda-12-2 cuda-toolkit-12-2

# 配置符号链接
sudo update-alternatives --install /usr/local/cuda cuda /usr/local/cuda-11.8 118
sudo update-alternatives --install /usr/local/cuda cuda /usr/local/cuda-12.2 122

# 切换版本（交互式选择）
sudo update-alternatives --config cuda
```

## 3. 编译环境验证

```bash
# 编译示例程序
cd /usr/local/cuda/samples/1_Utilities/deviceQuery
sudo make
./deviceQuery

# 期望输出最后显示 Result = PASS
# 带宽测试验证
cd ../bandwidthTest
sudo make
./bandwidthTest
```

## 4. 兼容性检查表

CUDA版本 | 支持的最高GCC版本
---|---
CUDA 12.x | GCC 11
CUDA 11.x | GCC 9
CUDA 10.2 | GCC 8
CUDA 9.2  | GCC 7

## cuDNN安装步骤（tar包方式）

```bash
# 解压cuDNN tar包（需提前下载）
tar -xzvf cudnn-linux-x86_64-8.9.5.29_cuda12-archive.tar.xz

# 复制文件到CUDA目录
sudo cp cudnn-*-archive/include/cudnn*.h /usr/local/cuda/include/
sudo cp -P cudnn-*-archive/lib/libcudnn* /usr/local/cuda/lib64/
sudo chmod a+r /usr/local/cuda/include/cudnn*.h /usr/local/cuda/lib64/libcudnn*

# 验证安装
cat /usr/local/cuda/include/cudnn_version.h | grep CUDNN_MAJOR -A 2
```

> 注意：所有路径中的版本号需根据实际下载文件修改，建议优先使用NVIDIA官方文档验证最新版本兼容性

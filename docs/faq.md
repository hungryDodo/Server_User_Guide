## 高频问题解答

### Q1: 无法检测到GPU
1. 检查驱动状态：`systemctl status nvidia-driver`
2. 验证PCIe连接：`lspci | grep NVIDIA`
3. 查看内核日志：`dmesg | grep -i nvidia`

### Q2: CUDA版本冲突
使用conda隔离环境：
```bash
conda create -n pytorch_env cudatoolkit=11.8
```
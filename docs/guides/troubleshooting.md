# 服务器故障排查手册

## 1. GPU显存不足处理流程

### 错误日志样例
```
RuntimeError: CUDA out of memory. 
Tried to allocate 4.00 GiB (GPU 0; 23.70 GiB total capacity; 15.21 GiB already allocated; 3.94 GiB free; 17.21 GiB reserved in total by PyTorch)
```

### 处理步骤
1. 查看GPU显存占用情况
```bash
nvidia-smi --query-gpu=index,memory.used,memory.total --format=csv
```

2. 找到占用显存的进程
```bash
sudo fuser -v /dev/nvidia*  # 列出所有GPU相关进程
```

3. 终止异常进程（示例PID为1234）
```bash
kill -9 1234  # 强制终止指定进程
```

4. 预防性措施
```bash
watch -n 1 nvidia-smi  # 实时监控显存变化
```

### 验证测试
```bash
python -c "import torch; x = torch.randn(1024, 1024).cuda()"  # 测试显存分配
```

---

## 2. NCCL通信错误解决方案

### 错误日志样例
```
NCCL error in: ../torch/csrc/distributed/c10d/ProcessGroupNCCL.cpp:825, 
internal error, NCCL version 2.7.8
```

### 处理流程
1. 检查InfiniBand网卡状态
```bash
ibstat         # 查看IB卡基本信息
ibstatus       # 检查端口物理状态
iblinkinfo     # 查看链路连接质量
```

2. 验证NCCL通信
```bash
nccl-tests/build/all_reduce_perf -b 8 -e 256M -f 2 -g 2  # 测试通信带宽
```

3. 系统级检查
```bash
systemctl status opensm    # 检查IB子网管理器
service rdma status        # 验证RDMA服务状态
```

### 验证测试
```bash
# 运行PyTorch分布式测试
python -c "import torch.distributed as dist; dist.init_process_group('nccl')"
```

---

## 3. 磁盘空间清理指南

### 空间不足报错
```
No space left on device
```

### 清理步骤
1. 查找大文件（>1GB）
```bash
find / -type f -size +1G -exec ls -lh {} \; 2>/dev/null
```

2. 按修改时间清理（示例清理7天前日志）
```bash
find /var/log -name "*.log" -mtime +7 -exec rm -f {} \;
```

3. 查看目录大小
```bash
du -h --max-depth=1 /home  # 分析用户目录空间占用
```

### 验证命令
```bash
df -h  # 查看清理后的磁盘空间
```

---

## 4. 权限问题修复步骤

### 典型错误
```
Permission denied: '/data/input_file'
```

### 修复流程
1. 递归修改目录权限
```bash
chmod -R 755 /data/experiment  # 设置目录可执行权限
```

2. 修改文件所有权
```bash
chown -R user:group /data/models  # 修改属主和属组
```

3. 特殊权限处理
```bash
setfacl -R -m u:username:rwx /shared_dir  # 设置ACL权限
```

### 验证方法
```bash
ls -l /data/experiment  # 查看权限变更
sudo -u testuser cat /data/file  # 模拟用户访问
```

> 注意：使用`chmod 777`需谨慎，建议优先使用精确权限设置

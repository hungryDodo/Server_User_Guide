# 实验室服务器常见问题解答 (FAQ)

## 基础操作类

### Q1: 如何连接到服务器？

**操作步骤**：

```bash
ssh 用户名@服务器IP地址 -p 端口号
# 示例：ssh zhangsan@192.168.1.100 -p 2222
```

**注意事项**：

- 首次连接需输入密码
- 建议后续[配置SSH密钥登录](../configuration/base-env.md)

---

### Q2: 传输文件到服务器的方法？

**推荐工具**：

- **命令行传输**：

```bash
scp -P 2222 本地文件路径 zhangsan@服务器IP:目标路径
```

- **可视化工具**：FileZilla/WinSCP（协议选SFTP，端口2222）

---

### Q3: 提示「权限被拒绝」怎么办？

**解决方法**：

```bash
# 1. 检查文件权限
ls -l 文件名

# 2. 添加执行权限
chmod +x 文件名

# 3. 需要管理员权限时
sudo 命令
```

---

## GPU使用类

### Q4: 运行程序提示「CUDA不可用」？

**检查步骤**：

1. 验证驱动状态：

```bash
nvidia-smi  # 应有GPU状态输出
```

2. 检查CUDA版本：

```bash
nvcc --version
```

3. 确认PyTorch/TensorFlow版本匹配（[版本对照表](../environment/conda.md)）

---

### Q5: 出现「CUDA out of memory」错误？

**解决方案**：

1. 检查显存占用：

```bash
watch -n 1 nvidia-smi
```

2. 释放残留进程：

```bash
kill -9 $(ps -ef | grep python | awk '{print $2}')
```

3. 代码调整：

```python
torch.cuda.empty_cache()  # PyTorch清理缓存
```

---

### Q6: 多卡训练如何指定GPU？

**代码示例**：

```python
import os
os.environ["CUDA_VISIBLE_DEVICES"] = "0,1"  # 使用前两张卡
```

**命令行方式**：

```bash
CUDA_VISIBLE_DEVICES=0,1 python train.py
```

---

## 环境配置类

### Q7: Conda环境激活失败？

**解决方法**：

```bash
# 1. 初始化conda
source ~/miniconda3/bin/activate

# 2. 更新conda
conda update -n base -c defaults conda

# 3. 常见错误修复
conda init bash  # 重写shell配置
```

---

### Q8: 如何安装指定版本的包？

**语法示例**：

```bash
conda install pytorch==2.0.1 
pip install tensorflow==2.12.0 --user
```

**版本查询**：

```bash
conda search pytorch  # Conda可用版本
pip index versions tensorflow  # Pip可用版本
```

---

### Q9: Docker命令需要sudo怎么办？
**永久解决**：

```bash
sudo usermod -aG docker $USER
newgrp docker  # 立即生效
```

**验证**：

```bash
docker run hello-world  # 不应提示权限错误
```

---

## 系统管理类

### Q10: 磁盘空间不足怎么办？

**排查步骤**：

1. 查看磁盘使用：

```bash
df -h  # 查看各分区
du -sh * | sort -h  # 定位大文件
```

2. 清理缓存：

```bash
sudo apt clean  # 清理包缓存
docker system prune  # 清理Docker缓存
```

---

### Q11: 任务突然中断如何恢复？

**建议方案**：

1. 使用`tmux`会话保护：

```bash
tmux new -s mytask  # 新建会话
Ctrl+b d            # 暂时退出
tmux attach -t mytask  # 重新连接
```

2. 检查系统日志：

```bash
journalctl -u service_name --since "10分钟前"
```

---

### Q12: 如何查看CPU/内存使用？

**常用命令**：

```bash
htop         # 交互式资源监控
free -h      # 内存使用情况
nvidia-smi   # GPU状态
```

---

## 网络问题类

### Q13: 无法连接服务器怎么办？

**排查步骤**：

1. 检查本地网络：

```bash
ping 服务器IP
```

2. 检查防火墙：

```bash
sudo ufw status  # 查看防火墙规则
```

3. 联系管理员确认服务器状态

---

### Q14: 下载速度慢如何加速？

**镜像源配置**：

1. Conda清华源：

```bash
conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/main
```

2. Pip阿里云源：

```bash
pip install 包名 -i https://mirrors.aliyun.com/pypi/simple
```

---

## 其他实用技巧

### Q15: 如何后台运行程序？

**nohup方式**：

```bash
nohup python train.py > log.txt 2>&1 &
```

**tmux方式**：

```bash
tmux new -s myjob
# 运行程序后按Ctrl+b d退出
```

---

### Q16: 历史命令如何复用？

**快捷键**：

- ↑/↓ 键：浏览历史命令
- Ctrl+r：反向搜索历史命令
- !命令编号：快速执行历史命令

---

### Q17: 如何解压文件？

**常见格式**：

```bash
tar -xzvf file.tar.gz      # .tar.gz
unzip file.zip             # .zip
7z x file.7z               # .7z
```

---

**遇到未列出的问题？**  

请联系实验室管理员：  

- 张三：zhangsan@lab.com 📞138-1234-5678  
- 紧急支持：🕒24小时值班电话 400-800-1234

---

> 提示：更多详细配置请参考[服务器使用完整手册](../index.md)

# VSCode 远程开发全链路配置指南

## 一、Remote-SSH 插件配置
### 多主机管理配置
1. 安装 Remote-SSH 扩展（ID: ms-vscode-remote.remote-ssh）
2. 创建 SSH 配置文件：
```bash
mkdir -p ~/.ssh
vim ~/.ssh/config
```

```config
# 基础配置模板
Host dev-server
    HostName 192.168.1.100
    User ubuntu
    IdentityFile ~/.ssh/id_rsa

# 带跳板机配置
Host production
    HostName 10.10.8.20
    User root
    ProxyCommand ssh -W %h:%p jump-server

Host jump-server
    HostName gateway.example.com
    User admin
```

3. 快速连接技巧：
   • 右键主机选择「Connect to Host in New Window」
   • 使用命令面板（Ctrl+Shift+P）执行「Remote-SSH: Connect to Host」

▶️ 故障排查：若出现`Could not establish connection»`，检查`sshd_config`中`AllowTcpForwarding`是否为yes

---

## 二、Python Conda 环境配置
### launch.json 配置模板
```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Python: Conda Env",
            "type": "python",
            "request": "launch",
            "program": "${file}",
            "python": "/opt/miniconda3/envs/myenv/bin/python",
            "console": "integratedTerminal",
            "preLaunchTask": "conda activate myenv"
        }
    ]
}
```

### 环境选择方案
1. **自动识别**：安装 Python 扩展后，状态栏选择解释器路径
2. **手动指定**：
   ```bash
   # 在集成终端执行
   conda init bash
   source ~/.bashrc
   conda activate myenv
   ```

▶️ 注意：若环境未正常识别，在`.bashrc`中添加：
```bash
export PATH="/opt/miniconda3/bin:$PATH"
```

---

## 三、GPU 监控与 NSight 集成
### NVIDIA 插件配置
1. 安装扩展：
   • NVIDIA NSight (nsight-vscode-edition)
   • Nsight Visual Studio Code Edition

2. 配置监控面板：
```json
"nvidia.nsight-vscode-edition.options": {
    "cuda.visibleDevices": "0,1",
    "metrics.interval": 2000
}
```

### 实时监控命令
```bash
watch -n 1 nvidia-smi
```

▶️ 高级分析：通过 NSight 启动性能分析会话，需在`.vscode/settings.json`中配置CUDA路径

---

## 四、协作开发与权限控制
### Live Share 安全配置
1. 安装 Live Share 扩展包（ms-vsliveshare.vsliveshare）
2. 创建会话时设置：
   ```json
   "liveshare.guestApprovalRequired": true,
   "liveshare.commandPalette.visibility": "collaborators"
   ```

3. 权限分级控制：
   • `/controlGuestDebugging` 限制调试权限
   • `/limited` 禁止文件下载
   • `/hide` 隐藏敏感文件

▶️ 安全建议：定期使用`vsls logout`重置身份令牌

---

## 五、Jupyter 内核连接解决方案
### 内核绑定方法
1. 内核选择器指定路径：
   ```python
   import sys
   sys.executable  # 验证当前内核路径
   ```

2. 内核配置文件创建：
   ```bash
   python -m ipykernel install --user --name myenv --display-name "Python (MyEnv)"
   ```

3. 强制内核配置：
   ```json
   "jupyter.notebookFileRoot": "/remote/path",
   "python.defaultInterpreterPath": "/opt/conda/envs/ml/bin/python"
   ```

### 典型故障处理
| 现象 | 解决方案 |
|------|----------|
| Kernel not starting | 执行 `pip install --upgrade ipykernel` |
| No module in remote | 在SSH终端运行 `conda install pandas` |
| 内核超时 | 设置 `"jupyter.launchTimeout": 60` |

---

> 配置验证工具：使用`Remote Explorer`面板检查连接状态，通过`Python: Select Interpreter`命令验证环境绑定效果。建议在`.devcontainer`中固化开发环境配置。
# PyCharm Professional 远程开发配置指南

![PyCharm Professional Logo](https://resources.jetbrains.com/storage/products/pycharm/img/meta/pycharm_logo_300x300.png)

> 测试环境：PyCharm 2023.2.3 (Professional Edition)

## 一、远程解释器配置（SSH认证）

### 1.1 创建SSH密钥对（本地终端执行）
```bash
ssh-keygen -t rsa -b 4096 -C "pycharm_remote"  # 生成密钥对
cat ~/.ssh/id_rsa.pub | ssh user@remote_host "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"  # 公钥上传
```

### 1.2 PyCharm配置步骤
1. **File > Settings > Project: <名称> > Python Interpreter**
2. 点击⚙图标选择 **Add New Interpreter > On SSH**
3. 填写远程服务器信息：
   - Host：`your-remote-host.com`
   - Port：`22`
   - Username：`your_username`
   - Authentication type：`Key pair`
   - Private key file：选择本地`id_rsa`文件
   ![SSH认证配置截图](pycharm-ssh-auth.png)

## 二、代码同步排除规则

### 2.1 配置部署排除路径
1. **Tools > Deployment > Configuration**
2. 选择对应SSH连接 → **Mappings**标签页
3. 在`Excluded Paths`添加：
   ```text
   .idea/
   */__pycache__/
   *.log
   *.tmp
   ```
   
### 2.2 .gitignore补充配置（可选）
```gitignore
# PyCharm工程文件
.idea/
*.iml
*.ipr
```

## 三、调试器端口映射

### 3.1 分布式调试配置
1. **Run > Edit Configurations**
2. 添加`Python Debug Server`配置：
   ```properties
   Host：0.0.0.0
   Port：12345  # 需与远程代码调试端口一致
   ```
3. 远程代码添加调试器初始化：
```python
import pydevd_pycharm
pydevd_pycharm.settrace('localhost', port=12345, stdoutToServer=True, stderrToServer=True)
```

### 3.2 防火墙规则（Ubuntu示例）
```bash
sudo ufw allow 12345/tcp
sudo ufw reload
```

## 四、性能优化参数

### 4.1 修改内存分配（pycharm.vmoptions）
文件路径：`/Applications/PyCharm.app/Contents/bin/pycharm.vmoptions` (MacOS)

```properties
-Xms2048m
-Xmx4096m
-XX:ReservedCodeCacheSize=1024m
```

### 4.2 禁用非必要插件
**File > Settings > Plugins** 禁用：
- CVS Integration
- Copyright
- Terminal插件（使用系统终端替代）

## 附录：专业版离线激活方法

### 合法授权声明
> 建议通过JetBrains官网购买正版授权。以下方法仅限学习研究使用，请于24小时内删除。

1. 下载激活补丁包：[ja-netfilter.zip](https://github.com/ja-netfilter/ja-netfilter)
2. 解压到非中文路径：`C:\ja-netfilter`
3. 修改`pycharm.vmoptions`：
```properties
-javaagent:C:/ja-netfilter/ja-netfilter.jar
```
4. 启动PyCharm输入激活码：
```text
5AXR6T6JEY-eyJsaWNlbnNlSWQiOiI1QVhSNlQ2SkVZIiwibGljZW5zZWVOYW1lIjoi5rC45LmF5r+A5rS7IHd3d8K3YWppaHVvwrdjb20iLCJhc3NpZ25lZU5hbWUiOiIiLCJhc3NpZ25lZUVtYWlsIjoiIiwibGljZW5zZVJlc3RyaWN0aW9uIjoiIiwiY2hlY2tDb25jdXJyZW50VXNlIjpmYWxzZSwicHJvZHVjdHMiOlt7ImNvZGUiOiJJSSIsInBhaWRVcFRvIjoiMjAyNC0wMy0wNyJ9LHsiY29kZSI6IkFDIiwicGFpZFVwVG8iOiIyMDI0LTAzLTA3In0seyJjb2RlIjoiRFBOIiwicGFpZFVwVG8iOiIyMDI0LTAzLTA3In0seyJjb2RlIjoiUFMiLCJwYWlkVXBUbyI6IjIwMjQtMDMtMDcifSx7ImNvZGUiOiJHTyIsInBhaWRVcFRvIjoiMjAyNC0wMy0wNyJ9LHsiY29kZSI6Ik1DQSIsInBhaWRVcFRvIjoiMjAyNC0wMy0wNyJ9XSwiaGFzaCI6IjEzOTg4NDY3LzAiLCJncmFjZVBlcmlvZERheXMiOjcsImF1dG9Qcm9sb25nYXRlZCI6ZmFsc2UsImlzQXV0b1Byb2xvbmdhdGVkIjpmYWxzZX0=
```

[^建议]: 配置完成后建议重启PyCharm使所有设置生效

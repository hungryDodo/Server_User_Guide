# TigerVNC 服务端配置与优化指南

## 1. TigerVNC 服务端安装与配置
### 安装步骤（Linux）
```bash
# Debian/Ubuntu
sudo apt install tigervnc-standalone-server tigervnc-xorg-extension

# RHEL/CentOS
sudo yum install tigervnc-server
```

### Systemd 服务文件配置
创建 `/etc/systemd/system/vncserver@.service`：
```ini
[Unit]
Description=Remote desktop service (VNC)
After=syslog.target network.target

[Service]
Type=forking
User=%i
ExecStartPre=/bin/sh -c '/usr/bin/vncserver -kill %i > /dev/null 2>&1 || :'
ExecStart=/usr/bin/vncserver -geometry 1920x1080 -localhost no -SecurityTypes X509Vnc
ExecStop=/usr/bin/vncserver -kill %i

[Install]
WantedBy=multi-user.target
```

启用并启动服务：
```bash
sudo systemctl daemon-reload
sudo systemctl enable vncserver@1.service
sudo systemctl start vncserver@1.service
```

---

## 2. 分辨率配置方法
修改配置文件 `~/.vnc/config`：
```ini
geometry=1920x1080
depth=24
```

参数说明：
• `geometry=WidthxHeight` 设置初始分辨率（支持动态调整）
• 常用分辨率示例：
  • `1280x720`  720p
  • `1920x1080` 1080p
  • `2560x1440` 2K
  • `3840x2160` 4K

---

## 3. 连接加密设置（X509证书）
### 证书生成流程
```bash
# 生成CA私钥
openssl genrsa -out ca.key 4096

# 创建CA证书（有效期10年）
openssl req -x509 -new -nodes -key ca.key -sha256 -days 3650 -out ca.crt \
  -subj "/CN=VNC-CA/O=My Organization"

# 生成服务器私钥
openssl genrsa -out server.key 4096

# 创建证书签名请求
openssl req -new -key server.key -out server.csr \
  -subj "/CN=my-server.example.com/O=My Organization"

# 签发服务器证书
openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
  -out server.crt -days 3650 -sha256

# 合并证书链
cat server.crt ca.crt > combined.crt
```

配置VNC使用证书：
```ini
# 在 ~/.vnc/config 中添加
X509Cert=/path/to/combined.crt
X509Key=/path/to/server.key
SecurityTypes=X509Vnc
```

---

## 4. 多用户管理策略
### 端口分配规则
| 用户    | 显示编号 | 对应端口 |
|---------|----------|----------|
| user1   | :1       | 5901     |
| user2   | :2       | 5902     |
| user3   | :3       | 5903     |

配置示例（为不同用户创建独立服务）：
```bash
# 创建用户专用服务文件
sudo cp /etc/systemd/system/vncserver@.service /etc/systemd/system/vncserver@:2.service

# 修改服务文件中的启动参数
ExecStart=/usr/bin/vncserver -geometry 1600x900 -localhost no -SecurityTypes X509Vnc -rfbport 5902
```

---

## 低延迟优化配置
### 关闭合成器设置
在 `~/.vnc/xstartup` 中添加：
```bash
# 禁用桌面特效
xfconf-query -c xfwm4 -p /general/use_compositing -s false

# 停止合成管理器
pkill compton || true
```

### 附加优化参数
```ini
# ~/.vnc/config
dpi=96
desktop=sandbox
alwaysshared
httpdir=/usr/share/vnc/classes
```

### 编码参数调整
```bash
vncserver -xstartup /etc/X11/xinit/xinitrc \
  -noxdamage \
  -audit 0 \
  -encoding tight -quality 9 \
  -compresslevel 6
```

---

> **注意事项**：
> 1. 防火墙需开放对应端口（5900+显示编号）
> 2. 建议配合SSH隧道增强安全性：`ssh -L 5901:localhost:5901 user@server`
> 3. 证书有效期结束后需重新签发
> 4. 多用户场景建议配置独立的SELinux策略
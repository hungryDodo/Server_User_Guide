# SSH连接优化与配置指南

## 1. 密钥对生成（ed25519算法）
```shell
# 生成ED25519密钥对（安全性优于RSA）
ssh-keygen -t ed25519 -C "your_comment" -f ~/.ssh/id_ed25519

# 参数说明：
# -t ed25519 : 使用椭圆曲线加密算法，密钥长度256位
# -C         : 添加密钥注释（通常为邮箱/用途标识）
# -f         : 指定密钥存储路径

# 设置密钥权限
chmod 600 ~/.ssh/id_ed25519
chmod 644 ~/.ssh/id_ed25519.pub
```

## 2. 连接保持配置
**客户端配置（~/.ssh/config）**：
```config
Host *
    ServerAliveInterval 60    # 每60秒发送心跳包
    ServerAliveCountMax 3     # 连续3次无响应则断开
    TCPKeepAlive yes          # 启用TCP层保活
```

## 3. 端口转发示例
| 参数 | 命令示例 | 应用场景 |
|------|----------|----------|
| `-L` | `ssh -L 8080:localhost:80 user@gateway` | 本地访问内网Web服务 |
| `-R` | `ssh -R 3306:localhost:3306 user@public-server` | 暴露内网数据库到公网 |
| `-D` | `ssh -D 1080 user@proxy-server` | 创建SOCKS5代理隧道 |

## 4. 安全加固配置
修改`/etc/ssh/sshd_config`：
```ini
PasswordAuthentication no         # 禁用密码登录
ChallengeResponseAuthentication no
PermitRootLogin no
PubkeyAuthentication yes         # 强制密钥认证

# 重启服务生效
sudo systemctl restart sshd
```

## 5. ProxyJump多级跳转模板
```config
# ~/.ssh/config
Host bastion
    HostName 203.0.113.10
    User jump_user
    Port 2222
    IdentityFile ~/.ssh/id_bastion

Host internal-server
    HostName 10.8.0.100
    User app_user
    ProxyJump bastion
    IdentityFile ~/.ssh/id_app

# 使用方式：ssh internal-server
```

## 高级技巧
- 复用连接：`ControlMaster auto` + `ControlPath` 配置
- 端口敲门：`knock <host> 1000 2000 3000`
- 证书认证：结合`ssh-keygen -s`生成CA签名证书

> 注意：生产环境建议配置两步验证(2FA)和防火墙白名单策略

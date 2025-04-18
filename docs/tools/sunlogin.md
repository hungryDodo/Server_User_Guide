# 向日葵远程控制内网穿透指南

---

## 目录
1. [客户端安装](#客户端安装)  
2. [设备绑定流程](#设备绑定流程)  
3. [安全策略配置](#安全策略配置)  
4. [故障诊断方法](#故障诊断方法)  
5. [ARM版本兼容性说明](#arm版本兼容性说明)  

---

## 客户端安装

### 下载链接与安装命令
- **Linux x86_64**  
  ```bash
  wget https://down.oray.com/sunlogin/linux/sunloginclient-11.0.1-amd64.deb
  sha256sum sunloginclient-11.0.1-amd64.deb  # 校验码: a1b2c3d4e5...
  sudo dpkg -i sunloginclient-11.0.1-amd64.deb
  ```

- **ARM版本**  
  ```bash
  wget https://down.oray.com/sunlogin/linux/sunloginclient-11.0.1-arm64.deb
  sha256sum sunloginclient-11.0.1-arm64.deb  # 校验码: f6g7h8i9j0...
  sudo dpkg -i sunloginclient-11.0.1-arm64.deb
  ```

---

## 设备绑定流程

### SN码注册方法
1. **获取SN码**  
   - 打开向日葵客户端 → **设备列表** → **本机信息** → 查看12位SN码。
   - 或通过命令行获取：  
     ```bash
     cat /usr/local/sunlogin/etc/sunlogin_linux.conf | grep sn
     ```

2. **绑定设备**  
   - 登录向日葵官网 [Oray](https://sunlogin.oray.com) → **控制台** → **设备管理** → **添加设备** → 输入SN码完成绑定。

---

## 安全策略配置

### IP白名单设置
1. 登录向日葵控制台 → **安全设置** → **IP白名单**。
2. 点击 **添加规则**，输入允许访问的IP或IP段（如 `192.168.1.0/24`）。
3. 保存后仅白名单IP可访问设备。

---

## 故障诊断方法

### 日志文件路径
```bash
tail -f /usr/local/sunlogin/log.txt
```

### 常见错误码
| 错误码 | 含义                  | 解决方案                |
|--------|-----------------------|-------------------------|
| 1001   | 连接服务器失败        | 检查网络或防火墙设置    |
| 1003   | 认证超时              | 重新登录账号           |
| 2005   | 端口被占用            | 更换端口或终止占用进程  |
| 3007   | 设备未绑定            | 检查SN码绑定状态       |

---

## ARM版本兼容性说明

1. **支持架构**  
   - 向日葵客户端兼容ARMv8（aarch64）及以上架构，适用于树莓派、NVIDIA Jetson等设备。

2. **注意事项**  
   - 安装前确保系统已安装依赖库：  
     ```bash
     sudo apt-get install libgtk2.0-0 libxtst6 libxss1
     ```
   - ARM版本功能与x86版本一致，但性能可能受硬件限制。

---

> **提示**：访问 [向日葵官网](https://sunlogin.oray.com) 获取最新版本和文档支持。

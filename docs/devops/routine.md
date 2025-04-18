# 日常维护检查清单

## 1. 硬件巡检项目

### GPU温度检查
```bash
# 查看GPU温度（NVIDIA显卡）
nvidia-smi --query-gpu=temperature.gpu --format=csv,noheader

# 设置报警阈值（通常阈值范围）
• 警告阈值：80°C 
• 紧急阈值：90°C
建议通过监控系统（如Prometheus）配置阈值告警

# 带温度检测的IPMI命令（服务器整机）
ipmitool sensor list | grep "Temp"
```

### RAID阵列健康检查
```bash
# 软件RAID检查
sudo mdadm --detail /dev/md0 | grep -i 'state'

# 硬件RAID检查（适用MegaRAID）
sudo megacli -LDInfo -LAll -aAll | grep 'State'

# 硬盘SMART状态检测
sudo smartctl -a /dev/sda | grep 'SMART overall-health'
```

---

## 2. 日志轮转配置

### logrotate配置示例
```conf
# /etc/logrotate.d/myapp
/var/log/myapp/*.log {
    daily
    missingok
    rotate 30
    compress
    delaycompress
    notifempty
    create 0640 www-data adm
    sharedscripts
    postrotate
        systemctl reload myapp.service > /dev/null
    endscript
}
```

### Cron定时任务
```bash
# 每天凌晨执行日志轮转
0 0 * * * /usr/sbin/logrotate /etc/logrotate.conf
```

---

## 3. 安全更新流程

### 无人值守更新配置
```conf
# /etc/apt/apt.conf.d/50unattended-upgrades
Unattended-Upgrade::Allowed-Origins {
    "${distro_id}:${distro_codename}-security";
};
Unattended-Upgrade::AutoFixInterruptedDpkg "true";
Unattended-Upgrade::MinimalSteps "true";
Unattended-Upgrade::Remove-Unused-Dependencies "true";
Unattended-Upgrade::Automatic-Reboot "true";
```

### 手动验证更新
```bash
sudo unattended-upgrade --dry-run --debug
```

---

## 4. 用户磁盘用量统计脚本

### 磁盘统计命令
```bash
# 排除缓存检测（使用--exclude参数）
du -sh --exclude=*/cache/* /home/* 2>/dev/null

# 进阶版（统计前10大用户）
find /home -type d -maxdepth 1 -exec du -sh --exclude=*/cache/* {} \; 2>/dev/null | sort -hr | head -n10
```

### 定时统计脚本
```bash
#!/bin/bash
REPORT_FILE="/var/log/disk_usage_$(date +%Y%m%d).log"
echo "磁盘用量报告 $(date)" > $REPORT_FILE
du -sh --exclude=*/cache/* /home/* 2>/dev/null >> $REPORT_DATE
```

# 每周一凌晨执行
0 0 * * 1 /path/to/script.sh

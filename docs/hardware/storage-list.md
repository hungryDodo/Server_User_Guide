# 存储设备清单文档

---

## 1. HDD/SSD 详细参数

| 设备位置 | 类型 | 品牌/型号 | 容量 | 接口类型 | 最大读取IOPS | 最大写入IOPS | SMART健康状态 |
|----------|------|-----------|------|----------|--------------|--------------|---------------|
| /dev/sda | SSD  | Samsung 870 EVO | 1TB | SATA III | 98K          | 88K          | PASSED        |
| /dev/sdb | HDD  | Seagate Exos X18 | 8TB | SAS 12G  | 220          | 280          | PASSED        |
| /dev/nvme0n1 | NVMe | WD Black SN850 | 2TB | PCIe 4.0 | 700K         | 1M           | PASSED        |

---

## 2. RAID卡配置信息

**RAID卡型号**: LSI MegaRAID 9460-8i  
**缓存配置**:
```bash
# 查看缓存大小
sudo MegaCli -AdpAllInfo -aALL | grep "Memory Size"
▶ 输出: Memory Size: 4096MB

# 电池状态检查
sudo MegaCli -AdpBbuCmd -GetBbuStatus -aALL | grep "Battery State"
▶ 输出: Battery State: Optimal
```
**当前RAID级别**: RAID-10 (VD0: 4x8TB HDD)  
**缓存模式**: WriteBack (带电池保护)

---

## 3. 文件系统统计

```bash
# 获取inode使用情况
df -i | awk '{print $1,$2,$3,$4,$5,$6}'
```
| 挂载点       | 总inode数 | 已用inode | 可用inode | 使用率 |
|--------------|-----------|-----------|-----------|--------|
| /dev/sda1    | 6.5M      | 1.2M      | 5.3M      | 18%    |
| /dev/mapper/raid_vg0 | 25M       | 18M       | 7M        | 72%    |
| /dev/nvme0n1p1 | 4.8M     | 0.9M      | 3.9M      | 19%    |

---

## 4. 坏盘更换流程

### 步骤1 - 确认故障盘
```bash
sudo smartctl -H /dev/sdX  # 检查健康状态
sudo megacli -PDList -aALL | grep "Firmware state"  # 查看RAID状态
```

### 步骤2 - 触发定位LED
```bash
# 启用定位灯（需安装sas2ircu工具）
sudo sas2ircu 0 locate 5:2 ON  # 控制器0，Enclosure 5 Slot 2

# 更换后关闭定位灯
sudo sas2ircu 0 locate 5:2 OFF
```

### 步骤3 - 物理更换
1. 根据LED灯定位故障盘槽位
2. 热插拔更换磁盘
3. 等待RAID自动重建 (`sudo megacli -LDRecon -Start -r5 -L0 -a0`)

---

## 5. SMART健康检查定时任务

```cron
# 每日凌晨检查所有磁盘健康状态
0 2 * * * root /usr/sbin/smartctl --scan | awk '{print $1}' | xargs -I {} /usr/sbin/smartctl -H {} | grep "RESULT" > /var/log/smart_status.log

# 邮件告警配置（需配置mailutils）
0 3 * * * root grep -q "FAIL" /var/log/smart_status.log && mail -s "SMART Alert" admin@example.com < /var/log/smart_status.log
```

---

> **注意事项**：  
> 1. RAID卡电池需每3年强制校准：`sudo MegaCli -AdpBbuCmd -BbuLearn -a0`  
> 2. inode使用率超过90%需进行文件系统清理  
> 3. 企业级SSD需定期执行`blkdiscard`维护

# 存储系统配置指南

---

## 1. LVM扩展操作步骤

### 1.1 扩展卷组（VG）
```bash
# 查看可用磁盘
lsblk

# 创建物理卷（PV）
pvcreate /dev/sdX

# 扩展卷组（将新PV加入VG）
vgextend <vg_name> /dev/sdX

# 验证VG容量
vgdisplay <vg_name>

### 1.2 扩展逻辑卷（LV）
```bash
# 扩展逻辑卷（+10G或100%FREE）
lvextend -L +10G /dev/<vg_name>/<lv_name>
# 或
lvextend -l +100%FREE /dev/<vg_name>/<lv_name>

# 调整文件系统
# 对于ext4：
resize2fs /dev/<vg_name>/<lv_name>
# 对于xfs：
xfs_growfs /mount/point

---

## 2. NFS服务端配置

### 2.1 `/etc/exports` 文件参数
```bash
# 示例配置
/share 192.168.1.0/24(rw,sync,no_subtree_check,no_root_squash)

| 参数              | 说明                                                                 |
|-------------------|----------------------------------------------------------------------|
| rw                | 读写权限                                                             |
| ro                | 只读权限                                                             |
| sync              | 同步写入模式（数据安全优先）                                         |
| async             | 异步写入模式（性能优先）                                             |
| no_root_squash    | 允许root用户访问                                                     |
| subtree_check     | 验证子树完整性（可能影响性能）                                       |

### 2.2 应用配置
```bash
# 安装NFS服务（Ubuntu）
apt install nfs-kernel-server

# 重载配置
exportfs -ra
systemctl restart nfs-server

---

## 3. 磁盘性能测试（fio）

### 3.1 顺序读测试
```bash
fio --name=seq_read --ioengine=libaio --direct=1 --bs=1M --iodepth=64 \
--rw=read --numjobs=4 --size=1G --runtime=60 --time_based --group_reporting

### 3.2 随机写测试
```bash
fio --name=rand_write --ioengine=libaio --direct=1 --bs=4k --iodepth=256 \
--rw=randwrite --numjobs=4 --size=4G --runtime=300 --time_based --group_reporting

---

## 4. S3对象存储挂载（s3fs-fuse）

### 4.1 安装配置
```bash
# Ubuntu安装
apt install s3fs

# 创建认证文件
echo <access_key>:<secret_key> > ~/.passwd-s3fs
chmod 600 ~/.passwd-s3fs

### 4.2 挂载存储
```bash
s3fs <bucket_name> /mnt/s3 \
-o passwd_file=~/.passwd-s3fs \
-o url=https://s3.<region>.amazonaws.com \
-o use_path_request_style \
-o allow_other

---

## 5. ZFS快照自动化脚本

### 5.1 快照脚本（`/usr/local/bin/zfs_snapshot.sh`）
```bash
#!/bin/bash
POOL="zpool1"
DATASET="data"
RETENTION_DAYS=7

# 创建快照
SNAPSHOT_NAME="${POOL}/${DATASET}@auto_$(date +%Y%m%d_%H%M)"
zfs snapshot $SNAPSHOT_NAME

# 清理旧快照
find /${POOL}/${DATASET}/.zfs/snapshot -name "auto_*" -mtime +$RETENTION_DAYS -exec zfs destroy {} \;

### 5.2 定时任务
```bash
# 每天凌晨执行
0 0 * * * /usr/local/bin/zfs_snapshot.sh
```

> **注意**：  
> - LVM操作前务必备份数据  
> - NFS配置后验证防火墙规则  
> - S3挂载建议使用`nofail`参数防止启动失败  
> - ZFS脚本需测试权限和路径有效性

# 全量/增量备份方案

---

## 1. ZFS 快照管理

### 定时快照策略（每小时）
```bash
# 每小时创建一次快照（保留24小时）
0 * * * * /sbin/zfs snapshot zpool/data@$(date +\%Y\%m\%d-\%H\%M)
```

### 快照清理策略
```bash
# 每日清理脚本（保留策略）
0 2 * * * /sbin/zfs list -t snapshot -o name | grep zpool/data@ | sort -r | awk 'NR>24 {print "zfs destroy " $1}' | sh
0 3 * * * /sbin/zfs list -t snapshot -o name | grep zpool/data@ | sort -r | awk 'NR>168 {print "zfs destroy " $1}' | sh  # 保留7天
```

---

## 2. BorgBackup 配置

### 初始化加密仓库
```bash
borg init --encryption=repokey-blake2 /backup/borg-repo
```

### 执行增量备份
```bash
borg create --stats --progress --compression zstd \
  /backup/borg-repo::$(hostname)-{now:%Y%m%d-%H%M} \
  /mnt/zfs_snapshot
```

### 挂载备份仓库
```bash
borg mount /backup/borg-repo::archive-name /mnt/borg-mount
```

---

## 3. 异地同步流程

### Rclone 加密传输配置
```bash
rclone copy --progress --transfers 4 \
  --crypt-remote remote:encrypted-bucket \
  /backup/borg-repo \
  --crypt-password-file=/etc/rclone-secret.txt
```

### 配置文件示例 (~/.config/rclone/rclone.conf)
```ini
[encrypted-remote]
type = crypt
remote = s3:bucket/path
password = 复杂密码或文件路径
password2 = 可选的二级盐值
```

---

## 4. 恢复演练流程

### 查找备份版本
```bash
borg list /backup/borg-repo
```

### 提取特定文件
```bash
borg extract /backup/borg-repo::archive-name path/to/file
```

### 完整恢复操作
```bash
borg extract --numeric-owner --strip-components=3 /backup/borg-repo::archive-name
```

---

## 5. MySQL 数据库备份

### 定时备份脚本 (/usr/local/bin/mysql-backup.sh)
```bash
#!/bin/bash
DATE=$(date +%Y%m%d-%H%M)
DUMPDIR=/backup/mysql
mkdir -p $DUMPDIR

mysqldump --single-transaction --quick \
  -u backupuser -p'securepassword' \
  --all-databases | gzip > $DUMPDIR/full-$DATE.sql.gz

# 保留策略（保留30天）
find $DUMPDIR -name "*.sql.gz" -mtime +30 -delete
```

### Crontab 定时任务
```bash
0 1 * * * /usr/local/bin/mysql-backup.sh
```

---

## 监控与验证
- **完整性检查**：`borg check /backup/borg-repo`
- **传输验证**：`rclone check /backup/borg-repo remote:encrypted-bucket`
- **空间监控**：`zpool list` + `borg info /backup/borg-repo`

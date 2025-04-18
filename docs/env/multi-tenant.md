# 多租户环境隔离配置指南

## 1. 用户组权限设置 - GPU设备隔离
```udev
# /etc/udev/rules.d/99-gpu-access.rules
# NVIDIA GPU设备访问规则
SUBSYSTEM=="drm", KERNEL=="card[0-9]", ATTRS{vendor}=="0x10de", GROUP="gpuusers", MODE="0660"

# AMD GPU设备访问规则 
SUBSYSTEM=="drm", KERNEL=="card[0-9]", ATTRS{vendor}=="0x1002", GROUP="gpuusers", MODE="0660"

操作步骤：
1. 创建用户组：`groupadd gpuusers`
2. 添加用户到组：`usermod -aG gpuusers [username]`
3. 重载udev规则：`udevadm control --reload && udevadm trigger`
4. 重启相关服务：`systemctl restart systemd-logind`

## 2. 磁盘配额配置
```bash
# /etc/fstab 配置示例
/dev/sdb1  /data  ext4  defaults,usrquota,grpquota  0 0

# 初始化配额系统
mount -o remount /data
quotacheck -cugm /data
quotaon /data

# 设置用户配额
edquota -u user1
```
```quota
# 用户配额配置示例
Disk quotas for user user1 (uid 1001):
  Filesystem   blocks    soft    hard  inodes  soft  hard
  /dev/sdb1       0    500000  550000       0  1000  1500

# 检查配额状态
repquota -ug /data

## 3. Cgroups资源限制
```bash
# 创建cgroup子系统
cgcreate -g cpu,memory:/tenant_group

# 设置CPU权重（默认1024）
cgset -r cpu.shares=512 tenant_group

# 内存限制（1GB）
cgset -r memory.limit_in_bytes=1G tenant_group

# 进程绑定
cgclassify -g cpu,memory:tenant_group $(pidof process)

# 持久化配置
echo 'group tenant_group {
  cpu {
    cpu.shares = 512;
  }
  memory {
    memory.limit_in_bytes = 1G;
  }
}' > /etc/cgconfig.conf

## 4. 审计日志配置
```bash
# /etc/audit/rules.d/tenant.rules
-a always,exit -F dir=/home -F uid!=0 -F auid>=1000 -F auid!=4294967295 -k tenant_activity
-a always,exit -S execve -F gid=1001 -k tenant_exec

# 关键配置项：
-w /etc/passwd -p wa -k tenant_auth
-w /data/tenant_storage -p rwxa -k tenant_storage

# 日志查看命令
ausearch -k tenant_activity --start today

## 5. nsjail容器化隔离模板
```bash
# nsjail.cfg
mode: l
user: 1000
group: 1001
hostname: tenant-vm

cgroup {
  mem_max: 1073741824;
  cpu_shares: 512;
  pids_max: 100;
}

mount {
  src: "/data/tenant"
  dst: "/home"
  is_bind: true
  rw: false
}

network {
  use_clone_new_net: true
  use_loopback: false
}

exec_bin {
  path: "/bin/bash"
  arg: ["--login"]
}

# 启动命令
nsjail --config nsjail.cfg --disable_proc
```

## 实施注意事项
1. 用户组隔离需配合SELinux/AppArmor使用
2. 配额系统需要文件系统支持（ext4/xfs）
3. Cgroups v2需要调整配置语法
4. 审计日志需定期轮转和归档
5. nsjail需启用Linux CAP_*权限控制
6. 所有配置变更后需进行功能验证

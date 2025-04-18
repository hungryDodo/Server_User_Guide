# Ubuntu 服务器定制安装与优化指南

## 1. Ubuntu 镜像定制（Preseed 自动应答）
```preseed
### 基础配置
d-i debian-installer/locale string en_US.UTF-8
d-i time/zone string Asia/Shanghai
d-i keyboard-configuration/xkb-keymap select us

### 网络配置
d-i netcfg/choose_interface select auto
d-i netcfg/dhcp_timeout string 60

### 镜像源
d-i mirror/country string manual
d-i mirror/http/hostname string mirrors.aliyun.com
d-i mirror/http/directory string /ubuntu

### 用户配置
d-i passwd/user-fullname string Admin
d-i passwd/username string admin
d-i passwd/user-password-crypted password $6$salt$HxY...

### 分区配置（配合下文章节）
d-i partman-auto/disk string /dev/sda /dev/sdb
d-i partman-auto/method string raid
```

制作定制镜像：
```bash
# 解压原ISO
xorriso -osirrox on -indev ubuntu-22.04.iso -extract / custom-iso

# 修改preseed配置
echo "preseed.cfg" >> custom-iso/isolinux/txt.cfg

# 重新打包
mkisofs -r -V "Custom Ubuntu" -o custom.iso \
  -J -l -b isolinux/isolinux.bin -c isolinux/boot.cat \
  -no-emul-boot -boot-load-size 4 -boot-info-table custom-iso
```

## 2. 磁盘分区与 RAID 配置
### RAID 1 创建命令
```bash
sudo mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/sda1 /dev/sdb1
sudo mdadm --create /dev/md1 --level=1 --raid-devices=2 /dev/sda2 /dev/sdb2

# 文件系统创建
mkfs.xfs /dev/md0  # /boot
mkfs.xfs /dev/md1  # /

# 更新mdadm配置
mdadm --detail --scan >> /etc/mdadm/mdadm.conf
update-initramfs -u
```

/etc/fstab 配置示例：
```
/dev/md1   /     xfs  defaults,noatime,nodiratime  0 0
/dev/md0   /boot xfs  defaults,noatime         0 0
```

## 3. 内核编译优化
### 关键配置参数
```config
# CPU调度器选择
CONFIG_HZ=1000
CONFIG_HZ_1000=y
CONFIG_HZ_PERIODIC=y

# 调度器配置
CONFIG_CGROUP_SCHED=y
CONFIG_FAIR_GROUP_SCHED=y
CONFIG_MUQSS_SCHED=y
```

编译步骤：
```bash
sudo apt install build-essential libncurses-dev flex bison libssl-dev
make menuconfig
make -j$(nproc) bindeb-pkg
sudo dpkg -i ../linux-*.deb
```

## 4. 安全加固配置
### fail2ban 配置（/etc/fail2ban/jail.local）
```ini
[sshd]
enabled = true
maxretry = 3
bantime = 1h
findtime = 300
logpath = %var/log/auth.log
```

### UFW 防火墙规则
```bash
sudo ufw default deny incoming
sudo ufw allow 22/tcp
sudo ufw allow 80,443/tcp
sudo ufw limit 22/tcp
sudo ufw logging on
sudo ufw enable
```

## GRUB 引导参数优化建议
```grub
# /etc/default/grub 关键参数
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash elevator=deadline mitigations=off transparent_hugepage=never nmi_watchdog=0"

# 推荐优化参数组合：
• noibrs noibpb nopti nospectre_v2 nospec
• intel_iommu=on iommu=pt
• mem_sleep_default=deep
• consoleblank=0

# 更新配置
sudo update-grub
```

## 注意事项
1. RAID 配置前务必确认磁盘序列号
2. 内核编译建议保留原内核作为回退
3. 修改GRUB参数前做好配置备份
4. 安全加固后需测试SSH连通性
5. 生产环境建议先进行硬件兼容性测试

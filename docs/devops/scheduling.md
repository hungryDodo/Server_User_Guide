# Slurm 作业调度系统配置指南

## 1. Slurm集群配置（slurm.conf）

```conf
# 基础配置
ClusterName=my_cluster         # 集群名称
ControlMachine=master01        # 主控节点主机名
SlurmctldPort=6817             # 控制端口
SlurmdPort=6818                # 计算节点通信端口
AuthType=auth/munge            # 认证协议

# 节点配置
NodeName=node[01-10] CPUs=48 Sockets=2 CoresPerSocket=12 ThreadsPerCore=2 RealMemory=257000
PartitionName=compute Nodes=node[01-10] Default=YES MaxTime=INFINITE State=UP

# 高级参数
SelectType=select/cons_res     # 资源选择算法
SelectTypeParameters=CR_CPU    # 基于CPU核心分配
JobCompType=jobcomp/file       # 作业记录方式
ProctrackType=proctrack/cgroup # 进程跟踪方式

# 调度策略
SchedulerType=sched/backfill   # 支持回填调度
PreemptType=preempt/partition_prio # 分区优先级抢占
```

## 2. 队列权限控制（QoS配置）

```bash
# 创建QoS策略
sacctmgr add qos name=high_prio \
  Priority=100 \
  MaxJobs=1000 \
  MaxSubmitJobsPerUser=50 \
  GrpJobs=200 \
  GrpTRES=cpu=100,gres/gpu=10

# 用户关联QoS
sacctmgr modify user alice set qos=high_prio
```

## 3. GPU资源分配策略

```conf
# slurm.conf 添加
GresTypes=gpu

# nodes.conf 节点定义
NodeName=gpu01 Gres=gpu:a100:2

# gres.conf 模板
AutoDetect=nvml             # 自动检测GPU类型
Name=gpu Type=a100 File=/dev/nvidia0  # 指定设备路径
```

## 4. 计费系统对接

```conf
# slurmdbd.conf 配置
AccountingStorageType=accounting_storage/slurmdbd
AccountingStorageHost=ldap-db01
AccountingStoragePort=6819
AuthType=auth/ldap

# LDAP映射配置
LDAPURI=ldap://ldap.example.com
LDAPBindDN=cn=admin,dc=example,dc=com
LDAPUserSearch=(uid=%u)
```

## 5. SBATCH脚本模板

```bash
#!/bin/bash
#SBATCH --job-name=myjob          # 作业名称
#SBATCH --output=job_%j.out       # 输出文件
#SBATCH --error=job_%j.err        # 错误文件
#SBATCH --partition=gpu           # 提交分区
#SBATCH --nodes=2                 # 节点数量
#SBATCH --ntasks-per-node=4       # 每节点任务数
#SBATCH --cpus-per-task=8         # CPU核心数
#SBATCH --gres=gpu:a100:2         # GPU类型和数量
#SBATCH --mem=64G                 # 内存总量
#SBATCH --time=24:00:00           # 运行时间上限
#SBATCH --qos=high_prio           # QoS策略
#SBATCH --account=project01       # 计费账户

module load cuda/11.4            # 加载CUDA环境
srun hostname | sort             # 并行任务示例
```

### 参数说明
- `--ntasks`: 总MPI进程数
- `--cpus-per-task`: 每个任务分配的CPU核心
- `--gres`: 指定加速器资源（格式：类型:型号:数量）
- `--mem`: 包含内存和swap的总容量限制
- `--exclusive`: 请求节点独占模式

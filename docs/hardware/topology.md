# 服务器拓扑结构文档

---

## 1. 机架布局图

### 机架物理布局（42U标准机架）

| U位范围 | 设备类型              | 电源走线说明                     |
|---------|---------------------|------------------------------|
| U1-U4   | 预留空间              | 主电源走线（左侧垂直PDU）            |
| U5-U10  | 计算节点（8台）         | 双路电源接入，主备线路分离             | 
| U11-U15 | 网络交换机（IB/以太网）   | 冗余电源接入，走线路径与计算节点隔离       |
| U16-U19 | KVM/IPMI管理模块      | 独立电源回路，右侧PDU供电             |
| U20-U22 | 存储节点              | 双电源走线沿机架后侧通道捆绑固定         |
| U23-U24 | 电源分配单元（PDU）      | 主备PDU分别位于机架左右两侧            |

> 电源规范：  
> - 主备电源线路需物理隔离，间隔至少10cm  
> - 单机柜总功耗不超过16kW（220V/63A）  
> - 电源线缆使用16AWG规格，最大长度不超过2米

---

## 2. PCIe通道分配表

### PCIe拓扑分析（基于lspci -vvv解析）

| 设备名称                | PCIe插槽位置 | 链路带宽 | NUMA节点 | 用途说明          |
|------------------------|-------------|---------|----------|-----------------|
| NVIDIA A100 80GB       | Slot1       | x16 Gen4 | Node0    | GPU计算加速       |
| Intel XXV710-DA2       | Slot3       | x8 Gen3  | Node1    | 100Gb IB网络      |
| Samsung PM1735 NVMe    | Slot5       | x4 Gen4  | Node0    | 高速存储设备        |
| Broadcom 9400-16i      | Slot7       | x8 Gen4  | Node1    | RAID控制器        |
| Intel E810-CQDA2       | Slot9       | x16 Gen4 | Node0    | 100Gb以太网        |

> 关键配置建议：  
> - GPU设备建议独占x16通道，避免带宽争用  
> - 网络设备需确保PCIe插槽与NUMA节点对齐  
> - 存储控制器需启用ACS功能保证IO隔离

---

## 3. NUMA节点绑定策略

### 推荐numactl配置参数

```bash
# 查看NUMA拓扑
numactl -H

# 内存绑定策略（强制本地内存分配）
numactl --cpunodebind=0 --membind=0 ./application

# 跨节点访问优化（当本地内存不足时）
numactl --preferred=1 ./memory_intensive_app

# IRQ平衡配置（示例：绑定网卡中断到Node1）
echo "node1" > /sys/class/net/ib0/device/numa_node
```

**绑定策略说明**：  
- 计算密集型任务绑定到Node0（连接GPU和本地NVMe）  
- 网络密集型服务绑定到Node1（直连IB HCA设备）  
- 大内存应用使用`--interleave=all`实现跨节点内存交错

---

## 4. 网络布线规范

### DAC线缆规格对照表

| 线缆长度 | 最大支持速率 | 适用场景                |
|---------|-------------|-----------------------|
| 0.7m    | 200Gb/s     | 机架内设备互连           |
| 1m      | 100Gb/s     | 相邻机柜跨架连接         |
| 3m      | 56Gb/s      | 同列机柜末端设备连接       |
| 5m      | 40Gb/s      | 跨机柜骨干连接（需中继）    |

### InfiniBand Subnet Manager配置

**配置步骤**：  
1. 安装Subnet Manager服务：
   ```bash
   yum install opensm libibverbs-utils
   ```

2. 生成配置文件：
   ```bash
   opensm -c /etc/opensm/opensm.conf
   ```

3. 修改关键参数：
   ```ini
   sm_priority = 15
   force_linear_lmc = 1
   subnet_prefix = fe80:0000:0000:0001
   qos_policy = max_bandwidth
   ```

4. 启动服务并设置自启：
   ```bash
   systemctl enable --now opensm
   ```

5. 验证配置状态：
   ```bash
   ibstat | grep "State active"
   ibnetdiscover -p
   ```

**布线规范**：  
- 200Gb链路必须使用官方认证的QSFP56 DAC线缆  
- 跨机柜连接需使用光纤跳线（OM4多模光纤最大长度40m）  
- 所有线缆必须标注双端设备信息（源/目标机柜-设备-端口）

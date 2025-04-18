## 硬件配置清单

### 计算节点
| 组件        | 规格                          |
|-------------|------------------------------|
| CPU         | Intel Xeon Gold 6338 x2      |
| 内存        | 512GB DDR4 ECC               |
| GPU         | NVIDIA RTX 4090 x8           |
| 存储        | 4TB NVMe SSD + 16TB HDD      |

### 网络配置
```plantuml
@startuml
node "Switch" as sw
node "Server" as svr {
  component "NIC1" as nic1
  component "NIC2" as nic2
}
nic1 --> sw : 10GbE
nic2 --> sw : 10GbE
@enduml
# Conda 环境管理指南

## 1. 清华源配置
创建/修改 `~/.condarc` 文件：

```yaml
channels:
  • defaults
show_channel_urls: true
default_channels:
  • https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/main
  • https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/r
  • https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/msys2
custom_channels:
  conda-forge: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud
  pytorch: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud
```

## 2. 环境克隆方法
```bash
conda create --name new_env --clone base
```
**使用限制**：
- 需在相同操作系统和Conda版本下使用
- 无法跨不同架构平台克隆（如x86到ARM）
- 源环境不能正在运行中

## 3. 依赖导出技巧
| 方法                | 命令                                  | 特点                          |
|---------------------|---------------------------------------|-----------------------------|
| 基础导出            | `conda env export > environment.yml` | 包含完整依赖树和精确版本       |
| 仅历史记录(--from-history) | `conda env export --from-history`    | 仅保留显式安装的包，无版本约束 |
| 精确版本导出         | `conda list --explicit > spec-file.txt` | 生成可直接安装的精确版本列表   |

## 4. 多用户共享方案
```bash
# 修改conda包存储目录权限
sudo chmod -R 755 /opt/anaconda3/pkgs
sudo chown -R :shared_group /opt/anaconda3/pkgs

# 共享环境目录
conda config --prepend envs_dirs /shared/envs
```

## 5. 环境打包移植（conda-pack）
**完整流程**：
```bash
# 1. 安装conda-pack
conda install -c conda-forge conda-pack

# 2. 激活待打包环境
conda activate target_env

# 3. 打包环境（生成tar.gz）
conda pack -n target_env -o target_env.tar.gz

# 4. 传输到目标机器
scp target_env.tar.gz user@remote:/path/

# 5. 在目标机器解压
mkdir -p /opt/envs
tar -xzf target_env.tar.gz -C /opt/envs

# 6. 使用环境
source /opt/envs/bin/activate

# （可选）添加环境目录到conda
conda config --append envs_dirs /opt/envs
```

**注意事项**：
- 要求相同操作系统和架构
- 打包文件包含Python解释器
- 解压后需通过绝对路径激活环境
- 建议使用conda-pack而非直接复制环境目录

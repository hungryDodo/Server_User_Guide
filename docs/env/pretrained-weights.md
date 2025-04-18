# 预训练模型管理指南

## 1. 权重文件存储结构规范
### 目录规划原则
```
models/
├── nlp/
│   ├── bert/
│   │   ├── v1.0/
│   │   │   ├── config.json
│   │   │   ├── pytorch_model.bin
│   │   │   └── special_tokens_map.json
│   │   └── v2.0/
│   ├── gpt-2/
├── cv/
│   ├── resnet/
│   └── vit/
└── multimodal/
    └── clip/
```

### 文件命名规范
- 必须包含模型类型、版本、格式三元组  
  (例：`bert-base-uncased_v1.0_safetensors`)
- 附加SHA256校验文件  
  (例：`bert-base-uncased_v1.0.sha256`)

## 2. HuggingFace镜像配置
### 环境变量设置
```bash
# Linux/MacOS
export HF_ENDPOINT=https://hf-mirror.com

# Windows(PowerShell)
$env:HF_ENDPOINT = "https://hf-mirror.com"
```

### 验证配置
```bash
huggingface-cli download --repo-type model bert-base-uncased
```

## 3. 模型格式转换工具
### ckpt转safetensors脚本
```python
from transformers import AutoModel

def convert_ckpt_to_safetensors(input_path, output_path):
    model = AutoModel.from_pretrained(input_path)
    model.save_pretrained(
        output_path,
        safe_serialization=True,
        max_shard_size="2GB"
    )
    
# 使用示例
convert_ckpt_to_safetensors("./ckpt_model", "./safetensors_model")
```

### 转换工具功能要求
- 支持格式互转：ckpt ↔ safetensors ↔ h5
- 自动生成版本迁移日志
- 保留原始模型配置信息

## 4. 完整性校验流程
### SHA256校验操作
```bash
# 生成校验文件
sha256sum model.safetensors > model.sha256

# 验证完整性
sha256sum -c model.sha256
```

### 校验策略
1. 下载完成后立即生成哈希文件
2. 版本升级时重建哈希库
3. 执行训练前强制校验

## Git LFS管理规范
### 仓库初始化
```bash
git lfs install
git lfs track "*.bin" "*.safetensors" "*.h5"
git add .gitattributes
```

### 大文件管理建议
- 单文件超过100MB必须使用LFS
- 禁止直接提交原始ckpt文件
- 保持存储库总大小小于1GB

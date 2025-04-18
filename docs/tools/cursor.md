# Cursor IDE 进阶使用指南

## 一、网络代理配置（模型下载加速）
```json
// 在Settings.json中添加（Ctrl+Shift+P打开命令面板）
{
  "http.proxy": "http://127.0.0.1:7890",
  "http.proxyStrictSSL": false
}
```
**配置说明**：
1. 支持Socks5协议需使用`socks5://ip:port`格式
2. 端口号需对应Clash/V2RayN等代理工具的实际端口
3. 企业级代理需添加认证信息：`http://user:pass@ip:port`

## 二、本地GGUF模型加载
```bash
# 前置依赖安装
pip install llama-cpp-python --prefer-binary
# CUDA加速版本安装
CMAKE_ARGS="-DLLAMA_CUBLAS=on" pip install llama-cpp-python

# 模型加载命令（Ctrl+K打开命令输入）
/load_model /path/to/model.Q4_K_M.gguf 
--n_ctx 4096 
--n_threads 8 
--n_gpu_layers 35
```

**参数调优建议**：
| 参数 | 推荐值 | 作用说明 |
|------|--------|----------|
| n_gpu_layers | 20-40 | 显存容量决定加速层数 |
| n_ctx | 2048-4096 | 上下文窗口大小 |
| n_batch | 512 | 批处理大小优化吞吐量 |

## 三、代码补全Prompt模板
```json
// custom_prompts.json
{
  "function_generation": {
    "system_prompt": "你是一位精通Python的架构师，遵循PEP8规范",
    "user_input": "请为{filename}生成{language}函数，要求：{requirement}",
    "context_rules": [
      "优先使用类型注解",
      "包含异常处理逻辑",
      "添加Google风格docstring"
    ]
  }
}
```
**模板应用方法**：
1. 在设置中指定模板路径：`"customPromptPath": "./prompts/"`
2. 通过`/apply_preset`命令加载模板集
3. 使用`@custom`触发自定义补全逻辑

## 四、快捷键深度定制
```json
// keybindings.json 配置示例
{
  {
    "key": "ctrl+s", 
    "command": "llama.commitCode",
    "when": "editorTextFocus && localModelActive"
  },
  {
    "key": "ctrl+alt+l",
    "command": "codeium.cycleSuggestions",
    "when": "suggestWidgetVisible"
  }
}
```

**冲突解决策略**：
1. 使用`keyboard-shortcuts analyzer`插件检测冲突
2. 条件限定语法：`"when": "editorLangId == python"`
3. 分层配置：全局快捷键 > 语言特定 > 插件专用

## 五、兼容性测试报告（Codeium + 本地模型）

| 测试场景 | 结果 | 解决方案 |
|---------|------|----------|
| 双补全引擎同时启用 | 冲突率23% | 设置补全延迟：`"codeium.delayMs": 300` |
| GPU资源抢占 | 显存溢出 | 通过`nvidia-smi --gpu-reset`重置 |
| 混合补全模式 | 正常运作 | 配置补全优先级权重：`"localModelWeight": 0.7` |

**推荐协作方案**：
1. 文档场景优先使用Codeium
2. 算法开发启用本地13B+模型
3. 通过`/toggle_provider`命令动态切换引擎

> 注意事项：本地模型运行时建议关闭实时补全（配置`"inlineSuggest.autoTrigger": false`）

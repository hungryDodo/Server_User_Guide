# 版本更新日志

## 1. 版本号命名规则
本产品采用[语义化版本控制 2.0.0](https://semver.org/lang/zh-CN/)规范：
- **主版本号 (MAJOR)**：不兼容的 API 修改
- **次版本号 (MINOR)**：向下兼容的功能新增
- **修订号 (PATCH)**：向下兼容的问题修正

## 2. 变更类型标记
- `🆕` 新增功能
- `🐛` 缺陷修复  
- `🗑️` 弃用功能
- `🚀` 性能优化
- `🔒` 安全更新

## 3. 影响范围评估
重大变更将提供迁移指南：
| 版本范围 | 迁移文档 |
|---------|----------|
| v1.x → v2.x | [迁移指南](./migration-v2.md) |
| v2.2.x → v2.3.x | [升级说明](./upgrade-v2.3.md) |

## 4. 审阅者列表
技术评审组成员：
- @tech-lead-zhang
- @qa-engineer-li
- @security-team-wang

## 版本历史记录

| 版本号 | 发布日期   | 变更类型 | 影响范围 | 修改文件                | 审阅者              |
|--------|------------|----------|----------|-------------------------|---------------------|
| v2.3.1 | 2023-08-15 | 🐛🔒      | 局部     | `src/auth/api.go`<br>`docs/security.md` | @security-team-wang |
| v2.3.0 | 2023-08-10 | 🆕🚀      | 中度     | `src/core/processor.py`<br>`config/settings.yaml` | @tech-lead-zhang    |
| v2.2.3 | 2023-08-01 | 🐛🗑️      | 局部     | `legacy/old_module.js`<br>`tests/unit_test.py` | @qa-engineer-li     |
| v2.2.2 | 2023-07-25 | 🔒        | 全局     | `package.json`<br>`Dockerfile.prod` | @security-team-wang |

## 重大变更说明
### v2.3.0 版本更新
- 新增流式数据处理模块（标记为🆕）
- 查询性能提升 40%（标记为🚀）
- 影响配置文件格式，需更新 `settings.yaml`

### v2.2.3 版本更新
- 弃用旧版加密模块（标记为🗑️）
- 修复空指针异常问题（标记为🐛）
- 需要手动清理 `old_module.js` 相关调用

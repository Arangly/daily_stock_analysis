# Code Wiki 索引

本目录存放项目的结构化文档，分为两类：

## 文档列表

| 文档 | 面向对象 | 内容 |
|------|---------|------|
| [CODE_WIKI.md](CODE_WIKI.md) | 开发者 / 维护者 | 项目整体架构、主要模块职责、关键类与函数说明、依赖关系、项目运行方式、CI/CD 与部署、测试体系 |
| [USER_GUIDE.md](USER_GUIDE.md) | 终端用户 | 核心功能、操作步骤、计算逻辑（评分系统/市场温度计/决策信号生命周期）、配置说明、通知推送、使用注意事项、常见问题 |

## 阅读建议

- **想了解项目架构与代码组织**：从 [CODE_WIKI.md](CODE_WIKI.md) 开始
- **想使用项目进行股票分析**：从 [USER_GUIDE.md](USER_GUIDE.md) 开始
- **想快速上手**：参考 [USER_GUIDE.md 的快速开始章节](USER_GUIDE.md#3-快速开始)
- **想深入理解计算逻辑**：参考 [USER_GUIDE.md 的计算逻辑说明章节](USER_GUIDE.md#5-计算逻辑说明)

## 与其他文档的关系

本目录文档是对项目现有 `docs/` 文档的补充与整合：

- `docs/README_EN.md` / `docs/INDEX.md`：项目入口与文档导航
- `docs/full-guide.md`：完整配置与部署指南（配置细节的权威来源）
- `docs/LLM_CONFIG_GUIDE.md`：LLM 渠道配置指南
- `docs/wiki/CODE_WIKI.md`：技术架构总览（本文档）
- `docs/wiki/USER_GUIDE.md`：用户功能与操作指南（本文档）

如遇配置细节冲突，以 `docs/full-guide.md` 和实际代码为准。

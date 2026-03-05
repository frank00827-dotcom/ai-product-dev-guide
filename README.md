# AI 驱动产品开发指南

> 从 PRD 到产品交付的全链路 AI 辅助开发方法论 · 零基础可执行

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![GitHub stars](https://img.shields.io/github/stars/frank00827-dotcom/ai-product-dev-guide)](https://github.com/frank00827-dotcom/ai-product-dev-guide/stargazers)
[![GitHub issues](https://img.shields.io/github/issues/frank00827-dotcom/ai-product-dev-guide)](https://github.com/frank00827-dotcom/ai-product-dev-guide/issues)
[![CI](https://github.com/frank00827-dotcom/ai-product-dev-guide/actions/workflows/ci.yml/badge.svg)](https://github.com/frank00827-dotcom/ai-product-dev-guide/actions/workflows/ci.yml)

---

## 📖 简介

本指南提供一套**完整可执行**的 AI 驱动产品开发方法论，以「慢病随访与数字疗法」微信小程序为贯穿案例，覆盖：

- **产品发现** → **架构设计** → **AI 开发工作流** → **模块开发** → **测试 QA** → **部署运维** → **持续迭代**

**核心理念**：
1. AI 是副驾驶，人是决策者
2. Prompt 即文档
3. 渐进式信任

---

## 🚀 快速开始

### 30-60 分钟跑通完整项目

```bash
# 1. 克隆实战项目
git clone https://github.com/frank00827-dotcom/chronic-care.git
cd chronic-care

# 2. 启动基础设施
docker compose -f deploy/docker-compose.yml up -d

# 3. 初始化后端
cd server
python -m venv venv
source venv/bin/activate
pip install -r requirements.txt
flask db upgrade
flask run

# 4. 启动前端
cd ../client
npm install
npm run dev:weapp
```

详细步骤见 [QUICKSTART.md](chronic-care/QUICKSTART.md)

---

## 📚 文档导航

| 章节 | 主题 | 核心产出 |
|------|------|---------|
| [00](./00-guide-overview.md) | 指南总览与工具链 | AI 工具选型、环境准备 |
| [01](./01-product-discovery.md) | 产品发现与 PRD | 用户画像、PRD、优先级矩阵 |
| [02](./02-architecture-design.md) | 架构设计 | 系统架构、技术选型、API 契约 |
| [03](./03-compliance-security.md) | 合规与安全设计 | 等保方案、数据分类分级 |
| [04](./04-ai-dev-workflow.md) | AI 开发工作流 | Prompt 工程、代码审查流程 |
| [05](./05-module-development.md) | 模块开发实战 | 完整代码示例、Prompt 模板 |
| [06](./06-testing-qa.md) | 测试与质量保障 | 141 个测试用例、86% 覆盖率 |
| [07](./07-deployment-ops.md) | 部署与运维 | CI/CD、监控告警 |
| [08](./08-iteration.md) | 迭代与持续改进 | 数据驱动迭代、A/B 测试 |
| [附录 A1](./appendix/prompt-templates.md) | Prompt 模板库 | 30+ 可复用模板 |
| [附录 A2](./appendix/checklists.md) | 检查清单汇总 | 各阶段质量门禁 |
| [附录 A3](./appendix/adr-template.md) | ADR 模板 | 架构决策记录 |

---

## 📊 实战验证成果

| 指标 | 数值 |
|------|------|
| 指南文档 | 18 个文件，~175KB |
| 实战项目 | 47 个文件，~5,500 行代码 |
| 测试用例 | 141 个（后端 59 + 前端 82） |
| 测试覆盖率 | 86%（行覆盖）/ 79%（分支覆盖） |
| 开发效率提升 | 7.4 倍 |

---

## 🎯 适用场景

**推荐使用**：
- ✅ AI 辅助产品开发项目
- ✅ 快速原型验证
- ✅ 全栈开发团队培训
- ✅ 医疗/慢病管理类产品

**不推荐**：
- ❌ 高并发/高性能场景（需额外优化）
- ❌ 实时性要求极高的系统
- ❌ 已有成熟技术栈的大型企业

---

## 🛠️ 技术栈

| 层级 | 技术 |
|------|------|
| 前端 | Taro 3.x + React 18 + TypeScript + NutUI + Zustand |
| 后端 | Flask 3.x + SQLAlchemy 2.x + PostgreSQL 16 |
| 基础设施 | Docker + MinIO + Redis |
| 测试 | pytest + Jest + React Testing Library |

---

## 📋 执行报告

- [执行报告](./EXECUTION-REPORT.md) - 过程验证记录
- [最终报告](./FINAL-REPORT.md) - 交付总结
- [审查报告](./FINAL-REVIEW-REPORT.md) - 推理模式深度 Review

---

## 🤝 贡献

欢迎贡献！详见 [贡献指南](./CONTRIBUTING.md)

**快速参与**：
1. [Fork 仓库](https://github.com/frank00827-dotcom/ai-product-dev-guide/fork)
2. 创建功能分支 (`git checkout -b feature/AmazingFeature`)
3. 提交修改 (`git commit -m 'Add some AmazingFeature'`)
4. 推送到分支 (`git push origin feature/AmazingFeature`)
5. 创建 [Pull Request](https://github.com/frank00827-dotcom/ai-product-dev-guide/pulls)

---

## 📄 开源协议

[MIT License](./LICENSE)

---

## 📞 支持

- [Issue 反馈](https://github.com/frank00827-dotcom/ai-product-dev-guide/issues)
- [讨论区](https://github.com/frank00827-dotcom/ai-product-dev-guide/discussions)

---

*本指南以多工具协作方式生成与验证（OpenClaw / Claude Code / Cursor 等均可替换），所有过程均有完整记录可追溯；核心标准是"按步骤可复现跑通"。*

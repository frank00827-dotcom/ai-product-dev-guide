# 00 - 指南总览与AI工具链

## 0.1 核心理念

### AI 辅助开发的三条原则

1. **AI 是副驾驶，人是决策者** - AI 生成初稿，人类审查决策。任何 AI 产出物都必须经过人工确认后才能进入下一阶段
2. **Prompt 即文档** - 每一次有价值的 AI 交互都应记录 Prompt 和产出，形成可复用的知识资产
3. **渐进式信任** - 从低风险环节开始使用 AI，逐步扩展到核心业务逻辑，建立团队对 AI 产出质量的校准能力

### 全链路 AI 覆盖矩阵

```
阶段          AI 参与度    人工参与度    决策权
─────────────────────────────────────────────
产品发现       ████░░░ 60%  ████████ 高    人
PRD 编写       ██████░ 80%  ██████░░ 中    人
架构设计       █████░░ 70%  ████████ 高    人
合规设计       ███░░░░ 40%  ████████ 高    人
代码生成       ████████ 90% ████░░░░ 低    AI+人
代码审查       ██████░ 80%  ██████░░ 中    人
测试生成       ████████ 90% ███░░░░░ 低    AI+人
部署配置       █████░░ 70%  ██████░░ 中    人
运维监控       ████░░░ 50%  ████████ 高    人
```

## 0.1.5 新手一键跑通（最小闭环）

> 目标：哪怕你是新手，也能在 **30-60 分钟**内把「前端小程序 + 后端 API + PostgreSQL + MinIO」跑起来，并完成一次端到端联调。

### 📦 完整脚手架已备好

项目已生成在 `chronic-care/` 目录，包含：
- ✅ 完整的前后端代码（可运行）
- ✅ Docker Compose 配置（一键启动基础设施）
- ✅ 数据库模型和迁移脚本
- ✅ 微信登录 + 健康数据录入功能

**快速开始**: 直接阅读 [`chronic-care/QUICKSTART.md`](../../chronic-care/QUICKSTART.md)

### 你将得到什么（跑通标准）
- 前端（Taro）：微信登录、健康数据录入页面
- 后端（Flask）：认证接口、健康数据 CRUD 接口
- PostgreSQL：患者档案、健康记录数据表
- MinIO：对象存储（用于后续文件上传功能）

### 环境准备（最少安装清单）
1. Node.js 18+（前端）
2. Python 3.11+（后端）
3. Docker Desktop（推荐，用于一键启动 PostgreSQL + MinIO）

### 环境变量配置（必须）

在 `chronic-care/server/` 目录下创建 `.env` 文件（或复制 `.env.example`）：

```bash
# 数据库（与 docker-compose.yml 一致）
DATABASE_URL=postgresql://app:app@localhost:5432/chronic_care

# JWT 配置
JWT_SECRET_KEY=your-secret-key-change-in-production
JWT_ACCESS_TOKEN_EXPIRES=2h
JWT_REFRESH_TOKEN_EXPIRES=7d

# 微信配置（开发阶段可用测试值）
WX_APP_ID=wx_test_app_id
WX_APP_SECRET=wx_test_app_secret

# MinIO 配置
MINIO_ENDPOINT=localhost:9000
MINIO_ACCESS_KEY=minio
MINIO_SECRET_KEY=minio123456

# Redis 配置（可选）
REDIS_URL=redis://localhost:6379
```

> ⚠️ **重要**: `.env` 文件包含敏感信息，**不要提交到 Git**（已在 `.gitignore` 中排除）

### 一键启动基础设施（Docker Compose）
脚手架已包含完整配置：

```bash
cd chronic-care
docker compose -f deploy/docker-compose.yml up -d
```

启动后验证：
```bash
# 检查容器状态
docker compose ps

# 应该看到：
# NAME                STATUS              PORTS
# chronic-care-postgres   Up (healthy)     5432/tcp
# chronic-care-minio      Up (healthy)     9000/tcp, 9001/tcp
# chronic-care-redis      Up (healthy)     6379/tcp
```

### 完整步骤（30-60 分钟跑通）

> 详细步骤见 **[`chronic-care/QUICKSTART.md`](../../chronic-care/QUICKSTART.md)**，以下是快速概览：

#### Step 1: 启动基础设施（5 分钟）
```bash
cd chronic-care
docker compose -f deploy/docker-compose.yml up -d
docker compose ps  # 验证全部 healthy
```

#### Step 2: 初始化后端（10 分钟）
```bash
cd server
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate
pip install -r requirements.txt
flask db upgrade  # 执行数据库迁移
flask run  # 启动后端，访问 http://localhost:5000/health 验证
```

#### Step 3: 启动前端（10 分钟）
```bash
cd client
npm install
npm run dev:weapp  # 启动开发服务器
```
然后在微信开发者工具中导入 `client/dist` 目录，勾选"不校验合法域名"。

#### Step 4: 端到端测试（5 分钟）
1. 小程序点击"微信一键登录"
2. 录入一条血糖记录
3. 查看后端日志确认数据写入

#### 常见问题排查
- 端口冲突 → 修改 `docker-compose.yml` 端口映射
- 依赖安装失败 → 使用镜像源（pip/清华，npm/淘宝）
- 小程序网络错误 → 勾选"不校验合法域名"

### 常见新手坑（已验证）
- ✅ 端口冲突：修改 `docker-compose.yml` 端口映射
- ✅ Windows 下 Docker 网络：优先用 `localhost`，别用容器名
- ✅ 小程序 HTTPS：微信开发者工具 → 勾选「不校验合法域名」
- ✅ Python 依赖安装失败：使用清华镜像源 `-i https://pypi.tuna.tsinghua.edu.cn/simple`

## 0.2 AI 工具链推荐

### 推荐组合（Best Practice）

> 说明：工具不绑定。你可以使用 OpenClaw / Claude Code / Cursor / Copilot 等任意组合；指南核心是“流程与产出物标准”，而不是某个具体工具。

| 环节 | 主力工具 | 辅助工具 | 选择理由 |
|------|----------|----------|----------|
| 产品发现/PRD | Claude (Web/API) | - | 长文本理解与生成能力强，适合结构化文档 |
| 架构设计 | Claude Code (CLI) | Mermaid Live Editor | 支持多轮对话迭代架构，可直接生成代码框架 |
| UI/UX 设计 | Cursor + v0.dev | Figma AI | v0 快速生成组件原型，Cursor 调整细节 |
| 日常编码 | Cursor | GitHub Copilot | Cursor 的 Composer 模式适合多文件编辑 |
| 复杂模块开发 | Claude Code (CLI) | Cursor | Claude Code 擅长理解全局上下文和复杂重构 |
| 代码审查 | Claude Code | - | 可对整个 PR diff 进行深度审查 |
| 测试生成 | Claude Code | Cursor | 理解业务上下文后生成高质量测试用例 |
| 文档生成 | Claude (Web) | - | 生成 API 文档、用户手册等 |
| CI/CD 配置 | Claude Code | - | 生成 pipeline 配置和部署脚本 |

### 工具安装与配置

```bash
# 1. Claude Code CLI 安装
npm install -g @anthropic-ai/claude-code

# 2. 验证安装
claude --version

# 3. 项目初始化（在项目根目录执行）
claude init

# 4. Cursor 安装
# 下载地址: https://cursor.sh
# 安装后配置 Claude 作为默认模型

# 5. 推荐 VS Code 扩展（如使用 VS Code）
# - GitHub Copilot
# - Taro Snippets
# - ESLint + Prettier
# - Thunder Client (API 调试)
```

## 0.3 项目协作规范

### Git 分支策略

```
main (生产)
 └── develop (开发主线)
      ├── feature/patient-module    (功能分支)
      ├── feature/followup-module
      ├── fix/login-bug
      └── release/v1.0.0           (发布分支)
```

### Commit Message 规范

```
<type>(<scope>): <subject>

<body>

<footer>

# type 枚举:
# feat     - 新功能
# fix      - 修复 bug
# docs     - 文档变更
# style    - 代码格式（不影响逻辑）
# refactor - 重构
# test     - 测试相关
# chore    - 构建/工具变更
# ai       - AI 生成代码（需标注 AI 工具和 Prompt 摘要）

# 示例:
# ai(patient): 生成患者档案 CRUD 模块
#
# Tool: Claude Code
# Prompt: 基于患者数据模型生成完整的 CRUD 接口
# Review: [reviewer_name] confirmed
```

### AI 产出物标注规范

所有 AI 生成的代码和文档必须标注：

```typescript
/**
 * @ai-generated
 * @tool Claude Code
 * @prompt "基于 Patient 数据模型生成 RESTful CRUD 接口"
 * @reviewed-by [reviewer_name]
 * @review-date [date]
 */
```

## 0.4 项目目录结构规范

```
chronic-care/
├── client/                     # 小程序前端（Taro）
│   ├── src/
│   │   ├── app.config.ts       # 全局配置
│   │   ├── app.ts
│   │   ├── pages/              # 页面
│   │   │   ├── patient/        # 患者模块页面
│   │   │   ├── followup/       # 随访模块页面
│   │   │   ├── therapy/        # 数字疗法页面
│   │   │   └── mine/           # 个人中心
│   │   ├── components/         # 公共组件
│   │   ├── services/           # API 调用层
│   │   ├── stores/             # 状态管理
│   │   ├── utils/              # 工具函数
│   │   ├── types/              # TypeScript 类型定义
│   │   └── assets/             # 静态资源
│   ├── config/                 # 构建配置
│   └── pyproject.toml / requirements.txt
├── server/                     # 后端服务（Flask）
│   ├── src/
│   │   ├── modules/            # 业务模块
│   │   │   ├── patient/        # 患者管理
│   │   │   ├── followup/       # 随访管理
│   │   │   ├── therapy/        # 数字疗法
│   │   │   ├── health-data/    # 健康数据
│   │   │   ├── notification/   # 通知提醒
│   │   │   └── auth/           # 认证授权
│   │   ├── common/             # 公共模块（拦截器、过滤器、守卫）
│   │   ├── config/             # 配置管理
│   │   └── app.py
│   ├── sqlalchemy/                 # 数据库 Schema
│   └── pyproject.toml / requirements.txt
├── docs/                       # 项目文档
│   ├── prd/                    # 产品需求文档
│   ├── architecture/           # 架构设计文档
│   ├── api/                    # API 文档
│   └── ai-logs/                # AI 交互记录
├── scripts/                    # 构建/部署脚本
├── .claude/                    # Claude Code 项目配置
│   └── settings.json
└── README.md
```

## 0.5 知识沉淀机制

### AI 交互日志模板

每次重要的 AI 辅助开发交互，在 `docs/ai-logs/` 下记录：

```markdown
# AI 交互记录 - [日期]-[序号]

## 基本信息
- **日期**: 2024-xx-xx
- **阶段**: [产品发现/架构设计/模块开发/...]
- **工具**: [Claude Code / Cursor / ...]
- **操作者**: [name]

## Prompt
[完整的 Prompt 内容]

## AI 产出
[AI 的关键输出，可摘要]

## 人工调整
[对 AI 产出做了哪些修改，为什么]

## 质量评估
- 可用性: [直接可用 / 需少量修改 / 需大量修改 / 不可用]
- 准确性: [高 / 中 / 低]
- 经验总结: [这次交互的关键收获]
```

### 架构决策记录（ADR）

重大技术决策使用 ADR 格式记录，详见 [附录-ADR模板](./appendix/adr-template.md)。

---

**下一步**: [01 - 产品发现与PRD](./01-product-discovery.md)

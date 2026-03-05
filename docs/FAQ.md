# 常见问题解答（FAQ）

**最后更新**: 2026-03-05  
**版本**: v1.0

---

## 一、快速开始相关

### Q1: 我没有编程基础，能跟着指南做出来吗？

**A**: 可以，但建议有基础认知：
- ✅ 了解基本的编程概念（变量、函数、循环）
- ✅ 了解基本的命令行操作
- ✅ 有耐心，愿意一步步跟着做

指南的 00 章包含完整的环境准备和快速开始步骤，30-60 分钟可以跑通完整项目。

**学习路径建议**：
1. 先跑通 chronic-care 项目（不求甚解）
2. 再读 00 章和 04 章（理解方法论）
3. 尝试修改代码（改 UI、改业务逻辑）
4. 参考 05 章开发自己的模块

---

### Q2: 运行 `docker compose up -d` 失败怎么办？

**常见问题**：

| 错误信息 | 原因 | 解决方案 |
|---------|------|---------|
| `command not found: docker` | 未安装 Docker | 安装 Docker Desktop |
| `port already in use` | 端口被占用 | 修改 docker-compose.yml 端口映射 |
| `healthcheck failed` | 容器启动慢 | 等待 1-2 分钟后重试 `docker compose ps` |
| `pull access denied` | 网络问题 | 配置 Docker 镜像加速器 |

**验证步骤**：
```bash
# 1. 检查 Docker 是否运行
docker ps

# 2. 检查容器状态
docker compose -f deploy/docker-compose.yml ps

# 3. 查看容器日志
docker compose logs postgres
docker compose logs minio
```

---

### Q3: `pip install -r requirements.txt` 失败

**常见原因**：
1. Python 版本不兼容（需要 3.11+）
2. 网络问题（使用镜像源）
3. 依赖冲突

**解决方案**：
```bash
# 检查 Python 版本
python --version  # 需要 3.11+

# 使用清华镜像源
pip install -r requirements.txt -i https://pypi.tuna.tsinghua.edu.cn/simple

# 或手动创建虚拟环境
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate
pip install --upgrade pip
pip install -r requirements.txt
```

---

### Q4: 小程序无法连接后端 API

**检查清单**：
- [ ] 后端是否启动（访问 http://localhost:5000/health）
- [ ] 微信开发者工具是否勾选"不校验合法域名"
- [ ] client/.env 中 API_BASE_URL 是否正确
- [ ] 防火墙是否阻止连接

**解决方案**：
```bash
# 1. 验证后端启动
curl http://localhost:5000/health

# 2. 微信开发者工具设置
# 详情 → 本地设置 → 勾选"不校验合法域名、web-view（业务域名）、TLS 版本以及 HTTPS 证书"

# 3. 检查前端配置
cat client/.env
# 应该是：TARO_APP_API_BASE=http://localhost:5000
```

---

## 二、技术栈相关

### Q5: 为什么选择 Flask 而不是 FastAPI/Django？

**决策理由**：
1. **模块化架构** - Flask 的 Blueprint 天然支持模块边界划分
2. **AI 生成质量** - Flask 训练数据丰富，AI 生成代码质量高
3. **学习曲线** - 比 Django 轻量，比 FastAPI 成熟
4. **演进路径** - 可平滑迁移到 FastAPI 或微服务

**如果已有 FastAPI 经验**：
- 可以将 Flask 代码改为 FastAPI，结构基本一致
- 主要改动：装饰器、依赖注入、异步支持

---

### Q6: 为什么选择 Taro 而不是 uni-app/原生小程序？

**决策理由**：
1. **React + TypeScript** - AI 生成质量最高，类型安全
2. **跨端能力** - 可编译到 H5、支付宝小程序、React Native
3. **生态成熟** - 京东维护，组件库丰富（NutUI）
4. **状态管理** - Zustand/MobX 成熟方案

**如果已有 Vue 经验**：
- uni-app 是更好的选择（Vue 生态）
- 指南的方法论同样适用，只需调整技术栈

---

### Q7: 可以用其他数据库吗？

**可以**，指南支持：
- ✅ PostgreSQL（推荐，默认）
- ✅ MySQL（修改 SQLAlchemy 连接字符串）
- ✅ SQLite（开发测试用）

**切换数据库**：
```bash
# 修改 .env
DATABASE_URL=mysql://user:password@localhost:3306/chronic_care

# 修改 docker-compose.yml
# postgres 服务改为 mysql 服务

# 重新执行迁移
flask db downgrade  # 如果有数据先回滚
flask db upgrade
```

---

## 三、AI 辅助开发相关

### Q8: 必须使用 Claude Code 吗？

**不是**。指南支持多种 AI 工具：
- ✅ Claude Code（推荐，CLI 集成好）
- ✅ Cursor（IDE 集成，Composer 模式）
- ✅ GitHub Copilot（代码补全）
- ✅ OpenClaw（本指南的协作平台）
- ✅ 其他支持长上下文的 LLM

**核心是 Prompt 框架**，而不是具体工具。指南中的 Prompt 模板可以在任何 AI 工具中使用。

---

### Q9: AI 生成的代码安全吗？

**需要人工审查**。指南强调：
1. **AI 是副驾驶** - 人是决策者，所有 AI 产出必须人工确认
2. **安全检查清单** - 06 章包含代码审查清单
3. **测试验证** - 141 个测试用例验证功能正确性

**必须审查的点**：
- 🔒 敏感数据处理（加密、脱敏）
- 🔒 权限校验（防止越权访问）
- 🔒 SQL 注入防护（使用 ORM）
- 🔒 日志中不包含敏感信息

---

### Q10: Prompt 模板可以直接用吗？

**可以，但建议调整**：
1. **替换技术栈** - 如改用 FastAPI，调整 Prompt 中的框架声明
2. **补充业务上下文** - 粘贴你的数据模型和已有代码
3. **添加约束条件** - 安全、性能、可维护性要求

**好的 Prompt 结构**（CRATE 框架）：
```
C - Context（上下文）：项目背景、技术栈
R - Role（角色）：AI 扮演的专家角色
A - Action（任务）：具体要做什么
T - Template（模板）：输出格式和代码规范
E - Example（示例）：期望输出的参考样例（可选）
```

---

## 四、部署与运维相关

### Q11: 如何部署到生产环境？

**推荐方案**：
1. **云服务器** - 阿里云/腾讯云 ECS + Docker
2. **容器服务** - 阿里云 ACK / 腾讯云 TKE
3. **Serverless** - 阿里云函数计算 / AWS Lambda

**部署步骤**（以云服务器为例）：
```bash
# 1. 购买云服务器（2 核 4G 起步）
# 2. 安装 Docker
curl -fsSL https://get.docker.com | bash

# 3. 克隆项目
git clone https://github.com/frank00827-dotcom/chronic-care.git
cd chronic-care

# 4. 配置环境变量
cp server/.env.example server/.env
# 编辑 .env，修改为生产配置

# 5. 启动服务
docker compose -f deploy/docker-compose.yml up -d

# 6. 配置 Nginx 反向代理
# 7. 配置 HTTPS 证书（Let's Encrypt）
```

---

### Q12: 微信小程序如何上线？

**流程**：
1. 注册微信小程序账号（https://mp.weixin.qq.com）
2. 完成开发者认证
3. 在小程序后台配置服务器域名
4. 使用微信开发者工具上传代码
5. 提交审核（1-3 个工作日）
6. 审核通过后发布

**注意事项**：
- ⚠️ 医疗类小程序可能需要特殊资质
- ⚠️ 服务器必须使用 HTTPS
- ⚠️ 用户隐私政策必须完善

---

## 五、贡献与协作相关

### Q13: 如何贡献代码？

**流程**：
1. Fork 仓库
2. 创建功能分支 (`git checkout -b feature/AmazingFeature`)
3. 开发并测试
4. 提交代码 (`git commit -m "feat: add AmazingFeature"`)
5. 推送到分支 (`git push origin feature/AmazingFeature`)
6. 创建 Pull Request

**代码要求**：
- ✅ 通过 lint 检查
- ✅ 测试全部通过
- ✅ 覆盖率不下降
- ✅ 更新相关文档

详见 [CONTRIBUTING.md](../CONTRIBUTING.md)

---

### Q14: 发现 Bug 怎么办？

**提交 Issue**：
1. 访问 https://github.com/frank00827-dotcom/ai-product-dev-guide/issues
2. 点击 "New issue"
3. 选择 "Bug report" 模板
4. 填写：
   - Bug 描述
   - 复现步骤
   - 预期行为
   - 实际行为
   - 环境信息

**加速解决**：
- 📸 提供截图/录屏
- 📝 提供错误日志
- 🔧 提供修复建议（如有）

---

## 六、商业使用相关

### Q15: 可以商用吗？

**可以**。本项目使用 MIT 协议：
- ✅ 允许商业使用
- ✅ 允许修改
- ✅ 允许分发
- ✅ 允许私有化部署

**唯一要求**：保留原始 LICENSE 文件中的版权声明。

---

### Q16: 可以提供付费培训吗？

**可以**。MIT 协议允许任何形式的商业使用，包括：
- 付费培训课程
- 企业内训
- 咨询服务
- 二次开发销售

**建议**：
- 注明原始项目来源
- 如有改进，欢迎回馈社区

---

## 七、其他问题

### Q17: 后续会更新吗？

**会**。迭代计划：
- **v1.1**（2026-03-15）- Swagger 集成、CI/CD 完善
- **v2.0**（2026-04-01）- 微服务演进、多租户支持

关注仓库 Release 页面获取更新通知。

---

### Q18: 有交流群吗？

**计划中**。目前可以通过以下方式交流：
- GitHub Discussions: https://github.com/frank00827-dotcom/ai-product-dev-guide/discussions
- Issue: https://github.com/frank00827-dotcom/ai-product-dev-guide/issues

---

### Q19: 如何获取更新通知？

**方式**：
1. GitHub 点击 "Watch" 按钮
2. 订阅 Release RSS
3. 关注作者社交媒体（如有）

---

**还有问题？** [提交 Issue](https://github.com/frank00827-dotcom/ai-product-dev-guide/issues/new) 或 [参与讨论](https://github.com/frank00827-dotcom/ai-product-dev-guide/discussions)

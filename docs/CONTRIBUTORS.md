# 贡献者指南

**欢迎为《AI 驱动产品开发指南》贡献！** 🎉

---

## 🚀 快速开始

### 1. Fork 仓库

访问 https://github.com/frank00827-dotcom/ai-product-dev-guide
点击右上角 "Fork" 按钮

### 2. 克隆到本地

```bash
git clone https://github.com/YOUR_USERNAME/ai-product-dev-guide.git
cd ai-product-dev-guide/ai-product-dev-guide
```

### 3. 创建分支

```bash
git checkout -b feature/your-feature-name
```

### 4. 开发并提交

```bash
# 修改文件
# ...

# 提交
git add .
git commit -m "feat: add your feature description"
```

### 5. 推送并创建 PR

```bash
git push origin feature/your-feature-name
```

然后访问 GitHub 创建 Pull Request。

---

## 📝 贡献类型

我们欢迎以下类型的贡献：

### 🐛 Bug 修复
- 文档错误（错别字、链接失效、命令错误）
- 代码 bug（功能异常、测试失败）
- 配置问题（Docker、CI/CD）

### ✨ 新功能
- 新的业务场景案例
- 新的 Prompt 模板
- 新的测试用例
- 工具/脚本增强

### 📖 文档改进
- 文字润色（更清晰的表达）
- 示例补充（更丰富的代码示例）
- 截图/图表更新
- 翻译（多语言支持）

### 🧪 测试补充
- 单元测试
- 集成测试
- E2E 测试
- 性能测试

### 🎨 设计与 UX
- Logo 设计
- 文档排版优化
- 图表/流程图绘制
- 视频/动画制作

---

## 🔧 开发环境设置

### 指南文档仓库

```bash
# 克隆仓库
git clone https://github.com/YOUR_USERNAME/ai-product-dev-guide.git
cd ai-product-dev-guide/ai-product-dev-guide

# 验证文档结构
ls -la
```

### 实战项目仓库（如需修改代码）

```bash
# 克隆仓库
git clone https://github.com/YOUR_USERNAME/chronic-care.git
cd chronic-care

# 启动开发环境
docker compose -f deploy/docker-compose.yml up -d

# 后端设置
cd server
python -m venv venv
source venv/bin/activate
pip install -r requirements.txt

# 前端设置
cd client
npm install
```

---

## 📏 代码/文档规范

### Commit Message 规范

```
<type>(<scope>): <subject>

# type 枚举:
# feat     - 新功能
# fix      - 修复 bug
# docs     - 文档变更
# style    - 格式调整（不影响代码逻辑）
# refactor - 重构（非 bug 修复、非新功能）
# test     - 测试相关
# chore    - 构建/工具/配置变更
```

**示例**：
```
docs(05 章): 补充 Flask 模块生成示例
fix(06 章): 修正测试命令错误
feat(appendix): 新增 Prompt 模板 5 个
refactor(server): 重构认证模块结构
test(client): 添加登录页面测试用例
```

### 文档规范

- 使用 Markdown 格式
- 代码块标注语言类型（\`\`\`python）
- 命令示例确保可复制执行
- 新增章节更新文档导航
- 图片使用相对路径，放在 `docs/images/` 目录

### Python 代码规范

- 遵循 PEP 8
- 使用 `ruff` 和 `black` 格式化
- 函数/类需要 docstring
- 类型注解（Type Hints）

```bash
# 格式化代码
cd server
black .
ruff check --fix .
```

### TypeScript 代码规范

- 使用 ESLint + Prettier
- 严格模式（strict: true）
- 接口/类型定义清晰

```bash
# 格式化代码
cd client
npm run lint
npm run format
```

---

## ✅ 提交前检查清单

提交 PR 前请确认：

- [ ] 代码通过 lint 检查
- [ ] 测试全部通过
- [ ] 测试覆盖率不下降
- [ ] 更新了相关文档
- [ ] Commit message 规范
- [ ] 无敏感信息泄露（API Key、密码等）
- [ ] 新增功能有对应测试

---

## 🔍 审查流程

### 1. CI 自动检查

提交 PR 后，GitHub Actions 会自动运行：
- ✅ 文件完整性检查
- ✅ 链接检查（如启用）
- ✅ 测试运行（如修改代码）

### 2. Maintainer 审查

Maintainer 会审查：
- 📝 内容准确性
- 🔗 文档一致性
- 🧪 测试覆盖率
- 🎯 与项目目标对齐

### 3. 合并

审查通过后：
- PR 合并到 main 分支
- 贡献者名字进入贡献者名单
- 自动发布到 GitHub Pages（如配置）

---

## 🏆 贡献者荣誉

### 贡献者名单

所有贡献者会被记录在：
- GitHub Contributors 页面：https://github.com/frank00827-dotcom/ai-product-dev-guide/graphs/contributors
- README.md 贡献者章节（如贡献达到一定规模）

### 贡献等级

| 等级 | 要求 | 权益 |
|------|------|------|
| 🥉 青铜 | 1 次 PR 合并 | 贡献者名单 |
| 🥈 白银 | 3 次 PR 合并 | 贡献者名单 + 特别感谢 |
| 🥇 黄金 | 5 次 PR 合并 | 贡献者名单 + Collaborator 邀请 |
| 💎 钻石 | 10 次 PR 合并 | Collaborator + 决策参与权 |

---

## 💡 新手任务

第一次贡献？从这些任务开始：

### good first issues

访问 https://github.com/frank00827-dotcom/ai-product-dev-guide/issues?q=is%3Aissue+is%3Aopen+label%3A%22good+first+issue%22

这些任务特点：
- ✅ 难度低
- ✅ 范围清晰
- ✅ 有详细说明
- ✅ Maintainer 会提供指导

### 推荐任务

| 任务 | 难度 | 说明 |
|------|------|------|
| 修正文档错别字 | ⭐ | 阅读文档，发现并修正错误 |
| 补充示例代码 | ⭐⭐ | 为某个章节添加代码示例 |
| 新增测试用例 | ⭐⭐ | 为某个函数添加边界测试 |
| 翻译文档 | ⭐⭐ | 翻译成英文或其他语言 |
| 绘制架构图 | ⭐⭐⭐ | 使用 Mermaid 绘制流程图 |

---

## ❓ 常见问题

### Q: 我不知道从哪里开始

**A**: 从 good first issues 开始，或者：
1. 阅读文档，找到你觉得可以改进的地方
2. 创建 Issue 说明你的想法
3. 我们会给你指导

### Q: 我的 PR 多久会被审查？

**A**: 通常 1-3 个工作日。如果超过一周没有回复，可以：
- 在 PR 中 @maintainer
- 在 Discussions 中提醒

### Q: 我的 PR 被拒绝了怎么办？

**A**: 别灰心！审查意见是为了保证质量：
1. 阅读审查意见，理解原因
2. 根据意见修改
3. 如有异议，可以讨论

### Q: 我可以一次提交多个修改吗？

**A**: 建议一个 PR 只做一件事：
- ✅ 一个功能/修复一个 PR
- ❌ 多个不相关的修改放一个 PR

这样便于审查和回滚。

---

## 📞 需要帮助？

- [创建 Issue](https://github.com/frank00827-dotcom/ai-product-dev-guide/issues/new)
- [参与讨论](https://github.com/frank00827-dotcom/ai-product-dev-guide/discussions)
- [邮件联系](mailto:frank00827@gmail.com)

---

## 🙏 感谢所有贡献者！

<a href="https://github.com/frank00827-dotcom/ai-product-dev-guide/graphs/contributors">
  <img src="https://contrib.rocks/image?repo=frank00827-dotcom/ai-product-dev-guide" />
</a>

**感谢你的贡献！** 🎉

---

*本指南采用 MIT 协议开源，详情见 [LICENSE](../LICENSE)*

# 贡献指南

感谢你为《AI 驱动产品开发指南》贡献！🎉

## 如何贡献

### 1. 报告问题（Issues）

发现问题或有改进建议？[创建 Issue](https://github.com/frank00827-dotcom/ai-product-dev-guide/issues)

**报告问题时请包含**：
- 问题描述（清晰简洁）
- 复现步骤
- 预期行为 vs 实际行为
- 环境信息（Node/Python 版本、操作系统）

### 2. 提交修改（Pull Requests）

**PR 流程**：
1. Fork 本仓库
2. 创建功能分支 (`git checkout -b feature/AmazingFeature`)
3. 提交修改 (`git commit -m 'Add some AmazingFeature'`)
4. 推送到分支 (`git push origin feature/AmazingFeature`)
5. 创建 Pull Request

### 3. 贡献内容类型

我们欢迎以下类型的贡献：

| 类型 | 说明 | 示例 |
|------|------|------|
| 🐛 Bug 修复 | 修正文档错误、代码 bug | 链接失效、命令错误 |
| ✨ 新功能 | 新增章节、Prompt 模板、案例 | 新的业务场景案例 |
| 📝 文档改进 | 文字润色、示例补充、截图更新 | 更清晰的步骤说明 |
| 🧪 测试补充 | 新增测试用例、提高覆盖率 | 边界条件测试 |
| 🚀 性能优化 | 优化代码性能、减少依赖 | 更高效的算法 |
| 🔒 安全加固 | 修复安全漏洞、加强验证 | SQL 注入防护 |

## 开发环境设置

### 指南文档仓库

```bash
# 克隆仓库
git clone https://github.com/frank00827-dotcom/ai-product-dev-guide.git
cd ai-product-dev-guide

# 验证文档结构
ls -la ai-product-dev-guide/
```

### 实战项目仓库（chronic-care）

```bash
# 克隆仓库
git clone https://github.com/frank00827-dotcom/chronic-care.git
cd chronic-care

# 启动开发环境
docker compose -f deploy/docker-compose.yml up -d

# 后端设置
cd server
python -m venv venv
source venv/bin/activate
pip install -r requirements.txt

# 前端设置
cd ../client
npm install
```

## 代码/文档规范

### 文档规范

- 使用 Markdown 格式
- 代码块标注语言类型（\`\`\`python）
- 命令示例确保可复制执行
- 新增章节更新文档导航

### Commit Message 规范

```
<type>(<scope>): <subject>

# type 枚举:
# feat     - 新功能
# fix      - 修复 bug
# docs     - 文档变更
# style    - 格式调整
# refactor - 重构
# test     - 测试相关
# chore    - 构建/工具变更
```

**示例**：
```
docs(05 章): 补充 Flask 模块生成示例
fix(06 章): 修正测试命令错误
feat(appendix): 新增 Prompt 模板 5 个
```

## 审查流程

1. **CI 检查** - 自动验证文件完整性
2. **Maintainer 审查** - 内容准确性、一致性
3. **合并** - 合入 main 分支

## 常见问题

### Q: 我可以贡献自己的项目案例吗？

A: 当然可以！请将案例放在 `examples/` 目录，并更新 README 说明。

### Q: 如何测试我的修改？

A: 
- 文档修改：确保链接有效、命令可执行
- 代码修改：运行测试确保覆盖率不下降

### Q: 多久能收到回复？

A: 通常 1-3 个工作日内会有 Maintainer 审查。

## 致谢

所有贡献者都会被记录在 [contributors](https://github.com/frank00827-dotcom/ai-product-dev-guide/graphs/contributors) 页面。

---

**首次贡献？** 从 [good first issues](https://github.com/frank00827-dotcom/ai-product-dev-guide/issues?q=is%3Aissue+is%3Aopen+label%3A%22good+first+issue%22) 开始！

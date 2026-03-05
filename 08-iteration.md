# 08 - 迭代与持续改进

## 8.1 数据驱动迭代框架

### 核心指标体系（North Star Metrics）

```
一级指标（北极星）:
  月活跃患者数 (MAU) × 人均健康数据录入频次

二级指标（驱动因子）:
┌──────────────┬──────────────┬──────────────┬──────────────┐
│   获客        │   激活        │   留存        │   健康价值    │
├──────────────┼──────────────┼──────────────┼──────────────┤
│ 新注册用户数  │ 首次录入率    │ 7日留存率     │ 血糖达标率    │
│ 注册转化率    │ 建档完成率    │ 30日留存率    │ 随访完成率    │
│ 渠道来源分布  │ 首日任务完成率│ 周均使用天数  │ 用药依从性    │
└──────────────┴──────────────┴──────────────┴──────────────┘
```

### 埋点规范

```typescript
// 埋点事件定义（前端统一使用此格式）
interface TrackEvent {
  event: string;        // 事件名: module.action
  properties: {
    page: string;       // 当前页面
    module: string;     // 所属模块
    [key: string]: any; // 自定义属性（禁止包含敏感数据）
  };
}

// 核心埋点事件清单
const TRACK_EVENTS = {
  // 健康数据模块
  'health.record_start':    '开始录入健康数据',
  'health.record_submit':   '提交健康数据',
  'health.record_abandon':  '放弃录入（退出页面）',
  'health.trend_view':      '查看趋势图',

  // 随访模块
  'followup.remind_click':  '点击随访提醒',
  'followup.form_start':    '开始填写随访问卷',
  'followup.form_submit':   '提交随访问卷',

  // 数字疗法模块
  'therapy.task_view':      '查看今日任务',
  'therapy.task_complete':  '完成任务打卡',
  'therapy.edu_click':      '点击健康教育内容',
  'therapy.edu_finish':     '完成阅读/观看',
};
```

## 8.2 AI 辅助数据分析

### Prompt 模板：用户行为分析

```markdown
# 角色
你是数据分析师，擅长产品数据分析和用户行为洞察。

# 数据
以下是慢病管家最近30天的核心指标数据：
[粘贴数据表格或 CSV]

# 任务
1. 识别关键趋势和异常点
2. 分析留存率下降/上升的可能原因
3. 找出功能使用漏斗中的最大流失环节
4. 给出 3-5 条可执行的产品优化建议
5. 每条建议标注预期影响（高/中/低）和实施复杂度

# 约束
- 建议必须基于数据，不做无依据的推测
- 考虑慢病患者的特殊性（年龄大、使用频率与病情相关）
- 区分「产品问题」和「用户自然流失」
```

### Prompt 模板：版本规划

```markdown
# 角色
你是产品经理，正在规划慢病管家的下一个迭代版本。

# 输入
1. 当前版本的用户反馈汇总: [粘贴]
2. 数据分析结论: [粘贴上一步的分析结果]
3. 技术债务清单: [粘贴]
4. 团队可用资源: [描述]

# 任务
制定下一个迭代的版本计划，包含：
1. 版本目标（聚焦解决什么问题）
2. 功能列表（按优先级排序，标注 Must/Should/Could）
3. 技术债务偿还计划
4. 风险评估

# 约束
- 每个迭代聚焦 1-2 个核心目标，不贪多
- Must 级别功能不超过 5 个
- 技术债务偿还占比不超过 20%
```

## 8.3 A/B 测试框架

### 适合 A/B 测试的场景

| 场景 | 测试变量 | 核心指标 |
|------|----------|----------|
| 血糖录入流程 | 步骤数（1步 vs 2步） | 录入完成率 |
| 用药提醒文案 | 关怀型 vs 提醒型 | 点击率、用药依从性 |
| 健康教育推荐 | 按疾病类型 vs 按行为偏好 | 阅读完成率 |
| 任务激励机制 | 连续打卡 vs 积分制 | 7日任务完成率 |
| 首次建档流程 | 一次性填写 vs 分步引导 | 建档完成率 |

### 实现方案

```typescript
// 轻量级 A/B 测试实现（基于用户 ID 哈希分桶）

interface Experiment {
  id: string;
  name: string;
  variants: { id: string; weight: number }[];
  status: 'running' | 'paused' | 'completed';
}

function getVariant(userId: string, experiment: Experiment): string {
  const hash = simpleHash(userId + experiment.id);
  const bucket = hash % 100;

  let cumulative = 0;
  for (const variant of experiment.variants) {
    cumulative += variant.weight;
    if (bucket < cumulative) return variant.id;
  }
  return experiment.variants[0].id;
}

// 前端使用
const variant = useExperiment('glucose-input-flow');
if (variant === 'one-step') {
  return <OneStepGlucoseInput />;
} else {
  return <TwoStepGlucoseInput />;
}
```

## 8.4 用户反馈闭环

### 反馈收集渠道

```
渠道                  适用场景              处理方式
──────────────────────────────────────────────────────
小程序内反馈入口      功能建议、Bug报告      自动归类 + 人工处理
随访问卷附加题        使用体验评分           自动统计
微信客服消息          紧急问题              人工即时响应
应用商店评价          公开评价              定期回复
用户访谈              深度需求挖掘           产品经理主导
```

### AI 辅助反馈分析

```markdown
# Prompt 模板：用户反馈聚类分析

# 角色
你是用户研究专家。

# 数据
以下是最近一个月收集的用户反馈（共 N 条）：
[粘贴反馈数据]

# 任务
1. 将反馈按主题聚类（如：功能请求、Bug报告、体验问题、表扬）
2. 每个类别统计数量和占比
3. 提取 Top 5 高频问题
4. 对每个高频问题给出：
   - 问题本质分析
   - 影响范围评估
   - 建议的解决方案
   - 优先级建议

# 约束
- 区分「个别用户的特殊需求」和「普遍性问题」
- 关注老年用户的特殊反馈（操作困难、字体太小等）
- 医疗相关反馈需要标注（可能涉及产品安全性）
```

## 8.5 AI 辅助能力持续提升

### 团队 AI 能力成熟度模型

```
Level 1 — 初始使用
  ✓ 能使用 AI 生成简单代码片段
  ✓ 能使用 AI 回答技术问题
  目标: 所有团队成员达到此级别

Level 2 — 流程集成
  ✓ AI 工具集成到日常开发流程
  ✓ 建立了 Prompt 模板库
  ✓ AI 代码审查成为标准流程
  目标: 项目启动 1 个月内达到

Level 3 — 质量优化
  ✓ 能评估 AI 产出质量并持续优化 Prompt
  ✓ 建立了 AI 产出的质量基线
  ✓ 团队共享最佳实践
  目标: 项目启动 3 个月内达到

Level 4 — 创新应用
  ✓ 探索 AI 在非编码环节的应用（需求分析、架构决策）
  ✓ 自定义 AI 工作流和自动化
  ✓ 向其他团队输出方法论
  目标: 持续演进
```

### Prompt 模板库维护机制

```
每两周一次 Prompt Review:
1. 收集本周期内所有团队成员使用的 Prompt
2. 评估每个 Prompt 的产出质量
3. 将高质量 Prompt 纳入模板库
4. 淘汰低效 Prompt
5. 分享 Prompt 优化技巧

模板库组织结构:
appendix/prompt-templates.md
├── 产品发现类
├── 架构设计类
├── 代码生成类（按模块）
├── 测试生成类
├── 代码审查类
├── 数据分析类
└── 文档生成类
```

## 8.6 阶段交付物检查清单

- [ ] 核心指标体系定义（北极星指标 + 二级指标）
- [ ] 埋点方案文档（事件清单 + 属性定义）
- [ ] 数据看板搭建（日活、留存、功能使用率）
- [ ] A/B 测试框架实现
- [ ] 用户反馈收集渠道建立
- [ ] 迭代规划模板
- [ ] Prompt 模板库 Review 机制建立
- [ ] 团队 AI 能力评估基线

---

**上一步**: [07 - 部署与运维](./07-deployment-ops.md)
**下一步**: [附录 - Prompt模板库](./appendix/prompt-templates.md)
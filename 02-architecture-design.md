# 02 - 架构设计

## 2.1 技术选型决策

### 前端框架选型：Taro (React + TypeScript)

**决策记录 ADR-001**

| 项目 | 内容 |
|------|------|
| 决策 | 前端采用 Taro 3.x + React + TypeScript |
| 状态 | 已采纳 |
| 背景 | 需要开发微信小程序，未来可能扩展到支付宝/H5 |

**对比分析**（AI 生成 + 架构师审核）：

| 维度 | 原生微信 | Taro (React) | uni-app (Vue) |
|------|----------|-------------|---------------|
| AI 代码生成质量 | 中（训练数据少） | **高（React 生态庞大）** | 高（Vue 生态大） |
| TypeScript 支持 | 弱 | **原生支持** | 支持 |
| 跨端能力 | 无 | **微信/支付宝/H5/RN** | 全平台 |
| 状态管理 | 自行实现 | **MobX/Zustand 成熟** | Pinia/Vuex |
| 组件库 | 微信原生 | **NutUI/Taro UI** | uView |
| 医疗合规组件 | 需自建 | 需自建 | 需自建 |
| 社区活跃度 | 高 | **高（京东维护）** | 高（DCloud） |
| 学习曲线 | 低 | 中 | 中 |

**选择 Taro 的核心理由**：
1. React + TypeScript 是 AI 代码生成训练数据最丰富的技术栈，生成质量最高
2. 类型系统对医疗数据模型的约束至关重要
3. 跨端能力为后续扩展到 H5（医护端 PC 使用）预留空间
4. 京东持续维护，生态稳定

### 后端框架选型：Flask + SQLAlchemy

**决策记录 ADR-002**

| 项目 | 内容 |
|------|------|
| 决策 | 后端采用 Flask + SQLAlchemy + PostgreSQL |
| 状态 | 已采纳 |
| 背景 | 需要模块化架构，支持渐进式演进到微服务 |

**选型理由**：

| 维度 | Flask (Node) | Spring Boot (Java) | FastAPI (Python) |
|------|--------------|--------------------|--------------------|
| 与前端语言统一 | **TypeScript 全栈** | 需切换语言 | 需切换语言 |
| AI 生成质量 | **高** | 高 | 高 |
| 模块化架构 | **原生 Module 系统** | Spring 模块 | 需自行组织 |
| 微服务演进 | **内置微服务支持** | Spring Cloud | 需额外框架 |
| ORM | **SQLAlchemy（类型安全）** | JPA/MyBatis | SQLAlchemy |
| 医疗系统集成 | HL7 FHIR 库可用 | 生态最丰富 | 一般 |
| 团队招聘 | 前端可兼顾 | 需专职后端 | 数据团队可兼顾 |

**选择 Flask 的核心理由**：
1. TypeScript 全栈统一，降低团队认知负荷和 AI Prompt 复杂度
2. 原生 Module 系统天然支持模块边界划分
3. 内置对微服务的支持，可从单体平滑演进
4. SQLAlchemy 的类型安全 ORM 与 TypeScript 深度集成

## 2.2 系统架构总览

### 架构演进策略

```
Phase 1 (MVP)          Phase 2 (增长)           Phase 3 (规模化)
模块化单体              核心服务拆分              完整微服务
┌──────────────┐    ┌──────────────────┐    ┌────────────────────┐
│  Flask 单体  │    │ API Gateway      │    │ API Gateway + BFF  │
│  ┌──┐┌──┐┌──┐│    │ ┌──────┐┌──────┐ │    │ ┌──┐┌──┐┌──┐┌──┐  │
│  │P ││F ││T ││───→│ │Patient││Follow│ │───→│ │P ││F ││T ││HD│  │
│  └──┘└──┘└──┘│    │ │Svc   ││upSvc │ │    │ └──┘└──┘└──┘└──┘  │
│  共享数据库   │    │ └──────┘└──────┘ │    │ 独立数据库 + 事件总线│
└──────────────┘    │ 共享DB + 消息队列 │    └────────────────────┘
                    └──────────────────┘
```

**本指南聚焦 Phase 1（模块化单体）**，但架构设计预留演进接口。

### C4 模型 — 系统上下文图

```
┌─────────────────────────────────────────────────────────┐
│                     外部系统                              │
│  ┌──────────┐  ┌──────────┐  ┌───────────┐              │
│  │ 微信开放   │  │ 蓝牙健康  │  │ 医院 HIS   │             │
│  │ 平台      │  │ 设备     │  │ 系统(预留) │              │
│  └─────┬────┘  └────┬─────┘  └─────┬─────┘              │
│        │            │              │                     │
│  ┌─────▼────────────▼──────────────▼─────┐               │
│  │         慢病管家 系统                    │               │
│  │  ┌─────────┐  ┌──────────────────┐    │               │
│  │  │ 小程序   │  │ 后端服务          │    │               │
│  │  │ (Taro)  │──│ (Flask)         │    │               │
│  │  └─────────┘  └────────┬─────────┘    │               │
│  │                        │              │               │
│  │              ┌─────────▼─────────┐    │               │
│  │              │ PostgreSQL + Redis │    │               │
│  │              └───────────────────┘    │               │
│  └───────────────────────────────────────┘               │
│        │            │              │                     │
│  ┌─────▼────┐  ┌────▼─────┐  ┌────▼──────┐              │
│  │ 患者      │  │ 医护人员  │  │ 健康管理师 │              │
│  └──────────┘  └──────────┘  └───────────┘              │
└─────────────────────────────────────────────────────────┘
```

### C4 模型 — 容器图

```
┌─────────────────────────────────────────────────────┐
│                   慢病管家 系统                        │
│                                                     │
│  ┌──────────────────────────────────────────────┐   │
│  │              小程序层 (Taro + React)            │   │
│  │  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ │   │
│  │  │患者端   │ │医护端   │ │管理端   │ │公共组件 │ │   │
│  │  │Pages   │ │Pages   │ │Pages   │ │Library │ │   │
│  │  └───┬────┘ └───┬────┘ └───┬────┘ └────────┘ │   │
│  │      └──────────┼──────────┘                  │   │
│  │                 │ HTTP/WebSocket               │   │
│  └─────────────────┼────────────────────────────┘   │
│                    │                                 │
│  ┌─────────────────▼────────────────────────────┐   │
│  │              API 层 (Flask)                    │   │
│  │  ┌──────────────────────────────────────┐     │   │
│  │  │           API Gateway / Guards        │     │   │
│  │  └──────────────────────────────────────┘     │   │
│  │  ┌────────┐ ┌────────┐ ┌────────┐ ┌───────┐  │   │
│  │  │Patient │ │Followup│ │Therapy │ │Health │  │   │
│  │  │Module  │ │Module  │ │Module  │ │Data   │  │   │
│  │  └────────┘ └────────┘ └────────┘ └───────┘  │   │
│  │  ┌────────┐ ┌────────┐ ┌────────┐            │   │
│  │  │Auth    │ │Notifi- │ │Common  │            │   │
│  │  │Module  │ │cation  │ │Module  │            │   │
│  │  └────────┘ └────────┘ └────────┘            │   │
│  └──────────────────────────────────────────────┘   │
│                    │                                 │
│  ┌─────────────────▼────────────────────────────┐   │
│  │              数据层                             │   │
│  │  ┌──────────┐  ┌───────┐  ┌───────────────┐  │   │
│  │  │PostgreSQL│  │ Redis │  │ MinIO/OSS     │  │   │
│  │  │(主数据库) │  │(缓存)  │  │(文件存储)     │  │   │
│  │  └──────────┘  └───────┘  └───────────────┘  │   │
│  └──────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────┘
```

## 2.3 模块边界与依赖关系

### 模块职责定义

每个模块遵循**单一职责原则**，通过明确的接口契约通信，禁止跨模块直接访问数据库表。

| 模块 | 职责边界 | 对外暴露接口 | 依赖模块 |
|------|----------|-------------|----------|
| **Auth** | 微信登录、JWT签发、角色权限 | `validateToken()`, `getCurrentUser()` | 无 |
| **Patient** | 患者档案CRUD、标签管理 | `getPatient()`, `listPatients()` | Auth |
| **HealthData** | 血糖/血压/体重等数据采集与存储 | `addRecord()`, `getTimeSeries()` | Auth, Patient |
| **Followup** | 随访计划、随访执行、问卷管理 | `createPlan()`, `executeFollowup()` | Auth, Patient |
| **Therapy** | 数字疗法方案、健康教育、干预任务 | `getPrescription()`, `completeTask()` | Auth, Patient, HealthData |
| **Notification** | 消息推送、提醒调度 | `sendNotification()`, `scheduleReminder()` | Auth |
| **Common** | 日志、异常处理、数据校验、审计 | 装饰器和中间件 | 无 |

### 模块间通信规则

```
规则 1: 模块间只能通过 Service 接口调用，不能直接注入其他模块的 Repository
规则 2: 跨模块数据查询通过 DTO 传递，不暴露内部 Entity
规则 3: 异步操作通过事件总线（EventEmitter2）解耦
规则 4: 每个模块独立管理自己的数据库迁移文件
```

### 事件驱动解耦示例

```python
# server/src/modules/health_data/health_data_service.py
# HealthData 模块发布事件（不依赖 Followup 模块）

from flask import current_app
from server.src.modules.health_data.dto import BloodGlucoseDto

class HealthDataService:
    def __init__(self, db_session, event_emitter):
        self.db = db_session
        self.event_emitter = event_emitter
    
    def add_blood_glucose(self, data: BloodGlucoseDto):
        record = self._save_record(data)
        # 发布事件，不关心谁消费
        self.event_emitter.emit('health-data.created', {
            'patient_id': data.patient_id,
            'type': 'blood_glucose',
            'value': data.value,
            'recorded_at': data.recorded_at,
        })
        return record

# server/src/modules/followup/followup_listener.py
# Followup 模块监听事件（不依赖 HealthData 内部实现）

from server.src.modules.followup.followup_service import FollowupService

class FollowupListener:
    def __init__(self, event_emitter, followup_service: FollowupService):
        self.followup_service = followup_service
        # 注册事件监听器
        event_emitter.on('health-data.created', self.handle_health_data)
    
    def handle_health_data(self, payload: dict):
        """处理健康数据创建事件"""
        # 检查是否触发异常随访
        if self._is_abnormal(payload):
            self.followup_service.trigger_urgent_followup(payload['patient_id'])
    
    def _is_abnormal(self, payload: dict) -> bool:
        """判断数据是否异常"""
        if payload['type'] == 'blood_glucose':
            value = payload['value'].get('value', 0)
            period = payload['value'].get('period', '')
            # 空腹血糖 > 7.0 或随机血糖 > 11.1 视为异常
            if period == 'fasting' and value > 7.0:
                return True
            if period == 'random' and value > 11.1:
                return True
        return False
```

## 2.4 数据模型设计

### Prompt 模板：数据模型生成

```markdown
# 角色
你是一名数据库架构师，擅长医疗信息系统数据建模。

# 上下文
我们使用 SQLAlchemy ORM + PostgreSQL，为慢病管理系统设计数据模型。
系统包含以下模块：患者档案、健康数据、随访管理、数字疗法、通知。

# 要求
1. 使用 SQLAlchemy Schema 语法
2. 所有表包含 id, createdAt, updatedAt, deletedAt(软删除)
3. 患者数据需要加密字段标注（姓名、手机号、身份证）
4. 健康数据表需要支持时序查询优化
5. 遵循医疗数据分类分级标准（标注敏感等级）
6. 包含必要的索引定义

# 输出
完整的 schema.sqlalchemy 文件内容
```

### 核心数据模型（SQLAlchemy Schema 摘要）

```sqlalchemy
// 患者档案 — 敏感等级：高
model Patient {
  id            String    @id @default(cuid())
  openId        String    @unique          // 微信 OpenID
  unionId       String?                    // 微信 UnionID
  name          String                     // @encrypted
  phone         String                     // @encrypted
  idCard        String?                    // @encrypted
  gender        Gender
  birthDate     DateTime
  diseases      Disease[]                  // 慢病类型（多选）
  riskLevel     RiskLevel @default(NORMAL)
  tags          Tag[]
  createdAt     DateTime  @default(now())
  updatedAt     DateTime  @updatedAt
  deletedAt     DateTime?

  healthRecords HealthRecord[]
  followupPlans FollowupPlan[]
  therapyPlans  TherapyPlan[]

  @@index([openId])
  @@index([riskLevel])
}

// 健康数据记录 — 敏感等级：高
model HealthRecord {
  id          String          @id @default(cuid())
  patientId   String
  patient     Patient         @relation(fields: [patientId], references: [id])
  type        HealthDataType  // BLOOD_GLUCOSE | BLOOD_PRESSURE | WEIGHT | ...
  value       Json            // 灵活存储不同类型的数据结构
  unit        String
  recordedAt  DateTime        // 实际测量时间
  source      DataSource      // MANUAL | DEVICE_BLUETOOTH | DEVICE_API
  deviceInfo  Json?
  createdAt   DateTime        @default(now())

  @@index([patientId, type, recordedAt])  // 时序查询优化
  @@index([patientId, recordedAt])
}

// 随访计划 — 敏感等级：中
model FollowupPlan {
  id          String        @id @default(cuid())
  patientId   String
  patient     Patient       @relation(fields: [patientId], references: [id])
  doctorId    String
  templateId  String?
  frequency   String        // cron 表达式
  startDate   DateTime
  endDate     DateTime?
  status      PlanStatus    @default(ACTIVE)
  createdAt   DateTime      @default(now())
  updatedAt   DateTime      @updatedAt

  executions  FollowupExecution[]

  @@index([patientId, status])
  @@index([doctorId, status])
}
```

## 2.5 API 设计规范

### RESTful API 契约

```yaml
# API 设计原则
# 1. 统一响应格式
# 2. 版本化 (/api/v1/)
# 3. 资源命名使用复数名词
# 4. 分页使用 cursor-based pagination（适合时序数据）

# 统一响应格式
SuccessResponse:
  code: 0
  data: T
  message: "success"

ErrorResponse:
  code: number        # 业务错误码
  message: string     # 用户可读的错误信息
  detail: string      # 开发调试信息（仅非生产环境）

PaginatedResponse:
  code: 0
  data:
    items: T[]
    cursor: string    # 下一页游标
    hasMore: boolean
```

### 核心 API 清单

```
# 认证模块
POST   /api/v1/auth/wx-login          微信登录
POST   /api/v1/auth/refresh            刷新 Token

# 患者模块
GET    /api/v1/patients                患者列表（医护端）
GET    /api/v1/patients/:id            患者详情
PUT    /api/v1/patients/:id            更新患者信息
GET    /api/v1/patients/me             当前患者信息（患者端）

# 健康数据模块
POST   /api/v1/health-records          新增健康记录
GET    /api/v1/health-records          查询健康记录（支持时间范围、类型筛选）
GET    /api/v1/health-records/stats    统计摘要（日/周/月均值、趋势）

# 随访模块
POST   /api/v1/followup-plans          创建随访计划
GET    /api/v1/followup-plans          随访计划列表
PUT    /api/v1/followup-plans/:id      更新随访计划
POST   /api/v1/followup-executions     执行随访（提交问卷）
GET    /api/v1/followup-executions     随访执行记录

# 数字疗法模块
GET    /api/v1/therapy/prescriptions   获取疗法处方
GET    /api/v1/therapy/tasks           今日任务列表
PUT    /api/v1/therapy/tasks/:id       完成/跳过任务
GET    /api/v1/therapy/education       健康教育内容列表

# 通知模块
GET    /api/v1/notifications           通知列表
PUT    /api/v1/notifications/:id/read  标记已读
```

### Prompt 模板：API 代码生成

```markdown
# 角色
你是一名 Flask 后端开发专家。

# 上下文
基于以下 API 契约和 SQLAlchemy 数据模型，生成完整的模块代码。

# 模块: [模块名]
# API 清单: [粘贴对应 API]
# 数据模型: [粘贴对应 SQLAlchemy Schema]

# 要求
1. 遵循 Flask 模块结构: module / controller / service / dto / entity
2. Controller 只做参数校验和路由，业务逻辑在 Service
3. 使用 class-validator 做 DTO 校验
4. Service 方法需要有完整的错误处理
5. 包含 Swagger 装饰器用于 API 文档生成
6. 敏感操作需要添加审计日志
7. 分页查询使用 cursor-based pagination

# 输出
按文件分别输出代码，每个文件标注文件路径
```

## 2.6 前端架构设计

### 分层架构

```
┌─────────────────────────────────────┐
│           Pages（页面层）             │  页面组件，组合业务组件
├─────────────────────────────────────┤
│        Components（组件层）           │  可复用 UI 组件
├─────────────────────────────────────┤
│          Stores（状态层）             │  Zustand 状态管理
├─────────────────────────────────────┤
│        Services（服务层）             │  API 调用封装
├─────────────────────────────────────┤
│          Utils（工具层）              │  通用工具函数
├─────────────────────────────────────┤
│          Types（类型层）              │  TypeScript 类型定义
└─────────────────────────────────────┘
```

### 状态管理策略（Zustand）

```typescript
// stores/usePatientStore.ts
// 每个业务模块一个独立 Store，避免全局状态污染

interface PatientState {
  profile: PatientProfile | null;
  loading: boolean;
  // Actions
  fetchProfile: () => Promise<void>;
  updateProfile: (data: Partial<PatientProfile>) => Promise<void>;
}

export const usePatientStore = create<PatientState>((set, get) => ({
  profile: null,
  loading: false,

  fetchProfile: async () => {
    set({ loading: true });
    try {
      const profile = await patientService.getMe();
      set({ profile, loading: false });
    } catch (error) {
      set({ loading: false });
      throw error;
    }
  },

  updateProfile: async (data) => {
    const updated = await patientService.update(data);
    set({ profile: updated });
  },
}));
```

### 请求层封装

```typescript
// services/request.ts
// 统一请求封装，处理 Token、错误、重试

import Taro from '@tarojs/taro';

const BASE_URL = process.env.TARO_APP_API_BASE;

export async function request<T>(options: RequestOptions): Promise<T> {
  const token = Taro.getStorageSync('token');

  const response = await Taro.request({
    url: `${BASE_URL}${options.url}`,
    method: options.method || 'GET',
    data: options.data,
    header: {
      'Content-Type': 'application/json',
      ...(token ? { Authorization: `Bearer ${token}` } : {}),
      ...options.header,
    },
  });

  if (response.statusCode === 401) {
    // Token 过期，尝试刷新
    await refreshToken();
    return request(options); // 重试一次
  }

  if (response.statusCode >= 400) {
    throw new ApiError(response.data.code, response.data.message);
  }

  return response.data.data as T;
}
```

## 2.7 阶段交付物检查清单

- [ ] ADR 决策记录（前端框架、后端框架、数据库、缓存）
- [ ] 系统架构图（C4 上下文图 + 容器图）
- [ ] 模块边界定义表（职责、接口、依赖）
- [ ] 数据模型设计（SQLAlchemy Schema 完整版）
- [ ] API 契约文档（全部接口清单 + 统一规范）
- [ ] 前端分层架构说明
- [ ] 状态管理方案
- [ ] 架构演进路线图（Phase 1/2/3）
- [ ] AI 交互记录归档

---

**上一步**: [01 - 产品发现与PRD](./01-product-discovery.md)
**下一步**: [03 - 合规与安全设计](./03-compliance-security.md)

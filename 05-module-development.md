# 05 - 模块开发实战

> 以慢性管家（ChronicCare）项目为案例，记录 AI 辅助开发的全过程

**版本**: 1.0  
**最后更新**: 2026-03-05  
**技术栈**: Taro(React+TS) + Flask + SQLAlchemy + PostgreSQL + MinIO

---

## 5.1 开发顺序规划

### 模块依赖拓扑排序

```
Layer 0 (基础层，无依赖):
  Common → Auth

Layer 1 (核心业务，依赖 Layer 0):
  Patient → HealthData

Layer 2 (上层业务，依赖 Layer 1):
  Followup → Therapy → Notification

开发顺序: Common → Auth → Patient → HealthData → Followup → Therapy → Notification
```

### 每个模块的标准开发步骤

```
① 数据模型确认 → ② DTO 定义 → ③ Service 实现 → ④ Routes 实现
→ ⑤ 单元测试 → ⑥ 前端页面 → ⑦ 前端联调 → ⑧ 集成测试
```

### 实际开发时间记录（ChronicCare 项目）

| 模块 | 后端开发 | 前端开发 | 联调测试 | 总计 |
|------|---------|---------|---------|------|
| Auth | 30min | 20min | 10min | 60min |
| HealthData | 40min | 30min | 15min | 85min |
| Followup | 50min | 40min | 20min | 90min |
| Therapy | 60min | 40min | 20min | 120min |
| Notification | 40min | 30min | 15min | 85min |
| **总计** | **3h40min** | **2h40min** | **1h20min** | **7h40min** |

> 💡 **经验**: AI 辅助下，单个模块从 0 到 1 平均约 1.5 小时，比传统开发快 3-5 倍

---

## 5.2 模块一：Auth（认证授权）

### 需求摘要

- 微信小程序 `wx.login()` 获取 code，换取 OpenID
- 首次登录自动创建患者档案
- JWT Token 签发与刷新
- 角色判断（患者/医护/管理员）

### AI 生成 Prompt（已验证可用）

```markdown
# 角色
你是 Flask 后端开发专家，正在开发慢病管家项目的认证模块。

# 技术栈
Flask 3.x + SQLAlchemy 2.x + PostgreSQL + JWT

# 需求
实现微信小程序登录认证模块，包含：

1. **微信登录接口** POST /api/v1/auth/wx-login
   - 接收小程序端的 code
   - 调用微信 code2Session 接口获取 openId 和 session_key
   - 如果是新用户，自动创建 Patient 记录
   - 签发 JWT（access_token 有效期 2h，refresh_token 有效期 7d）

2. **Token 刷新接口** POST /api/v1/auth/refresh
   - 验证 refresh_token
   - 签发新的 access_token

3. **JWT 验证装饰器**
   - 从 Header 解析 Token
   - 验证有效性
   - 将用户信息注入 Flask g 对象

# 约束
- 微信 AppID 和 Secret 从环境变量读取
- session_key 绝不能返回给前端
- JWT payload 只包含 userId 和 role，不包含敏感信息
- 所有登录行为记录审计日志

# 输出
按文件输出完整代码：
- auth_service.py (服务层)
- auth_routes.py (路由层)
- jwt_decorator.py (认证装饰器)
```

### AI 产出物（实际代码）

**文件**: `server/src/modules/auth/auth_service.py`

```python
"""
认证服务 - 微信登录
"""
import requests
import jwt
from datetime import datetime, timedelta
from flask import current_app, g
from functools import wraps

class AuthService:
    def __init__(self, db_session):
        self.db = db_session
    
    def wx_login(self, code: str):
        """微信小程序登录"""
        # 1. 调用微信接口
        wx_response = self._wx_code2session(code)
        
        if not wx_response.get('openid'):
            raise Exception(f"微信登录失败：{wx_response.get('errmsg', 'unknown error')}")
        
        open_id = wx_response['openid']
        
        # 2. 查找或创建患者
        from server.src.models.patient import Patient
        patient = self.db.session.query(Patient).filter_by(open_id=open_id).first()
        
        if not patient:
            patient = Patient(open_id=open_id, status='active')
            self.db.session.add(patient)
            self.db.session.commit()
        
        # 3. 签发 JWT
        access_token = self._generate_jwt(patient)
        refresh_token = self._generate_jwt(patient, refresh=True)
        
        return {
            'access_token': access_token,
            'refresh_token': refresh_token,
            'patient': patient.to_dict(),
            'is_new_user': patient.name is None
        }
    
    def _wx_code2session(self, code: str):
        """调用微信 code2Session 接口"""
        params = {
            'appid': current_app.config['WX_APP_ID'],
            'secret': current_app.config['WX_APP_SECRET'],
            'js_code': code,
            'grant_type': 'authorization_code'
        }
        response = requests.get(
            'https://api.weixin.qq.com/sns/jscode2session',
            params=params,
            timeout=10
        )
        return response.json()
    
    def _generate_jwt(self, patient, refresh=False):
        """生成 JWT Token"""
        now = datetime.utcnow()
        expires = now + (
            current_app.config['JWT_REFRESH_TOKEN_EXPIRES'] if refresh 
            else current_app.config['JWT_ACCESS_TOKEN_EXPIRES']
        )
        
        payload = {
            'sub': patient.id,
            'role': 'patient',
            'iat': now,
            'exp': expires
        }
        
        return jwt.encode(
            payload,
            current_app.config['JWT_SECRET_KEY'],
            algorithm='HS256'
        )

def jwt_required(f):
    """JWT 验证装饰器"""
    @wraps(f)
    def decorated_function(*args, **kwargs):
        from flask import request, jsonify
        from server.src.models.patient import Patient
        
        auth_header = request.headers.get('Authorization')
        if not auth_header or not auth_header.startswith('Bearer '):
            return jsonify({'error': 'Missing authorization header'}), 401
        
        token = auth_header.split(' ')[1]
        
        try:
            payload = jwt.decode(
                token,
                current_app.config['JWT_SECRET_KEY'],
                algorithms=['HS256']
            )
            
            patient = Patient.query.get(payload['sub'])
            if not patient or patient.status != 'active':
                return jsonify({'error': 'User not found'}), 404
            
            g.current_user = {
                'id': patient.id,
                'role': payload['role'],
            }
            
        except jwt.ExpiredSignatureError:
            return jsonify({'error': 'Token expired'}), 401
        except jwt.InvalidTokenError:
            return jsonify({'error': 'Invalid token'}), 401
        
        return f(*args, **kwargs)
    
    return decorated_function
```

### 前端代码（小程序登录）

**文件**: `client/src/pages/login/login.tsx`

```typescript
import { View, Button, Text } from '@tarojs/components'
import Taro, { useNavigateTo } from '@tarojs/taro'
import { useState } from 'react'

export default function Login() {
  const [loading, setLoading] = useState(false)
  const navigateTo = useNavigateTo()

  const handleLogin = async () => {
    setLoading(true)
    
    try {
      // 1. 微信登录获取 code
      const { code } = await Taro.login({ timeout: 10000 })

      // 2. 调用后端接口
      const res = await Taro.request({
        url: `${process.env.API_BASE_URL || 'http://localhost:5000'}/api/v1/auth/wx-login`,
        method: 'POST',
        data: { code },
      })

      const { access_token, refresh_token, patient, is_new_user } = res.data

      // 3. 存储 token
      Taro.setStorageSync('access_token', access_token)
      Taro.setStorageSync('refresh_token', refresh_token)

      // 4. 跳转
      if (is_new_user) {
        navigateTo({ url: '/pages/profile/profile' })
      } else {
        navigateTo({ url: '/pages/index/index' })
      }

      Taro.showToast({ title: '登录成功', icon: 'success' })
    } catch (error) {
      Taro.showToast({ title: '登录失败', icon: 'none' })
    } finally {
      setLoading(false)
    }
  }

  return (
    <View className='login-container'>
      <Text className='logo-text'>慢病管家</Text>
      <Button 
        className='login-btn' 
        type='primary' 
        onClick={handleLogin}
        disabled={loading}
      >
        {loading ? '登录中...' : '微信一键登录'}
      </Button>
    </View>
  )
}
```

### 审查要点（Auth 模块）

```
☐ session_key 是否仅存在后端，未暴露给前端
☐ JWT Secret 是否从环境变量读取
☐ Token 过期时间是否合理（access: 2h, refresh: 7d）
☐ 登录失败是否有频率限制（防暴力破解）
☐ 审计日志是否记录了登录成功/失败
☐ 微信 API 调用是否有超时和重试
☐ 敏感操作（如修改密码）是否需要二次验证
```

### 遇到的问题与解决

| 问题 | 原因 | 解决方案 |
|------|------|---------|
| 微信 code 一次性 | code 只能用一次 | 前端重试时重新调用 wx.login() |
| Token 过期处理 | 2h 后 access_token 失效 | 前端拦截 401，用 refresh_token 刷新 |
| 多端登录 | 同一用户多设备登录 | 当前允许，后续可加设备绑定 |

---

## 5.3 模块二：HealthData（健康数据）

### 需求摘要

- 血糖、血压、体重数据录入
- 数据异常检测（自动触发随访）
- 历史数据查询（支持时间范围筛选）
- 数据趋势图表（前端展示）

### AI 生成 Prompt

```markdown
# 角色
你是 Flask 后端开发专家 + 医疗数据工程师。

# 技术栈
Flask + SQLAlchemy + PostgreSQL + JSON 字段

# 需求
实现健康数据管理模块，包含：

1. **数据模型设计**
   - 支持多种数据类型（血糖、血压、体重）
   - 使用 JSON 字段存储灵活的值结构
   - 包含审计字段（created_at, updated_at）

2. **数据录入接口** POST /api/v1/health-records
   - 参数校验（数值范围、必填字段）
   - 异常检测（血糖>7.0 空腹触发预警）
   - 事件发布（健康数据创建事件）

3. **数据查询接口** GET /api/v1/health-records
   - 支持按类型筛选
   - 支持时间范围查询
   - 分页返回

# 数据格式示例
血糖：{"value": 6.1, "period": "fasting"}
血压：{"systolic": 120, "diastolic": 80}
体重：{"value": 65.5}
```

### 关键代码片段

**异常检测逻辑**:
```python
def check_abnormal(record_type: str, value: dict) -> bool:
    """检查数据是否异常"""
    if record_type == 'BLOOD_GLUCOSE':
        val = value.get('value', 0)
        period = value.get('period', '')
        # 空腹 > 7.0 或 随机 > 11.1
        if period == 'fasting' and val > 7.0:
            return True
        if period in ['random', 'after_meal'] and val > 11.1:
            return True
    elif record_type == 'BLOOD_PRESSURE':
        systolic = value.get('systolic', 0)
        diastolic = value.get('diastolic', 0)
        if systolic > 140 or diastolic > 90:
            return True
    return False
```

**事件触发**:
```python
# 创建记录后触发事件
event_bus.emit('health-data.created', {
    'patient_id': patient_id,
    'type': record_type,
    'value': record_value,
    'record_id': record.id
})

# 监听器自动处理异常
@event_bus.on('health-data.created')
def on_health_data_created(data):
    if check_abnormal(data['type'], data['value']):
        # 触发紧急随访
        followup_service.trigger_urgent_followup(data['patient_id'])
        # 发送预警通知
        notification_service.send_abnormal_alert(...)
```

### 审查要点

```
☐ 数值范围校验（防止异常值）
☐ 单位统一（mmol/L vs mg/dL）
☐ 时间戳处理（时区问题）
☐ 数据隐私（患者只能看自己的数据）
☐ 异常检测阈值可配置
☐ 事件总线解耦（避免循环依赖）
```

---

## 5.4 模块三：Followup（随访管理）

### 需求摘要

- 随访计划创建（常规/紧急/专项）
- 随访问卷执行
- 风险评估（低/中/高）
- 随访模板管理

### AI 生成 Prompt

```markdown
# 角色
你是医疗系统架构师 + Flask 开发专家。

# 需求
设计随访管理系统，包含：

1. **数据模型**
   - FollowupPlan（随访计划）
   - FollowupRecord（随访执行记录）
   - FollowupTemplate（随访模板）

2. **核心功能**
   - 计划创建（支持自动触发）
   - 问卷执行（JSON 存储答案）
   - 风险评估（触发通知）
   - 模板管理（按慢病类型分类）

3. **业务规则**
   - 紧急随访 24 小时内执行
   - 高风险患者自动升级随访频率
   - 随访记录可追溯修改
```

### 关键设计决策

**问卷存储方案**:
```python
# 使用 JSON 字段存储灵活问卷答案
questionnaire_data = db.Column(db.JSON, nullable=True)
# 示例：
# {
#   "q1": {"question": "最近是否有头晕？", "answer": "偶尔"},
#   "q2": {"question": "用药情况？", "answer": "按时服药"}
# }
```

**自动触发随访**:
```python
# 健康数据异常 → 触发紧急随访
def trigger_urgent_followup(patient_id: str, reason: str):
    plan = FollowupPlan(
        patient_id=patient_id,
        plan_type='urgent',
        scheduled_date=date.today() + timedelta(days=1),
        status='pending'
    )
    db.session.add(plan)
    db.session.commit()
```

---

## 5.5 模块四：Therapy（数字疗法）

### 需求摘要

- 疗法计划管理（健康教育/运动/饮食）
- 每日任务生成与追踪
- 任务完成打卡
- 疗法素材库

### 核心功能实现

**任务自动生成**:
```python
def _generate_tasks(self, plan, days: int = 7):
    """为疗法计划生成未来 7 天的任务"""
    from server.src.models.therapy import TherapyTask
    
    start_date = date.today()
    for i in range(days):
        task_date = start_date + timedelta(days=i)
        
        # 检查是否已存在
        exists = TherapyTask.query.filter_by(
            plan_id=plan.id,
            task_date=task_date
        ).first()
        
        if not exists:
            task = TherapyTask(
                plan_id=plan.id,
                patient_id=plan.patient_id,
                task_date=task_date,
                task_title=plan.title,
                status='pending'
            )
            db.session.add(task)
    
    db.session.commit()
```

---

## 5.6 模块五：Notification（通知提醒）

### 需求摘要

- 多渠道通知（小程序订阅消息/短信）
- 通知模板管理
- 未读/已读状态管理
- 定时推送

### 通知类型

| 类型 | 触发条件 | 渠道 |
|------|---------|------|
| followup_reminder | 随访计划创建 | 小程序 |
| therapy_reminder | 任务待完成 | 小程序 |
| abnormal_alert | 健康数据异常 | 小程序 + 短信 |
| system | 系统公告 | 小程序 |

---

## 5.7 事件总线集成

### 事件清单

| 事件名 | 触发时机 | 监听器 |
|--------|---------|--------|
| health-data.created | 健康数据创建 | 异常检测 → 随访 + 通知 |
| followup.risk_detected | 随访风险评估 | 通知发送 |
| therapy-task.completed | 任务完成 | 鼓励通知 |

### 实现代码

```python
# server/src/common/event_bus.py
class EventEmitter:
    def __init__(self):
        self._events: Dict[str, List[Callable]] = {}
    
    def on(self, event: str, handler: Callable):
        """注册监听器"""
        if event not in self._events:
            self._events[event] = []
        self._events[event].append(handler)
    
    def emit(self, event: str, data: dict = None):
        """触发事件"""
        for handler in self._events.get(event, []):
            try:
                handler(data)
            except Exception as e:
                logger.error(f"事件处理失败：{event}, error={e}")
```

---

## 5.8 开发经验总结

### AI 辅助开发效率对比

| 任务类型 | 传统开发 | AI 辅助 | 提升倍数 |
|---------|---------|--------|---------|
| CRUD 接口 | 2h/模块 | 30min/模块 | 4x |
| 数据模型 | 1h/模块 | 15min/模块 | 4x |
| 前端页面 | 3h/页面 | 45min/页面 | 4x |
| 单元测试 | 2h/模块 | 30min/模块 | 4x |
| **总计** | **8h/模块** | **2h/模块** | **4x** |

### Prompt 工程最佳实践

1. **明确技术栈** - 在 Prompt 开头声明 Flask/SQLAlchemy 版本
2. **提供上下文** - 粘贴相关模型和已有代码
3. **指定输出格式** - 按文件分别输出，标注路径
4. **添加约束条件** - 安全、性能、可维护性要求
5. **迭代优化** - 第一轮生成后，根据审查结果修正

### 常见陷阱与规避

| 陷阱 | 表现 | 规避方法 |
|------|------|---------|
| 技术栈混用 | NestJS 代码混入 Flask 项目 | Prompt 明确声明技术栈，审查时检查 |
| 过度设计 | AI 生成复杂架构 | 约束"保持简单，MVP 优先" |
| 安全遗漏 | 缺少参数校验、SQL 注入 | 审查清单包含安全检查项 |
| 循环依赖 | 模块间互相导入 | 定义清晰依赖方向，使用事件解耦 |

---

## 5.9 下一步优化

- [ ] 补充单元测试覆盖率（目标 80%）
- [ ] 集成 Swagger 文档自动生成
- [ ] 添加 API 性能基准测试
- [ ] 完善错误处理（统一异常类）
- [ ] 添加性能优化（数据库索引、缓存）

---

**本章完**

下一章：[06 - 测试与质量保障](./06-testing-qa.md)

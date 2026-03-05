# 03 - 合规与安全设计

## 3.1 医疗数据合规框架

### 适用法规清单

| 法规/标准 | 适用范围 | 对本项目的影响 |
|-----------|----------|---------------|
| 《个人信息保护法》(PIPL) | 所有个人信息处理 | 需明确告知、取得同意、最小必要原则 |
| 《数据安全法》 | 数据全生命周期 | 数据分类分级、安全评估 |
| 《网络安全法》 | 网络运营者 | 等保定级、安全审计 |
| 《医疗卫生机构网络安全管理办法》 | 医疗信息系统 | 等保三级要求 |
| 《健康医疗大数据标准》 | 健康数据 | 数据标准化、互操作性 |
| GB/T 35273 个人信息安全规范 | 个人信息 | 隐私设计、影响评估 |
| GB/T 39725 健康医疗数据安全指南 | 健康医疗数据 | 数据分类、安全措施 |

### 等保三级核心要求映射

```
等保三级要求                    系统实现方案
──────────────────────────────────────────────────────
身份鉴别                  →    微信OAuth + JWT + 双因素(医护端)
访问控制                  →    RBAC 角色权限模型
安全审计                  →    全链路操作审计日志
数据完整性                →    数据签名 + 校验和
数据保密性                →    传输加密(TLS) + 存储加密(AES-256)
数据备份恢复              →    自动备份 + 异地容灾
入侵防范                  →    WAF + 异常检测
恶意代码防范              →    输入校验 + CSP策略
安全管理中心              →    集中日志 + 安全监控面板
```

## 3.2 数据分类分级

### 分级标准

| 等级 | 定义 | 数据示例 | 安全措施 |
|------|------|----------|----------|
| **L4 极敏感** | 泄露将严重损害个人权益 | 身份证号、诊断报告原文 | AES-256加密存储、访问审计、脱敏展示 |
| **L3 高敏感** | 泄露将损害个人权益 | 姓名、手机号、血糖值、用药记录 | AES-256加密存储、访问审计 |
| **L2 中敏感** | 泄露可能影响个人 | 随访计划、健康教育记录 | 访问控制、传输加密 |
| **L1 低敏感** | 泄露影响有限 | 系统配置、通用健康知识 | 基本访问控制 |

### 字段级加密方案

```python
# server/src/common/crypto.py
# 使用 SQLAlchemy 事件监听器实现透明加密解密

from cryptography.fernet import Fernet
from sqlalchemy import event
from typing import Dict, List
import os

# 密钥管理：生产环境应从密钥管理系统获取
ENCRYPTION_KEY = os.getenv('ENCRYPTION_KEY')  # 32 字节 URL-safe base64 编码
cipher = Fernet(ENCRYPTION_KEY)

# 加密字段注册表
ENCRYPTED_FIELDS: Dict[str, List[str]] = {
    'Patient': ['name', 'phone', 'id_card'],
    'HealthRecord': [],  # value 字段使用应用层加密
}

def encrypt_value(value: str) -> str:
    """加密单个值"""
    if value is None:
        return None
    return cipher.encrypt(value.encode()).decode()

def decrypt_value(value: str) -> str:
    """解密单个值"""
    if value is None:
        return None
    return cipher.decrypt(value.encode()).decode()

# SQLAlchemy 事件监听器 — 写入时加密
@event.listens_for(Patient, 'before_insert')
@event.listens_for(Patient, 'before_update')
def encrypt_before_save(mapper, connection, target):
    """在写入数据库前加密敏感字段"""
    fields_to_encrypt = ENCRYPTED_FIELDS.get(target.__tablename__, [])
    for field in fields_to_encrypt:
        value = getattr(target, field, None)
        if value:
            setattr(target, field, encrypt_value(value))

# SQLAlchemy 事件监听器 — 读取时解密
@event.listens_for(Patient, 'load')
def decrypt_on_load(target, context):
    """在从数据库加载后解密敏感字段"""
    fields_to_decrypt = ENCRYPTED_FIELDS.get(target.__tablename__, [])
    for field in fields_to_decrypt:
        value = getattr(target, field, None)
        if value:
            setattr(target, field, decrypt_value(value))
```

## 3.3 权限模型设计（RBAC）

### 角色定义

```
角色                权限范围
──────────────────────────────────────────
patient (患者)      只能访问自己的数据，只读随访计划
doctor (医生)       管理自己负责的患者，创建随访计划和疗法处方
health_manager      管理分配的患者群组，执行随访
admin (管理员)      系统管理，数据导出，用户管理
```

### 权限矩阵

| 资源 | patient | doctor | health_manager | admin |
|------|---------|--------|----------------|-------|
| 自己的档案 | RU | — | — | CRUD |
| 自己的健康数据 | CRU | — | — | CRUD |
| 负责患者的档案 | — | RU | R | CRUD |
| 负责患者的健康数据 | — | R | R | CRUD |
| 随访计划 | R(自己的) | CRUD | RU | CRUD |
| 数字疗法处方 | R(自己的) | CRUD | R | CRUD |
| 健康教育内容 | R | R | R | CRUD |
| 系统配置 | — | — | — | CRUD |
| 数据导出 | — | — | — | R(脱敏) |

### 权限守卫实现

```python
# server/src/common/decorators/roles.py
# Flask 角色权限装饰器

from functools import wraps
from flask import request, jsonify, g
from typing import List, Callable
from enum import Enum

class Role(Enum):
    PATIENT = 'patient'
    DOCTOR = 'doctor'
    HEALTH_MANAGER = 'health_manager'
    ADMIN = 'admin'

def roles_required(*required_roles: Role):
    """
    角色权限装饰器
    用法：@roles_required(Role.DOCTOR, Role.ADMIN)
    """
    def decorator(f: Callable):
        @wraps(f)
        def decorated_function(*args, **kwargs):
            # 从 g 中获取当前用户（由 JWT 验证中间件注入）
            user = getattr(g, 'current_user', None)
            if not user:
                return jsonify({'error': 'Unauthorized'}), 401
            
            # 检查用户角色是否在允许的角色列表中
            if user.role not in [r.value for r in required_roles]:
                return jsonify({'error': 'Forbidden'}), 403
            
            return f(*args, **kwargs)
        return decorated_function
    return decorator

# 使用示例
# server/src/modules/followup/followup_routes.py

from flask import Blueprint, request, jsonify
from server.src.common.decorators.roles import roles_required, Role
from server.src.common.decorators.jwt import jwt_required

followup_bp = Blueprint('followup', __name__)

@followup_bp.route('/followup-plans', methods=['POST'])
@jwt_required
@roles_required(Role.DOCTOR, Role.ADMIN)  # 只有医生和管理员可创建
def create_followup():
    from server.src.common.utils import get_current_user
    user = get_current_user()
    dto = request.get_json()
    return jsonify(followup_service.create(dto, user))

@followup_bp.route('/followup-plans', methods=['GET'])
@jwt_required
@roles_required(Role.PATIENT, Role.DOCTOR, Role.HEALTH_MANAGER)
def list_followups():
    from server.src.common.utils import get_current_user
    user = get_current_user()
    query = request.args
    # Service 层根据角色自动过滤数据范围
    return jsonify(followup_service.list(user, query))
}
```

## 3.4 安全审计设计

### 审计日志模型

```prisma
model AuditLog {
  id          String   @id @default(cuid())
  userId      String
  userRole    String
  action      String   // CREATE | READ | UPDATE | DELETE | EXPORT | LOGIN
  resource    String   // 资源类型: Patient | HealthRecord | ...
  resourceId  String?  // 资源 ID
  detail      Json?    // 操作详情（脱敏后）
  ip          String
  userAgent   String
  result      String   // SUCCESS | FAILED | DENIED
  createdAt   DateTime @default(now())

  @@index([userId, createdAt])
  @@index([resource, action, createdAt])
  @@index([createdAt])  // 用于定期归档
}
```

### 审计拦截器

```python
# server/src/common/middleware/audit_middleware.py
# Flask 审计日志中间件

from flask import request, g, jsonify
from datetime import datetime
from functools import wraps
import logging

logger = logging.getLogger(__name__)

def audit_log(action_map=None):
    """
    审计日志装饰器
    用法：@audit_log(action_map={'GET': 'READ', 'POST': 'CREATE', ...})
    """
    def decorator(f):
        @wraps(f)
        def decorated_function(*args, **kwargs):
            start_time = datetime.now()
            
            # 执行请求
            try:
                response = f(*args, **kwargs)
                
                # 记录成功日志
                log_audit(
                    user_id=getattr(g, 'current_user', {}).get('id'),
                    user_role=getattr(g, 'current_user', {}).get('role'),
                    action=action_map.get(request.method, request.method) if action_map else request.method,
                    resource=request.endpoint,
                    resource_id=request.view_args.get('id'),
                    ip=request.remote_addr,
                    user_agent=request.headers.get('User-Agent'),
                    result='SUCCESS',
                    duration=(datetime.now() - start_time).total_seconds()
                )
                
                return response
                
            except Exception as e:
                # 记录失败日志
                log_audit(
                    user_id=getattr(g, 'current_user', {}).get('id'),
                    user_role=getattr(g, 'current_user', {}).get('role'),
                    action=action_map.get(request.method, request.method) if action_map else request.method,
                    resource=request.endpoint,
                    result='FAILED' if getattr(e, 'status_code', 500) != 403 else 'DENIED',
                    detail=str(e)
                )
                raise
        
        return decorated_function
    return decorator

def log_audit(**kwargs):
    """写入审计日志到数据库"""
    from server.src.modules.audit.audit_service import AuditService
    audit_service = AuditService()
    audit_service.log(**kwargs)
```

## 3.5 数据脱敏策略

### 脱敏规则

| 字段类型 | 原始值 | 脱敏后 | 适用场景 |
|----------|--------|--------|----------|
| 姓名 | 张三丰 | 张**  | 列表展示、日志 |
| 手机号 | 13812345678 | 138****5678 | 列表展示 |
| 身份证 | 110101199001011234 | 1101\*\*\*\*\*\*\*\*1234 | 列表展示 |
| 诊断 | 2型糖尿病 | 不脱敏 | 医护端正常展示 |
| 血糖值 | 7.8 mmol/L | 不脱敏 | 医护端正常展示 |

### 脱敏装饰器

```typescript
// common/decorators/desensitize.decorator.ts
export function Desensitize(type: DesensitizeType) {
  return (target: any, propertyKey: string) => {
    Reflect.defineMetadata('desensitize', type, target, propertyKey);
  };
}

// DTO 中使用
export class PatientListItemDto {
  id: string;

  @Desensitize(DesensitizeType.NAME)
  name: string;

  @Desensitize(DesensitizeType.PHONE)
  phone: string;

  riskLevel: RiskLevel;
}
```

## 3.6 隐私合规设计

### 用户知情同意流程

```
首次进入小程序
    │
    ▼
┌──────────────┐    拒绝    ┌──────────────┐
│ 隐私政策弹窗  │──────────→│ 仅可浏览公开  │
│ (必须主动勾选) │           │ 健康教育内容  │
└──────┬───────┘           └──────────────┘
       │ 同意
       ▼
┌──────────────┐
│ 微信授权登录  │
│ (获取OpenID)  │
└──────┬───────┘
       │
       ▼
┌──────────────┐    跳过    ┌──────────────┐
│ 补充个人信息  │──────────→│ 基础功能可用  │
│ (分步骤收集)  │           │ 高级功能受限  │
└──────┬───────┘           └──────────────┘
       │ 完成
       ▼
┌──────────────┐
│ 全功能可用    │
└──────────────┘
```

### 数据最小必要原则

```
功能                    必要数据              非必要数据（需单独授权）
────────────────────────────────────────────────────────
血糖记录              血糖值、时间            位置信息
用药提醒              药品名、时间            —
随访管理              基本信息、病史          身份证号
蓝牙设备同步          设备数据               设备型号（用于兼容性）
健康报告              健康数据汇总            分享给第三方
```

## 3.7 阶段交付物检查清单

- [ ] 数据分类分级清单（所有字段标注敏感等级）
- [ ] 等保三级要求映射表（要求 → 技术实现方案）
- [ ] RBAC 权限矩阵（角色 × 资源 × 操作）
- [ ] 字段级加密方案（算法、密钥管理、SQLAlchemy 事件监听器）
- [ ] 审计日志方案（数据模型、拦截器、归档策略）
- [ ] 数据脱敏规则表
- [ ] 隐私政策文本（需法务审核）
- [ ] 用户知情同意流程设计
- [ ] 数据最小必要原则对照表
- [ ] 安全威胁建模（STRIDE 分析）
- [ ] 渗透测试计划（上线前执行）

---

**上一步**: [02 - 架构设计](./02-architecture-design.md)
**下一步**: [04 - AI开发工作流](./04-ai-dev-workflow.md)

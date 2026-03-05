# 07 - 部署与运维

## 7.1 部署架构

### 推荐方案：云原生容器化部署

```
                        ┌─────────────────────┐
                        │    微信小程序 CDN     │
                        │  (小程序包托管)       │
                        └──────────┬──────────┘
                                   │
                        ┌──────────▼──────────┐
                        │    Nginx / 云 LB     │
                        │  (SSL终止 + 反向代理) │
                        └──────────┬──────────┘
                                   │
              ┌────────────────────┼────────────────────┐
              │                    │                    │
    ┌─────────▼────────┐ ┌────────▼────────┐ ┌────────▼────────┐
    │  Flask 实例 #1   │ │ Flask 实例 #2  │ │ Flask 实例 #3  │
    │  (Docker)         │ │ (Docker)        │ │ (Docker)        │
    └─────────┬────────┘ └────────┬────────┘ └────────┬────────┘
              │                    │                    │
              └────────────────────┼────────────────────┘
                                   │
              ┌────────────────────┼────────────────────┐
              │                    │                    │
    ┌─────────▼────────┐ ┌────────▼────────┐ ┌────────▼────────┐
    │   PostgreSQL      │ │     Redis       │ │   MinIO/OSS     │
    │   (主从 + 备份)   │ │   (哨兵模式)    │ │   (文件存储)    │
    └──────────────────┘ └─────────────────┘ └─────────────────┘
```

### 环境规划

| 环境 | 用途 | 配置 | 数据 |
|------|------|------|------|
| local | 本地开发 | Docker Compose | 模拟数据 |
| dev | 联调测试 | 单实例 | 测试数据 |
| staging | 预发布验证 | 与生产同构 | 脱敏生产数据 |
| production | 生产环境 | 多实例 + 高可用 | 真实数据 |

## 7.2 Docker 配置

### Prompt 模板：Docker 配置生成

```markdown
# 角色
你是 DevOps 工程师，擅长容器化部署。

# 上下文
慢病管家项目结构：
- client/ — Taro 小程序前端
- server/ — Flask 后端
- 数据库: PostgreSQL 15 + Redis 7

# 任务
生成以下 Docker 配置文件：

1. **server/Dockerfile** — 多阶段构建，生产优化
   - 构建阶段: python:3.11-slim（或 3.12-slim），安装依赖
   - 运行阶段: python:3.11-slim，仅复制必要文件
   - 非 root 用户运行
   - 健康检查端点（/health）

2. **deploy/docker-compose.yml** — 本地开发环境（与示例项目保持一致）
   - PostgreSQL 16（持久化卷）
   - MinIO（对象存储）
   - Redis（可选）

3. **（可选）docker-compose.prod.yml** — 生产环境
   - Flask 多实例（建议由 K8s / 容器平台编排，而非 compose 扩副本）
   - PostgreSQL 主从/云托管
   - Redis 哨兵/云托管
   - 日志收集/监控

# 约束
- 镜像体积尽量小（alpine 基础镜像）
- 敏感配置通过环境变量注入，不打入镜像
- 数据库密码使用 Docker Secrets
```

### AI 产出：deploy/docker-compose.yml（开发环境，示例项目同款）

> 说明：指南以 `chronic-care/deploy/docker-compose.yml` 为准；后端/前端本地开发通常直接在宿主机跑（便于热更新与调试），基础设施用 compose 起。

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_DB: chronic_care
      POSTGRES_USER: app
      POSTGRES_PASSWORD: app
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d chronic_care"]
      interval: 5s
      timeout: 5s
      retries: 20

  minio:
    image: minio/minio:latest
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: minio
      MINIO_ROOT_PASSWORD: minio123456
    ports:
      - "9000:9000"
      - "9001:9001"
    volumes:
      - minio_data:/data
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"]
      interval: 10s
      timeout: 5s
      retries: 10

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 5s
      retries: 10

volumes:
  pgdata:
  minio_data:
```

## 7.3 CI/CD 流水线

### GitHub Actions 配置

```markdown
# Prompt 模板：CI/CD 配置生成

# 角色
你是 DevOps 工程师。

# 任务
为慢病管家项目生成 GitHub Actions CI/CD 配置，包含：

1. **CI 流水线**（PR 触发）
   - 代码规范检查（ESLint + Prettier + TypeScript）
   - 单元测试 + 覆盖率报告
   - 集成测试（使用 service container 启动 PostgreSQL + Redis）
   - 安全扫描（npm audit + Semgrep）
   - 构建验证

2. **CD 流水线**（合入 main 触发）
   - 构建 Docker 镜像
   - 推送到容器镜像仓库
   - 部署到 staging 环境
   - 运行冒烟测试
   - 人工确认后部署到 production

3. **小程序发布流水线**（tag 触发）
   - Taro 构建
   - 使用 miniprogram-ci 上传到微信后台
   - 通知团队审核发布
```

### 流水线示意

```
PR 提交
  │
  ▼
┌──────────────────────────────────────────┐
│              CI Pipeline                  │
│  lint → typecheck → test → security scan │
└──────────────────┬───────────────────────┘
                   │ 全部通过
                   ▼
              人工 Review
                   │ Approve
                   ▼
            合入 develop/main
                   │
                   ▼
┌──────────────────────────────────────────┐
│              CD Pipeline                  │
│  build image → push → deploy staging     │
│       → smoke test → manual gate         │
│       → deploy production                │
└──────────────────────────────────────────┘
```

## 7.4 微信小程序发布流程

```
开发完成
  │
  ▼
Taro 构建 (npm run build:weapp)
  │
  ▼
微信开发者工具预览/真机调试
  │
  ▼
miniprogram-ci 上传体验版
  │
  ▼
产品经理 + 测试验收
  │
  ▼
提交微信审核
  │
  ▼
审核通过 → 灰度发布（10% → 50% → 100%）
```

### 自动化上传脚本

```typescript
// scripts/upload-miniprogram.ts
import * as ci from 'miniprogram-ci';

const project = new ci.Project({
  appid: process.env.WX_APP_ID,
  type: 'miniProgram',
  projectPath: './client/dist',
  privateKeyPath: './keys/wx-upload-key.key', // 不入 Git
  ignores: ['node_modules/**/*'],
});

async function upload() {
  const version = process.env.VERSION || '1.0.0';
  const desc = process.env.RELEASE_DESC || `v${version} 自动构建`;

  await ci.upload({
    project,
    version,
    desc,
    setting: {
      es6: true,
      minify: true,
      autoPrefixWXSS: true,
    },
  });
  console.log(`上传成功: v${version}`);
}

upload().catch(console.error);
```

## 7.5 监控与告警

### 监控体系

```
层级            监控内容                    工具推荐
──────────────────────────────────────────────────────
基础设施        CPU/内存/磁盘/网络          Prometheus + Grafana
应用层          API 响应时间/错误率/QPS     Prometheus + 自定义指标
业务层          日活/功能使用率/转化率       自建埋点 + 数据看板
安全层          异常登录/越权访问/数据导出   审计日志分析
小程序端        页面加载/JS错误/API耗时     微信小程序性能监控
```

### 关键告警规则

| 指标 | 阈值 | 告警级别 | 通知方式 |
|------|------|----------|----------|
| API 错误率 | >5% (5min) | P1 紧急 | 电话 + 企微 |
| API P99 延迟 | >3s (5min) | P2 重要 | 企微 |
| 数据库连接池 | >80% | P2 重要 | 企微 |
| 磁盘使用率 | >85% | P3 一般 | 邮件 |
| 异常登录尝试 | >10次/min | P1 紧急 | 电话 + 企微 |
| 敏感数据批量导出 | 任意触发 | P1 紧急 | 电话 + 企微 |

### 健康检查端点

```python
# server/src/health/health_routes.py
# Flask 健康检查端点

from flask import Blueprint, jsonify
from server.src.extensions import db
from server.src.common.redis_client import redis_client
import psutil
import os

health_bp = Blueprint('health', __name__)

@health_bp.route('/health', methods=['GET'])
def health_check():
    """
    健康检查端点
    返回：{ status: 'ok' | 'degraded' | 'error', checks: {...} }
    """
    checks = {}
    overall_status = 'ok'
    
    # 数据库检查
    try:
        db.session.execute('SELECT 1')
        checks['database'] = {'status': 'ok'}
    except Exception as e:
        checks['database'] = {'status': 'error', 'error': str(e)}
        overall_status = 'error'
    
    # Redis 检查
    try:
        redis_client.ping()
        checks['redis'] = {'status': 'ok'}
    except Exception as e:
        checks['redis'] = {'status': 'error', 'error': str(e)}
        overall_status = 'error'
    
    # 磁盘空间检查
    try:
        disk = psutil.disk_usage('/')
        usage_percent = disk.percent / 100
        if usage_percent > 0.95:
            checks['disk'] = {'status': 'error', 'usage': f'{disk.percent:.1f}%'}
            overall_status = 'error'
        elif usage_percent > 0.85:
            checks['disk'] = {'status': 'warning', 'usage': f'{disk.percent:.1f}%'}
            if overall_status == 'ok':
                overall_status = 'degraded'
        else:
            checks['disk'] = {'status': 'ok', 'usage': f'{disk.percent:.1f}%'}
    except Exception as e:
        checks['disk'] = {'status': 'error', 'error': str(e)}
        overall_status = 'error'
    
    return jsonify({
        'status': overall_status,
        'checks': checks
    }), 200 if overall_status == 'ok' else 503
  }
}
```

## 7.6 数据备份与灾备

### 备份策略

```
备份类型        频率          保留周期      存储位置
──────────────────────────────────────────────────
全量备份        每日 02:00    30天          异地对象存储
增量备份        每小时        7天           同机房对象存储
WAL 归档        实时          7天           异地对象存储
Redis RDB       每6小时       3天           同机房
```

### 恢复演练

```
每季度执行一次灾备恢复演练:
1. 从备份恢复数据库到独立环境
2. 验证数据完整性（行数对比、校验和）
3. 验证应用可正常启动和访问
4. 记录 RTO（恢复时间目标）和 RPO（恢复点目标）
5. 归档演练报告
```

## 7.7 阶段交付物检查清单

- [ ] Dockerfile（多阶段构建、非 root 用户）
- [ ] docker-compose.yml（开发环境）
- [ ] CI 流水线配置（lint + test + security scan）
- [ ] CD 流水线配置（build + deploy + smoke test）
- [ ] 小程序自动化上传脚本
- [ ] 监控面板配置（Grafana Dashboard）
- [ ] 告警规则配置
- [ ] 健康检查端点
- [ ] 数据备份脚本和恢复文档
- [ ] 灰度发布方案
- [ ] 运维手册（常见问题排查 Runbook）

---

**上一步**: [06 - 测试与质量保障](./06-testing-qa.md)
**下一步**: [08 - 迭代与持续改进](./08-iteration.md)

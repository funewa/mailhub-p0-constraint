# MailHub P0 实现规范 — 交给 HY3 直接写代码

> 版本：v1.0 ｜ 2026-07-29
> 本文件是唯一权威实现依据，覆盖 P0 阶段全部代码
> 读完本文件即可直接开始编码，不需要再问任何设计问题

---

## 一、技术栈锁定（不可更改）

| 层 | 技术 | 版本 | 原因 |
|---|---|---|---|
| 语言 | Python | 3.11+ | SOAP client 生态好（zeep） |
| Web 框架 | FastAPI | 0.110+ | 异步原生支持，自动 OpenAPI 文档 |
| SOAP Client | zeep | 4.2+ | 唯一成熟的 Python SOAP 库 |
| 数据库 | PostgreSQL | 16+ | JSONB、行级安全 |
| 队列 | Redis + BullMQ (via `arq`) | 7+ | 异步任务 + 缓存一体 |
| ORM | SQLAlchemy | 2.0+ (async) | 异步原生，Alembic 迁移 |
| 迁移 | Alembic | 1.13+ | 标准方案 |
| 校验 | Pydantic | 2.x | FastAPI 内置 |
| HTTP Client | httpx | 0.27+ | 异步 HTTP 调用 |
| 定时任务 | arq cron | - | Sync Engine / Health Check |
| 测试 | pytest + pytest-asyncio | 8+ | 标准 |
| 前端 | React 18 + Vite + Tailwind | - | 白标换肤基础 |
| 部署 | Docker Compose | - | 一键启动 |

**禁止引入**：Django（太重）、Flask（无异步）、Celery（用 arq 替代）、requests（用 httpx 替代）

---

## 二、项目结构（必须照此创建）

```
mailhub/
├── docker-compose.yml
├── Dockerfile
├── .env.example
├── alembic.ini
├── alembic/
│   └── versions/
├── app/
│   ├── __init__.py
│   ├── main.py                  # FastAPI 入口
│   ├── config.py                # 配置管理（pydantic-settings）
│   ├── database.py              # async engine + session
│   ├── deps.py                  # FastAPI 依赖注入
│   │
│   ├── models/                  # SQLAlchemy 模型（14 张表）
│   │   ├── __init__.py
│   │   ├── base.py              # Base + TimestampMixin
│   │   ├── tenant.py
│   │   ├── contact.py
│   │   ├── list.py
│   │   ├── list_contact.py
│   │   ├── campaign.py
│   │   ├── send_job.py
│   │   ├── api_key.py
│   │   ├── sender_domain.py
│   │   ├── template.py
│   │   ├── sync_log.py
│   │   ├── email_event.py
│   │   ├── webhook_config.py
│   │   ├── unsubscribe_log.py
│   │   └── usage_record.py
│   │
│   ├── schemas/                 # Pydantic 请求/响应模型
│   │   ├── __init__.py
│   │   ├── common.py            # ApiResponse, PaginatedResponse
│   │   ├── auth.py
│   │   ├── contact.py
│   │   ├── list.py
│   │   ├── campaign.py
│   │   └── unsubscribe.py
│   │
│   ├── api/                     # FastAPI 路由
│   │   ├── __init__.py
│   │   ├── deps.py              # get_current_tenant, rate_limit
│   │   ├── v1/
│   │   │   ├── __init__.py
│   │   │   ├── auth.py          # POST /v1/auth/token
│   │   │   ├── contacts.py      # CRUD + import + status
│   │   │   ├── lists.py         # CRUD + contacts
│   │   │   ├── campaigns.py     # create + publish + status
│   │   │   ├── emails.py        # transactional send
│   │   │   ├── unsubscribe.py   # webhook + confirm page
│   │   │   └── health.py        # GET /health
│   │
│   ├── services/                # 业务逻辑层
│   │   ├── __init__.py
│   │   ├── contact_service.py
│   │   ├── list_service.py
│   │   ├── campaign_service.py
│   │   ├── compliance_service.py # 退订闭环
│   │   └── report_service.py
│   │
│   ├── adapter/                 # SOAP 适配层（核心）
│   │   ├── __init__.py
│   │   ├── base.py              # SendDriver ABC
│   │   ├── spread_client.py     # zeep SOAP client
│   │   ├── spread_driver.py     # SpreadDriver implements SendDriver
│   │   ├── circuit_breaker.py   # Circuit Breaker
│   │   ├── error_mapper.py      # 错误统一映射
│   │   ├── retry.py             # 指数退避重试
│   │   └── idempotency.py       # 幂等键去重
│   │
│   ├── workers/                 # 异步 Worker
│   │   ├── __init__.py
│   │   ├── settings.py          # arq 配置
│   │   ├── send_worker.py       # 消费发送队列
│   │   ├── sync_engine.py       # 定时轮询赛邮（每 5 min）
│   │   └── health_check.py      # 定时探活（每 60s）
│   │
│   ├── core/                    # 横切组件
│   │   ├── __init__.py
│   │   ├── security.py          # JWT / API Key 验证
│   │   ├── rate_limit.py        # 限流
│   │   ├── cache.py             # Redis 缓存
│   │   ├── alerts.py            # 钉钉/Slack 告警
│   │   └── audit.py             # 审计日志
│   │
│   └── utils/
│       ├── __init__.py
│       └── validators.py        # email/phone 格式校验
│
├── tests/
│   ├── conftest.py
│   ├── test_adapter/
│   ├── test_api/
│   └── test_services/
│
└── frontend/                    # React 白标 UI（P0 简化版）
    ├── package.json
    ├── vite.config.ts
    ├── tailwind.config.js
    └── src/
        ├── App.tsx
        ├── pages/
        │   ├── Dashboard.tsx
        │   ├── Contacts.tsx
        │   ├── Lists.tsx
        │   └── SendEmail.tsx
        └── components/
```

---

## 三、关键约束（违反任何一条 = 打回重写）

### 约束 1：赛邮凭据安全
```
- SPREAD_LOGIN_EMAIL 和 SPREAD_PASSWORD 只从环境变量读取
- 绝不写入数据库、绝不写入日志、绝不返回给前端
- SOAP 调用必须走 https://service.rspread.com/Service.asmx
- 禁止 http://
```

### 约束 2：SOAP 调用必须过适配层
```
- 业务层（services/）禁止 import zeep
- 业务层禁止构造 SOAP envelope
- 所有 SOAP 调用必须经过 adapter/spread_driver.py
- 适配层返回统一的 ApiResponse，不暴露 SOAP 原始返回
```

### 约束 3：幂等必须实现
```
- 所有创建类接口（createCampaign, addSubscriber*）必须接受 idempotency_key
- 先查 send_job 表，key 存在 → 直接返回上次结果，不调 SOAP
- idempotency_key 由客户端生成（UUID），服务端只做去重
```

### 约束 4：退订必须双向同步
```
- 路径 A：用户在我方退订 → 本地标记 + 调 UpdateSubscriberStatus + 写 sync_log
- 路径 B：Sync Engine 每 5 分钟轮询赛邮 → 发现差异 → 反向写入本地 + 写 sync_log
- 两条路径缺一不可
```

### 约束 5：发送必须走队列
```
- createCampaign 不直接调 SOAP，先写 send_job 表 + 入 Redis 队列
- Send Worker 消费队列 → 调适配层 → 更新 send_job.status
- 用户立即收到 "queued" 响应，不等待 SOAP 返回
```

### 约束 6：错误必须统一映射
```
- 赛邮返回 "Success"/"Fail"/"Error"/1/-1 → 统一 ApiResponse
- code=0 成功，code!=0 失败
- 前端永远看不到 SOAP 原始返回
```

### 约束 7：多租户隔离
```
- 每张表都有 tenant_id 字段
- 所有查询必须过滤 tenant_id
- API Gateway 层强制注入 tenant_id
```

### 约束 8：文件上传安全
```
- 大小上限 10MB
- 类型白名单：.csv, .xls, .xlsx, .zip
- 上传后不直接处理，先存 S3，再异步 Worker 解析
```

### 约束 9：所有外部调用必须有超时
```
- SOAP 调用超时：30 秒
- HTTP 调用超时：15 秒
- 数据库查询超时：10 秒
```

### 约束 10：日志规范
```
- 使用 structlog，JSON 格式输出
- 敏感字段（email, password, api_key）必须脱敏
- 每次 SOAP 调用记录：方法名、耗时、成功/失败、错误码
```

---

## 四、数据库建表 SQL（直接执行）

```sql
-- alembic/versions/001_initial.sql

CREATE TABLE tenant (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    spread_account VARCHAR(255),         -- 赛邮子账号（可为空）
    plan VARCHAR(50) DEFAULT 'starter',
    status VARCHAR(20) DEFAULT 'active', -- active/suspended/deleted
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE contact (
    id SERIAL PRIMARY KEY,
    tenant_id INTEGER NOT NULL REFERENCES tenant(id),
    email VARCHAR(255) NOT NULL,
    name VARCHAR(255),
    company VARCHAR(255),
    phone VARCHAR(50),
    status VARCHAR(20) DEFAULT 'Active',
    -- Active/Unsubscribed/Unconfirmed/DoNotMail/Blacklist/Deleted
    source VARCHAR(50) DEFAULT 'manual', -- manual/import/api
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(tenant_id, email)
);
CREATE INDEX idx_contact_tenant ON contact(tenant_id);
CREATE INDEX idx_contact_status ON contact(tenant_id, status);
CREATE INDEX idx_contact_email ON contact(tenant_id, email);

CREATE TABLE list (
    id SERIAL PRIMARY KEY,
    tenant_id INTEGER NOT NULL REFERENCES tenant(id),
    name VARCHAR(255) NOT NULL,
    description TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(tenant_id, name)
);

CREATE TABLE list_contact (
    list_id INTEGER NOT NULL REFERENCES list(id) ON DELETE CASCADE,
    contact_id INTEGER NOT NULL REFERENCES contact(id) ON DELETE CASCADE,
    added_at TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (list_id, contact_id)
);

CREATE TABLE campaign (
    id SERIAL PRIMARY KEY,
    tenant_id INTEGER NOT NULL REFERENCES tenant(id),
    type VARCHAR(10) NOT NULL,           -- email/sms/mms
    name VARCHAR(255) NOT NULL,
    subject VARCHAR(500),
    body TEXT,
    list_id INTEGER REFERENCES list(id),
    status VARCHAR(20) DEFAULT 'draft',  -- draft/queued/sending/completed/failed
    sent_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE send_job (
    id SERIAL PRIMARY KEY,
    campaign_id INTEGER NOT NULL REFERENCES campaign(id),
    idempotency_key VARCHAR(64) UNIQUE,  -- 幂等键
    status VARCHAR(20) DEFAULT 'queued', -- queued/sending/completed/failed
    retry_count INTEGER DEFAULT 0,
    error TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_send_job_key ON send_job(idempotency_key);
CREATE INDEX idx_send_job_status ON send_job(status);

CREATE TABLE api_key (
    id SERIAL PRIMARY KEY,
    tenant_id INTEGER NOT NULL REFERENCES tenant(id),
    key_hash VARCHAR(128) NOT NULL,      -- SHA-256 hash，不存明文
    name VARCHAR(100) NOT NULL,
    last_used_at TIMESTAMPTZ,
    revoked_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE sender_domain (
    id SERIAL PRIMARY KEY,
    tenant_id INTEGER NOT NULL REFERENCES tenant(id),
    domain VARCHAR(255) NOT NULL,
    spf_ok BOOLEAN DEFAULT FALSE,
    dkim_ok BOOLEAN DEFAULT FALSE,
    dmarc_ok BOOLEAN DEFAULT FALSE,
    verified_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(tenant_id, domain)
);

CREATE TABLE template (
    id SERIAL PRIMARY KEY,
    tenant_id INTEGER NOT NULL REFERENCES tenant(id),
    name VARCHAR(255) NOT NULL,
    html_body TEXT,
    zip_path VARCHAR(500),
    thumbnail VARCHAR(500),
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE sync_log (
    id SERIAL PRIMARY KEY,
    direction VARCHAR(5) NOT NULL,       -- in/out
    entity VARCHAR(50) NOT NULL,         -- contact/campaign/list
    entity_id INTEGER NOT NULL,
    spread_status VARCHAR(50),
    local_status VARCHAR(50),
    synced_at TIMESTAMPTZ DEFAULT NOW(),
    error TEXT
);
CREATE INDEX idx_sync_log_entity ON sync_log(entity, entity_id);

CREATE TABLE email_event (
    id SERIAL PRIMARY KEY,
    campaign_id INTEGER NOT NULL,
    contact_id INTEGER NOT NULL,
    type VARCHAR(20) NOT NULL,           -- sent/delivered/open/click/bounce/unsub/complaint
    metadata JSONB,
    at TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_event_campaign ON email_event(campaign_id, type);
CREATE INDEX idx_event_contact ON email_event(contact_id, type);

CREATE TABLE webhook_config (
    id SERIAL PRIMARY KEY,
    tenant_id INTEGER NOT NULL REFERENCES tenant(id),
    event_type VARCHAR(50) NOT NULL,     -- open/click/bounce/unsub
    target_url VARCHAR(500) NOT NULL,
    secret VARCHAR(128),
    active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE unsubscribe_log (
    id SERIAL PRIMARY KEY,
    contact_id INTEGER NOT NULL,
    channel VARCHAR(20),                 -- link/api/spread_admin
    reason TEXT,
    synced_to_spread BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE usage_record (
    id SERIAL PRIMARY KEY,
    tenant_id INTEGER NOT NULL REFERENCES tenant(id),
    type VARCHAR(20) NOT NULL,           -- email/sms/mms
    qty INTEGER NOT NULL,
    cost DECIMAL(10,4),
    created_at TIMESTAMPTZ DEFAULT NOW()
);
CREATE INDEX idx_usage_tenant ON usage_record(tenant_id, type, created_at);
```

---

## 五、核心代码示例

### 5.1 配置管理 `app/config.py`

```python
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    # 数据库
    DATABASE_URL: str = "postgresql+asyncpg://mailhub:mailhub@localhost:5432/mailhub"
    
    # Redis
    REDIS_URL: str = "redis://localhost:6379/0"
    
    # 赛邮 SOAP（只从环境变量读取，禁止硬编码）
    SPREAD_WSDL_URL: str = "https://service.rspread.com/Service.asmx?WSDL"
    SPREAD_LOGIN_EMAIL: str = ""         # 必须设置
    SPREAD_PASSWORD: str = ""            # 必须设置（或 API Key）
    SPREAD_TIMEOUT: int = 30             # SOAP 调用超时秒数
    
    # JWT
    JWT_SECRET: str = "change-me-in-production"
    JWT_ALGORITHM: str = "HS256"
    JWT_EXPIRE_MINUTES: int = 60 * 24    # 24 小时
    
    # 限流
    RATE_LIMIT_PER_MINUTE: int = 60
    
    # Sync Engine
    SYNC_INTERVAL_SECONDS: int = 300     # 5 分钟
    SYNC_ALERT_THRESHOLD: int = 3        # 连续失败 3 次告警
    
    # Circuit Breaker
    CB_FAILURE_THRESHOLD: int = 5        # 连续失败 5 次熔断
    CB_RECOVERY_TIMEOUT: int = 30        # 30 秒恢复探测
    
    # 告警（钉钉 webhook，可选）
    DINGTALK_WEBHOOK_URL: str = ""
    
    # 文件上传
    UPLOAD_MAX_SIZE_MB: int = 10
    UPLOAD_ALLOWED_EXTENSIONS: list[str] = [".csv", ".xls", ".xlsx", ".zip"]
    UPLOAD_DIR: str = "/tmp/mailhub/uploads"

    class Config:
        env_file = ".env"

settings = Settings()
```

### 5.2 统一响应模型 `app/schemas/common.py`

```python
from typing import Generic, TypeVar, Optional, Any
from pydantic import BaseModel

T = TypeVar("T")

class ApiResponse(BaseModel, Generic[T]):
    """统一 API 响应格式。赛邮返回 Success/Fail/Error/1/-1 全部映射为此结构。"""
    code: int = 0          # 0=成功, 1xxx=参数错, 2xxx=鉴权错, 3xxx=限流, 9xxx=未知
    message: str = "success"
    data: Optional[T] = None

class PaginatedResponse(BaseModel, Generic[T]):
    code: int = 0
    message: str = "success"
    data: list[T] = []
    total: int = 0
    page: int = 1
    page_size: int = 20
```

### 5.3 错误映射器 `app/adapter/error_mapper.py`

```python
"""将赛邮 SOAP 的混乱返回值统一映射为 ApiResponse"""

SPREAD_ERROR_MAP = {
    "Success": (0, "success"),
    "Fail": (9001, "operation failed"),
    "Error": (9002, "service error"),
    "1": (0, "success"),        # PHP 示例返回 1 表示成功
    "-1": (9001, "operation failed"),
    "": (9003, "empty response"),
    None: (9003, "empty response"),
}

def map_spread_result(raw_result: str | int | None, method: str = "") -> dict:
    """
    将赛邮 SOAP 原始返回统一映射。
    赛邮文档说返回 Success/Fail/Error，但 PHP 示例返回 1/-1。
    """
    key = str(raw_result) if raw_result is not None else ""
    code, message = SPREAD_ERROR_MAP.get(key, (9999, f"unknown response: {key}"))
    return {
        "code": code,
        "message": f"[{method}] {message}" if method else message,
        "raw": raw_result,
    }
```

### 5.4 SOAP 客户端 `app/adapter/spread_client.py`

```python
"""赛邮 SOAP 客户端封装。业务层禁止直接使用此模块，必须通过 SpreadDriver。"""

import zeep
from zeep.transports import Transport
from zeep.exceptions import Fault
from app.config import settings
from app.adapter.error_mapper import map_spread_result
import structlog

logger = structlog.get_logger()

class SpreadSoapClient:
    """赛邮 SOAP asmx 客户端。单例模式，全局只创建一个 zeep client。"""

    _instance = None
    _client = None

    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance

    def _get_client(self):
        if self._client is None:
            transport = Transport(timeout=settings.SPREAD_TIMEOUT)
            self._client = zeep.Client(
                wsdl=settings.SPREAD_WSDL_URL,
                transport=transport,
            )
            logger.info("spread_soap_client_initialized", wsdl=settings.SPREAD_WSDL_URL)
        return self._client

    def call(self, method: str, **kwargs) -> dict:
        """
        统一 SOAP 调用入口。
        自动注入 loginEmail 和 password，业务层不需要关心鉴权。
        """
        client = self._get_client()
        try:
            result = getattr(client.service, method)(
                loginEmail=settings.SPREAD_LOGIN_EMAIL,
                password=settings.SPREAD_PASSWORD,
                **kwargs,
            )
            mapped = map_spread_result(result, method)
            logger.info("spread_soap_call", method=method, code=mapped["code"])
            return mapped
        except Fault as e:
            logger.error("spread_soap_fault", method=method, fault=str(e))
            return {"code": 9002, "message": f"[{method}] SOAP fault: {e.message}", "raw": None}
        except Exception as e:
            logger.error("spread_soap_error", method=method, error=str(e))
            return {"code": 9999, "message": f"[{method}] {str(e)}", "raw": None}
```

### 5.5 发送驱动接口 `app/adapter/base.py`

```python
"""可插拔发送驱动抽象。业务层只依赖此接口，不知道底层是赛邮还是 SendGrid。"""

from abc import ABC, abstractmethod
from dataclasses import dataclass
from typing import Optional

@dataclass
class SendResult:
    success: bool
    code: int
    message: str
    external_id: Optional[str] = None   # 赛邮返回的 campaign_id 等

@dataclass
class ReportData:
    sent: int = 0
    delivered: int = 0
    opens: int = 0
    clicks: int = 0
    bounces: int = 0
    unsubscribes: int = 0
    complaints: int = 0
    conversions: int = 0

class SendDriver(ABC):
    """发送驱动接口。所有发送通道必须实现此接口。"""

    @abstractmethod
    async def add_contact(self, email: str, list_id: int, **kwargs) -> SendResult: ...

    @abstractmethod
    async def add_contacts_batch(self, contacts: list[dict], list_id: int) -> SendResult: ...

    @abstractmethod
    async def create_list(self, name: str) -> SendResult: ...

    @abstractmethod
    async def get_lists(self) -> SendResult: ...

    @abstractmethod
    async def update_contact_status(self, email: str, status: str) -> SendResult: ...

    @abstractmethod
    async def create_email_campaign(self, campaign_data: dict) -> SendResult: ...

    @abstractmethod
    async def send_transactional(self, to: str, subject: str, body: str, **kwargs) -> SendResult: ...

    @abstractmethod
    async def create_sms_campaign(self, campaign_data: dict) -> SendResult: ...

    @abstractmethod
    async def create_mms_campaign(self, campaign_data: dict) -> SendResult: ...

    @abstractmethod
    async def get_campaign_report(self, campaign_id: str) -> ReportData: ...

    @abstractmethod
    async def get_campaign_opens(self, campaign_id: str, **kwargs) -> SendResult: ...

    @abstractmethod
    async def get_campaign_bounces(self, campaign_id: str, **kwargs) -> SendResult: ...

    @abstractmethod
    async def get_campaign_unsubscribes(self, campaign_id: str, **kwargs) -> SendResult: ...

    @abstractmethod
    async def get_campaign_abuses(self, campaign_id: str, **kwargs) -> SendResult: ...
```

### 5.6 赛邮驱动实现 `app/adapter/spread_driver.py`

```python
"""赛邮 SOAP 驱动。实现 SendDriver 接口，内部调 SpreadSoapClient。"""

from app.adapter.base import SendDriver, SendResult, ReportData
from app.adapter.spread_client import SpreadSoapClient
from app.adapter.circuit_breaker import CircuitBreaker
from app.adapter.retry import with_retry

class SpreadDriver(SendDriver):
    def __init__(self):
        self.client = SpreadSoapClient()
        self.breaker = CircuitBreaker(
            failure_threshold=5,
            recovery_timeout=30,
        )

    @with_retry(max_retries=3, backoff_base=1.0)
    async def add_contact(self, email: str, list_id: int, **kwargs) -> SendResult:
        """添加联系人到名单 → addSubscriberByInfo"""
        def _call():
            return self.breaker.call(
                self.client.call,
                "addSubscriberByInfo",
                subscriberEmail=email,
                subscriptionID=list_id,
                **kwargs,
            )
        result = _call()
        return SendResult(
            success=result["code"] == 0,
            code=result["code"],
            message=result["message"],
        )

    @with_retry(max_retries=3, backoff_base=1.0)
    async def create_list(self, name: str) -> SendResult:
        """创建联系人名单 → createSubscription"""
        result = self.breaker.call(
            self.client.call,
            "createSubscription",
            subscriptionName=name,
        )
        return SendResult(
            success=result["code"] == 0,
            code=result["code"],
            message=result["message"],
        )

    @with_retry(max_retries=2, backoff_base=2.0)
    async def update_contact_status(self, email: str, status: str) -> SendResult:
        """更新联系人状态（退订等）→ UpdateSubscriberStatus"""
        result = self.breaker.call(
            self.client.call,
            "UpdateSubscriberStatus",
            subscriberEmail=email,
            subscriberStatus=status,
        )
        return SendResult(
            success=result["code"] == 0,
            code=result["code"],
            message=result["message"],
        )

    @with_retry(max_retries=2, backoff_base=2.0)
    async def create_email_campaign(self, campaign_data: dict) -> SendResult:
        """创建邮件活动 → createCampaign"""
        result = self.breaker.call(
            self.client.call,
            "createCampaign",
            **campaign_data,
        )
        return SendResult(
            success=result["code"] == 0,
            code=result["code"],
            message=result["message"],
            external_id=result.get("raw"),
        )

    # ... 其余方法同理，全部通过 self.breaker.call() 调用

    async def get_campaign_report(self, campaign_id: str) -> ReportData:
        """获取活动报告 → GetCampaignReport"""
        result = self.breaker.call(
            self.client.call,
            "GetCampaignReport",
            campaignID=campaign_id,
        )
        # 解析赛邮返回的报表数据，映射到 ReportData
        # 实际字段名需根据 WSDL 确认
        return ReportData(
            sent=result.get("sent", 0),
            delivered=result.get("delivered", 0),
            opens=result.get("opens", 0),
            clicks=result.get("clicks", 0),
            bounces=result.get("bounces", 0),
            unsubscribes=result.get("unsubscribes", 0),
            complaints=result.get("abuses", 0),
            conversions=result.get("conversions", 0),
        )
```

### 5.7 熔断器 `app/adapter/circuit_breaker.py`

```python
"""Circuit Breaker — 赛邮故障时熔断，防止雪崩。"""

import time
import structlog
from enum import Enum

logger = structlog.get_logger()

class CircuitState(Enum):
    CLOSED = "closed"       # 正常
    OPEN = "open"           # 熔断中
    HALF_OPEN = "half_open" # 恢复探测

class CircuitBreaker:
    def __init__(self, failure_threshold: int = 5, recovery_timeout: int = 30):
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.state = CircuitState.CLOSED
        self.failure_count = 0
        self.last_failure_time = 0
        self.success_count = 0

    def call(self, func, *args, **kwargs):
        if self.state == CircuitState.OPEN:
            if time.time() - self.last_failure_time >= self.recovery_timeout:
                self.state = CircuitState.HALF_OPEN
                self.success_count = 0
                logger.info("circuit_breaker_half_open")
            else:
                logger.warn("circuit_breaker_open_rejected")
                return {
                    "code": 9100,
                    "message": "Service temporarily unavailable (circuit breaker open)",
                    "raw": None,
                }

        try:
            result = func(*args, **kwargs)
            if result.get("code", 9999) == 0:
                self._on_success()
            else:
                self._on_failure()
            return result
        except Exception as e:
            self._on_failure()
            logger.error("circuit_breaker_exception", error=str(e))
            return {"code": 9999, "message": str(e), "raw": None}

    def _on_success(self):
        self.failure_count = 0
        if self.state == CircuitState.HALF_OPEN:
            self.success_count += 1
            if self.success_count >= 2:  # 连续 2 次成功才恢复
                self.state = CircuitState.CLOSED
                logger.info("circuit_breaker_closed")

    def _on_failure(self):
        self.failure_count += 1
        self.last_failure_time = time.time()
        if self.failure_count >= self.failure_threshold:
            self.state = CircuitState.OPEN
            logger.error("circuit_breaker_opened", failures=self.failure_count)
            # TODO: 触发告警（钉钉/Slack）
```

### 5.8 重试装饰器 `app/adapter/retry.py`

```python
"""指数退避重试装饰器"""

import asyncio
import functools
import structlog

logger = structlog.get_logger()

def with_retry(max_retries: int = 3, backoff_base: float = 1.0):
    """
    指数退避重试。
    backoff_base=1.0 → 1s, 2s, 4s, 8s, 16s...
    """
    def decorator(func):
        @functools.wraps(func)
        async def wrapper(*args, **kwargs):
            last_error = None
            for attempt in range(max_retries + 1):
                try:
                    return await func(*args, **kwargs)
                except Exception as e:
                    last_error = e
                    if attempt < max_retries:
                        wait = backoff_base * (2 ** attempt)
                        logger.warn("retry_scheduled",
                                  func=func.__name__,
                                  attempt=attempt + 1,
                                  wait_seconds=wait,
                                  error=str(e))
                        await asyncio.sleep(wait)
                    else:
                        logger.error("retry_exhausted",
                                   func=func.__name__,
                                   attempts=max_retries + 1,
                                   error=str(e))
            raise last_error
        return wrapper
    return decorator
```

### 5.9 幂等中间件 `app/adapter/idempotency.py`

```python
"""幂等键去重。防止重复创建 campaign / 重复添加联系人。"""

from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select
from app.models.send_job import SendJob
import structlog

logger = structlog.get_logger()

async def check_idempotency(db: AsyncSession, key: str) -> SendJob | None:
    """检查幂等键是否已存在。存在 → 返回已有记录，不调 SOAP。"""
    if not key:
        return None
    result = await db.execute(
        select(SendJob).where(SendJob.idempotency_key == key)
    )
    existing = result.scalar_one_or_none()
    if existing:
        logger.info("idempotency_hit", key=key, job_id=existing.id, status=existing.status)
    return existing

async def create_send_job(db: AsyncSession, campaign_id: int, key: str) -> SendJob:
    """创建发送任务记录（幂等键唯一约束保证不重复）"""
    job = SendJob(
        campaign_id=campaign_id,
        idempotency_key=key,
        status="queued",
    )
    db.add(job)
    await db.flush()
    return job
```

### 5.10 退订服务 `app/services/compliance_service.py`

```python
"""退订闭环服务。路径 A：用户在我方退订 → 本地 + 赛邮双向标记。"""

from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select
from app.models.contact import Contact
from app.models.unsubscribe_log import UnsubscribeLog
from app.models.sync_log import SyncLog
from app.adapter.spread_driver import SpreadDriver
import structlog

logger = structlog.get_logger()

class ComplianceService:
    def __init__(self, db: AsyncSession, driver: SpreadDriver):
        self.db = db
        self.driver = driver

    async def unsubscribe(self, tenant_id: int, email: str, channel: str = "link") -> dict:
        """
        路径 A：用户在我方系统退订。
        必须同时完成：① 本地标记 ② 调赛邮 UpdateSubscriberStatus ③ 写日志
        """
        # 1. 查本地联系人
        result = await self.db.execute(
            select(Contact).where(
                Contact.tenant_id == tenant_id,
                Contact.email == email,
            )
        )
        contact = result.scalar_one_or_none()
        if not contact:
            return {"code": 1001, "message": "Contact not found"}

        # 2. 本地标记退订
        contact.status = "Unsubscribed"
        await self.db.flush()

        # 3. 调赛邮同步
        spread_result = await self.driver.update_contact_status(email, "Unsubscribed")

        # 4. 写退订日志
        log = UnsubscribeLog(
            contact_id=contact.id,
            channel=channel,
            synced_to_spread=(spread_result.code == 0),
        )
        self.db.add(log)

        # 5. 写同步日志
        sync_log = SyncLog(
            direction="out",
            entity="contact",
            entity_id=contact.id,
            spread_status="Unsubscribed" if spread_result.code == 0 else "FAILED",
            local_status="Unsubscribed",
            error=spread_result.message if spread_result.code != 0 else None,
        )
        self.db.add(sync_log)

        await self.db.commit()

        logger.info("unsubscribe_completed",
                   email=email, channel=channel,
                   spread_synced=(spread_result.code == 0))

        return {
            "code": 0 if spread_result.code == 0 else 9001,
            "message": "Unsubscribed successfully" if spread_result.code == 0 else f"Local OK, Spread sync failed: {spread_result.message}",
        }
```

### 5.11 Sync Engine `app/workers/sync_engine.py`

```python
"""Sync Engine — 路径 B：定时轮询赛邮，反向同步退订/弹回/投诉。"""

from arq import cron
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select
from app.database import async_session
from app.models.contact import Contact
from app.models.sync_log import SyncLog
from app.adapter.spread_driver import SpreadDriver
from app.config import settings
import structlog

logger = structlog.get_logger()

async def sync_from_spread(ctx: dict):
    """
    定时任务：每 5 分钟执行一次。
    从赛邮拉取退订/弹回/投诉列表，反向写入本地 DB。
    """
    driver = SpreadDriver()
    async with async_session() as db:
        try:
            # 1. 拉取赛邮退订列表
            unsub_result = await driver.get_campaign_unsubscribes(
                campaign_id="",  # 空 = 全部活动
            )
            if unsub_result.code == 0 and unsub_result.raw:
                await _process_unsubscribes(db, unsub_result.raw)

            # 2. 拉取弹回列表
            bounce_result = await driver.get_campaign_bounces(campaign_id="")
            if bounce_result.code == 0 and bounce_result.raw:
                await _process_bounces(db, bounce_result.raw)

            # 3. 拉取投诉列表
            abuse_result = await driver.get_campaign_abuses(campaign_id="")
            if abuse_result.code == 0 and abuse_result.raw:
                await _process_abuses(db, abuse_result.raw)

            await db.commit()
            logger.info("sync_engine_completed")

        except Exception as e:
            logger.error("sync_engine_failed", error=str(e))
            await db.rollback()

async def _process_unsubscribes(db: AsyncSession, emails: list[str]):
    """处理赛邮侧退订的联系人"""
    for email in emails:
        result = await db.execute(
            select(Contact).where(Contact.email == email, Contact.status != "Unsubscribed")
        )
        contact = result.scalar_one_or_none()
        if contact:
            contact.status = "Unsubscribed"
            sync_log = SyncLog(
                direction="in",
                entity="contact",
                entity_id=contact.id,
                spread_status="Unsubscribed",
                local_status="Unsubscribed",
            )
            db.add(sync_log)
            logger.info("sync_unsubscribe_from_spread", email=email)

async def _process_bounces(db: AsyncSession, emails: list[str]):
    """处理弹回联系人"""
    for email in emails:
        result = await db.execute(
            select(Contact).where(Contact.email == email, Contact.status == "Active")
        )
        contact = result.scalar_one_or_none()
        if contact:
            contact.status = "DoNotMail"
            sync_log = SyncLog(
                direction="in", entity="contact", entity_id=contact.id,
                spread_status="Bounced", local_status="DoNotMail",
            )
            db.add(sync_log)

async def _process_abuses(db: AsyncSession, emails: list[str]):
    """处理投诉联系人 → 加入黑名单"""
    for email in emails:
        result = await db.execute(
            select(Contact).where(Contact.email == email, Contact.status != "Blacklist")
        )
        contact = result.scalar_one_or_none()
        if contact:
            contact.status = "Blacklist"
            sync_log = SyncLog(
                direction="in", entity="contact", entity_id=contact.id,
                spread_status="Abuse", local_status="Blacklist",
            )
            db.add(sync_log)

# arq 定时任务注册
class WorkerSettings:
    functions = [sync_from_spread]
    cron_jobs = [
        cron(sync_from_spread, second=0, minute=set(range(0, 60, 5))),  # 每 5 分钟
    ]
```

### 5.12 联系人 API 路由 `app/api/v1/contacts.py`

```python
"""联系人 API。所有端点必须经过认证 + 租户隔离。"""

from fastapi import APIRouter, Depends, HTTPException, UploadFile
from sqlalchemy.ext.asyncio import AsyncSession
from app.database import get_db
from app.api.deps import get_current_tenant
from app.schemas.contact import ContactCreate, ContactUpdate, ContactStatusUpdate
from app.schemas.common import ApiResponse, PaginatedResponse
from app.services.contact_service import ContactService
from app.config import settings

router = APIRouter(prefix="/contacts", tags=["contacts"])

@router.post("", response_model=ApiResponse)
async def create_contact(
    data: ContactCreate,
    tenant=Depends(get_current_tenant),
    db: AsyncSession = Depends(get_db),
):
    """添加联系人 → addSubscriberByInfo / addSubscriberByEmail"""
    svc = ContactService(db, tenant.id)
    return await svc.create(data)

@router.post("/batch", response_model=ApiResponse)
async def batch_create_contacts(
    contacts: list[ContactCreate],
    tenant=Depends(get_current_tenant),
    db: AsyncSession = Depends(get_db),
):
    """批量添加 → addSubscribersByInfo"""
    svc = ContactService(db, tenant.id)
    return await svc.batch_create(contacts)

@router.post("/import", response_model=ApiResponse)
async def import_contacts(
    file: UploadFile,
    tenant=Depends(get_current_tenant),
    db: AsyncSession = Depends(get_db),
):
    """导入 CSV/XLS/XLSX → UploadContactListFile"""
    # 约束 8：检查文件大小和类型
    if file.size > settings.UPLOAD_MAX_SIZE_MB * 1024 * 1024:
        raise HTTPException(413, f"File too large (max {settings.UPLOAD_MAX_SIZE_MB}MB)")
    ext = "." + file.filename.rsplit(".", 1)[-1].lower()
    if ext not in settings.UPLOAD_ALLOWED_EXTENSIONS:
        raise HTTPException(400, f"File type {ext} not allowed")
    svc = ContactService(db, tenant.id)
    return await svc.import_file(file)

@router.get("", response_model=PaginatedResponse)
async def list_contacts(
    page: int = 1,
    page_size: int = 20,
    status: str = None,
    search: str = None,
    tenant=Depends(get_current_tenant),
    db: AsyncSession = Depends(get_db),
):
    """查询联系人列表"""
    svc = ContactService(db, tenant.id)
    return await svc.list(page, page_size, status, search)

@router.post("/{contact_id}/status", response_model=ApiResponse)
async def update_contact_status(
    contact_id: int,
    data: ContactStatusUpdate,
    tenant=Depends(get_current_tenant),
    db: AsyncSession = Depends(get_db),
):
    """更新联系人状态（退订等）→ UpdateSubscriberStatus"""
    svc = ContactService(db, tenant.id)
    return await svc.update_status(contact_id, data.status)
```

### 5.13 发送活动 API 路由 `app/api/v1/campaigns.py`

```python
"""活动 API。发送必须走队列（约束 5），不直发。"""

from fastapi import APIRouter, Depends
from sqlalchemy.ext.asyncio import AsyncSession
from app.database import get_db
from app.api.deps import get_current_tenant
from app.schemas.campaign import CampaignCreate
from app.schemas.common import ApiResponse
from app.services.campaign_service import CampaignService
import uuid

router = APIRouter(prefix="/campaigns", tags=["campaigns"])

@router.post("/email", response_model=ApiResponse)
async def create_email_campaign(
    data: CampaignCreate,
    tenant=Depends(get_current_tenant),
    db: AsyncSession = Depends(get_db),
):
    """
    创建邮件活动。
    约束 3：幂等。约束 5：走队列不直发。
    """
    # 幂等键：客户端传入或自动生成
    idem_key = data.idempotency_key or str(uuid.uuid4())
    svc = CampaignService(db, tenant.id)
    return await svc.create_and_queue(data, idem_key)

@router.post("/{campaign_id}/publish", response_model=ApiResponse)
async def publish_campaign(
    campaign_id: int,
    tenant=Depends(get_current_tenant),
    db: AsyncSession = Depends(get_db),
):
    """发布/暂停活动 → ChangePublishStatus"""
    svc = CampaignService(db, tenant.id)
    return await svc.change_publish_status(campaign_id)

@router.get("/{campaign_id}/status", response_model=ApiResponse)
async def get_campaign_status(
    campaign_id: int,
    tenant=Depends(get_current_tenant),
    db: AsyncSession = Depends(get_db),
):
    """获取活动状态 → getCampaignStatus"""
    svc = CampaignService(db, tenant.id)
    return await svc.get_status(campaign_id)
```

### 5.14 活动服务（队列模式）`app/services/campaign_service.py`

```python
"""活动服务。约束 5：发送走队列，不直发。"""

from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select
from app.models.campaign import Campaign
from app.models.send_job import SendJob
from app.schemas.campaign import CampaignCreate
from app.schemas.common import ApiResponse
from app.adapter.idempotency import check_idempotency, create_send_job
from app.workers.send_worker import enqueue_send_job
import structlog

logger = structlog.get_logger()

class CampaignService:
    def __init__(self, db: AsyncSession, tenant_id: int):
        self.db = db
        self.tenant_id = tenant_id

    async def create_and_queue(self, data: CampaignCreate, idem_key: str) -> ApiResponse:
        """创建活动并入队。不直接调 SOAP。"""

        # 1. 幂等检查（约束 3）
        existing = await check_idempotency(self.db, idem_key)
        if existing:
            return ApiResponse(
                code=0,
                message="Already queued (idempotent)",
                data={"job_id": existing.id, "status": existing.status},
            )

        # 2. 创建 campaign 记录
        campaign = Campaign(
            tenant_id=self.tenant_id,
            type="email",
            name=data.name,
            subject=data.subject,
            body=data.body,
            list_id=data.list_id,
            status="draft",
        )
        self.db.add(campaign)
        await self.db.flush()

        # 3. 创建 send_job + 幂等键
        job = await create_send_job(self.db, campaign.id, idem_key)

        # 4. 更新 campaign 状态
        campaign.status = "queued"
        await self.db.commit()

        # 5. 入 Redis 队列 → Send Worker 异步处理
        await enqueue_send_job(job.id)

        logger.info("campaign_queued", campaign_id=campaign.id, job_id=job.id, key=idem_key)

        return ApiResponse(
            code=0,
            message="Campaign queued for sending",
            data={"campaign_id": campaign.id, "job_id": job.id, "status": "queued"},
        )
```

### 5.15 Send Worker `app/workers/send_worker.py`

```python
"""Send Worker — 消费队列，调适配层发送。"""

from arq import cron
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select
from app.database import async_session
from app.models.send_job import SendJob
from app.models.campaign import Campaign
from app.adapter.spread_driver import SpreadDriver
import structlog

logger = structlog.get_logger()

async def enqueue_send_job(job_id: int):
    """入队（由 API 层调用）"""
    from arq.connections import create_pool
    from app.workers.settings import WorkerSettings
    pool = await create_pool(WorkerSettings.redis_settings)
    await pool.enqueue_job("process_send_job", job_id)

async def process_send_job(ctx: dict, job_id: int):
    """
    Worker 消费：取出 send_job → 调适配层 → 更新状态。
    """
    driver = SpreadDriver()
    async with async_session() as db:
        # 1. 取出任务
        result = await db.execute(select(SendJob).where(SendJob.id == job_id))
        job = result.scalar_one_or_none()
        if not job or job.status != "queued":
            return

        # 2. 取出 campaign
        result = await db.execute(select(Campaign).where(Campaign.id == job.campaign_id))
        campaign = result.scalar_one_or_none()
        if not campaign:
            job.status = "failed"
            job.error = "Campaign not found"
            await db.commit()
            return

        # 3. 更新状态为 sending
        job.status = "sending"
        campaign.status = "sending"
        await db.commit()

        # 4. 调适配层发送
        try:
            send_result = await driver.create_email_campaign({
                "campaignName": campaign.name,
                "subject": campaign.subject,
                "body": campaign.body,
                "listID": campaign.list_id,
            })

            if send_result.success:
                job.status = "completed"
                campaign.status = "completed"
                campaign.sent_at = func.now()
            else:
                job.status = "failed"
                job.error = send_result.message
                job.retry_count += 1
                campaign.status = "failed"

        except Exception as e:
            job.status = "failed"
            job.error = str(e)
            job.retry_count += 1
            campaign.status = "failed"
            logger.error("send_job_failed", job_id=job_id, error=str(e))

        await db.commit()
        logger.info("send_job_processed", job_id=job_id, status=job.status)
```

### 5.16 健康检查 `app/workers/health_check.py`

```python
"""Health Check — 定时探活赛邮 + 指标异常告警。"""

from arq import cron
from app.adapter.spread_client import SpreadSoapClient
from app.config import settings
import structlog

logger = structlog.get_logger()

async def health_check(ctx: dict):
    """每 60 秒执行一次"""
    client = SpreadSoapClient()

    # 轻量探活：获取名单列表（最快的调用）
    result = client.call("getAllSubscription")

    if result["code"] != 0:
        logger.error("health_check_failed", code=result["code"], message=result["message"])
        # TODO: 触发告警
    else:
        logger.info("health_check_ok")

# arq 注册
class HealthCheckSettings:
    functions = [health_check]
    cron_jobs = [
        cron(health_check, second=0),  # 每分钟
    ]
```

### 5.17 前端入口 `frontend/src/App.tsx`（P0 简化版）

```tsx
import { BrowserRouter, Routes, Route, Navigate } from 'react-router-dom';
import Dashboard from './pages/Dashboard';
import Contacts from './pages/Contacts';
import Lists from './pages/Lists';
import SendEmail from './pages/SendEmail';
import Sidebar from './components/Sidebar';

function App() {
  return (
    <BrowserRouter>
      <div className="flex h-screen bg-gray-50">
        <Sidebar />
        <main className="flex-1 overflow-auto p-6">
          <Routes>
            <Route path="/" element={<Navigate to="/dashboard" />} />
            <Route path="/dashboard" element={<Dashboard />} />
            <Route path="/contacts" element={<Contacts />} />
            <Route path="/lists" element={<Lists />} />
            <Route path="/send" element={<SendEmail />} />
          </Routes>
        </main>
      </div>
    </BrowserRouter>
  );
}

export default App;
```

---

## 六、环境配置 `.env.example`

```bash
# 数据库
DATABASE_URL=postgresql+asyncpg://mailhub:mailhub@localhost:5432/mailhub

# Redis
REDIS_URL=redis://localhost:6379/0

# 赛邮 SOAP（必须填写，从 https://app.rspread.com/MyAccount/AccountAPI.aspx 获取）
SPREAD_WSDL_URL=https://service.rspread.com/Service.asmx?WSDL
SPREAD_LOGIN_EMAIL=your-spread-account@example.com
SPREAD_PASSWORD=your-spread-password-or-api-key

# JWT
JWT_SECRET=generate-a-random-secret-key-here
JWT_ALGORITHM=HS256
JWT_EXPIRE_MINUTES=1440

# 限流
RATE_LIMIT_PER_MINUTE=60

# Sync Engine
SYNC_INTERVAL_SECONDS=300
SYNC_ALERT_THRESHOLD=3

# Circuit Breaker
CB_FAILURE_THRESHOLD=5
CB_RECOVERY_TIMEOUT=30

# 告警（可选）
DINGTALK_WEBHOOK_URL=

# 文件上传
UPLOAD_MAX_SIZE_MB=10
UPLOAD_DIR=/tmp/mailhub/uploads
```

---

## 七、Docker Compose

```yaml
version: "3.9"

services:
  api:
    build: .
    ports:
      - "8000:8000"
    env_file: .env
    depends_on:
      - db
      - redis
    volumes:
      - ./app:/app/app
    command: uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload

  worker:
    build: .
    env_file: .env
    depends_on:
      - db
      - redis
    volumes:
      - ./app:/app/app
    command: arq app.workers.settings.WorkerSettings

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: mailhub
      POSTGRES_PASSWORD: mailhub
      POSTGRES_DB: mailhub
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  frontend:
    build: ./frontend
    ports:
      - "3000:3000"
    volumes:
      - ./frontend/src:/app/src

volumes:
  pgdata:
```

---

## 八、实现顺序（严格按此顺序写代码）

```
Step 1: 基础设施
  ├── config.py          配置管理
  ├── database.py        async engine + session
  ├── models/base.py     Base + TimestampMixin
  └── alembic init + 001_initial.sql

Step 2: 适配层（最核心，先写）
  ├── error_mapper.py    错误统一映射
  ├── spread_client.py   zeep SOAP client
  ├── base.py            SendDriver ABC
  ├── circuit_breaker.py 熔断器
  ├── retry.py           重试装饰器
  ├── idempotency.py     幂等去重
  └── spread_driver.py   赛邮驱动实现

Step 3: 数据模型
  ├── models/ 全部 14 张表
  └── alembic migration

Step 4: 业务服务
  ├── contact_service.py
  ├── list_service.py
  ├── campaign_service.py (队列模式)
  └── compliance_service.py (退订闭环)

Step 5: API 路由
  ├── api/deps.py        认证 + 租户注入
  ├── api/v1/auth.py
  ├── api/v1/contacts.py
  ├── api/v1/lists.py
  ├── api/v1/campaigns.py
  ├── api/v1/emails.py
  ├── api/v1/unsubscribe.py
  └── api/v1/health.py

Step 6: Workers
  ├── workers/settings.py
  ├── workers/send_worker.py
  ├── workers/sync_engine.py
  └── workers/health_check.py

Step 7: 横切组件
  ├── core/security.py   JWT
  ├── core/rate_limit.py
  └── core/cache.py

Step 8: 前端（P0 简化版）
  ├── React + Vite + Tailwind 初始化
  ├── Sidebar 布局
  ├── Dashboard / Contacts / Lists / SendEmail 四个页面
  └── 调后端 API

Step 9: 测试
  ├── test_adapter/      适配层单元测试
  ├── test_api/          API 集成测试
  └── test_services/     业务逻辑测试

Step 10: Docker
  ├── Dockerfile
  ├── docker-compose.yml
  └── .env.example
```

---

## 九、验收标准（写完必须全部通过）

```
□ 1. docker compose up 一键启动，api + worker + db + redis 全部正常
□ 2. POST /v1/auth/token 返回 JWT token
□ 3. POST /v1/contacts 添加联系人，本地 DB 有记录 + 赛邮侧也有
□ 4. POST /v1/contacts/{id}/status {"status":"Unsubscribed"}，本地 + 赛邮双向标记
□ 5. POST /v1/campaigns/email 创建活动，返回 queued 状态（不直发）
□ 6. Send Worker 消费队列，campaign.status 变为 completed
□ 7. 同一 idempotency_key 重复调用，返回已有结果，不重复调 SOAP
□ 8. Sync Engine 每 5 分钟运行，sync_log 表有记录
□ 9. 赛邮 SOAP 连续失败 5 次，Circuit Breaker 熔断，返回 9100 错误码
□ 10. 所有 API 返回统一 ApiResponse 格式，前端看不到 SOAP 原始返回
```

---

## 十、禁止事项（写了直接打回）

```
× 禁止在业务层 import zeep
× 禁止在前端代码中出现 SPREAD_PASSWORD
× 禁止 createCampaign 直接调 SOAP（必须走队列）
× 禁止退订只写本地不调赛邮（必须双向）
× 禁止 SOAP 调用走 http://（必须 https://）
× 禁止在日志中输出 email/password/api_key 明文
× 禁止跳过幂等检查直接创建 send_job
× 禁止使用 requests 库（用 httpx）
× 禁止使用 Celery（用 arq）
× 禁止 models 中缺 tenant_id 字段
```

# Техническая архитектура

## Общая схема

```
┌─────────────────────────────────────────────────────────┐
│                    Внешний мир                          │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  ┌───────────┐    ┌──────────┐    ┌───────────────┐   │
│  │  Клиенты  │───▶│ Mail.ru  │◀───│  Поставщики   │   │
│  │   Email   │    │   IMAP   │    │     Email     │   │
│  └───────────┘    │   SMTP   │    └───────────────┘   │
│                    └──────────┘                          │
│                         │                                │
│  ┌─────────────────────┼───────────────────────────┐   │
│  │  Менеджеры          │                           │   │
│  │  ┌──────────┐   ┌───▼──────┐   ┌────────────┐  │   │
│  │  │ Telegram │   │   Web    │   │   Mobile   │  │   │
│  │  │   Bot    │   │  Admin   │   │  (future)  │  │   │
│  │  └──────────┘   └──────────┘   └────────────┘  │   │
│  └──────────────────────────────────────────────────┘   │
└──────────────────────┬──────────────────────────────────┘
                        │
┌───────────────────────▼──────────────────────────────────┐
│              Backend (FastAPI + Python)                   │
├───────────────────────────────────────────────────────────┤
│                                                            │
│  ┌─────────────────────────────────────────────────────┐ │
│  │                API Layer (FastAPI)                   │ │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────────────┐  │ │
│  │  │ Orders   │  │Suppliers │  │    Analytics     │  │ │
│  │  │   API    │  │   API    │  │       API        │  │ │
│  │  └──────────┘  └──────────┘  └──────────────────┘  │ │
│  └─────────────────────────────────────────────────────┘ │
│                                                            │
│  ┌─────────────────────────────────────────────────────┐ │
│  │              Services Layer                          │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────┐  │ │
│  │  │Email Service │  │  AI Parser   │  │   PDF    │  │ │
│  │  │IMAP/SMTP     │  │ OpenAI GPT-4 │  │Generator │  │ │
│  │  └──────────────┘  └──────────────┘  └──────────┘  │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────┐  │ │
│  │  │ Notification │  │  Numbering   │  │  Audit   │  │ │
│  │  │   Service    │  │   Service    │  │  Logger  │  │ │
│  │  └──────────────┘  └──────────────┘  └──────────┘  │ │
│  └─────────────────────────────────────────────────────┘ │
│                                                            │
│  ┌─────────────────────────────────────────────────────┐ │
│  │          Telegram Bot (aiogram 3.x)                  │ │
│  │  ┌──────────┐  ┌───────────┐  ┌────────────────┐   │ │
│  │  │ Orders   │  │ Suppliers │  │ Notifications  │   │ │
│  │  │ Handler  │  │  Handler  │  │    Handler     │   │ │
│  │  └──────────┘  └───────────┘  └────────────────┘   │ │
│  └─────────────────────────────────────────────────────┘ │
│                                                            │
│  ┌─────────────────────────────────────────────────────┐ │
│  │         Database Layer (SQLAlchemy ORM)              │ │
│  │  ┌──────┐ ┌─────────┐ ┌──────────┐ ┌──────────┐    │ │
│  │  │Models│ │Migrati│ │Connectio│ │Repositor│    │ │
│  │  │      │ │ons    │ │n Pool   │ │ies      │    │ │
│  │  └──────┘ └─────────┘ └──────────┘ └──────────┘    │ │
│  └─────────────────────────────────────────────────────┘ │
└───────────────────────┬──────────────────────────────────┘
                         │
┌────────────────────────▼─────────────────────────────────┐
│                 Data Layer                                │
├───────────────────────────────────────────────────────────┤
│  ┌──────────────┐  ┌──────────┐  ┌──────────────────┐   │
│  │ PostgreSQL   │  │  Redis   │  │ Cloudflare R2    │   │
│  │  (Supabase)  │  │  Cache   │  │  (File Storage)  │   │
│  └──────────────┘  └──────────┘  └──────────────────┘   │
└───────────────────────────────────────────────────────────┘
```

## Модули системы

### 1. API Layer (FastAPI)

**Отвественность**: REST API endpoints для веб-интерфейса

**Компоненты**:
- `api/orders.py` - CRUD заявок
- `api/suppliers.py` - CRUD поставщиков
- `api/clients.py` - CRUD клиентов
- `api/quotes.py` - Работа с котировками
- `api/analytics.py` - Аналитика и отчеты

**Технологии**:
- FastAPI 0.109+
- Pydantic для валидации
- JWT для аутентификации

### 2. Services Layer

#### Email Service (`services/email_service.py`)
- IMAP polling (каждые 30 сек)
- SMTP отправка
- Парсинг email (from, subject, body, message-id)
- Трекинг через In-Reply-To

#### AI Parser (`services/ai_parser.py`)
- OpenAI GPT-4o-mini API
- Промпты для парсинга заявок и ответов
- Retry логика (3 попытки)
- Fallback при ошибке

#### PDF Generator (`services/pdf_generator.py`)
- Генерация КП (ReportLab/WeasyPrint)
- Шаблоны документов
- Сохранение в Cloudflare R2

#### Notification Service (`services/notification_service.py`)
- Отправка уведомлений в Telegram
- Форматирование сообщений
- Inline кнопки

### 3. Telegram Bot (aiogram)

**Handlers**:
- `bot/handlers/orders.py` - Управление заявками
- `bot/handlers/suppliers.py` - CRUD поставщиков
- `bot/handlers/notifications.py` - Обработка уведомлений

**Keyboards**:
- Inline клавиатуры для быстрых действий
- Reply клавиатуры для навигации

### 4. Database Layer

**SQLAlchemy Models**:
- Декларативные модели всех таблиц
- Relationships для связей
- Hybrid properties для вычисляемых полей

**Alembic Migrations**:
- Версионирование схемы БД
- Автогенерация миграций
- Rollback механизм

## Поток данных

### Email → Заявка

```
1. Email приходит на mail.ru
2. IMAP polling обнаруживает новое письмо
3. Email Service парсит headers + body
4. AI Parser извлекает структурированные данные
5. Order Service создает заявку + order_items
6. Audit Log фиксирует создание
7. Notification Service → Telegram уведомление
```

### Рассылка запросов

```
1. Менеджер выбирает поставщиков (Web/Telegram)
2. Email Service генерирует уникальный Message-ID
3. SMTP отправка письма поставщику
4. Supplier Request сохраняется в БД (status: sent)
5. Audit Log фиксирует отправку
```

### Прием ответов

```
1. Поставщик отвечает на письмо
2. IMAP polling обнаруживает ответ
3. Email Service находит исходный запрос по In-Reply-To
4. AI Parser извлекает цену, срок, условия
5. Supplier Quote создается в БД
6. Notification Service → Telegram уведомление
7. Audit Log фиксирует получение ответа
```

## Фоновые задачи

### Email Polling (async loop)
```python
async def email_polling_task():
    while True:
        try:
            await email_service.check_new_emails()
        except Exception as e:
            logger.error(f"Email polling failed: {e}")
        await asyncio.sleep(30)  # Каждые 30 сек
```

### Timeout проверка запросов
```python
async def check_request_timeouts():
    # Каждый час проверяем запросы старше 48 часов
    while True:
        timeout_requests = db.query(SupplierRequest).filter(
            SupplierRequest.status == 'sent',
            SupplierRequest.email_sent_at < now() - timedelta(hours=48)
        ).all()

        for req in timeout_requests:
            req.status = 'timeout'
            # Уведомление в Telegram

        await asyncio.sleep(3600)  # Каждый час
```

## Конфигурация

### Environment Variables (.env)
```env
# Database
DATABASE_URL=postgresql://user:pass@host:5432/db
REDIS_URL=redis://localhost:6379/0

# Telegram
TELEGRAM_BOT_TOKEN=123456:ABC...
TELEGRAM_ADMIN_IDS=123456789,987654321

# Email
EMAIL_USERNAME=your@mail.ru
EMAIL_PASSWORD=password
IMAP_SERVER=imap.mail.ru
SMTP_SERVER=smtp.mail.ru

# OpenAI
OPENAI_API_KEY=sk-...
OPENAI_MODEL=gpt-4o-mini

# Files
CLOUDFLARE_R2_ACCOUNT_ID=...
CLOUDFLARE_R2_ACCESS_KEY=...
CLOUDFLARE_R2_SECRET_KEY=...

# App
SECRET_KEY=random-secret-key
DEBUG=false
LOG_LEVEL=INFO
```

### Config класс (`backend/config.py`)
```python
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    # Database
    database_url: str
    redis_url: str

    # Telegram
    telegram_bot_token: str
    telegram_admin_ids: list[int]

    # Email
    email_username: str
    email_password: str
    imap_server: str = "imap.mail.ru"
    smtp_server: str = "smtp.mail.ru"

    # OpenAI
    openai_api_key: str
    openai_model: str = "gpt-4o-mini"

    # App
    secret_key: str
    debug: bool = False
    log_level: str = "INFO"

    class Config:
        env_file = ".env"

settings = Settings()
```

## Аутентификация

### Telegram Bot
- Проверка `user_id` в whitelist (`TELEGRAM_ADMIN_IDS`)
- Middleware для фильтрации

### Web Admin
- JWT токены
- Expiration: 24 часа
- Refresh tokens: нет (MVP)

### API
- JWT Bearer tokens
- Для будущих интеграций

## Логирование

### Structured Logging
```python
import structlog

logger = structlog.get_logger()

logger.info("order_created",
    order_id=52,
    order_number="ORD-2024-0052",
    client_id=15
)
```

### Уровни
- DEBUG - детальная информация
- INFO - обычные события
- WARNING - потенциальные проблемы
- ERROR - ошибки с fallback
- CRITICAL - система не работает

## Обработка ошибок

### Graceful Degradation
- AI парсинг упал → ручная обработка
- БД недоступна → кеш в Redis
- Email не отправился → retry 3 раза

### Circuit Breaker (для будущего)
```python
from pybreaker import CircuitBreaker

email_breaker = CircuitBreaker(fail_max=5, timeout_duration=60)

@email_breaker
def send_email(...):
    # Если 5 ошибок подряд → breaker открывается на 60 сек
    ...
```

## Мониторинг

### Health Checks
```python
@app.get("/health")
async def health():
    return {
        "status": "ok",
        "database": await check_db(),
        "redis": await check_redis(),
        "email": await check_email_connection()
    }
```

### Metrics (для будущего)
- Prometheus + Grafana
- Отслеживание: requests/sec, latency, errors

## Масштабирование (v2.0+)

### Horizontal Scaling
- Несколько инстансов FastAPI за Load Balancer
- Shared PostgreSQL + Redis

### Celery для фоновых задач
- Email polling → Celery beat
- AI парсинг → Celery worker
- PDF generation → Celery worker

### Кеширование
- Redis для частых запросов (список поставщиков)
- Cache invalidation при изменениях

---

**Важно**: Архитектура MVP - монолитная. Все компоненты в одном процессе. При росте нужно будет разделить на микросервисы.

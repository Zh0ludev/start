# Установка и деплой

## Локальная разработка

### Требования

- Python 3.11+
- PostgreSQL 15+
- Redis 7+
- Git

### Установка

```bash
# 1. Клонирование
git clone https://github.com/Zh0ludev/start.git
cd start

# 2. Виртуальное окружение
python -m venv venv
source venv/bin/activate  # Linux/Mac
# или
venv\Scripts\activate  # Windows

# 3. Зависимости
pip install -r requirements.txt

# 4. PostgreSQL + Redis (через Docker)
docker-compose up -d

# 5. Конфигурация
cp .env.example .env
# Отредактируйте .env
```

### .env файл

```env
# Database
DATABASE_URL=postgresql://postgres:postgres@localhost:5432/start_db
REDIS_URL=redis://localhost:6379/0

# Telegram
TELEGRAM_BOT_TOKEN=123456:ABC-YOUR-TOKEN
TELEGRAM_ADMIN_IDS=123456789,987654321

# Email (mail.ru)
EMAIL_USERNAME=your@mail.ru
EMAIL_PASSWORD=your_password
IMAP_SERVER=imap.mail.ru
SMTP_SERVER=smtp.mail.ru

# OpenAI
OPENAI_API_KEY=sk-YOUR-KEY
OPENAI_MODEL=gpt-4o-mini

# Cloudflare R2 (опционально для MVP)
CLOUDFLARE_R2_ACCOUNT_ID=your_account_id
CLOUDFLARE_R2_ACCESS_KEY=your_access_key
CLOUDFLARE_R2_SECRET_KEY=your_secret_key

# App
SECRET_KEY=generate-random-32-char-string
DEBUG=true
LOG_LEVEL=DEBUG
```

### Миграции БД

```bash
# Создание БД
createdb start_db

# Применение миграций
alembic upgrade head

# Создание тестовых данных (опционально)
python -m backend.database.seed
```

### Запуск

```bash
# Terminal 1: FastAPI
python backend/main.py
# http://localhost:8000

# Terminal 2: Telegram Bot
python backend/bot/main.py

# Terminal 3: Email Polling (если отдельный процесс)
python backend/services/email_polling.py
```

### Проверка

```bash
# Health check
curl http://localhost:8000/health

# API docs
open http://localhost:8000/docs

# Telegram bot
# Отправьте /start в Telegram
```

## Production деплой (Railway.app)

### Подготовка

```bash
# 1. Установка Railway CLI
npm install -g @railway/cli

# 2. Логин
railway login

# 3. Создание проекта
railway init
```

### railway.toml

```toml
[build]
builder = "NIXPACKS"
buildCommand = "pip install -r requirements.txt"

[deploy]
startCommand = "python backend/main.py & python backend/bot/main.py"
healthcheckPath = "/health"
healthcheckTimeout = 300
restartPolicyType = "ON_FAILURE"
restartPolicyMaxRetries = 10
```

### Переменные окружения

```bash
# Через Railway Dashboard или CLI
railway variables set DATABASE_URL=postgresql://...
railway variables set REDIS_URL=redis://...
railway variables set TELEGRAM_BOT_TOKEN=...
railway variables set OPENAI_API_KEY=...
railway variables set SECRET_KEY=...
railway variables set DEBUG=false
```

### PostgreSQL на Railway

```bash
# Создание БД
railway add postgresql

# Railway автоматически добавит DATABASE_URL

# Миграции (через Railway CLI)
railway run alembic upgrade head
```

### Деплой

```bash
# Push в GitHub → auto-deploy
git push origin main

# Или через Railway CLI
railway up

# Логи
railway logs
```

## Production деплой (Render.com)

### render.yaml

```yaml
services:
  - type: web
    name: start-api
    env: python
    buildCommand: "pip install -r requirements.txt"
    startCommand: "python backend/main.py & python backend/bot/main.py"
    envVars:
      - key: DATABASE_URL
        fromDatabase:
          name: start-db
          property: connectionString
      - key: REDIS_URL
        fromService:
          type: redis
          name: start-redis
          property: connectionString
      - key: PYTHON_VERSION
        value: 3.11.0

databases:
  - name: start-db
    databaseName: start_db
    user: start_user

  - name: start-redis
    plan: starter
```

### Деплой

1. Push в GitHub
2. Render Dashboard → New → Web Service
3. Connect repository
4. Render auto-deploy на каждый push

## Docker (опционально)

### Dockerfile

```dockerfile
FROM python:3.11-slim

WORKDIR /app

# Dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Code
COPY . .

# Alembic migrations
RUN alembic upgrade head

# Run
CMD ["sh", "-c", "python backend/main.py & python backend/bot/main.py"]
```

### docker-compose.yml

```yaml
version: '3.8'

services:
  db:
    image: postgres:15
    environment:
      POSTGRES_DB: start_db
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  app:
    build: .
    depends_on:
      - db
      - redis
    environment:
      DATABASE_URL: postgresql://postgres:postgres@db:5432/start_db
      REDIS_URL: redis://redis:6379/0
    env_file:
      - .env
    ports:
      - "8000:8000"

volumes:
  postgres_data:
```

### Запуск

```bash
docker-compose up -d
```

## Мониторинг

### Railway Dashboard
- Metrics: CPU, Memory, Requests
- Logs: real-time
- Auto-restart on failure

### UptimeRobot (бесплатно)
```
Monitor: HTTP(s)
URL: https://your-app.railway.app/health
Interval: 5 minutes
Alert: Telegram
```

### Sentry (для v1.1)

```bash
pip install sentry-sdk

# backend/main.py
import sentry_sdk
sentry_sdk.init(
    dsn="https://...@sentry.io/...",
    traces_sample_rate=0.1
)
```

## Backup стратегия

### PostgreSQL

**Railway автоматические бэкапы:**
- Daily snapshots
- 7 days retention

**Ручной бэкап:**
```bash
# Экспорт
pg_dump $DATABASE_URL > backup_$(date +%Y%m%d).sql

# Импорт
psql $DATABASE_URL < backup_20240206.sql
```

### Cloudflare R2

- Automatic versioning
- Lifecycle rules (delete after 90 days)

## Безопасность

### Secrets

```bash
# НЕ коммитить в Git
echo ".env" >> .gitignore
echo "*.pem" >> .gitignore
echo "*.key" >> .gitignore

# Использовать secrets manager (Railway/Render)
```

### HTTPS

- Railway: автоматический SSL
- Render: автоматический SSL
- Custom domain: бесплатный Let's Encrypt

### Firewall

```bash
# PostgreSQL - только с Railway IP
# Redis - только internal network
# API - rate limiting (v1.1)
```

## Troubleshooting

### Проблема: Telegram bot не отвечает

```bash
# Проверка токена
curl https://api.telegram.org/bot<TOKEN>/getMe

# Логи
railway logs --filter="telegram"

# Webhook конфликт
curl https://api.telegram.org/bot<TOKEN>/deleteWebhook
```

### Проблема: Email не приходят

```bash
# Проверка IMAP/SMTP
python -c "import imaplib; imaplib.IMAP4_SSL('imap.mail.ru').login('user', 'pass')"

# Логи
railway logs --filter="email"

# Mail.ru лимиты
# Проверьте: не заблокирован ли аккаунт за спам
```

### Проблема: БД медленная

```bash
# Анализ slow queries
SELECT query, calls, total_time
FROM pg_stat_statements
ORDER BY total_time DESC LIMIT 10;

# Vacuum
VACUUM ANALYZE;

# Добавить индексы (см. DATABASE.md)
```

## Scaling (для будущего)

### Horizontal scaling
```yaml
# railway.toml
[deploy]
numReplicas = 2  # Railway Pro план
```

### Caching
```python
# Redis cache для GET запросов
@cache(expire=300)
def get_suppliers():
    return db.query(Supplier).all()
```

### CDN для статики
- Cloudflare CDN
- Static files → R2
- Frontend → CDN edge

---

**Важно**: MVP деплоим на Railway Free tier ($5/мес). При росте - переход на Pro ($20/мес) с replicas и metrics.

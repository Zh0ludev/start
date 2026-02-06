# Схема базы данных

## Концепция: Сквозная связь сделок

Каждая заявка имеет **полную историю** от первого email до отправки КП. Все данные связаны через внешние ключи, что позволяет восстановить полную картину любой сделки.

## Основные таблицы

### users - Пользователи системы
```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    telegram_id BIGINT UNIQUE NOT NULL,
    username VARCHAR(100),
    full_name VARCHAR(255),
    role VARCHAR(50) DEFAULT 'admin',  -- admin, user
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT NOW()
);
```

### clients - Клиенты
```sql
CREATE TABLE clients (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    inn VARCHAR(12) UNIQUE NOT NULL,

    -- Контакты
    contact_person VARCHAR(255),
    email VARCHAR(255),
    phone VARCHAR(50),
    address TEXT,
    city VARCHAR(100),

    -- Метаданные
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    created_by INTEGER REFERENCES users(id)
);

CREATE INDEX idx_clients_inn ON clients(inn);
CREATE INDEX idx_clients_name ON clients(name);
```

### orders - Заявки
```sql
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    order_number VARCHAR(50) UNIQUE NOT NULL,  -- ORD-2024-0001
    client_id INTEGER REFERENCES clients(id),

    -- Исходные данные
    source_email_id VARCHAR(255),
    source_email_from VARCHAR(255),
    source_email_subject VARCHAR(500),
    source_email_body TEXT,
    source_email_date TIMESTAMP,

    -- Статус
    status VARCHAR(50) DEFAULT 'new',
        -- new, processing, quoted, closed, cancelled
    priority VARCHAR(20) DEFAULT 'normal',
        -- low, normal, high, urgent

    -- Спецификация
    specification TEXT,
    notes TEXT,

    -- Даты
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    deadline TIMESTAMP,
    closed_at TIMESTAMP,

    -- Кто работает
    assigned_to INTEGER REFERENCES users(id),
    created_by INTEGER REFERENCES users(id)
);

CREATE INDEX idx_orders_number ON orders(order_number);
CREATE INDEX idx_orders_status ON orders(status);
CREATE INDEX idx_orders_client ON orders(client_id);
CREATE INDEX idx_orders_created_at ON orders(created_at DESC);
```

### order_items - Позиции заявки
```sql
CREATE TABLE order_items (
    id SERIAL PRIMARY KEY,
    order_id INTEGER REFERENCES orders(id) ON DELETE CASCADE,

    item_number INTEGER NOT NULL,
    product_name VARCHAR(500) NOT NULL,
    quantity DECIMAL(10,2),
    unit VARCHAR(50),
    specifications TEXT,

    -- Результат
    selected_supplier_quote_id INTEGER,
    final_price DECIMAL(12,2),

    created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_order_items_order ON order_items(order_id);
```

### suppliers - Поставщики
```sql
CREATE TABLE suppliers (
    id SERIAL PRIMARY KEY,
    company_name VARCHAR(255) NOT NULL,
    inn VARCHAR(12) UNIQUE NOT NULL,

    -- Контакты
    contact_person VARCHAR(255),
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    email VARCHAR(255) NOT NULL,
    phone VARCHAR(50),
    website VARCHAR(255),
    address TEXT,
    city VARCHAR(100),

    -- Коммерческие условия
    categories TEXT[],
    min_order_amount DECIMAL(12,2),
    payment_terms TEXT,
    working_hours VARCHAR(100),

    -- Рейтинг
    rating DECIMAL(3,2) DEFAULT 0,
    priority INTEGER DEFAULT 0,
    is_favorite BOOLEAN DEFAULT false,

    -- Статистика
    total_requests INTEGER DEFAULT 0,
    total_quotes INTEGER DEFAULT 0,
    avg_response_time_hours INTEGER,

    -- Метаданные
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    created_by INTEGER REFERENCES users(id)
);

CREATE INDEX idx_suppliers_inn ON suppliers(inn);
CREATE INDEX idx_suppliers_categories ON suppliers USING GIN(categories);
CREATE INDEX idx_suppliers_city ON suppliers(city);
CREATE INDEX idx_suppliers_rating ON suppliers(rating DESC);
```

### supplier_requests - Запросы к поставщикам
```sql
CREATE TABLE supplier_requests (
    id SERIAL PRIMARY KEY,
    request_number VARCHAR(50) UNIQUE NOT NULL,  -- REQ-2024-0001
    order_id INTEGER REFERENCES orders(id) ON DELETE CASCADE,
    supplier_id INTEGER REFERENCES suppliers(id),

    -- Email рассылка
    email_subject VARCHAR(500),
    email_body TEXT,
    email_sent_at TIMESTAMP,
    email_message_id VARCHAR(255) UNIQUE,

    -- Статус
    status VARCHAR(50) DEFAULT 'pending',
        -- pending, sent, replied, timeout, error

    -- Ответ
    reply_received_at TIMESTAMP,
    reply_email_id VARCHAR(255),
    reply_email_body TEXT,

    created_at TIMESTAMP DEFAULT NOW(),
    created_by INTEGER REFERENCES users(id)
);

CREATE INDEX idx_supplier_requests_order ON supplier_requests(order_id);
CREATE INDEX idx_supplier_requests_supplier ON supplier_requests(supplier_id);
CREATE INDEX idx_supplier_requests_message_id ON supplier_requests(email_message_id);
CREATE INDEX idx_supplier_requests_status ON supplier_requests(status);
```

### supplier_quotes - Котировки поставщиков
```sql
CREATE TABLE supplier_quotes (
    id SERIAL PRIMARY KEY,
    quote_number VARCHAR(50) UNIQUE NOT NULL,  -- QUO-2024-0001
    supplier_request_id INTEGER REFERENCES supplier_requests(id),
    order_item_id INTEGER REFERENCES order_items(id),

    -- Коммерческие условия
    price DECIMAL(12,2) NOT NULL,
    currency VARCHAR(10) DEFAULT 'RUB',
    delivery_days INTEGER,
    payment_terms TEXT,
    additional_terms TEXT,

    -- Техническое решение
    technical_solution TEXT,
    alternative_offered BOOLEAN DEFAULT false,

    -- Статус
    is_selected BOOLEAN DEFAULT false,

    -- Метаданные
    created_at TIMESTAMP DEFAULT NOW(),
    parsed_by_ai BOOLEAN DEFAULT false
);

CREATE INDEX idx_supplier_quotes_request ON supplier_quotes(supplier_request_id);
CREATE INDEX idx_supplier_quotes_item ON supplier_quotes(order_item_id);
CREATE INDEX idx_supplier_quotes_selected ON supplier_quotes(is_selected);
```

### client_offers - КП клиентам
```sql
CREATE TABLE client_offers (
    id SERIAL PRIMARY KEY,
    offer_number VARCHAR(50) UNIQUE NOT NULL,  -- OFF-2024-0001
    order_id INTEGER REFERENCES orders(id),

    -- Содержание
    total_amount DECIMAL(12,2),
    margin_percent DECIMAL(5,2),
    delivery_days INTEGER,
    payment_terms TEXT,
    validity_days INTEGER DEFAULT 7,

    -- Файлы
    pdf_file_path VARCHAR(500),
    pdf_file_url VARCHAR(500),

    -- Статус
    status VARCHAR(50) DEFAULT 'draft',
        -- draft, sent, accepted, rejected
    sent_at TIMESTAMP,

    created_at TIMESTAMP DEFAULT NOW(),
    created_by INTEGER REFERENCES users(id)
);

CREATE INDEX idx_client_offers_order ON client_offers(order_id);
CREATE INDEX idx_client_offers_status ON client_offers(status);
```

### supplier_comments - Комментарии о поставщиках
```sql
CREATE TABLE supplier_comments (
    id SERIAL PRIMARY KEY,
    supplier_id INTEGER REFERENCES suppliers(id) ON DELETE CASCADE,
    user_id INTEGER REFERENCES users(id),

    comment TEXT NOT NULL,

    created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_supplier_comments_supplier ON supplier_comments(supplier_id);
```

### audit_log - Журнал аудита
```sql
CREATE TABLE audit_log (
    id SERIAL PRIMARY KEY,
    entity_type VARCHAR(50) NOT NULL,
        -- orders, suppliers, clients, etc.
    entity_id INTEGER NOT NULL,
    action VARCHAR(50) NOT NULL,
        -- create, update, delete

    old_values JSONB,
    new_values JSONB,

    user_id INTEGER REFERENCES users(id),
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_audit_log_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_log_user ON audit_log(user_id);
CREATE INDEX idx_audit_log_created_at ON audit_log(created_at DESC);
```

## Связи и кардинальность

```
users (1) ----< (M) orders
users (1) ----< (M) clients
users (1) ----< (M) suppliers

clients (1) ----< (M) orders
orders (1) ----< (M) order_items
orders (1) ----< (M) supplier_requests
orders (1) ----< (M) client_offers

suppliers (1) ----< (M) supplier_requests
suppliers (1) ----< (M) supplier_comments

supplier_requests (1) ----< (M) supplier_quotes
order_items (1) ----< (M) supplier_quotes

client_offers (1) --- (1) orders
```

## Пример сквозной истории

### Заявка ORD-2024-0052

```sql
-- 1. Заявка
SELECT * FROM orders WHERE order_number = 'ORD-2024-0052';
-- Результат: id=52, client_id=15, status='quoted'

-- 2. Клиент
SELECT * FROM clients WHERE id = 15;
-- Результат: ООО "Стройка", ИНН: 1234567890

-- 3. Позиции
SELECT * FROM order_items WHERE order_id = 52;
-- Результат:
--   #1: Вентилятор ВР-300, 2 шт
--   #2: Фильтр F7, 10 шт

-- 4. Запросы к поставщикам
SELECT sr.*, s.company_name
FROM supplier_requests sr
JOIN suppliers s ON sr.supplier_id = s.id
WHERE sr.order_id = 52;
-- Результат:
--   REQ-2024-0143 → ООО "Вент" (отправлено, ответ получен)
--   REQ-2024-0144 → ИП Сидоров (отправлено, ответ получен)
--   REQ-2024-0145 → Вентех (отправлено, timeout)

-- 5. Котировки
SELECT sq.*, sr.request_number, s.company_name
FROM supplier_quotes sq
JOIN supplier_requests sr ON sq.supplier_request_id = sr.id
JOIN suppliers s ON sr.supplier_id = s.id
WHERE sr.order_id = 52;
-- Результат:
--   QUO-2024-0287: ООО "Вент", 15000₽, 5 дней, ВЫБРАНО
--   QUO-2024-0288: ИП Сидоров, 14500₽, 7 дней

-- 6. КП клиенту
SELECT * FROM client_offers WHERE order_id = 52;
-- Результат:
--   OFF-2024-0089: 18000₽ (маржа 20%), отправлено

-- 7. История изменений
SELECT * FROM audit_log
WHERE entity_type = 'orders' AND entity_id = 52
ORDER BY created_at;
-- Результат:
--   2024-02-06 10:00 - create (user: admin)
--   2024-02-06 10:15 - update status: new → processing
--   2024-02-06 15:30 - update status: processing → quoted
```

## Триггеры и автоматика

### Автоматическое обновление updated_at
```sql
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER update_orders_updated_at
    BEFORE UPDATE ON orders
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();

-- Аналогично для clients, suppliers
```

### Аудит логирование
```sql
CREATE OR REPLACE FUNCTION log_audit()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO audit_log (entity_type, entity_id, action, old_values, new_values, user_id)
    VALUES (
        TG_TABLE_NAME,
        COALESCE(NEW.id, OLD.id),
        TG_OP,
        row_to_json(OLD),
        row_to_json(NEW),
        current_setting('app.current_user_id', TRUE)::INTEGER
    );
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER audit_orders
    AFTER INSERT OR UPDATE OR DELETE ON orders
    FOR EACH ROW
    EXECUTE FUNCTION log_audit();

-- Аналогично для других критичных таблиц
```

## Оптимизация

### Партиционирование (для будущего)
```sql
-- Партиционирование заявок по месяцам
CREATE TABLE orders_2024_02 PARTITION OF orders
    FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');

CREATE TABLE orders_2024_03 PARTITION OF orders
    FOR VALUES FROM ('2024-03-01') TO ('2024-04-01');
```

### Vacuum и analyze
```sql
-- Автоматическая очистка и обновление статистики
ALTER TABLE orders SET (autovacuum_vacuum_scale_factor = 0.1);
ALTER TABLE audit_log SET (autovacuum_vacuum_scale_factor = 0.05);
```

## Backup стратегия

### Ежедневный бэкап
```bash
# Через pg_dump
pg_dump -U postgres -h localhost start_db > backup_$(date +%Y%m%d).sql

# Через Supabase Dashboard
# Settings → Database → Create backup
```

### Point-in-time recovery
```sql
-- Supabase автоматически сохраняет WAL
-- Можно восстановить на любой момент за последние 7 дней
```

## Миграции (Alembic)

### Создание миграции
```bash
# Автогенерация
alembic revision --autogenerate -m "Add suppliers table"

# Ручная миграция
alembic revision -m "Add index on orders.status"
```

### Применение
```bash
# Все миграции
alembic upgrade head

# Откат
alembic downgrade -1
```

## Seed данные (для разработки)

```python
# backend/database/seed.py

def seed_data():
    # Тестовые пользователи
    admin = User(telegram_id=123456789, username="admin", role="admin")

    # Тестовые поставщики
    supplier1 = Supplier(
        company_name="ООО Вент",
        inn="7700123456",
        email="vent@mail.ru",
        categories=["вентиляция"]
    )

    # Тестовые клиенты
    client1 = Client(
        name="ООО Стройка",
        inn="1234567890",
        email="stroyka@mail.ru"
    )

    db.session.add_all([admin, supplier1, client1])
    db.session.commit()
```

---

**Важно**: Схема оптимизирована для небольшого объема данных (< 10k заявок). При росте нужно будет добавить партиционирование и оптимизировать индексы.

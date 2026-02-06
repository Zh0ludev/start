# Бизнес-процессы и примеры кода

## Workflow 1: Прием заявки по email

### Шаги

1. Email приходит на mail.ru
2. IMAP polling обнаруживает письмо
3. AI парсит заявку
4. Создается заявка в БД
5. Уведомление в Telegram

### Код

```python
# services/email_service.py

async def check_new_emails():
    """Проверка новых писем"""
    mail = imaplib.IMAP4_SSL(settings.imap_server)
    mail.login(settings.email_username, settings.email_password)
    mail.select('INBOX')

    # Непрочитанные письма
    status, messages = mail.search(None, 'UNSEEN')

    for num in messages[0].split():
        status, data = mail.fetch(num, '(RFC822)')
        email_msg = email.message_from_bytes(data[0][1])

        # Парсинг
        parsed = parse_email(email_msg)

        # Проверка - это заявка от клиента или ответ поставщика?
        if is_supplier_reply(parsed):
            await process_supplier_reply(parsed)
        else:
            await process_client_order(parsed)

    mail.close()
    mail.logout()


async def process_client_order(email_data):
    """Обработка заявки от клиента"""

    # AI парсинг
    try:
        parsed_data = await ai_parser.parse_order_email(
            email_body=email_data['body']
        )
    except ParsingError as e:
        # Fallback - уведомление о ручной обработке
        await notification_service.send_unparsed_email_alert(email_data)
        return

    # Создание/поиск клиента
    client = await get_or_create_client(
        name=parsed_data['client_name'],
        inn=parsed_data['inn'],
        email=email_data['from']
    )

    # Создание заявки
    order = Order(
        order_number=generate_order_number(),
        client_id=client.id,
        source_email_id=email_data['message_id'],
        source_email_from=email_data['from'],
        source_email_subject=email_data['subject'],
        source_email_body=email_data['body'],
        source_email_date=email_data['date'],
        status='new',
        specification=parsed_data['specification']
    )
    db.add(order)

    # Создание позиций
    for idx, item in enumerate(parsed_data['items'], 1):
        order_item = OrderItem(
            order_id=order.id,
            item_number=idx,
            product_name=item['name'],
            quantity=item.get('quantity'),
            unit=item.get('unit'),
            specifications=item.get('specifications')
        )
        db.add(order_item)

    db.commit()

    # Аудит
    log_audit('orders', order.id, 'create', new_values=order_to_dict(order))

    # Уведомление
    await bot.send_message(
        chat_id=settings.telegram_admin_ids[0],
        text=f"📧 Новая заявка #{order.id}\n"
             f"Номер: {order.order_number}\n"
             f"От: {client.name}\n"
             f"Товары: {len(parsed_data['items'])} поз.\n\n"
             f"[Открыть в веб](/orders/{order.id})",
        reply_markup=get_order_actions_keyboard(order.id)
    )
```

## Workflow 2: Рассылка запросов поставщикам

### Шаги

1. Менеджер выбирает поставщиков
2. Генерируется шаблон письма
3. Отправка SMTP
4. Сохранение запроса в БД

### Код

```python
# services/email_service.py

async def send_supplier_requests(order_id: int, supplier_ids: list[int]):
    """Рассылка запросов выбранным поставщикам"""

    order = db.query(Order).get(order_id)
    suppliers = db.query(Supplier).filter(Supplier.id.in_(supplier_ids)).all()

    for supplier in suppliers:
        # Генерация уникального Message-ID для трекинга
        message_id = f"<req-{order.order_number}-{supplier.id}@yourdomain.ru>"

        # Создание письма
        msg = MIMEMultipart()
        msg['From'] = settings.email_username
        msg['To'] = supplier.email
        msg['Subject'] = f"Запрос КП {order.order_number}"
        msg['Message-ID'] = message_id

        # Тело письма из шаблона
        body = render_template('supplier_request.txt',
            order=order,
            supplier=supplier,
            items=order.items
        )
        msg.attach(MIMEText(body, 'plain', 'utf-8'))

        # Отправка
        try:
            server = smtplib.SMTP_SSL(settings.smtp_server, 465)
            server.login(settings.email_username, settings.email_password)
            server.send_message(msg)
            server.quit()

            # Сохранение запроса
            request = SupplierRequest(
                request_number=generate_request_number(),
                order_id=order.id,
                supplier_id=supplier.id,
                email_subject=msg['Subject'],
                email_body=body,
                email_message_id=message_id,
                email_sent_at=datetime.now(),
                status='sent'
            )
            db.add(request)
            db.commit()

            # Обновление статистики поставщика
            supplier.total_requests += 1
            db.commit()

            logger.info("supplier_request_sent",
                request_number=request.request_number,
                supplier_id=supplier.id,
                order_id=order.id
            )

        except Exception as e:
            logger.error("supplier_request_failed",
                supplier_id=supplier.id,
                error=str(e)
            )
            # Уведомление в Telegram об ошибке
            await notification_service.send_email_error_alert(supplier, str(e))
```

## Workflow 3: Прием ответов поставщиков

### Шаги

1. Поставщик отвечает на письмо
2. IMAP обнаруживает ответ
3. Поиск исходного запроса
4. AI парсит ответ
5. Создается котировка

### Код

```python
# services/email_service.py

async def process_supplier_reply(email_data):
    """Обработка ответа поставщика"""

    # Находим исходный запрос по In-Reply-To
    original_message_id = email_data.get('in_reply_to')
    if not original_message_id:
        logger.warning("supplier_reply_no_thread",
            from_email=email_data['from']
        )
        return

    request = db.query(SupplierRequest).filter(
        SupplierRequest.email_message_id == original_message_id
    ).first()

    if not request:
        logger.warning("supplier_reply_no_match",
            message_id=original_message_id
        )
        return

    # Обновление запроса
    request.status = 'replied'
    request.reply_received_at = datetime.now()
    request.reply_email_id = email_data['message_id']
    request.reply_email_body = email_data['body']
    db.commit()

    # AI парсинг ответа
    try:
        quote_data = await ai_parser.parse_supplier_quote(
            email_body=email_data['body']
        )
    except ParsingError as e:
        # Fallback - сохраняем исходное письмо
        await notification_service.send_unparsed_quote_alert(request, email_data)
        return

    # Создание котировки
    quote = SupplierQuote(
        quote_number=generate_quote_number(),
        supplier_request_id=request.id,
        order_item_id=request.order.items[0].id,  # Упрощение для MVP
        price=quote_data['price'],
        delivery_days=quote_data.get('delivery_days'),
        payment_terms=quote_data.get('payment_terms'),
        technical_solution=quote_data.get('technical_solution'),
        parsed_by_ai=True
    )
    db.add(quote)
    db.commit()

    # Обновление статистики поставщика
    supplier = request.supplier
    supplier.total_quotes += 1
    response_time = (request.reply_received_at - request.email_sent_at).total_seconds() / 3600
    if supplier.avg_response_time_hours:
        supplier.avg_response_time_hours = (supplier.avg_response_time_hours + response_time) / 2
    else:
        supplier.avg_response_time_hours = response_time
    db.commit()

    # Уведомление
    await bot.send_message(
        chat_id=settings.telegram_admin_ids[0],
        text=f"📨 Ответ от {supplier.company_name}\n"
             f"Заявка: {request.order.order_number}\n"
             f"Цена: {quote.price:,.0f}₽\n"
             f"Срок: {quote.delivery_days} дн.\n\n"
             f"[Открыть сравнение](/orders/{request.order_id}/quotes)",
        reply_markup=get_quote_actions_keyboard(quote.id)
    )
```

## Workflow 4: Генерация КП

### Шаги

1. Менеджер выбирает лучшую котировку
2. Устанавливает маржу
3. Генерируется PDF
4. Отправка клиенту

### Код

```python
# services/pdf_generator.py

from weasyprint import HTML
from jinja2 import Template

def generate_offer_pdf(order, selected_quotes, margin_percent):
    """Генерация PDF коммерческого предложения"""

    # Расчет итоговой суммы
    total_cost = sum(q.price for q in selected_quotes)
    total_amount = total_cost * (1 + margin_percent / 100)

    # Загрузка шаблона
    with open('templates/offer_template.html') as f:
        template = Template(f.read())

    # Рендеринг HTML
    html_content = template.render(
        order=order,
        client=order.client,
        items=order.items,
        quotes=selected_quotes,
        total_amount=total_amount,
        margin_percent=margin_percent,
        date=datetime.now().strftime('%d.%m.%Y')
    )

    # Генерация PDF
    pdf_bytes = HTML(string=html_content).write_pdf()

    # Сохранение в Cloudflare R2
    file_name = f"OFF-{order.order_number}.pdf"
    file_path = upload_to_r2(file_name, pdf_bytes)

    # Создание записи о КП
    offer = ClientOffer(
        offer_number=generate_offer_number(),
        order_id=order.id,
        total_amount=total_amount,
        margin_percent=margin_percent,
        delivery_days=max(q.delivery_days for q in selected_quotes),
        pdf_file_path=file_path,
        pdf_file_url=get_r2_url(file_path),
        status='draft'
    )
    db.add(offer)

    # Обновление выбранных котировок
    for quote in selected_quotes:
        quote.is_selected = True

    # Обновление статуса заявки
    order.status = 'quoted'

    db.commit()

    return offer


# API endpoint
@app.post("/api/orders/{order_id}/offers")
async def create_offer(
    order_id: int,
    quote_ids: list[int],
    margin_percent: float
):
    """Создание КП"""

    order = db.query(Order).get(order_id)
    quotes = db.query(SupplierQuote).filter(
        SupplierQuote.id.in_(quote_ids)
    ).all()

    offer = generate_offer_pdf(order, quotes, margin_percent)

    # Отправка КП клиенту
    await send_offer_to_client(offer)

    return {"offer_number": offer.offer_number, "pdf_url": offer.pdf_file_url}


async def send_offer_to_client(offer):
    """Отправка КП клиенту по email"""

    order = offer.order
    client = order.client

    # Создание письма
    msg = MIMEMultipart()
    msg['From'] = settings.email_username
    msg['To'] = client.email
    msg['Subject'] = f"Коммерческое предложение {offer.offer_number}"

    # Тело письма
    body = f"""
Добрый день!

Направляем коммерческое предложение по вашему запросу.

Сумма: {offer.total_amount:,.2f}₽
Срок поставки: {offer.delivery_days} дней
Условия оплаты: {offer.payment_terms}

КП действительно {offer.validity_days} дней с момента выставления.

С уважением,
ООО "Ваша компания"
"""
    msg.attach(MIMEText(body, 'plain', 'utf-8'))

    # Прикрепление PDF
    pdf_data = download_from_r2(offer.pdf_file_path)
    pdf_attachment = MIMEApplication(pdf_data, _subtype='pdf')
    pdf_attachment.add_header('Content-Disposition', 'attachment',
                             filename=f"{offer.offer_number}.pdf")
    msg.attach(pdf_attachment)

    # Отправка
    server = smtplib.SMTP_SSL(settings.smtp_server, 465)
    server.login(settings.email_username, settings.email_password)
    server.send_message(msg)
    server.quit()

    # Обновление статуса
    offer.status = 'sent'
    offer.sent_at = datetime.now()
    db.commit()

    logger.info("offer_sent",
        offer_number=offer.offer_number,
        client_id=client.id
    )
```

## Утилиты

### Генерация номеров

```python
# utils/numbering.py

def generate_order_number():
    """Генерация номера заявки: ORD-2024-0001"""
    year = datetime.now().year
    last = db.query(Order).filter(
        Order.order_number.like(f'ORD-{year}-%')
    ).order_by(Order.id.desc()).first()

    num = int(last.order_number.split('-')[-1]) + 1 if last else 1
    return f"ORD-{year}-{num:04d}"

# Аналогично для REQ-, QUO-, OFF-
```

### AI Промпты

```python
# services/ai_parser.py

ORDER_PARSING_PROMPT = """
Извлеки из email заявки следующие данные:

1. Название компании клиента
2. ИНН (если есть)
3. Контактное лицо
4. Email
5. Телефон
6. Список товаров с количеством

Верни JSON:
{
  "client_name": "...",
  "inn": "...",
  "contact_person": "...",
  "email": "...",
  "phone": "...",
  "items": [
    {"name": "...", "quantity": ..., "unit": "..."}
  ],
  "specification": "общее описание заявки"
}

Email:
---
{email_body}
---
"""

QUOTE_PARSING_PROMPT = """
Извлеки из ответа поставщика:

1. Цену (только число)
2. Срок поставки (в днях)
3. Условия оплаты

Верни JSON:
{
  "price": ...,
  "delivery_days": ...,
  "payment_terms": "..."
}

Ответ:
---
{email_body}
---
"""
```

---

**Примечание**: Эти примеры кода показывают основную логику. В реальной реализации нужно добавить error handling, logging, и тесты.

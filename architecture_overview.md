# architecture_overview.md
# Архитектура новой CRM-системы

> Версия: 1.0 | Дата: 2026-02-25

## 1. Введение и цели

Документ описывает архитектуру новой CRM-системы для управления многомиллионными базами телефонных номеров.

### Основные требования:
- Работа с многомиллионными базами номеров
- Управление проектами сбора лидов (поставщики, гео, дни недели)
- Конверсионная аналитика по воронке статусов
- SFTP-обмен с партнёрами
- Интеграция с SoftPhone
- Роли: Администратор и Менеджер
- Аудит-логирование
- Локальный запуск (Docker Compose) + перенос на сервер/облако

---

## 2. Анализ текущей CRM

### Основные разделы (crm.it-datamaster.ru):
- **CRM (лиды)**: список контактов с атрибутами
- **Мои проекты**: управление проектами, тегами, лимитами
- **Конверсия проектов**: таблица по статусам воронки
- **Удалённые проекты**: soft delete с сохранением данных
- **Импорт CSV**: загрузка номеров
- **Телефония**: журнал звонков
- **Дашборд**: сводная статистика

### Воронка статусов:
```
База → Недозвон → Переговоры → Ожидаем оплаты → Партнёрка → 
Оплачено → Закрыто и не реализовано → Тест драйв → Горячий → 
На замену → Конечный недозвон
```

---

## 3. Архитектура новой системы

```
Frontend (React 18 + TypeScript + Ant Design)
    ↓ REST API (JSON, JWT)
Backend (Python 3.12 + FastAPI)
    ├─ Auth & Roles
    ├─ Projects CRUD
    ├─ Numbers/Contacts (import, dedup, filter)
    ├─ Conversion Analytics
    ├─ SFTP Integration
    ├─ SoftPhone Webhook
    └─ Audit Log
    ↓
PostgreSQL 15+ (партиционирование, pg_trgm)
Redis 7 (Celery очереди для фоновых задач)
```

---

## 4. Выбранный стек

| Компонент | Технология | Обоснование |
|---|---|---|
| Backend | Python 3.12 + FastAPI | Async, автогенерация OpenAPI, быстрая разработка |
| ORM | SQLAlchemy 2.x (async) | Гибкость, сложные запросы, Alembic миграции |
| БД | PostgreSQL 15+ | Партиционирование, JSONB, надёжность |
| Очереди | Celery + Redis | Фоновый импорт миллионов номеров |
| Frontend | React 18 + TypeScript | Зрелая экосистема, React Query |
| UI | Ant Design 5 | Готовые Table, Filter, Form компоненты |
| Auth | JWT (access + refresh) | Stateless, масштабируемость |
| Deploy | Docker Compose | Локальный запуск + перенос на prod |

---

## 5. Основные сущности

```
User (пользователь)
 ├─ role: admin | manager
 └─ assigned_contacts[]

Supplier (поставщик)
 └─ projects[]

Project (проект)
 ├─ supplier_id
 ├─ tag, name, status, type
 ├─ daily_limit
 ├─ work_days (Mon-Sun)
 ├─ geo_filter (regions)
 └─ deleted_at (soft delete)

Contact/Number (лид)
 ├─ phone (unique)
 ├─ project_id, manager_id
 ├─ status (воронка)
 ├─ region, company, email
 ├─ comment, reminder
 └─ is_duplicate

ConversionStatus (справочник воронки)
Region (регионы + UTC)
Tag (теги, M2M)
Call (звонки из SoftPhone)
SftpJob (задачи SFTP)
AuditLog (аудит)
```

---

## 6. Роли и права

| Операция | Администратор | Менеджер |
|---|---|---|
| Просмотр всех контактов | ✅ | только свои |
| Создание проекта | ✅ | ✅ (ограничено) |
| Редактирование проекта | ✅ | только свои |
| Удаление проекта | ✅ | ❌ |
| Импорт/экспорт номеров | ✅ | только свои проекты |
| Просмотр аудит-лога | ✅ | ❌ |
| Управление пользователями | ✅ | ❌ |
| Настройки системы | ✅ | ❌ |

---

## 7. Интеграции

### SFTP Module
- Импорт/экспорт файлов с номерами
- Расписание (cron) или ручной запуск
- Логирование ошибок, статистика обработки
- Поддержка CSV/TXT форматов

### SoftPhone Integration
- Webhook endpoint для событий звонков
- Привязка звонка к Contact/Project
- Журнал: duration, missed, charged_amount
- Расширяемая архитектура (JSONB payload)

### Audit Log
- Логирование: CREATE/UPDATE/DELETE/LOGIN/EXPORT/IMPORT
- Хранение: user_id, action, entity_type, entity_id, diff (JSONB), ip_address
- Доступ: только Администратор
- Retention: настраиваемый период хранения

---

## 8. Масштабируемость

### Для больших объёмов данных:
- **Партиционирование таблицы contacts** по created_at (по месяцам)
- **Индексы**: phone (unique), project_id + status, region_id
- **Celery workers** для фоновых задач (импорт, экспорт, дедупликация)
- **Pagination** всех API (cursor-based для больших списков)
- **pg_trgm** для полнотекстового поиска по company_name, email

### Локальный запуск:
```yaml
# docker-compose.yml
services:
  api:
    build: ./backend
    ports: ["8000:8000"]
  frontend:
    build: ./frontend
    ports: ["3000:3000"]
  postgres:
    image: postgres:15
    volumes: [postgres_data:/var/lib/postgresql/data]
  redis:
    image: redis:7
  celery:
    build: ./backend
    command: celery -A app.worker worker -l info
```

---

## 9. Использование ИИ-помощников

Разработчики могут использовать ИИ (ChatGPT, Claude, GitHub Copilot) для:
- Генерации типового кода (CRUD endpoints, Pydantic models)
- Генерации SQL-миграций (Alembic)
- Написания unit/integration тестов
- Генерации документации/комментариев

**Обязательное правило**: весь код от ИИ проходит ручное ревью и тестирование.

---

## 10. Следующие шаги

1. Изучить `backend_plan_dev1.md` (Backend разработчик)
2. Изучить `frontend_plan_dev2.md` (Frontend разработчик)
3. Изучить `api_contracts.md` (API контракты для интеграции)

---

**Автор**: Системный архитектор CRM  
**Дата**: 2026-02-25  
**Статус**: Ready for development

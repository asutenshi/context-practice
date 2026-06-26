# Техническое задание: Evidence Locker (MVP)

Данный документ представляет собой техническое задание и план реализации модуля Evidence Locker (хранилища свидетельств). Проект спроектирован для реализации командой из 3 человек (Тимлид + 2 разработчика) за 4-дневный спринт.

## 1. Общая Архитектура
* **Стек:** Python 3.12, FastAPI (веб-фреймворк), SQLAlchemy 2.0 (ORM), **SQLite** (БД), Docker (с использованием **uv**).
* **Аутентификация:** Простая аутентификация через токены в файле `.env`. На MVP закладываем три токена:
  - `TEACHER_TOKEN` — дает права на смену статусов и привязку компетенций.
  - `COLLECTOR_TOKEN` — дает права на отправку новых данных от систем.
  - `STUDENT_TOKEN` — дает права на просмотр только своих подтверждений (на будущее).

## 2. Схема Базы Данных (Спроектирована Тимлидом)

В базе требуются минимум 2 основные таблицы:

**Таблица `evidence_records`** (Основное хранилище)
- `id` (UUID, Primary Key)
- `actor_id` (String) — кто совершил действие (парсится из `actor` xAPI)
- `source_system` (String) — откуда пришло
- `source_type` (String) — тип артефакта (парсится из `object.definition.type` xAPI)
- `evidence_link` (String) — URL на артефакт (парсится из `object.id` xAPI)
- `context` (String) — название проекта/репозитория (парсится из `context` xAPI)
- `timestamp` (DateTime) — время события. Берется из xAPI `timestamp`.
- `review_status` (String/Enum) — `draft` | `pending` | `reviewed` | `rejected`
- `reviewed_by` (String) — id преподавателя. *На время MVP, пока нет системы выдачи персональных токенов, бэкенд будет всегда записывать сюда `0` при успешном approve.*
- `created_at` (DateTime) — когда запись попала в базу

**Таблица `evidence_competencies`** (Связь Many-to-Many или One-to-Many)
- `id` (UUID, PK)
- `evidence_id` (UUID, Foreign Key)
- `competency_id` (String) — внешний ID компетенции
- `proposed_by` (String) — *Поле нужно для аналитики. Компетенция может быть предложена автоматически ботом-сборщиком ("collector"), либо добавлена вручную преподавателем при ревью ("teacher").*

---

## 3. Распределение задач (на 4 дня)

### 3.1. Тимлид (Архитектура, Среда и Демонстрация)
Отвечает за инфраструктуру и концептуальное проектирование. Код ядра не пишет.

**Задачи:**
1. Настроить репозиторий и контейнеризацию: `Dockerfile` (с использованием `uv` для сверхбыстрой сборки) и `docker-compose.yml` (только для FastAPI сервиса + маунт папки для SQLite).
2. Задать структуру таблиц БД в документации (передать Разработчикам 1 и 2).
3. Создать `.env.example` и заложить туда 3 токена (`TEACHER_TOKEN`, `COLLECTOR_TOKEN`, `STUDENT_TOKEN`).
4. **Написание Демо-Сборщика (Client):** Скрипт `scripts/demo_git_collector.py`, который локально читает историю Git и шлет строгие xAPI POST-запросы в API для защиты проекта.

### 3.2. Разработчик 1 (Data Ingestion & Auth)
Отвечает за то, чтобы данные корректно входили в систему, а также за защиту эндпоинтов.

**Задачи:**
1. Написать ORM-модели (SQLAlchemy) по схеме, которую дал тимлид.
2. **Реализовать функцию проверки API-токена** (Dependency Injection для FastAPI) для ограничения доступа.
3. **`POST /api/v1/evidences`**
   - *Input:* JSON со **строгим соблюдением стандарта xAPI Statement**. 
     Пример ожидаемой структуры:
     `{"actor": {"account": {"name": "student_1"}}, "verb": {"id": "http://adlnet.gov/expapi/verbs/completed"}, "object": {"id": "http://github.com/.../commit/...", "definition": {"type": "commit"}}, "context": {...}, "timestamp": "2026-06-26T12:00:00Z"}`
   - *Логика:* Строгая валидация Pydantic-модели `xAPI Statement`. Парсинг вложенных полей в плоскую структуру таблицы `evidence_records`. Создание записи со статусом `pending`.
   - *Output:* HTTP 201 Created. Логирование события `evidence.created`.

### 3.3. Разработчик 2 (Business Workflow, Search & Relations)
Отвечает за выдачу данных, изменение статусов и связи.

**Задачи:**
1. **`GET /api/v1/evidences`** (Поиск и Фильтрация)
   - *Input:* Query-параметры: `?actor_id=...` (по учащемуся), `?competency_id=...` (по компетенции), `?context=...` (по проекту), `?review_status=...` (по статусу).
   - *Логика:* Динамическое построение SQL-запроса.
   - *Output:* Список карточек подтверждений.
2. **`PATCH /api/v1/evidences/{id}/review`** (Смена статуса)
   - *Input:* JSON `{"status": "reviewed", "note": "..."}`. (ID преподавателя передавать не нужно). Требует `TEACHER_TOKEN`.
   - *Логика:* Проверка бизнес-правил. Запись `0` в поле `reviewed_by`. 
   - *Output:* HTTP 200 OK. Вызов внутреннего события-лога `evidence.reviewed` или `evidence.rejected`.
3. **`POST /api/v1/evidences/{id}/competencies`**
   - *Input:* JSON `{"competency_id": "teamwork", "proposed_by": "teacher"}`
   - *Логика:* Создание связи в таблице `evidence_competencies`.
   - *Output:* HTTP 201 Created. Логирование события `evidence.linked`.

---

## 4. Механизм логирования событий (Event Emission)

Куда попадают события `evidence.created`, `evidence.reviewed` и другие, которые требует спецификация?

Для MVP-версии в архитектуре не предусмотрен брокер сообщений (RabbitMQ/Kafka). Поэтому логирование этих машинно-читаемых событий будет осуществляться в **stdout (консоль контейнера)** с помощью стандартного модуля `logging` (уровень INFO). 
Они будут выводиться в формате JSON-строки (JSON Lines). 

**Зачем это нужно:**
При таком подходе соседние модули (например, Progressor или LRS) смогут читать эти логи через механизмы Docker (`docker logs -f evidence_locker`) или системы сбора логов (например, Fluentd/Vector), не нарушая слабую связанность модулей.

---

## 5. Verification Plan (Критерии успеха для MVP)

Для проверки готовности системы будет проведен следующий сценарий:
1. Запуск `docker-compose up`. Поднятие API. База SQLite создастся автоматически как файл.
2. Тимлид запускает `demo_git_collector.py` с `COLLECTOR_TOKEN`. Скрипт шлет валидный xAPI Statement.
3. Делается тестовый запрос `GET /api/v1/evidences?review_status=pending` (работа Разработчика 2). API должно вернуть список.
4. Выполняется `PATCH /api/v1/evidences/{id}/review` (работа Разработчика 2) с `TEACHER_TOKEN`, устанавливающий статус `reviewed`.
5. Повторный `GET` запрос показывает изменения, `reviewed_by` установлен в `0`. В консоли докера (`docker logs`) видны сгенерированные события в JSON-формате.

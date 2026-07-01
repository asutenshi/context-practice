# Задачи (Issues) для разработчиков

В этом файле собраны задачи для команды разработки Evidence Locker. Каждую задачу можно скопировать из блока с кодом и перенести в таск-трекер (например, GitHub Issues).

## Разработчик 1 (Backend Ingestion & Auth)

### Issue 1: SQLAlchemy модели и настройка БД
````markdown
```markdown
## Описание
Необходимо реализовать слой хранения данных: создать подключение к базе данных SQLite и описать таблицы через ORM (SQLAlchemy 2.0). 
*Важное архитектурное решение:* Мы используем xAPI Profile для Evidence Locker. Это означает, что помимо базовых полей xAPI, мы также извлекаем ключевые бизнес-поля (источник, тип, проект) из расширений и сохраняем их в выделенные колонки БД, не теряя при этом исходный JSON в поле `raw_data`.

## Задачи
- [ ] Добавить настройку подключения к БД SQLite в `app/db/database.py`.
- [ ] Описать модель `evidence_records` со всеми полями (actor_id, verb_id, object_id, raw_data, review_status и т.д.).
- [ ] Описать модель `evidence_competencies` (включая поля `status`, `reviewed_by`, `created_at`, `updated_at`) и связь с `evidence_records`.
- [ ] Убедиться, что при запуске приложения таблицы создаются автоматически, если их нет.

## Критерии приема
- Модели SQLAlchemy 2.0 созданы и соответствуют ER-диаграмме из технического задания.
- БД создается в папке `data/evidence.db`.
- Типы данных корректны, связи между таблицами работают.

**Labels:** `role: backend-ingestion`, `type: feature`, `status: todo`
```
````

### Issue 2: Pydantic модели для xAPI Statement
````markdown
```markdown
## Описание
Для строгой валидации входящих данных нужно описать контракты данных (Pydantic схемы), которые будут использоваться во всем приложении для xAPI Statement с учетом расширений профиля Evidence Locker (например, `source_system` и `source_type`).

## Задачи
- [ ] Создать структуру Pydantic моделей в `app/schemas/`.
- [ ] Описать поля входящего xAPI Statement (Actor, Verb, Object, Context).
- [ ] **Добавить логику извлечения бизнес-полей**, таких как `source_system` и `source_type` (из extensions), и `context_id` (из context).
- [ ] **Добавить логику нормализации объекта `actor`**, чтобы извлекать из различных форматов (`account`, `mbox` и т.д.) единую строку `actor_id` для записи в БД.
- [ ] Описать модели ответов (Response Models) для вывода информации о свидетельствах.
- [ ] Добавить строгую валидацию полей (проверка длины, допустимых форматов).

## Критерии приема
- Модели наследуются от `BaseModel` (Pydantic).
- Некорректный JSON вызывает ошибку валидации при использовании схемы.

**Labels:** `role: backend-ingestion`, `type: feature`, `status: todo`
```
````

### Issue 3: Скрипт-сидер для генерации тестовых данных
````markdown
```markdown
## Описание
Необходимо создать вспомогательный скрипт `scripts/seed.py`, который будет наполнять базу данных тестовыми записями для проверки. Это необходимо для того, чтобы Разработчик 2 мог тестировать фильтрацию и ревью-статусы.

## Задачи
- [ ] Создать файл `scripts/seed.py`.
- [ ] Подключить созданные Pydantic и SQLAlchemy модели.
- [ ] Сгенерировать 10-15 тестовых записей свидетельств с разными статусами (`pending`, `reviewed`, `rejected`).
- [ ] Реализовать сохранение этих записей в SQLite.

## Критерии приема
- При запуске `python scripts/seed.py` (или через Makefile) БД наполняется валидными данными.
- Скрипт использует текущие Pydantic-модели для гарантии правильной структуры данных.

**Labels:** `role: backend-ingestion`, `type: infrastructure`, `status: todo`
```
````

### Issue 6: Эндпоинт приема данных (POST /api/v1/evidences)
````markdown
```markdown
## Описание
Основной эндпоинт системы. Должен принимать xAPI Statement и строго валидировать его. 
Входящий JSON должен обязательно содержать поля: `id` (UUID), `actor` (объект с идентификатором пользователя, например mbox или account), `verb.id`, `object.id` и `timestamp`. 
Также коллекторы обязаны передавать через механизм расширений (extensions) бизнес-поля: `source_system` и `source_type`. Идентификатор `context_id` опционально извлекается из поля `context`, а поле `note` (короткое текстовое пояснение) также извлекается как опциональное.
В БД запись сохраняется со статусом `pending`.

## Задачи
- [ ] Добавить роутер `api/v1/ingestion.py`.
- [ ] Реализовать эндпоинт `POST /api/v1/evidences`.
- [ ] Настроить строгую валидацию тела запроса через Pydantic-модели (из Issue 2).
- [ ] Извлекать `actor_id` (путем нормализации объекта actor), `source_system`, `source_type` (из расширений), а также опциональные `context_id` и `note`. Сохранять их в соответствующие колонки таблицы `evidence_records`.
- [ ] Сохранять весь исходный JSON Statement в колонку `raw_data`.
- [ ] Устанавливать статус записи `pending`.
- [ ] Добавить логирование события `evidence.created` в `stdout` (JSON Lines).

## Критерии приема
- Эндпоинт возвращает 201 Created при успешном сохранении.
- Невалидный xAPI Statement (отсутствие обязательных xAPI-полей или расширений Evidence Locker) возвращает 422 Unprocessable Entity.
- При успешном запросе в консоли (stdout) появляется JSON строка с логом `evidence.created`.
- Данные корректно раскладываются по колонкам БД.

**Labels:** `role: backend-ingestion`, `type: feature`, `status: todo`
```
````

### Issue 7: Система авторизации по статичным токенам (Dependency Injection)
````markdown
```markdown
## Описание
Реализовать базовую проверку прав доступа на основе статичных токенов, заданных в файле `.env` (TEACHER_TOKEN, COLLECTOR_TOKEN, STUDENT_TOKEN). Доступ к эндпоинтам должен быть защищен соответствующими зависимостями FastAPI (Depends).

## Задачи
- [ ] Добавить логику загрузки токенов из переменных окружения.
- [ ] Создать файл `app/api/dependencies.py`.
- [ ] Реализовать функции-зависимости (dependencies) для проверки наличия правильного токена в заголовке `Authorization`:
  - `verify_collector_token` (для отправки данных)
  - `verify_teacher_token` (для изменения статусов и ручной привязки компетенций)
  - `verify_student_token` (для будущего эндпоинта просмотра своих результатов)
- [ ] *Дополнительно:* Настроить функцию `verify_teacher_or_collector_token`, которая пропускает либо Teacher, либо Collector, и возвращает роль вызвавшего (например, строку `"teacher"` или `"collector"`), чтобы использовать эту информацию в эндпоинте привязки компетенций.
- [ ] Подключить зависимости к соответствующим роутерам/эндпоинтам.

## Критерии приема
- Запрос без токена или с неверным токеном возвращает 401 Unauthorized или 403 Forbidden.
- Токены корректно разграничивают доступ (Коллектор не может вызывать эндпоинт смены статуса `review` и т.д.).

**Labels:** `role: backend-ingestion`, `type: security`, `status: todo`
```
````

### Issue 10: Отсутствует строгая валидация обязательных расширений xAPI
````markdown
```markdown
## Описание
В файле `app/schemas/evidence.py` при извлечении бизнес-полей из `context.extensions` используются значения по умолчанию (например, `extensions.get("source_system", "unknown_system")`). Согласно Issue 6, коллекторы обязаны передавать `source_system` и `source_type`. Это означает, что если этих полей нет, должна срабатывать ошибка валидации (422 Unprocessable Entity), а не подставляться дефолтное значение.

## Задачи
- [ ] Изменить логику в `extract_business_fields` в `app/schemas/evidence.py`.
- [ ] Если `source_system` или `source_type` отсутствуют в расширениях, выбрасывать `ValueError`, чтобы Pydantic возвращал ошибку 422.

**Labels:** `role: backend-ingestion`, `type: bug`, `status: todo`, `priority: medium`
```
````

### Issue 11: Покрытие тестами (Backend Ingestion & Auth)
````markdown
```markdown
## Описание
Проект требует минимального покрытия тестами, чтобы убедиться в корректности сбора данных и авторизации. На данный момент тесты полностью отсутствуют.

## Задачи
- [ ] Написать тесты для проверки Pydantic моделей (валидный/невалидный xAPI Statement).
- [ ] Написать API-тесты (используя `TestClient` из `fastapi.testclient`) для эндпоинта `POST /api/v1/evidences`.
- [ ] Написать тесты для проверки зависимостей авторизации (`verify_collector_token` и `verify_teacher_token`).

**Labels:** `role: backend-ingestion`, `type: feature`, `status: todo`, `priority: medium`
```
````

## Разработчик 2 (Backend Workflow, Search & Relations)

### Issue 4: Эндпоинт GET /api/v1/evidences с фильтрацией
````markdown
```markdown
## Описание
Реализовать API-метод для получения списка свидетельств. Метод должен поддерживать фильтрацию (по статусу, ID пользователя, компетенции), чтобы преподаватели и другие системы могли искать нужные записи.

**Блокировки / Зависимости:**
Для локального тестирования этого эндпоинта необходимо предварительно запустить скрипт-сидер (`scripts/seed.py` от Разработчика 1), чтобы в БД были тестовые данные.

## Задачи
- [ ] Добавить роутер `api/v1/workflow.py`.
- [ ] Создать эндпоинт `GET /api/v1/evidences`.
- [ ] Реализовать фильтрацию через параметры запроса (query parameters) средствами SQLAlchemy.
- [ ] Возвращать список валидированных через Pydantic объектов.

## Критерии приема
- Эндпоинт корректно возвращает данные из базы.
- Работает фильтрация (например, `?review_status=pending`).
- Формат ответа соответствует Pydantic схеме.

**Labels:** `role: backend-workflow`, `type: feature`, `status: todo`
```
````

### Issue 5: Эндпоинт PATCH /api/v1/evidences/{id}/review
````markdown
```markdown
## Описание
Реализовать метод изменения статуса свидетельства (например, с `pending` на `reviewed`). Метод должен менять статус в БД и логировать событие в стандартный вывод (`stdout`) для отправки в соседние микросервисы.

**Блокировки / Зависимости:**
Для локального тестирования необходимо предварительно запустить скрипт-сидер (`scripts/seed.py` от Разработчика 1).

## Задачи
- [ ] Создать эндпоинт `PATCH /api/v1/evidences/{id}/review` в `api/v1/workflow.py`.
- [ ] Реализовать логику обновления поля `review_status` в базе данных.
- [ ] Добавить вывод информации в `stdout` в формате JSON (событие `evidence.reviewed`).
- [ ] Обрабатывать ошибки (например, если ID не существует, возвращать 404).

## Критерии приема
- При успешном запросе статус в БД меняется.
- В логах (консоли) появляется запись о событии в формате JSON.
- Эндпоинт возвращает HTTP 200 OK.

**Labels:** `role: backend-workflow`, `type: feature`, `status: todo`
```
````

### Issue 8: Эндпоинт привязки компетенций (POST /api/v1/evidences/{id}/competencies)
````markdown
```markdown
## Описание
Эндпоинт для связи конкретного свидетельства с определенной компетенцией. Эта связь может быть инициирована как преподавателем вручную при ревью, так и коллектором автоматически. Данные сохраняются в таблицу `evidence_competencies`.

## Задачи
- [ ] Создать эндпоинт `POST /api/v1/evidences/{id}/competencies` в роутере `api/v1/workflow.py`.
- [ ] Эндпоинт должен принимать JSON с полем `competency_id` (строка).
- [ ] Эндпоинт должен быть доступен по двум токенам (Teacher и Collector). Использовать зависимость, которая возвращает роль пользователя, сделавшего запрос.
- [ ] Если связь создает Collector, то поле `proposed_by` устанавливается в `"collector"`, а `status` связи — `pending`.
- [ ] Если связь создает Teacher, то поле `proposed_by` устанавливается в `"teacher"`, `status` связи — `approved`, а `reviewed_by` — `"0"`.
- [ ] Добавить логирование события `evidence.linked` в `stdout` (JSON Lines).

## Критерии приема
- Эндпоинт разрешает доступ с токеном Collector или Teacher (и возвращает ошибку в других случаях).
- Привязка от Collector создает запись со статусом `pending`.
- Привязка от Teacher создает запись со статусом `approved`.
- В базе успешно появляется запись о связи свидетельства и компетенции.
- Эндпоинт возвращает HTTP 201 Created.
- В `stdout` выводится лог `evidence.linked`.

**Labels:** `role: backend-workflow`, `type: feature`, `status: todo`
```
````

### Issue 9: Отсутствует авторизация в эндпоинтах GET /evidences и PATCH /review
````markdown
```markdown
## Описание
Часть критически важных эндпоинтов в файле `app/api/v1/workflow.py` не защищена токенами.
- `GET /api/v1/evidences` (получение списка свидетельств) доступен без токена, хотя по паспорту проекта запрос должен выполняться с `TEACHER_TOKEN` (или другими токенами в зависимости от роли).
- `PATCH /api/v1/evidences/{evidence_id}/review` (изменение статуса) не защищен, любой пользователь может изменить статус ревью. Согласно заданию, он должен быть защищен `verify_teacher_token`.

## Задачи
- [ ] Добавить зависимость для проверки прав (например, `Depends(verify_teacher_token)`) в параметры эндпоинта `get_evidences`.
- [ ] Добавить `Depends(verify_teacher_token)` в параметры эндпоинта `review_evidence`.

**Labels:** `role: backend-workflow`, `type: feature`, `status: todo`, `priority: high`
```
````

### Issue 12: Покрытие тестами (Backend Workflow)
````markdown
```markdown
## Описание
Необходимо обеспечить минимальное тестовое покрытие для бизнес-логики (ревью и поиск), чтобы гарантировать, что фильтрация и изменение статусов работают корректно.

## Задачи
- [ ] Написать API-тесты (с использованием `TestClient`) для эндпоинта `GET /api/v1/evidences`, проверив базовую фильтрацию.
- [ ] Написать API-тесты для эндпоинта `PATCH /api/v1/evidences/{evidence_id}/review`, убедившись, что статус меняется только при правильном токене.
- [ ] Написать тесты для эндпоинта `POST /api/v1/evidences/{evidence_id}/competencies` (привязка компетенций с ролями Teacher и Collector).

**Labels:** `role: backend-workflow`, `type: feature`, `status: todo`, `priority: medium`
```
````

## Тимлид (Инфраструктура и Демонстрация)

### Issue 13: Пустой скрипт demo_git_collector.py
````markdown
```markdown
## Описание
Файл `scripts/demo_git_collector.py` создан, но внутри него нет кода. Этот скрипт необходим для полевых испытаний (отправка валидного POST-запроса с xAPI Statement в систему).

## Задачи
- [ ] Написать скрипт `scripts/demo_git_collector.py`.
- [ ] Скрипт должен формировать валидный JSON (xAPI Statement с расширениями `source_system` и `source_type`).
- [ ] Скрипт должен читать `COLLECTOR_TOKEN` из переменных окружения и делать POST-запрос на `/api/v1/evidences`.

**Labels:** `role: teamlead`, `type: feature`, `status: todo`, `priority: high`
```
````

### Issue 14: Отсутствует Makefile
````markdown
```markdown
## Описание
В корне проекта отсутствует `Makefile`, который требуется по техническому заданию для стандартизации команд запуска и тестирования.

## Задачи
- [ ] Создать `Makefile` в корне проекта.
- [ ] Добавить команды:
  - `install`: `uv sync` (установка зависимостей из pyproject.toml и uv.lock)
  - `test`: `uv run pytest`
  - `run`: `uv run uvicorn app.main:app --reload --port 8000`
  - `run-demo`: `uv run python scripts/demo_git_collector.py`
  - `docker-build`: `docker-compose build`
  - `docker-up`: `docker-compose up -d`

**Labels:** `role: teamlead`, `type: infrastructure`, `status: todo`, `priority: high`
```
````

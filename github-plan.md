# План оформления Git-репозитория Evidence-Locker

Данный документ описывает структуру и правила оформления Git-репозитория для модуля Evidence Locker в рамках 4-дневного спринта, согласно техническому заданию и контексту практики.

## 1. Архитектура папок (Folder Structure)

Основываясь на стеке (FastAPI, SQLite, Docker, `uv`), принята следующая структура:

```text
evidence-locker/
├── app/                        # Основной код приложения (FastAPI)
│   ├── api/                    # Маршрутизаторы (endpoints)
│   │   ├── dependencies.py     # Инъекции зависимостей (проверка токенов)
│   │   └── v1/
│   │       ├── ingestion.py    # Эндпоинты для сбора (POST /evidences)
│   │       └── workflow.py     # Эндпоинты для ревью и поиска (GET, PATCH, POST competencies)
│   ├── core/                   # Ядро (настройки, логирование)
│   ├── db/                     # Работа с базой данных SQLite
│   │   ├── database.py         # Подключение к SQLite
│   │   └── models.py           # SQLAlchemy модели (evidence_records, evidence_competencies)
│   ├── schemas/                # Pydantic модели (xAPI Statement и др.)
│   └── main.py                 # Точка входа FastAPI приложения
├── scripts/                    # Вспомогательные скрипты
│   └── demo_git_collector.py   # Демо-сборщик из тех. задания
├── data/                       # Папка для локальной SQLite базы (mount в Docker)
├── .env.example                # Шаблон переменных окружения (TEACHER_TOKEN, etc.)
├── .gitignore                  # Исключения для Git (venv, .env, data/*.sqlite)
├── Dockerfile                  # Инструкции для сборки образа (с использованием uv)
├── docker-compose.yml          # Compose-файл для поднятия API и БД
├── pyproject.toml              # Управление зависимостями через uv
├── README.md                   # Главный документ репозитория
└── CONTRIBUTING.md             # Правила командной разработки
```

## 2. Структура README.md

В `README.md` должны быть включены следующие разделы:
1. **Название и Описание проекта:** Концепция Evidence Locker как легковесного хранилища событий xAPI.
2. **Архитектура и Стек:** Краткое описание стека (Python 3.12, FastAPI, SQLite, uv).
3. **Запуск проекта (Quick Start):** Шаги по запуску через `docker-compose up` и настройке `.env`.
4. **API Endpoints:** Краткая справка по реализованным маршрутам.
5. **Процесс логирования:** Пояснение, что события пишутся в `stdout` (JSON Lines) для других модулей (Skill-matrix и др.).
6. **Паспорт модуля:** Ссылка на файл `паспорт_модуля.md` (на этапе старта будет создана заглушка "В разработке").

## 3. Структура CONTRIBUTING.md

Поскольку над проектом работает 3 человека, `CONTRIBUTING.md` должен фиксировать следующие правила:
1. **Workflow (Git Flow):** Использование Feature-веток (например, `feature/1-models-setup`), создание Pull Requests, правила ревью.
2. **Формат коммитов (подсказка):** Строгое использование Conventional Commits: `feat:` (новый функционал), `fix:` (багфикс), `docs:` (документация), `chore:` (настройки/инфраструктура). Пример: `feat: add evidence creation endpoint`.
3. **Ежедневная отчетность:** Напоминание об обязательном ведении `.md` файлов по задачам SMART со ссылками на PR.
4. **Установка и работа с `uv`:** Краткая инструкция по установке пакетов и быстрому управлению зависимостями (например, `uv pip install -r pyproject.toml`).
5. **Линтинг и форматирование:** Требование использовать `ruff` для проверки и форматирования кода перед коммитом (`ruff check .`, `ruff format .`).
6. **Запуск контейнера для проверки:** Инструкция для разработчиков по локальному тестированию (`docker-compose up --build`) и просмотру логов через `docker logs`.

## 4. Labels для задач (Issues) без milestones

Система меток для организации задач:

* **Roles (Роли):**
  * `role: teamlead` — задачи Тимлида.
  * `role: backend-ingestion` — задачи разработчика, отвечающего за сбор данных и Auth.
  * `role: backend-workflow` — задачи разработчика, отвечающего за бизнес-логику ревью, связи и поиск.
* **Types (Типы задач):**
  * `type: feature` — новый функционал.
  * `type: bug` — ошибка.
  * `type: docs` — документация (паспорт, README, дневники).
  * `type: infrastructure` — Docker, uv, настройки окружения.
* **Status (Статус):**
  * `status: todo`, `status: in-progress`, `status: in-review`, `status: done`.
* **Priority (Приоритет):**
  * `priority: high` — высокий приоритет (критично для MVP).
  * `priority: medium` — средний приоритет.
  * `priority: low` — низкий приоритет (можно отложить).

## 5. Issues относительно Технического Задания

Задачи должны быть созданы в трекере и разбиты строго по ролям из ТЗ.

**Тимлид:**
* **Issue 1:** Настроить репозиторий, `pyproject.toml` (uv) и `.gitignore`.
* **Issue 2:** Создать `Dockerfile` и `docker-compose.yml`.
* **Issue 3:** Настроить `.env.example` с токенами `TEACHER_TOKEN`, `COLLECTOR_TOKEN`, `STUDENT_TOKEN`.
* **Issue 4:** Описать структуру таблиц БД в issue/документации.
* **Issue 5:** Написать демо-сборщик `scripts/demo_git_collector.py`.
* **Issue 6:** Подготовить паспорт модуля (файл-заглушку `паспорт_модуля.md`).

**Разработчик 1 (Backend Ingestion & Auth):**
* **Issue 7:** Написать SQLAlchemy модели (`evidence_records`, `evidence_competencies`).
* **Issue 8:** Реализовать Dependency Injection для валидации API-токенов.
* **Issue 9:** Написать Pydantic модели для строгой валидации xAPI Statement с учетом профиля Evidence Locker (извлечение `source_system`, `source_type`).
* **Issue 10:** Реализовать эндпоинт `POST /api/v1/evidences` в `api/v1/ingestion.py`.
  * *Подзадача:* Добавить логирование события `evidence.created` в `stdout` (JSON-строка).

**Разработчик 2 (Backend Workflow, Search & Relations):**
* **Issue 11:** Реализовать эндпоинт `GET /api/v1/evidences` с фильтрацией (динамический SQL) в `api/v1/workflow.py`.
* **Issue 12:** Реализовать эндпоинт `PATCH /api/v1/evidences/{id}/review` и логику смены статуса.
  * *Подзадача:* Добавить логирование события `evidence.reviewed` (или `evidence.rejected`) в `stdout` (JSON-строка).
* **Issue 13:** Реализовать эндпоинт `POST /api/v1/evidences/{id}/competencies` для связи с компетенциями.
  * *Подзадача:* Добавить логирование события `evidence.linked` в `stdout` (JSON-строка).

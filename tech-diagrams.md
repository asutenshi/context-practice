# Визуализация работы Evidence-Locker

Ниже представлены Mermaid-диаграммы, которые помогут визуально понять архитектуру, структуру базы данных и процессы (workflow) модуля **Evidence Locker**, основываясь на техническом задании и контексте практики.

## 1. Архитектура и интеграция модулей (Context & Architecture)

Система строится на принципе слабой связанности (Loose coupling). Данная диаграмма показывает место Evidence Locker среди других компонентов "Учебной рабочей среды".

```mermaid
graph TD
    subgraph "Источники данных (Сборщики)"
        GC["Demo Git Collector<br/>(scripts/demo_git_collector.py)"]
        P[Progressor]
        T[Transcriptor-Tracker]
    end

    subgraph "Evidence Locker (MVP)"
        API[FastAPI Backend]
        DB[(SQLite БД)]
        STDOUT["stdout / JSON Logs<br/>Event Emission"]
    end

    subgraph "Потребители данных"
        SM[Skill-matrix]
        TEA["Teacher / Frontend<br/>(Преподаватель)"]
    end

    %% Data Flow
    GC -- "POST /api/v1/evidences\n(xAPI Statement + COLLECTOR_TOKEN)" --> API
    P -. "xAPI Statement" .-> API
    T -. "xAPI Statement" .-> API

    API -- "Сохранение/Чтение" --> DB
    API -- "Логирование событий\n(evidence.created, etc.)" --> STDOUT

    STDOUT -- "docker logs / Fluentd" --> SM
    SM -- "GET /api/v1/evidences\n(Опциональное получение данных)" --> API
    
    TEA -- "GET /api/v1/evidences\nPATCH .../review\n(TEACHER_TOKEN)" --> API
```

> [!NOTE]
> В MVP версии брокер сообщений не используется. События (`evidence.created`, `evidence.reviewed`) пишутся в **stdout**, откуда их могут забирать другие модули (например, Skill-matrix).

---

## 2. Схема базы данных (ER-диаграмма)

Диаграмма "Entity-Relationship" для основных таблиц, спроектированных Тимлидом.

```mermaid
erDiagram
    evidence_records {
        UUID id PK
        String actor_id "Кто совершил действие"
        String source_system "Откуда пришло"
        String source_type "Тип артефакта"
        String evidence_link "URL на артефакт"
        String context "Название проекта/репозитория"
        DateTime timestamp "Время события (xAPI)"
        String review_status "draft | pending | reviewed | rejected"
        String reviewed_by "ID преподавателя (0 в MVP)"
        DateTime created_at "Когда запись попала в базу"
    }

    evidence_competencies {
        UUID id PK
        UUID evidence_id FK
        String competency_id "Внешний ID компетенции"
        String proposed_by "teacher или collector"
    }

    evidence_records ||--o{ evidence_competencies : "имеет компетенции"
```

---

## 3. Процесс верификации свидетельства (Verification Workflow)

Sequence-диаграмма (Диаграмма последовательности), описывающая путь свидетельства от момента генерации до проверки преподавателем.

```mermaid
sequenceDiagram
    actor Collector as "Модуль-сборщик<br/>(напр. Git Collector)"
    participant API as "Evidence Locker API<br/>(FastAPI)"
    participant DB as "База Данных<br/>(SQLite)"
    actor Teacher as Преподаватель
    participant Logs as "Stdout (Логи)"

    %% Создание
    Collector->>API: POST /api/v1/evidences (xAPI JSON)
    activate API
    API->>API: Валидация xAPI Statement
    API->>DB: INSERT в `evidence_records`<br/>(status: 'pending')
    API->>Logs: INFO: evidence.created (JSONL)
    API-->>Collector: 201 Created
    deactivate API

    %% Просмотр преподавателем
    Teacher->>API: GET /api/v1/evidences?review_status=pending
    activate API
    API->>DB: SELECT *
    DB-->>API: Данные
    API-->>Teacher: Список подтверждений (JSON)
    deactivate API

    %% Проверка и привязка компетенций
    Teacher->>API: POST /api/v1/evidences/{id}/competencies
    activate API
    API->>DB: INSERT в `evidence_competencies`
    API->>Logs: INFO: evidence.linked
    API-->>Teacher: 201 Created
    deactivate API

    %% Смена статуса
    Teacher->>API: PATCH /api/v1/evidences/{id}/review<br/>{"status": "reviewed"}
    activate API
    API->>DB: UPDATE `evidence_records`<br/>SET status='reviewed', reviewed_by='0'
    API->>Logs: INFO: evidence.reviewed
    API-->>Teacher: 200 OK
    deactivate API
```

> [!TIP]
> Токены аутентификации (`COLLECTOR_TOKEN` и `TEACHER_TOKEN`) передаются в заголовках или параметрах запросов (через Dependency Injection) для ограничения прав доступа на каждом этапе (Developer 1).

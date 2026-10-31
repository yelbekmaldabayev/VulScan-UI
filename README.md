# Технический план реализации: система выявления IDOR, SQLi и XSS в веб-приложениях

Документ состоит из частей A–G (структура задана в ТЗ, п. 16). Все имена классов, топиков, таблиц и полей согласованы между частями; при расхождении источником истины является Часть B (контракты).

---

# Часть A. Анализ предоставленных материалов

## A.1. Изученные файлы

| #   | Файл                      | Что это                                                                                                                                                           | Назначение                   |
| --- | ------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------- |
| 1   | `attachment-2a606cd0.txt` | HTML-страница «XSS Scanner: frontend, backend and database design» (SVG-диаграмма архитектуры, таблица API, три макета экранов, схема сканирования, ER-диаграмма) | Дизайн-концепция XSS-сканера |
| 2   | `attachment.txt`          | Промпт-ТЗ на разработку плана (Русский язык)                                                                                                                      | Требования к этому плану     |

**Важное ограничение.** Файл 1 — это дизайн-документ (разметка и скрипт, рисующий диаграммы), а не исходный код XSS-модуля. Классов, методов, тестов и конфигурации сборки в нём нет. Репозитория с кодом среди материалов нет. Поэтому:

- реализацию XSS-модуля нельзя «изучить» — можно изучить только идеи дизайна;
- числа на макетах (14 open findings, 11 min и т. п.) — иллюстрации, не результаты измерений;
- существующий код не изменялся и не предполагался.

## A.2. Что берём из примера как идею

| Идея из файла 1                                                                                                | Как используем                                                                         |
| -------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------- |
| Канарейка — уникальный короткий маркер на пару (запрос, параметр); по нему находим, где ввод оказался в ответе | `CanaryFactory` в XSS-модуле; `canary` связывается с параметром в памяти job (не в БД) |
| Классификация контекста отражения (HTML body / attribute / JS string / URL)                                    | `HtmlContextClassifier` + `JsStringLexer` в XSS-модуле                                 |
| Отражение ≠ уязвимость; статус «подозрение» до подтверждения                                                   | Шкала `verdict`: `SUSPECTED` → `LIKELY` → `CONFIRMED` (см. B.8.3 и этапы X-4/X-5)      |
| Ownership-проверка target перед сканированием; scope include/exclude; rate limit; audit log                    | `Target.status=VERIFIED`, `ScopePolicyService`, `scan_policies`, `audit_events`        |
| Дедупликация findings по параметру, типу и контексту                                                           | `fingerprint` (B.4)                                                                    |
| Библиотека безопасных проб, отключаемая без правки кода                                                        | `ProbeLibrary` (в MVP — конфигурируемый список в `application.yml`, не таблица БД)     |
| Жизненный цикл finding (new → confirmed/false positive → fixed)                                                | `FindingTriageStatus`: `OPEN`, `FALSE_POSITIVE`, `FIXED`                               |
| Экраны Dashboard / New scan / Finding detail                                                                   | Страницы frontend 7.2                                                                  |
| Общая схема таблиц project/target/scan/finding для трёх модулей                                                | Расширена до `Scan` → `ScanJob` → `Finding` с полем `module_type`                      |

## A.3. Что необходимо переработать

| В примере                                                                               | Проблема                                                        | Решение в этом плане                                                                                                                          |
| --------------------------------------------------------------------------------------- | --------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| Единый «API server + worker» на FastAPI/NestJS, Redis-очередь                           | Не соответствует обязательному стеку (Kotlin/Spring Boot/Kafka) | Core Backend + три отдельных Kotlin-сервиса, Kafka                                                                                            |
| Воркеры пишут в PostgreSQL и object storage напрямую («results, evidence rows»)         | Размывает источник истины; модуль получает доступ к БД          | Модули БД не видят; результаты только через Kafka, хранит Core                                                                                |
| Таблицы `endpoints`, `parameters`, `test_cases` на каждый запрос                        | Взрыв числа строк, не нужен для MVP                             | Endpoint inventory на уровне target (`target_endpoints`); proba-запросы не персистятся, в finding хранится только evidence                    |
| `ownership_verified` как bool                                                           | Нет способа верификации, нет аудита                             | `targets.status` + токен-файл/подтверждение ADMIN, `audit_events`                                                                             |
| «Confirmed in browser» как единственный способ подтверждения; Chromium в каждом воркере | Тяжёлая зависимость, вопросы изоляции                           | Браузер вынесен в опциональный этап (решение D-9, Часть G); MVP подтверждает инъекцию разбором DOM/JS-лексером и честно называет это `LIKELY` |
| DOM-XSS через статический анализ JS + runtime hooks                                     | Объём сопоставим со всей остальной системой                     | Вне MVP, фиксируется как ограничение                                                                                                          |
| WebSocket для live-прогресса                                                            | Дополнительный канал и безопасность                             | Polling через TanStack Query                                                                                                                  |
| Отчёты PDF/JSON/SARIF, Object storage                                                   | Не нужны для цели диплома                                       | Исключены; экспорт findings в JSON — только для evaluation                                                                                    |
| Роли admin/analyst/viewer                                                               | Избыточно                                                       | Две роли: `USER`, `ADMIN`                                                                                                                     |
| Scope guard описан одной строкой                                                        | Нет защиты от DNS rebinding, redirect, SSRF                     | Раздел 12 (Часть C, этап S-\*)                                                                                                                |

## A.4. Что не используем

- Redis/RQ/BullMQ, FastAPI/NestJS, S3-хранилище, SARIF-экспорт.
- Автоматическое «деструктивное» подтверждение (выполнение полезных нагрузок с побочным эффектом).
- Схему «один probe → одна строка БД».

## A.5. Требования, не покрытые материалами

1. Любые сведения о SQLi- и IDOR-модулях (в файле 1 они только упомянуты как «будущие»).
2. Kafka-контракты, идемпотентность, outbox, retry/DLQ.
3. Источник списка endpoint/параметров (в примере — crawler; здесь решение A-5 ниже).
4. Аутентификация пользователей системы и хранение секретов IDOR-идентичностей.
5. Метрики оценки качества, разметка ground truth.
6. Docker Compose-окружение и тестовые стенды.
7. Тесты — в файле 1 не упоминаются.

## A.6. Фиксируемые предположения

| ID   | Предположение                                                                                                                                                                                                                      | Последствие                                                                                    |
| ---- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------- |
| A-1  | Команда из трёх человек, срок — около одного года вместе с текстом диплома; MVP должен быть реализуем за ~6–7 месяцев, остальное — оценка и документация                                                                           | В Части D задан порядок «что вырезать первым»                                                  |
| A-2  | Все модули и Core пишутся на Kotlin/Spring Boot; общий код — одна Gradle-библиотека `scanner-kit` (не сервис)                                                                                                                      | Критичный для безопасности код (`SafeHttpClient`, редактор секретов, контракты) не дублируется |
| A-3  | Монорепозиторий с каталогами `frontend/`, `core-backend/`, `xss-module/`, `sqli-module/`, `idor-module/`, `libs/scanner-kit/`, `contracts/`, `infra/`, `testbeds/`, `evaluation/`                                                  | Единый Docker Compose и контрактные тесты                                                      |
| A-4  | Версии библиотек и образов фиксируются в `gradle/libs.versions.toml`, `package-lock.json`, `infra/.env` на этапе 0.3, после проверки совместимости Spring Boot ↔ JDK (LTS) ↔ Spring Kafka                                          | В плане версии не указываются намеренно                                                        |
| A-5  | Список endpoint и параметров хранит Core (`target_endpoints`): ручной ввод + импорт OpenAPI 3 (JSON). Crawler в MVP **не** реализуется                                                                                             | Модули получают снимок endpoint в сообщении задания; crawler — расширение (риск R-6)           |
| A-6  | Приложение запускается локально/в лаборатории; публичные цели — только с подтверждённым владением                                                                                                                                  | Private-сети разрешаются только через явный allowlist CIDR (B.5, 12.1)                         |
| A-7  | Аутентификация пользователей системы — серверная сессия (HttpOnly cookie + CSRF), а не JWT в localStorage                                                                                                                          | Frontend не хранит токены                                                                      |
| A-8  | IDOR-проверки в MVP только чтение (GET/HEAD); изменяющие проверки — этап повышенного риска, по умолчанию выключены                                                                                                                 | Раздел 11.2                                                                                    |
| A-9  | Одна PostgreSQL-база и одна схема, принадлежит Core; у модулей своих БД нет                                                                                                                                                        | Решение D-6                                                                                    |
| A-10 | Если официальная формулировка темы диплома делает акцент на конкретный регион, этот акцент выражается в данных (CWE-/OWASP-маппинг и рекомендации в finding, ограничения по законодательству в тексте диплома), а не в архитектуре | Не влияет на код                                                                               |

---

# Часть B. Архитектурные решения

## B.1. Итоговая архитектура

```mermaid
flowchart LR
  subgraph Browser
    FE[Frontend SPA<br/>React + TS]
  end
  subgraph bus-net
    K[(Kafka)]
  end
  subgraph db-net
    PG[(PostgreSQL)]
  end
  FE -- "HTTPS /api/v1 (cookie session + CSRF)" --> CORE[Core Backend<br/>Kotlin / Spring Boot]
  CORE <-- JDBC --> PG
  CORE -- "scan.jobs.{xss,sqli,idor}.v1" --> K
  K -- "scan.job-events.v1" --> CORE
  K --> XSS[XSS module]
  K --> SQLI[SQLi module]
  K --> IDOR[IDOR module]
  XSS & SQLI & IDOR -- "HTTP /internal/v1 (claim, heartbeat, credentials)" --> CORE
  XSS & SQLI & IDOR -- "HTTP(S) проб-запросы через SafeHttpClient" --> T[(Authorized target / lab-app)]
```

Принципы разделения:

- **Core Backend** — единственный владелец PostgreSQL, пользователей, targets, политик, заданий и findings. Не содержит алгоритмов обнаружения.
- **Detection module** — stateless-сервис: получил задание → выполнил проверки → опубликовал findings и итог. Не знает о пользователях, БД и других модулях.
- **Frontend** — только отображение и ввод; все проверки безопасности дублируются на сервере.
- **`scanner-kit`** — библиотека (не сервис): DTO контрактов, `SafeHttpClient`, `IpPolicy`, `Redactor`, рантайм воркера (`WorkerRuntime`). Подключается к Core и модулям.

## B.2. Ответственность пяти проектов

| Проект         | Отвечает за                                                                                                                                                                                            | Не отвечает за                                                           |
| -------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------ |
| `frontend`     | Login, управление target/endpoint/identity, создание сканов, dashboard, findings, безопасное отображение evidence                                                                                      | Валидацию scope «по-настоящему», хранение секретов                       |
| `core-backend` | Аутентификация/авторизация, target + верификация, endpoint inventory, policies, Scan/ScanJob state machine, outbox, приём результатов, findings, агрегация, аудит, watchdog, internal API для воркеров | Отправку проб на target (кроме проверки владения target), анализ ответов |
| `xss-module`   | Reflected XSS (+ stored XSS по заданным endpoint)                                                                                                                                                      | Хранение, пользователи, сравнение разных типов уязвимостей               |
| `sqli-module`  | Error-/boolean-/time-based признаки SQLi                                                                                                                                                               | То же                                                                    |
| `idor-module`  | Horizontal/vertical проверки на основе нескольких идентичностей и матрицы ожидаемых прав                                                                                                               | Определение владельца объекта «самостоятельно»                           |

## B.3. Каналы взаимодействия

| Операция                                          | Канал                                | Почему                                                                                                                       |
| ------------------------------------------------- | ------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------- |
| UI ↔ Core                                         | HTTP REST `/api/v1`, JSON            | Request/response, пользовательские действия                                                                                  |
| Постановка заданий                                | Kafka `scan.jobs.<module>.v1`        | Развязка по времени, масштабирование модулей                                                                                 |
| Результаты (finding, завершение, ошибка)          | Kafka `scan.job-events.v1`           | Ключ = `jobId` → порядок «findings → completed» внутри задания                                                               |
| Claim задания, heartbeat, отмена, выдача секретов | HTTP `/internal/v1/jobs/{jobId}/...` | Нужна синхронная compare-and-set семантика (кто владеет заданием), ответ с флагом отмены, секреты не должны попадать в Kafka |
| Dead letters                                      | Kafka `scan.dlq.v1`                  | Невалидные/необрабатываемые сообщения                                                                                        |

Отдельного топика отмены нет: отмена доставляется ответом на heartbeat (решение D-4).

## B.4. Источники истины

| Данные                                                   | Источник истины                                                                   |
| -------------------------------------------------------- | --------------------------------------------------------------------------------- |
| Пользователи, роли, сессии                               | PostgreSQL (Core)                                                                 |
| Target, статус верификации, endpoint inventory, policies | PostgreSQL (Core)                                                                 |
| Scan, ScanJob, статусы, attempt, lease                   | PostgreSQL (Core). Kafka — транспорт, не состояние                                |
| Findings и evidence                                      | PostgreSQL (Core)                                                                 |
| IDOR-секреты                                             | PostgreSQL (Core), зашифрованы AES-GCM                                            |
| Состояние выполнения проб (canary-карта, baseline)       | Память job-а в модуле; теряется при падении, задание перезапускается Core целиком |
| Ground truth для оценки                                  | `testbeds/*/ground-truth.yaml` в репозитории                                      |

## B.5. Решения, требующие обоснования

**D-1. Топики: отдельный на модуль, а не общий с маршрутизацией.**
Рекомендация: `scan.jobs.xss.v1`, `scan.jobs.sqli.v1`, `scan.jobs.idor.v1` (по 3 партиции для dev; ключ = `jobId`). Обоснование: неподходящий модуль физически не получает чужие задания; независимое масштабирование consumer group-ов; независимая задержка. Последствия: Core выбирает топик по `moduleType` в `OutboxEventFactory`; модуль дополнительно проверяет `payload.moduleType` и при несовпадении отправляет сообщение в DLQ.

**D-2. Результаты — один топик `scan.job-events.v1` с полем `type`.**
Типы: `FINDING_REPORTED`, `JOB_COMPLETED`, `JOB_FAILED`. Ключ `jobId` гарантирует порядок внутри задания. `JOB_COMPLETED` содержит `findingsCount`; Core сверяет с числом сохранённых findings и ставит `result_integrity = INCOMPLETE`, если счётчик не сошёлся (потерянный finding в DLQ).

**D-3. Идемпотентность и «ровно одно выполнение».**
Kafka — at-least-once, поэтому три уровня защиты:

1. Producer Core: транзакционный outbox + `enable.idempotence=true`, `acks=all`.
2. Claim: перед запуском модуль вызывает `POST /internal/v1/jobs/{jobId}/claim {attempt, workerId}`. Core выполняет `UPDATE scan_jobs SET status='RUNNING', worker_id=?, lease_expires_at=? WHERE id=? AND status='QUEUED' AND attempt=?`; `1` строка → 200, иначе 409 (уже захвачено/устарел `attempt`) или 410 (отменено). Дубликат сообщения не запускает второе выполнение.
3. Consumer Core: таблица `processed_messages(message_id)` + уникальность `findings(job_id, fingerprint)`; повторная доставка результата — no-op.
   Offset модуля коммитится **после** успешного claim; дальнейшая отказоустойчивость — на watchdog Core (lease), а не на Kafka-redelivery.

**D-4. Отмена и зависшие задания.**
Heartbeat каждые 10 с (`POST .../heartbeat {requestsSent, endpointsDone}`) продлевает lease на 45 с и возвращает `{"cancel": true|false}`. Модуль проверяет флаг между запросами и прекращает работу без публикации итогового события: Core сам переводит задание в `CANCELLED` в момент, когда отдаёт `cancel=true`. Findings, опубликованные до отмены, сохраняются. Зависшие: `StuckJobWatchdog` (см. Core, этап C-7) обрабатывает `RUNNING` с истёкшим lease, `RUNNING` с `deadline_at < now`, `QUEUED` дольше `queueTimeout`.

**D-5. Retry и DLQ.**

- Транзиентные ошибки Kafka-consumer-а (сбой HTTP claim, недоступность Core): `DefaultErrorHandler` с `FixedBackOff` (3 попытки по 2 с), затем `DeadLetterPublishingRecoverer` → `scan.dlq.v1`.
- Невалидное сообщение (не парсится, неверный `schemaVersion`/`moduleType`) → сразу DLQ, без retry.
- Retry **задания** — решение Core: `JOB_FAILED(retryable=true)`, потеря lease, `QUEUED`-таймаут → `attempt+1` и новое outbox-сообщение, если `attempt < maxAttempts` (`maxAttempts = 2`: исходный запуск + один повтор). `retryable=false` (ошибка конфигурации, scope violation) повторов не даёт.
- DLQ в MVP только накапливается и просматривается вручную (`kafka-console-consumer`); автоматического reprocessing нет.

**D-6. Одна БД и одна схема.** Модулям БД не нужна (stateless); Core — единственный владелец. Отдельные схемы не вводятся. Flyway-миграции только Core.

**D-7. Секреты IDOR.** Хранятся в `test_identities.secret_encrypted` (AES-256-GCM, ключ из переменной окружения `SCANNER_SECRET_KEY`, `key_version` для ротации). Через Kafka передаются только `identityId`. Модуль получает расшифрованные учётные данные вызовом `GET /internal/v1/jobs/{jobId}/identities` — разрешено только владельцу активного lease (`X-Worker-Id`), событие записывается в `audit_events`, ответ не логируется (`Cache-Control: no-store`).

**D-8. Масштабирование и ограничение параллелизма.**

- Модуль масштабируется добавлением инстансов в ту же consumer group (число партиций ≥ числа инстансов).
- Внутри инстанса — `maxConcurrentJobs` (по умолчанию 1): при отсутствии свободного слота listener вызывает `pause()` контейнера, claim не выполняется, backpressure остаётся в Kafka.
- На стороне Core: не более `maxActiveScansPerUser` (2) и `maxActiveScansGlobal` (4) в статусах `PENDING|RUNNING|CANCELLING`; превышение → `429 TOO_MANY_ACTIVE_SCANS`.

**D-9. Браузерное подтверждение XSS — вне MVP.** Причины: образ Chromium сотни МБ, ~сотни МБ ОЗУ на контекст, песочница Chromium требует либо seccomp-профиль, либо `--no-sandbox` (небезопасно без контейнерной изоляции). План: опциональный этап X-7 (флаг `moduleOptions.xss.browserVerification`, по умолчанию `false`), реализуется только если MVP готов к месяцу М5; метрики с браузером и без сравниваются в главе об оценке.

**D-10. Частичный отказ.** Scan не «падает» из-за одного модуля: итог вычисляется по таблице B.7; findings успешных модулей сохраняются и показываются всегда.

**D-11. Фронтенд: React, не Angular.**
Сравнение для данной архитектуры: Angular даёт встроенные DI, роутер, формы, HttpClient и строгую структуру — это сильные стороны для больших команд. Для данного проекта решающие факторы: (1) основной сценарий — серверное состояние с polling и фильтрами; TanStack Query решает кэш/refetch/инвалидацию меньшим объёмом кода, чем RxJS-цепочки; (2) типы из OpenAPI (`openapi-typescript`) одинаково работают в обоих, но в React не требуется кодогенерация сервисов; (3) меньше шаблонного кода на 8–9 страниц. Выбран **React + Vite + TypeScript**.

**D-12. Взаимодействие Core ↔ модуль для claim — HTTP, а не Kafka.** Требуется атомарная проверка владения заданием с немедленным ответом; через Kafka это потребовало бы request/reply-топика и таймаутов.

**D-13. Crawler вне MVP** (A-5): endpoint inventory задаёт пользователь; импорт OpenAPI 3 снижает ручной труд. Это ограничивает «автоматичность» сканера и фиксируется в Части G (R-6).

## B.6. Корреляция, логирование, наблюдаемость

- `correlationId` (UUID) создаётся Core при `POST /scans` (или берётся из заголовка `X-Correlation-Id`, если он UUID), сохраняется в `scans.correlation_id`, попадает в envelope каждого Kafka-сообщения и в HTTP-заголовок `X-Correlation-Id` internal-вызовов.
- Логи JSON (одна строка — одно событие); MDC-поля: `correlationId`, `scanId`, `jobId`, `module`, `attempt`. `Redactor` из `scanner-kit` применяется ко всем логируемым заголовкам/телам.
- Spring Actuator: `/actuator/health/liveness`, `/actuator/health/readiness`, метрики Micrometer (`scanner.jobs.started`, `scanner.jobs.failed`, `scanner.requests.sent`, `scanner.findings.reported`). Prometheus/Grafana не добавляются.

## B.7. Модель статусов (единая для всех проектов)

**ScanJob** — `PENDING → QUEUED → RUNNING → COMPLETED | FAILED | CANCELLED`

| Из                  | В           | Условие / кто выполняет                                                                   |
| ------------------- | ----------- | ----------------------------------------------------------------------------------------- |
| `PENDING`           | `QUEUED`    | `OutboxRelay` получил ack от Kafka                                                        |
| `PENDING`, `QUEUED` | `CANCELLED` | Отмена до старта                                                                          |
| `QUEUED`            | `RUNNING`   | Успешный claim (`attempt` совпадает)                                                      |
| `QUEUED`            | `FAILED`    | `queueTimeout` исчерпан и `attempt == maxAttempts` или DLQ-невалидное сообщение           |
| `QUEUED`, `RUNNING` | `QUEUED`    | Retry: `attempt+1`, новое outbox-сообщение                                                |
| `RUNNING`           | `COMPLETED` | `JOB_COMPLETED` (для текущего `attempt`)                                                  |
| `RUNNING`           | `FAILED`    | `JOB_FAILED(retryable=false)`; retryable при исчерпанных попытках; `deadline_at` превышен |
| `RUNNING`           | `CANCELLED` | Heartbeat после запроса отмены                                                            |

Терминальные: `COMPLETED`, `FAILED`, `CANCELLED`. Любой другой переход — `IllegalStateTransitionException`.

**Scan** — `PENDING, RUNNING, CANCELLING, COMPLETED, PARTIALLY_COMPLETED, FAILED, CANCELLED`.

| Состояние заданий                                              | Статус Scan           |
| -------------------------------------------------------------- | --------------------- |
| Все задания `PENDING`                                          | `PENDING`             |
| Есть не терминальные задания, отмена не запрошена              | `RUNNING`             |
| Отмена запрошена, есть не терминальные                         | `CANCELLING`          |
| Все терминальные, отмена запрошена                             | `CANCELLED`           |
| Все терминальные, нет `FAILED`, нет `CANCELLED`                | `COMPLETED`           |
| Все терминальные, есть `COMPLETED` и есть `FAILED`/`CANCELLED` | `PARTIALLY_COMPLETED` |
| Все терминальные, нет `COMPLETED`                              | `FAILED`              |

Вычисляется чистой функцией `ScanStatusAggregator.aggregate(jobs, cancelRequested)`, вызывается в той же транзакции, что и изменение статуса задания.

```mermaid
stateDiagram-v2
  [*] --> PENDING
  PENDING --> QUEUED: outbox ack
  PENDING --> CANCELLED
  QUEUED --> RUNNING: claim OK
  QUEUED --> CANCELLED
  QUEUED --> QUEUED: retry (attempt+1)
  QUEUED --> FAILED: queue timeout, no attempts left
  RUNNING --> QUEUED: retry (lease lost / retryable)
  RUNNING --> COMPLETED: JOB_COMPLETED
  RUNNING --> FAILED: non-retryable / deadline
  RUNNING --> CANCELLED: cancel via heartbeat
  COMPLETED --> [*]
  FAILED --> [*]
  CANCELLED --> [*]
```

## B.8. Контракты

Источник истины форматов:

- HTTP API Frontend ↔ Core: OpenAPI, генерируется из контроллеров (`springdoc`), снимок хранится в `contracts/openapi/core-api.v1.json`; TS-типы генерируются в `frontend/src/api/generated/` (`openapi-typescript`).
- Kafka и internal HTTP: JSON Schema в `contracts/schemas/*.json` + примеры в `contracts/examples/*.json`. Kotlin-DTO — в `libs/scanner-kit/.../contracts`. Контрактный тест проверяет, что каждый пример валиден по схеме **и** десериализуется в DTO, а DTO-сериализация валидна по схеме.

Общие правила: JSON UTF-8, `camelCase`, время — ISO-8601 UTC (`2026-10-22T15:04:05Z`), идентификаторы — UUID v4, неизвестные поля **игнорируются** потребителем (прямая совместимость), обязательные — нет.

### B.8.1. Kafka envelope (общий для всех сообщений)

| Поле            | Тип       | Обяз. | Правила                                                       |
| --------------- | --------- | ----- | ------------------------------------------------------------- |
| `schemaVersion` | int       | да    | сейчас `1`; неизвестная major-версия → DLQ                    |
| `messageId`     | UUID      | да    | уникален; ключ дедупликации (`processed_messages`)            |
| `type`          | string    | да    | `SCAN_JOB`, `FINDING_REPORTED`, `JOB_COMPLETED`, `JOB_FAILED` |
| `correlationId` | UUID      | да    | из `scans.correlation_id`                                     |
| `occurredAt`    | timestamp | да    |                                                               |
| `payload`       | object    | да    | зависит от `type`                                             |

Kafka-заголовки дублируют `messageId`, `correlationId`, `schemaVersion` (для DLQ-обработки без парсинга тела). Топики: ключ = `jobId`, значение — JSON, макс. размер сообщения — 1 МБ (валидируется Core при сборке задания: ≤ 200 endpoint, ≤ 20 параметров на endpoint).

Версионирование: суффикс `.v1` в имени топика + `schemaVersion`. Обратно-несовместимое изменение → новый топик `.v2` и параллельное чтение на переходный период; добавление опциональных полей версию не меняет.

### B.8.2. `SCAN_JOB` (Core → модуль), пример для XSS

```json
{
  "schemaVersion": 1,
  "messageId": "5b6c1f6e-0d2a-4a43-9d5e-1a8f2c3a7b10",
  "type": "SCAN_JOB",
  "correlationId": "c0a8f3d2-6a0e-4b7e-9d58-0c1f4f1e2a11",
  "occurredAt": "2026-10-22T15:04:05Z",
  "payload": {
    "jobId": "8f0d6d3e-9b52-4f55-8d0e-4f6a2d9e7c01",
    "scanId": "2c1e7a9a-3b18-4f6e-a4a0-6a9b8d1b5e22",
    "moduleType": "XSS",
    "attempt": 1,
    "deadlineAt": "2026-10-22T15:34:05Z",
    "policyId": "a3f1d4c2-1f21-4a0e-8d11-9b0c7e5d6f33",
    "target": {
      "targetId": "7d9b0f5e-5a35-4b6c-b0aa-3c3e6e9a1d44",
      "baseUrl": "http://lab-app:8080",
      "allowedHosts": [{ "host": "lab-app", "ports": [8080] }]
    },
    "scope": {
      "includePrefixes": ["http://lab-app:8080/"],
      "excludePrefixes": ["http://lab-app:8080/logout"]
    },
    "limits": {
      "maxRequests": 600,
      "requestsPerSecond": 5,
      "maxConcurrency": 2,
      "httpTimeoutMs": 10000,
      "maxResponseBytes": 1048576,
      "maxDurationSec": 1800
    },
    "endpoints": [
      {
        "endpointId": "e1d0c4aa-4d5b-4b9b-8a3a-0f1d2e3c4b55",
        "method": "GET",
        "url": "http://lab-app:8080/xss/search",
        "idempotent": true,
        "parameters": [
          { "name": "q", "location": "QUERY", "sampleValue": "test" }
        ]
      }
    ],
    "moduleOptions": {
      "xss": {
        "reflected": true,
        "stored": false,
        "browserVerification": false
      }
    }
  }
}
```

| Поле `payload`                                   | Тип                    | Обяз. | Валидация                                                                                                        |
| ------------------------------------------------ | ---------------------- | ----- | ---------------------------------------------------------------------------------------------------------------- |
| `jobId`, `scanId`, `policyId`, `target.targetId` | UUID                   | да    | —                                                                                                                |
| `moduleType`                                     | enum `XSS\|SQLI\|IDOR` | да    | должен совпадать с топиком и модулем                                                                             |
| `attempt`                                        | int ≥ 1                | да    | передаётся в claim                                                                                               |
| `deadlineAt`                                     | timestamp              | да    | позже `occurredAt`; модуль не стартует после дедлайна                                                            |
| `target.baseUrl`                                 | URL                    | да    | `http`/`https`, без userinfo                                                                                     |
| `target.allowedHosts[]`                          | `{host, ports[]}`      | да    | ≥ 1; модуль разрешает соединения только к этим host:port                                                         |
| `scope.includePrefixes[]`                        | URL-префиксы           | да    | URL запроса должен начинаться с одного из них и ни с одного из `excludePrefixes`                                 |
| `limits.*`                                       | int                    | да    | верхние границы заданы конфигурацией модуля; задание не может их превысить                                       |
| `endpoints[]`                                    | см. выше               | да    | `method` ∈ `GET,POST,PUT,PATCH,DELETE,HEAD`; `idempotent=false` исключает endpoint из SQLi boolean/time-проверок |
| `endpoints[].parameters[].location`              | enum                   | да    | `QUERY, BODY_FORM, BODY_JSON, PATH, HEADER, COOKIE`                                                              |
| `moduleOptions`                                  | object                 | да    | ровно одна ветка: `xss`, `sqli` или `idor` (по `moduleType`)                                                     |

`moduleOptions.sqli`: `{"errorBased":true,"booleanBased":true,"timeBased":false,"dbmsHint":null}` (`dbmsHint` ∈ `POSTGRESQL|MYSQL|MSSQL|ORACLE|SQLITE|null`).
`moduleOptions.idor` (Core собирает из таблиц `test_identities`, `resource_access_rules`):

```json
"idor": {
  "mode": "READ_ONLY",
  "identities": [
    {"identityId": "11111111-0000-4000-8000-000000000001", "label": "alice", "role": "USER"},
    {"identityId": "11111111-0000-4000-8000-000000000002", "label": "bob", "role": "USER"},
    {"identityId": "11111111-0000-4000-8000-000000000003", "label": "admin", "role": "ADMIN"}
  ],
  "resources": [
    {
      "resourceId": "22222222-0000-4000-8000-000000000001",
      "endpointId": "e2f0c4aa-4d5b-4b9b-8a3a-0f1d2e3c4b66",
      "idParameter": {"name": "id", "location": "PATH"},
      "idValue": "1001",
      "ownerIdentityId": "11111111-0000-4000-8000-000000000001",
      "sensitivity": "MEDIUM",
      "ownerMarkers": ["$.ownerEmail", "$.items[0].sku"],
      "access": [
        {"identityId": "11111111-0000-4000-8000-000000000002", "expected": "DENY"},
        {"identityId": "11111111-0000-4000-8000-000000000003", "expected": "ALLOW"}
      ]
    }
  ],
  "sessionCheck": {"endpointId": "e2f0c4aa-4d5b-4b9b-8a3a-0f1d2e3c4b77", "expectStatus": 200}
}
```

`expected` ∈ `ALLOW | DENY | PUBLIC`. Валидация Core: каждая identity, упомянутая в `access`/`ownerIdentityId`, существует и принадлежит этому target; `mode=READ_ONLY` ⇒ все `endpoints[].method ∈ {GET, HEAD}`; секретов в сообщении нет.

### B.8.3. `FINDING_REPORTED` (модуль → Core)

```json
{
  "schemaVersion": 1,
  "messageId": "0a0f3a2e-6d93-4c8f-b2b6-d1f7e0c4a901",
  "type": "FINDING_REPORTED",
  "correlationId": "c0a8f3d2-6a0e-4b7e-9d58-0c1f4f1e2a11",
  "occurredAt": "2026-10-22T15:05:11Z",
  "payload": {
    "jobId": "8f0d6d3e-9b52-4f55-8d0e-4f6a2d9e7c01",
    "attempt": 1,
    "finding": {
      "moduleType": "XSS",
      "vulnType": "XSS_REFLECTED",
      "cwe": "CWE-79",
      "endpointId": "e1d0c4aa-4d5b-4b9b-8a3a-0f1d2e3c4b55",
      "method": "GET",
      "url": "http://lab-app:8080/xss/search",
      "parameter": { "name": "q", "location": "QUERY" },
      "verdict": "LIKELY",
      "severity": "MEDIUM",
      "confidence": 78,
      "fingerprint": "e7c0f2b1c1a94c0a8c0c1b7f0a9d3b6f2d8e4a1c5b7f9e0d1a2b3c4d5e6f7a8b",
      "title": "Reflected XSS: parameter q (HTML text context)",
      "summary": "Значение параметра q попадает в HTML-текст без кодирования символов < и >; парсер распознал внедрённый элемент.",
      "recommendation": "Кодировать вывод для HTML-контекста (HTML entity encoding), использовать шаблонизатор с автоэкранированием; Content-Security-Policy как дополнительный слой.",
      "details": {
        "type": "XSS_REFLECTED",
        "context": "HTML_TEXT",
        "survivingChars": ["<", ">", "\"", "'"],
        "injectionProof": "DOM_STRUCTURE",
        "mitigations": []
      },
      "evidence": {
        "request": {
          "method": "GET",
          "url": "http://lab-app:8080/xss/search?q=vsx4k2m9qpa7%3Cvsxtag%20id%3Dvsx4k2m9qpa7%3E",
          "headers": { "Accept": "text/html" },
          "bodyExcerpt": null
        },
        "response": {
          "status": 200,
          "headers": { "Content-Type": "text/html;charset=UTF-8" },
          "bodyExcerpt": "<p>Results for vsx4k2m9qpa7<vsxtag id=vsx4k2m9qpa7></p>",
          "highlights": [{ "start": 13, "end": 25 }]
        },
        "observations": [
          { "key": "reflectionCount", "value": "1" },
          {
            "key": "domProof",
            "value": "element vsxtag[id=vsx4k2m9qpa7] present in parsed DOM"
          }
        ]
      }
    }
  }
}
```

| Поле finding                     | Тип                          | Обяз. | Правила                                                                                                                                                                                    |
| -------------------------------- | ---------------------------- | ----- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `vulnType`                       | enum                         | да    | `XSS_REFLECTED`, `XSS_STORED`, `SQLI_ERROR_BASED`, `SQLI_BOOLEAN_BASED`, `SQLI_TIME_BASED`, `IDOR_HORIZONTAL`, `IDOR_VERTICAL`                                                             |
| `verdict`                        | enum                         | да    | `SUSPECTED`, `LIKELY`, `CONFIRMED`                                                                                                                                                         |
| `severity`                       | enum                         | да    | `INFO, LOW, MEDIUM, HIGH, CRITICAL`                                                                                                                                                        |
| `confidence`                     | int 1–99                     | да    |                                                                                                                                                                                            |
| `fingerprint`                    | hex(64)                      | да    | SHA-256 нормализованного ключа (см. X-5 / Q-6 / I-5)                                                                                                                                       |
| `details`                        | object, дискриминатор `type` | да    | поля зависят от `vulnType`                                                                                                                                                                 |
| `evidence`                       | object                       | да    | ≤ 16 КБ после сериализации; `bodyExcerpt` ≤ 2 КБ; заголовки прошли `Redactor`                                                                                                              |
| `parameter`                      | `{name, location}`           | нет   | для IDOR — параметр идентификатора                                                                                                                                                         |
| `evidence.response.highlights[]` | `{start,end}`                | нет   | индексы **внутри `bodyExcerpt`**, считаются в UTF-16 code units строки после JSON-разбора (как индексы строк JavaScript и Kotlin); `0 ≤ start < end ≤ bodyExcerpt.length`, не пересекаются |

### B.8.4. `JOB_COMPLETED` и `JOB_FAILED`

```json
{
  "schemaVersion": 1,
  "messageId": "6d9a...",
  "type": "JOB_COMPLETED",
  "correlationId": "c0a8f3d2-...",
  "occurredAt": "2026-10-22T15:09:40Z",
  "payload": {
    "jobId": "8f0d6d3e-...",
    "attempt": 1,
    "stats": {
      "findingsCount": 3,
      "requestsSent": 212,
      "durationMs": 338000,
      "endpointsTested": 4,
      "endpointsSkipped": [
        { "endpointId": "e1d0...", "reason": "OUT_OF_SCOPE" }
      ],
      "inconclusive": 1
    }
  }
}
```

```json
{
  "schemaVersion": 1,
  "messageId": "91ab...",
  "type": "JOB_FAILED",
  "correlationId": "c0a8f3d2-...",
  "occurredAt": "2026-10-22T15:06:02Z",
  "payload": {
    "jobId": "8f0d6d3e-...",
    "attempt": 1,
    "errorCode": "TARGET_UNREACHABLE",
    "message": "connect timeout after 3 attempts",
    "retryable": true,
    "stats": { "findingsCount": 0, "requestsSent": 9, "durationMs": 31000 }
  }
}
```

`errorCode` ∈ `INVALID_JOB`, `SCOPE_VIOLATION`, `TARGET_UNREACHABLE`, `AUTH_FAILED`, `REQUEST_BUDGET_EXHAUSTED`, `DEADLINE_EXCEEDED`, `INTERNAL_ERROR`. `retryable=true` только для `TARGET_UNREACHABLE`, `INTERNAL_ERROR`. `REQUEST_BUDGET_EXHAUSTED` — не ошибка: модуль завершает `JOB_COMPLETED` с `stats.budgetExhausted=true`.

### B.8.5. Internal HTTP (модуль → Core)

Аутентификация: `Authorization: Bearer <service-token>` (по токену на модуль, из `SCANNER_SERVICE_TOKEN_<MODULE>`), `X-Worker-Id: <module>-<hostname>-<pid>`, `X-Correlation-Id`.

| Метод и путь                               | Тело запроса                                         | Ответ                                                                                                                                |
| ------------------------------------------ | ---------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| `POST /internal/v1/jobs/{jobId}/claim`     | `{"attempt":1,"workerId":"xss-7f9c-1"}`              | `200 {"leaseExpiresAt":"...","heartbeatIntervalSec":10}`; `409 {"code":"JOB_NOT_CLAIMABLE"}`; `410 {"code":"JOB_CANCELLED"}`; `404`  |
| `POST /internal/v1/jobs/{jobId}/heartbeat` | `{"attempt":1,"requestsSent":120,"endpointsDone":2}` | `200 {"cancel":false,"leaseExpiresAt":"..."}`; `409` если lease потерян                                                              |
| `GET /internal/v1/jobs/{jobId}/identities` | —                                                    | `200 [{"identityId","label","role","auth":{"type":"FORM_LOGIN",...}}]` с секретами; только владелец lease; `Cache-Control: no-store` |

### B.8.6. Формат ошибок HTTP (единый для `/api/v1` и `/internal/v1`)

```json
{
  "timestamp": "2026-10-22T15:04:05Z",
  "status": 422,
  "code": "SCOPE_VIOLATION",
  "message": "Target host is not allowed",
  "correlationId": "c0a8f3d2-...",
  "fieldErrors": [
    {
      "field": "targetId",
      "code": "TARGET_NOT_VERIFIED",
      "message": "Target is not verified"
    }
  ]
}
```

`code` — стабильный машинный идентификатор (frontend ветвится по нему, а не по `message`).

## B.9. Основные HTTP-запросы Frontend ↔ Core

```json
// POST /api/v1/auth/login
{"username":"student","password":"***"}
// 200 {"id":"…","username":"student","role":"USER","csrfToken":"…"}

// POST /api/v1/scans   (заголовок X-XSRF-TOKEN)
{
  "targetId": "7d9b0f5e-5a35-4b6c-b0aa-3c3e6e9a1d44",
  "modules": ["XSS", "SQLI", "IDOR"],
  "policyId": "a3f1d4c2-1f21-4a0e-8d11-9b0c7e5d6f33",
  "endpointIds": ["e1d0c4aa-...", "e2f0c4aa-..."],
  "scope": {"excludePrefixes": ["http://lab-app:8080/logout"]},
  "moduleOptions": {"xss": {"reflected": true, "stored": false}, "sqli": {"timeBased": false}, "idor": {"identityIds": ["…A","…B","…C"]}}
}
// 202 Accepted
{"id":"2c1e7a9a-…","status":"PENDING","correlationId":"c0a8f3d2-…","jobs":[{"id":"8f0d…","moduleType":"XSS","status":"PENDING","attempt":1}]}
```

Ответ `202` означает **только** «сканирование принято и задания записаны»; выполнение — асинхронное.

## B.10. Модель данных (обзор; DDL — в Core, этап C-2)

```mermaid
erDiagram
  users ||--o{ targets : owns
  targets ||--o{ target_endpoints : has
  targets ||--o{ test_identities : has
  targets ||--o{ resource_access_rules : has
  targets ||--o{ scans : scanned_by
  scan_policies ||--o{ scans : applied
  users ||--o{ scans : started_by
  scans ||--|{ scan_jobs : consists_of
  scan_jobs ||--o{ findings : produces
  scan_jobs ||--o{ outbox_events : emits
  users ||--o{ audit_events : actor
```

| Таблица                                    | Назначение                                                                                        |
| ------------------------------------------ | ------------------------------------------------------------------------------------------------- |
| `users`                                    | Учётные записи, роль `USER`/`ADMIN`, хэш пароля (bcrypt/argon2 через `DelegatingPasswordEncoder`) |
| `targets`                                  | Разрешённые цели: `base_url`, `allowed_hosts jsonb`, `status`, `verification_token`, `owner_id`   |
| `target_endpoints`                         | Endpoint inventory: метод, URL, параметры `jsonb`, `idempotent`                                   |
| `scan_policies`                            | Пресеты лимитов (`LIGHT`, `NORMAL`); `limits jsonb`                                               |
| `test_identities`, `resource_access_rules` | IDOR: идентичности (секрет шифрован) и матрица «ресурс → владелец → ожидаемый доступ»             |
| `scans`, `scan_jobs`                       | Запрос на сканирование и его задания по модулям (по одному на модуль)                             |
| `findings`                                 | Результаты; `UNIQUE(job_id, fingerprint)`                                                         |
| `outbox_events`, `processed_messages`      | Надёжная публикация и дедупликация потребления                                                    |
| `audit_events`                             | Аудит действий пользователей и выдачи секретов                                                    |
| Spring Session JDBC (`spring_session*`)    | Серверные сессии (создаются миграцией, не вручную в коде)                                         |

**Различие сущностей.** `Scan` — намерение пользователя («проверить target T модулями XSS и SQLi с политикой P»), хранит конфигурацию-снимок и агрегированный статус. `ScanJob` — единица работы для одного модуля с собственным жизненным циклом (`attempt`, `lease`, `deadline`). `Finding` принадлежит `ScanJob` (через `job_id`) и денормализованно хранит `scan_id` для быстрых запросов dashboard. Один `Scan` содержит от 1 до 3 `ScanJob` (`UNIQUE(scan_id, module_type)`).

## B.11. Жизненный цикл сканирования (обзор)

```mermaid
sequenceDiagram
  participant UI as Frontend
  participant C as Core
  participant DB as PostgreSQL
  participant K as Kafka
  participant M as Module
  participant T as Target
  UI->>C: POST /api/v1/scans
  C->>C: authz, ScopePolicyService, validate
  C->>DB: TX: scan + scan_jobs(PENDING) + outbox_events
  C-->>UI: 202 {scanId}
  C->>DB: OutboxRelay: SELECT FOR UPDATE SKIP LOCKED
  C->>K: SCAN_JOB (key=jobId)
  K-->>C: ack
  C->>DB: outbox SENT, job QUEUED
  K->>M: SCAN_JOB
  M->>C: POST /internal/.../claim
  C->>DB: CAS QUEUED→RUNNING (lease)
  C-->>M: 200
  M->>K: commit offset
  loop каждые 10 с
    M->>C: heartbeat (requestsSent)
    C-->>M: {cancel:false}
  end
  M->>T: проб-запросы (SafeHttpClient)
  M->>K: FINDING_REPORTED ×N
  M->>K: JOB_COMPLETED
  K->>C: scan.job-events.v1
  C->>DB: TX: dedupe, save finding, job COMPLETED, aggregate scan
  UI->>C: GET /api/v1/scans/{id} (polling)
```

Обработка сбоев по шагам детально описана в Core, этап C-5/C-7 и сводной таблице Части C (раздел «Матрица отказов»).

---

# Часть C. Пошаговый план разработки

Идентификаторы этапов: `S-*` — общий слой (репозиторий, `scanner-kit`), `C-*` — Core Backend, `F-*` — Frontend, `X-*` — XSS, `Q-*` — SQLi, `I-*` — IDOR, `T-*` — тестовые стенды, `E-*` — оценка, `N-*` — инфраструктура. Формат этапа: Цель / Предварительные условия / Файлы / Реализация / Интерфейсы / Проверки / Критерии завершения.

Базовый пакет Kotlin: `io.vulnscan.<project>` (например `io.vulnscan.core`, `io.vulnscan.xss`, `io.vulnscan.kit`).

## Рекомендуемый порядок и параллельность

```
S-1 ─► S-2 (контракты) ─┬─► C-1..C-4 ─► C-5 ─► C-6 ─► C-7 ─► C-8
                        ├─► F-1..F-2 (на mock API по OpenAPI)  ─► F-3..F-5
                        └─► S-3 ─► S-4 ─► X-1..X-6 / Q-1..Q-6 / I-1..I-6
T-1 (lab-app) параллельно с S-3; N-1 (compose) после C-1; E-* после первого рабочего модуля
```

Интерфейсы, которые должны быть согласованы **до** параллельной работы: `contracts/schemas/*.json` и `contracts/examples/*.json` (S-2), `contracts/openapi/core-api.v1.json` (черновик после C-3), `ground-truth.yaml` формат (T-1).

---

## Проект S. Репозиторий и общая библиотека `scanner-kit`

### Этап S-1 — Каркас монорепозитория

- **Цель:** собираемый пустой монорепозиторий с единой фиксацией версий.
- **Предварительные условия:** нет.
- **Файлы:**
  - `settings.gradle.kts` (включает `core-backend`, `xss-module`, `sqli-module`, `idor-module`, `libs:scanner-kit`, `testbeds:lab-app`, `evaluation:runner`)
  - `gradle/libs.versions.toml` — все версии зависимостей
  - `frontend/package.json` + `package-lock.json`
  - `README.md` (как запустить, где контракты)
  - `.editorconfig`, `.gitignore`, `.github/workflows/ci.yml` (или эквивалент)
- **Реализация:**
  1. Выбрать JDK LTS и совместимые версии Spring Boot/Spring Kafka/Kotlin по документации релиза; записать их в `libs.versions.toml` (зафиксировать до начала S-2).
  2. Создать Gradle-модули с подключёнными плагинами Kotlin/Spring (kotlin-spring, kotlin-jpa только в Core).
  3. Настроить `ktlint`/`detekt` (одна конфигурация на репозиторий) и `./gradlew check` как единую команду проверки.
  4. CI: job `backend` (`./gradlew check`), job `frontend` (`npm ci && npm run lint && npm test && npm run build`).
- **Интерфейсы:** —
- **Проверки:** CI на пустом коммите зелёный.
- **Критерии завершения:** `./gradlew check` и `npm ci && npm run build` проходят на чистом клоне; версии в одном месте.

### Этап S-2 — Контракты: схемы, примеры, DTO

- **Цель:** формальные контракты B.8 в виде JSON Schema, примеров и Kotlin-DTO; контрактный тест, ловящий расхождения.
- **Предварительные условия:** S-1.
- **Файлы:**
  - `contracts/schemas/envelope.v1.json`, `scan-job.v1.json`, `finding-reported.v1.json`, `job-completed.v1.json`, `job-failed.v1.json`, `internal-claim.v1.json`, `internal-heartbeat.v1.json`
  - `contracts/examples/*.json` — примеры из B.8 (минимум по одному валидному и по два невалидных на тип)
  - `libs/scanner-kit/src/main/kotlin/io/vulnscan/kit/contracts/` — `Envelope.kt`, `ScanJobPayload.kt`, `FindingPayload.kt`, `FindingDetails.kt` (sealed interface: `XssDetails`, `SqliDetails`, `IdorDetails`), `JobEvents.kt`, `InternalApi.kt`, `Enums.kt` (`ModuleType`, `VulnType`, `Verdict`, `Severity`, `ParamLocation`, `ExpectedAccess`, `ErrorCode`)
  - `libs/scanner-kit/src/main/kotlin/io/vulnscan/kit/contracts/Json.kt` — единственный сконфигурированный `ObjectMapper` (Kotlin module, `FAIL_ON_UNKNOWN_PROPERTIES=false`, `WRITE_DATES_AS_TIMESTAMPS=false`, ISO-8601)
  - `libs/scanner-kit/src/main/kotlin/io/vulnscan/kit/contracts/Fingerprint.kt` — `object Fingerprint { fun of(vararg parts: String): String }` (SHA-256 hex от частей, разделённых `\u0000`)
  - `libs/scanner-kit/src/test/kotlin/.../ContractSchemaTest.kt`
- **Реализация:**
  1. Описать схемы по таблицам B.8.1–B.8.5 (`required`, `enum`, `format: uuid/date-time`, `maxLength` для `bodyExcerpt`, `maxItems` для `endpoints` = 200).
  2. Реализовать DTO как `data class` с `@JsonTypeInfo(use = NAME, property = "type")` для `FindingDetails`.
  3. Добавить `ScanJobPayloadValidator.validate(payload): List<ViolationCode>` — семантические проверки, которые схема не выразит: `moduleOptions` соответствует `moduleType`; `deadlineAt > occurredAt`; `limits` положительные; `READ_ONLY` ⇒ только GET/HEAD; каждый `endpoint.url` лежит под одним из `scope.includePrefixes`.
  4. Контрактный тест: для каждого `examples/valid/*.json` — (a) валиден по схеме (библиотека JSON Schema Validator), (b) `Json.mapper.readValue` → `writeValueAsString` → снова валиден; для `examples/invalid/*.json` — схема отклоняет.
- **Интерфейсы:** потребляется Core, тремя модулями, Frontend (только OpenAPI, не эти схемы).
- **Проверки:** `ContractSchemaTest` (все примеры); `ScanJobPayloadValidatorTest` — по одному тесту на каждое правило (module/options mismatch, deadline в прошлом, GET-only для READ_ONLY, URL вне scope).
- **Критерии завершения:** все примеры из B.8 лежат в `contracts/examples` и проходят тест; несовместимое изменение DTO без правки схемы ломает сборку.

### Этап S-3 — Сетевая безопасность: `IpPolicy`, `SafeHttpClient`, `Redactor`

- **Цель:** единственная реализация защищённого HTTP-клиента и маскирования секретов, используемая Core (проверка владения target) и всеми модулями.
- **Предварительные условия:** S-1.
- **Файлы** (`libs/scanner-kit/src/main/kotlin/io/vulnscan/kit/`):
  - `net/IpPolicy.kt` — `class IpPolicy(allowedPrivateCidrs: List<Cidr>)`; `fun check(addr: InetAddress): IpVerdict`
  - `net/Cidr.kt` — разбор и `contains(InetAddress)` для IPv4/IPv6
  - `net/ScopeMatcher.kt` — `class ScopeMatcher(allowedHosts, includePrefixes, excludePrefixes)`; `fun check(url: HttpUrl): ScopeDecision`
  - `net/SafeHttpClient.kt` — обёртка над OkHttp: `fun execute(req: ProbeRequest): ProbeResponse`
  - `net/SafeDns.kt` — реализация `okhttp3.Dns` (резолв + проверка + возврат только проверенных адресов)
  - `net/ProbeRequest.kt`, `ProbeResponse.kt` (status, headers, `bodyBytes` ≤ лимит, `truncated`, `elapsedMs`, `finalUrl`, `redirectChain`)
  - `net/ResponseLimits.kt`
  - `redact/Redactor.kt`
  - тесты: `IpPolicyTest.kt`, `ScopeMatcherTest.kt`, `SafeHttpClientTest.kt` (MockWebServer), `RedactorTest.kt`
- **Реализация:**
  1. `IpPolicy.check` запрещает: loopback (`127.0.0.0/8`, `::1`), unspecified (`0.0.0.0`, `::`), link-local (`169.254.0.0/16` — в т. ч. метаданные облака, `fe80::/10`), private (`10/8`, `172.16/12`, `192.168/16`, `fc00::/7`), CGNAT (`100.64/10`), multicast, broadcast, IPv4-mapped IPv6 (разворачивать и проверять вложенный IPv4), документационные диапазоны. Исключение: адрес входит в `allowedPrivateCidrs` (задаётся только конфигурацией, в dev-профиле — подсеть `lab-net`, по умолчанию пусто).
  2. `SafeDns.lookup(host)`: строгий разбор литерала IP (отклонять десятичные/восьмеричные/шестнадцатеричные формы `2130706433`, `0177.0.0.1`, `0x7f.1`: допускается только стандартная запись IPv4/IPv6 или DNS-имя); резолв через `InetAddress.getAllByName`; **каждый** адрес проходит `IpPolicy`, иначе `ScopeViolationException(PRIVATE_ADDRESS)`; возвращается только список проверенных адресов. OkHttp подключается к ним без повторного резолва — это закрывает окно DNS-rebinding между проверкой и соединением.
  3. `SafeHttpClient`: (а) перед запросом — `ScopeMatcher.check(url)` (схема `http|https`, нет userinfo, host/port в `allowedHosts`, префикс include/exclude, URL нормализован: `HttpUrl` разбор, punycode, удаление fragment, запрет `\` и управляющих символов); (б) `followRedirects(false)` — редиректы обрабатываются вручную: максимум 3 перехода, каждый `Location` разрешается относительно текущего URL и снова проходит `ScopeMatcher` + `SafeDns`; редирект вне scope не выполняется, возвращается ответ 3xx с `redirectBlocked=true`; (в) network-интерсептор после установки соединения повторно проверяет `connection.socket().inetAddress` через `IpPolicy` (защита от обхода через прокси/нестандартный `Dns`); (г) таймауты connect/read/call из `limits.httpTimeoutMs`; (д) чтение тела через ограниченный источник: не более `maxResponseBytes`, остаток отбрасывается, `truncated=true`; (е) без прокси (`proxy(Proxy.NO_PROXY)`), отключены автоматические декомпрессия/кэш/куки (куки — только явный `CookieJar` IDOR-идентичности); (ж) единый `User-Agent: VulnScanDiploma/<version> (+authorized-testing)`.
  4. `Redactor`: `fun headers(map): Map<String,String>` — значения `Authorization`, `Cookie`, `Set-Cookie`, `Proxy-Authorization`, `X-API-Key`, `X-Auth-Token`, `X-CSRF-Token` заменяются на `[REDACTED]`; `fun url(u): String` и `fun body(text): String` — значения параметров с именами по шаблону `(?i)(pass(word)?|pwd|secret|token|session|sid|api[_-]?key|auth|jwt|csrf)` заменяются на `[REDACTED]`; JSON-тело разбирается и маскируется по ключам; `fun excerpt(text, max): String` — обрезка с суффиксом `…[truncated]`.
- **Интерфейсы:** используется `WorkerRuntime` (S-4), Core `TargetVerificationService` (C-3), все детекторы.
- **Проверки:**
  - `IpPolicyTest`: таблица адресов → ожидаемый вердикт (127.0.0.1, 10.0.0.5, 169.254.169.254, `::1`, `::ffff:127.0.0.1`, `fe80::1`, `100.64.0.1`, 8.8.8.8 разрешён; 172.28.0.5 разрешён только при allowlist).
  - `SafeHttpClientTest`: (1) запрос вне scope не отправляется (MockWebServer получил 0 запросов); (2) редирект на другой host блокируется; (3) редирект на `127.0.0.1` блокируется; (4) 4-й редирект подряд обрывается; (5) тело 5 МБ при лимите 1 МБ → `truncated=true`, прочитано ≤ 1 МБ; (6) таймаут чтения → `ProbeTimeoutException`; (7) подмена `Dns`: имя резолвится в два адреса, один запрещён → весь запрос отклонён; (8) ответ 200 с `Set-Cookie` не сохраняется между запросами.
  - `ScopeMatcherTest`: `http://lab-app:8080@evil/`, `http://lab-app:8080.evil.com/`, `http://lab-app:8081/`, `HTTP://LAB-APP:8080/Logout` (регистр хоста), `http://lab-app:8080/%2e%2e/admin` — каждое в ожидаемом решении.
  - `RedactorTest`: заголовки, query, form-body, JSON с вложенными ключами.
- **Критерии завершения:** все перечисленные тесты зелёные; ни один публичный метод `SafeHttpClient` не позволяет передать «сырой» `OkHttpClient`; в модулях прямое использование OkHttp запрещено правилом ArchUnit `NoDirectHttpClientRule` (тест в каждом модуле).

### Этап S-4 — Рантайм воркера: `WorkerRuntime`

- **Цель:** общий каркас «получить задание → claim → heartbeat → выполнить → опубликовать», чтобы три модуля реализовывали только `DetectionEngine`.
- **Предварительные условия:** S-2, S-3.
- **Файлы** (`libs/scanner-kit/src/main/kotlin/io/vulnscan/kit/worker/`):
  - `DetectionEngine.kt` — `interface DetectionEngine { val moduleType: ModuleType; fun run(ctx: JobContext) }`
  - `JobContext.kt` — `http: BudgetedHttp`, `job: ScanJobPayload`, `report(f: FindingPayload)`, `isCancelled(): Boolean`, `identities(): List<IdentityCredentials>` (только IDOR), `stats: MutableJobStats`
  - `BudgetedHttp.kt` — оборачивает `SafeHttpClient`: считает запросы, применяет `RateLimiter`, бросает `BudgetExhaustedException` при `maxRequests`, `DeadlineExceededException` при `deadlineAt`, `CancelledException` при `isCancelled`
  - `RateLimiter.kt` — token bucket (`requestsPerSecond`), потокобезопасный
  - `CoreControlClient.kt` — `claim()`, `heartbeat()`, `fetchIdentities()` (HTTP на Core, тоже через отдельный `OkHttpClient` без `SafeHttpClient`: адрес Core — доверенный из конфигурации)
  - `LeaseHeartbeat.kt` — планировщик (daemon-поток), каждые `heartbeatIntervalSec` шлёт счётчики, выставляет `cancelled` по ответу; при `409` выставляет `leaseLost` и прерывает работу
  - `ResultPublisher.kt` — `KafkaTemplate<String,String>`; `publishFinding`, `publishCompleted`, `publishFailed`; ключ = `jobId`; `send().get(10s)` с повтором 3 раза; создаёт `messageId`
  - `JobListener.kt` — `@KafkaListener` базовый: парсинг → `ScanJobPayloadValidator` → проверка `moduleType` → claim → ack → передача в `JobExecutor`
  - `JobExecutor.kt` — пул из `maxConcurrentJobs` потоков; управление `pause()/resume()` контейнера
  - `WorkerProperties.kt` — `@ConfigurationProperties("worker")`: `maxConcurrentJobs`, `hardLimits` (верхние границы `limits`), `coreBaseUrl`, `serviceToken`, `allowedPrivateCidrs`
  - тесты: `BudgetedHttpTest`, `LeaseHeartbeatTest`, `JobListenerTest`, интеграционный `WorkerRuntimeKafkaIT` (Testcontainers Kafka + MockWebServer вместо Core)
- **Реализация:**
  1. `JobListener.onMessage`: (а) невалидный JSON/схема/`moduleType` → `DeadLetterPublishingRecoverer` без retry; (б) `payload.limits` усекаются по `hardLimits`; (в) если `deadlineAt` в прошлом → `claim` не вызывается, сообщение подтверждается и итог не публикуется (Core сам завершит по таймауту); (г) нет свободного слота → `pause()`, `nack`; (д) `claim`: `200` → `ack`, запуск; `409/410` → `ack` без запуска; сетевая ошибка → исключение → `DefaultErrorHandler` (3 попытки по 2 с, затем DLQ).
  2. `JobExecutor.run`: создаёт `JobContext`, запускает `LeaseHeartbeat`, вызывает `engine.run`. Исход: нормальное завершение → `JOB_COMPLETED(stats)`; `BudgetExhausted` → `JOB_COMPLETED(budgetExhausted=true)`; `Cancelled` → ничего не публикует; `leaseLost` → ничего не публикует; `DeadlineExceeded` → `JOB_FAILED(DEADLINE_EXCEEDED, retryable=false)`; `ScopeViolationException` на стартовой проверке → `JOB_FAILED(SCOPE_VIOLATION, false)`; прочее → `JOB_FAILED(INTERNAL_ERROR, true)` без стек-трейса в `message` (стек — в лог).
  3. Все публикуемые finding проходят `FindingSanitizer.apply`: `Redactor` на evidence, усечение до 16 КБ, проверка `confidence ∈ 1..99`, `fingerprint` hex-64.
  4. Метрики Micrometer: `scanner.jobs.started{module}`, `scanner.jobs.failed{module,code}`, `scanner.requests.sent{module}`, `scanner.findings.reported{module,vulnType}`.
- **Интерфейсы:** Kafka-топики D-1/D-2, internal HTTP B.8.5.
- **Проверки:**
  - `BudgetedHttpTest`: 600-й+1 запрос → `BudgetExhaustedException`; `RateLimiter` при 5 rps выдаёт ≤ 6 запросов за 1 с на 20 параллельных потоков; после `cancelled=true` следующий запрос → `CancelledException`.
  - `LeaseHeartbeatTest`: при ответе `cancel=true` флаг выставляется ≤ 1 интервала; при `409` — `leaseLost`.
  - `WorkerRuntimeKafkaIT` (реальный Kafka): (1) валидное задание + `claim=200` → вызван `engine.run` один раз, опубликован `JOB_COMPLETED`; (2) то же сообщение дважды + второй `claim=409` → `engine.run` один раз; (3) сообщение с неверным `moduleType` → запись в `scan.dlq.v1`, `engine.run` не вызван; (4) `engine` бросает `RuntimeException` → `JOB_FAILED(INTERNAL_ERROR, retryable=true)`; (5) `maxConcurrentJobs=1`, два задания подряд → второе claim выполняется только после завершения первого.
- **Критерии завершения:** тестовый `EchoEngine` (в тестах) проходит весь путь в `WorkerRuntimeKafkaIT`; три модуля могут подключить рантайм, реализовав только `DetectionEngine`.

---

## Проект 2. Core Backend (`core-backend`)

### Структура пакетов `io.vulnscan.core`

| Пакет           | Содержимое                                                                                                                                             | Правило зависимости                                          |
| --------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------ | --------------------------------------------------- |
| `api`           | `*Controller`, `dto/\*Request                                                                                                                          | Response`, `ApiExceptionHandler`                             | зависит от `application`; не знает про JPA-сущности |
| `application`   | use-case сервисы: `ScanService`, `TargetService`, `FindingQueryService`, `JobResultService`, `JobControlService`, `ScopePolicyService`, `AuditService` | зависит от `domain`, `persistence` (интерфейсы репозиториев) |
| `domain`        | `ScanStatus`, `JobStatus`, `JobStateMachine`, `ScanStatusAggregator`, `Verdict`…, исключения                                                           | чистый Kotlin, без Spring и JPA                              |
| `persistence`   | JPA-сущности `*Entity`, `*Repository`, `FindingSpecifications`, мапперы                                                                                | реализует доступ к БД                                        |
| `messaging`     | `OutboxRelay`, `OutboxEventFactory`, `JobEventsConsumer`, `KafkaConfig`, `KafkaTopics`                                                                 | вызывает `application`                                       |
| `security`      | `SecurityConfig`, `ServiceTokenFilter`, `RateLimitFilter`, `SecretCipher`, `CurrentUser`                                                               |                                                              |
| `config`        | `@ConfigurationProperties` (`ScannerProperties`)                                                                                                       |                                                              |
| `observability` | `CorrelationFilter`, `MdcKeys`, `Metrics`                                                                                                              |                                                              |
| `scheduling`    | `StuckJobWatchdog`, `RetentionCleaner`                                                                                                                 | вызывает `application`                                       |

Правило ArchUnit (`ArchitectureTest`): `domain` не зависит от Spring/JPA; `api` не обращается к `persistence` напрямую; `messaging` и `scheduling` не используют `api`. Интерфейсы-абстракции вводятся только там, где есть вторая реализация или mock в тестах (`Clock`, `DnsResolver`, `KafkaPublisher`).

### Этап C-1 — Каркас приложения

- **Цель:** запускаемое приложение с миграциями, единым форматом ошибок, корреляцией и базой для интеграционных тестов.
- **Предварительные условия:** S-1, S-2.
- **Файлы:**
  - `core-backend/src/main/kotlin/io/vulnscan/core/CoreApplication.kt`
  - `.../config/ScannerProperties.kt` — `scope.allowedPrivateCidrs`, `scope.allowedPorts` (по умолчанию `80,443,8080,8443`), `limits.maxActiveScansPerUser=2`, `limits.maxActiveScansGlobal=4`, `jobs.maxAttempts=2`, `jobs.queueTimeoutSec=600`, `jobs.leaseSec=45`, `jobs.maxEndpointsPerScan=200`, `secret.keyBase64` (`SCANNER_SECRET_KEY`)
  - `.../api/ApiExceptionHandler.kt` — `@RestControllerAdvice`: `DomainException` → `ApiError` (B.8.6); `MethodArgumentNotValidException`/`ConstraintViolationException` → 400 + `fieldErrors`; `AccessDeniedException` → 403; необработанное → 500 `INTERNAL_ERROR` без деталей
  - `.../api/dto/ApiError.kt`
  - `.../domain/Exceptions.kt` — `NotFound`, `Conflict`, `ScopeViolation`, `IllegalStateTransition`, `TooManyActiveScans` с полем `code`
  - `.../observability/CorrelationFilter.kt` — читает/создаёт `X-Correlation-Id`, кладёт в MDC и ответ
  - `core-backend/src/main/resources/application.yml`, `application-dev.yml`, `application-test.yml`, `logback-spring.xml` (JSON)
  - `core-backend/src/test/kotlin/.../support/PostgresKafkaTestBase.kt` — абстрактный класс с Testcontainers (PostgreSQL + Kafka) и `@DynamicPropertySource`
  - `.../ArchitectureTest.kt`
- **Реализация:**
  1. Подключить Flyway, `spring.jpa.hibernate.ddl-auto=validate`, `open-in-view=false`.
  2. `ApiExceptionHandler`: каждый код ошибки — константа в `ErrorCodes.kt`; HTTP-статус определяется в одном `when`.
  3. `CorrelationFilter` — самый ранний фильтр; невалидный UUID во входящем заголовке игнорируется.
  4. `application-test.yml`: короткие интервалы (`jobs.leaseSec=3`, `outbox.pollMs=100`) для быстрых тестов.
- **Интерфейсы:** формат ошибок B.8.6.
- **Проверки:** `ApiExceptionHandlerTest` (MockMvc: каждый тип исключения → статус, `code`, отсутствие стек-трейса в теле); `CorrelationFilterTest` (заголовок принят/сгенерирован/невалидный заменён); `ArchitectureTest`; `CoreApplicationIT` (контекст стартует против Testcontainers, `GET /actuator/health` = UP).
- **Критерии завершения:** приложение стартует, миграции V1 применяются, все тесты зелёные.

### Этап C-2 — Схема БД, сущности, репозитории

- **Цель:** полная схема PostgreSQL с ограничениями целостности и индексами; репозитории.
- **Предварительные условия:** C-1.
- **Файлы:**
  - `core-backend/src/main/resources/db/migration/V1__users_targets.sql`, `V2__policies_identities.sql`, `V3__scans_jobs_findings.sql`, `V4__outbox_processed_audit.sql`, `V5__spring_session.sql`, `V9__seed_policies.sql`
  - `.../persistence/*Entity.kt`, `*Repository.kt`, `FindingSpecifications.kt`, `Mappers.kt`
  - `.../persistence/MigrationIT.kt`
- **Реализация (DDL-контракт).** Типы: `uuid` (PK, `gen_random_uuid()`), `timestamptz`, `jsonb`, `text` с `CHECK` вместо PostgreSQL enum (проще миграции).

  `users(id PK, username text NOT NULL UNIQUE, password_hash text NOT NULL, role text NOT NULL CHECK(role IN ('USER','ADMIN')), enabled bool NOT NULL DEFAULT true, created_at)`

  `targets(id PK, owner_id FK users ON DELETE RESTRICT, name text NOT NULL, base_url text NOT NULL, allowed_hosts jsonb NOT NULL, status text CHECK IN ('PENDING_VERIFICATION','VERIFIED','DISABLED'), verification_token text NOT NULL, verified_at, created_at; UNIQUE(owner_id, base_url))`; индекс `(owner_id)`.

  `target_endpoints(id PK, target_id FK targets ON DELETE CASCADE, method text, url text NOT NULL, parameters jsonb NOT NULL DEFAULT '[]', idempotent bool NOT NULL DEFAULT false, label text, created_at; UNIQUE(target_id, method, url))`; индекс `(target_id)`.

  `scan_policies(id PK, name text UNIQUE NOT NULL, limits jsonb NOT NULL, created_at)` — seed `LIGHT` (`maxRequests=200, rps=2, concurrency=1`), `NORMAL` (`600, 5, 2`).

  `test_identities(id PK, target_id FK ON DELETE CASCADE, label text NOT NULL, role text NOT NULL, auth_type text CHECK IN ('FORM_LOGIN','BEARER_STATIC','COOKIE_STATIC','HEADER_STATIC','JSON_LOGIN'), auth_config jsonb NOT NULL, secret_encrypted bytea NOT NULL, key_version int NOT NULL, created_at; UNIQUE(target_id,label))` — `auth_config` содержит несекретную часть (URL логина, имена полей, JSONPath токена); секреты (пароль/токен/cookie) — только в `secret_encrypted`.

  `resource_access_rules(id PK, target_id FK ON DELETE CASCADE, endpoint_id FK target_endpoints ON DELETE CASCADE, id_parameter jsonb NOT NULL, id_value text NOT NULL, owner_identity_id FK test_identities ON DELETE CASCADE, sensitivity text CHECK IN ('LOW','MEDIUM','HIGH'), owner_markers jsonb NOT NULL, access jsonb NOT NULL)`; индекс `(target_id)`.

  `scans(id PK, target_id FK RESTRICT, policy_id FK RESTRICT, started_by FK users RESTRICT, status text NOT NULL, cancel_requested bool NOT NULL DEFAULT false, correlation_id uuid NOT NULL, config jsonb NOT NULL, created_at, finished_at)`; индексы `(started_by, created_at DESC)`, `(status)` частичный по `status IN ('PENDING','RUNNING','CANCELLING')`.

  `scan_jobs(id PK, scan_id FK ON DELETE CASCADE, module_type text CHECK IN ('XSS','SQLI','IDOR'), status text NOT NULL, attempt int NOT NULL DEFAULT 1, worker_id text, lease_expires_at timestamptz, deadline_at timestamptz NOT NULL, queued_at, started_at, finished_at, requests_sent int NOT NULL DEFAULT 0, endpoints_done int NOT NULL DEFAULT 0, error_code text, error_message text, result_integrity text NOT NULL DEFAULT 'OK' CHECK IN ('OK','INCOMPLETE'), stats jsonb, UNIQUE(scan_id, module_type))`; индексы `(status, lease_expires_at)`, `(status, queued_at)`.

  `findings(id PK, job_id FK scan_jobs ON DELETE CASCADE, scan_id FK scans ON DELETE CASCADE, target_id FK targets ON DELETE CASCADE, module_type, vuln_type text, verdict text, severity text, confidence smallint CHECK 1..99, fingerprint char(64) NOT NULL, endpoint_id uuid NULL (без FK: endpoint может быть удалён), method text, url text, parameter_name text, parameter_location text, title text, summary text, recommendation text, cwe text, details jsonb, evidence jsonb, triage_status text NOT NULL DEFAULT 'OPEN' CHECK IN ('OPEN','FALSE_POSITIVE','FIXED'), created_at, UNIQUE(job_id, fingerprint))`; индексы: `(scan_id, severity, created_at DESC)`, `(target_id, triage_status)`, `(vuln_type)`, `(fingerprint)`.

  `outbox_events(id PK, aggregate_id uuid NOT NULL (jobId), topic text NOT NULL, message_key text NOT NULL, payload jsonb NOT NULL, headers jsonb NOT NULL, status text CHECK IN ('PENDING','SENT'), attempts int NOT NULL DEFAULT 0, next_attempt_at timestamptz NOT NULL, created_at, sent_at)`; индекс частичный `(next_attempt_at) WHERE status='PENDING'`.

  `processed_messages(message_id uuid PK, consumer text NOT NULL, processed_at)`; чистка записей старше 7 суток (`RetentionCleaner`).

  `audit_events(id PK, actor_id uuid NULL FK users ON DELETE SET NULL, action text NOT NULL, subject_type text, subject_id uuid, details jsonb, created_at)`; индекс `(created_at DESC)`.

  Правила удаления: пользователь и target с историей сканов не удаляются (`RESTRICT`) — вместо этого `enabled=false` / `status=DISABLED`; удаление `scan` каскадирует на `scan_jobs` и `findings`.
  Стратегия миграций: только вперёд, файлы неизменяемы после слияния; `V9` — seed-данные; тест `MigrationIT` применяет все миграции на пустую БД и проверяет `ddl-auto=validate`.

- **Интерфейсы:** репозитории используются `application`-слоем; JSON-колонки — через `@JdbcTypeCode(SqlTypes.JSON)` и DTO из `scanner-kit` (`ScanLimits`, `AllowedHost`, `EndpointParameter`).
- **Проверки:** `MigrationIT` (миграции + `validate`); `ConstraintsIT` — нарушение каждого `UNIQUE`/`CHECK` даёт ожидаемое исключение (дубль `(scan_id, module_type)`, дубль `(job_id, fingerprint)`, `confidence=0`, неверный `status`); `FindingSpecificationsIT` — фильтры по `severity`, `verdict`, `module_type`, `triage_status`, `parameter_name` на 1 000 сгенерированных строк возвращают корректные подмножества и не превышают `pageSize`.
- **Критерии завершения:** схема применяется с нуля; Hibernate `validate` проходит; все ограничения проверены тестами.

### Этап C-3 — Безопасность, targets, endpoint inventory, identities

- **Цель:** аутентификация, авторизация по владельцу, управление target (с верификацией и scope-политикой), endpoints, IDOR-идентичностями.
- **Предварительные условия:** C-2, S-3.
- **Файлы:**
  - `.../security/SecurityConfig.kt` — цепочки: `/api/v1/**` (сессия + CSRF `CookieCsrfTokenRepository.withHttpOnlyFalse()`), `/internal/v1/**` (без сессии, `ServiceTokenFilter`), `/actuator/health/**` открыто, остальное закрыто
  - `.../security/ServiceTokenFilter.kt` — сравнение токена через `MessageDigest.isEqual`; сопоставляет токен с `moduleType`, кладёт в `SecurityContext` как `ModuleWorker(moduleType, workerId)`
  - `.../security/RateLimitFilter.kt` — ограничение `POST /api/v1/auth/login` (5/мин на IP + имя) и `POST /api/v1/scans` (10/мин на пользователя); `429 RATE_LIMITED` + `Retry-After`
  - `.../security/SecretCipher.kt` — `encrypt(plain: ByteArray): EncryptedSecret(keyVersion, bytes)`, `decrypt(...)`; AES-256-GCM, случайный 12-байтный nonce на запись, формат `nonce‖ciphertext‖tag`
  - `.../api/AuthController.kt` — `POST /auth/login`, `POST /auth/logout`, `GET /auth/me`
  - `.../api/TargetController.kt`, `EndpointController.kt`, `IdentityController.kt`, `AccessRuleController.kt`, `PolicyController.kt`
  - `.../application/TargetService.kt`, `TargetVerificationService.kt`, `ScopePolicyService.kt`, `EndpointService.kt` (+ `OpenApiImporter.kt`), `IdentityService.kt`, `AuditService.kt`
  - тесты: `SecurityIT`, `TargetAccessIT`, `ScopePolicyServiceTest`, `TargetVerificationServiceTest`, `SecretCipherTest`, `OpenApiImporterTest`, `IdentityServiceIT`
- **Реализация:**
  1. **Аутентификация.** `DaoAuthenticationProvider` + `DelegatingPasswordEncoder`; session fixation protection по умолчанию; `server.servlet.session.cookie.http-only=true`, `same-site=strict`, `secure` в docker/prod-профиле; тайм-аут сессии 30 мин; Spring Session JDBC. Единый ответ `401 UNAUTHENTICATED` на `/api/**` (без redirect на login-страницу). Первичный ADMIN создаётся `AdminBootstrap` из `SCANNER_BOOTSTRAP_ADMIN_USER/PASSWORD` только если таблица `users` пуста.
  2. **REST (все под `/api/v1`):**

  | Метод, путь                                      | Запрос → ответ                                                                                                                                                                         | Доступ                    | Статусы/ошибки                                                 |
  | ------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------- | -------------------------------------------------------------- |
  | `POST /auth/login`                               | `LoginRequest{username,password}` → `MeResponse{id,username,role,csrfToken}`                                                                                                           | все                       | 200, 401 `BAD_CREDENTIALS`, 429                                |
  | `POST /auth/logout`                              | — → 204                                                                                                                                                                                | аутентифицирован          | 204                                                            |
  | `GET /auth/me`                                   | → `MeResponse`                                                                                                                                                                         | аутентифицирован          | 200, 401                                                       |
  | `GET /targets`                                   | `?page,size` → `Page<TargetResponse>`                                                                                                                                                  | владелец; ADMIN видит все | 200                                                            |
  | `POST /targets`                                  | `CreateTargetRequest{name,baseUrl}` → `TargetResponse{status=PENDING_VERIFICATION, verification:{method:"FILE", path:"/.well-known/vulnscan-<token>.txt", expectedContent:"<token>"}}` | USER                      | 201, 400, 422 `SCOPE_VIOLATION`                                |
  | `POST /targets/{id}/verify`                      | — → `TargetResponse`                                                                                                                                                                   | владелец                  | 200, 409 `VERIFICATION_FAILED`, 404                            |
  | `POST /targets/{id}/approve`                     | — → `TargetResponse` (VERIFIED без проверки файла, для лаборатории)                                                                                                                    | ADMIN                     | 200; пишет `audit_events`                                      |
  | `DELETE /targets/{id}`                           | — → 204 (DISABLED, если есть сканы)                                                                                                                                                    | владелец                  | 204, 404                                                       |
  | `GET/POST /targets/{id}/endpoints`               | `CreateEndpointRequest{method,url,parameters[],idempotent,label}`                                                                                                                      | владелец                  | 201, 422 `SCOPE_VIOLATION` (URL вне host target), 409 дубликат |
  | `POST /targets/{id}/endpoints/import-openapi`    | `{document: <OpenAPI 3 JSON>, serverUrl}` → `ImportResult{created,skipped[]}`                                                                                                          | владелец                  | 200, 422                                                       |
  | `DELETE /targets/{id}/endpoints/{eid}`           | → 204                                                                                                                                                                                  | владелец                  |                                                                |
  | `GET/POST/DELETE /targets/{id}/identities`       | `CreateIdentityRequest{label,role,authType,authConfig,secret}` → `IdentityResponse{id,label,role,authType,hasSecret:true}` — секрет **никогда** не возвращается                        | владелец                  | 201, 409                                                       |
  | `POST /targets/{id}/identities/{iid}/test-login` | → `{ok, detail}`; Core запрашивает логин через `SafeHttpClient`                                                                                                                        | владелец                  | 200                                                            |
  | `GET/POST/DELETE /targets/{id}/access-rules`     | `AccessRuleRequest` (см. `resources[]` в B.8.2)                                                                                                                                        | владелец                  | 201, 422                                                       |
  | `GET /policies`                                  | → `PolicyResponse[]`                                                                                                                                                                   | аутентифицирован          |                                                                |
  | `GET /audit-events`                              | `?page,size`                                                                                                                                                                           | ADMIN                     |                                                                |
  3. **Контроль доступа к объектам.** Каждый сервис принимает `CurrentUser` и выполняет `findByIdAndOwnerId` (ADMIN — без фильтра по владельцу); чужой объект → `404 NOT_FOUND` (не 403, чтобы не раскрывать существование). Правило `OwnershipPolicy.requireAccess(user, ownerId)` — единственное место проверки.
  4. **`ScopePolicyService.validateTarget(baseUrl)`:** строгий разбор URL (`HttpUrl`), схема `http|https`, нет userinfo, порт ∈ `scope.allowedPorts`, host — DNS-имя или стандартный литерал IP; `SafeDns` резолв, каждый адрес через `IpPolicy` (приватные только если в `allowedPrivateCidrs`). Ошибка → `422 SCOPE_VIOLATION` с подкодом (`PRIVATE_ADDRESS`, `PORT_NOT_ALLOWED`, `BAD_URL`). `allowed_hosts` target по умолчанию = `[{host, ports:[port]}]`; расширяется только ADMIN-ом (отдельного endpoint в MVP нет — поле правится при `approve`).
  5. **`TargetVerificationService.verify`:** `GET <baseUrl>/.well-known/vulnscan-<token>.txt` через `SafeHttpClient`; совпадение тела с токеном → `VERIFIED`; иначе `409 VERIFICATION_FAILED`. Токен — 32 случайных байта, base64url.
  6. **Секреты identity.** `IdentityService.create`: `secret` (JSON `{"username":"…","password":"…"}` либо `{"token":"…"}`) → `SecretCipher.encrypt`; plaintext не логируется (поле в DTO помечено `@JsonIgnore` на выходе и исключено из `toString`); аудит `IDENTITY_CREATED`.
  7. **`OpenApiImporter`:** из `paths` создаёт endpoints (`GET/POST`), параметры `query`/`path`/`header` и верхнеуровневые свойства JSON-тела (`requestBody.content.application/json.schema.properties`) как `BODY_JSON`; `idempotent=true` только для `GET`; `$ref` не разворачивается глубже 1 уровня; пропущенное — в `skipped[]` с причиной.
  8. **`AuditService.record(action, subject)`** вызывается из: login success/failure, create/delete target, verify/approve, create/delete identity, выдачи секретов модулю, создания/отмены scan.

- **Интерфейсы:** HTTP B.9; `ScopePolicyService` используется в C-4; `SecretCipher` — в C-5 (internal identities).
- **Проверки:**
  - `SecurityIT`: запрос без сессии → 401; POST без CSRF → 403; неверный пароль 6 раз подряд → 429; `/internal/**` без токена → 401, с токеном другого модуля к чужому ресурсу → 403.
  - `TargetAccessIT`: пользователь B запрашивает target пользователя A → 404; ADMIN → 200.
  - `ScopePolicyServiceTest` (параметризованный, с подменяемым `DnsResolver`): `http://localhost`, `http://127.0.0.1`, `http://[::1]`, `http://169.254.169.254`, `http://2130706433`, `http://user:pw@host`, `ftp://host`, `http://host:22`, host → два адреса (один приватный), приватный адрес из allowlist → разрешено, публичный `93.184.216.34`-подобный (подставной) → разрешено.
  - `TargetVerificationServiceTest` (MockWebServer): верный токен → `VERIFIED`; неверный/404/редирект на другой host → `VERIFICATION_FAILED`.
  - `SecretCipherTest`: round-trip; два шифрования одного текста дают разный шифртекст; изменённый байт → исключение аутентификации; неверный `keyVersion` → понятная ошибка.
  - `IdentityServiceIT`: ответ API и все логи теста не содержат строку пароля (перехват через `OutputCaptureExtension`).
  - `OpenApiImporterTest`: фикстура с 3 путями (GET с query, POST с JSON-телом, путь с `$ref`) → ожидаемые endpoints.
- **Критерии завершения:** все эндпоинты таблицы работают по тестам; ни один ответ API не содержит секрет; чужие объекты недоступны; scope-тест-таблица полностью проходит.

### Этап C-4 — Создание сканирования и публикация заданий (outbox)

- **Цель:** `POST /api/v1/scans` атомарно создаёт `Scan`, `ScanJob`-ы и записи outbox; `OutboxRelay` надёжно публикует их в Kafka.
- **Предварительные условия:** C-3, S-2.
- **Файлы:**
  - `.../api/ScanController.kt` — `POST /scans`, `GET /scans`, `GET /scans/{id}`, `POST /scans/{id}/cancel`, `POST /scans/{id}/rerun`
  - `.../api/dto/CreateScanRequest.kt`, `ScanResponse.kt`, `ScanJobResponse.kt`, `ScanListItem.kt`
  - `.../application/ScanService.kt` — `create(user, req)`, `get`, `list`, `cancel`, `rerun`
  - `.../application/ScanJobFactory.kt` — собирает `ScanJobPayload` из target/endpoints/policy/identities
  - `.../application/ActiveScanLimiter.kt`
  - `.../messaging/KafkaTopics.kt` — константы имён топиков и `forModule(ModuleType)`
  - `.../messaging/OutboxEventFactory.kt` — `jobMessage(job, payload, correlationId): OutboxEventEntity`
  - `.../messaging/OutboxRelay.kt`
  - `.../messaging/KafkaConfig.kt` — producer (`acks=all`, `enable.idempotence=true`, `delivery.timeout.ms` ограничен), создание топиков через `NewTopic` (dev), consumer-фабрика
  - тесты: `ScanServiceIT`, `ScanJobFactoryTest`, `OutboxRelayIT`, `ActiveScanLimiterIT`
- **Реализация (алгоритм `ScanService.create`, соответствует шагам 1–7 из ТЗ 8.4):**
  1. Контроллер проверяет `@Valid CreateScanRequest`: `modules` не пуст и без дубликатов; `endpointIds` ≤ `jobs.maxEndpointsPerScan`; при `IDOR` — `idor.identityIds` ≥ 2 и существует ≥ 1 `access-rule`.
  2. `TargetService.requireVerified(user, targetId)`: владелец + `status=VERIFIED`, иначе `409 TARGET_NOT_VERIFIED`.
  3. `ScopePolicyService.revalidate(target, endpoints)`: повторный DNS-резолв и проверка `IpPolicy` на момент создания (адрес мог измениться после верификации) + все `endpoint.url` лежат под host target; `excludePrefixes` из запроса добавляются к политике target (убрать ограничение нельзя).
  4. `ActiveScanLimiter.checkAndLock(user)`: `SELECT count(*)` активных сканов пользователя и глобально внутри транзакции с `pg_advisory_xact_lock(hashtext('active-scans'))`; превышение → `429 TOO_MANY_ACTIVE_SCANS`.
  5. Транзакция: вставить `scans(status=PENDING, config=<снимок запроса+policy limits>)`; для каждого `moduleType` — `scan_jobs(status=PENDING, attempt=1, deadline_at=now+limits.maxDurationSec+queueTimeout)`; для каждого задания `ScanJobFactory.build` → `OutboxEventFactory.jobMessage` → `outbox_events(status=PENDING, topic=KafkaTopics.forModule(...), message_key=jobId)`. Размер payload > 900 КБ → `422 PAYLOAD_TOO_LARGE` до коммита. `audit_events(SCAN_CREATED)`.
  6. Ответ `202` с `ScanResponse` (статус `PENDING`).
  7. **`OutboxRelay`** (`@Scheduled(fixedDelayString="${scanner.outbox.pollMs:500}")`, один поток на инстанс): в транзакции `SELECT … FROM outbox_events WHERE status='PENDING' AND next_attempt_at<=now() ORDER BY created_at LIMIT 50 FOR UPDATE SKIP LOCKED`; для каждой записи `kafkaTemplate.send(topic,key,payload).get(5s)`; успех → `status=SENT, sent_at`, и если событие — `SCAN_JOB`, `scan_jobs.status=QUEUED, queued_at=now()` (`PENDING→QUEUED` через `JobStateMachine`); ошибка → `attempts+1`, `next_attempt_at = now + min(2^attempts, 60) с`. Повторная публикация возможна (падение между `send` и `commit`) — безопасно благодаря claim (D-3).
  8. **`rerun`:** создаёт **новый** `Scan` с тем же `config` (повторного запуска существующего scan нет); доступно только для терминальных сканов.
- **Интерфейсы:** Kafka `scan.jobs.<module>.v1`, формат B.8.2.
- **Проверки:**
  - `ScanServiceIT`: (1) валидный запрос → 202, в БД 1 scan, N jobs, N outbox-записей, **ничего** ещё не в Kafka до работы relay; (2) не `VERIFIED` target → 409; (3) endpoint чужого target → 404; (4) payload > лимита → 422 и в БД нет строк (откат); (5) третий активный скан одного пользователя → 429.
  - `ScanJobFactoryTest`: JSON задания проходит `ScanJobPayloadValidator` и схему `scan-job.v1.json`; для IDOR — в сообщении нет поля с секретом (поиск подстроки пароля в сериализованном JSON).
  - `OutboxRelayIT` (реальный Kafka): (1) запись `PENDING` → сообщение в топике, job `QUEUED`; (2) Kafka недоступен (остановить контейнер) → запись остаётся `PENDING`, `attempts` растёт; после старта — публикуется; (3) два параллельных relay (две экземпляра контекста) → каждое сообщение опубликовано ровно один раз из-за `SKIP LOCKED`; (4) падение после `send` до `commit` (инъекция исключения через тестовый хук) → сообщение публикуется повторно, но `scan_jobs` корректно переходит в `QUEUED` один раз.
  - `ActiveScanLimiterIT`: 5 параллельных запросов при лимите 2 → ровно 2 успеха.
- **Критерии завершения:** создание скана не теряет задания при сбое Kafka; повторная публикация не ломает состояние; лимиты работают.

### Этап C-5 — Internal API: claim, heartbeat, credentials; машина состояний

- **Цель:** модули могут захватить задание, удерживать lease, получить отмену и секреты.
- **Предварительные условия:** C-4.
- **Файлы:**
  - `.../domain/JobStatus.kt`, `JobStateMachine.kt` — `fun transition(from, to): Unit` по таблице B.7, иначе `IllegalStateTransition`
  - `.../domain/ScanStatusAggregator.kt`
  - `.../api/InternalJobController.kt` — `/internal/v1/jobs/{jobId}/claim|heartbeat|identities`
  - `.../application/JobControlService.kt` — `claim`, `heartbeat`, `identitiesFor`
  - `.../persistence/ScanJobRepository.kt` — методы `claim(jobId, attempt, workerId, leaseUntil): Int` (нативный `UPDATE … WHERE status='QUEUED' AND attempt=:attempt`), `extendLease(jobId, attempt, workerId, leaseUntil, requestsSent, endpointsDone): Int` (`WHERE status='RUNNING' AND worker_id=:w AND attempt=:a`)
  - тесты: `JobStateMachineTest`, `ScanStatusAggregatorTest`, `JobControlServiceIT`, `InternalApiSecurityIT`
- **Реализация:**
  1. `claim`: `rows==1` → читает `scans.cancel_requested`; если `true` — немедленно `RUNNING→CANCELLED` (в той же транзакции) и ответ `410 JOB_CANCELLED`; иначе `200 {leaseExpiresAt, heartbeatIntervalSec=10}`; `started_at=now()` при первом claim. `rows==0` → если job `CANCELLED` — 410; если не найден — 404; иначе 409 `JOB_NOT_CLAIMABLE`.
  2. `heartbeat`: `extendLease`; `rows==0` → `409 LEASE_LOST`; иначе читает `cancel_requested`; если `true` → job `RUNNING→CANCELLED`, ответ `{cancel:true}`; затем `ScanStatusAggregator` пересчитывает scan. `lease_expires_at = now + jobs.leaseSec`.
  3. `identitiesFor`: условия — `job.status=RUNNING`, `job.worker_id == X-Worker-Id`, `job.module_type == IDOR`, токен принадлежит IDOR-модулю; иначе 403. Расшифровка `SecretCipher`, ответ с `Cache-Control: no-store`; `audit_events(IDENTITY_SECRETS_ISSUED, job_id)`; не более 3 вызовов на задание (счётчик в `scan_jobs.stats`), далее 429.
  4. `ScanStatusAggregator` реализует таблицу B.7 как чистую функцию; вызывается из каждого места, меняющего статус задания.
- **Интерфейсы:** B.8.5.
- **Проверки:**
  - `JobStateMachineTest`: все пары (статус × статус) 6×6 — допустимые по B.7 проходят, остальные → исключение (параметризованная таблица).
  - `ScanStatusAggregatorTest`: ≥ 12 строк таблицы: (`COMPLETED,COMPLETED`)→`COMPLETED`; (`COMPLETED,FAILED`)→`PARTIALLY_COMPLETED`; (`FAILED,FAILED`)→`FAILED`; (`COMPLETED,CANCELLED` без cancel)→`PARTIALLY_COMPLETED`; (`CANCELLED,CANCELLED`+cancel)→`CANCELLED`; (`RUNNING,COMPLETED`+cancel)→`CANCELLING`; и т. д.
  - `JobControlServiceIT`: (1) два параллельных `claim` одного задания → один 200, один 409 (многопоточный тест); (2) `claim` с неверным `attempt` → 409; (3) после `cancel` scan: `heartbeat` → `cancel:true`, job `CANCELLED`; (4) `heartbeat` другим `workerId` → 409; (5) `identities` для XSS-токена → 403; для IDOR-токена чужого worker → 403; для верного → 200 и запись в `audit_events`.
  - `InternalApiSecurityIT`: без токена 401; токен XSS на `claim` задания IDOR → 403.
- **Критерии завершения:** конкурентный claim безопасен; отмена достигает модуля ≤ 1 heartbeat-интервала; секреты выдаются только владельцу lease.

### Этап C-6 — Приём результатов, findings, dashboard

- **Цель:** идемпотентное сохранение findings, завершение заданий, агрегация, API чтения.
- **Предварительные условия:** C-5, S-2.
- **Файлы:**
  - `.../messaging/JobEventsConsumer.kt` — `@KafkaListener(topics="scan.job-events.v1", groupId="core-results")`, `AckMode.MANUAL`
  - `.../application/JobResultService.kt` — `onFinding`, `onCompleted`, `onFailed` (каждый в одной транзакции вместе с дедупликацией)
  - `.../persistence/ProcessedMessageRepository.kt`, `FindingRepository.kt`
  - `.../api/FindingController.kt` — `GET /findings`, `GET /findings/{id}`, `PATCH /findings/{id}`
  - `.../api/DashboardController.kt` — `GET /dashboard/summary`
  - `.../application/FindingQueryService.kt`, `DashboardService.kt`
  - `.../api/dto/FindingListItem.kt`, `FindingDetailResponse.kt`, `UpdateFindingRequest.kt`, `DashboardSummary.kt`
  - тесты: `JobEventsConsumerIT`, `JobResultServiceTest`, `FindingApiIT`, `DashboardQueryIT`
- **Реализация:**
  1. **Consumer.** Парсинг envelope; ошибка парсинга/схемы → публикация в `scan.dlq.v1` и `ack`. Затем `JobResultService.handle(envelope)` внутри `@Transactional`: `INSERT INTO processed_messages(message_id,'core-results')`; при `DuplicateKeyException` — пропустить (дубликат), `ack`. Offset подтверждается **после** коммита транзакции.
  2. `onFinding`: найти job (нет → в DLQ с причиной `UNKNOWN_JOB`); принять finding от любого `attempt` ≤ текущего; статус job должен быть `RUNNING` или `COMPLETED` (поздний finding после завершения допустим, но увеличивает `stats`); `INSERT … ON CONFLICT (job_id, fingerprint) DO NOTHING`; `target_id`, `scan_id` берутся из job (не из сообщения); повторная проверка ограничений: `evidence` ≤ 16 КБ, `confidence` 1..99 — иначе finding отклоняется в DLQ.
  3. `onCompleted`: если `payload.attempt != job.attempt` → игнорировать; `RUNNING→COMPLETED` (`finished_at`, `stats`, `requests_sent`); сверка `stats.findingsCount` с `count(findings WHERE job_id)`; несовпадение → `result_integrity=INCOMPLETE` + запись в `audit_events(RESULT_INCOMPLETE)`; пересчёт статуса scan.
  4. `onFailed`: `attempt` не совпадает → игнор; `retryable && job.attempt < maxAttempts` → `RUNNING→QUEUED`, `attempt+1`, `worker_id=null`, новое outbox-сообщение (`messageId` новый, тот же `jobId`); иначе `RUNNING→FAILED`, `error_code/message` сохраняются (текст усекается до 500 символов).
  5. **Findings API.** `GET /findings?scanId&targetId&moduleType&vulnType&severity&verdict&minConfidence&triageStatus&endpoint&q&sort=createdAt,desc&page&size` (size ≤ 100, по умолчанию 25, сортировка по whitelist полей `createdAt,severity,confidence,url`; severity сортируется через `CASE`-выражение, а не по строке); реализация — `Specification` + проекция `FindingListItem` **без** `evidence`/`details` (тяжёлые jsonb не читаются в списке). Доступ: только findings сканов своего пользователя (`scans.started_by`), ADMIN — все. `GET /findings/{id}` — полный объект. `PATCH /findings/{id}` `{triageStatus}` — допустимо `OPEN↔FALSE_POSITIVE↔FIXED`; `audit_events(FINDING_TRIAGED)`.
  6. **Dashboard.** `GET /dashboard/summary` → `{openFindingsBySeverity:{CRITICAL..INFO}, findingsByModule, findingsByVerdict, activeScans, recentScans[5], avgScanDurationSec}`; запросы — агрегатные `GROUP BY` по индексам `(scan_id, severity)`/`(target_id, triage_status)`; ограничение «открытые findings последних N сканов пользователя» (N=50) для предсказуемой стоимости. Идеализированные числа из HTML-макета не используются.
  7. **Scan API чтение:** `GET /scans/{id}` возвращает `ScanResponse{status, createdAt, finishedAt, jobs[{id,moduleType,status,attempt,requestsSent,endpointsDone,errorCode,resultIntegrity}], findingCounts{bySeverity}, availableModules}`; список — keyset не нужен, пагинация offset (≤100).
- **Интерфейсы:** Kafka B.8.3–B.8.4; HTTP — OpenAPI.
- **Проверки:**
  - `JobEventsConsumerIT` (реальные PostgreSQL+Kafka): (1) `FINDING_REPORTED` ×1 → 1 строка; (2) то же сообщение ×2 (одинаковый `messageId`) → 1 строка; (3) два сообщения с разным `messageId`, но тем же `fingerprint` → 1 строка; (4) `JOB_COMPLETED` с `findingsCount=3`, в БД 2 → `result_integrity=INCOMPLETE`; (5) битый JSON → запись в DLQ, consumer продолжает работу; (6) падение транзакции (исключение в сервисе) → offset не подтверждён, сообщение обработано при повторе.
  - `JobResultServiceTest`: `JOB_FAILED(retryable=true)` при `attempt=1` → job `QUEUED`, `attempt=2`, создан outbox; при `attempt=2` → `FAILED`; `JOB_COMPLETED` с устаревшим `attempt` игнорируется; `JOB_FAILED(retryable=false)` → `FAILED` без outbox.
  - `FindingApiIT`: фильтры по каждому параметру; `size=1000` → 400 (или усечено до 100 — выбрать одно и зафиксировать тестом: **400 `PAGE_SIZE_TOO_LARGE`**); сортировка по `severity` даёт порядок `CRITICAL>HIGH>MEDIUM>LOW>INFO`; пользователь B не видит findings пользователя A; `PATCH` меняет только `triage_status`.
  - `DashboardQueryIT`: на фикстуре из 5 000 findings ответ ≤ 300 мс (порог — как регрессионный, фиксируется после первого измерения) и значения равны ручному подсчёту по SQL.
- **Критерии завершения:** повторная доставка любого события не меняет результат; пагинация/фильтры корректны; evidence не попадает в список.

### Этап C-7 — Watchdog, retry, отмена

- **Цель:** система самовосстанавливается при падении модуля, потере сообщений и зависших заданиях.
- **Предварительные условия:** C-6.
- **Файлы:**
  - `.../scheduling/StuckJobWatchdog.kt`
  - `.../application/JobRecoveryService.kt` — `recoverLostLease(job)`, `expireQueued(job)`, `expireDeadline(job)`
  - `.../application/ScanService.kt` — `cancel(user, scanId)`
  - тесты: `StuckJobWatchdogIT`, `ScanCancelIT`, `FailureMatrixE2E`
- **Реализация:**
  1. `StuckJobWatchdog` (`@Scheduled` каждые `jobs.watchdogMs=15000`), три запроса `… FOR UPDATE SKIP LOCKED LIMIT 20`:
     - `RUNNING AND lease_expires_at < now()` → `recoverLostLease`: если `attempt < maxAttempts` → `QUEUED`, `attempt+1`, outbox; иначе `FAILED(HEARTBEAT_LOST)`.
     - `QUEUED AND queued_at < now() - queueTimeout` → если `attempt < maxAttempts` → переопубликовать (`attempt+1`), иначе `FAILED(QUEUE_TIMEOUT)` (модуль недоступен).
     - `RUNNING AND deadline_at < now()` → `FAILED(DEADLINE_EXCEEDED)` без retry.
       После каждого изменения — `ScanStatusAggregator`.
  2. `cancel`: владелец/ADMIN; `scans.cancel_requested=true`, статус scan `CANCELLING`; `PENDING/QUEUED` jobs → `CANCELLED` (запись outbox `PENDING` для такого job помечается `status=SENT, headers.cancelled=true` — не публикуется); `RUNNING` jobs остаются до ближайшего heartbeat. Повторный вызов идемпотентен (200, без изменений). Терминальный scan → `409 SCAN_ALREADY_FINISHED`.
  3. Запоздалый результат после `FAILED/CANCELLED` job: findings сохраняются (они валидны), статус job не меняется.
- **Проверки:**
  - `StuckJobWatchdogIT` (время управляется тестовым `Clock`): (1) `RUNNING` с просроченным lease, `attempt=1` → `QUEUED`, `attempt=2`, в outbox новое сообщение; (2) то же при `attempt=2` → `FAILED(HEARTBEAT_LOST)`; (3) `QUEUED` старше `queueTimeout` → повтор/`QUEUE_TIMEOUT`; (4) два экземпляра watchdog одновременно → одно действие над заданием.
  - `ScanCancelIT`: отмена `PENDING` скана → все jobs `CANCELLED`, scan `CANCELLED`, ничего не опубликовано; отмена `RUNNING` → `CANCELLING` → после heartbeat `CANCELLED`; повторная отмена → 200; отмена завершённого → 409.
- **Критерии завершения:** убийство процесса модуля приводит к перезапуску задания ≤ `leaseSec + watchdogMs`; все ветки покрыты.

### Матрица отказов (используется как чек-лист интеграционных тестов `FailureMatrixE2E`)

| Сбой                                          | Обнаружение                                         | Реакция                                      | Итог                                      |
| --------------------------------------------- | --------------------------------------------------- | -------------------------------------------- | ----------------------------------------- |
| Падение Core после записи в БД, до публикации | запись outbox `PENDING`                             | `OutboxRelay` после рестарта публикует       | Задание не потеряно                       |
| Kafka недоступен при публикации               | исключение `send`                                   | backoff, `attempts++`                        | Публикация позже, UI показывает `PENDING` |
| Дубликат сообщения задания                    | второй claim → 409                                  | модуль не запускает                          | Одно выполнение                           |
| Модуль упал после claim                       | нет heartbeat                                       | watchdog: retry или `FAILED(HEARTBEAT_LOST)` | ≤ 2 запуска                               |
| Модуль не запущен                             | `queueTimeout`                                      | retry → `FAILED(QUEUE_TIMEOUT)`              | Scan `PARTIALLY_COMPLETED`/`FAILED`       |
| Дубликат результата                           | `processed_messages` / `UNIQUE(job_id,fingerprint)` | no-op                                        | Нет дублей findings                       |
| Потерян finding (DLQ)                         | `findingsCount` ≠ факт                              | `result_integrity=INCOMPLETE`                | Пользователь видит предупреждение         |
| Таймаут задания                               | `deadline_at`                                       | `FAILED(DEADLINE_EXCEEDED)`                  | Findings до таймаута сохранены            |
| Один модуль успешен, другой — нет             | агрегатор                                           | `PARTIALLY_COMPLETED`                        | Findings успешного модуля видны           |
| Отмена                                        | `cancel_requested`                                  | `410` на claim/`cancel:true` на heartbeat    | `CANCELLED`                               |

### Этап C-8 — Закрепление: ограничения, наблюдаемость, e2e

- **Цель:** защитные ограничения Core и сквозной тест без UI.
- **Предварительные условия:** C-7, N-1 (compose), модули хотя бы в виде `EchoEngine`.
- **Файлы:** `.../security/RequestSizeFilter.kt` (отклоняет `Content-Length` > 1 МБ → `413`; для import-openapi лимит 2 МБ), `.../observability/Metrics.kt`, `logback-spring.xml` (маскирование через `Redactor`-совместимый converter), `core-backend/src/test/.../e2e/ScanLifecycleE2E.kt`, `LogLeakageTest.kt`.
- **Реализация:**
  1. Метрики: `scanner.scans.created`, `scanner.jobs.transitions{from,to}`, `scanner.outbox.pending` (gauge), `scanner.watchdog.recoveries{kind}`.
  2. Readiness: `KafkaProducerHealthIndicator` (метаданные топика) + БД.
  3. `RetentionCleaner` (ежедневно): `processed_messages` > 7 суток, `outbox_events SENT` > 7 суток, истёкшие сессии.
  4. Логи: перед записью `Authorization`/`Cookie` не логируются вообще; тело запросов `/auth/login` и identity-endpoint не логируется.
- **Проверки:** `ScanLifecycleE2E` (Testcontainers PostgreSQL+Kafka, `EchoEngine`-воркер, запущенный в тесте): создать target→verified→endpoint→scan XSS+SQLI → оба `COMPLETED` → scan `COMPLETED`, findings сохранены; сценарий «SQLI-воркер публикует `JOB_FAILED(retryable=false)`» → scan `PARTIALLY_COMPLETED`. `LogLeakageTest`: выполнить полный сценарий с identity-паролем `S3cretPassw0rd!` и проверить, что строка отсутствует в логах и во всех JSON-ответах.
- **Критерии завершения:** Core проходит `./gradlew :core-backend:check` с интеграционными тестами; e2e без UI зелёный; ни одного секрета в логах.

---

## Проект 1. Frontend (`frontend`)

Выбор: React + Vite + TypeScript (обоснование — D-11). Состав библиотек (версии фиксируются в `package-lock.json` на этапе S-1): React Router (маршрутизация), TanStack Query (серверное состояние), React Hook Form + Zod (формы и валидация), TanStack Table (таблицы), Recharts (диаграммы dashboard), компонентная библиотека MUI, `openapi-typescript` (типы API), Vitest + React Testing Library + MSW (unit/component/integration), Playwright (e2e).

Клиентского глобального состояния (Redux и т. п.) нет: серверные данные — в TanStack Query, состояние фильтров — в URL (`useSearchParams`), сессия — в `AuthContext`.

### Структура каталогов

```
frontend/
  src/
    api/
      client.ts            # fetch-обёртка: credentials:'include', CSRF, корреляция, нормализация ошибок
      errors.ts            # ApiError, isApiError, коды из B.8.6
      generated/schema.d.ts# из contracts/openapi/core-api.v1.json
      endpoints/           # targets.ts, scans.ts, findings.ts, identities.ts, dashboard.ts, auth.ts
    auth/                  # AuthContext.tsx, RequireAuth.tsx, RequireRole.tsx
    components/            # StatusChip, SeverityChip, VerdictBadge, DataTable, FilterBar, ErrorState, EmptyState, EvidenceViewer, ConfirmDialog
    features/
      targets/ scans/ findings/ identities/ dashboard/   # страницы + хуки + формы фичи
    hooks/                 # useDebounce, usePollingInterval
    routes.tsx
    config.ts              # чтение import.meta.env (только VITE_API_BASE_URL, по умолчанию '/api/v1')
    main.tsx
  tests/                   # setup.ts (MSW), mocks/handlers.ts, e2e/*.spec.ts
  nginx.conf               # прокси /api → core-backend, security headers
```

Правила: компоненты `components/` не вызывают API; страницы получают данные только через хуки `features/*/use*.ts`; типы API — только из `generated/schema.d.ts`, ручных копий нет (решает требование «одинаковые модели frontend/backend»).

### Этап F-1 — Каркас, API-клиент, аутентификация

- **Цель:** приложение с логином, защищёнными маршрутами и единым API-клиентом.
- **Предварительные условия:** S-1; OpenAPI-снимок из C-3 (до этого — handwritten mock-спецификация в `contracts/openapi/`, сверяется на C-3).
- **Файлы:** `src/api/client.ts`, `errors.ts`, `src/auth/AuthContext.tsx`, `RequireAuth.tsx`, `RequireRole.tsx`, `src/features/auth/LoginPage.tsx`, `src/routes.tsx`, `src/components/ErrorState.tsx`, `nginx.conf`.
- **Реализация:**
  1. `client.ts`: `request<T>(method, path, body?)`: `fetch(..., {credentials:'include'})`; на `POST/PATCH/DELETE` добавляет `X-XSRF-TOKEN` из cookie `XSRF-TOKEN`; добавляет `X-Correlation-Id` (UUID на запрос); при статусе `>=400` разбирает тело по B.8.6 в `ApiError{status, code, message, fieldErrors, correlationId}`; сетевая ошибка → `ApiError{code:'NETWORK_ERROR'}`.
  2. Глобальный обработчик: `401` (кроме `/auth/login`) → очистка `AuthContext`, навигация на `/login?next=<текущий путь>`, показ уведомления «Сессия истекла»; `403` → страница «Нет доступа»; `429` → уведомление с `Retry-After`.
  3. `AuthContext`: при старте `GET /auth/me`; `RequireAuth` показывает spinner до ответа, затем `Outlet` или `Navigate`; `RequireRole role="ADMIN"` для админских страниц.
  4. `LoginPage`: форма (React Hook Form + Zod: `username` 1–64 символа, `password` 1–128), состояния: ввод, отправка (кнопка disabled), ошибка `BAD_CREDENTIALS` (сообщение под формой), `429`.
  5. `nginx.conf`: проксирует `/api/` на `core-backend:8080`; заголовки `Content-Security-Policy: default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; frame-ancestors 'none'`, `X-Content-Type-Options: nosniff`, `Referrer-Policy: no-referrer`; `/internal/` не проксируется.
- **Интерфейсы:** `/api/v1/auth/*`.
- **Проверки:**
  - `client.test.ts` (MSW): CSRF-заголовок только на изменяющих методах; тело ошибки → `ApiError` с кодом и `fieldErrors`; недоступный сервер → `NETWORK_ERROR`.
  - `RequireAuth.test.tsx`: без сессии → редирект на `/login?next=/scans`; с сессией — рендер; роль USER на ADMIN-маршруте → «Нет доступа».
  - `LoginPage.test.tsx`: пустая форма → ошибки Zod; успешный вход → переход на `next`; `401 BAD_CREDENTIALS` → сообщение, поле пароля очищено; во время отправки кнопка заблокирована (нет двойной отправки).
  - `session-expiry.test.tsx`: любой запрос возвращает 401 → пользователь на `/login`, `next` сохранён.
- **Критерии завершения:** вход/выход работают против Core (или MSW); все перечисленные тесты зелёные; в коде нет `localStorage`/`sessionStorage` для токенов.

### Этап F-2 — Targets, endpoints, identities

- **Цель:** страницы управления разрешёнными целями, их endpoint-ами и IDOR-идентичностями.
- **Предварительные условия:** F-1, C-3.
- **Файлы:** `src/features/targets/TargetsPage.tsx`, `TargetFormDialog.tsx`, `TargetDetailPage.tsx` (вкладки «Endpoints», «Identities», «Access rules»), `EndpointTable.tsx`, `EndpointFormDialog.tsx`, `OpenApiImportDialog.tsx`, `src/features/identities/IdentityFormDialog.tsx`, `AccessRuleFormDialog.tsx`, `useTargets.ts`, `useEndpoints.ts`, `useIdentities.ts`.
- **Реализация (по страницам):**
  - **Targets (`/targets`)**: данные — `GET /targets`; таблица: имя, `baseUrl`, статус-чип (`PENDING_VERIFICATION`/`VERIFIED`/`DISABLED`), действия «Verify», «Open». Форма создания: `name` (1–80), `baseUrl` (Zod: `http|https`, без userinfo — подсказка клиентская, решение принимает сервер); при `201` показать инструкцию верификации (путь и ожидаемое содержимое, кнопка «Копировать»); при `422 SCOPE_VIOLATION` — сообщение по подкоду (`PRIVATE_ADDRESS`: «Адрес не разрешён политикой сервера»). Состояния: загрузка (skeleton), пусто («Добавьте первую цель»), ошибка (`ErrorState` с «Повторить»). Кнопка «Verify» → `POST /targets/{id}/verify`; `409` → «Файл не найден или не совпал».
  - **Target detail (`/targets/:id`)**: Endpoints — таблица, добавление (метод, URL, параметры как динамический список `name/location/sampleValue`, флаг `idempotent`), импорт OpenAPI (загрузка JSON-файла ≤ 2 МБ, показ `created/skipped`). Identities — таблица без секретов (`hasSecret`), форма создания (поле секрета `type="password"`, после отправки очищается и не хранится в состоянии страницы), кнопка «Test login» → результат. Access rules — форма «ресурс → владелец → ожидаемый доступ других».
- **Интерфейсы:** endpoints из таблицы C-3.
- **Проверки:** `TargetFormDialog.test.tsx` (валидация URL, отображение `SCOPE_VIOLATION`); `TargetsPage.test.tsx` (loading/empty/error/данные; Verify-успех и 409); `EndpointFormDialog.test.tsx` (динамические параметры, дубликат → 409 как ошибка поля); `IdentityFormDialog.test.tsx` (секрет не появляется в DOM после отправки; в запросе MSW он есть, в ответе — нет); `OpenApiImportDialog.test.tsx` (файл > 2 МБ отклонён на клиенте).
- **Критерии завершения:** весь CRUD работает против Core; секрет нигде не отображается повторно.

### Этап F-3 — Создание сканирования и просмотр статуса

- **Цель:** пользовательский сценарий «выбрать цель → запустить → наблюдать».
- **Предварительные условия:** F-2, C-4, C-6 (чтение scan).
- **Файлы:** `src/features/scans/NewScanPage.tsx`, `ModuleSelector.tsx`, `ScopeWarning.tsx`, `ScansPage.tsx`, `ScanDetailPage.tsx`, `JobCard.tsx`, `useCreateScan.ts`, `useScan.ts` (polling), `scanSchema.ts` (Zod).
- **Реализация:**
  1. **`/scans/new`**: шаги в одной форме: (1) выбор target — только `VERIFIED` (остальные disabled с подсказкой «не подтверждён»); (2) выбор модулей (чекбоксы XSS/SQLI/IDOR); IDOR недоступен, если у target < 2 identities или нет access rules — причина показывается рядом; (3) выбор endpoints (таблица с чекбоксами, «выбрать все», лимит 200 — счётчик); (4) политика (`LIGHT/NORMAL`); (5) дополнительные `excludePrefixes`; (6) параметры модулей: SQLI `timeBased` (по умолчанию выкл. с предупреждением «увеличивает нагрузку»), XSS `stored`; (7) блок `ScopeWarning`: показывает `baseUrl`, список excluded, лимиты политики (`maxRequests`, `rps`), текст «Сканирование отправляет тестовые запросы на указанный адрес. Продолжайте только для систем, на проверку которых у вас есть разрешение»; чекбокс подтверждения обязателен для активации кнопки.
  2. Отправка `POST /scans` → при `202` переход на `/scans/:id`. Ошибки: `409 TARGET_NOT_VERIFIED`, `422 SCOPE_VIOLATION`, `429 TOO_MANY_ACTIVE_SCANS` («Дождитесь завершения активных сканирований»), `422 PAYLOAD_TOO_LARGE` — сообщения по `code`.
  3. **`/scans/:id`**: `useScan` с `refetchInterval` 2 с пока статус ∈ `PENDING|RUNNING|CANCELLING`, затем останавливается (функция `refetchInterval: q => isActive(q.state.data?.status) ? 2000 : false`); при ошибке сети — интервал растёт до 10 с. Отображение: общий статус-чип, прогресс (`endpointsDone` по заданиям), карточка на каждый job (модуль, статус, попытка, `requestsSent`, `errorCode` на человекочитаемом языке); баннер для `PARTIALLY_COMPLETED` («XSS завершён, SQLi — ошибка `QUEUE_TIMEOUT`: модуль недоступен; результаты XSS ниже»); предупреждение при `resultIntegrity=INCOMPLETE`; кнопки «Отменить» (при активном) и «Повторить» (при терминальном → `POST /scans/:id/rerun`), ссылка «Все findings этого скана».
  4. **`/scans`**: таблица (дата, target, модули, статус, число findings, длительность), сортировка по дате, пагинация (`page`/`size` в URL).
- **Проверки:** `NewScanPage.test.tsx` (target не `VERIFIED` недоступен; IDOR заблокирован без identities; кнопка запуска disabled до подтверждения; `payload` запроса соответствует B.9; `429` → сообщение); `ScanDetailPage.test.tsx` (polling останавливается на `COMPLETED`; для `PARTIALLY_COMPLETED` виден баннер; отмена → вызов `POST /cancel` и статус `CANCELLING`; `INCOMPLETE` показывает предупреждение); `ScansPage.test.tsx` (пагинация меняет `page` в URL и запрос); `scan-launch.integration.test.tsx` (MSW: логин → создание скана → поллинг `RUNNING → COMPLETED` → переход к findings).
- **Критерии завершения:** сценарий проходит на MSW и на реальном Core; frontend не позволяет выбрать неподтверждённый target, но **не** является единственной защитой (серверные тесты C-4).

### Этап F-4 — Findings, evidence, dashboard

- **Цель:** безопасный просмотр результатов.
- **Предварительные условия:** F-3, C-6.
- **Файлы:** `src/features/findings/FindingsPage.tsx`, `FindingFilters.tsx`, `FindingDetailPage.tsx`, `components/EvidenceViewer.tsx`, `components/HighlightedText.tsx`, `components/SeverityChip.tsx`, `VerdictBadge.tsx`, `src/features/dashboard/DashboardPage.tsx`, `SeverityChart.tsx`, `useFindings.ts`, `useFinding.ts`, `useDashboard.ts`.
- **Реализация:**
  1. **`/findings`**: фильтры (модуль, `vulnType`, `severity` (мульти), `verdict`, `minConfidence` слайдер, `triageStatus`, `endpoint` (подстрока), `scanId` из URL) синхронизированы с `useSearchParams`; изменение фильтра сбрасывает `page=0`; поиск `q` с debounce 400 мс; сортировка по `createdAt|severity|confidence|url`; `size` ∈ {25, 50, 100}; пагинация серверная; при `totalElements > 10 000` показ подсказки «Уточните фильтры». Строка: severity-чип, verdict-бейдж, `title`, `method url`, параметр, confidence, статус triage, ссылка на scan.
  2. **`/findings/:id`**: заголовок (title, severity, verdict, confidence, CWE), описание (`summary`), параметры (метод, URL, параметр), вкладки **Evidence** / **Details** / **Recommendation**; кнопки triage («False positive», «Fixed», «Reopen») → `PATCH`. Ссылки: на scan, на target, на endpoint (`/targets/:id?endpoint=...`), к «похожим» (`?fingerprint`-фильтр не нужен — переход по `vulnType+url`).
  3. **`EvidenceViewer`**: показывает `request`/`response` как **текст** в `<pre>` через React-узлы; `HighlightedText({text, highlights})` режет строку по смещениям `highlights[{start,end}]` и оборачивает фрагменты в `<mark>` — смещения клиентом валидируются (`0 ≤ start < end ≤ text.length`, непересекающиеся), невалидные игнорируются; никакого `dangerouslySetInnerHTML`, `innerHTML`, `eval`, `new Function`; ссылки на `url` показываются как текст, а не `<a href>`, чтобы клик не открывал цель (есть кнопка «Копировать URL»). Длинные тела сворачиваются (первые 2 КБ + «Показать всё»). Правило ESLint `react/no-danger: error` + `no-restricted-syntax` на `innerHTML`.
  4. **Dashboard (`/`)**: плитки (`openFindingsBySeverity`, активные сканы, среднее время), столбчатая диаграмма по severity (Recharts, цвет + подпись значений — не только цвет), таблица последних 5 сканов; пустое состояние при отсутствии данных; данные — `GET /dashboard/summary`, обновление раз в 15 с на открытой вкладке.
- **Проверки:**
  - `EvidenceViewer.test.tsx` (**ключевой тест безопасности**): evidence с `<img src=x onerror=window.__pwned=1>`, `<script>window.__pwned=1</script>`, `"><svg onload=...>`, `javascript:alert(1)` → в DOM нет элементов `img/script/svg` внутри viewer, `window.__pwned` не определён, текст присутствует буквально.
  - `HighlightedText.test.tsx`: корректные смещения; смещения за пределами/пересекающиеся игнорируются без исключения; кириллица и surrogate pairs (смещения в UTF-16 code units — договорённость с модулями, зафиксирована в B.8.3; `EvidenceBuilderTest` (X-5) проверяет это на строке с эмодзи; описание индексов — в B.8.3).
  - `FindingFilters.test.tsx`: каждый фильтр → корректный query string; смена фильтра сбрасывает страницу; сброс фильтров очищает URL.
  - `FindingsPage.test.tsx`: пагинация (следующая/предыдущая страница → запрос с нужным `page`), сортировка, пустой результат («Ничего не найдено, измените фильтры»), ошибка API с «Повторить».
  - `FindingDetailPage.test.tsx`: triage-кнопки вызывают `PATCH` и обновляют бейдж; `404` → страница «Finding не найден».
  - `DashboardPage.test.tsx`: значения плиток соответствуют MSW-ответу; пустое состояние.
- **Критерии завершения:** на стенде с реальными findings все данные видны; тест с вредоносным evidence проходит; lint-правила включены в CI.

### Этап F-5 — Отложенное и доводка

- **Цель:** страница ошибок/истории и e2e.
- **Предварительные условия:** F-4, N-1.
- **Файлы:** `src/features/scans/ScanJobsTable.tsx` (история попыток job-а: `attempt`, `errorCode`, `workerId`), `src/features/audit/AuditPage.tsx` (ADMIN), `tests/e2e/scan-flow.spec.ts`, `tests/e2e/negative-scope.spec.ts`, `tests/e2e/session-expiry.spec.ts`.
- **Реализация:** `AuditPage` — таблица `GET /audit-events` с пагинацией, только ADMIN; Playwright-сценарии работают против docker-compose профиля `lab`.
- **Проверки (e2e):** (1) логин → target `lab-app` VERIFIED (через ADMIN approve) → scan XSS → видны findings из ground-truth; (2) попытка создать target `http://127.0.0.1:5432` → сообщение «Адрес не разрешён политикой сервера»; (3) принудительное удаление cookie сессии → следующий клик → `/login`.
- **Критерии завершения:** три e2e-сценария зелёные в CI на compose-окружении.

---

## Проект 3. XSS Detection Module (`xss-module`)

Пакет `io.vulnscan.xss`. Модуль — Spring Boot-приложение; Kafka-listener, claim, heartbeat, публикация результатов и лимиты — из `scanner-kit` (S-4). Модуль реализует только `XssDetectionEngine : DetectionEngine`.

**Что покрывает, а что нет (фиксируется в тексте диплома).**

| Вид                                                             | MVP                   | Комментарий                                          |
| --------------------------------------------------------------- | --------------------- | ---------------------------------------------------- |
| Reflected XSS (GET/POST form/JSON параметры)                    | да                    | этапы X-1…X-5                                        |
| Stored XSS (second-order) по явно заданным страницам наблюдения | да, X-6               | только если пользователь указал `observeEndpointIds` |
| DOM-based XSS                                                   | нет                   | требует разбора JS / браузера; ограничение R-3       |
| Исполнение JavaScript в браузере                                | нет (опционально X-7) | решение D-9                                          |

**Различие «отражение» ≠ «уязвимость».** Модуль различает четыре уровня наблюдения, и только последние два порождают finding:

| Уровень                    | Признак                                                                                                           | Результат                                     |
| -------------------------- | ----------------------------------------------------------------------------------------------------------------- | --------------------------------------------- |
| L0 Reflection              | канарейка найдена в ответе                                                                                        | счётчик `reflections` в stats, **не finding** |
| L1 Neutralized             | канарейка есть, но опасные символы закодированы (`&lt;`, `\x3c`, `%3C`)                                           | не finding; `reflectionsNeutralized++`        |
| L2 Character survival      | в данном контексте не закодированы символы, нужные для выхода из контекста                                        | `SUSPECTED` (`LOW`)                           |
| L3 Structural injection    | разбор ответа (DOM-парсер / JS-лексер) показывает, что внедрённая разметка/атрибут/код **стала частью структуры** | `LIKELY` (`MEDIUM`)                           |
| L4 Browser execution (X-7) | маркер выполнен в headless-браузере                                                                               | `CONFIRMED` (`HIGH`)                          |

### Этап X-1 — Каркас модуля и валидация задания

- **Цель:** запускаемый сервис, принимающий `SCAN_JOB` типа `XSS` и завершающий его `JOB_COMPLETED` с пустым результатом.
- **Предварительные условия:** S-4.
- **Файлы:** `xss-module/src/main/kotlin/io/vulnscan/xss/XssModuleApplication.kt`; `.../XssDetectionEngine.kt`; `.../config/XssProperties.kt` (`worker.*` из `WorkerProperties` + `xss.maxParamsPerEndpoint=10`, `xss.repeatReflectionCheck=true`); `.../job/XssJobValidator.kt`; `xss-module/src/main/resources/application.yml`; `xss-module/Dockerfile`; тесты `XssJobValidatorTest.kt`, `XssModuleBootIT.kt`.
- **Реализация:**
  1. `XssJobValidator.validate(job)`: `moduleOptions.xss` присутствует; `reflected || stored`; каждый endpoint содержит хотя бы один параметр с `location ∈ {QUERY, BODY_FORM, BODY_JSON, PATH}` (параметры `HEADER`/`COOKIE` в MVP пропускаются с `skipped.reason=UNSUPPORTED_LOCATION`); ошибка → `JOB_FAILED(INVALID_JOB, retryable=false)`.
  2. `XssDetectionEngine.run(ctx)`: цикл по endpoint → `ParameterPlanner` → `ReflectedXssDetector`; между endpoint и перед каждой пробой — проверка `ctx.isCancelled()` (внутри `BudgetedHttp` уже встроена); в конце `ctx.stats` заполняется.
  3. `Dockerfile`: multi-stage, непривилегированный пользователь, `ENTRYPOINT` на jar; healthcheck через actuator.
- **Интерфейсы:** `DetectionEngine` (S-4).
- **Проверки:** `XssJobValidatorTest` (job без `xss`-опций; `reflected=false,stored=false`; endpoint только с HEADER-параметрами); `XssModuleBootIT` (Testcontainers Kafka + MockWebServer-Core: валидный job с пустым списком проб → `JOB_COMPLETED`).
- **Критерии завершения:** сервис проходит весь цикл «claim → complete» на заглушке.

### Этап X-2 — Планирование проб и канарейки

- **Цель:** детерминированный и ограниченный по бюджету план запросов.
- **Предварительные условия:** X-1.
- **Файлы:** `.../plan/ParameterPlanner.kt` (`fun plan(endpoints, limits): List<ProbeTarget>`), `.../plan/ProbeTarget.kt` (`endpoint`, `parameter`, `estimatedRequests`), `.../canary/CanaryFactory.kt`, `.../canary/CanaryRegistry.kt`, `.../http/RequestBuilder.kt` (подстановка значения параметра в запрос по `location`), тесты `ParameterPlannerTest.kt`, `CanaryFactoryTest.kt`, `RequestBuilderTest.kt`.
- **Реализация:**
  1. `CanaryFactory.next()`: `"vsx" + 10` символов `[a-z0-9]` из `SecureRandom` (≈ 51 бит) — не должен встречаться в исходных данных; `CanaryRegistry` хранит `canary → (endpointId, paramName, location)` в `ConcurrentHashMap` в пределах job; при коллизии генерируется новое.
  2. `ParameterPlanner`: бюджет на параметр = **5** запросов (reflection ×2, character survival ×1, structure proof ×≤2); общее `Σ ≤ limits.maxRequests`; при нехватке бюджета параметры отбираются по приоритету: `QUERY` > `BODY_FORM` > `BODY_JSON` > `PATH`, затем по порядку; пропущенные попадают в `stats.endpointsSkipped(reason=BUDGET)` — без ошибки.
  3. `RequestBuilder.build(endpoint, parameter, value)`: URL-кодирует значение; для `BODY_JSON` подставляет значение в строковое поле по JSON-пути, остальные поля берутся из `sampleValue`; `Content-Type` из endpoint; для `PATH` заменяет сегмент `{name}`.
- **Проверки:** `CanaryFactoryTest` (формат, 10 000 значений без коллизий, значение не содержит спецсимволов); `ParameterPlannerTest` (бюджет 12 при 5 параметрах → выбраны 2 параметра по приоритету, остальные — `BUDGET`); `RequestBuilderTest` (QUERY/FORM/JSON/PATH: значение попадает ровно в нужное место, спецсимволы кодируются, остальные параметры сохраняют `sampleValue`).
- **Критерии завершения:** план не превышает `maxRequests` ни на одном тесте; построенные запросы корректны для всех `location`.

### Этап X-3 — Обнаружение отражения и классификация контекста

- **Цель:** надёжно находить канарейку и определять синтаксический контекст.
- **Предварительные условия:** X-2.
- **Файлы:** `.../analysis/ReflectionLocator.kt`, `.../analysis/HtmlLexer.kt`, `.../analysis/HtmlContextClassifier.kt`, `.../analysis/JsStringLexer.kt`, `.../analysis/ReflectionContext.kt` (enum), тесты `ReflectionLocatorTest.kt`, `HtmlLexerTest.kt`, `HtmlContextClassifierTest.kt`, `JsStringLexerTest.kt`.
- **Реализация:**
  1. `ReflectionLocator.find(body, canary): List<Reflection(offset, encoding)>`: поиск «как есть» и в закодированных формах канарейки (для буквенно-цифровой канарейки кодирование её самой не меняет, поэтому дополнительно ищется её **окружение** в X-4); возвращает все вхождения.
  2. `ReflectionContext` = `HTML_TEXT`, `HTML_COMMENT`, `ATTR_DOUBLE_QUOTED`, `ATTR_SINGLE_QUOTED`, `ATTR_UNQUOTED`, `ATTR_URL` (значение `href|src|action|formaction`), `SCRIPT_CODE`, `SCRIPT_STRING_DQ`, `SCRIPT_STRING_SQ`, `SCRIPT_STRING_TEMPLATE`, `STYLE`, `RCDATA` (`<textarea>`, `<title>`), `TAG_NAME_OR_ATTR_NAME`, `UNKNOWN`.
  3. `HtmlLexer` — собственный конечный автомат по HTML5 токенизации (состояния: data, tag open, tag name, before/after attribute name, attribute name, attribute value (double/single/unquoted), comment, `script`/`style`/`textarea`/`title` raw-text до закрывающего тега). Метод `stateAt(body, offset): LexState(context, tagName?, attrName?)`. Не строит DOM — только состояние в позиции смещения. Регистронезависимость тегов; обработка `<!-- -->` и `<![CDATA[`.
  4. Для `SCRIPT_CODE` — `JsStringLexer.stateAt(scriptText, offsetInScript)`: распознаёт строки `'…'`, `"…"`, шаблонные `` `…` `` (без `${}` вложенности в MVP → `UNKNOWN`), комментарии `//`, `/* */`, регулярные литералы не разбираются (при неоднозначности `UNKNOWN`).
  5. Если у ответа `Content-Type` не `text/html`/`application/xhtml+xml` — контексты не вычисляются; результат `NON_HTML_RESPONSE` (отражение в `application/json`/`text/plain` с `nosniff` не создаёт finding; без `nosniff` и с `text/plain` — `INFO`-finding не создаётся в MVP).
- **Интерфейсы:** используется детектором X-4.
- **Проверки:** `HtmlLexerTest` — таблица из ≥ 25 строк «HTML-фрагмент с маркером `§` → ожидаемый контекст», в т. ч.: `<p>§</p>`→`HTML_TEXT`; `<a href="§">`→`ATTR_URL`; `<input value="§">`→`ATTR_DOUBLE_QUOTED`; `<input value='§'>`→`ATTR_SINGLE_QUOTED`; `<input value=§>`→`ATTR_UNQUOTED`; `<!-- § -->`→`HTML_COMMENT`; `<script>var a="§";</script>`→`SCRIPT_STRING_DQ`; `<script>var a=§;</script>`→`SCRIPT_CODE`; `<textarea>§</textarea>`→`RCDATA`; `<p title=x data-a="§>">` (закрывающий `>` внутри кавычек); `<SCRIPT>` в верхнем регистре; незакрытый атрибут; `<style>§</style>`; `<p §>`→`TAG_NAME_OR_ATTR_NAME`. `JsStringLexerTest` — экранированные кавычки `"a\"§"` остаются внутри строки; комментарий `// "§`; `ReflectionLocatorTest` — 0/1/N вхождений; пересекающиеся вхождения не дублируются.
- **Критерии завершения:** ≥ 95 % строк таблицы классификации проходят; для остальных `UNKNOWN`, но **никогда** неверный «уверенный» контекст (проверяется отдельным тестом «неизвестное не классифицируется как HTML_TEXT»).

### Этап X-4 — Выживание символов и структурное подтверждение

- **Цель:** разделить L1/L2/L3.
- **Предварительные условия:** X-3.
- **Файлы:** `.../detect/ReflectedXssDetector.kt`, `.../detect/CharacterSurvivalProbe.kt`, `.../detect/StructureProof.kt` (интерфейс `proof(context, canary, body): ProofResult`), `.../detect/HtmlDomProof.kt` (jsoup-парсер), `.../detect/JsStringProof.kt`, `.../detect/ProbeLibrary.kt` (шаблоны проб по контексту, конфигурируемые), тесты `CharacterSurvivalProbeTest.kt`, `HtmlDomProofTest.kt`, `JsStringProofTest.kt`, `ReflectedXssDetectorTest.kt`.
- **Реализация (последовательность для одного `ProbeTarget`):**
  1. **Шаг 1 — reflection.** Отправить `canary`; найти вхождения; 0 → завершить параметр (`reflections=0`). Если ответ нестабилен (повтор того же запроса, `repeatReflectionCheck=true`, даёт другое число вхождений) → пометка `unstable`, confidence −10.
  2. **Шаг 2 — character survival.** Отправить `canary + "<>\"'`/" + canary`(спецсимволы **не** образуют исполняемой нагрузки). Для каждого вхождения взять область между метками`canary…canary`; определить, какие из `< > " ' \` /`пришли без кодирования (сравнение буквальных символов с HTML-сущностями`&lt;`, `&#x3c;`, `&#60;`, JS-escape `\x3c`, `<`, URL-escape `%3c`). Результат `survivingChars`. Выход из контекста возможен, если: `HTML_TEXT`/`RCDATA`— выживает`<`; `ATTR_DOUBLE_QUOTED`—`"`; `ATTR_SINGLE_QUOTED`—`'`; `ATTR_UNQUOTED`— пробел или`>`; `SCRIPT_STRING_DQ/SQ`— соответствующая кавычка не экранирована`\`; `ATTR_URL` — вариант не доказывается без браузера → максимум `SUSPECTED`/`INFO`. Если требуемых символов нет → L1 Neutralized, **нет finding**.
  3. **Шаг 3 — structural proof** (только если шаг 2 показал возможность выхода). Пробы без выполнения кода: `HTML_TEXT` — `canary<vsxtag id=canary>`; `ATTR_DOUBLE_QUOTED` — `canary" vsxattr="canary`; `ATTR_SINGLE_QUOTED` — аналог с `'`; `SCRIPT_STRING_*` — `canary";vsxvar=1;//` (для `SQ` соответствующая кавычка). Проверки: `HtmlDomProof` парсит ответ jsoup-ом и ищет **элемент** `vsxtag` с `id=canary` (для атрибута — элемент, у которого есть атрибут `vsxattr=canary`, при этом родительский элемент до пробы был другим тегом); `JsStringProof` прогоняет `JsStringLexer` по содержимому `<script>`: после закрывающей кавычки пробы лексер должен оказаться в состоянии `CODE`, и в токенах присутствует идентификатор `vsxvar`. Успех → L3.
  4. **Шаг 4 — классификация и finding.** L2 → `verdict=SUSPECTED`, `severity=LOW`, `confidence` по 9.3; L3 → `LIKELY`, `severity=MEDIUM`; finding формирует `XssFindingFactory`.
  5. **Ограничения:** пробы содержат только буквенно-цифровую канарейку и разметку без атрибутов-обработчиков событий и без `script`-тегов; запрещено включать в `ProbeLibrary` строки, содержащие `on*=`, `javascript:`, `<script`, `<svg`, `<img` (валидатор библиотеки при старте, тест `ProbeLibraryTest` проверяет отказ запуска при таких строках).
  6. 429/503: `Retry-After` соблюдается (≤ 30 с), после 3 подряд — параметр помечается `RATE_LIMITED`, `inconclusive++`.
- **Интерфейсы:** `ctx.http.execute(...)` — только через `BudgetedHttp`.
- **Проверки:** `CharacterSurvivalProbeTest` — по таблице «ответ-заглушка → survivingChars» (сущности, `\x3c`, `%3c`, смешанный случай); `HtmlDomProofTest` — (1) `<vsxtag id=X>` внутри `<p>` → proof=true; (2) то же внутри `<textarea>` → false (RCDATA не создаёт элемент); (3) в `<!-- -->` → false; (4) закодированный `&lt;vsxtag` → false; (5) атрибут: `<input value="X" vsxattr="X">` → true, `<input value="X&quot; vsxattr=&quot;X">` → false; `JsStringProofTest` — `var a="X";vsxvar=1;//";` → true; `var a="X\";vsxvar=1;//";` (кавычка экранирована) → false; `ReflectedXssDetectorTest` (MockWebServer) — сценарии: (a) отражение без кодирования в тексте → `LIKELY`; (b) отражение с HTML-кодированием → нет finding; (c) отражение внутри `<textarea>` без кодирования `<` но с `</textarea>` не выживающим → `SUSPECTED`/нет по правилам; (d) ответ `application/json` → нет finding; (e) нестабильный ответ → confidence ниже; (f) параметр не отражается → нет finding и ровно 1 запрос на параметр (+1 повтор).
- **Критерии завершения:** на наборе заглушек нет ни одного finding для случаев (b),(d); все перечисленные unit-тесты зелёные; число запросов на параметр ≤ 5.

### Этап X-5 — Evidence, severity, confidence, дедупликация, публикация

- **Цель:** воспроизводимое и безопасное представление результатов.
- **Предварительные условия:** X-4.
- **Файлы:** `.../report/XssFindingFactory.kt`, `.../report/XssConfidenceScorer.kt`, `.../report/XssSeverityPolicy.kt`, `.../report/EvidenceBuilder.kt`, `.../report/Recommendations.kt`, тесты `XssConfidenceScorerTest.kt`, `XssSeverityPolicyTest.kt`, `EvidenceBuilderTest.kt`, `XssFingerprintTest.kt`, `XssFindingContractTest.kt`.
- **Реализация:**
  1. **Fingerprint:** `Fingerprint.of("XSS","REFLECTED", method, normalizedPathWithoutQueryValues, paramLocation, paramName, contextClass)`, где `contextClass` ∈ `HTML_TEXT | ATTR | SCRIPT_STRING | OTHER` (несколько вхождений в одном классе контекста схлопываются; разные классы дают разные findings).
  2. **Confidence** (`XssConfidenceScorer`): базовые `SUSPECTED=30`, `LIKELY=70`, `CONFIRMED=95`; поправки: +5 — структурное подтверждение получено в двух независимых пробах; −10 — нестабильный ответ; −15 — `Content-Type` не `text/html` (но HTML-парсинг применён); −10 — отражение только при `POST` с `Content-Type: application/json`; clamp 1..99. Формула и каждое слагаемое пишутся в `evidence.observations` (`confidence.base`, `confidence.adj.*`).
  3. **Severity** (`XssSeverityPolicy`): `SUSPECTED→LOW`; `LIKELY→MEDIUM`; `CONFIRMED→HIGH`; **CRITICAL не назначается никогда**; при наличии в ответе заголовка `Content-Security-Policy` без `unsafe-inline` для `script-src` (или `default-src`) — severity понижается на одну ступень, `details.mitigations=["CSP"]`. Тест фиксирует, что «наличие признака» не даёт > `MEDIUM` без структурного доказательства.
  4. **Evidence** (`EvidenceBuilder`): `request` (метод, URL, заголовки после `Redactor`), `response` (статус, заголовки `Content-Type`, `Content-Security-Policy`, `X-Content-Type-Options`; `bodyExcerpt` — окно ±200 символов вокруг первого вхождения, ≤ 2 КБ; `highlights` — смещения внутри `bodyExcerpt` в UTF-16 code units), `observations` (список ключ-значение: контекст, выжившие символы, тип доказательства, шаги пробы). Суммарно ≤ 16 КБ; превышение → усечение `bodyExcerpt`.
  5. **Рекомендации** (`Recommendations`): по контексту: кодирование для HTML-текста, для атрибута (кавычки обязательны + кодирование), для JS-строки (JSON-кодирование/не вставлять в скрипт), CSP как дополнительный слой, ссылка на OWASP XSS Prevention Cheat Sheet (текстом, без активной ссылки в UI).
  6. **Публикация:** `ctx.report(finding)`; один finding на fingerprint за job (внутри job `HashSet<String>`).
- **Проверки:** `XssConfidenceScorerTest` (таблица: вердикт × поправки → число, включая clamp); `XssSeverityPolicyTest` (CSP понижает; CRITICAL недостижим при любых входах — property-based перебор); `EvidenceBuilderTest` (заголовок `Cookie`/`Authorization` в evidence отсутствует/`[REDACTED]`; `bodyExcerpt` ≤ 2 КБ; смещения указывают на канарейку, в т. ч. для строки с эмодзи); `XssFingerprintTest` (одинаковые значения параметров, разные значения query → одинаковый fingerprint; разные контексты → разные); `XssFindingContractTest` (сериализованный finding валиден по `finding-reported.v1.json`).
- **Критерии завершения:** два прогона на одном стенде дают одинаковые fingerprints; evidence не содержит секретов; контракт соблюдён.

### Этап X-6 — Stored XSS (second-order) — опционально в MVP

- **Цель:** обнаружение сохранённого отражения на указанных страницах наблюдения.
- **Предварительные условия:** X-5.
- **Файлы:** `.../stored/StoredXssDetector.kt`, `.../stored/ObservationPlan.kt`, тест `StoredXssDetectorTest.kt`; в контракт `moduleOptions.xss` добавляется `observeEndpointIds: [uuid]` (опциональное поле — версию схемы не меняет).
- **Реализация:** (1) отправить канарейку через endpoint-источник (только `idempotent=false` разрешено для этой проверки и только в `POST`-форме, помеченной пользователем флагом `allowStore=true` в `moduleOptions`); (2) запросить каждый endpoint из `observeEndpointIds`; найденное вхождение проходит этапы X-3/X-4 (survival + structure proof) уже на странице наблюдения; (3) `vulnType=XSS_STORED`, fingerprint включает `sourceEndpoint` и `observeEndpoint`; (4) лимит: ≤ 3 записи на job (каждая «пишет» данные в приложение) — `details.writesPerformed`.
- **Проверки:** `StoredXssDetectorTest` (MockWebServer с in-memory «гостевой книгой»): канарейка из POST появляется на странице наблюдения без кодирования → `LIKELY`; с кодированием → нет finding; `allowStore=false` → запись не выполняется, `inconclusive++`; 4-я запись → пропущена.
- **Критерии завершения:** на `lab-app` (T-1) обнаруживается stored-стенд и не срабатывает safe-стенд.

### Этап X-7 — Браузерное подтверждение (опционально, решение D-9)

- **Цель:** перевод `LIKELY → CONFIRMED` при реальном исполнении маркера.
- **Предварительные условия:** X-5, N-1; достаточный запас времени.
- **Файлы:** `.../browser/BrowserVerifier.kt`, `.../browser/BrowserPool.kt`, `xss-browser/Dockerfile` (отдельный образ, не образ основного модуля), тест `BrowserVerifierIT.kt`.
- **Реализация:** Playwright for Java в отдельном контейнере `xss-browser` с `cap_drop: ALL`, seccomp-профилем, `mem_limit`, без доступа к сети кроме `lab-net`; `BrowserVerifier.verify(url, canary)`: страница открывается с ограничением 10 с; в контекст добавляется init-script, определяющий `window.__vsx = []`; проба для контекста содержит безвредное присваивание `window.__vsx.push('<canary>')` (не обращается к cookie/DOM/сети); успех = в `__vsx` появилась канарейка. Все сетевые запросы браузера проходят через разрешённый `route` (только host target), блокировка остального. Пул ≤ 2 контекстов. Недоступность браузера → finding остаётся `LIKELY`, `observations.browser=UNAVAILABLE`.
- **Проверки:** `BrowserVerifierIT` — уязвимая страница `lab-app` → `CONFIRMED`; страница с CSP, блокирующей inline → остаётся `LIKELY`; запрос к внешнему хосту из страницы блокируется.
- **Критерии завершения:** метрики precision/recall с браузером и без — сравнены в E-3; при невыполнении к М5 этап исключается без влияния на остальное.

**Тестовые сценарии XSS-модуля (сводно, источник — `testbeds/lab-app`, T-1):** `/xss/reflect-text` (L3), `/xss/reflect-attr-dq`, `/xss/reflect-attr-sq`, `/xss/reflect-js-string`, `/xss/stored-guestbook`; безопасные: `/safe/xss/encoded-text`, `/safe/xss/encoded-attr`, `/safe/xss/json-echo` (JSON), `/safe/xss/no-reflect`, `/safe/xss/textarea-encoded`. Ожидания записаны в `ground-truth.yaml` и проверяются в E-2.

---

## Проект 4. SQLi Detection Module (`sqli-module`)

Пакет `io.vulnscan.sqli`. Переиспользуются только `scanner-kit` (рантайм, `SafeHttpClient`, контракты). Анализаторы ответов и детекторы — собственные: XSS-архитектура (поиск отражения) здесь не применима, главный метод — **сравнение ответов с базовым уровнем**.

**Что считается SQLi.** Только результат двухсторонней проверки: параметр управляет синтаксисом SQL-запроса, что проявляется воспроизводимым различием «пара контрольных значений». Не признаются доказательством сами по себе: HTTP 500; изменение длины ответа; единичная задержка; наличие слова `sql` в теле.

**Безопасность проб (общие правила модуля).**

- Только пробы без побочных эффектов: нет `;`-стековых запросов, нет `DROP/INSERT/UPDATE/DELETE/EXEC/xp_`, нет `UNION` с выборкой данных, нет `OR`-тавтологий (`OR 1=1` может расширить `UPDATE/DELETE`); boolean-проверки только на `AND`.
- Boolean-/time-проверки выполняются только для endpoint с `idempotent=true`; error-based — тоже только для `idempotent=true` (одиночная кавычка в `UPDATE` безопасна, но политика единая и простая).
- `ProbeCatalog` валидируется при старте: строки с запрещёнными токенами → отказ запуска (`ProbeCatalogTest`).

### Этап Q-1 — Каркас и планирование параметров

- **Цель:** сервис принимает `SCAN_JOB(SQLI)` и строит план проверяемых параметров.
- **Предварительные условия:** S-4.
- **Файлы:** `sqli-module/src/main/kotlin/io/vulnscan/sqli/SqliModuleApplication.kt`, `SqliDetectionEngine.kt`, `config/SqliProperties.kt` (`maxTimeProbesPerJob=20`, `timeDelaySec=3`, `timeDelayMaxSec=5`, `baselineSamples=3`, `latencySamples=5`), `job/SqliJobValidator.kt`, `plan/ParameterPlanner.kt`, `plan/ValueTypeInferer.kt`, `Dockerfile`; тесты `SqliJobValidatorTest`, `ParameterPlannerTest`, `ValueTypeInfererTest`.
- **Реализация:**
  1. `SqliJobValidator`: наличие `moduleOptions.sqli`; хотя бы один из `errorBased|booleanBased|timeBased`; `timeBased=true` требует `limits.maxDurationSec ≥ 300` (иначе `INVALID_JOB`).
  2. `ValueTypeInferer.infer(sampleValue): NUMERIC | STRING` — `NUMERIC`, если значение целое/десятичное без ведущих нулей; JSON-числа — `NUMERIC`; иначе `STRING`; пустое → `STRING`.
  3. `ParameterPlanner`: параметры `QUERY`, `BODY_FORM`, `BODY_JSON`, `PATH`; бюджет на параметр: baseline 3 + error 4 + boolean 9 + (time: ≤ 9 при включении: 5 замеров латентности + 3 пробы + 1 повтор общих). Если сумма превышает `maxRequests`, приоритет error → boolean → time; `time` — только для параметров, где error/boolean дали `SUSPECTED`+ либо `dbmsHint` задан.
- **Проверки:** `ParameterPlannerTest` (бюджет, приоритеты, `idempotent=false` исключает параметр с `reason=NOT_IDEMPOTENT`); `ValueTypeInfererTest` (`"42"`, `"4.2"`, `"042"`→STRING, `"abc"`, `""`, `"1e5"`→STRING).
- **Критерии завершения:** `claim → план → JOB_COMPLETED` на заглушке; неидемпотентные параметры не получают ни одного запроса (проверка счётчика MockWebServer).

### Этап Q-2 — Baseline, нормализация и сравнение ответов

- **Цель:** формальная модель «насколько два ответа различаются».
- **Предварительные условия:** Q-1.
- **Файлы:** `.../baseline/BaselineCollector.kt`, `.../baseline/Baseline.kt`, `.../compare/ResponseNormalizer.kt`, `.../compare/ResponseComparator.kt`, `.../compare/ResponseSignature.kt`, тесты `ResponseNormalizerTest`, `ResponseComparatorTest`, `BaselineCollectorTest`.
- **Реализация:**
  1. `ResponseNormalizer.normalize(body, contentType)`: удаляет значения, зависящие от времени/запроса — ISO-даты и unix-time, UUID, длинные hex/base64-токены (≥ 16 символов), `csrf`/`nonce`-значения в `<input>`/`<meta>`, отражённое значение самого параметра (подстрока тестового значения заменяется `§`), множественные пробелы; для JSON — канонизация с сортировкой ключей и маскированием перечисленных выше значений.
  2. `ResponseSignature(status, contentType, normalizedLength, tokenSet (множество «слов» длиной ≥ 3), structureHash (хэш последовательности HTML-тегов или JSON-ключей))`.
  3. `ResponseComparator.similarity(a, b): Double` ∈ [0,1] — `0.5·Jaccard(tokenSet) + 0.3·(structureHash совпал ? 1 : 0) + 0.2·(1 − |len_a − len_b| / max(len_a, len_b))`; `status` различается → `sameStatus=false` отдельно (в `ComparisonResult`), а не через similarity.
  4. `BaselineCollector.collect(endpoint, parameter)`: `baselineSamples=3` идентичных запросов с `sampleValue`; вычисляет `Baseline(signature, minPairwiseSimilarity, medianLatencyMs, madLatencyMs, stable)`; `stable = minPairwiseSimilarity ≥ 0.98 && одинаковый status`; порог различимости `θ = min(0.90, minPairwiseSimilarity − 0.03)`. Нестабильный baseline (`stable=false`) → boolean-проверка для параметра **не выполняется** (`inconclusive`, `reason=UNSTABLE_BASELINE`); error-based допускается.
- **Проверки:** `ResponseNormalizerTest` (даты/UUID/CSRF/отражённое значение маскируются; текст товара — нет); `ResponseComparatorTest` (идентичные → 1.0; разный список результатов → < θ; страница с изменённой датой → ≥ 0.98 после нормализации; разный статус → `sameStatus=false`); `BaselineCollectorTest` (стабильный ответ → `stable=true`; ответ со случайным блоком → `stable=false`; медиана/MAD латентности вычислены верно на заданной серии).
- **Критерии завершения:** на фикстурах из `testbeds` (по 3 стабильных и 3 нестабильных ответа) классификация `stable` совпадает с ожиданием.

### Этап Q-3 — Error-based

- **Цель:** обнаружение по сигнатурам ошибок СУБД с контрольной парой.
- **Предварительные условия:** Q-2.
- **Файлы:** `.../detect/ErrorSignatureCatalog.kt` (+ `src/main/resources/sqli/error-signatures.yml`), `.../detect/ErrorBasedDetector.kt`, `.../detect/DbmsFingerprint.kt`, тесты `ErrorSignatureCatalogTest`, `ErrorBasedDetectorTest`.
- **Реализация:**
  1. Каталог регулярных выражений по СУБД (PostgreSQL: `syntax error at or near`, `unterminated quoted string`, `PSQLException`, `pg_query`; MySQL/MariaDB: `You have an error in your SQL syntax`, `check the manual that corresponds to your`; MSSQL: `Unclosed quotation mark`, `Incorrect syntax near`; Oracle: `ORA-0\d{4}`, `quoted string not properly terminated`; SQLite: `unrecognized token`, `SQLITE_ERROR`; общие JDBC-/ODBC-подписи) — каждая запись: `dbms`, `regex`, `weight`.
  2. Алгоритм для параметра: (а) baseline не содержит сигнатур (иначе параметр `inconclusive: SIGNATURE_IN_BASELINE`); (б) **error-проба** — значение `v` + `'` (для `STRING`) или `v` + `'` также для `NUMERIC` (числовой контекст ломается так же); дополнительно `v` + `"` и `v` + `\`; (в) если в ответе появилась сигнатура, которой не было в baseline → кандидат; (г) **контрольная проба** — `v` + `''` (экранированная пара) для `STRING`; ответ не должен содержать сигнатуру и `similarity(control, baseline) ≥ θ`; (д) **повтор** error-пробы — сигнатура снова; (е) вердикт: кандидат + контроль чист + повтор → `CONFIRMED`-класс «error-based» → у модуля `verdict=LIKELY`, если ошибка выдана единственной сигнатурой слабого веса, и `CONFIRMED`, если сумма весов ≥ порога и dbms определена однозначно; кандидат без чистого контроля → `SUSPECTED`; ответ 500 без сигнатуры → не finding (только `observations.http500=true` в stats).
  3. `DbmsFingerprint` собирает `dbms` по весам совпавших сигнатур; при конфликте двух СУБД → `dbms=UNKNOWN`.
- **Проверки:** `ErrorSignatureCatalogTest` (каждая запись матчит свой эталонный текст и **не** матчит текст обычной страницы документации «SQL syntax» из фикстуры — защита от ложных срабатываний); `ErrorBasedDetectorTest`: (1) уязвимый (кавычка → PG-ошибка, `''` → норма) → `CONFIRMED`; (2) 500 без сигнатуры → нет finding; (3) сигнатура присутствует и в baseline → `inconclusive`; (4) сигнатура на `'`, но и на `''` → `SUSPECTED`; (5) ошибка не повторяется на втором запросе → нет finding; (6) блокировка WAF (403 «request blocked») → нет finding.
- **Критерии завершения:** все 6 сценариев зелёные; в evidence присутствуют оба запроса (проба и контроль) и фрагмент сигнатуры.

### Этап Q-4 — Boolean-based

- **Цель:** обнаружение по различию ответов на пару логически противоположных условий.
- **Предварительные условия:** Q-2, Q-3 (общий каркас детекторов).
- **Файлы:** `.../detect/BooleanBasedDetector.kt`, `.../detect/BooleanProbeSets.kt`, тесты `BooleanBasedDetectorTest`.
- **Реализация:**
  1. `BooleanProbeSets` по `ValueType`: `STRING`: `v' AND '1'='1` / `v' AND '1'='2`, затем `v' AND 'a'='a` / `v' AND 'a'='b`; `NUMERIC`: `v AND 1=1` / `v AND 1=2`, затем `v AND 2=2` / `v AND 2=3`. Комментарные суффиксы (`-- `, `#`) не добавляются в MVP (нужны только если приложение обрезает хвост; фиксируется как ограничение).
  2. Условие подтверждения для одной пары `(T, F)`: `similarity(T, baseline) ≥ θ` **и** `similarity(F, baseline) < θ` **и** `similarity(T, F) < θ`. Подтверждение метода: обе пары выполняют условие, и ещё контроль «синтаксически невалидное, но не логическое» (`v' AND '1'='1'x`) **не** воспроизводит поведение T (отсекает приложения, нормализующие ввод).
  3. Статус ответа: если `F` возвращает 4xx/5xx, а `T` 200 — различие по статусу считается за признак, но confidence не повышается (приложение могло упасть на синтаксисе — проверяется контролем).
  4. Рандомизация порядка отправки T/F (чтобы отсечь эффект кэширования/счётчиков); каждая пара повторяется 2 раза (по 4 запроса на каждую из двух пар + 1 контрольный = 9 запросов на параметр; это значение использует `ParameterPlanner` в Q-1).
  5. Вердикты: обе пары + контроль + повторяемость → `LIKELY` (`CONFIRMED` не присваивается без второго независимого признака; если boolean и error-based подтверждены для одного параметра — `CONFIRMED` в единственном finding, см. Q-6); одна пара из двух → `SUSPECTED`; иначе нет finding.
- **Проверки:** `BooleanBasedDetectorTest` (MockWebServer-«приложение» с настраиваемым поведением): (1) настоящее boolean-поведение → `LIKELY`; (2) приложение игнорирует значение (всегда один ответ) → нет finding; (3) ответы случайно различаются (нестабильный baseline) → `inconclusive`, запросов ≤ бюджета; (4) приложение фильтрует кавычки и возвращает «ошибку ввода» на T и F одинаково → нет finding; (5) T отличается от baseline из-за счётчика просмотров → отсеивается нормализацией/повтором; (6) контрольная проба воспроизводит F → нет finding.
- **Критерии завершения:** все сценарии зелёные; ни один запрос не содержит `OR`.

### Этап Q-5 — Time-based (включается явно)

- **Цель:** осторожное обнаружение по задержке.
- **Предварительные условия:** Q-4.
- **Файлы:** `.../detect/TimeBasedDetector.kt`, `.../detect/LatencyModel.kt`, `.../detect/TimeProbeCatalog.kt`, тесты `LatencyModelTest`, `TimeBasedDetectorTest`.
- **Реализация:**
  1. Условия запуска: `timeBased=true`; выбранная СУБД известна (`dbmsHint` либо `DbmsFingerprint` из Q-3; «перебор всех СУБД» **запрещён**); `endpoint.idempotent=true`; лимит `maxTimeProbesPerJob`.
  2. `LatencyModel`: `latencySamples=5` запросов с `sampleValue`, ровно по одному (без параллелизма); `median`, `MAD`; нет проверки при `median > 2000 мс` или `MAD > 0.5·median` (`inconclusive: HIGH_JITTER`).
  3. Пробы (`TimeProbeCatalog`): PostgreSQL `v' AND 1=(SELECT 1 FROM pg_sleep(N))--`-подобные варианты для `STRING`/`NUMERIC` — каталогом закрепляется точный текст на СУБД и тип; `N = timeDelaySec` (по умолчанию 3, максимум `timeDelayMaxSec`), контроль — `N=0`.
  4. Последовательность: `sleep(N)` → задержка `t1`; `sleep(0)` → `t0`; `sleep(2N)` → `t2` (2N ≤ 2·`timeDelayMaxSec` и ≤ `httpTimeoutMs`/2); подтверждение: `t1 ≥ median + 0.8·N·1000` и `t1 > median + 5·MAD` и `t0 ≤ median + 3·MAD` и `t2 − t1 ≥ 0.6·N·1000` (задержка масштабируется); все три условия + повтор последовательности → `LIKELY`; иначе — нет finding (одна задержка не доказательство).
  5. Нагрузка: пробы строго последовательны; пауза между пробами ≥ `N` секунд; суммарное время time-проб ≤ 30 % `maxDurationSec`; при любом `5xx`/таймауте чтения на пробе — параметр прекращается.
- **Проверки:** `LatencyModelTest` (median/MAD на фиксированных сериях; выброс не сдвигает медиану); `TimeBasedDetectorTest` (управляемый «сервер» с настраиваемой задержкой): (1) задержка пропорциональна `N` → `LIKELY`; (2) постоянная медленная страница (3 с независимо от ввода) → нет finding; (3) одиночный случайный выброс 3 с → нет finding после повтора; (4) `timeBased=false` → ни одного запроса с `sleep`; (5) `dbmsHint=null` и dbms не определена → time-проверка пропускается; (6) превышение `maxTimeProbesPerJob` → остальные параметры `skipped`.
- **Критерии завершения:** на `lab-app` time-based стенд распознаётся, медленный-но-безопасный — нет.

### Этап Q-6 — Классификация, severity, confidence, evidence

- **Цель:** единый итог по параметру.
- **Предварительные условия:** Q-3…Q-5.
- **Файлы:** `.../report/SqliClassifier.kt`, `.../report/SqliConfidenceScorer.kt`, `.../report/SqliSeverityPolicy.kt`, `.../report/SqliFindingFactory.kt`, `.../report/SqliEvidenceBuilder.kt`, тесты `SqliClassifierTest`, `SqliConfidenceScorerTest`, `SqliSeverityPolicyTest`, `SqliFindingContractTest`.
- **Реализация:**
  1. `SqliClassifier.classify(results: List<DetectorResult>)`: один finding на `(endpoint, parameter)`; `vulnType` — по сильнейшему признаку (`ERROR_BASED` > `BOOLEAN_BASED` > `TIME_BASED`); `verdict=CONFIRMED` если ≥ 2 независимых метода дали `LIKELY`+ (или error-based `CONFIRMED`); `LIKELY` — один метод `LIKELY`; `SUSPECTED` — иначе.
  2. Confidence: `SUSPECTED=30`, `LIKELY=70`, `CONFIRMED=92`; +5 за определённую СУБД; −10 за `unstable baseline` в любом из методов; −10, если контроль-проба хотя бы частично воспроизвела эффект; clamp 1..99.
  3. Severity: `CONFIRMED→HIGH`, `LIKELY→HIGH` для error-/boolean-based, `MEDIUM` для time-based; `SUSPECTED→LOW`; **CRITICAL не назначается** (извлечение данных не выполняется и не доказывается).
  4. Fingerprint: `Fingerprint.of("SQLI", method, normalizedPath, location, paramName)` (без вида метода, чтобы несколько методов одного параметра не плодили дубли).
  5. Evidence: baseline-сигнатура (статус, длина, `similarity`), пары проб с запросами/ответами (по 2 КБ), значения `similarity` и `θ`, замеры латентности (для time), совпавшая сигнатура СУБД; всё через `Redactor`.
- **Проверки:** `SqliClassifierTest` (комбинации результатов → вердикт: error+boolean → `CONFIRMED`; только boolean → `LIKELY`; только time → `LIKELY/MEDIUM`; error `SUSPECTED` → `SUSPECTED/LOW`); `SqliSeverityPolicyTest` (property-based: CRITICAL недостижим); `SqliFindingContractTest` (валидность по схеме; одинаковый fingerprint для двух методов одного параметра).
- **Критерии завершения:** модуль целиком проходит интеграционный тест `SqliModuleKafkaIT` против `lab-app` (после T-1) без ложных findings на `/safe/sqli/*`.

**Тестовые сценарии SQLi-модуля (источник — `lab-app` с PostgreSQL, T-1):** уязвимые — `/sqli/error/products?id=` (текст ошибки СУБД возвращается), `/sqli/boolean/search?name=` (ошибки скрыты, поведение различается), `/sqli/time/lookup?code=` (одинаковый ответ, но выполнение зависит от `pg_sleep`); защищённые — `/safe/sqli/products` (параметризованный запрос), `/safe/sqli/search`; «ложные признаки» — `/fp/sqli/always500` (всегда 500), `/fp/sqli/random-block` (случайный блок в ответе), `/fp/sqli/slow` (всегда 3 с), `/fp/sqli/sql-in-docs` (текст страницы содержит «You have an error in your SQL syntax» как цитату), `/fp/sqli/waf` (403 на кавычки), `/fp/sqli/rate-limit` (429 после N запросов); отказы — таймаут (сервер молчит), `connection reset`, недоступная БД стенда (`/sqli/error/products` при остановленном `lab-db` → стабильный 500 без сигнатур — нет finding). Ожидания — в `ground-truth.yaml`.

---

## Проект 5. IDOR Detection Module (`idor-module`)

Пакет `io.vulnscan.idor`. Идентификатор в URL — не уязвимость. Уязвимость — это **различие между ожидаемым и фактическим доступом**: пользователь X получает объект, принадлежащий O, хотя в матрице прав X не имеет доступа. Поэтому модуль не угадывает владельцев: матрица `resources[].ownerIdentityId` + `access[]` — обязательная часть задания (B.8.2).

**Ограничения автоматического IDOR-сканирования (фиксируются в дипломе).**

1. Без матрицы ожидаемых прав вердикт невозможен — модуль **не** делает «выводов по форме ответа» без владельца.
2. Оракул «тот же объект» опирается на `ownerMarkers` (JSONPath значений, уникальных для объекта); без них допустим только вердикт `SUSPECTED`.
3. MVP — только чтение (A-8). Изменяющие проверки (PUT/PATCH/DELETE) — расширение с `disposable`-ресурсами (риск R-8).
4. Обнаружение скрытых идентификаторов (перебор соседних значений) выключено по умолчанию; допускается только для явно перечисленных `idValue`.

### Этап I-1 — Каркас, валидация, получение идентичностей

- **Цель:** модуль принимает `SCAN_JOB(IDOR)`, проверяет полноту матрицы, получает секреты по internal API.
- **Предварительные условия:** S-4, C-5 (или mock Core).
- **Файлы:** `idor-module/src/main/kotlin/io/vulnscan/idor/IdorModuleApplication.kt`, `IdorDetectionEngine.kt`, `config/IdorProperties.kt`, `job/IdorJobValidator.kt`, `identity/IdentityVault.kt`, `identity/IdentityContext.kt`, `Dockerfile`; тесты `IdorJobValidatorTest`, `IdentityVaultTest`.
- **Реализация:**
  1. `IdorJobValidator`: `mode=READ_ONLY` ⇒ все методы `GET|HEAD`; ≥ 2 identities; у каждого resource: существующий `ownerIdentityId`, ≥ 1 запись `access`, каждая identity в `access` из списка `identities`; `ownerMarkers` не пуст — иначе ресурс помечается `markersMissing` (проверка допускается, но максимум `SUSPECTED`); нарушение критичных правил → `JOB_FAILED(INVALID_JOB, retryable=false)`.
  2. `IdentityVault.load(ctx)`: `ctx.identities()` (internal HTTP, B.8.5) → хранение **только в памяти** (`ConcurrentHashMap<identityId, IdentityContext>`); `IdentityCredentials` — класс с переопределённым `toString()` (маска) и без сериализации (`@JsonIgnore`/не `data class`); по завершении job — явное обнуление байтовых массивов (`fill(0)`).
  3. `IdentityContext`: собственный `CookieJar` и свой экземпляр `SafeHttpClient`-обёртки (общий `RateLimiter` и общий бюджет запросов на job — через `BudgetedHttp`, но разные cookie jar и заголовки аутентификации) — исключает смешивание сессий.
- **Проверки:** `IdorJobValidatorTest` (HEAD/GET допустимы, PUT при `READ_ONLY` отклоняется; identity в `access`, но не в `identities`; ресурс без владельца); `IdentityVaultTest` (`toString` не содержит пароля; после `close()` массивы обнулены; два контекста не разделяют cookie — Set-Cookie от alice не попадает в запрос bob).
- **Критерии завершения:** на заглушке Core модуль получает identities один раз за job и не пишет секреты в лог (проверка `OutputCapture`).

### Этап I-2 — Аутентификация и проверка сессий

- **Цель:** надёжно получать и поддерживать сессии идентичностей.
- **Предварительные условия:** I-1.
- **Файлы:** `.../auth/SessionManager.kt`, `.../auth/Authenticator.kt` (интерфейс) и реализации `FormLoginAuthenticator.kt`, `JsonLoginAuthenticator.kt`, `StaticCredentialAuthenticator.kt` (`BEARER_STATIC`, `COOKIE_STATIC`, `HEADER_STATIC`), `.../auth/SessionCheck.kt`, тесты `FormLoginAuthenticatorTest`, `JsonLoginAuthenticatorTest`, `SessionManagerTest`.
- **Реализация:**
  1. `SessionManager.establish(identity)`: по `authType` вызывает соответствующий `Authenticator`; `FORM_LOGIN` — POST формы на `authConfig.loginUrl` с полями `usernameField/passwordField` (+ CSRF-токен формы, если задан `csrfFieldRegex`), успех — признак `successIndicator` (подстрока тела/redirect на путь/наличие cookie); `JSON_LOGIN` — POST JSON, токен извлекается по `tokenJsonPath` и подставляется в `Authorization: Bearer`.
  2. `SessionCheck.verify(identity)`: запрос `sessionCheck.endpointId` от имени identity; ожидание `expectStatus` и (если задан) `identityMarker` (например `$.username == label`). Не прошла → identity `INVALID` (статус `AUTH_FAILED` для всех её проверок, **не** «доступ запрещён»).
  3. Истечение сессии: ответ на проверку со статусом `401` или `302` на `loginUrl`, либо `sessionCheck` внезапно не прошёл → однократный `re-login` + повтор запроса; повторный отказ → identity `EXPIRED`; все оставшиеся проверки, где она участвует, → `INCONCLUSIVE(SESSION_EXPIRED)`. Количество re-login ≤ 2 на identity за job.
  4. Проверка соответствия: после аутентификации выполняется `sessionCheck` для **каждой** identity и сравнивается `identityMarker`, чтобы исключить ситуацию «обе идентичности фактически вошли под одним пользователем».
- **Проверки:** `FormLoginAuthenticatorTest` (MockWebServer: успех, неверный пароль → `AUTH_FAILED`, CSRF-поле формы извлекается); `JsonLoginAuthenticatorTest` (токен по JSONPath; отсутствие поля → `AUTH_FAILED`); `SessionManagerTest`: (1) сессия протухает на 5-м запросе → один re-login, проверки продолжаются; (2) re-login не помогает → `EXPIRED`, зависящие проверки `INCONCLUSIVE`; (3) обе identity вернули `username=alice` → job завершается `JOB_FAILED(AUTH_FAILED, retryable=false)`; (4) пароль не встречается ни в одном исходящем логе.
- **Критерии завершения:** отдельный счётчик `stats.inconclusive{reason}` отражает причины; ни один отказ аутентификации не превращается в finding.

### Этап I-3 — Pipeline горизонтальных проверок

- **Цель:** алгоритм сравнения доступов (11.2 ТЗ) для read-only горизонтального IDOR.
- **Предварительные условия:** I-2.
- **Файлы:** `.../check/HorizontalAccessCheck.kt`, `.../check/AccessProbe.kt`, `.../check/AccessOutcome.kt` (enum), `.../check/ResponseClassifier.kt`, `.../check/OwnerMarkerMatcher.kt`, `.../check/DenialShape.kt`, `.../check/ResourceRequestFactory.kt`, тесты `ResponseClassifierTest`, `OwnerMarkerMatcherTest`, `HorizontalAccessCheckTest`.
- **Реализация (для каждого `resource R`, владелец `O`, другая identity `X`, у которой `expected=DENY`):**
  1. **Owner baseline:** `O` запрашивает `R` (через `ResourceRequestFactory`: подстановка `idValue` в `idParameter`). Ожидается `2xx` и найденные `ownerMarkers`. Иначе `INCONCLUSIVE(OWNER_BASELINE_FAILED)` (неверная конфигурация или протухшая сессия) — без finding.
  2. **Anonymous control:** запрос без аутентификации. Если `2xx` и `ownerMarkers` присутствуют → ресурс `PUBLIC_OBSERVED`: проверки X для него не создают finding (`stats.publicResources++`), если в матрице он не `PUBLIC` — предупреждение в `observations` (расхождение с конфигурацией).
  3. **Denial shape:** X запрашивает **несуществующий** идентификатор (значение, гарантированно отсутствующее: `idValue` + `-nonexistent-<canary>` для строковых либо `2^31-1` для числовых) — формируется `DenialShape(status, tokenSignature)` — «как выглядит отказ/пустота для X».
  4. **Probe:** X запрашивает `R`. `ResponseClassifier.classify(response, ownerResponse, denialShape, markers): AccessOutcome`:
     - `401/403` → `DENIED` (корректно);
     - `404/410` → `DENIED_HIDDEN` (корректно);
     - `3xx` на `loginUrl` → `AUTH_REQUIRED` → проверка сессии X → при проваленной — `INCONCLUSIVE(SESSION_EXPIRED)`;
     - `5xx`, таймаут → `ERROR` (`inconclusive`);
     - `2xx`, **все/доля** `ownerMarkers` найдена в ответе X → `UNAUTHORIZED_ACCESS` (сильное доказательство);
     - `2xx`, маркеры не найдены, `similarity(response, denialShape) ≥ θ` → `DENIED_SOFT` (200 «пустой»/«нет доступа» — корректно);
     - `2xx`, маркеры не найдены, но структура ответа совпадает со структурой `ownerResponse` (одинаковые JSON-ключи) и не похожа на `denialShape` → `AMBIGUOUS_STRUCTURE` (`SUSPECTED`, максимум);
     - иное → `AMBIGUOUS` (`inconclusive`, без finding).
  5. **Повторность:** `UNAUTHORIZED_ACCESS` перепроверяется вторым запросом (идентичный) — оба должны дать результат; иначе `AMBIGUOUS`.
  6. **Различение ожиданий:** `expected=ALLOW` — выполняется только проверка «доступ есть» (расхождение — предупреждение конфигурации, не finding); `expected=PUBLIC` — только контроль публичности.
  7. **Finding:** при `UNAUTHORIZED_ACCESS` формируется `IDOR_HORIZONTAL` (см. I-5).
- **Интерфейсы:** `BudgetedHttp` и `IdentityContext` из I-1/I-2.
- **Проверки:**
  - `OwnerMarkerMatcherTest`: JSONPath `$.ownerEmail`, `$.items[0].sku` совпадают/не совпадают; значения маскируются в evidence (`al***@lab.test`).
  - `ResponseClassifierTest` — таблица по всем веткам выше (≥ 14 строк); особое: 200 с телом `{"error":"forbidden"}` → `DENIED_SOFT` при совпадении с `denialShape`.
  - `HorizontalAccessCheckTest` (MockWebServer-«приложение» с политиками): **сценарии 1–7 ниже**, плюс (8) запросов на один ресурс и одну пару ≤ 6 (baseline, anonymous, denial-shape, probe ×2, +1).
- **Критерии завершения:** все семь сценариев дают ожидаемые исходы (таблица ниже); отсутствуют finding для `INCONCLUSIVE`-веток.

**Таблица сценариев IDOR (ожидаемый результат модуля):**

| #   | Сценарий                                           | Конфигурация                                                                   | Ожидаемый результат                                                                          |
| --- | -------------------------------------------------- | ------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------- |
| 1   | A читает свой объект                               | owner=A, A запрашивает                                                         | baseline OK, finding нет                                                                     |
| 2   | B читает объект A; сервер отдаёт данные            | expected(B)=DENY, ответ 200 с маркерами A                                      | `IDOR_HORIZONTAL`, `CONFIRMED`                                                               |
| 3   | B читает объект A; сервер отказывает 403/404       | expected(B)=DENY                                                               | `DENIED`, finding нет                                                                        |
| 4   | Общий объект                                       | expected(B)=ALLOW                                                              | finding нет; доступ подтверждён                                                              |
| 5   | Публичный объект                                   | anonymous 200 с маркерами                                                      | `PUBLIC_OBSERVED`, finding нет                                                               |
| 6   | Сессия B недействительна                           | login провален/401 после re-login                                              | `INCONCLUSIVE(AUTH_FAILED/SESSION_EXPIRED)`, finding нет                                     |
| 7   | Несуществующий идентификатор / неоднозначный ответ | owner baseline падает (404) либо ответ B не похож ни на отказ, ни на владельца | `INCONCLUSIVE`, finding нет (при «похоже по структуре, но без маркеров» — `SUSPECTED`/`LOW`) |

### Этап I-4 — Вертикальные проверки

- **Цель:** доступ низкопривилегированной идентичности к функциям роли выше.
- **Предварительные условия:** I-3.
- **Файлы:** `.../check/VerticalAccessCheck.kt`, `.../check/RoleOrder.kt`, тест `VerticalAccessCheckTest`.
- **Реализация:** ресурсы с `expected=DENY` для identity `USER` и `expected=ALLOW` для `ADMIN` (в матрице) проверяются так же, как в I-3, но `vulnType=IDOR_VERTICAL`; оракул «тот же ответ» — `ownerMarkers` берутся из ответа ADMIN (ADMIN выступает «эталонным владельцем» функции); `RoleOrder` задаёт порядок ролей `USER < ADMIN`, проверка выполняется только если роль X ниже роли эталонной identity; endpoint типа «список всех пользователей» без маркеров — `SUSPECTED` на основании структуры (без утверждения «нарушено»). Изменяющие функции не вызываются.
- **Проверки:** `VerticalAccessCheckTest`: (1) USER получает `/api/admin/users` (маркеры — email других пользователей) → `IDOR_VERTICAL CONFIRMED`; (2) USER получает 403 → нет finding; (3) ADMIN сам не прошёл baseline → `INCONCLUSIVE`; (4) роль X не ниже эталонной → проверка не выполняется.
- **Критерии завершения:** вертикальные и горизонтальные findings различаются `vulnType` и fingerprint.

### Этап I-5 — Finding, severity, confidence, evidence

- **Цель:** формирование безопасных, объяснимых результатов.
- **Предварительные условия:** I-3, I-4.
- **Файлы:** `.../report/IdorFindingFactory.kt`, `.../report/IdorConfidenceScorer.kt`, `.../report/IdorSeverityPolicy.kt`, `.../report/IdorEvidenceBuilder.kt`, тесты `IdorConfidenceScorerTest`, `IdorSeverityPolicyTest`, `IdorEvidenceBuilderTest`, `IdorFindingContractTest`.
- **Реализация:**
  1. **Fingerprint:** `Fingerprint.of("IDOR", vulnType, method, normalizedPathTemplate (идентификатор заменён на `{id}`), idParameter.name, ownerIdentityId, attackerIdentityId)`.
  2. **Verdict:** `CONFIRMED` — маркеры владельца найдены в ответе атакующего, повтор подтвердил, anonymous control отказал, `sessionCheck` обеих identity в норме; `LIKELY` — найдены не все маркеры (≥ 50 %) либо anonymous control не выполнен; `SUSPECTED` — только структурное совпадение/отсутствие `ownerMarkers`.
  3. **Confidence:** `CONFIRMED=95`, `LIKELY=75`, `SUSPECTED=35`; −10 если доля найденных маркеров < 100 %; −15 если `denialShape` не удалось получить; clamp 1..99.
  4. **Severity:** по `sensitivity` ресурса: `HIGH→HIGH`, `MEDIUM→MEDIUM`, `LOW→LOW`; `IDOR_VERTICAL` повышает на одну ступень, максимум `CRITICAL` только при `sensitivity=HIGH` **и** `CONFIRMED`; `SUSPECTED` — максимум `LOW`.
  5. **Evidence:** запросы атакующей и владельческой identity (заголовки через `Redactor`: `Authorization`, `Cookie` → `[REDACTED]`); тело ответа атакующего — усечённый фрагмент (≤ 2 КБ) с маскированием значений маркеров (`alice@lab.test` → `al***@lab.test`; числовые — первые/последние символы); `observations`: `ownerBaselineStatus`, `anonymousStatus`, `attackerStatus`, `markersMatched` (список JSONPath и маска), `repeatConfirmed`, `denialShapeSimilarity`. Идентификатор identity в evidence — `label`, не `identityId`/логин.
  6. **Рекомендации:** проверять авторизацию на сервере для каждого обращения к объекту (по владельцу/роли), использовать непрозрачные идентификаторы как дополнительную меру (не заменяет проверку), централизованный authorization-слой, негативные тесты на межпользовательский доступ.
- **Проверки:** `IdorConfidenceScorerTest`, `IdorSeverityPolicyTest` (таблицы; CRITICAL только при 3 условиях); `IdorEvidenceBuilderTest` (в evidence нет пароля/токена/cookie/`identityId`; маски применены; ≤ 16 КБ); `IdorFindingContractTest` (валиден по схеме).
- **Критерии завершения:** строка секрета не встречается ни в одном finding (проверка поиском подстроки во всех опубликованных сообщениях `IdorModuleKafkaIT`).

### Этап I-6 — Интеграция и нагрузка

- **Цель:** сквозной прогон против `lab-app`.
- **Предварительные условия:** I-5, T-1, C-5.
- **Файлы:** `.../discovery/OwnerListingDiscovery.kt` (опционально), `idor-module/src/test/kotlin/.../IdorModuleKafkaIT.kt`.
- **Реализация:**
  1. `OwnerListingDiscovery` (включается `moduleOptions.idor.discovery=[{endpointId, identityId, idJsonPath}]`): запрашивает список объектов от имени identity и считает их принадлежащими ей (допущение, фиксируется в `details.assumption=OWNER_BY_LISTING`); confidence таких findings −10.
  2. Бюджет запросов: `≈ 6 × (число пар resource×attacker) + число identity × 2`; при превышении `maxRequests` — ресурсы с высшей `sensitivity` первыми, остальные → `skipped(BUDGET)`.
- **Проверки:** `IdorModuleKafkaIT` (реальный Kafka + `lab-app` в Testcontainers): запуск задания с матрицей `alice/bob/admin` → публикуются findings ровно по ground-truth `lab-app` (IDOR-кейсы I-01…I-08), ни одного finding на `/api/safe/*`, публичных и общих объектах; запросов ≤ бюджета; в логах нет паролей.
- **Критерии завершения:** TP = число положительных кейсов IDOR из `ground-truth.yaml`, FP = 0 на стенде (фиксируется в E-2 как результат, **не** как предположение заранее).

**Тестовое приложение IDOR (часть `lab-app`, T-1):** пользователи `alice`, `bob` (`USER`), `admin` (`ADMIN`); сущности — `orders` (владелец), `documents` (с таблицей `shared_with`), `posts` (публичные), `profiles`. Эндпоинты: уязвимые `GET /api/orders/{id}` (без проверки владельца), `GET /api/documents/{id}` (проверка только аутентификации), `GET /api/admin/users` (нет проверки роли); безопасные `GET /api/safe/orders/{id}` (владелец или 404), `GET /api/safe/documents/{id}` (владелец или `shared_with`), `GET /api/posts/{id}` (публичный), `GET /api/safe/admin/users` (403 для USER); «шум» — `GET /api/fp/orders/{id}` (200 с пустым `{}` для чужих), `GET /api/fp/slow-session` (протухающая сессия). Ожидания — в `ground-truth.yaml`.

---

## Проект T/N/E. Тестовые стенды, инфраструктура, оценка

### Этап T-1 — Лабораторное приложение `lab-app` и ground truth

- **Цель:** изолированная цель с заранее известными уязвимостями и безопасными аналогами.
- **Предварительные условия:** S-1.
- **Файлы:** `testbeds/lab-app/build.gradle.kts`; `.../LabApplication.kt`; `.../xss/XssLabController.kt`; `.../sqli/SqliLabController.kt` (JDBC напрямую к `lab-db`, конкатенация строк **намеренно** только в `/sqli/**`); `.../idor/IdorLabController.kt`; `.../auth/LabAuthFilter.kt` (cookie-сессии и `POST /login`, `GET /api/me`); `.../safe/*`, `.../fp/*`; `testbeds/lab-app/src/main/resources/schema.sql`, `seed.sql`; `testbeds/lab-app/ground-truth.yaml`; `testbeds/lab-app/Dockerfile`; тест `GroundTruthConsistencyTest.kt`.
- **Реализация:**
  1. Приложение слушает только внутренний интерфейс docker-сети `lab-net`; его порт **не публикуется** на хост в профиле по умолчанию (в профиле `lab-open` публикуется на `127.0.0.1` для ручной отладки). Заглавная страница содержит предупреждение «намеренно уязвимое приложение».
  2. Эндпоинты — по спискам в разделах XSS/SQLi/IDOR выше; каждая уязвимость имеет безопасный двойник с тем же интерфейсом (для пары «уязвимый/безопасный»).
  3. **Формат `ground-truth.yaml`:** список кейсов
     ```yaml
     version: 1
     labAppVersion: "<git sha на момент разметки>"
     cases:
       - id: XSS-01
         module: XSS
         key: "GET /xss/reflect-text q" # method + path-template + parameterName
         expectedVulnerable: true
         vulnType: XSS_REFLECTED
         variant: HTML_TEXT
       - id: XSS-S01
         module: XSS
         key: "GET /safe/xss/encoded-text q"
         expectedVulnerable: false
       - id: IDOR-01
         module: IDOR
         key: "GET /api/orders/{id} id alice->bob" # + атакующий
         expectedVulnerable: true
         vulnType: IDOR_HORIZONTAL
     ```
     Правило построения `key` для сопоставления с finding (`CaseKey.of(finding)`): `method + " " + normalizedPathTemplate + " " + parameter.name` (+ ` ownerLabel->attackerLabel` для IDOR). Реализация `CaseKey` — в `evaluation/runner` и тестируется там.
  4. `GroundTruthConsistencyTest` (в `lab-app`): каждый `key` соответствует существующему маппингу контроллера (рефлексия по `@RequestMapping`); у каждого `expectedVulnerable=true` есть безопасный двойник в списке; нет дублирующихся `id`.
- **Проверки:** `GroundTruthConsistencyTest`; ручной smoke-скрипт `testbeds/smoke.sh` (curl по 10 эндпоинтам: уязвимый отражает, безопасный кодирует).
- **Критерии завершения:** стенд стартует в compose; ground truth проходит тест консистентности и **не** корректируется под результаты сканера (изменения ground truth — отдельным коммитом с обоснованием, см. E-1).

### Этап N-1 — Docker Compose и запуск

- **Цель:** воспроизводимое окружение одной командой.
- **Предварительные условия:** C-1 (образ Core), S-4 (образы модулей), F-1 (образ frontend).
- **Файлы:** `infra/docker-compose.yml`, `infra/docker-compose.lab.yml` (профиль `lab`: `lab-app`, `lab-db`), `infra/.env.example`, `infra/kafka/create-topics.sh`, `core-backend/Dockerfile`, `xss-module/Dockerfile`, `sqli-module/Dockerfile`, `idor-module/Dockerfile`, `frontend/Dockerfile` (+ `nginx.conf`), `infra/README.md` (команды), `infra/scripts/{up,down,reset,smoke}.sh`.
- **Реализация:**
  1. **Сервисы:** `postgres`, `kafka` (одноузловой KRaft — без отдельного ZooKeeper), `kafka-init` (одноразовое создание топиков `scan.jobs.xss.v1`, `scan.jobs.sqli.v1`, `scan.jobs.idor.v1`, `scan.job-events.v1`, `scan.dlq.v1`: 3 партиции, `replication-factor=1`), `core-backend`, `xss-module`, `sqli-module`, `idor-module`, `frontend` (nginx), профиль `lab`: `lab-app`, `lab-db`.
  2. **Сети:** `db-net` (postgres, core), `bus-net` (kafka, core, три модуля), `lab-net` (`internal: true`; модули, core, lab-app, lab-db), `edge` (frontend, публикует `127.0.0.1:8081`). Модули **не** подключены к `db-net` — доступа к БД нет; core подключён к `lab-net` только для DNS-валидации и проверки владения target.
  3. **Контейнеры модулей и Core:** непривилегированный пользователь; `read_only: true` + `tmpfs: /tmp`; `cap_drop: [ALL]`; `security_opt: [no-new-privileges:true]`; `mem_limit`/`cpus` (параметры `.env`); секреты (`SCANNER_SECRET_KEY`, `SCANNER_SERVICE_TOKEN_XSS|SQLI|IDOR`, пароль БД, bootstrap-админ) — только в `.env` (не в образе, `.env` в `.gitignore`, `.env.example` с пустыми значениями и командой генерации).
  4. **Health/readiness:** `healthcheck` для всех сервисов (`core`: `/actuator/health/readiness`; модули: `/actuator/health`; kafka: `kafka-topics.sh --list`; postgres: `pg_isready`); `depends_on: condition: service_healthy`; Core стартует после `kafka-init` (`service_completed_successfully`).
  5. **Профили Spring:** `docker` (cookie `Secure=false` для локального http, включён allowlist `scope.allowedPrivateCidrs=<подсеть lab-net>`); `test` (Testcontainers). В `.env.example` подсеть lab-net задаётся явно (`LAB_SUBNET=...`) и используется и в compose (`ipam`), и в `ALLOWED_PRIVATE_CIDRS` — чтобы значения не расходились.
  6. **Миграции:** Flyway выполняется Core при старте (единственный владелец схемы).
  7. **Логи:** JSON в stdout, `docker compose logs -f <service>`; ротация `json-file` (`max-size`, `max-file`) в compose.
  8. **Тестовые учётные данные:** lab-пользователи `alice`, `bob`, `admin` и их пароли генерируются `infra/scripts/up.sh` в `infra/.lab-secrets.env` (не коммитится) и подставляются в `lab-app` seed и `evaluation/runner`.
  9. **Остановка/перезапуск:** `down.sh` (останов без удаления томов), `reset.sh` (с удалением томов `pgdata`, `kafka-data`), `docker compose restart xss-module` — допустимо во время скана (проверяется сценарием R-1 ниже).
- **Проверки:** `infra/scripts/smoke.sh`: поднять стек → `GET /actuator/health/readiness` Core = UP → логин bootstrap-админом → `POST /targets` на `lab-app` → `approve` → `POST /scans` XSS → ожидание `COMPLETED` ≤ 5 минут → ≥ 1 finding. Сценарии устойчивости (`infra/scripts/resilience/*.sh`, запускаются вручную перед демонстрацией и в E-3): **R-1** `docker restart` модуля во время `RUNNING` → задание перезапускается (attempt=2), scan завершается; **R-2** остановка `kafka` на 60 с во время создания скана → после старта задания публикуются; **R-3** остановка `core-backend` на 30 с во время скана → модуль повторяет heartbeat/результаты (retry), после возврата Core обрабатывает результаты; **R-4** остановка `lab-db` → SQLi-проверки дают `inconclusive`, не findings.
- **Критерии завершения:** `up.sh && smoke.sh` проходит на чистой машине с Docker; все R-сценарии дают описанный результат.

### Этап E-1 — Методика оценки и инструмент

- **Цель:** автоматический, воспроизводимый расчёт метрик.
- **Предварительные условия:** T-1; Core API (C-6).
- **Файлы:** `evaluation/runner/src/main/kotlin/io/vulnscan/eval/EvalMain.kt`; `CoreApiClient.kt`; `GroundTruth.kt`; `CaseKey.kt`; `Matcher.kt`; `Metrics.kt`; `ReportWriter.kt` (CSV + Markdown); `RunManifest.kt`; тесты `MetricsTest`, `MatcherTest`, `CaseKeyTest`; `evaluation/README.md`; `evaluation/runs/` (результаты, в `.gitignore` кроме манифестов).
- **Реализация:**
  1. **Запуск:** `EvalMain --module XSS|SQLI|IDOR|ALL --repeats 5 --threshold LIKELY` → для каждого повтора: создать scan через API (политика `NORMAL`), дождаться терминального статуса (poll), выгрузить findings (`GET /findings?scanId=…&size=100` постранично), сохранить сырой JSON.
  2. **Сопоставление (`Matcher`):** `CaseKey.of(finding)` → кейс ground truth. Finding считается **положительным**, если `verdict ≥ threshold` (по умолчанию `LIKELY`; чувствительность дополнительно считается для `SUSPECTED` и `CONFIRMED`). Для каждого кейса:
     - `expectedVulnerable=true` и есть ≥ 1 положительный finding → **TP** (один раз на кейс, независимо от числа дублей);
     - `expectedVulnerable=true` и положительных findings нет → **FN**;
     - `expectedVulnerable=false` и есть положительный finding → **FP** (по одному на кейс);
     - finding, не сопоставленный ни с одним кейсом → **FP** (`unmatched`, учитывается отдельно в отчёте, чтобы не скрывать недоразмеченные участки).
       Дубли (одинаковый `CaseKey`) схлопываются; число дублей фиксируется отдельной метрикой `duplicates` (должно быть 0 благодаря fingerprint).
  3. **Метрики на модуль и порог:** `TP, FP, FN`, `Precision = TP/(TP+FP)`, `Recall = TP/(TP+FN)`; нулевой знаменатель: `Precision` при `TP+FP=0` и `Recall` при `TP+FN=0` выводятся как `N/A` (не 0 и не 1), рядом печатаются исходные счётчики; для `ALL` публикуется таблица **по модулям**, а агрегированная строка («micro» — суммы TP/FP/FN по всем модулям) допускается только с подписью «micro-average, модули разного размера»; смешивание «в один процент» без таблицы по модулям запрещено форматом отчёта.
  4. **Время и запросы:** `scan_jobs.started_at/finished_at` → длительность; `scan_jobs.requests_sent` → запросов на задание; по 5 повторам печатаются медиана и min–max.
  5. **Воспроизводимость (`RunManifest`):** `commit SHA` репозитория (`git rev-parse HEAD`, должен быть чистым), `labAppVersion`, версия `ground-truth.yaml` (SHA-256 файла), значения `scan_policies` (limits), `moduleOptions`, образы (digest из `docker inspect`), дата/время UTC, ОС/CPU/RAM хоста, количество повторов, `threshold`; пишется в `evaluation/runs/<timestamp>/manifest.json`.
  6. **Правила чистоты:** `ground-truth.yaml` размечается **до** первого запуска (отдельный коммит); изменения после первого запуска допускаются только с записью в `evaluation/GROUND_TRUTH_CHANGELOG.md` (причина, кто, какие кейсы); результаты предыдущих запусков при этом не переписываются.
- **Проверки:** `CaseKeyTest` (нормализация путей/параметров; для IDOR включение `owner->attacker`); `MatcherTest` (двойной finding на один кейс → один TP; finding без кейса → FP `unmatched`; кейс без findings → FN; порог `LIKELY` отбрасывает `SUSPECTED`); `MetricsTest` (ручные таблицы: TP=8,FP=2,FN=0 → P=0.8,R=1.0; TP=0,FP=0 → P=N/A; TP=0,FN=5 → R=0.0; TP+FN=0 → R=N/A).
- **Критерии завершения:** `EvalMain` повторяемо воспроизводит метрики из сохранённых сырых findings без обращения к Core (`--from-raw`).

### Этап E-2 — Эксперименты по модулям

- **Цель:** измерить качество каждого модуля отдельно.
- **Предварительные условия:** E-1, готовность соответствующего модуля.
- **Файлы:** `evaluation/scenarios/xss.yaml`, `sqli.yaml`, `idor.yaml` (параметры запусков: endpoints, identities, options), `evaluation/negatives/` (негативные цели без уязвимостей), `evaluation/RESULTS.md` (заполняется **по результатам**, шаблон без чисел).
- **Реализация (для каждого модуля):**
  1. **XSS:** положительные кейсы XSS-01…XSS-05 (контексты text / attr-dq / attr-sq / js-string / stored), отрицательные — `/safe/xss/*` (кодирование, JSON, нет отражения, textarea). Дополнительно сравнение «без браузера» и (если X-7 готов) «с браузером».
  2. **SQLi:** положительные — error / boolean / time; отрицательные — `/safe/sqli/*`; «ложные признаки» — `/fp/sqli/*` (все должны быть `expectedVulnerable=false`). Time-based включается отдельным запуском (`timeBased=true`) и считается отдельной строкой, так как нагрузка отличается.
  3. **IDOR:** положительные I-01…I-08; отрицательные — `/api/safe/*`, общие/публичные объекты; кейсы `INCONCLUSIVE` (протухшая сессия) учитываются как `expectedVulnerable=false`, чтобы показать отсутствие ложных срабатываний.
  4. **Негативные цели:** (а) целевое приложение без единой уязвимости соответствующего класса (`lab-app` профиль `clean` — те же маршруты, но безопасные реализации); (б) ожидается TP=0, FP=0, Precision=N/A — результат интерпретируется по FP.
  5. Для каждого модуля — 5 повторов; отчёт: таблица TP/FP/FN/P/R/время/запросы, список FN и FP с причинами (разбор вручную с `findingId`, evidence, гипотеза).
- **Проверки:** повторные запуски дают одинаковые TP/FP/FN (детерминизм); если отличаются — фиксируется как нестабильность в `RESULTS.md`.
- **Критерии завершения:** `RESULTS.md` содержит таблицы по каждому модулю с манифестом; чисел «заранее» в документе нет.

### Этап E-3 — Сквозная проверка от UI до finding

- **Цель:** подтвердить работу всей цепочки и нагрузочные свойства.
- **Предварительные условия:** E-2, F-5, N-1.
- **Файлы:** `evaluation/scenarios/e2e.yaml`, `frontend/tests/e2e/*.spec.ts`, `infra/scripts/resilience/*.sh`.
- **Реализация:** (1) Playwright: вход → создание target → verify/approve → scan XSS+SQLI+IDOR → ожидание → переход к findings → открыть finding → просмотр evidence → отметить false positive; (2) параллельный запуск 2 сканов одним пользователем → третий отклонён (429); (3) сценарии R-1…R-4; (4) время «создание скана → первый finding» и «→ `COMPLETED`» по 5 прогонам; (5) отмена скана во время `RUNNING` → `CANCELLED` ≤ 15 с.
- **Критерии завершения:** все сценарии зелёные; результаты времён внесены в `RESULTS.md` вместе с манифестом.

---

## Безопасность самого сканера — где какая проверка выполняется

| Проверка                                                                  | Frontend                                | Core                                                         | Модуль (`scanner-kit`)                                                                        | Реализация / тест                                       |
| ------------------------------------------------------------------------- | --------------------------------------- | ------------------------------------------------------------ | --------------------------------------------------------------------------------------------- | ------------------------------------------------------- |
| Схема/порт/userinfo target                                                | подсказка (Zod)                         | **обязательно** (`ScopePolicyService`)                       | повтор `ScopeMatcher`                                                                         | C-3, S-3 (`ScopePolicyServiceTest`, `ScopeMatcherTest`) |
| Приватные/внутренние адреса, link-local, метаданные                       | —                                       | `IpPolicy` при создании target/скана                         | `IpPolicy` перед **каждым** соединением                                                       | S-3 (`IpPolicyTest`)                                    |
| DNS-rebinding (адрес изменился после проверки)                            | —                                       | повторный резолв при создании скана                          | `SafeDns` возвращает только проверенные адреса + проверка `socket.inetAddress` после connect  | S-3 (`SafeHttpClientTest` п.7)                          |
| Redirect                                                                  | —                                       | редиректы при верификации — вручную через `SafeHttpClient`   | ручное следование ≤ 3, повторная проверка каждого hop                                         | S-3 (п.2–4)                                             |
| Include/exclude scope                                                     | показ пользователю                      | формирует итоговый список (target + запрос), нельзя ослабить | `ScopeMatcher` на каждом запросе                                                              | S-3, C-4                                                |
| Владение target                                                           | блок «нельзя выбрать неподтверждённый»  | **обязательно** `status=VERIFIED`                            | —                                                                                             | C-4 (`ScanServiceIT` п.2)                               |
| Rate limit/параллелизм/макс. запросов/длительность/размер ответа/таймауты | —                                       | лимиты политики; `maxActiveScans*`; `deadline_at`            | `BudgetedHttp`, `RateLimiter`, `ResponseLimits`, `hardLimits`                                 | S-4, C-4, C-7                                           |
| Backpressure                                                              | —                                       | `queueTimeout`, лимит активных сканов                        | `pause()` consumer при занятых слотах                                                         | S-4 (п.5 `WorkerRuntimeKafkaIT`)                        |
| Изоляция воркера                                                          | —                                       | —                                                            | контейнер: non-root, read-only FS, `cap_drop ALL`, нет доступа к `db-net`, `lab-net internal` | N-1                                                     |
| Секреты identity                                                          | не хранит                               | AES-GCM, выдача только владельцу lease, аудит                | только в памяти, обнуление, маска в `toString`                                                | C-3, C-5, I-1, `LogLeakageTest`                         |
| Маскирование токенов/cookie/параметров                                    | —                                       | в логах (запрет логировать)                                  | `Redactor` в evidence и логах                                                                 | S-3, X-5/Q-6/I-5                                        |
| Ограничение размера evidence                                              | показ с усечением                       | проверка ≤ 16 КБ при приёме                                  | усечение при формировании                                                                     | C-6, X-5                                                |
| Безопасное отображение HTML                                               | **обязательно** (`EvidenceViewer`, CSP) | evidence — только данные (`application/json`)                | —                                                                                             | F-4 (`EvidenceViewer.test`)                             |
| Изоляция результатов пользователей                                        | —                                       | фильтр по `scans.started_by` в каждом запросе                | —                                                                                             | C-6 (`FindingApiIT`)                                    |
| Очистка временных данных                                                  | —                                       | `RetentionCleaner`; сессии                                   | `tmpfs`, нет записи на диск                                                                   | C-8, N-1                                                |

Принцип: проверка во Frontend — только UX; Core и модуль выполняют все проверки самостоятельно (двойной контур для scope: Core при постановке, модуль при каждом сетевом соединении).

---

# Часть D. Межпроектная интеграция

Каждый шаг интеграции начинается только когда перечисленные контракты существуют в `contracts/` и проходят `ContractSchemaTest`.

| Шаг       | Что интегрируется                                 | Готовы к шагу                                        | Контракты должны существовать                              | Сценарий, который должен работать (критерий)                                                                                                      |
| --------- | ------------------------------------------------- | ---------------------------------------------------- | ---------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------- |
| **INT-0** | Контракты и схемы                                 | S-1, S-2                                             | все `contracts/schemas/*`, `examples/*`                    | `ContractSchemaTest` зелёный в CI; Frontend и Core согласовали черновик OpenAPI                                                                   |
| **INT-1** | Core ↔ PostgreSQL                                 | C-1, C-2                                             | DDL                                                        | `MigrationIT`, `ConstraintsIT`                                                                                                                    |
| **INT-2** | Frontend ↔ Core (без сканов)                      | F-1, F-2, C-3                                        | `core-api.v1.json` (auth, targets, endpoints, identities)  | e2e: вход → создание target → verify/approve → добавление endpoint; типы TS сгенерированы из реального OpenAPI, `tsc` проходит                    |
| **INT-3** | Core ↔ Kafka ↔ `EchoEngine`-воркер                | C-4, C-5, C-6, S-4                                   | `scan-job`, `finding-reported`, `job-*`, internal API      | `ScanLifecycleE2E`: scan → outbox → Kafka → claim → heartbeat → findings → `COMPLETED`; дубликат сообщения не создаёт второго выполнения          |
| **INT-4** | Core ↔ XSS-модуль ↔ `lab-app`                     | X-1…X-5, T-1, N-1                                    | схема `finding-reported` (`XssDetails`)                    | scan XSS на `lab-app` через API: findings = ожидаемые по ground truth для XSS; Frontend показывает их (F-3, F-4)                                  |
| **INT-5** | + SQLi-модуль                                     | Q-1…Q-6, `lab-db`                                    | `SqliDetails`                                              | scan SQLI: error/boolean — findings; `/fp/sqli/*` — нет; scan XSS+SQLI одновременно → два job, статус `COMPLETED`                                 |
| **INT-6** | + IDOR-модуль (identities, access rules)          | I-1…I-6, C-3 (identities), C-5 (internal identities) | `moduleOptions.idor`, `IdorDetails`, internal `identities` | scan IDOR: findings по ground truth IDOR; секрет не встречается в Kafka/логах/ответах (`LogLeakageTest`, поиск по `kafka-console-consumer` дампу) |
| **INT-7** | Все три модуля + частичный отказ и восстановление | INT-4…INT-6, C-7                                     | —                                                          | сценарии R-1…R-4 (N-1); остановленный SQLi-модуль → scan `PARTIALLY_COMPLETED`, findings XSS видны; отмена `RUNNING` скана ≤ 15 с                 |
| **INT-8** | Оценка качества                                   | E-1, E-2                                             | `ground-truth.yaml` (зафиксирован до запуска)              | `EvalMain` воспроизводимо формирует таблицы для трёх модулей                                                                                      |
| **INT-9** | Сквозная проверка и демонстрация                  | E-3, F-5                                             | —                                                          | Playwright-сценарии + сценарий демонстрации (Часть F) проходят на чистом `up.sh`                                                                  |

**Параллельность (три потока, интерфейсы согласованы заранее):**

| Поток                  | Работа                                                                        | Не блокируется после                                |
| ---------------------- | ----------------------------------------------------------------------------- | --------------------------------------------------- |
| П-1 (backend)          | S-2→S-3→S-4, C-1…C-8                                                          | INT-0                                               |
| П-2 (frontend + тесты)a| F-1…F-5 на MSW по `core-api.v1.json`; T-1 (`lab-app`, ground truth); N-1; E-1 | INT-0 (OpenAPI-черновик), T-1 нужен модулям к INT-4 |
| П-3 (модули)           | X-1…X-6, Q-1…Q-6, I-1…I-6 против mock-Core и `lab-app`                        | S-4                                                 |

Модули разрабатываются без реального Core: тесты используют MockWebServer-заглушку internal API; реальная интеграция — INT-4…INT-6.

**Порядок исключения при нехватке времени (от первого к последнему):** X-7 (браузер) → X-6 (stored XSS) → I-4 (вертикальные проверки) → Q-5 (time-based) → `OwnerListingDiscovery` (I-6) → OpenAPI-импорт (C-3.7) → AuditPage (F-5). Ядро, которое остаётся всегда: S-1…S-4, C-1…C-7, F-1…F-4, X-1…X-5, Q-1…Q-4+Q-6, I-1…I-3+I-5, T-1, N-1, E-1, E-2.

---

# Часть E. Матрица трассируемости

| ID      | Требование                                                          | Проект                  | Компонент                                                              | Тест                                                                                 | Критерий приёмки                                                 |
| ------- | ------------------------------------------------------------------- | ----------------------- | ---------------------------------------------------------------------- | ------------------------------------------------------------------------------------ | ---------------------------------------------------------------- |
| FR-01   | Вход пользователя, серверная сессия, CSRF                           | Core, Frontend          | `SecurityConfig`, `AuthController`, `AuthContext`                      | `SecurityIT`, `LoginPage.test`                                                       | Без сессии 401; POST без CSRF 403; после 401 UI на `/login`      |
| FR-02   | Управление разрешёнными targets с верификацией                      | Core, Frontend          | `TargetService`, `TargetVerificationService`, `TargetsPage`            | `TargetVerificationServiceTest`, `TargetsPage.test`                                  | Скан возможен только для `VERIFIED`                              |
| FR-03   | Endpoint inventory (ручной ввод, импорт OpenAPI)                    | Core, Frontend          | `EndpointService`, `OpenApiImporter`, `EndpointFormDialog`             | `OpenApiImporterTest`, `EndpointFormDialog.test`                                     | Импорт фикстуры даёт ожидаемые endpoints; дубликат → 409         |
| FR-04   | Создание скана с несколькими модулями                               | Core, Frontend          | `ScanService`, `ScanJobFactory`, `NewScanPage`                         | `ScanServiceIT`, `NewScanPage.test`                                                  | 202; N заданий; outbox записан атомарно                          |
| FR-05   | Надёжная доставка заданий (outbox)                                  | Core                    | `OutboxRelay`                                                          | `OutboxRelayIT`                                                                      | Задание не теряется при недоступном Kafka и падении Core         |
| FR-06   | Идемпотентность выполнения заданий                                  | Core, modules           | `JobControlService.claim`, `JobListener`                               | `JobControlServiceIT` п.1, `WorkerRuntimeKafkaIT` п.2                                | Дубликат сообщения → одно выполнение                             |
| FR-07   | Идемпотентность приёма результатов                                  | Core                    | `JobResultService`, `processed_messages`, `UNIQUE(job_id,fingerprint)` | `JobEventsConsumerIT` п.2–3                                                          | Повтор → одна строка                                             |
| FR-08   | Машина состояний job/scan, итоговый статус, частичный успех         | Core                    | `JobStateMachine`, `ScanStatusAggregator`                              | `JobStateMachineTest`, `ScanStatusAggregatorTest`                                    | Все пары переходов проверены; таблица B.7 покрыта                |
| FR-09   | Retry, зависшие задания, DLQ                                        | Core, modules           | `StuckJobWatchdog`, `DefaultErrorHandler`                              | `StuckJobWatchdogIT`, `WorkerRuntimeKafkaIT` п.3                                     | Падение модуля → повтор ≤ `maxAttempts`; невалидное → DLQ        |
| FR-10   | Отмена сканирования                                                 | Core, modules, Frontend | `ScanService.cancel`, heartbeat, `ScanDetailPage`                      | `ScanCancelIT`, `ScanDetailPage.test`                                                | `RUNNING` → `CANCELLED` ≤ 15 с                                   |
| FR-11   | Findings: фильтры, сортировка, пагинация                            | Core, Frontend          | `FindingQueryService`, `FindingsPage`                                  | `FindingApiIT`, `FindingsPage.test`                                                  | Фильтры по всем полям; size ≤ 100; чужие недоступны              |
| FR-12   | Dashboard                                                           | Core, Frontend          | `DashboardService`, `DashboardPage`                                    | `DashboardQueryIT`, `DashboardPage.test`                                             | Значения = SQL-подсчёту                                          |
| FR-13   | Triage findings                                                     | Core, Frontend          | `FindingController.PATCH`                                              | `FindingApiIT`, `FindingDetailPage.test`                                             | Меняется только `triage_status`, пишется аудит                   |
| FR-14   | Обнаружение reflected XSS                                           | XSS                     | `ReflectedXssDetector`, `HtmlLexer`, `HtmlDomProof`, `JsStringProof`   | `ReflectedXssDetectorTest`, `HtmlLexerTest`, `HtmlDomProofTest`                      | Положительные кейсы XSS-01…05 найдены на `lab-app`               |
| FR-15   | Отражение ≠ уязвимость                                              | XSS                     | `CharacterSurvivalProbe`                                               | `ReflectedXssDetectorTest` (b),(d)                                                   | Закодированное/JSON-отражение не даёт finding                    |
| FR-16   | Обнаружение SQLi: error/boolean/time                                | SQLi                    | `ErrorBasedDetector`, `BooleanBasedDetector`, `TimeBasedDetector`      | `ErrorBasedDetectorTest`, `BooleanBasedDetectorTest`, `TimeBasedDetectorTest`        | Положительные кейсы найдены                                      |
| FR-17   | Нет finding по одиночному 500/длине/задержке                        | SQLi                    | `SqliClassifier`, контрольные пробы                                    | `ErrorBasedDetectorTest` п.2, `TimeBasedDetectorTest` п.2–3, `/fp/sqli/*` в E-2      | FP на `/fp/sqli/*` = 0                                           |
| FR-18   | IDOR с несколькими идентичностями                                   | IDOR                    | `SessionManager`, `HorizontalAccessCheck`                              | `SessionManagerTest`, `HorizontalAccessCheckTest`                                    | Сценарии 1–7 дают ожидаемые исходы                               |
| FR-19   | Вертикальные проверки                                               | IDOR                    | `VerticalAccessCheck`                                                  | `VerticalAccessCheckTest`                                                            | `USER→/admin` = finding; 403 = нет                               |
| FR-20   | Различение публичного/отказа/ошибки аутентификации/неоднозначного   | IDOR                    | `ResponseClassifier`                                                   | `ResponseClassifierTest`                                                             | Все ветки таблицы покрыты                                        |
| FR-21   | Единый контракт finding и evidence                                  | все                     | `scanner-kit.contracts`, `*FindingFactory`                             | `ContractSchemaTest`, `*FindingContractTest`                                         | Все findings валидны по схеме                                    |
| SEC-01  | Scope enforcement (схема, порт, host, include/exclude)              | Core, kit               | `ScopePolicyService`, `ScopeMatcher`                                   | `ScopePolicyServiceTest`, `ScopeMatcherTest`                                         | Таблица обходов полностью отклонена                              |
| SEC-02  | SSRF: private/loopback/link-local/metadata                          | Core, kit               | `IpPolicy`, `SafeDns`                                                  | `IpPolicyTest`, `SafeHttpClientTest`                                                 | Запрос к запрещённому адресу не отправляется                     |
| SEC-03  | DNS rebinding и redirect-обходы                                     | kit                     | `SafeDns`, ручные редиректы, проверка `socket.inetAddress`             | `SafeHttpClientTest` п.2–4,7                                                         | Все блокируются                                                  |
| SEC-04  | Контроль доступа к объектам пользователей                           | Core                    | `OwnershipPolicy`                                                      | `TargetAccessIT`, `FindingApiIT`                                                     | Чужое → 404                                                      |
| SEC-05  | Секреты IDOR: шифрование, выдача, отсутствие в логах/findings/Kafka | Core, IDOR              | `SecretCipher`, `IdentityVault`                                        | `SecretCipherTest`, `LogLeakageTest`, `IdorEvidenceBuilderTest`, `IdorModuleKafkaIT` | Строка секрета нигде не найдена                                  |
| SEC-06  | Ограничения нагрузки                                                | Core, kit               | `BudgetedHttp`, `RateLimiter`, `ResponseLimits`                        | `BudgetedHttpTest`, `ActiveScanLimiterIT`                                            | Лимиты соблюдены                                                 |
| SEC-07  | Изоляция воркеров                                                   | infra                   | compose-конфигурация                                                   | `smoke.sh` + проверка `docker inspect` (user, read-only, caps)                       | Модуль не достигает `postgres` (тест `nc` из контейнера → отказ) |
| SEC-08  | Безопасный показ evidence                                           | Frontend                | `EvidenceViewer`, CSP                                                  | `EvidenceViewer.test`, e2e                                                           | Вредоносный evidence не создаёт DOM-элементов и не выполняется   |
| SEC-09  | Rate limiting API, размер запросов, аудит                           | Core                    | `RateLimitFilter`, `RequestSizeFilter`, `AuditService`                 | `SecurityIT`, `RequestSizeFilterTest`                                                | 429/413; события аудита записаны                                 |
| EVAL-01 | Размеченный ground truth                                            | testbeds                | `ground-truth.yaml`                                                    | `GroundTruthConsistencyTest`                                                         | Консистентен, зафиксирован до запуска                            |
| EVAL-02 | Метрики TP/FP/FN, Precision, Recall; `N/A` при нулевом знаменателе  | evaluation              | `Matcher`, `Metrics`                                                   | `MatcherTest`, `MetricsTest`                                                         | Ручные таблицы совпали                                           |
| EVAL-03 | Время и запросы на задание                                          | evaluation              | `ReportWriter`                                                         | `EvalMain` e2e-прогон                                                                | Медиана и min–max по 5 повторам                                  |
| EVAL-04 | Воспроизводимость                                                   | evaluation              | `RunManifest`                                                          | `--from-raw` повтор                                                                  | Метрики идентичны                                                |
| EVAL-05 | Негативные цели без уязвимостей                                     | testbeds, evaluation    | профиль `clean`                                                        | E-2                                                                                  | FP фиксируется отдельно                                          |
| OPS-01  | Воспроизводимый запуск                                              | infra                   | compose, `up.sh`                                                       | `smoke.sh`                                                                           | Чистая машина → `COMPLETED`                                      |
| OPS-02  | Устойчивость к рестартам                                            | infra                   | R-1…R-4                                                                | `resilience/*.sh`                                                                    | Ожидаемые исходы из N-1                                          |

---

# Часть F. План дипломной демонстрации

Используются только контейнеры проекта (`lab-app`, `lab-db`); внешние адреса не сканируются.

| Шаг | Действие                                                                                                                                                                                                                  | Ожидаемое на экране                                                |
| --- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------ |
| 0   | Предварительно: `infra/scripts/up.sh` и `smoke.sh` (за 10 минут до начала); открыты UI (`127.0.0.1:8081`), терминал с `docker compose logs -f core-backend xss-module`, вкладка с `evaluation/runs/<последний>/report.md` | Все сервисы healthy                                                |
| 1   | Вход пользователем `student`. Страница Targets → target `lab-app` (`VERIFIED`, подтверждён админом)                                                                                                                       | Статус-чип `VERIFIED`; показать инструкцию верификации             |
| 2   | **Негатив:** создать target `http://127.0.0.1:5432`                                                                                                                                                                       | «Адрес не разрешён политикой сервера» (`422 SCOPE_VIOLATION`)      |
| 3   | New scan: target `lab-app`, модули XSS+SQLI+IDOR, политика `NORMAL`, подтверждение владения                                                                                                                               | Scan создан, статус `PENDING → RUNNING`                            |
| 4   | В терминале: `kafka-console-consumer` на `scan.jobs.xss.v1` (заранее открыт)                                                                                                                                              | Видно сообщение `SCAN_JOB` с `correlationId`; в нём нет секретов   |
| 5   | Страница скана: карточки заданий XSS/SQLI/IDOR; в логах модулей — `claim`, `heartbeat` с тем же `correlationId`                                                                                                           | Прогресс по заданиям растёт                                        |
| 6   | **Устойчивость (по времени):** `docker restart sqli-module` во время `RUNNING`                                                                                                                                            | Карточка SQLI: попытка 2; скан завершается                         |
| 7   | Скан завершён → Findings: фильтр `severity=HIGH`, `verdict≥LIKELY`                                                                                                                                                        | Findings по XSS/SQLi/IDOR; dashboard — счётчики по severity        |
| 8   | Открыть XSS finding: Evidence (запрос, ответ с подсветкой отражения, контекст, выжившие символы, шаги доказательства), Recommendation                                                                                     | Показать, что доказательство — структурное (не «просто отражение») |
| 9   | Открыть IDOR finding: два запроса (`alice` владелец / `bob` атакующий), маски значений, статус anonymous control                                                                                                          | В evidence нет паролей/токенов                                     |
| 10  | **Негативный сценарий качества:** запустить скан по профилю `clean` (или по `/safe/*`)                                                                                                                                    | 0 findings; в stats — `reflectionsNeutralized`, `DENIED` счётчики  |
| 11  | Открыть `report.md`: таблица по модулям (TP/FP/FN, Precision/Recall, время, запросы), манифест, список FN/FP с разбором                                                                                                   | Числа берутся из файла; прокомментировать FN/FP и ограничения      |
| 12  | Показать отмену: запустить новый скан, нажать «Отменить»                                                                                                                                                                  | `CANCELLING → CANCELLED` ≤ 15 с                                    |

Резервный план: если Kafka/окружение не стартует — показать записанный прогон (`evaluation/runs/…/raw`) и e2e-видео Playwright.

---

# Часть G. Критические риски и открытые вопросы

| ID   | Проблема                                                                                                 | Последствия                                                                                 | Рекомендуемое решение                                                                                                                                                     | Когда решить                                |
| ---- | -------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------- |
| R-1  | Объём работы: 5 проектов + общий слой + стенды + оценка при команде из трёх человек и ограниченном сроке | Не успеем довести модули до качества, достаточного для измерений                            | Следовать порядку исключения из Части D; зафиксировать «ядро» до старта; контрольные точки INT-3 (М3) и INT-5 (М5)                                                        | До S-1                                      |
| R-2  | Точность XSS без браузера                                                                                | `LIKELY` — не доказательство исполнения; возможны FP в нестандартных контекстах (`UNKNOWN`) | Честно называть уровни verdict; различать в оценке `LIKELY` и `CONFIRMED`; X-7 по возможности                                                                             | Решение о X-7 к концу М5                    |
| R-3  | DOM-based XSS не покрывается                                                                             | Класс уязвимостей не обнаруживается                                                         | Явно описать в ограничениях; расширение — через браузерный воркер                                                                                                         | До написания диплома (раздел «Ограничения») |
| R-4  | Оценка на собственных стендах оптимистична (авторы стенда = авторы детектора)                            | Метрики не обобщаются                                                                       | Ground truth размечается до запуска; дополнительно — 1–2 внешних учебных приложения (запуск локально, с разметкой кейсов вручную); обсуждение угрозы валидности в дипломе | До E-2                                      |
| R-5  | Docker-сеть `lab-net` приватная ⇒ нужен allowlist CIDR; ошибка конфигурации ослабит SSRF-защиту          | Сканер может ходить во внутренние адреса                                                    | Allowlist только в профиле `docker`; в `prod`-профиле пуст; тест `ScopePolicyServiceTest` на пустой allowlist; одно значение `LAB_SUBNET` в `.env`                        | N-1                                         |
| R-6  | Нет crawler: endpoint задаёт пользователь                                                                | «Автоматизация» ограничена; демонстрация вручную трудоёмка                                  | OpenAPI-импорт + подготовленные endpoints для `lab-app`; crawler — как работа «на будущее»                                                                                | Решение сейчас (A-5); пересмотр после INT-5 |
| R-7  | Kafka-инфраструктура в локальной разработке (память/время старта)                                        | Медленные интеграционные тесты и CI                                                         | Один `KafkaContainer` на тестовый класс-суперкласс; unit-тесты без Kafka; интеграционные — отдельный Gradle task                                                          | C-1                                         |
| R-8  | Изменяющие IDOR-проверки (PUT/DELETE) небезопасны для реальных систем                                    | Порча данных                                                                                | Вне MVP; допустимы только на `disposable`-ресурсах `lab-app` и при явном флаге                                                                                            | После INT-6                                 |
| R-9  | Time-based SQLi создаёт нагрузку и нестабилен                                                            | Ложные/пропущенные срабатывания, замедление цели                                            | Включение только явным флагом, `dbmsHint`, жёсткие лимиты (Q-5); оценка отдельной строкой                                                                                 | Q-5                                         |
| R-10 | Совместимость версий (Spring Boot ↔ Spring Kafka ↔ Kotlin ↔ JDK ↔ Testcontainers)                        | Ломающиеся сборки                                                                           | Зафиксировать на S-1 по документации релизов; обновления только отдельным PR после зелёного CI                                                                            | S-1                                         |
| R-11 | Правовые аспекты тестирования чужих систем                                                               | Ответственность пользователя/авторов                                                        | Верификация владения, предупреждение в UI, аудит; в дипломе — раздел об ответственном использовании; сканировать только лабораторные цели                                 | Текст диплома                               |
| R-12 | Название темы: если официальная формулировка содержит региональный акцент, в проекте он нигде не выражен | Замечание на защите о несоответствии теме                                                   | Определить, чем он обеспечивается (A-10): например, раздел диплома о требованиях/нормативной базе и маппинг findings на CWE/OWASP                                         | До начала написания диплома                 |

**Открытые вопросы, меняющие объём (требуют ответа команды):**

1. Нужен ли stored XSS (X-6) в защищаемом объёме или достаточно reflected?
2. Достаточно ли вертикального IDOR на одном admin-эндпоинте (I-4) или нужна полноценная матрица ролей?
3. Допускается ли замена Kafka-кластерной конфигурации на одноузловой KRaft для демонстрации (план предполагает да)?

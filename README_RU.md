🚧 Статус проекта: Этап проектирования и архитектуры (RFC) > На данный момент репозиторий содержит подробную техническую спецификацию и архитектурные чертежи системы. Этап активной разработки ядра и SDK запланирован в рамках Roadmap.
# Sentinel.EXE
**Sentinel.EXE** — это высокопроизводительная self-hosted платформа для мониторинга инфраструктуры и автоматизации проверок. В отличие от классических uptime-сервисов, Sentinel построен по принципу **модульного оркестратора**, где ядро системы полностью изолировано от логики исполнения.
### Основная идея

В основе проекта лежит идея **"The Puzzle Architecture"** (Архитектура-пазл). Вы не ограничены стандартными проверками на доступность. Благодаря протоколу **gRPC**, вы можете подключать любые микросервисы (Addons), написанные на любом языке, и собирать из их операций (Operations) сложные сценарии проверки (Pipelines).
### Почему Sentinel?

- **gRPC Core:** Бинарный протокол обеспечивает минимальные задержки и строгую типизацию контрактов между ядром и исполнителями.
- **Hybrid Runtime:** Используйте легковесный "Standard Bundle" для базовых задач или развертывайте тяжелые изолированные воркеры (например, на Playwright или Python) для специфических нужд. 
- **Contextual Intelligence:** Каждый шаг проверки дополняет общий JSON-контекст. Это позволяет строить умные зависимости (например, "сделать скриншот только если HEAD-запрос вернул 503").
- **Distributed Responsibility:** Ядро — это чистый оркестратор. Оно не хранит логи операций или историю проверок. Вся «память» системы распределена по аддонам. Это позволяет масштабировать хранилище под конкретные нужды (например, хранить терабайты скриншотов в одном аддоне, не нагружая основную БД ядра).
- **Self-Hosted & Private:** Полный контроль над вашими данными и инфраструктурой мониторинга.
- **Role-Based Security:** Встроенная система аутентификации и авторизации. Вы сами определяете, кто имеет доступ к дашбордам и конфигурации пайплайнов. Все коммуникации с аддонами могут быть защищены на уровне внутренней сети или mTLS.
### Основные понятия (Core Concepts)

- **Monitor (Монитор)** — Центральный объект наблюдения (например, конкретный домен `ktod.ru` или API-эндпоинт). Является контейнером для стратегий проверки. Один Монитор может содержать несколько **Пайплайнов**, работающих параллельно с разной частотой (например: быстрый пинг каждые 30с и тяжелая проверка со скриншотом раз в час).
- **Pipeline (Пайплайн)** — Конкретный сценарий проверки, привязанный к Монитору. Имеет собственный интервал запуска и описывает логику прохождения шагов. Пайплайн — это законченная стратегия (например, "Проверка SSL" или "Smoke-тест авторизации").
- **Flow (Поток)** — Внутренняя структура Пайплайна. Последовательность (или граф) операций, которая определяет порядок выполнения, логику переходов (`on_success` / `on_failure`) и передачу данных.
- **Addon (Аддон)** — Физический gRPC-микросервис (контейнер), выступающий в роли провайдера функциональности. Один аддон может предоставлять множество различных операций.
- **Operation (Операция)** — Атомарный функциональный блок внутри аддона (например, `http_head`, `screenshot`). Это "исполнитель" конкретного шага в Flow.
- **Context (Контекст)** — Изолированная область памяти (JSON), которая живет в рамках одного запуска Пайплайна. Позволяет операциям внутри Flow обмениваться данными.
- **Params (Параметры)** — Набор конфигурационных данных. Могут определяться как на уровне всего Пайплайна (общие настройки), так и индивидуально для каждой Операции внутри Flow (например, конкретный таймаут или искомый текст на странице).
## Architecture: The Grid

Архитектура Sentinel.EXE построена на жестком разделении обязанностей между **Control Plane** (Ядром) и **Data Plane** (Аддонами). Это позволяет ядру оставаться сверхбыстрым и «невежественным» относительно специфики данных.

### 1. Control Plane (The Brain & Clock)

Написано на **Go**. Это единственный компонент, который знает о существовании пользователя и его настройках.

- **Orchestration:** Ядро хранит конфигурации **Monitors** и **Pipelines**. Оно является инициатором любого запуска.
- **Identity & Access (IAM):** Хранение базы пользователей и управление доступом к Dashboard. Любое обращение к системе начинается с проверки прав.
- **API Gateway:** Ядро проксирует запросы к аддонам. Фронтенд никогда не общается с аддонами напрямую — только через защищенный слой Ядра.
- **Event Bus:** Служит ретранслятором событий между аддонами и фронтендом.
### 2. Data Plane (The Execution & Memory)

Сеть независимых аддонов.

- **Persistence (Distributed Logs):** В отличие от классических систем, история проверок хранится в самих аддонах (например, в `base-addon`). Ядро не знает подробностей `http_head`, оно лишь получает финальный сигнал.
- **Isolation:** Каждый аддон — это «черный ящик». Если операция требует сохранения истории цен или скриншотов за год, аддон сам управляет своей БД (PostgreSQL/ClickHouse/S3).
- **Operation Runtime:** Операции в аддонах выполняют атомарную работу, не зная о существовании пайплайна в целом.
### 3. The Real-time Pulse (WebSockets & Events)

Для обеспечения реактивности интерфейса используется каскадная система уведомлений:

1. **Operation Signal:** По завершении операции аддон возвращает результат ядру.
    
2. **Core Broadcast:** Ядро обновляет статус монитора в своей базе и мгновенно пушит событие в **WebSocket**.
    
3. **UI Refresh:** Фронтенд (Angular), получив сигнал по сокету, понимает, какие именно виджеты нужно перерисовать, и идет за данными напрямую в соответствующий аддон через ядро.
## Entity Examples & Contracts
### 1. The gRPC Contract (`sentinel.proto`)
Это главный закон системы. Любой аддон должен имплементировать этот интерфейс, чтобы Ядро могло с ним работать.

```protobuf
syntax = "proto3";

package sentinel;

service SentinelAddon {
  // Discovery: The Core queries the addon's capabilities and manifests
  rpc GetManifest (Empty) returns (Manifest);

  // Execution: The Core triggers a specific functional operation
  rpc ExecuteOperation (ExecuteRequest) returns (ExecuteResponse);

  // Observability: The Core fetches data from the addon to render UI widgets
  rpc GetWidgetData (WidgetDataRequest) returns (WidgetDataResponse);
}

message Manifest {
  string addon_id = 1;
  string name = 2;
  repeated OperationInfo operations = 3;
}

message OperationInfo {
  string id = 1;            // Example: "http_head"
  string description = 2;
  string config_schema = 3; // UI-Schema or JSON Schema for the config form
}

message ExecuteRequest {
  string operation_id = 1;
  string context_json = 2;  // Current Pipeline Context (JSON)
  string params_json = 3;   // Specific Operation configuration
}

message ExecuteResponse {
  string update_json = 1;  // Data to be merged/patched into the Context
  string signal = 2;       // Execution signals: CONTINUE, STOP, RETRY
  string error = 3;        // Error message (if any)
}
```

### 2. Addon Manifest (Пример ответа `GetManifest`)

Манифест позволяет Ядру узнать, какие настройки отрисовать в UI для пользователя.

```json
{
  "addon_id": "network_bundle",
  "name": "Standard Network Pack",
  "operations": [
    {
      "id": "http_check",
      "description": "Проверка доступности по HTTP",
      "config_schema": {
        "url": { "type": "string", "required": true },
        "method": { "type": "string", "enum": ["GET", "HEAD", "POST"], "default": "HEAD" },
        "timeout": { "type": "integer", "default": 5 }
      }
    }
  ]
}
```
### 3. Pipeline Configuration (Как хранится Flow)

Пример того, как в базе Ядра описан конкретный Пайплайн для Монитора.

```json
{
  "pipeline_id": "main_check",
  "interval": "60s",
  "flow": [
    {
      "step_id": "ping_step",
      "operation_ref": "network_bundle.http_check",
      "params": { "method": "HEAD", "timeout": 10 },
      "on_failure": "notify_error"
    },
    {
      "step_id": "notify_error",
      "operation_ref": "notifiers.telegram",
      "params": { "chat_id": "12345", "template": "Site down!" }
    }
  ]
}
```
### 4. Shared Context (Данные в движении)

Пример объекта, который «пухнет» во время прохождения по Flow.

```json
{
  "execution_id": "exec_88291",
  "target_url": "https://ktod.ru",
  "steps": {
    "ping_step": {
      "status_code": 200,
      "latency_ms": 142,
      "server": "nginx"
    },
    "ssl_check": {
      "is_valid": true,
      "expiry_date": "2026-12-01T00:00:00Z",
      "days_left": 263
    },
    "seo_analyzer": {
      "title": "KtoD - IT Community",
      "has_h1": true,
      "links_found": 42
    }
  }
}
```
## SDK Examples (Go)

Sentinel предоставляет легковесный SDK, который берет на себя всю работу с gRPC-сервером, оставляя разработчику только написание бизнес-логики.
### 1. Регистрация нового Аддона

Разработчику достаточно описать структуру своего сервиса и зарегистрировать операции.

```go
package main

import (
    "github.com/sentinel-exe/sdk-go"
    "context"
)

func main() {
    // Создаем новый аддон
    addon := sdk.NewAddon("network_bundle", "Standard Network Pack")

    // Регистрируем операцию HTTP HEAD
    addon.RegisterOperation(sdk.Operation{
        ID:          "http_head",
        Description: "Быстрая проверка статус-кода через HEAD запрос",
        ConfigSchema: sdk.NewSchema().
            Number("timeout", 5, "Таймаут запроса в секундах").
            Bool("follow_redirects", true, "Следовать ли редиректам"),
        
        // Сама логика выполнения
        Handler: HandleHttpHead,
    })

    // Запускаем gRPC сервер на порту 50051
    addon.Run(":50051")
}
```
### 2. Реализация обработчика (Handler)

Обработчик — это чистая функция, которая работает с входящим контекстом и параметрами.

```go
func HandleHttpHead(ctx context.Context, req sdk.ExecuteRequest) (sdk.ExecuteResponse, error) {
    // 1. Извлекаем параметры, которые пользователь задал в UI
    timeout := req.Params.GetInt("timeout")
    url := req.Context.GetString("target_url")

    // 2. Выполняем логику (например, HTTP запрос)
    resp, err := myHttpClient.Head(url, timeout)
    
    if err != nil {
        // Возвращаем ошибку и сигнал STOP, чтобы прервать пайплайн
        return sdk.StopResponse(err.Error()), nil
    }

    // 3. Возвращаем данные для обновления контекста и сигнал продолжать
    return sdk.ContinueResponse(map[string]interface{}{
        "status_code": resp.StatusCode,
        "latency_ms":  resp.Duration.Milliseconds(),
    }), nil
}
```

### 3. Пример для Python (Контракт универсален)

Благодаря gRPC, написать аддон на Python так же просто:

```python
from sentinel_sdk import Addon, ContinueResponse

addon = Addon("ml_checker", "AI Analysis Pack")

@addon.operation(id="analyze_sentiment")
def analyze(context, params):
    text = context.get("page_text")
    result = ai_model.predict(text)
    return ContinueResponse({"sentiment_score": result})

addon.run(port=50051)
```
### Почему это удобно:

- **Типизация:** SDK предоставляет удобные методы для извлечения данных из JSON (`req.Params.GetInt`, `req.Context.GetString`).
- **Абстракция:** Разработчик не видит gRPC-кода, протобуфов и сетевых сокетов.
- **Гибкость:** Вы можете упаковать в один `main.go` хоть 100 разных операций, и они все будут доступны Ядру через один порт.
## The Handshake & Dynamic UI

**Sentinel.EXE** использует концепцию **Dynamic Discovery**. Ядру не нужно знать о функционале аддона заранее — оно получает все спецификации и правила визуализации в реальном времени через gRPC.
### 1. Процесс "Рукопожатия" (Handshake)

1. **Connection:** Администратор добавляет адрес нового аддона (например, `grpc://network-bundle:50051`) в Dashboard.
2. **Handshake:** Ядро вызывает метод `GetManifest`.
3. **Registration:** Аддон возвращает полный паспорт своих возможностей: список операций, их схемы и описания дашбордов..
4. **On-Demand UI:** Фронтенд Sentinel (Angular) парсит эти схемы и динамически строит формы ввода для шагов пайплайна или виджеты для отображения накопленных данных.
### 2. Спецификации Discovery

#### А. Настройки операций (`config_schema`)

Определяет, как будет выглядеть форма настройки конкретного шага в пайплайне.

```json
{
  "operation_id": "http_check_pro",
  "config_schema": {
    "url": { 
      "type": "string", 
      "widget": "url_input", 
      "params": { "placeholder": "https://example.com" }
    },
    "method": { 
      "type": "string", 
      "widget": "select", 
      "params": { "options": ["GET", "POST", "HEAD"] }
    },
    "check_interval": {
       "type": "integer",
       "widget": "slider",
       "params": { "min": 1, "max": 60, "unit": "s" }
    }
  }
}
```
#### Б. Контракт данных (`provides`)

Декларация ключей, которые операция обязуется записать в **Context**. Это позволяет ядру подсвечивать доступные переменные для последующих шагов Flow.

```json
{
  "operation_id": "ssl_scanner",
  "provides": [
    "ssl.expiry_date",
    "ssl.issuer",
    "ssl.is_valid"
  ]
}
```

### 3. Sentinel.EXE UI: Интерактивные Дашборды

Если аддон хранит собственную статистику или результаты работы (например, историю цен), он может описать способ их визуализации. Вместо отрисовки пикселей, аддон использует системный **UI-Kit** Ядра.

```json
{
  "dashboard_config": {
    "title": "Shop Analytics",
    "widgets": [
      {
        "id": "price_history_table",
        "type": "table", // Системный компонент Ядра (напр. Ag-Grid)
        "data_func": "GetPriceHistory", // gRPC метод для подгрузки данных
        "params": {
          "columns": [
            { "header": "Товар", "field": "name" },
            { "header": "Цена", "field": "price", "format": "currency" }
          ]
        }
      }
    ]
  }
}
```
### 4. Динамическое обновление

- **Hot Reload:** При изменении кода аддона (добавлении полей или новых виджетов) достаточно нажать **"Refresh Manifest"**.
- **Zero Downtime:** Новые возможности появляются в интерфейсе мгновенно. Ядру не требуется пересборка или перезагрузка для поддержки новых типов проверок или отображения новых данных.
### 5. Event Subscription:

В манифесте виджета аддон может указать список событий, на которые он подписан (например, `on_pipeline_finish`). При наступлении такого события ядро «пинает» фронтенд через сокеты, заставляя его обновить данные конкретного виджета.
### Почему это важно:

Это превращает **Sentinel.EXE** из набора скриптов в полноценную **экосистему**.

- **Разработчик аддона** полностью управляет тем, как пользователь взаимодействует с его логикой, используя мощные готовые компоненты Ядра.
- **Ядро** остается стабильным "фундаментом", обеспечивая единый UX, авторизацию и оркестрацию, не вникая в специфику конкретных бизнес-задач (будь то аптайм или парсинг цен).
---
## Roadmap: The Sentinel Evolution

### Milestone 1: The gRPC Backbone (Протокол и Скелет)

- [ ] **Contract:** Финализация `.proto` интерфейса (методы `Execute`, `GetManifest`, `GetWidgetData`).
    
- [ ] **Core Engine (Go):** Базовый оркестратор, способный вызывать операции аддонов по расписанию.
    
- [ ] **Standard Bundle:** Реализация первого аддона с атомарными операциями (`http_head`, `tcp_ping`) для тестирования связи.
    
- [ ] **CLI Manager:** Консольная утилита для регистрации аддонов и ручного запуска пайплайнов.
### Milestone 2: Identity & Security (Периметр)

- [ ] **Auth System:** Реализация в ядре системы пользователей, JWT-авторизации и защиты API.
    
- [ ] **Gateway Logic:** Механизм проксирования запросов от будущего UI к аддонам через слой авторизации ядра.
    
- [ ] **Persistence:** База данных ядра для хранения настроек мониторов, пайплайнов и учетных записей.
    
- [ ] **Base Addon (Stateful):** Создание стандартного хранилища логов, которое будет «памятью» для всех базовых проверок.
### Milestone 3: Discovery & Dynamic UI (Интерфейс)

- [ ] **Sentinel Dashboard (Angular):** Базовый интерфейс со списком мониторов и их статусами.
    
- [ ] **Handshake Engine:** Логика автоматического импорта манифестов и регистрации операций.
    
- [ ] **Dynamic Form Builder:** Генератор форм настроек в UI на основе `config_schema` из аддонов.
    
- [ ] **Discovery UI:** Страница управления аддонами (подключение, обновление манифеста, статус связи).
### Milestone 4: Smart Flow & Context (Интеллект)

- [ ] **Context Manager:** Реализация передачи и мерджа JSON-данных между шагами пайплайна.
    
- [ ] **Logic Engine:** Поддержка условий переходов (`on_success` / `on_failure`) и сигналов управления (`STOP`, `RETRY`).
    
- [ ] **Variable Injection:** Поддержка использования данных из контекста в параметрах операций (например, `url: {{context.base_url}}`).
### Milestone 5: Reactive Ecosystem (Визуализация и Real-time)

- [ ] **WebSocket Bus:** Система пуш-уведомлений от ядра к фронтенду о завершении пайплайнов.
    
- [ ] **Widget Engine:** Реализация системных UI-компонентов (Table, Chart, StatCard) в Angular.
    
- [ ] **Data Stream:** Полноценная поддержка `GetWidgetData` для отображения накопленной в аддонах аналитики.
    
- [ ] **SDK Release:** Публикация библиотек (Go/Python) для быстрой разработки кастомных аддонов сообществом.

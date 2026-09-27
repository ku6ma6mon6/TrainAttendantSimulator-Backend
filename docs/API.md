# API Тренажёра проводников ВСМ — Справочник

REST API бэкенда иммерсивного тренажёра проводников. Описание всех эндпоинтов
в том же порядке, что и в Swagger UI (`/docs`), с русскими пояснениями, схемами
запросов/ответов и примерами.

- **Базовый URL (прод):** `http://91.229.11.11:8001` (или `https://<домен>`)
- **Swagger UI:** `/docs` (интерактивная документация)
- **OpenAPI JSON:** `/openapi.json`
- **Формат данных:** JSON

---

## Аутентификация

Почти все эндпоинты (кроме `register`, `login` и служебных) требуют JWT-токен.
Токен передаётся в заголовке:

```
Authorization: Bearer <access_token>
```

Токен выдаётся при регистрации или входе и действует **720 минут** (12 часов,
настраивается через `ACCESS_TOKEN_EXPIRE_MINUTES`). Токен «stateless» — сервер
ничего не хранит, выход (`logout`) выполняется на стороне клиента простым
удалением токена.

### Общие коды ответов

| Код | Значение |
|-----|----------|
| 200 | Успех |
| 201 | Ресурс создан (регистрация) |
| 400 | Некорректный запрос (например, email уже занят) |
| 401 | Не авторизован / токен недействителен или истёк |
| 404 | Объект не найден (сценарий, результат, пользователь) |
| 409 | Конфликт состояния |
| 422 | Ошибка валидации тела запроса |
| 500 | Внутренняя ошибка сервера |

---

## Группа: Authentication

### POST /api/v1/auth/register

> **Назначение.** Регистрация нового пользователя и выдача JWT-токена.

**Тело запроса** (`RegisterRequest`):

| Поле | Тип | Обязат. | Описание |
|------|-----|---------|----------|
| email | string (email) | да | Электронная почта (уникальная) |
| password | string | да | Пароль (минимум 1 символ) |
| first_name | string | да | Имя |
| second_name | string | да | Фамилия |
| third_name | string / null | нет | Отчество |

**Пример запроса:**

```json
{
  "email": "user@example.com",
  "password": "secret",
  "first_name": "Иван",
  "second_name": "Иванов",
  "third_name": "Иванович"
}
```

**Ответ 201** (`TokenResponse`):

```json
{
  "access_token": "eyJhbGciOiJIUzI1NiIs...",
  "token_type": "bearer",
  "expires_in": 43200,
  "user": {
    "user_uuid": "937f0b56-6edc-4aa4-ad87-ee099985933d",
    "email": "user@example.com",
    "first_name": "Иван",
    "second_name": "Иванов",
    "third_name": "Иванович",
    "exp": 0,
    "competension_score": 0
  }
}
```

**Ошибка 400.** Если email уже зарегистрирован:
`{"detail": "Email already registered"}`.

### POST /api/v1/auth/login

> **Назначение.** Вход пользователя и выдача нового JWT-токена.

**Тело запроса** (`LoginRequest`):

| Поле | Тип | Обязат. | Описание |
|------|-----|---------|----------|
| email | string (email) | да | Электронная почта |
| password | string | да | Пароль |

**Пример запроса:**

```json
{ "email": "user@example.com", "password": "secret" }
```

**Ответ 200** (`TokenResponse`) — структура аналогична регистрации.

**Ошибка 401.** `{"detail": "Invalid email or password"}`.

---

### GET /api/v1/auth/me

> **Назначение.** Вернуть данные текущего пользователя по Bearer-токену.

**Требует авторизации:** да.

**Ответ 200:**

```json
{
  "user_uuid": "937f0b56-6edc-4aa4-ad87-ee099985933d",
  "email": "user@example.com",
  "first_name": "Иван",
  "second_name": "Иванов",
  "third_name": "Иванович",
  "exp": 65,
  "competension_score": 0
}
```

---

### POST /api/v1/auth/logout

> **Назначение.** Выход. JWT «stateless» — сервер ничего не хранит, поэтому
> клиенту достаточно удалить токен у себя (localStorage / память). Эндпоинт
> оставлен для совместимости API и всегда возвращает успех.

**Ответ 200:** `{"message": "Logged out"}`.

---

## Группа: Scenarios

### GET /api/v1/scenarios

> **Назначение.** Получить список всех сценариев.

**Требует авторизации:** да.

**Ответ 200** — массив `ScenarioResponse`:

```json
[
  {
    "scenario_uuid": "8f3a...",
    "name": "Обход вагона",
    "description": "Стандартный сценарий обхода вагона"
  }
]
```

| Поле | Тип | Описание |
|------|-----|----------|
| scenario_uuid | string | Идентификатор сценария |
| name | string | Название |
| description | string / null | Описание |

---

### POST /api/v1/scenarios/finish

> **Назначение.** Завершить прохождение сценария и сохранить результат.
> Сервер **сам** генерирует `result_id` (UUID) и привязывает результат к
> `user_uuid` авторизованного пользователя (из JWT-токена). Игра присылает
> только игровые показатели — указывать идентификатор результата или
> пользователя не нужно.

**Требует авторизации:** да.

**Тело запроса** (`ScenarioFinishRequest`):

| Поле | Тип | Обязат. | Ограничение | Описание |
|------|-----|---------|-------------|----------|
| passenger_loyality | integer | да | ≥ 0 | Лояльность пассажиров |
| security_rating | integer | да | ≥ 0 | Рейтинг безопасности |
| duration_playtime | integer | да | ≥ 0 | Длительность прохождения (сек.) |
| result_json | объект / null | нет | — | Произвольные данные результата |

**Пример запроса:**

```json
{
  "passenger_loyality": 7,
  "security_rating": 8,
  "duration_playtime": 120,
  "result_json": { "score": 99, "level": "expert" }
}
```

**Ответ 200** (`ScenarioFinishResponse`):

```json
{
  "id": "04412246-5c83-47cd-af0a-811c819db639",
  "user_uuid": "937f0b56-6edc-4aa4-ad87-ee099985933d",
  "scenario_uuid": null,
  "passenger_loyality": 7,
  "security_rating": 8,
  "time_end": "2026-09-27T18:17:22.648528",
  "duration_playtime": 120,
  "result_json": { "score": 99, "level": "expert" },
  "exp_earned": 65,
  "exp_total": 65
}
```

| Поле | Тип | Описание |
|------|-----|----------|
| id | string | `result_id`, сгенерированный сервером |
| user_uuid | string | Кому принадлежит результат (из токена) |
| scenario_uuid | string / null | Сценарий (сейчас `null`, т.к. игра его не передаёт) |
| passenger_loyality | integer | Повторяет запрос |
| security_rating | integer | Повторяет запрос |
| time_end | string / null | Время завершения |
| duration_playtime | integer | Повторяет запрос |
| result_json | объект / null | Повторяет запрос |
| exp_earned | integer | Получено опыта за попытку |
| exp_total | integer | Итоговый опыт пользователя |

**Формула опыта.** `exp_earned = 50 (база) + security_rating + passenger_loyality`.
В примере: `50 + 8 + 7 = 65`. Начисленный опыт прибавляется к общему опыту
пользователя (`exp_total`).

---

### POST /api/v1/scenarios/leaderboard

> **Назначение.** Таблица лидеров по сценарию (сортировка по минимальному
> времени прохождения).

**Требует авторизации:** да.

**Тело запроса** (`LeaderboardRequest`):

| Поле | Тип | Обязат. | Описание |
|------|-----|---------|----------|
| scenario_uuid | UUID | да | Идентификатор сценария |

**Пример запроса:**

```json
{ "scenario_uuid": "8f3a5c1e-1111-2222-3333-444455556666" }
```

**Ответ 200** — массив `LeaderboardEntry`:

```json
[
  {
    "first_name": "Иван",
    "second_name": "Иванов",
    "duration_playtime": 120,
    "passenger_loyality": 7,
    "security_rating": 8,
    "time_end": "2026-09-27T18:17:22.648528"
  }
]
```

**Ошибка 404**, если сценарий не найден.

---

### GET /api/v1/scenarios/results

> **Назначение.** Результаты текущего пользователя.

**Требует авторизации:** да.

**Ответ 200** — массив `ScenarioResultOut`:

```json
[
  {
    "id": "04412246-5c83-47cd-af0a-811c819db639",
    "user_uuid": "937f0b56-6edc-4aa4-ad87-ee099985933d",
    "scenario_uuid": null,
    "scenario_name": null,
    "scenario_description": null,
    "passenger_loyality": 7,
    "security_rating": 8,
    "time_end": "2026-09-27T18:17:22.648528",
    "duration_playtime": 120,
    "result_json": { "score": 99 }
  }
]
```

---

### GET /api/v1/scenarios/results/scenario/{scenario_uuid}

> **Назначение.** Все результаты по конкретному сценарию.

**Требует авторизации:** да.

**Параметр пути:** `scenario_uuid` (UUID сценария).

**Ответ 200** — массив `ScenarioResultOut`.

---

### GET /api/v1/scenarios/results/id/{result_id}

> **Назначение.** Получить конкретный результат по его `id`.

**Требует авторизации:** да.

**Параметр пути:** `result_id` (UUID результата).

**Ответ 200** — `ScenarioResultOut`. **Ошибка 404**, если результат не найден.

---

### GET /api/v1/scenarios/results/{user_uuid}

> **Назначение.** Результаты указанного пользователя.

**Требует авторизации:** да.

**Параметр пути:** `user_uuid` (UUID пользователя).

**Ответ 200** — массив `ScenarioResultOut`.

---

### GET /api/v1/scenarios/{scenario_uuid}

> **Назначение.** Детали сценария и лучший результат текущего пользователя.

**Требует авторизации:** да.

**Параметр пути:** `scenario_uuid` (UUID сценария).

**Ответ 200** (`ScenarioDetailOut`):

```json
{
  "scenario_uuid": "8f3a5c1e-1111-2222-3333-444455556666",
  "name": "Обход вагона",
  "description": "Стандартный сценарий обхода вагона",
  "best_result": null,
  "completed_attempts": 3
}
```

`best_result` — лучший завершённый результат (или `null`), отбор по убыванию
`security_rating`, затем `passenger_loyality`.

---

## Группа: Admin

### POST /api/v1/admin/seed

> **Назначение.** Наполнить БД стартовым набором сценариев (seed). Если
> сценарии уже существуют — возвращает сообщение без дублирования.

**Ответ 200.** Пример при первом запуске:

```json
{
  "message": "Seed data created successfully",
  "scenarios": [
    { "scenario_uuid": "...", "name": "Train Walkthrough", "description": "Standard train car walkthrough scenario" },
    { "scenario_uuid": "...", "name": "Emergency Evacuation", "description": "Emergency evacuation procedure training" },
    { "scenario_uuid": "...", "name": "Passenger Service", "description": "Customer service and passenger interaction" }
  ]
}
```

Если данные уже есть: `{"message": "Seed data already exists (3 scenarios)"}`.

---

## Служебные эндпоинты

### GET /

> Сводка о сервисе: версия, адрес документации, список групп эндпоинтов.

### GET /health

> Проверка работоспособности: `{"status": "ok"}`.

---

## Работа со Swagger UI

1. Откройте `/docs`.
2. Для защищённых эндпоинтов нажмите **Authorize** (замок) и вставьте токен,
   полученный из `/api/v1/auth/register` или `/api/v1/auth/login`.
3. Заполните тело запроса и нажмите **Execute**.
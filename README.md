[![Spotify API Automated Tests](https://github.com/quasidasein/spotify-api-postman-suite/actions/workflows/newman.yml/badge.svg)](https://github.com/quasidasein/spotify-api-postman-suite/actions/workflows/newman.yml)

# Spotify Web API — Postman Test Suite 🎵

Набор автоматизированных проверок и запросов для контрактного тестирования, валидации схемы, обработки ошибок и бизнес-логики Spotify Web API с интеграцией в CI/CD пайплайн.

---

## 🛠 Технический стек

* **Инструменты:** Postman, Newman (CLI Runner)
* **CI/CD & Среда:** GitHub Actions, Ubuntu Linux
* **Протоколы:** REST, HTTP/HTTPS, OAuth 2.0 (Client Credentials Flow)
* **Скрипты:** JavaScript (Chai Assertion Library)
* **Методологии тест-дизайна:** Boundary Value Analysis (BVA), Equivalence Partitioning (EP), Contract Testing, Schema Validation

---

## 🔐 Архитектура безопасности и авторизация

В проекте реализованы практики безопасности (Security Best Practices) по защите чувствительных данных:

### 1. Изоляция учетных данных (Secrets Management)
* Секретные ключи приложения не захардкожены в запросах.
* Публичный экспорт коллекции содержит только безопасные плейсхолдеры (`YOUR_CLIENT_ID`, `YOUR_CLIENT_SECRET`).
* В GitHub Actions ключи изолированы через зашифрованные переменные окружения (**GitHub Secrets: `SPOTIFY_CLIENT_ID` / `SPOTIFY_CLIENT_SECRET`**).

### 2. Автоматизация токенов (Request Chaining)
Реализован автоматический флоу получения и обновления Bearer-токена (TTL: 3600 сек). В запросе авторизации выполняется Post-response скрипт, который парсит JSON-ответ и динамически обновляет переменную `access_token`:

```javascript
const response = pm.response.json();
if (response.access_token) {
    pm.collectionVariables.set("access_token", response.access_token);
}
```

Все последующие эндпоинты коллекции автоматически обращаются к актуальному токену через заголовок `Authorization: Bearer {{access_token}}`.

### 3. Маршрутизация через переменные окружения
* `{{base_url}}` — `https://api.spotify.com`
* `{{auth_url}}` — `https://accounts.spotify.com`

---

## 📁 Структура коллекции и тестовое покрытие

### 1. Позитивное тестирование (Positive Suite)
* **GET Artist by ID** (`GET {{base_url}}/v1/artists/6TsAG8Ve1icEC8ydeHm3C8`)
  * Статус: `200 OK`
  * Валидация наличия обязательных ключей профиля артиста (`id`, `name`).
  * Тестирование производительности: время отклика не превышает допустимый порог (`responseTime < 1000ms`).

### 2. Негативное тестирование и валидация параметров (Negative & Security Suite)

#### Аутентификация и маршрутизация ID:
* **[Negative] GET Artist - Invalid ID** (`GET {{base_url}}/v1/artists/invalid_id_123`)
  * Статус: `400 Bad Request` | Валидация формата: `message === "Invalid base62 id"`.
* **[Negative] GET Artist - Non-existing ID** (`GET {{base_url}}/v1/artists/0000000000000000000000`)
  * Статус: `404 Not Found` | Проверка ресурса: `message === "Resource not found"`.
* **[Negative] GET Artist - Invalid / Expired Token**
  * Статус: `401 Unauthorized` | Заголовок `Authorization: Bearer <tampered_token>` | Валидация сообщения безопасности: `"Missing/invalid/expired access token"`.
* **[403] GET Artist - Top Tracks (Forbidden Scope)**
  * Статус: `403 Forbidden` | Подтверждение недоступности ручки без пользовательского скоупа авторизации.

#### Граничные значения пагинации (BVA: limit & offset):
* **[Negative] GET Artist's Albums - Limit Below Minimum (0)**
  * `limit=0` → `400 Bad Request` (`message === "Invalid limit"`).
* **[Negative] GET Artist's Albums - Limit Above Maximum (11)**
  * `limit=11` → `400 Bad Request` (валидация выхода за диапазон 1–10).
* **[Negative] GET Artist's Albums - Invalid Limit Format**
  * `limit=invalid` → `400 Bad Request` (обработка строкового типа в числовом поле).
* **[Boundary] GET Artist's Albums - Out of Bounds Offset**
  * `offset=100000` → `200 OK`.
  * Возврат пустого массива (`items: []`).
  * Отключение следующей страницы (`next === null`) и сохранение валидной ссылки на предыдущую (`previous !== null`).
  * Эхо-валидация смещения (`offset === 100000`).

#### Заметки по архитектуре валидации (Exploratory Findings):
* **Санитаризация (Input Trimming & Fallback):** передача пустого значения или пробелов (`?limit=%20`) не ломает сервис, а откатывается к поведению по умолчанию (`limit=5`, `200 OK`).
* **Сетевой уровень (Edge Gateway):** передача недопустимых символов URL-протокола (`&&^*&^`) отсекается обратным прокси на сетевом уровне со статусом `400 Bad Request` до передачи в ядро бэкенда.

### 3. Контрактное тестирование схемы (Contract Testing)
* **GET Artist's Albums** (`GET {{base_url}}/v1/artists/6TsAG8Ve1icEC8ydeHm3C8/albums`)
  * Статус: `200 OK`
  * Валидация метаданных пагинации: `href`, `limit`, `next`, `offset`, `previous`, `total`, `items` (проверка типов данных: string, number, array).
  * Граничное состояние начального смещения (`offset = 0`): `previous === null`.
  * Валидация контракта объекта альбома (`SimplifiedAlbumObject`): проверка обязательных атрибутов (`id`, `name`, `album_type`, `total_tracks`, `release_date`, `type`, `uri`, `artists`).
  * Строгая проверка Enums:
    * `album_type` ∈ `["album", "single", "compilation"]`
    * `release_date_precision` ∈ `["year", "month", "day"]`
    * `type === "album"`

---

## 🐛 Выявленные дефекты спецификации (Contract Drift / Bug Reports)

В ходе валидации контракта `GET /v1/artists/{id}/albums` на соответствие официальной спецификации Spotify Web API обнаружены расхождения между документацией и реальным бэкендом:

| Поле / Параметр | Статус в спецификации | Фактическое поведение API | Описание проблемы |
| :--- | :--- | :--- | :--- |
| `available_markets` | Required, Deprecated | Отсутствует в теле ответа | Поле выведено из эксплуатации на сервере, но в схеме ошибочно отмечено как обязательное. Ломает строгие автотесты. |
| `album_group` | Required, Deprecated | Отсутствует в теле ответа | Устаревшее свойство удалено из ответа API, однако спецификация требует его наличия. |
| `limit` (query param) | Minimum: 1, но Range: 0 - 10 | 400 Bad Request при `limit=0` | Внутреннее противоречие документации: диапазон разрешает 0, но сервер отвергает запрос с ошибкой `"Invalid limit"`. |

---

## ⚙️ Автоматизация и CI/CD пайплайн

В проекте настроен пайплайн **GitHub Actions** (`.github/workflows/newman.yml`), запускающий сьют тестов в headless-режиме:
* **Среда выполнения:** Виртуальная машина Ubuntu Latest (Node.js 18).
* **Триггеры:** Автоматический запуск при каждом `push` или `pull_request` в ветку `main`, а также возможность ручного запуска через `workflow_dispatch`.
* **Безопасность:** Передача учетных данных через GitHub Repository Secrets.

---

## 🚀 Инструкция по запуску

### Вариант 1. Автоматический запуск в консоли (через Newman)

1. Клонируйте репозиторий:
   ```bash
   git clone https://github.com/quasidasein/spotify-api-postman-suite.git
cd spotify-api-postman-suite
   ```
2. Установите Newman (требуется Node.js):
   ```bash
   npm install -g newman
   ```
3. Запустите тесты с передачей учетных данных:
   ```bash
   newman run spotify_collection.json \
     --env-var "client_id=ВАШ_CLIENT_ID" \
     --env-var "client_secret=ВАШ_CLIENT_SECRET"
   ```

### Вариант 2. Запуск через Postman GUI

1. Импортируйте `spotify_collection.json` в Postman.
2. В настройках коллекции во вкладке **Variables** вставьте ваши Client ID и Client Secret в колонку **Current Value**.
3. Запустите коллекцию через **Collection Runner**.

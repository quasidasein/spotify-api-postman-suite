# Spotify Web API — Postman Test Suite 🎵

Набор автоматизированных проверок и запросов для тестирования контрактов, обработки ошибок и бизнес-логики Spotify Web API.

## 🛠 Технический стек
* **Инструмент:** Postman
* **Протокол:** REST, OAuth 2.0 (Client Credentials Flow)
* **Скрипты:** JavaScript (Chai Assertion Library)
* **Методологии тест-дизайна:** Boundary Value Analysis (BVA), Equivalence Partitioning (EP), Schema Validation

---

## 🔐 Архитектура безопасности и авторизация
В проекте реализованы практики безопасности (Security Best Practices) по защите чувствительных данных при интеграции со сторонними сервисами:

### 1. Изоляция учетных данных (Secrets Management)
Секретные ключи приложения не захардкожены в теле запросов. Вместо этого используется система переменных Postman:
* Публичный экспорт коллекции содержит только плейсхолдеры (`YOUR_CLIENT_ID`, `YOUR_CLIENT_SECRET`) в колонке **Shared Value**.
* Для локального запуска реальные ключи подставляются в **Value**, что исключает их утечку в публичный репозиторий.

### 2. Автоматизация токенов (Request Chaining)
Реализован автоматический флоу обновления Bearer-токена (TTL: 3600 сек). В запросе авторизации написан `Post-response` скрипт, который парсит JSON ответа сервера и динамически перезаписывает переменную коллекции `access_token`:

```javascript
const response = pm.response.json();
if (response.access_token) {
    pm.collectionVariables.set("access_token", response.access_token);
}
```

Все последующие эндпоинты автоматически используют актуальный токен в заголовке `Authorization: Bearer {{access_token}}`.

### 3. Маршрутизация через переменные
Базовые домены вынесены в переменные коллекции для удобного управления окружением:
* `{{base_url}}` — `https://api.spotify.com`
* `{{auth_url}}` — `https://accounts.spotify.com`

---

## 📁 Структура коллекции и тестовое покрытие

### 1. Позитивное тестирование (Positive Suite)

* **GET Artist by ID**
  * **Эндпоинт:** `GET {{base_url}}/v1/artists/6TsAG8Ve1icEC8ydeHm3C8`
  * **Статус:** `200 OK`
  * **Проверки:**
    * Валидация наличия обязательных ключей профиля артиста (`id`, `name`).
    * Тестирование производительности: время отклика не превышает допустимый порог (`responseTime < 1000ms`).

### 2. Негативное тестирование и валидация параметров (Negative & Security Suite)

#### Аутентификация и маршрутизация ID
* **[Negative] GET Artist - Invalid ID**
  * **Эндпоинт:** `GET {{base_url}}/v1/artists/invalid_id_123`
  * **Ожидаемый статус:** `400 Bad Request`
  * **Проверки:** соответствие `status === 400` и сообщения `message === "Invalid base62 id"`.

* **[Negative] GET Artist - Non-existing ID**
  * **Эндпоинт:** `GET {{base_url}}/v1/artists/0000000000000000000000`
  * **Ожидаемый статус:** `404 Not Found`
  * **Проверки:** соответствие `status === 404` и сообщения `message === "Resource not found"`.

* **[Negative] GET Artist - Invalid / Expired Token**
  * **Эндпоинт:** `GET {{base_url}}/v1/artists/0000000000000000000000`
  * **Заголовки:** Модифицированный `Authorization: Bearer <tampered_token>`
  * **Ожидаемый статус:** `401 Unauthorized`
  * **Проверки:** соответствие `status === 401` и валидация сообщения системы безопасности: `"Missing/invalid/expired access token"`.

* **[403] GET Artist - Top Tracks (Forbidden Scope)**
  * **Эндпоинт:** `GET {{base_url}}/v1/artists/6TsAG8Ve1icEC8ydeHm3C8/top-tracks`
  * **Ожидаемый статус:** `403 Forbidden`
  * **Проверки:** подтверждение недоступности ручки без пользовательского скоупа авторизации.

#### Граничные значения пагинации (BVA & Type Validation: `limit`)
* **[Negative] GET Artist's Albums - Limit Below Minimum (0)**
  * **Эндпоинт:** `GET {{base_url}}/v1/artists/6TsAG8Ve1icEC8ydeHm3C8/albums?limit=0`
  * **Ожидаемый статус:** `400 Bad Request`
  * **Проверки:** валидация ошибки выхода за нижнюю границу (`message === "Invalid limit"`).

* **[Negative] GET Artist's Albums - Limit Above Maximum (11)**
  * **Эндпоинт:** `GET {{base_url}}/v1/artists/6TsAG8Ve1icEC8ydeHm3C8/albums?limit=11`
  * **Ожидаемый статус:** `400 Bad Request`
  * **Проверки:** валидация ошибки выхода за верхнюю границу допустимого диапазона 1–10.

* **[Negative] GET Artist's Albums - Invalid Limit Format**
  * **Эндпоинт:** `GET {{base_url}}/v1/artists/6TsAG8Ve1icEC8ydeHm3C8/albums?limit=invalid`
  * **Ожидаемый статус:** `400 Bad Request`
  * **Проверки:** обработка строкового типа данных в целочисленном параметре (`status === 400`, `message === "Invalid limit"`).

> **Заметка по архитектуре валидации (Exploratory Findings):**
> * **Санитаризация (Input Trimming & Fallback):** передача пустого значения или строки из пробелов (`?limit=%20`) не ломает запрос, а откатывается к поведению по умолчанию (`Default: limit=5`, статус `200 OK`).
> * **Сетевой уровень (Edge Gateway):** передача недопустимых символов протокола URL (`&&^*&^`) отсекается обратным прокси на сетевом уровне со статусом `400 Bad Request` до передачи запроса в ядро бэкенда.

### 3. Контрактное тестирование схемы и пагинации (Contract Testing)

* **GET Artist's Albums**
  * **Эндпоинт:** `GET {{base_url}}/v1/artists/6TsAG8Ve1icEC8ydeHm3C8/albums`
  * **Статус:** `200 OK`
  * **Проверки метаданных пагинации:**
    * Валидация обязательных корневых полей: `href`, `limit`, `next`, `offset`, `previous`, `total`, `items`.
    * Валидация типов данных (`number`, `string`, `array`).
    * Проверка граничного состояния начального смещения (`offset = 0`): поле `previous` строго равно `null`.
  * **Проверка контракта объекта альбома (`SimplifiedAlbumObject`):**
    * Защитная проверка: массив `items` не пуст.
    * Валидация наличия обязательных атрибутов сущности: `id`, `name`, `album_type`, `total_tracks`, `release_date`, `release_date_precision`, `type`, `uri`, `external_urls`, `images`, `artists`.
    * Проверка допустимых значений (Enums):
      * `album_type` $\in$ `["album", "single", "compilation"]`
      * `release_date_precision` $\in$ `["year", "month", "day"]`
      * `type === "album"`

---

## 🐛 Выявленные дефекты спецификации (Contract Drift / Bug Reports)

В ходе валидации контракта ответа `GET /v1/artists/{id}/albums` на соответствие официальной спецификации Spotify Web API обнаружены расхождения между документацией и фактическим поведением бэкенда:

| Поле / Параметр | Статус в документации | Фактическое поведение API | Описание проблемы |
| :--- | :--- | :--- | :--- |
| `available_markets` | `Required`, `Deprecated` | **Отсутствует в ответе** | Свойство выведено из эксплуатации на бэкенде, но в схеме документации ошибочно сохраняет флаг `Required`. Вызывает падение автотестов строгой валидации. |
| `album_group` | `Required`, `Deprecated` | **Отсутствует в ответе** | Свойство устарело и не возвращается сервером, однако спецификация требует его обязательного присутствия. |
| `limit` *(query param)* | `Minimum: 1`, но `Range: 0 - 10` | **Внутреннее противоречие** | В текстовом описании минимальным порогом заявлена единица (`Minimum: 1`), однако блок допустимого диапазона разрешает ноль (`Range: 0 - 10`). При передаче `limit=0` сервер возвращает ошибку `400 Bad Request`. |

---

## 🚀 Как запустить коллекцию локально
1. Склонируйте репозиторий и импортируйте JSON-файл коллекции в Postman.
2. Создайте приложение в [Spotify Developer Dashboard](https://developer.spotify.com/dashboard).
3. Перейдите во вкладку **Variables** коллекции в Postman.
4. Вставьте ваши `Client ID` и `Client Secret` в колонку **Value**.
5. Выполните запрос `POST Request an access token`, чтобы инициализировать сессию.
6. Запустите коллекцию через **Collection Runner**.

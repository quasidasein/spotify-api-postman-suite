# Spotify Web API — Postman Test Suite 🎵

Набор автоматизированных проверок и запросов для тестирования контрактов, обработки ошибок и бизнес-логики Spotify Web API.

## 🛠 Технический стек
* **Инструмент:** Postman
* **Протокол:** REST, OAuth 2.0 (Client Credentials Flow)
* **Скрипты:** JavaScript (Chai Assertion Library)

---

## 🔐 Архитектура безопасности и авторизация
В проекте реализованы практики безопасности (Security Best Practices) по защите чувствительных данных при интеграции со сторонними сервисами:

### 1. Изоляция учетных данных (Secrets Management)
Секретные ключи приложения не захардкожены в теле запросов. Вместо этого используется система переменных Postman:
* Публичный экспорт коллекции содержит только плейсхолдеры (`YOUR_CLIENT_ID`, `YOUR_CLIENT_SECRET`) в колонке **Shared Value**.
* Для локального запуска реальные ключи подставляются в **Value**, что исключает их утечку в публичный репозиторий.

### 2. Автоматизация токенов (Request Chaining)
Реализован автоматический флоу обновления Bearer-токена (TTL: 3600 сек). 
В запросе авторизации написан `Post-response` скрипт, который парсит JSON ответа сервера и динамически перезаписывает переменную коллекции `access_token`:

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

### 2. Негативное тестирование и безопасность (Negative & Security Suite)

* **[Negative] GET Artist - Invalid ID**
  * **Эндпоинт:** `GET {{base_url}}/v1/artists/invalid_id_123`
  * **Ожидаемый статус:** `400 Bad Request`
  * **Проверки:**
    * Статус ответа `400`.
    * Проверка структуры ошибки и сообщения: `jsonData.error.message === "Invalid base62 id"`.
    * Соответствие внутреннего кода: `jsonData.error.status === 400`.

* **[Negative] GET Artist - Non-existing ID**
  * **Эндпоинт:** `GET {{base_url}}/v1/artists/0000000000000000000000`
  * **Ожидаемый статус:** `404 Not Found`
  * **Проверки:**
    * Статус ответа `404`.
    * Валидация сообщения: `jsonData.error.message === "Resource not found"`.
    * Соответствие внутреннего кода: `jsonData.error.status === 404`.

* **[Negative] GET Artist - Invalid / Expired Token**
  * **Эндпоинт:** `GET {{base_url}}/v1/artists/0000000000000000000000`
  * **Заголовки:** Модифицированный `Authorization: Bearer <tampered_token>`
  * **Ожидаемый статус:** `401 Unauthorized`
  * **Проверки:**
    * Статус ответа `401`.
    * Валидация сообщения системы безопасности: `jsonData.error.message === "Missing/invalid/expired access token"`.
    * Соответствие внутреннего кода: `jsonData.error.status === 401`.

* **[403] GET Artist - Top Tracks (Forbidden Scope)**
  * **Эндпоинт:** `GET {{base_url}}/v1/artists/6TsAG8Ve1icEC8ydeHm3C8/top-tracks`
  * **Ожидаемый статус:** `403 Forbidden`
  * **Проверки:**
    * Статус ответа `403` при попытке доступа к ресурсу с недостаточными правами токена.
    * Валидация тела ответа: `jsonData.error.message === "Forbidden"`.

### 3. Контрактное тестирование схемы и пагинации (Contract Testing)

* **GET Artist's Albums**
  * **Эндпоинт:** `GET {{base_url}}/v1/artists/6TsAG8Ve1icEC8ydeHm3C8/albums`
  * **Статус:** `200 OK`
  * **Проверки пагинации (Pagination Metadata):**
    * Валидация обязательных корневых полей: `href`, `limit`, `next`, `offset`, `previous`, `total`, `items`.
    * Валидация типов данных (`number`, `string`, `array`).
    * Проверка граничного состояния первого смещения (`offset = 0`): поле `previous` строго равно `null`.
  * **Проверка контракта объекта альбома (Album Object Contract):**
    * Защитная проверка: подтверждение непустого массива `items`.
    * Валидация наличия обязательных атрибутов сущности: `id`, `name`, `album_type`, `total_tracks`, `release_date`, `release_date_precision`, `type`, `uri`, `external_urls`, `images`, `artists`.
    * Проверка перечислений (Enums):
      * `album_type` $\in$ `["album", "single", "compilation"]`
      * `release_date_precision` $\in$ `["year", "month", "day"]`
      * `type === "album"`

---

## 🐛 Выявленные дефекты спецификации (Contract Drift / Bug Reports)

В ходе валидации контракта ответа `GET /v1/artists/{id}/albums` на соответствие официальной спецификации Spotify Web API обнаружены расхождения между документацией и фактическим поведением бэкенда:

| Поле | Статус в документации | Фактическое поведение API | Описание проблемы |
| :--- | :--- | :--- | :--- |
| `available_markets` | `Required`, `Deprecated` | **Отсутствует в ответе** | Свойство выведено из эксплуатации на бэкенде, но в схеме документации ошибочно сохраняет флаг `Required`. Вызывает падение автотестов строгой валидации. |
| `album_group` | `Required`, `Deprecated` | **Отсутствует в ответе** | Свойство устарело и не возвращается сервером, однако спецификация требует его обязательного присутствия. |
| `limit` *(query param)* | `Minimum: 1`, но `Range: 0 - 10` | **Внутреннее противоречие** | В текстовом описании минимальным порогом заявлена единица (`Minimum: 1`), однако блок допустимого диапазона разрешает ноль (`Range: 0 - 10`). Спецификация противоречит сама себе. |

---

## 🚀 Как запустить коллекцию локально
1. Склонируйте репозиторий и импортируйте JSON-файл коллекции в Postman.
2. Создайте приложение в [Spotify Developer Dashboard](https://developer.spotify.com/dashboard).
3. Перейдите во вкладку **Variables** коллекции в Postman.
4. Вставьте ваши `Client ID` и `Client Secret` в колонку **Value**.
5. Выполните запрос `POST Request an access token`, чтобы инициализировать сессию.
6. Запустите коллекцию через **Collection Runner**.

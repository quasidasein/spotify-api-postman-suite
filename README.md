# Spotify Web API — Postman Test Suite 🎵

Набор автоматизированных проверок и запросов для тестирования контрактов и бизнес-логики Spotify Web API.

## 🛠 Технический стек
* **Инструмент:** Postman
* **Протокол:** REST, OAuth 2.0 (Client Credentials Flow)
* **Скрипты:** JavaScript (Chai Assertion Library)

## 🔐 Архитектура безопасности и авторизация
В проекте реализованы практики безопасности (Security Best Practices) по защите чувствительных данных при интеграции со сторонними сервисами:

### 1. Изоляция учетных данных (Secrets Management)
Секретные ключи приложения не захардкожены в теле запросов. Вместо этого используется система переменных Postman:
* Публичный экспорт коллекции содержит только плейсхолдеры (`YOUR_CLIENT_ID`, `YOUR_CLIENT_SECRET`) в колонке **Shared Value**.
* Для локального запуска реальные ключи подставляются в **Value**, что исключает их утечку в публичный репозиторий.

### 2. Автоматизация токенов (Request Chaining)
Реализован автоматический флоу обновления Bearer-токена (TTL: 3600 сек). 
В запросе авторизации написан `Post-response` скрипт, который парсит JSON ответа сервера и динамически перезаписывает переменную коллекции `access_token`.

```javascript
const response = pm.response.json();
if (response.access_token) {
    pm.collectionVariables.set("access_token", response.access_token);
}
```

Все последующие эндпоинты автоматически используют актуальный токен в заголовке `Authorization: Bearer {{access_token}}`.

### 3. Маршрутизация через переменные
Базовые домены вынесены в переменные коллекции для удобного управления окружением:
* `{{base_url}}` — `[https://api.spotify.com](https://api.spotify.com)`
* `{{auth_url}}` — `[https://accounts.spotify.com](https://accounts.spotify.com)`

---

## 📁 Структура коллекции и тестовое покрытие

### 1. Позитивное тестирование (Positive Suite)

* **GET Artist by ID**
  * **Эндпоинт:** `GET {{base_url}}/v1/artists/6TsAG8Ve1icEC8ydeHm3C8`
  * **Статус:** `200 OK`
  * **Проверки (Assertions):**
    * Валидация наличия обязательных ключей профиля артиста (`id`, `name`).
    * Тестирование производительности: время отклика не превышает допустимый порог (`responseTime < 1000ms`).

### 2. Негативное тестирование и безопасность (Negative & Security Suite)

* **[Negative] GET Artist - Invalid ID**
  * **Эндпоинт:** `GET {{base_url}}/v1/artists/invalid_id_123`
  * **Ожидаемый статус:** `400 Bad Request`
  * **Техника тест-дизайна:** Эквивалентное разбиение (передача некорректного формата Base62).
  * **Проверки:**
    * `pm.response.to.have.status(400)`
    * Наличие объекта ошибки: `jsonData.error`
    * Точное системное сообщение: `jsonData.error.message === "Invalid base62 id"`
    * Код статуса в теле ответа: `jsonData.error.status === 400`

* **[Negative] GET Artist - Non-existing ID**
  * **Эндпоинт:** `GET {{base_url}}/v1/artists/0000000000000000000000`
  * **Ожидаемый статус:** `404 Not Found`
  * **Техника тест-дизайна:** Анализ граничных значений (синтаксически валидный 22-значный ID, отсутствующий в БД).
  * **Проверки:**
    * `pm.response.to.have.status(404)`
    * `jsonData.error.status === 404`
    * Сообщение об отсутствии записи: `jsonData.error.message === "Resource not found"`

* **[Negative] GET Artist - Invalid / Expired Token**
  * **Эндпоинт:** `GET {{base_url}}/v1/artists/0000000000000000000000`
  * **Заголовки:** Модифицированный заголовок `Authorization: Bearer <tampered_token>`
  * **Ожидаемый статус:** `401 Unauthorized`
  * **Техника тест-дизайна:** Security Testing (инъекция невалидного токена).
  * **Проверки:**
    * `pm.response.to.have.status(401)`
    * `jsonData.error.status === 401`
    * Ответ подсистемы аутентификации: `jsonData.error.message === "Missing/invalid/expired access token"`

* **[403] GET Artist - Top Tracks (Forbidden Scope)**
  * **Эндпоинт:** `GET {{base_url}}/v1/artists/6TsAG8Ve1icEC8ydeHm3C8/top-tracks`
  * **Ожидаемый статус:** `403 Forbidden`
  * **Техника тест-дизайна:** Проверка ролевой модели доступа (RBAC) и ограничений прав доступа клиентского токена.
  * **Проверки:**
    * `pm.response.to.have.status(403)`
    * `jsonData.error.status === 403`
    * Системное уведомление о запрете действия: `jsonData.error.message === "Forbidden"`

### 3. Контрактное тестирование схемы и пагинации (Schema Validation)

* **GET Artist's Albums**
  * **Эндпоинт:** `GET {{base_url}}/v1/artists/6TsAG8Ve1icEC8ydeHm3C8/albums`
  * **Статус:** `200 OK`
  * **Проверки структуры (Contract Testing):**
    * Проверка обязательных корневых полей: `href`, `limit`, `next`, `offset`, `previous`, `total`, `items`.
  * **Специфика типизации (JavaScript / Chai):**
    * Валидация числовых счетчиков (`limit`, `offset`, `total`) через тип `number`.
    * Проверка ссылок навигации (`href`, `next`) на тип `string`.
    * Проверка контейнера сущностей `items` на тип `array`.
  * **Граничные состояния (Boundary Value Analysis):**
    * Для начальной страницы выборки (`offset = 0`) поле навигации назад строго валидируется на значение `null` (`pm.expect(jsonData.previous).to.be.null`).

---

## 🚀 Как запустить коллекцию локально
1. Склонируйте репозиторий и импортируйте JSON-файл коллекции в Postman.
2. Создайте приложение в [Spotify Developer Dashboard](https://developer.spotify.com/dashboard).
3. Перейдите во вкладку **Variables** коллекции в Postman.
4. Вставьте ваши `Client ID` и `Client Secret` в колонку **Value**.
5. Выполните запрос `POST Request an access token`, чтобы инициализировать сессию.

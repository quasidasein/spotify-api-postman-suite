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
* `{{base_url}}` — `https://api.spotify.com`
* `{{auth_url}}` — `https://accounts.spotify.com`

## 🚀 Как запустить коллекцию локально
1. Склонируйте репозиторий и импортируйте JSON-файл коллекции в Postman.
2. Создайте приложение в [Spotify Developer Dashboard](https://developer.spotify.com/dashboard).
3. Перейдите во вкладку **Variables** коллекции в Postman.
4. Вставьте ваши `Client ID` и `Client Secret` в колонку **Value**.
5. Выполните запрос `POST Request an access token`, чтобы инициализировать сессию.

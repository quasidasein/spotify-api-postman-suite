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

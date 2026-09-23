# Dolphin{anty} API: Полное руководство разработчика

Данный документ описывает архитектуру и эндпоинты Dolphin{anty} для интеграции в систему FBStocker.

---

## 1. Архитектура Dolphin API

Dolphin{anty} состоит из двух независимых API-контуров:

| Параметр | **Cloud API** (Управление данными) | **Local API** (Запуск и Автоматизация) |
| :--- | :--- | :--- |
| **Базовый URL** | `https://dolphin-anty-api.com` | `http://localhost:3001` (порт по умолчанию) |
| **Где работает** | Облачный сервер Dolphin | Локальный десктопный клиент на вашем ПК |
| **За что отвечает** | Профили, папки, теги, прокси, куки, синхронизация | Физический запуск Chromium, выдача CDP-порта, остановка |
| **Авторизация** | Заголовок `Authorization: Bearer <JWT_TOKEN>` | Метод `POST /v1.0/auth/login-with-token` с тем же JWT |

> [!NOTE]
> **JWT Токен**: выдаётся в личном кабинете Dolphin на странице API / Настройки. Хранится в базе данных FBStocker в таблице `AppSetting` (`key = "dolphin_token"`).

---

## 2. Авторизация и проверка связи

### А. Локальная авторизация демона (порт 3001)
Перед вызовом команд запуска/остановки профилей локальный процесс Dolphin должен валидировать ваш токен:

```http
POST http://localhost:3001/v1.0/auth/login-with-token
Content-Type: application/json

{
  "token": "eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiIs..."
}
```

**Ответ (200 OK)**:
```json
{
  "success": true
}
```

### Б. Проверка статуса локального API
```http
GET http://localhost:3001/v1.0/browser_profiles
```
Если порт отвечает (даже кодом 401 при отсутствии авторизации), значит десктопное приложение запущено.

---

## 3. Папки профилей (Folders)

Dolphin предоставляет полноценный CRUD для организации профилей по папкам.

### 3.1. Получение списка всех папок
Возвращает список всех существующих папок с прикрепленными к ним профилями:

```http
GET https://dolphin-anty-api.com/folders
Authorization: Bearer <JWT_TOKEN>
```

**Ответ (200 OK)**:
```json
{
  "data": [
    {
      "id": 359814,
      "name": "wUS",
      "type": "personal",
      "emoji": "file_folder",
      "order": 1,
      "isPinned": false,
      "browserProfilesData": [
        {
          "id": 873283233,
          "name": "wUS-AUG26-01",
          "platform": "windows",
          "status": null
        }
      ]
    },
    {
      "id": 408127,
      "name": "тесты реги",
      "type": "personal",
      "emoji": "file_folder",
      "order": 2,
      "browserProfilesData": [
        { "id": 859430860, "name": "USA-SEP26" }
      ]
    }
  ]
}
```

### 3.2. Создание новой папки
```http
POST https://dolphin-anty-api.com/folders
Authorization: Bearer <JWT_TOKEN>
Content-Type: application/json

{
  "name": "Фарм США Сентябрь",
  "emoji": "tractor"
}
```

**Ответ (200 OK)**:
```json
{
  "success": true,
  "data": {
    "id": 415890,
    "name": "Фарм США Сентябрь",
    "emoji": "tractor"
  }
}
```

### 3.3. Перемещение профилей в папку (Attach)
Привязывает один или несколько профилей к указанной папке:

```http
POST https://dolphin-anty-api.com/folders/mass/attach-profiles
Authorization: Bearer <JWT_TOKEN>
Content-Type: application/json

{
  "folderId": 359814,
  "browserProfileIds": [873283233, 859430860]
}
```

**Ответ (200 OK)**:
```json
{
  "success": true
}
```

### 3.4. Извлечение профилей из папки (Detach)
Убирает профили из папки обратно в общий список:

```http
POST https://dolphin-anty-api.com/folders/mass/detach-profiles
Authorization: Bearer <JWT_TOKEN>
Content-Type: application/json

{
  "browserProfileIds": [873283233]
}
```

### 3.5. Получить только ID профилей из папки
```http
GET https://dolphin-anty-api.com/folders/{folderId}/profile-ids
Authorization: Bearer <JWT_TOKEN>
```

**Ответ (200 OK)**:
```json
{
  "success": true,
  "data": [873283233, 859430860, 859431153]
}
```

### 3.6. Переименование и удаление папки
* **Переименовать**:
  ```http
  PUT https://dolphin-anty-api.com/folders/{folderId}
  Content-Type: application/json

  { "name": "Новое имя", "emoji": "fire" }
  ```
* **Удалить папку**:
  ```http
  DELETE https://dolphin-anty-api.com/folders/{folderId}?deleteProfiles=0
  ```
  *(Если `deleteProfiles=0` — профили останутся и вернутся в корень; если `deleteProfiles=1` — профили удалятся вместе с папкой).*

---

## 4. Профили браузера (Browser Profiles)

### 4.1. Получение полной информации о профиле
Возвращает полную карточку профиля со всеми техническими параметрами, расширениями, закладками и прокси:

```http
GET https://dolphin-anty-api.com/browser_profiles/{browserProfileId}
Authorization: Bearer <JWT_TOKEN>
```

**Ключевые поля ответа**:
```json
{
  "data": {
    "id": 873283233,
    "name": "wUS-AUG26-01",
    "status": null,
    "platform": "windows",
    "platformVersion": "10.0.0",
    "screen": { "resolution": "1536x864" },
    "cpu": { "value": 8 },
    "memory": { "value": 16 },
    "timezone": { "mode": "manual", "value": "America/New_York" },
    "locale": { "mode": "manual", "value": "en_US" },
    
    "proxy": {
      "id": 633919133,
      "name": "mob_R",
      "type": "socks5",
      "host": "bproxy.site",
      "port": "15675",
      "login": "rYR6uH",
      "password": "...",
      "changeIpUrl": "https://aproxy.site/?proxy_key=f6c06be6...",
      "ip": "172.56.35.196"
    },

    "bookmarks": [
      { "name": "FB", "url": "http://fb.com" },
      { "name": "Outlook", "url": "https://outlook.live.com/mail/" }
    ],

    "extensions": [
      {
        "url": "https://chromewebstore.google.com/detail/fbstocker-key-hub/...",
        "type": "chromeWebStore"
      },
      {
        "url": "https://anty-assets.s3.eu-central-1.amazonaws.com/extensions/...",
        "type": "file"
      }
    ]
  }
}
```

### 4.2. Поиск и фильтрация профилей (Курсорная пагинация)
```http
GET https://dolphin-anty-api.com/browser_profiles/list-cursor?limit=50&folderIds[]=359814
Authorization: Bearer <JWT_TOKEN>
```

**Доступные параметры фильтрации**:
* `limit` (число, например `50`)
* `folderIds[]` (ID одной или нескольких папок)
* `tags[]` (теги)
* `statuses[]` (статусы Dolphin)
* `query[]` (поиск по названию профиля)
* `cursor` (указатель на следующую страницу)

### 4.3. Создание профиля
```http
POST https://dolphin-anty-api.com/browser_profiles
Authorization: Bearer <JWT_TOKEN>
Content-Type: application/json

{
  "name": "FB-AUTO-01",
  "platform": "windows",
  "browserType": "anty",
  "mainWebsite": "facebook",
  "proxy": {
    "type": "http",
    "host": "ru.resigw.com",
    "port": 2333,
    "login": "pa343891d43c965",
    "password": "...",
    "changeIpUrl": "https://mobileproxy.space/reload.html?..."
  },
  "fingerprint": {
    "osVersion": "10",
    "screen": { "resolution": "1920x1080" }
  }
}
```

---

## 5. Запуск и Автоматизация (Local API `localhost:3001`)

Управление запущенным браузером происходит через локальный процесс на порту 3001.

### 5.1. Запуск браузера с поддержкой автоматизации
```http
GET http://localhost:3001/v1.0/browser_profiles/{browserProfileId}/start?automation=1
```

> [!IMPORTANT]
> Параметр `?automation=1` обязателен! Без него Dolphin запустит браузер в обычном ручном режиме, не открывая порт DevTools.

**Ответ (200 OK)**:
```json
{
  "success": true,
  "automation": {
    "port": 54321,
    "wsEndpoint": "ws://127.0.0.1:54321/devtools/browser/7f8a9b..."
  }
}
```

### 5.2. Подключение Playwright к запущенному браузеру
Получив `wsEndpoint`, Playwright подключается по протоколу Chrome DevTools Protocol (CDP):

```typescript
import { chromium } from 'playwright';

async function connectToDolphin(wsEndpoint: string) {
  const browser = await chromium.connectOverCDP(wsEndpoint);
  const context = browser.contexts()[0]; // профиль уже содержит куки и открытые вкладки
  const page = context.pages()[0] || await context.newPage();

  await page.goto('https://facebook.com');
  console.log('Успешное управление профилем Dolphin!');
}
```

### 5.3. Остановка профиля и синхронизация
После завершения сценария автоматизации профиль **обязательно** нужно остановить через API, чтобы кэш, куки и локальные данные выгрузились обратно в облако Dolphin:

```http
GET http://localhost:3001/v1.0/browser_profiles/{browserProfileId}/stop
```

**Ответ (200 OK)**:
```json
{
  "success": true
}
```

---

## 6. Быстрые примеры вызовов (Node.js / TypeScript)

### Выгрузка всех профилей конкретной папки:
```typescript
async function getProfilesInFolder(folderId: number, token: string) {
  const res = await fetch(`https://dolphin-anty-api.com/folders/${folderId}/profile-ids`, {
    headers: { Authorization: `Bearer ${token}` }
  });
  const { data: profileIds } = await res.json();
  return profileIds; // Массив ID профилей, например [873283233, ...]
}
```

### Перемещение профиля в другую папку:
```typescript
async function moveProfileToFolder(profileId: number, folderId: number, token: string) {
  const res = await fetch('https://dolphin-anty-api.com/folders/mass/attach-profiles', {
    method: 'POST',
    headers: {
      Authorization: `Bearer ${token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      folderId,
      browserProfileIds: [profileId]
    })
  });
  return res.json();
}
```

### Полный цикл автоматизации профиля:
```typescript
import { chromium } from 'playwright';

async function runAutomation(profileId: number) {
  // 1. Старт в локальном API
  const startRes = await fetch(`http://localhost:3001/v1.0/browser_profiles/${profileId}/start?automation=1`);
  const startData = await startRes.json();

  if (!startData.success) {
    throw new Error(`Не удалось запустить профиль: ${startData.msg}`);
  }

  const wsEndpoint = startData.automation.wsEndpoint;

  // 2. Подключение Playwright
  const browser = await chromium.connectOverCDP(wsEndpoint);
  const page = browser.contexts()[0].pages()[0] || await browser.contexts()[0].newPage();

  // 3. Выполнение работы в FB
  await page.goto('https://facebook.com');
  // ... действия ...

  // 4. Отключение и закрытие
  await browser.close();
  await fetch(`http://localhost:3001/v1.0/browser_profiles/${profileId}/stop`);
}
```

# Архитектурный манифест экосистемы «FB Stocker Combine»
> **Версия:** 1.1.0  
> **Статус:** Утверждено  
> **Целевая платформа:** Локальный сервер (Intel Xeon 22C/44T, Windows 11, NVMe Storage)  
> **Паттерн:** Hub & Spoke (Центральный хаб управления + изолированные воркеры)  
> **Физическая локация:** `D:\dev\fb\AMS\`

---

## 1. Концепция и организация репозиториев

Экосистема создаётся для полного цикла производства и монетизации качественных аккаунтов Facebook:
1. **Авторегистрация на физическом Android-смартфоне** через ADB и аппаратный спуфинг (наивысший траст Meta).
2. **Фарминг и прогрев в антидетект-браузере** (Dolphin{anty} / Octo / AdsPower через универсальный адаптер) на мощном многопоточном сервере.
3. **Централизованный учёт, склад и управление (AMS — Account Management System)**.

### Физическая структура каталогов на диске:
Все три независимых сервиса и общая документация собраны в каталоге `D:\dev\fb\AMS\`:

```text
D:\dev\fb\AMS\
├── docs/                        # Центральные архитектурные манифесты и регламенты
│   └── SYSTEM_ARCHITECTURE.md
├── fbstocker-ams/               # [ХАБ] База данных SQLite, REST API, Web UI CRM (Порты: 4000, 5173)
│   ├── apps/server/             # Fastify REST API, Prisma ORM, WebSocket
│   ├── apps/web/                # React 19 + Vite веб-панель управления
│   └── packages/shared/         # Общие DTO и интерфейсы (AccountData, ProxyChannel и др.)
├── fbstocker-autoreg-adb/       # [ВОРКЕР 1] Авторег на реальном смартфоне (Порт: 3000)
│   ├── src/core/adb/            # UI Automator, ScreenDetector, DeviceCleaner
│   ├── src/flows/               # Регистрация FB Lite / Katana, 2FA, Email Binding
│   └── data/accounts.json       # Локальный буфер на случай недоступности Хаба
└── fbstocker-autofarm/          # [ВОРКЕР 2] Автофарм и прогрев через CDP (Порт: 5000)
    ├── src/providers/           # Адаптеры антидетект-браузеров (Dolphin, Octo, AdsPower)
    ├── src/scenarios/           # Сценарии прогрева (Reels, Feed, Groups, Cookies)
    └── src/orchestrator/        # Диспетчер батчей и очередей Playwright
```

### Схема взаимодействия сервисов (Hub & Spoke):

```
                               ┌────────────────────────────────────────┐
                               │       AMS (CENTRAL HUB & CRM)          │
                               │           (fbstocker-ams)              │
                               │                                        │
                               │  • Порт API: 4000 | Порт UI: 5173      │
                               │  • СУБД: SQLite (WAL-mode, 50k+ accs)  │
                               │  • Задачи: База, статусы, фильтры,     │
                               │    CRUD, экспорт в шопы, пул прокси    │
                               └───────────▲────────────────┬───────────┘
                                           │                │
             POST /api/v1/accounts/import  │                │ POST /api/v1/accounts/farm-queue/claim
             (Свежесозданный готовый акк)  │                │ PATCH /api/v1/accounts/:id/farm-report
                                           │                │
            ┌──────────────────────────────┴───┐    ┌───────▼──────────────────────────┐
            │       ADB AUTOREG WORKER         │    │         AUTOFARM WORKER          │
            │     (fbstocker-autoreg-adb)      │    │       (fbstocker-autofarm)       │
            │                                  │    │                                  │
            │ • Порт веб-пульта: 3000          │    │ • Порт диспетчера: 5000          │
            │ • Физический Xiaomi Mi A1 (USB)  │    │ • Универсальный Browser Adapter  │
            │ • Magisk Root + sing-box TUN     │    │ • Playwright через CDP           │
            │ • HeroSMS + Firstmail IMAP       │    │ • Прогрев: лента, Reels, куки    │
            │ • Fallback buffer: accounts.json │    │ • 10–20 параллельных профилей    │
            └──────────────────────────────────┘    └─────────────────┬────────────────┘
                                                                      │
                                                    ┌─────────────────▼────────────────┐
                                                    │  ANTIDETECT BROWSER ADAPTER      │
                                                    │  (Dolphin / Octo / AdsPower)     │
                                                    │  Local API: порт 3001 / custom   │
                                                    └──────────────────────────────────┘
```

---

## 2. Карта портов (Zero Collision Policy)

Чтобы сервисы никогда не конфликтовали за порты и не мешали друг другу:

| Сервис / Процесс | Порт | Протокол | Назначение |
| :--- | :---: | :---: | :--- |
| **ADB Autoreg Panel** | **`3000`** | HTTP / WS | Инженерный пульт управления смартфоном, логами и отладкой шагов регистрации. |
| **Antidetect Local API** | **`3001`** | HTTP REST | Локальный API запущенного антидетект-клиента (Dolphin порт 3001, Octo 58888, AdsPower 50325). |
| **AMS Central Hub (API)** | **`4000`** | HTTP REST / WS | Главный сервер базы данных, REST API для воркеров, WebSocket событий. |
| **Autofarm Orchestrator** | **`5000`** | HTTP / Internal | Фоновый диспетчер фарма (запуск/остановка очередей прогрева, воркеры Playwright). |
| **AMS Web CRM UI** | **`5173`** | HTTP | Веб-интерфейс CRM для оператора (Vite Dev Server или статика). |

---

## 3. Архитектура базы данных (SQLite WAL)

Серверная машина (22 ядра Xeon, быстрый NVMe-накопитель) идеально подходит для **SQLite в режиме WAL (Write-Ahead Logging)**.

### Преимущества:
1. **50 000 аккаунтов** в SQLite весят около **60–150 МБ** (включая JSON-куки и метаданные). Это мгновенные выборки (1–3 мс).
2. **Нулевой порог обслуживания:** База хранится в одном файле `apps/server/prisma/combine.sqlite`. Резервная копия делается обычным копированием файла.
3. **Отсутствие оверхеда:** Не требует тяжелого фонового сервера PostgreSQL/MySQL на Windows.

### КРИТИЧЕСКОЕ ПРАВИЛО: Single Writer Pattern (Один пишущий процесс)
> Чтобы на Windows не возникало блокировок файла (`SQLITE_BUSY: database is locked`), **напрямую к SQLite-файлу подключается ТОЛЬКО процесс AMS (Хаб)**.  
> Воркер Авторега и воркер Фарма **НИКОГДА не открывают `.sqlite` файл напрямую** — они делают HTTP-запросы в API Хаба. Внутри Fastify все записи выполняются последовательно через Prisma.

#### Конфигурация движка SQLite в AMS:
```sql
PRAGMA journal_mode = WAL;         -- Разрешает параллельное чтение во время записи
PRAGMA synchronous = NORMAL;       -- Максимальная производительность на SSD/NVMe
PRAGMA busy_timeout = 10000;       -- Ожидание снятия блокировки до 10 секунд
PRAGMA foreign_keys = ON;          -- Контроль связей таблиц
```

---

## 4. Паттерн «Адаптер антидетект-браузера» (Pluggable Browser Engine)

Для обеспечения независимости от конкретного софта (Dolphin{anty}) внедряется интерфейс **`IBrowserProfileProvider`**.  
Ни воркер фарма, ни сценарии Playwright не работают напрямую со специфичными вызовами Dolphin — они работают только через адаптер.

```typescript
export interface BrowserSessionInfo {
  profileId: string;
  wsEndpoint: string;    // WebSocket CDP URL для прямого подключения Playwright
  httpPort?: number;     // Порт CDP
}

export interface CreateProfileDTO {
  name: string;
  proxy?: {
    type: 'http' | 'socks5';
    host: string;
    port: number;
    login?: string;
    password?: string;
  };
  cookies?: any[];
  userAgent?: string;
}

export interface IBrowserProfileProvider {
  readonly providerName: 'DOLPHIN' | 'OCTO' | 'ADSPOWER' | 'PLAYWRIGHT_STEALTH';
  
  checkHealth(): Promise<boolean>;
  createProfile(data: CreateProfileDTO): Promise<{ profileId: string }>;
  startProfile(profileId: string): Promise<BrowserSessionInfo>;
  stopProfile(profileId: string): Promise<boolean>;
  deleteProfile(profileId: string): Promise<boolean>;
  exportCookies(profileId: string): Promise<any[]>;
  importCookies(profileId: string, cookies: any[]): Promise<boolean>;
}
```

### Как Playwright подключается к браузеру:
```typescript
// Воркер фарма работает одинаково с ЛЮБЫМ антидетектом:
const session = await browserProvider.startProfile(account.browserProfileId);

// Подключение через Chrome DevTools Protocol:
const browser = await chromium.connectOverCDP(session.wsEndpoint);
const context = browser.contexts()[0];
const page = context.pages()[0] || await context.newPage();

// Выполнение сценария прогрева (скролл, просмотр Reels, реакции)...
await runWarmupScenario(page);

// Сохранение кук и закрытие:
const updatedCookies = await context.cookies();
await browser.close();
await browserProvider.stopProfile(account.browserProfileId);
```

---

## 5. Единый контракт данных (State Machine)

### Жизненный цикл аккаунта:
```
[UNREGISTERED] 
       │
       ▼ (Autoreg Worker начинает сессию на телефоне)
[IN_PROGRESS] 
       │
       ▼ (Телефон подтвердил SMS, пароль, аватар, 2FA, Email)
[READY_FOR_FARM] 
       │  (Передано в AMS по REST API: POST /api/v1/accounts/import)
       ▼
   [FARMING] ◄─────────────────────────┐
       │ (Воркер крутит прогрев через) │ (Повторные циклы прогрева)
       ▼                               │
   [WARMED]  ──────────────────────────┘ (Прогрет, готов к отлежке/следующему дню)
       │
       ▼ (После N дней успешного фарма)
[READY_FOR_SALE] / [EXPORTED]

* В случае чекпоинта на любом этапе: [CHECKPOINT_SMS] / [CHECKPOINT_SELFIE] / [BANNED]
```

### Схема сущности `AccountData` (согласованный стандарт):
```typescript
export interface AccountData {
  id?: number;                          // Primary Key в AMS
  sessionId: string;                    // Внутренний ID сессии (fb_lite_timestamp)
  fullName?: string | null;             // Имя Фамилия
  dob?: string | null;                  // Дата рождения (ДД.ММ.ГГГГ)
  gender?: 'male' | 'female' | null;
  login: string;                        // Email или телефон для входа
  loginPassword?: string | null;        // Пароль аккаунта
  
  // Безопасность и контакты
  phone?: string | null;                // Номер регистрации (или "DELETED")
  email?: string | null;                // Привязанная почта Firstmail
  emailPassword?: string | null;        // Пароль от почты
  totpSecret?: string | null;           // 2FA TOTP секрет (Base32)
  backupCodes?: string[] | null;        // Резервные коды 2FA
  codes?: string | null;                // Сырая строка кодов

  // Метаданные железа и прокси (MobileProxy)
  proxyRegIp?: string | null;           // IP регистрации
  proxyId?: number | string | null;     // ID прокси в базе AMS или MobileProxy
  proxyComment?: string | null;         // Имя/комментарий прокси
  proxyGeo?: string | null;             // Нативное гео ("US", "PL")
  proxyOperator?: string | null;        // Нативный оператор ("AT&T (US)")

  // Браузерный профиль (Адаптер)
  browserProfileId?: string | null;     // ID профиля в антидетекте
  dolphinProfileId?: string | null;     // Алиас для обратной совместимости
  browserProvider?: string | null;      // "DOLPHIN" | "OCTO" | "ADSPOWER"
  cookies?: any[] | string | null;      // Сессионные куки JSON
  userAgent?: string | null;            // User-Agent профиля

  // Статус и трекинг
  status?: string | null;               // "READY_FOR_FARM", "FARMING", "WARMED"
  farmStatus?: FarmStatus | null;       // Внутренний статус AMS
  crmStatusId?: number | null;
  farmDaysCount?: number | null;        // Сколько дней аккаунт на прогреве
  lastFarmDate?: string | null;         // Время крайнего сеанса фарма
  apkVersion?: 'lite' | 'katana' | null;
  country?: string | null;              // Таргет ГЕО (US, PL)
  geo?: string | null;
  createdAt?: string | null;
  notes?: string | null;
  comment?: string | null;
  accountHubFormat?: string | null;     // Формат для выгрузки в магазин
  fanpageLinks?: string[] | null;
}
```

---

## 6. Спецификация межсервисного API (AMS REST Endpoints)

### 1. Приём нового аккаунта от Авторега
- **Эндпоинт:** `POST /api/v1/accounts/import`
- **Отправитель:** `fbstocker-autoreg-adb`
- **Тело запроса:** Объект `AccountData` со статусом `READY_FOR_FARM`.
- **Поведение:** 
  1. AMS валидирует наличие обязательных полей (`login`, `sessionId`).
  2. Идемпотентно сохраняет запись в SQLite (upsert по `sessionId`).
  3. Пишет запись в `AccountLog` (`level: SUCCESS`, `action: AUTOREG_IMPORT`).
  4. Транслирует событие по WebSocket в UI CRM.

### 2. Захват очереди на фарм (Lease / Claiming)
- **Эндпоинт:** `POST /api/v1/accounts/farm-queue/claim`
- **Отправитель:** `fbstocker-autofarm`
- **Тело запроса:**
  ```json
  {
    "limit": 10,
    "workerId": "worker-xeon-01",
    "leaseMinutes": 30,
    "geo": "US"
  }
  ```
- **Поведение:** 
  1. Атомарно выбирает N аккаунтов со статусом `READY` / `WARMUP_*`.
  2. Устанавливает статус `FARMING`, фиксирует `leasedUntil = now() + 30m`.
  3. Исключает состояние гонки: никакой другой поток не получит эти же аккаунты.

### 3. Отчёт о сессии фарма
- **Эндпоинт:** `PATCH /api/v1/accounts/:sessionId/farm-report`
- **Отправитель:** `fbstocker-autofarm`
- **Тело запроса:**
  ```json
  {
    "status": "WARMED",
    "cookies": [...],
    "actionsCompleted": ["watch_reels_5m", "scroll_feed_3m", "like_post_2"],
    "farmDurationMinutes": 8
  }
  ```
- **Поведение:** AMS обновляет куки, переводит статус в `WARMED`, обновляет `lastFarmAt` и фиксирует лог.

---

## 7. Отказоустойчивость и безопасность (Self-Healing)

1. **Защита от потери аккаунтов при падении AMS:**
   - В `fbstocker-autoreg-adb` локальный файл `data/accounts.json` работает как **Fallback Journal**.
   - Если AMS выключен, авторег спокойно завершает сессию на телефоне, сохраняет результат в JSON с флагом `syncedToHub: false`. При появлении связи по таймеру отправляет накопленные записи.

2. **Защита от зависших задач фарма (Lease Expiration):**
   - Если воркер фарма внезапно упал во время прогрева, через 30 минут (`leaseMinutes`) AMS автоматически возвращает зависшие аккаунты из `FARMING` обратно в `READY` с предупреждением в логе.

3. **Изоляция сбоев смартфона:**
   - Отвалы USB, зависания UI Automator или демона adbd локализованы исключительно внутри процесса `fbstocker-autoreg-adb` и не аффектят CRM и браузерный фарм.

4. **Правило мобильных прокси (1 Port = 1 Active Thread):**
   - Ротация IP (`changeIpUrl`) на мобильном прокси меняет адрес для всего порта.
   - Нельзя запускать два параллельных профиля на одном порту мобильного прокси, иначе смена IP одним профилем сломает сессию второму. AMS контролирует флаг `isBusy` у прокси-каналов.

---

## 8. Статус реализации модулей

| Модуль | Репозиторий | Статус готовности | Ближайшие задачи |
| :--- | :--- | :---: | :--- |
| **AMS Central Hub** | `fbstocker-ams` | **95%** | Финализация эндпоинта claim очереди, проверка веб-интерфейса CRM. |
| **Autoreg ADB** | `fbstocker-autoreg-adb` | **85%** | Доводка шагов привязки 2FA и Firstmail, добавление вызова `syncWithAms()`. |
| **Autofarm** | `fbstocker-autofarm` | **10%** | Реализация адаптера антидетекта (`DolphinProvider`), Playwright-скриптов прогрева. |

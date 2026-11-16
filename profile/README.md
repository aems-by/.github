<img width="2125" height="740" alt="logo" src="https://github.com/user-attachments/assets/f44fc79a-4337-4e6c-b608-44b8ea0a51ad" />


Это система управления мероприятиями и соревнованиями. Все процессы автоматизированы, что позволяет экономить ресурсы и время на организацию проведения соревнований разного уровня от городского до республиканского.

# Документация проекта «Автоматизированная система управления мероприятиями»

Полное техническое описание системы проведения соревнований (чемпионатов):
структура, архитектура, рабочие процессы и руководство по внесению правок.

Дата обзора: 14.11.2026.

---


## 1. Обзор

Система для организации и проведения соревнований (олимпиад, чемпионатов)
с номинациями, командами, групповыми этапами, плей-офф, швейцарской системой,
зачётами по критериям и генерацией протоколов/сертификатов.

Состоит из двух независимых приложений:

| Каталог | Стек | Назначение |
|---|---|---|
| `championships-backend-dev/` | Python 3.12, FastAPI, SQLAlchemy 2, Pydantic v2, Alembic, PostgreSQL | REST API, почта, PDF/Excel, турнирная логика |
| `championships-frontend-dev/` | React 18, CRA (react-app-rewired), Redux Toolkit, Ant Design 5, react-router 6, i18next | Веб-интерфейс (SPA) |

Прод-окружение (из `.env`): backend живёт под `https://e.catalyst.b-cloud.xyz`
с префиксом `/backend` (`uvicorn --root-path /backend`), фронт — на
`https://b-cloud.xyz`. CORS разрешает origin `b-cloud.xyz` и localhost:3000/3001.

---

## 2. Структура репозитория

```
SERVER/
├── championships-backend-dev/       # Python/FastAPI бэкенд
│   ├── main.py                      # Точка входа: app, CORS, middleware, роуты, статика
│   ├── urls.py                      # Enum URLs — ВСЕ пути API в одном месте
│   ├── config.py                    # Pydantic Settings (чтение .env), private.pem
│   ├── .env                         # Секреты (не коммитить!), example.env — шаблон
│   ├── private.pem                  # RSA-приватный ключ для паролей (!см. §9)
│   ├── create_admin.py              # Одноразовое создание админа (запуск вручную)
│   ├── alembic/                     # Миграции (versions/ — 12 шт.)
│   ├── alembic.ini
│   ├── db/
│   │   ├── database.py              # engine, SessionLocal, Base
│   │   ├── models/                  # ~33 модели SQLAlchemy (таблицы)
│   │   ├── schemas/                 # Pydantic-схемы (вход/выход API), по папкам на сущность
│   │   └── crud/                    # SQL-запросы, по папкам на сущность (+ general.py)
│   ├── routes/                      # API-слой: <entity>.py (роутер) + <entity>_service.py
│   ├── managers/                    # Бизнес-логика (30+ классов)
│   │   └── nomination_event_appenders/  # Стратегии добавления номинаций по типу турнира
│   ├── dependencies/dependencies.py # get_db(), authorized_only (проверка куки)
│   ├── utils/                       # протоколы (PDF), excel, турнирный граф, утилиты
│   ├── templates/                   # Jinja2-шаблоны писем и протоколов (10 шт.)
│   ├── static/                      # файлы: fonts/, images/, files/ (раздаются как /static/*)
│   ├── telegram_bot/bot.py          # Уведомления об ошибках в Telegram
│   ├── generated_code/              # Replay-скрипты запросов (диагностика 500)
│   ├── middleware/logging_middleware.py  # Мёртвый код (не подключён)
│   ├── tests/                       # pytest (по сути 1 рабочий макро-тест)
│   ├── Dockerfile, .github/workflows/    # Сборка образа, CI (dev/main ветки)
│   └── requirements.txt
│
└── championships-frontend-dev/      # React SPA
    ├── src/
    │   ├── index.jsx                # ReactDOM root
    │   ├── App.jsx                  # Все маршруты <Routes>
    │   ├── ProtectedRoute.jsx       # Ролевая защита маршрутов
    │   ├── Layout.jsx               # Прогрев профиля при старте
    │   ├── const/index.jsx          # КОНСТАНТЫ: ROUTES, роли, API_URL, почты, лимиты
    │   ├── api/                     # Axios-модули по сущностям (auth, event, user, …)
    │   ├── store/                   # Redux Toolkit: users, events, participants, team, language
    │   ├── contexts/MatchesContext.jsx  # Вся логика судейских матчей (группы/плей-офф)
    │   ├── components/              # Страницы/экраны (auth, judgment, participants, …)
    │   ├── modules/                 # Переиспользуемые поля и блоки форм
    │   ├── utils/index.js           # Хелперы: пагинация, RSA, дерево плей-офф, даты
    │   ├── errors/index.js          # Словарь «строка ошибки бэка → ключ перевода»
    │   ├── translations/            # i18n: locales/ru.json, locales/by.json
    │   └── sass/{lightTheme,darkTheme}/  # Темы (переключение через класс .dark-theme)
    ├── public/
    ├── build/                       # Готовая сборка (deploy-артефакт)
    ├── package.json, config-overrides.js  # CRA + алиас @ → src
    ├── Dockerfile, entrypoint.sh    # (!см. §9 — entrypoint сломан)
    └── .env                         # REACT_APP_API_URL, REACT_APP_PUBLIC_KEY, CAPTCHA
```

---

## 3. Backend: как устроен

### 3.1 Точка входа (`main.py`)

Порядок при импорте:

1. Импорт всех роутеров и моделей.
2. `app = FastAPI()` → `Base.metadata.create_all(...)` (создание таблиц, параллельно с Alembic).
3. **CORS**: жёсткий список origin'ов (localhost, b-cloud.xyz, netlify), `allow_credentials=True`.
4. `GET /` — health-check.
5. **HTTP-middleware** (логирование запросов в БД `requests`/`responses`, генерация
   replay-скрипта в `generated_code/code/`, отправка файла в Telegram при 500).
   Работает только когда `debug != "True"` (по умолчанию `debug="True"`, т.е. логирование выключено).
6. Подключение 22 роутеров (`app.include_router`).
7. Монтирование статики: `/static/images`, `/static/files`.
8. `__main__`: telegram-уведомление → `alembic upgrade head` → uvicorn на `0.0.0.0:8000`.

### 3.2 Слоистая архитектура

```
HTTP → routes/<x>/<x>.py (роутер: Depends(authorized_only), Depends(get_db))
     → routes/<x>/<x>_service.py (декод JWT, проверки ролей/владения)
     → managers/<…>.py (бизнес-правила, исключения HTTPException 409/403)
     → db/crud/<x>/<x>.py (SQLAlchemy-запросы)
     → db/models/<…>.py (таблицы)
```

* **`routes/`** — по два файла на сущность: роутер (сигнатуры эндпоинтов) и
  сервис (логика). Паттерн сервиса: `decode_token` → проверка роли/владения → менеджеры.
* **`managers/`** — бизнес-логика и все `raise_exception_if_*` (единый стиль
  ошибок: `HTTPException(409, detail={"error": "<строка>"})`, строки потом
  маппятся на фронте через `src/errors/index.js`).
* **`db/crud/`** — чистый SQL через ORM. Хэши паролей: `CryptContext(bcrypt)`.
* **`db/schemas/`** — Pydantic v2, `from_attributes=True`. Валидации: email
  (`pyisemail`, DNS-проверка), телефон паттерном, `min_length=7` для паролей.
* **`urls.py`** — единый `class URLs(Enum)`: префиксы (`/api/auth`, `/api/event`, …)
  и все пути. **Новый эндпоинт = новый пункт в URLs + роутер.**

### 3.3 Модель данных (главные таблицы)

**Пользователи и доступ**
* `User` — email (unique), hashed_password, ФИО, телефон, организация,
  `role` (`admin` | `judge` | `specialist`), `is_approved`. Связи: tokens,
  events (owner), nominations (owner), judged_events (M2M через `nomination_event_judge`).
* `Token` — PK = сам JWT-токен (серверный отзыв сессий при logout).
* `ResetToken`, `VerificationToken` — одноразовые bcrypt-хэши токенов писем
  (сброс пароля / верификация участника), сроки из настроек.
* `Annotation` — заметки админа.

**Мероприятия**
* `Event` — название (unique), логотип, `participant_question_email`
  (по умолчанию `info@b-cloud.xyz`), место, регламент (файл), даты (строки ISO),
  `published`, `event_level` (republic/regional/district/city/other), описание,
  `owner_id`. M2M: organizers (`event_organizer`), nominations (`nomination_event`).
* `Organizer` — M2M с Event и Certificate.
* `Nomination` — номинация (owner_id), M2M events.
* **`NominationEvent`** — центральная сущность: пара event+nomination (unique),
  флаги стадий (`registration_finished`, `tournament_started`,
  `group_stage_finished`, `play_off_stage_finished`, …) с datetime-полями,
  `type` (`time | criteria | group | olympic | swiss`), `reglament`,
  `race_round_amount`; связи: `groups`, `bracket`, `judges` (M2M),
  `team_participants` (M2M), `criterias`, `race_rounds`.

**Составные и турнирные**
* `Team` (creator_id) ↔ `Participant` (email unique, photo, `is_verified`,
  creator_id) через `TeamParticipant`; M2M с nomination_event через
  `TeamParticipantNominationEvent` (образование, ПО, оборудование, руководитель).
* `Group` / `Bracket` / `Match` — групповой этап и сетка плей-офф
  (счёты, winner, самоссылка `next_bracket_match_id`, `last_result_creator_id`).
* `Criteria` (+ `criteria_team` со score, `criteria_judge`) — зачёт по критериям.
* `RaceRound` — заезды по времени.
* `Software`, `Equipment` — справочники (M2M с TPNE).
* `Certificate` (+ `certificate_organizers`) — сертификаты (файл `certifiacte.py` — опечатка в имени).
* `Request`/`Response` — журнал HTTP middleware.

Схема связей (упрощённо):

```
User ──owner── Event ──M2M── Nomination
  │                │              │
  │                └──────┬───────┘
  │                       ▼
  │               NominationEvent ──M2M── User (judges)
  │                  │    │    │
  │      groups/bracket  │   criterias / race_rounds
  │                  ▼    ▼
  │               Match  Team ──M2M── Participant
  │                        │
  │                        └── TeamParticipantNominationEvent (состав на номинации)
  └── Token / ResetToken / VerificationToken
```

### 3.4 Аутентификация и авторизация

* **Кука `Authorization: Bearer <JWT>`**, `HttpOnly`, `samesite=lax`,
  `secure=False` (!), срок = exp + 1 день.
* JWT: HS256, секрет `jwt_token_secret` из `.env`, payload `{user_id, role, exp, milliseconds}`.
  Токен дополнительно хранится в таблице `tokens` — logout реально отзывает сессию.
* `dependencies/dependencies.py::authorized_only` — только проверяет наличие
  куки формата `Bearer <jwt>`; **валидация JWT** — в `TokenManager.decode_token`
  (проверка подписи, срока и наличия в БД), вызывается внутри сервисов.
* **Роли** (`UserRole`): `admin`, `judge`, `specialist`. Публичная регистрация
  (`POST /api/user/register`) → по умолчанию `judge`, `is_approved=False`.
  Одобрение — только админом (`PATCH /user/user_approve`).
* Вход (`POST /api/auth/login`): RSA-расшифровка пароля (`private.pem`) →
  bcrypt-сверка → проверка `is_approved` → выдача куки.
* Сброс пароля: `POST /system_notice/send_reset_password_link` (публичный) →
  письмо со ссылкой `reset_password?reset_token=` → `POST /auth/reset_password`.
* Первый вход/активация: ссылка `auth/first_login?token=` → `login_by_token`
  (проверяет подпись, одноразовость и **`is_approved`** — см. §8).
* **Типичные проверки ролей** (паттерны для новых эндпоинтов):
  ```python
  decoded = self.__token_manager.decode_token(token, response)
  self.__user_manager.raise_exception_if_user_is_not_admin(decoded.role)   # только admin
  self.__user_manager.raise_exception_if_user_specialist(...)               # запрет specialist
  # владение сущностью:
  self.__validator.validate_user_entity_ownership(...)                      # utils/validation_util.py
  # судья конкретной номинации:
  retriever.get_decoded_token_user_nomination_event_nomination_event_check_judge_in_command(...)
  ```


### 3.5 Турнирная логика (`routes/tournaments`, `managers/tournament.py`)

Типы соревнований в `NominationEvent.type`:

| Тип | Суть | Стадии |
|---|---|---|
| `group` | Групповой этап | start/finish_group_stage → матчи → результаты |
| `olympic` | Группы + плей-офф (сетка) | group → play_off (networkx-граф для протокола) |
| `swiss` | Швейцарская система | start_swiss_stage(rounds) → пары генерируются автоматически (`match_service.__check_and_update_round`) |
| `time` | Зачёт по времени (заезды) | start/finish_time_stage, `race_rounds` |
| `criteria` | Зачёт по критериям (судейские баллы) | start/finish_criteria_stage (⚠️ путь `finish_criteria_state` — опечатка), `criteria` + `criteria_judge` |

Общие правила:
* Выставление результатов (`/api/match/set_*_result[_in_bulk]`) разрешено **только
  судьям номинации** и только в окне правки (30 минут после финиша стадии,
  для time — 15 минут, `RESULTS_EDITABILITY_MINUTES` на фронте).
* Протокол: `GET /api/nomination_event/get_nomination_event_protocol` →
  `utils/protocol.py` рендерит HTML → wkhtmltopdf → защита PDF (pikepdf) →
  `static/files/` (файл удаляется фоном после отдачи).
* Excel: `utils/excel/*` (участники, протокол).
* Сертификаты: `managers/certificate_manager.py` + рассылка письмами.

### 3.6 Файлы

* Загрузка: `managers/file_manager.py` — белые списки расширений
  (`png,jpg,jpeg` / `doc,docx,pdf`), лимиты (128/512 МБ), имя = таймстамп
  (path traversal невозможен — имя не берётся из запроса).
* Фото участников: `utils/photo_manager.py` → WEBP quality=10, лимит папки 200 МБ.
* Раздача: `/static/images/*`, `/static/files/*` (Starlette StaticFiles).

### 3.7 Конфигурация (`config.py` + `.env`)

Ключевые настройки: `jwt_token_secret`, `jwt_token_expiration_time_days=2`,
`reset_token_expiration_time_days=1`, `verification_token_expiration_time_days=30`,
`db*` (postgres), `mail_*` (SMTP), `path_to_wkhtmltopdf`, `path_to_guide`,
`domain_url` (ссылки в письмах), `telegram_bot_token`, `debug`.
`private.pem` читается функцией `get_private_key()` (кэш) — расшифровка паролей.

### 3.8 Запуск и деплой

* Локально: `python main.py` (alembic upgrade head + uvicorn :8000) или
  `uvicorn main:app --reload`.
* Docker: `python:3.12-slim` + wkhtmltopdf + локали; CI (`.github/workflows`)
  при push в `dev`/`main` собирает и пушит образ
  `ncczubronok/championship_{dev,prod}_backend`, ключи подставляются из GitHub Secrets.
* Тесты: `tests/test_group_stage.py` — макро-сценарий против живого сервера
  на 127.0.0.1:8000 (⚠️ использует рабочую БД и заведомо падает: `assert 200 != 200`).

---

## 4. Frontend: как устроен

### 4.1 Стек и конфигурация

* CRA 5 + `react-app-rewired` (правки webpack в `config-overrides.js`: алиас `@` → `src`).
* Скрипты: `start`, `build` (⚠️ `entrypoint.sh` вызывает несуществующий `build:prod` — §9).
* `.env`: `REACT_APP_API_URL=https://b-cloud.xyz`, `REACT_APP_PUBLIC_KEY` (RSA
  для шифрования паролей на клиенте через `jsencrypt`), `REACT_APP_CAPTCHA_TOKEN`
  (Yandex SmartCaptcha sitekey).
* Все env-переменные читаются в одном месте — `src/const/index.jsx`.

### 4.2 Роутинг (`App.jsx` + `const/index.jsx`)

Константы `ROUTES` (пути, включая функции `PATH(id)`) и `ROUTER_ROUTES`
(сегменты для `<Route>`). Защита — `ProtectedRoute` с `allowedRoles`
(роль берётся из `GET /api/user/profile`):

| Группа маршрутов | Роль |
|---|---|
| `/` (вход), `/events*` (каталог, карточка), `/events/check-certificate`, `/reset_password`, `/auth/first_login`, `/email_verification`, `/logout` | публично |
| `/participants`, `/settings`, `/events/:id/registration` | admin, judge, specialist |
| `/users` | admin |
| `/judgment/events*` (список, создание, настройки) | admin, judge |
| `.../group-stage`, `.../play-off`, `.../time-matches`, `.../criteria`, `/participants/:id`, `/feedback`, `/about` | ⚠️ не обёрнуты в ProtectedRoute (см. §9) |

Внутренние страницы лежат внутри layout `AdminPanel` (боковое меню по ролям).

### 4.3 API-слой (`src/api/`)

* `instance = axios.create({ baseURL: API_URL + "/api", ... })` — **аутентификация
  через куки** (отдельного interceptor Authorization нет).
* Единственный response-interceptor показывает antd-тост из
  `src/errors/index.js` (словарь «строка бэка → ключ i18n»), кроме `/user/profile`.
* Модули: `auth.js`, `user.js`, `event.js`, `participant.js`, `team.js`,
  `judgment.js`/`match.js`, `timeMatches.js`, `competencies.js` (самый
  крупный — criteria + nomination_event + tournaments), `certificates.js`,
  `feedback.js`, `organizers.js`, `notification.js`, `team_participant.js`.
* Пагинация — `fetchWithPagination` (`utils/index.js`): рекурсивно собирает
  страницы по `limit=49`.

### 4.4 Состояние (Redux: `src/store/`)

Слайсы `slice.js + thunk.js + selectors.js`:

* `users` — профиль, список пользователей, судьи, одобрения, logout.
* `events` — текущее событие, фильтры/поиск/дата каталога.
* `participants`, `team` — данные таблиц.
* `language` — `ru` ↔ `by` (хранится в `localStorage.language`, меняет i18n).

`src/contexts/MatchesContext.jsx` — вся логика судейских экранов:
загрузка матчей, таблица очков (3 за победу, 1 за ничью), выставление счёта,
запуск/финиш стадий, автоматическое создание сертификатов после плей-офф.

### 4.5 Основные экраны (`src/components/`)

| Блок | Назначение |
|---|---|
| `auth/` | Вход (RSA-шифрование пароля), регистрация, восстановление пароля, активация, капча |
| `events/` | Публичный каталог и карточка события |
| `eventRegistration/` | Регистрация команд на событие (CRUD команд/участников) |
| `judgment/` | Судейская зона: список событий, `EventSettings` (создание/настройка, 734 стр.), стадии турнира |
| `participants/` | Участники: таблица, карточка, Excel-импорт, глобальный поиск |
| `usersControl/` | Админ-панель пользователей: одобрение заявок, рассылки |
| `checkCertificate/` | Публичная проверка сертификата |
| `userSupport/` | Обратная связь, «О программе», разработчики |
| `userSettings/`, `passwordReset/`, `verification/` | Профиль, смена пароля, верификация |
| `adminPanel/`, `settingsButton/` (тема/язык), `socialLink/`, `yandexMetrika/` | Обвязка |

### 4.6 Стили и i18n

* Темы: `sass/lightTheme` и `sass/darkTheme`; тёмная включается классом
  `.dark-theme` на `<html>` (хранится в `localStorage.theme`).
* Переводы: `translations/locales/{ru,by}.json` (по ~680 строк), ключи
  разделены по секциям (`COMMON`, `EVENTS`, `ERRORS`, …).

---

## 5. Каталог эндпоинтов (префиксы API)

| Префикс | Роутер | Ключевое |
|---|---|---|
| `/api/auth` | `routes/auth` | login, logout, authenticate, first_login, reset_password |
| `/api/user` | `routes/user` | profile, users (admin), register (public), check_email, approve |
| `/api/event` | `routes/event` | CRUD событий, логотип/регламент/фото, публичные списки |
| `/api/nomination` | `routes/nomination` | номинации |
| `/api/nomination_event` | `routes/nomination_event` | ядро: прикрепление номинаций, стадии, протокол, Excel |
| `/api/nomination_event_judge` | `routes/nomination_event_judge` | назначение судей |
| `/api/participant` | `routes/participant` | участники, фото, Excel, email-верификация |
| `/api/team`, `/api/team_participant`, `/api/team_nomination_event`, `/api/team_participant_nomination_event` | соответствующие | команды и составы |
| `/api/tournaments` | `routes/tournaments` | start/finish всех стадий |
| `/api/match` | `routes/match` | матчи, set_*_result (судьи), результаты |
| `/api/race_round`, `/api/criteria` | соответствующие | заезды, критерии |
| `/api/certificate` | `routes/certificate` | сертификаты (⚠️ без auth — §9) |
| `/api/system_notice` | `routes/system_notice` | системные письма (закрыты — §8) |
| `/api/feedback`, `/api/organizer`, `/api/annotation`, `/api/software`, `/api/equipment` | соответствующие | обратная связь, организаторы, заметки, справочники |

Актуальную схему всегда можно получить на `/docs` (Swagger) при запущенном сервере.

---

## 6. Бизнес-процессы (end-to-end)

1. **Создание события**: админ/специалист создаёт Event (multipart: логотип,
   регламент, данные) → добавляет номинации (`append_nomination_for_event_<type>`)
   → назначает судей → публикует.
2. **Регистрация участников**: судья/организатор создаёт участников (или Excel)
   → участники верифицируются по email → создаются команды → команды
   регистрируются на номинации события (с составом, ПО, оборудованием).
3. **Проведение**: старт стадии (`start_*_stage`) → судьи вносят результаты
   (окно правки 15–30 мин) → финиш стадии → переход к следующей.
4. **Итоги**: протокол (PDF), Excel-выгрузки, сетка плей-офф (reactflow на фронте,
   networkx на бэке), сертификаты + рассылка письмами.
5. **Пользователи**: регистрация → одобрение админом → письмо с активацией → вход.

---

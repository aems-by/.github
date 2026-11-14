
![Логотип](logo.png)

Это система управления мероприятиями и соревнованиями. Все процессы автоматизированы, что позволяет экономить ресурсы и время на организацию проведения соревнований разного уровня от городского до республиканского.

## Разработчики

 - Факультет информационных технологий и робототехники учреждения образования «Белорусский национальный технический университет»
 - Государственное учреждение образования «Гимназия №61 г.Минска»

# Документация проекта «Championships» (b-cloud)

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

### 3.5 Почта

* `managers/mail_manager.py` — SMTP_SSL (`mail.b-cloud.xyz:465`), отправитель из
  `.env`: `mail_user_sender` / `mail_system_sender` = `info@b-cloud.xyz`
  (обратная связь и системные письма). Отправка в фоне (`ThreadPoolExecutor`).
* `managers/feedback_manager.py` — рендер Jinja2 из `templates/` (⚠️ без
  `autoescape` — см. §9), генерация PDF протоколов и сертификатов
  (reportlab, шрифты `static/fonts/Montserrat-*`).
* `managers/system_notice_manager.py` — все системные письма (регистрация,
  сброс пароля, добавление судьи, рассылка всем, сертификаты).
* **Эндпоинты рассылок** (`routes/system_notice/`) — закрыты авторизацией
  (см. §8): админские (registration/password/verification/general_mail) и
  обычные «только залогинен» (participant/judge/nomination). Публичными
  остались `send_reset_password_link` и `resend_registration_notice`.

### 3.6 Турнирная логика (`routes/tournaments`, `managers/tournament.py`)

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

### 3.7 Файлы

* Загрузка: `managers/file_manager.py` — белые списки расширений
  (`png,jpg,jpeg` / `doc,docx,pdf`), лимиты (128/512 МБ), имя = таймстамп
  (path traversal невозможен — имя не берётся из запроса).
* Фото участников: `utils/photo_manager.py` → WEBP quality=10, лимит папки 200 МБ.
* Раздача: `/static/images/*`, `/static/files/*` (Starlette StaticFiles).

### 3.8 Конфигурация (`config.py` + `.env`)

Ключевые настройки: `jwt_token_secret`, `jwt_token_expiration_time_days=2`,
`reset_token_expiration_time_days=1`, `verification_token_expiration_time_days=30`,
`db*` (postgres), `mail_*` (SMTP), `path_to_wkhtmltopdf`, `path_to_guide`,
`domain_url` (ссылки в письмах), `telegram_bot_token`, `debug`.
`private.pem` читается функцией `get_private_key()` (кэш) — расшифровка паролей.

### 3.9 Запуск и деплой

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

## 7. Изменения, внесённые 14.11.2026

### 7.1 Почта → `info@b-cloud.xyz` везде

| Файл | Было | Стало |
|---|---|---|
| `frontend/src/const/index.jsx` (`mailZubronok`) | `support@championships.by` | `info@b-cloud.xyz` (mailto на входе и в «Обратной связи») |
| `backend/db/models/event.py` (default `participant_question_email`) | `test@mail.ru` | `info@b-cloud.xyz` |
| `backend/db/schemas/event/event_create.py` (default) | `test@mail.ru` | `info@b-cloud.xyz` |
| `backend/routes/user/users.py` (бывший create_admin) | `test@mail.ru` | эндпоинт удалён (§7.2) |
| `backend/tests/test_group_stage.py`, `test_login.py`, `test_users.py` | `test@mail.ru` и производные | `info@b-cloud.xyz` |
| `frontend/src/translations/locales/ru.json`, `by.json` (подсказка EMAIL_EXAMPLE) | `example@example.com` | `info@b-cloud.xyz` |

Уже бывшие b-cloud-адреса не менялись: `info@b-cloud.xyz` (SMTP-отправитель в `.env`),
`admin@b-cloud.xyz` (`create_admin.py`).

### 7.2 Критические уязвимости (C1–C6 + H5)

1. **C1 — саморегистрация с ролью `admin` и обход одобрения**:
   * `db/schemas/user/register.py`: `role: str` → `role: UserRole` (enum) —
     произвольная строка роли через API невозможна (422 на `role: "admin"`).
   * `routes/auth/auth_service.py::login_by_token`: добавлена проверка
     `raise_exception_if_user_is_not_approved(...)` — вход по токену из письма
     больше не обходит одобрение админом. Флоу не сломан: письма с first_login
     уходят только после approve (UnverifiedUsersTable) или при создании
     админом (`is_approved=True` по умолчанию в `UserCreateSchema`).
2. **C2 — секреты в дереве и Docker-образе**:
   * `backend/.dockerignore`: теперь исключает `.env*`, `example.env`, `*.pem`,
     `*.log`, `generated_code/code`, `venv`, `.git*` (раньше — только `.env`;
     `private.pem` попадал в образ).
   * `frontend/.dockerignore`: исключает `.env*`, `build`, `.git*`.
   * `example.env` очищен — реальные `app_secret_key`/`jwt_token_secret`
     заменены на `CHANGE_ME`-плейсхолдеры.
   * `.gitignore` дополнен: `.env.bak`, `.env.bak2`.
   * Удалены `generated_code/code/request_*.py` — replay-скрипты с **живыми
     куками/токенами** (включая `portainer_api_key`).
   * ⚠️ **Требуется ротация**: `jwt_token_secret`, `app_secret_key`, пароль
     SMTP `info@b-cloud.xyz`, `private.pem` — они всё ещё в `.env`/`private.pem`
     (работа бэкенда от них зависит, вычищать без замены нельзя).
3. **C3 — `GET /api/user/create_admin` (без auth, создавал админа)**:
   эндпоинт удалён из `routes/user/users.py` (+ неиспользуемые импорты,
   пометка в `urls.py`). Тест `test_group_stage.py` переведён на прямое
   создание админа в БД (идемпотентно, как `create_admin.py`).
4. **C4 — неаутентифицированные рассылки писем** (`routes/system_notice/`):
   * `send_general_mail` (рассылка всем!), `send_user_registration_notice`,
     `send_password_changed_notice`, `send_verification_link` —
     `authorized_only` + проверка **admin** в `SystemNoticeService`.
   * `send_participant_registration_notice`, `send_judge_added_notice`,
     `send_nomination_event_registation` — `authorized_only` (любой залогиненный).
   * Оставлены публичными (иначе ломаются флоу): `send_reset_password_link`
     («забыли пароль») и `resend_registration_notice` (активация по токену —
     токен валидируется и одноразов).
5. **C5 — подмена результатов соревнований любым залогиненным**:
   `routes/match/match_service.py` — все 6 методов `set_*_result[_in_bulk]`
   переведены на retriever `..._check_judge_in_command` (тот же, что уже
   использовался в tournaments/criteria): теперь результат может ставить только
   судья конкретной номинации.
6. **C6 — Stored XSS** (quill-HTML → `dangerouslySetInnerHTML`):
   * установлен `dompurify@3.4.18`;
   * `components/events/EventInformation.jsx` — `description` и
     `participation_needs` рендерятся через `DOMPurify.sanitize(...)`.
7. **H5 — учётные данные в логах** (`main.py`): заголовки `cookie`,
   `authorization`, `set-cookie` маскируются как `***` перед записью в таблицы
   `requests`/`responses` и генерацией replay-скриптов (куки больше не
   попадают на диск и в Telegram).

Попутно исправлено: `GET /` больше не падает с 500 (был `print(test)` → NameError
→ на каждый запрос корня 500 + файл в Telegram); `path_params` в логе теперь
реальные параметры пути (был `InstrumentedAttribute`).

Все изменённые Python-файлы проверены `py_compile` (OK).

---

## 8. Что ещё может быть не так (не исправлено)

### Высокий приоритет (следующий шаг)

1. **H1. Без auth изменяются справочники**: `PATCH /api/equipment/equipment`,
   `PATCH /api/software/update` — добавить `authorized_only` + проверку прав.
2. **H2. Сертификаты без auth**: весь `routes/certificate/*` (6 эндпоинтов) —
   чтение PII по email/id и **запись** без токена. Закрыть: GET — токен +
   владение/роль, POST — судья/админ.
3. **H3. IDOR пользователей**: `GET /api/user/user_data?user_id=` отдаёт
   email/телефон/роль любого пользователя любому залогиненному. Требовать
   `user_id == себя или admin`.
4. **H4. IDOR участников**: `get_participant_stats`, `get_is_verified`,
   `get_participants_in_system` (глобальный поиск по всем участникам) —
   добавить `raise_exception_if_owner_wrong`, глобальный поиск — judge/admin.
5. **H7. Нет rate limiting и серверной проверки капчи** — брутфорс
   `/api/auth/login`, `check_email` как оракул существования учёток.
   Решение: `slowapi` + серверная валидация SmartCaptcha (токен сейчас
   проверяется только на клиенте — обходится прямым запросом).
6. **H8. Cookie `secure=False`** + разрешён `http://b-cloud.xyz` в CORS —
   на проде ставить `secure=True` и только HTTPS-оригины, добавить редирект
   HTTP→HTTPS.
7. **Окно восстановления пароля**: `send_reset_password_link` публичный —
   нужен rate limit, иначе бомбинг почты письмами «сброс пароля».

### Безопасность данных/контента

8. **HTML-инъекция в письмах**: `feedback_manager.py` использует Jinja2 **без
   `autoescape`** — пользовательский текст уходит в письма как есть. Включить
   `select_autoescape(["html", "xml"])`.
9. **`enable-local-file-access: True` в wkhtmltopdf** (`file_manager.py`) —
   имя команды/участника с `<iframe src="file:///...">` может прочитать локальный
   файл в PDF (доступен по неаутентифицированному `/api/nomination_event/
   get_nomination_event_protocol`). Убрать опцию или жёстко экранировать поля
   в шаблоне протокола (`|e`).
10. **Ротация секретов** (из C2): `jwt_token_secret`, `app_secret_key`,
    SMTP-пароль, `private.pem`; удалить `.env.bak`, `.env.bak2`, `uvicorn.log`
    и вычистить их из истории git (если репозиторий был публичным/копировался).
11. **`static/` раздаётся без ограничений** — проверить, нет ли в нём PII.
12. **XSS через `javascript:` в поле `reglament`**: `window.open(record.reglament)`
    и `<a href={record.reglament}>` — валидировать `^https?://` на фронте и API.
13. **Дефолтный минимум паролей 7 символов** — поднять до 12 + сложность;
    пароли-примеры в `tests/`, `shell.py`, `create_admin.py` (`Admin12345`).

### Функциональные баги / техдолг

14. **`entrypoint.sh` фронта вызывает `npm run build:prod`, а скрипта нет** —
    Docker-контейнер фронта ничего не собирает и не раздаёт (нужен nginx/serve
    или исправить скрипт).
15. **axios: `credentials: "include"` — невалидный ключ** (axios ждёт
    `withCredentials: true`); фактически куки работают только потому, что фронт
    и API на одном origin. Починить для устойчивости.
16. **Неполная ролевая защита маршрутов фронта** (страницы стадий турнира,
    карточка участника, `/feedback`, `/about`) — обернуть в `ProtectedRoute`.
    (Бэкенд всё равно главный рубеж, но UX и безопаснее, и лучше.)
17. **Middleware**: `return` в `finally` (SyntaxWarning, возможна потеря
    ответа), ошибки возвращают 200 `{"message":"error is logged"}` вместо 500,
    тело запроса логируется целиком (DoS/рост таблиц) — нужен TTL/очистка
    таблиц `requests`/`responses`.
18. **Двойное управление схемой**: `create_all` при импорте + `alembic upgrade`
    при старте — оставить только Alembic.
19. **Мёртвый/опасный код**: `middleware/logging_middleware.py` (циклический
    импорт), `routes/feedback/mail_service.py` (await в синхронной функции),
    `test.py`, `to_delete.txt`, `main.py.bak`, `.env.bak*`,
    `FILE_UPLOADING.UPLOAD.action` → внешний `mockapi.io` (хардкод),
    `ParticipantReCaptcha.jsx` с необъявленной `WEB_SITE_KEY`.
20. **Тесты**: 1 из 4 файлов активен, бьёт по рабочей БД и всегда падает
    (`assert 200 != 200`); тестов фронта нет вообще; CI не гоняет ни линтеры,
    ни тесты (только docker build).
21. **Опечатки в API-путях** (менять только синхронно с фронтом):
    `finish_criteria_state`, `send_nomination_event_registation`,
    `get_event_participant_certifiactes`, имена файлов `certifiacte.py`,
    `nominatuin_event_judge.py`.
22. **Зависимости**: прогнать `pip-audit -r requirements.txt` и `npm audit`;
    внимательно к `Pillow==10.2.0`, `starlette==0.37.2`, `Jinja2==3.1.4`,
    `passlib` (не поддерживается), `react-scripts` (CRA EOL).
23. **`npm install --force` в Dockerfile фронта** — заменить на
    `--legacy-peer-deps` (у проекта реальный конфликт peer-зависимостей:
    `react-image-lightbox` требует React 16/17).
24. **Формат ошибок**: бэк отдаёт `detail={"error": "..."}`, фронт читает
    `error.response.data.detail.error` — любой другой формат (FastAPI-валидация
    422 отдаёт `detail` списком) даёт «Общую ошибку» из словаря `errors/index.js`.
25. **Селекторы Redux возвращают новые объекты** → лишние ререндеры; thunks
    `events`/`participants` кладут в стейт `response.config.data` (тело запроса)
    вместо ответа сервера.

---

## 9. Гайд по внесению правок

### Бэкенд

1. **Новый/изменённый эндпоинт**:
   * путь — в `urls.py` (enum);
   * роутер `routes/<x>/<x>.py` (всегда `Depends(authorized_only)` + `Depends(get_db)`,
     кроме явно публичных);
   * логика — в `routes/<x>/<x>_service.py` (сначала `decode_token`, затем
     проверка роли/владения);
   * бизнес-правила и тексты ошибок — в `managers/<x>.py` (`raise_exception_if_*`);
   * SQL — в `db/crud/<x>/`; Pydantic-схемы — в `db/schemas/<x>/`.
2. **Изменение таблицы**: создать миграцию `alembic revision --autogenerate`
   (в `alembic/env.py` уже подключены все модели) и применить
   `alembic upgrade head`. **Не** полагаться на `create_all`.
3. **Письма**: шаблон — `templates/*.html`, сборка письма —
   `managers/feedback_manager.py`, отправка — `managers/system_notice_manager.py`.
4. **Проверка после правки**: `py -m py_compile <файлы>`; при наличии окружения —
   `python -c "import main"` (создаст таблицы в БД!) и `pytest tests/`.
5. Стиль: flake8 (`ignore: E203,E266,E501,...`), black фактически не настроен
   (файл `tournament_backend.toml` не читается — нужен `pyproject.toml`).

### Фронт

1. **Новый API-вызов** — в `src/api/<module>.js` (через `instance`), реэкспорт
   в `src/api/index.js`.
2. **Новый маршрут** — `ROUTES` + `ROUTER_ROUTES` в `const/index.jsx` и `<Route>`
   в `App.jsx`; защищённый — обернуть в `ProtectedRoute allowedRoles={[...]}`.
3. **Тексты** — обязательно в оба `locales/ru.json` и `by.json` (иначе на
   белорусском будет ключ).
4. **Ошибки бэка** — новую строку-ошибку добавить в `src/errors/index.js`,
   иначе пользователь увидит «Общую ошибку».
5. **Состояние** — новый слайс по образцу `store/<name>/{slice,thunk,selectors,index}.js`.
6. Проверка: `npm run build` (запускать через `npm.cmd` — PowerShell-политика
   блокирует `npm.ps1`), линтеры: `npm run lint:fix`, `npm run prettier`.

### Чего не делать

* Не коммитить `.env`, `*.pem`, `.env.bak*` (уже в gitignore — не выключать).
* Не добавлять эндпоинты без `authorized_only` без явной причины и пометки.
* Не логировать cookie/authorization (см. §7.2, H5).
* Менять опечатки в путях API (`finish_criteria_state` и т.п.) только парой
  с фронтендом, иначе сломаются рабочие вызовы.

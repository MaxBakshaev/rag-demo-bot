# RAG-бот по документам на n8n

Учебный проект: Telegram-бот отвечает на вопросы по внутренним документам компании и указывает источник (файл и страницу). Если ответа в документах нет — честно говорит «Не знаю».

Документы в `docs/` — вымышленная компания ООО «ТехноЛайн», реальных данных в репозитории нет.

## Архитектура

Два потока в одном воркфлоу (`workflows/rag_demo.json`):

**Загрузка:** Form Trigger → HTTP Request «Очистить коллекцию» → Code (разбивка файлов по items) → Qdrant Vector Store (Insert)
с подузлами Default Data Loader (метаданные `source`, `loc.pageNumber`), Recursive Character Text Splitter и Embeddings.

**Ответы:** Telegram Trigger → Switch «Тип сообщения» → AI Agent → Send a text message.
У агента подузлы: Chat Model, Postgres Chat Memory (ключ сессии — `chat.id`) и Qdrant Vector Store в режиме «Retrieve as Tool» (`search_docs`).

Switch обрабатывает служебные сообщения до агента:

| Сообщение | Что происходит |
|---|---|
| `/start` (новый пользователь) | приветствие с примерами вопросов |
| `/new` | удаление истории диалога этого чата из Postgres и подтверждение |
| не текст (фото, стикер, голос) | ответ «понимаю только текст» |
| всё остальное | вопрос уходит агенту |

| Компонент | Выбор |
|---|---|
| LLM | `gemini-3.5-flash-lite` |
| Эмбеддинги | `gemini-embedding-2` (3072 измерения) |
| Хранилище | Qdrant, коллекция `rag_docs` |
| Память диалогов | Postgres Chat Memory, база `rag_memory`, 5 последних сообщений |
| Фрагменты | 800 символов, перекрытие 150 |
| top-k (Limit) | 4 |
| Интерфейс | Telegram-бот |

## Развёртывание

### Что нужно

- self-hosted n8n в Docker с публичным HTTPS-адресом (без него Telegram не доставит сообщения);
- API key Google Gemini ([Google AI Studio](https://aistudio.google.com/));
- Telegram-бот и его токен (создаётся у [@BotFather](https://t.me/BotFather));
- PostgreSQL для памяти диалогов (ниже — как добавить, если его нет).

### 1. Qdrant

Добавьте сервис в **тот же** `docker-compose.yml`, где запущен n8n. Если запустить Qdrant отдельным compose-файлом, он окажется в другой Docker-сети и n8n не найдёт его по имени `qdrant`.

В секцию `services`:

```yaml
  qdrant:
    image: qdrant/qdrant:v1.15.4      # закрепите актуальную версию
    restart: always
    environment:
      QDRANT__SERVICE__API_KEY: ${QDRANT_API_KEY}
      QDRANT__TELEMETRY_DISABLED: "true"
    volumes:
      - qdrant_storage:/qdrant/storage
    ports:
      - 127.0.0.1:6333:6333          # только localhost: наружу не публикуется
```

В корневую секцию `volumes`:

```yaml
  qdrant_storage:
```

В `.env` рядом с compose-файлом:

```bash
QDRANT_API_KEY=длинная_случайная_строка   # например: openssl rand -hex 32
```

Запуск без перезапуска остальных сервисов:

```bash
docker compose config --quiet && echo OK   # проверка синтаксиса
docker compose up -d qdrant
docker compose logs qdrant --tail 20       # ищите «Qdrant HTTP listening on 6333»
```

Проверка ключа:

```bash
KEY=$(grep '^QDRANT_API_KEY=' .env | cut -d= -f2)
curl -s localhost:6333/collections -H "api-key: $KEY"           # → {"result":{"collections":[]}...}
curl -s -o /dev/null -w "%{http_code}\n" localhost:6333/collections   # → 401
```

Dashboard Qdrant открывается через SSH-туннель: `ssh -L 6333:localhost:6333 user@server`, затем `http://localhost:6333/dashboard`.

### 2. Postgres для памяти диалогов

История диалогов хранится в отдельной базе `rag_memory` под отдельным пользователем `rag_bot`. Так бот не имеет доступа к базе n8n, а её данные — к истории бота.

**Если Postgres уже есть в compose** (типичная установка n8n в queue mode):

```bash
cd /path/to/n8n          # папка с docker-compose.yml
docker compose exec postgres sh -c 'psql -U "$POSTGRES_USER" -d postgres \
  -c "CREATE USER rag_bot WITH PASSWORD '"'"'надёжный_пароль'"'"';" \
  -c "CREATE DATABASE rag_memory OWNER rag_bot;"'
```

Должно вывести `CREATE ROLE` и `CREATE DATABASE`. Команда только создаёт новую базу: существующие базы, включая базу n8n, не затрагиваются.

Если база `rag_memory` уже создана ранее от имени основного пользователя, передайте её новому:

```bash
docker compose exec postgres sh -c 'psql -U "$POSTGRES_USER" -d postgres \
  -c "CREATE USER rag_bot WITH PASSWORD '"'"'надёжный_пароль'"'"';" \
  -c "ALTER DATABASE rag_memory OWNER TO rag_bot;"'
```

**Если Postgres нет** — добавьте сервис в тот же `docker-compose.yml`:

```yaml
  postgres:
    image: postgres:16
    restart: always
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: postgres
    volumes:
      - db_storage:/var/lib/postgresql/data
    ports:
      - 127.0.0.1:5432:5432           # только localhost
    healthcheck:
      test: ['CMD-SHELL', 'pg_isready -U ${POSTGRES_USER}']
      interval: 5s
      timeout: 5s
      retries: 10
```

В корневую секцию `volumes` добавьте `db_storage:`, в `.env` — `POSTGRES_USER` и `POSTGRES_PASSWORD`. Запуск и проверка:

```bash
docker compose up -d postgres
docker compose ps postgres            # статус healthy
```

Затем создайте пользователя и базу командой выше.

Проверка:

```bash
docker compose exec postgres sh -c 'psql -U "$POSTGRES_USER" -d postgres -c "\l"' | grep rag_memory
```

Таблицу `n8n_chat_histories` создавать не нужно: её создаст узел Postgres Chat Memory при первом вопросе боту.

### 3. Импорт воркфлоу

n8n → **Workflows → Import from File** → `workflows/rag_demo.json`.

### 4. Credentials

| Credential | Параметры | Узлы |
|---|---|---|
| Qdrant API | URL `http://qdrant:6333`, API key из `.env` | Очистить коллекцию, Insert Data to Store, Query Data Tool |
| Google Gemini (PaLM) API | API key из AI Studio | Google Gemini Chat Model, Embeddings Google Gemini |
| Postgres | Host `postgres`, Port `5432`, Database `rag_memory`, User `rag_bot`, SSL выключен | Postgres Chat Memory, Удалить историю чата |
| Telegram API | токен бота | Telegram Trigger, Send a text message, все узлы «Ответ: …» |

Если Qdrant доступен по другому адресу, исправьте URL и в узле «Очистить коллекцию» — он задан в самом узле.

Host `postgres` и `qdrant` — имена сервисов в Docker-сети, а не `localhost`: внутри контейнера n8n `localhost` указывает на сам контейнер.

### 5. Загрузка документов

Нажмите **Execute workflow** — откроется форма. Загрузите все три PDF из `docs/` **одной отправкой**: каждая загрузка удаляет коллекцию и создаёт её заново.

Проверка:

```bash
curl -s localhost:6333/collections/rag_docs -H "api-key: $KEY" | grep -o '"vectors":{[^}]*}'
curl -s localhost:6333/collections/rag_docs -H "api-key: $KEY" | grep -o '"points_count":[0-9]*'
```

Ожидается `"vectors":{"size":3072,"distance":"Cosine"` и около 10–12 точек.

### 6. Меню команд бота

В [@BotFather](https://t.me/BotFather): `/setcommands` → выберите бота → отправьте:

```
start - Начать
new - Очистить историю диалога
```

Команды появятся в меню рядом с полем ввода.

### 7. Запуск и проверка

Активируйте воркфлоу (переключатель **Active**) и напишите боту:

1. `/start` → приветствие;
2. «Сколько длится основной ежегодный отпуск?» → 28 дней, источник `reglament_otpuskov.pdf, стр. 1`;
3. сразу следом «А для стажёров?» → 2,33 дня за месяц (память работает);
4. перезапустите n8n (`docker compose restart n8n n8n-worker`) и снова спросите «А для стажёров?» → бот помнит контекст (память в Postgres);
5. `/new`, затем «А для стажёров?» → бот не понимает, о чём речь (история очищена);
6. «Положен ли сотрудникам полис ДМС?» → «Не знаю…» без источника;
7. отправьте стикер → «понимаю только текст».

Посмотреть сохранённую историю:

```bash
docker compose exec postgres sh -c 'psql -U "$POSTGRES_USER" -d rag_memory \
  -c "SELECT session_id, message->>'"'"'type'"'"' AS role, left(message->'"'"'data'"'"'->>'"'"'content'"'"', 60) FROM n8n_chat_histories ORDER BY id DESC LIMIT 10;"'
```

### Решение проблем

| Симптом | Причина | Что сделать |
|---|---|---|
| `Not existing vector name error` в Insert | коллекция создана не узлом (через Dashboard или с опцией Collection Config) — без векторов или с именованным вектором | удалить коллекцию (`curl -X DELETE localhost:6333/collections/rag_docs -H "api-key: $KEY"`), убрать Collection Config, загрузить заново |
| Credential Qdrant не сохраняется | n8n не видит Qdrant | Qdrant должен быть в том же compose; URL `http://qdrant:6333`, не `localhost` |
| В выходе «Очистить коллекцию» 401 | ключ не передаётся | проверить credential; при необходимости Authentication → Header Auth с именем `api-key` |
| Бот молчит | у бота один webhook: открыт тестовый режим или воркфлоу не активен | остановить «Execute workflow», включить Active |
| Ошибка «can't parse entities» при отправке | Telegram разбирает `_` в именах файлов как Markdown | Parse Mode = HTML (уже задан) |
| Credential Postgres не сохраняется | неверный host или пароль | host `postgres` (не `localhost`), база `rag_memory`, пользователь `rag_bot` |
| `permission denied for schema public` | база создана другим пользователем | `ALTER DATABASE rag_memory OWNER TO rag_bot;` |
| `/new` не отвечает | нет вывода у узла удаления | у «Удалить историю чата» должны быть включены Always Output Data и On Error: Continue |
| История не очищается после `/new` | `session_id` сравнивается как число | в запросе `chat.id` передаётся строкой (`String(...)`) — не меняйте это |

### Ограничения

- История диалогов в Postgres растёт без ограничений: в таблице нет даты сообщения, поэтому автоудаления по сроку нет. Пользователь очищает свою историю командой `/new`.
- В истории хранятся вопросы пользователей вместе с их Telegram ID. Для вымышленных документов это неважно, но при работе с реальными пользователями это персональные данные, и для них нужен срок хранения.
- Повторная загрузка формы полностью пересоздаёт коллекцию — инкрементального обновления документов нет.

## Оценка качества

Набор из 15 вопросов (`eval/rag_eval_questions.xlsx`) с эталонными ответами и ожидаемыми источниками:

| Категория | Кол-во | Что проверяет |
|---|---|---|
| Прямой факт | 5 | базовый поиск |
| Перефразирование | 3 | качество эмбеддингов |
| Несколько фрагментов | 3 | объединение источников |
| Вне документов (вкл. ловушку) | 3 | отказ без выдумки |
| Память | 1 | уточняющий вопрос в той же сессии |

Каждый ответ оценивался по трём метрикам (0/1): `correct` — ключевые факты верны; `source_ok` — верные файл и страница; `no_hallucination` — нет фактов вне документов. Максимум — 45. Каждый вопрос задавался в новой сессии чата. Подбор параметров (прогоны 1–8) шёл на Simple Vector Store во встроенном чате n8n; итоговая конфигурация затем перенесена на Qdrant, Postgres Chat Memory и Telegram.

### Результаты

| # | Модель | Chunk / Overlap | Limit | Промпт | correct | source_ok | no_halluc | Σ /45 |
|--:|---|---|--:|---|--:|--:|--:|--:|
| 1 | flash-lite | 800 / 150 | 4 | v1 | 14 | 13 | 15 | 42 |
| 2 | flash-lite | 400 / 80 | 4 | v1 | 14 | 11 | 15 | 40 |
| 3 | flash-lite | 200 / 40 | 4 | v1 | 13 | 5 | 15 | 33 |
| 4 | flash-lite | 800 / 150 | 2 | v1 | 14 | 13 | 15 | 42 |
| 5 | flash-lite | 800 / 150 | 8 | v1 | 14 | 14 | 15 | 43 |
| 6 | 3.6-flash | 800 / 150 | 4 | v1 | 14 | 13 | 15 | 42 |
| 7 | flash-lite | 800 / 150 | 4 | **v2** | 14 | 14 | 15 | **43** |
| 8 | flash-lite | 800 / 150 | 4 | **v2** | 14 | 14 | 15 | **43** (повтор) |

**Итоговая конфигурация — прогон 7/8**: результат повторился ответ в ответ.

## Выводы

**Размер фрагментов — главный параметр.** Уменьшение с 800 до 200 символов снизило результат с 42 до 33. Мелкие фрагменты хуже находятся по перефразированным вопросам (№8 провален), а модель чаще теряет или путает номер страницы: ошибок в источниках стало 10 вместо 2.

**top-k на маленькой базе почти не влияет.** В базе около 10–12 фрагментов, и Limit 2, 4 и 8 дали одинаковое качество поиска. Limit 8 здесь фактически означает «отдать почти всю базу», поэтому выбран 4.

**Более сильная модель не помогла.** `gemini-3.6-flash` дороже примерно в 5 раз и дала тот же результат, но со своими ошибками: исказила имя файла и добавила Markdown.

**Промпт исправил ошибки поведения.** В версии v2 добавлены правила: отказ без источника, имя файла копируется из метаданных дословно, текст без Markdown, поиск по каждой части вопроса. Эти ошибки исчезли в двух прогонах подряд.

**Галлюцинаций не было ни в одном прогоне** — включая вопрос-ловушку (№14), где рядом в документе есть похожий, но не тот факт.

## Типичные ошибки

| Ошибка | Где | Причина | Чем исправлено |
|---|---|---|---|
| Не найден фрагмент при перефразировании | №8, прогон 3 | слишком мелкие фрагменты | фрагменты 800 |
| Не указана или неверна страница | прогоны 2–3 | мелкие фрагменты | фрагменты 800 |
| Источник приложен к отказу | №14, прогоны 1, 3, 4 | модель | правило в промпте v2 |
| Искажено имя файла | №11, прогон 6 | модель перепечатывает имя | правило в промпте v2 |
| Markdown в ответе | прогон 6 | модель | правило в промпте v2 |
| Нет второго документа | №9, все прогоны | агент делает один поиск | **не исправлено** |

### Известное ограничение: многошаговый вопрос №9

На вопрос «Хочу месяц поработать из Испании — что нужно?» бот верно отвечает по политике удалённой работы (лимит 30 дней, согласование, только через VPN), но не ищет в ИТ FAQ, как получить VPN. Ошибка не исчезла ни при Limit 8, ни на более сильной модели, ни с общим правилом о дополнительном поиске. Это вопрос повышенной сложности: ответ требует второго поиска по теме, которую пользователь не назвал. Специфичные для теста слова в промпт сознательно не добавлялись, чтобы не подгонять бота под тест.

## Структура репозитория

```
docs/          тестовые PDF (вымышленная компания)
eval/          набор вопросов и результаты всех прогонов
workflows/     воркфлоу n8n (без credentials и webhook ID)
```

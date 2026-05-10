# Telegram — Проверка подписок и папка "База групп"

Два независимых GitHub Actions workflow для работы с Telegram-аккаунтом через Google Sheets.

---

## Структура репозитория

```
├── .github/workflows/
│   ├── check_subscriptions.yml   # Workflow 1 — проверка подписок
│   └── add_to_folder.yml         # Workflow 2 — добавление в папку
├── scripts/
│   ├── check_subscriptions.py    # Скрипт 1
│   ├── add_to_folder.py          # Скрипт 2
│   └── generate_session.py       # Запустить локально 1 раз
├── requirements.txt
└── README.md
```

---

## Формат Google Таблицы

Лист называется **Группы**

| A (Ссылка) | B (Статус) |
|---|---|
| https://t.me/batumi_group | ✅ подписан |
| https://t.me/batumiz | ❌ не подписан |

- Ссылки — в колонке **A**, начиная со строки **2** (строка 1 — заголовок)
- Скрипт 1 записывает результат в колонку **B**

---

## Workflow 1 — Проверка подписок

**Что делает:**
1. Читает ссылки из колонки A
2. Загружает все текущие подписки аккаунта одним запросом (`iter_dialogs`)
3. Для каждой ссылки резолвит ID и сравнивает со списком подписок
4. Записывает в колонку B: `✅ подписан` / `❌ не подписан` / `⚠️ недоступен`

**Риск FloodWait:** минимальный — между запросами пауза 1 сек, join не выполняется

---

## Workflow 2 — Добавление в папку "База групп"

**Что делает:**
1. Читает ссылки из колонки A
2. Загружает все текущие подписки аккаунта
3. Пропускает каналы, на которые аккаунт **не подписан**
4. Находит или создаёт папку **"База групп"**
5. Добавляет все подписанные каналы в эту папку (без дублей)

**Риск FloodWait:** низкий — join не выполняется, только чтение + одно сохранение папки

---

## Настройка (один раз)

### 1. Получить Telegram API ключи

Зайти на [https://my.telegram.org](https://my.telegram.org) → API development tools → создать приложение.  
Скопировать `api_id` и `api_hash`.

### 2. Сгенерировать сессию (локально)

```bash
pip install telethon
python scripts/generate_session.py
```

Ввести номер телефона и код из SMS. Скопировать напечатанную строку сессии.

### 3. Настроить Google Sheets доступ

1. [Google Cloud Console](https://console.cloud.google.com/) → новый проект
2. Включить **Google Sheets API**
3. Создать **Service Account** → скачать JSON-ключ
4. Открыть таблицу → **Настройки доступа** → добавить email сервис-аккаунта (роль: Редактор)
5. Скопировать **ID таблицы** из URL: `docs.google.com/spreadsheets/d/`**`ВОТ_ЭТО`**`/edit`

### 4. Добавить секреты в GitHub

**Settings → Secrets and variables → Actions → New repository secret**

| Название секрета | Значение |
|---|---|
| `TELEGRAM_API_ID` | Число из my.telegram.org |
| `TELEGRAM_API_HASH` | Строка из my.telegram.org |
| `TELEGRAM_SESSION_STRING` | Вывод generate_session.py |
| `GOOGLE_SPREADSHEET_ID` | ID таблицы из URL |
| `GOOGLE_SERVICE_ACCOUNT_JSON` | Полное содержимое скачанного JSON-файла |

---

## Порядок использования

```
1. Запустить Workflow 1 → таблица заполнится статусами
2. Вручную подписаться в Telegram на нужные каналы (❌ не подписан)
3. Запустить Workflow 2 → все подписанные каналы попадут в папку "База групп"
```

Workflow 2 можно запускать повторно — дубли не создаются.

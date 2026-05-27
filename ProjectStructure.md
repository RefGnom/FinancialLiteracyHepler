# ProjectStructure.md – Архитектура FinancialDiscipline

## Общая структура решения (.sln)
- **Finance.Domain** – чистые сущности, бизнес-правила, интерфейсы репозиториев, перечисления, исключения. Не имеет внешних зависимостей (кроме .NET BCL).
- **Finance.Application** – сценарии (use cases): импорт, редактирование, анализ, проверка дубликатов, управление связями, валидация. Зависит от Domain и абстракций инфраструктуры (интерфейсы репозиториев и сервисов, объявленные в Domain или отдельном проекте Abstractions).  
  _Примечание: Интерфейсы для инфраструктуры могут лежать в Domain, чтобы Application не тянул лишние зависимости._
- **Finance.Infrastructure** – реализации хранилищ (SQLite репозитории), парсеров файлов (CSV, XLSX, PDF), логгирования (Serilog), OCR-заглушка, миграции (FluentMigrator). Зависит от Application (или напрямую от Domain).
- **Finance.Cli** – консольное приложение, DI (Microsoft.Extensions.DependencyInjection), команды (System.CommandLine), интерактивные подсказки (Spectre.Console), конфигурация. Зависит от Application и Infrastructure.
- **Finance.TelegramBot** – отдельное консольное приложение, обработчик сообщений Telegram (Telegram.Bot), сохраняет фото в локальную папку. Имеет свой конфиг (bot-settings.json). Не зависит от других проектов (только от базовой библиотеки для работы с файловой системой).

Все проекты используют .NET 10.

## Размещение данных (файлов) на диске
- Базовый каталог для данных и конфигурации: `%APPDATA%\FinancialDiscipline\`.
- Внутри:
  - `settings.json` – общие настройки CLI (путь к БД, настройки парсинга, правила дубликатов, язык интерфейса, путь к логам и т.п.)
  - `finance.db` – файл SQLite (если используется SQLite; в будущем может быть заменён на другую БД, но путь к подключению задаётся в settings.json)
  - `logs\` – папка с файлами логов (Serilog, ротация 100 МБ, 10 файлов)
  - `backups\` – папка с бэкапами (дата-время)
  - `incoming_receipts\` – папка, куда TelegramBot сохраняет фотографии чеков (путь настраивается в bot-settings.json)
  - `bot-settings.json` - настройки Для TelegramBot 

## Ключевые архитектурные решения
- **Чистая архитектура (луковичная)**. Ядро – Domain, не знает о внешних слоях.
- **Агрегация данных для аналитики производится в памяти** (LINQ to Objects). Репозиторий загружает все необходимые сущности (с фильтрацией по датам/категориям на уровне БД для сокращения объёма) и передаёт в сервис аналитики, который группирует/агрегирует в памяти.
- **Транзакционность**: все операции, изменяющие несколько сущностей, оборачиваются в транзакцию БД (через репозитории).
- **Миграции** – FluentMigrator. Поддерживаются SQLite и (в будущем) PostgreSQL. Миграции размещаются в Infrastructure.
- **Интернационализация (i18n)** – только для пользовательского интерфейса:
  - Команды CLI на английском (неизменны).
  - Вывод CLI (текст, подсказки, сообщения) – русский (по умолчанию) или английский (переключается в settings.json).
  - Аналитика (названия колонок, итоги) – русский/английский.
  - Логи – всегда английский (для единообразия).
  - Реализация: ресурсные файлы `.resx` в проекте Cli + вспомогательный класс `Localizer`. Пока поддерживаем два языка.

## Модули (логическая группировка кода внутри проектов)

| Модуль          | Расположение (проект) | Назначение |
|----------------|----------------------|-------------|
| **Domain Models** | Finance.Domain | Сущности (Transaction, Position, Tag, Scope, Compensation, InternalTransfer), правила, интерфейсы репозиториев |
| **Storage**       | Finance.Infrastructure | Реализации репозиториев (SQLite), миграции, бэкапы (JSON Lines) |
| **Parsers**       | Finance.Infrastructure | Парсеры CSV/XLSX/PDF + фабрика |
| **Duplicate Checker** | Finance.Application | Логика поиска дубликатов по настраиваемым полям |
| **Analytics**     | Finance.Application | Фильтрация, группировка, агрегация (в памяти), экспорт в Excel/HTML/консоль |
| **Linking**       | Finance.Application | Управление скоупами, компенсациями, внутренними переводами (авто/ручное связывание) |
| **Receipt Ocr**   | Finance.Infrastructure | Интерфейс IReceiptOcrService и заглушка (Mock) |
| **CLI**           | Finance.Cli | Команды, интерактив, пагинация, DI, настройки локализации |
| **Telegram Bot**  | Finance.TelegramBot | Отдельное приложение, приём фото и сохранение в папку |
| **Backup/Restore**| Finance.Application + Infrastructure | Создание и восстановление бэкапов в формате JSON Lines |

## Зависимости между модулями (в коде)
- **Domain** не зависит ни от чего.
- **Application** зависит от Domain.
- **Infrastructure** зависит от Domain и от Application (реализует интерфейсы use cases/репозиториев). В идеале Infrastructure не должен знать о Application напрямую – через интерфейсы, объявленные в Domain. Но для удобства можно создать проект Finance.Application.Abstractions. Упростим: интерфейсы репозиториев и сервисов лежат в Domain.
- **Cli** зависит от Application и Infrastructure (через DI).
- **TelegramBot** не зависит от других проектов, только от стандартных библиотек.

## Конфигурация (settings.json)
Пример содержимого:
```json
{
  "Database": {
    "Provider": "SQLite",
    "ConnectionString": "Data Source=%APPDATA%/FinancialDiscipline/finance.db"
  },
  "Parsers": {
    "CSV": { "Delimiter": ";", "Encoding": "utf-8", "HasHeader": true },
    "XLSX": { "SheetIndex": 0 },
    "PDF": { "ExtractTablesOnly": true }
  },
  "DuplicateMatchingFields": [ "Date", "Amount", "Description", "AccountId" ],
  "Language": "ru", // или "en"
  "Logging": {
    "Path": "%APPDATA%/FinancialDiscipline/logs/log-.txt",
    "FileSizeLimitBytes": 104857600, // 100 MB
    "RetainedFileCount": 10
  },
  "Backup": {
    "Format": "JsonLines",
    "ChunkSize": 10000 // количество записей на файл
  }
}
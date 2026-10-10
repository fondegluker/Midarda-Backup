# Дизайн проекта Midarda-Backup

## 1. Архитектура

### 1.1. Компоненты

Установщик (installer.exe):
  - UAC / права администратора
  - Управление пользователем midarda-backup
  - Копирование файлов, EFS-шифрование
  - Регистрация службы в SCM
  - Мастер конфигурации (SMB/FTP/MSSQL/подсеть)
  - ACL, запуск, проверка

Служба (midarda-backup.exe):
  - ConfigLoader (TOML + EFS + hot-reload)
  - Watcher (ReadDirectoryChangesW, 3 уровня)
  - Stabilizer (закрытие дескриптора + пауза)
  - IntegrityChecker (RESTORE VERIFYONLY)
  - Copier (SMB, FTP/FTPS)
  - Hasher (.sha256)
  - Rotator (локально + удалённо)
  - Watchdog (heartbeat, само-мониторинг)
  - Retry (exponential backoff)
  - Notifier (SMTP)
  - Logger (5 МБ x 5, русский)
  - HealthServer (/health, /metrics, :91873)

### 1.2. Схема папок

%ProgramData%\Midarda-Backup\
  installer.exe          (опционально, для переустановки)
  midarda-backup.exe
  config.toml            (EFS-зашифрован)
  logs\
    midarda-backup.log
    midarda-backup-2024-01-01.log
    ...
  docs\
    README.md
    admin-guide.md

### 1.3. Поток данных

[Папка-источник] -> Watcher -> Stabilizer -> IntegrityChecker (MSSQL)
   -> Hasher -> Copier (SMB/FTP) -> Verify (SMB) -> Rotator -> Лог/Email

---

## 2. Дизайн установщика

### 2.1. Пошаговый сценарий (11 этапов)
1. Проверка административных привилегий (UAC).
2. Создание/обновление пользователя midarda-backup, запрос пароля.
3. Назначение права SeServiceLogonRight.
4. Копирование файлов проекта в %ProgramData%\Midarda-Backup.
5. EFS-шифрование конфига от имени midarda-backup.
6. Запись пароля в конфиг (до шифрования).
7. Регистрация службы (Automatic Delayed, от midarda-backup).
8. Мастер конфигурации.
9. Установка владельца и ACL на папку.
10. Запуск службы.
11. Проверка работоспособности (health-check).

### 2.2. Точки отказа

| Этап | Ошибка | Действие |
|------|--------|----------|
| 1 | UAC отклонён | Прерывание |
| 2 | Не удалось создать пользователя | Прерывание, лог |
| 5 | EFS недоступен | Предупреждение, прерывание |
| 7 | Не удалось зарегистрировать службу | Прерывание, лог |
| 8 | Нет доступа к SMB/FTP | Повтор ввода |
| 8 | Не активирован ни один метод | Повтор ввода |
| 10 | Служба не стартовала | Предупреждение, ручное исправление |
| 11 | Health-check не прошёл | Предупреждение, ручное исправление |

### 2.3. Режимы
- install — полная установка.
- upgrade — запрос: сохранять ли конфиг.
- uninstall — остановка/удаление службы, удаление пользователя, файлов,
  конфига (с подтверждением).

---

## 3. Дизайн службы

### 3.1. Основной цикл

loop:
  event = Watcher.NextEvent()          // ReadDirectoryChangesW
  if event.Type == FileClosed:
    Stabilizer.Wait(event.Path)        // пауза из конфига
    if not MatchesMask(event.Path): continue
    if Config.MSSQL.Enabled:
      if not IntegrityChecker.Verify(event.Path):
        Logger.Error(...); Notifier.Email(...); continue
    Hasher.Compute(event.Path)
    for res in [SMB, FTP]:
      if res.Enabled:
        Copier.Copy(event.Path, res)   // с retry + backoff
        if res == SMB and Config.Verify.SMB:
          VerifyChecksum(event.Path, res)
    Rotator.Rotate(event.Path)         // локально + удалённо
    Logger.Info(...)

### 3.2. Watchdog (само-мониторинг)
- Отдельный thread обновляет last_heartbeat каждые N секунд.
- Основной thread проверяет: если now - last_heartbeat > timeout —
  служба считается зависшей.
- Действие: запись в лог, email-уведомление, перезапуск процесса службы
  (через SCM или self-restart).

### 3.3. Retry с exponential backoff
- Библиотека: cenkalti/backoff/v7.
- Параметры из конфига: max_attempts (по умолчанию 1), initial_interval,
  max_interval, multiplier.
- Комбинируется с таймаутом операции (С-29).

### 3.4. Горячая перезагрузка конфига
- fsnotify на файл config.toml.
- При изменении — перечитать, расшифровать, валидировать.
- При ошибке — откат к предыдущей валидной версии, лог WARN.

---

## 4. Конфигурация

### 4.1. Формат: TOML

### 4.2. Полная схема

# === Общие ===
[general]
watch_path = "D:\\BackUp"
watch_depth = 3                  # 3 уровня
file_mask = ["*.bak", "*.trn"]
stabilization_delay_sec = 5
log_level = "INFO"
log_max_size_mb = 5
log_max_files = 5

# === Служебная учётная запись ===
[service_account]
username = "midarda-backup"
password = "..."                 # EFS-шифрован весь файл

# === SMB ===
[smb]
enabled = true
path = "\\\\FS-01\\BackUp"
verify_after_copy = true
timeout_sec = 300

# === FTP ===
[ftp]
enabled = true
url = "ftp://fs-01:21"
tls = true
timeout_sec = 300

# === MSSQL ===
[mssql_integrity_check]
enabled = true
instance = ".\\SQLEXPRESS"
auth = "windows"                 # или "sql"
username = ""
password = ""

# === Ротация ===
[rotation]
local_keep = 3
remote_keep = 5

# === Retry ===
[retry]
max_attempts = 1
initial_interval_sec = 5
max_interval_sec = 300
multiplier = 2.0

# === Health API ===
[health]
port = 91873
allowed_subnet = "192.168.1.0/24"

# === Email ===
[email]
enabled = true
smtp_host = "smtp.example.com"
smtp_port = 587
smtp_tls = true
smtp_username = ""
smtp_password = ""
from = "midarda-backup@example.com"
to = ["admin@example.com"]

### 4.3. EFS-шифрование
- Весь файл config.toml шифруется EFS от имени midarda-backup.
- Расшифровка — прозрачно для процесса службы (тот же пользователь).

### 4.4. Документация по смене пароля
Внимание! Смена пароля midarda-backup после установки:
- Смена через net user со старым паролем — безопасна.
- Сброс пароля администратором — делает профиль недоступным,
  EFS-сертификат теряется, конфиг нерасшифровываем.
- Пароль в SCM нужно обновлять вручную
  (sc config Midarda-Backup password= ...).
- Рекомендация: при необходимости смены — переустанавливать службу.

---

## 5. Health API

### 5.1. Endpoints
- GET /health — расширенный JSON.
- GET /metrics — Prometheus-формат + совместимость с Zabbix.

### 5.2. Формат /health

{
  "status": "healthy",
  "timestamp": "2024-01-01T12:00:00Z",
  "last_successful_transaction": "2024-01-01T11:55:00Z",
  "resources": {
    "smb": {"status": "ok", "writable": true},
    "ftp": {"status": "ok", "writable": true},
    "mssql": {"status": "ok", "reachable": true}
  },
  "copies": {
    "local": {"db": 3, "db_test": 2},
    "remote_smb": {"db": 5, "db_test": 5},
    "remote_ftp": {"db": 5, "db_test": 5}
  },
  "watchdog": {"last_heartbeat": "2024-01-01T12:00:00Z"}
}

### 5.3. Ограничения по подсети
- Проверка RemoteAddr входящего запроса.
- Если IP не в allowed_subnet — 403 Forbidden.

---

## 6. Безопасность

- EFS + предупреждения (см. 4.4).
- ACL: %ProgramData%\Midarda-Backup — только midarda-backup и SYSTEM.
- Служебная учётная запись — midarda-backup с правом SeServiceLogonRight.
- Health API — только подсеть из конфига.

---

## 7. Стратегия тестирования

- Unit-тесты: конфиг, hasher, rotator, backoff, парсер событий.
- Интеграционные: локальный FTP-сервер, SMB-шара (или эмулятор),
  MSSQL Express с тестовым .bak.
- E2E: полный цикл установки в тестовой VM.

---

## 8. Развёртывание и эксплуатация

- README.md — быстрый старт.
- admin-guide.md — установка, обновление, удаление, смена пароля, типовые ошибки.
- Диаграмма развёртывания.
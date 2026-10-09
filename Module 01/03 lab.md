# Лабораторная работа: Управление службами и юнитами systemd в Debian 12

**Тема:** Основы systemd: юниты, управление службами, таймеры, journald, targets, создание собственных юнитов.

**Цель работы:**
- Изучить архитектуру и основные понятия systemd.
- Освоить управление службами через `systemctl`.
- Научиться создавать собственные `.service`, `.timer`, `.target` юниты.
- Освоить работу с журналом `journald` и `journalctl`.
- Разобраться с целями загрузки (targets) и зависимостями между юнитами.

**Оборудование и ПО:**
- Виртуальная машина с Debian 12 (Bookworm).
- Пользователь `root` (пароль `111`), пользователь `user` (пароль `111`).

**Продолжительность:** 2 академических часа (120 минут).

---

## Теоретическая справка (5 мин)

**systemd** — система инициализации и менеджер служб в большинстве современных Linux-дистрибутивов. Заменяет классический `SysVinit`.

**Ключевые понятия:**
- **Unit (юнит)** — базовая единица управления systemd. Описывается в файле с расширением, зависящим от типа: `.service`, `.timer`, `.target`, `.mount`, `.socket`, `.path`, `.device`.
- **Секции юнита**: `[Unit]` (метаданные и зависимости), `[Service]` (для сервисов), `[Install]` (в какой target включать).
- **Target** — группа юнитов, аналог runlevel.
- **journald** — демон журналирования systemd, собирает логи всех служб.

**Расположение юнитов (в порядке приоритета):**
| Каталог | Назначение |
|---|---|
| `/usr/lib/systemd/system/` | Юниты из пакетов |
| `/run/systemd/system/` | Временные юниты (runtime) |
| `/etc/systemd/system/` | Локальные администраторские юниты (высший приоритет) |

---

## Этап 1. Подготовка (5 мин)

1. Войдите как `root`:
   ```bash
   su -
   # пароль: 111
   ```

2. Проверьте наличие пользователя `user`:
   ```bash
   id user || (useradd -m -s /bin/bash user && echo 'user:111' | chpasswd)
   ```

3. Проверьте версию systemd и что она активна как PID 1:
   ```bash
   systemctl --version
   ps -p 1 -o comm=
   ```
   Ожидается: `systemd`.

4. Установите полезные утилиты:
   ```bash
   apt update
   apt install -y systemd-timesyncd tree
   ```

---

## Этап 2. Управление службами (20 мин)

### 2.1. Основные команды `systemctl`

1. Запуск, остановка, перезапуск:
   ```bash
   systemctl start ssh
   systemctl status ssh
   systemctl stop ssh
   systemctl restart ssh
   ```

2. Автозапуск при загрузке:
   ```bash
   systemctl enable ssh
   systemctl is-enabled ssh
   systemctl disable ssh
   systemctl is-enabled ssh
   ```

3. Одновременный запуск и включение в автозагрузку:
   ```bash
   systemctl enable --now ssh
   ```

4. Перечитать конфигурацию без перезапуска:
   ```bash
   systemctl reload ssh
   systemctl reload-or-restart ssh
   ```

5. Маскировка (полный запрет запуска, включая зависимости):
   ```bash
   systemctl mask nginx 2>/dev/null || systemctl mask cups
   systemctl status cups
   systemctl unmask cups
   ```

### 2.2. Просмотр списков юнитов

1. Все активные юниты:
   ```bash
   systemctl list-units
   ```

2. Только службы:
   ```bash
   systemctl list-units --type=service
   ```

3. Только запущенные:
   ```bash
   systemctl list-units --type=service --state=running
   ```

4. Только упавшие:
   ```bash
   systemctl list-units --type=service --state=failed
   systemctl --failed
   ```

5. Все юниты (включая неактивные):
   ```bash
   systemctl list-units --all
   ```

6. Установленные юнит-файлы:
   ```bash
   systemctl list-unit-files --type=service | head -30
   ```

### 2.3. Просмотр свойств юнита

1. Все свойства службы `ssh`:
   ```bash
   systemctl show ssh
   ```

2. Конкретные свойства:
   ```bash
   systemctl show ssh -p MainPID,ExecStart,Restart,After
   ```

3. Показать содержимое юнит-файла:
   ```bash
   systemctl cat ssh
   ```

4. Определить, откуда загружен юнит:
   ```bash
   systemctl status ssh | grep Loaded
   ```

---

## Этап 3. Создание собственного `.service` (25 мин)

### 3.1. Простой сервис

1. Создайте скрипт:
   ```bash
   nano /usr/local/bin/hello-service.sh
   ```
   Содержимое:
   ```bash
   #!/bin/bash
   while true; do
       echo "hello-service $(date)" >> /var/log/hello-service.log
       sleep 10
   done
   ```
   ```bash
   chmod +x /usr/local/bin/hello-service.sh
   ```

2. Создайте юнит-файл `/etc/systemd/system/hello.service`:
   ```bash
   nano /etc/systemd/system/hello.service
   ```
   ```ini
   [Unit]
   Description=Hello Demo Service
   After=network.target

   [Service]
   Type=simple
   ExecStart=/usr/local/bin/hello-service.sh
   Restart=on-failure
   RestartSec=5

   [Install]
   WantedBy=multi-user.target
   ```

3. Перезагрузите конфигурацию и запустите:
   ```bash
   systemctl daemon-reload
   systemctl enable --now hello.service
   systemctl status hello.service
   ```

4. Проверьте работу:
   ```bash
   journalctl -u hello.service -f
   # Ctrl+C
   tail -f /var/log/hello-service.log
   # Ctrl+C
   ```

5. Остановите сервис:
   ```bash
   systemctl stop hello.service
   ```

### 3.2. Типы служб

**`Type=simple`** (по умолчанию) — основной процесс не форкается.
**`Type=forking`** — демон форкается и родитель завершается.
**`Type=oneshot`** — процесс выполняется один раз и завершается; systemd ждёт завершения.
**`Type=notify`** — процесс уведомляет systemd через `sd_notify`.
**`Type=idle`** — как `simple`, но запускается после всех остальных задач.

### 3.3. Служба типа `oneshot`

1. Создайте юнит `/etc/systemd/system/hello-oneshot.service`:
   ```bash
   nano /etc/systemd/system/hello-oneshot.service
   ```
   ```ini
   [Unit]
   Description=One-shot Demo

   [Service]
   Type=oneshot
   ExecStart=/bin/bash -c 'echo "Выполнено $(date)" >> /var/log/oneshot.log'

   [Install]
   WantedBy=multi-user.target
   ```

2. Запустите:
   ```bash
   systemctl daemon-reload
   systemctl start hello-oneshot.service
   systemctl status hello-oneshot.service
   cat /var/log/oneshot.log
   ```

   После завершения состояние — `inactive (dead)`, но код выхода `0` (успех).

### 3.4. Дополнительные директивы

- `User=`, `Group=` — от имени какого пользователя запускать.
- `WorkingDirectory=` — рабочий каталог.
- `Environment=` — переменные окружения.
- `EnvironmentFile=` — файл с переменными.
- `ExecStartPre=`, `ExecStartPost=`, `ExecStop=` — дополнительные действия.
- `Restart=on-failure | always | on-abnormal | no`.
- `RestartSec=` — задержка перед перезапуском.
- `StandardOutput=`, `StandardError=` — куда перенаправлять вывод (`journal`, `file:`, `null`).

**Пример сервиса от имени пользователя `user`:**
```ini
[Service]
User=user
Group=user
WorkingDirectory=/home/user
Environment="MYVAR=42"
ExecStart=/usr/local/bin/hello-service.sh
Restart=always
RestartSec=3
```

---

## Этап 4. Создание собственного `.timer` (20 мин)

Таймеры systemd заменяют `cron`.

### 4.1. Создание пары сервис + таймер

1. Сервис `/etc/systemd/system/date-logger.service`:
   ```bash
   nano /etc/systemd/system/date-logger.service
   ```
   ```ini
   [Unit]
   Description=Log current date

   [Service]
   Type=oneshot
   ExecStart=/bin/bash -c 'date >> /var/log/date-logger.log'
   ```

2. Таймер `/etc/systemd/system/date-logger.timer`:
   ```bash
   nano /etc/systemd/system/date-logger.timer
   ```
   ```ini
   [Unit]
   Description=Run date-logger every 1 minute

   [Timer]
   OnCalendar=*:0/1
   Persistent=true
   Unit=date-logger.service

   [Install]
   WantedBy=timers.target
   ```

3. Активируйте таймер (не сервис!):
   ```bash
   systemctl daemon-reload
   systemctl enable --now date-logger.timer
   ```

4. Проверьте список таймеров:
   ```bash
   systemctl list-timers --all
   ```

5. Через 1–2 минуты:
   ```bash
   cat /var/log/date-logger.log
   journalctl -u date-logger.service
   ```

### 4.2. Различные варианты расписания

| Директива | Значение |
|---|---|
| `OnCalendar=*-*-* 03:00:00` | Ежедневно в 3:00 |
| `OnCalendar=Mon *-*-* 09:00` | Каждый понедельник в 9:00 |
| `OnCalendar=*:0/15` | Каждые 15 минут |
| `OnBootSec=5min` | Через 5 минут после загрузки |
| `OnUnitActiveSec=1h` | Через 1 час после активации юнита |
| `OnActiveSec=30s` | Через 30 секунд после активации таймера |
| `Persistent=true` | Догнать пропущенные запуски после выключения |

Проверка синтаксиса расписания:
```bash
systemd-analyze calendar "*:0/15"
```

### 4.3. Остановка таймера

```bash
systemctl stop date-logger.timer
systemctl disable date-logger.timer
```

---

## Этап 5. Журнал systemd: journalctl (20 мин)

### 5.1. Основы

1. Все сообщения журнала:
   ```bash
   journalctl
   ```

2. Только за текущую загрузку:
   ```bash
   journalctl -b
   ```

3. Сообщения предыдущей загрузки:
   ```bash
   journalctl -b -1
   ```

4. Просмотр в реальном времени:
   ```bash
   journalctl -f
   # Ctrl+C
   ```

5. Сообщения конкретной службы:
   ```bash
   journalctl -u ssh
   journalctl -u hello.service
   ```

### 5.2. Фильтры по времени

1. С момента загрузки:
   ```bash
   journalctl --since "10 minutes ago"
   ```

2. За конкретный период:
   ```bash
   journalctl --since "2025-01-01 10:00" --until "2025-01-01 11:00"
   ```

3. По сегодняшнему дню:
   ```bash
   journalctl --since today
   ```

### 5.3. Фильтры по приоритету

Уровни важности (от 0 до 7):
0 — emerg, 1 — alert, 2 — crit, 3 — err, 4 — warning, 5 — notice, 6 — info, 7 — debug.

1. Только ошибки и критичнее:
   ```bash
   journalctl -p err
   ```

2. Ошибки и предупреждения:
   ```bash
   journalctl -p warning
   ```

3. С момента загрузки только ошибки:
   ```bash
   journalctl -b -p err
   ```

### 5.4. Форматы вывода

1. `json`:
   ```bash
   journalctl -u ssh -o json | head -20
   ```

2. `short-precise` (с микросекундами):
   ```bash
   journalctl -u ssh -o short-precise | head
   ```

3. Только сообщения без метаданных:
   ```bash
   journalctl -u ssh -o cat | head
   ```

### 5.5. Фильтры по полям

1. По PID:
   ```bash
   journalctl _PID=1
   ```

2. По исполняемому файлу:
   ```bash
   journalctl /usr/sbin/sshd
   ```

3. По пользователю:
   ```bash
   journalctl _UID=0 --since "10 min ago"
   ```

### 5.6. Обслуживание журнала

1. Размер журнала на диске:
   ```bash
   journalctl --disk-usage
   ```

2. Ограничить общий размер:
   ```bash
   nano /etc/systemd/journald.conf
   ```
   Раскомментировать и установить:
   ```ini
   SystemMaxUse=500M
   ```
   ```bash
   systemctl restart systemd-journald
   ```

3. Очистить старые записи (оставить последние 100 МБ):
   ```bash
   journalctl --vacuum-size=100M
   ```

4. Очистить записи старше 2 недель:
   ```bash
   journalctl --vacuum-time=2weeks
   ```

5. Проверка целостности:
   ```bash
   journalctl --verify
   ```

---

## Этап 6. Targets и зависимости (15 мин)

### 6.1. Основные targets

| Target | Аналог runlevel | Назначение |
|---|---|---|
| `poweroff.target` | 0 | Выключение |
| `rescue.target` | 1 | Однопользовательский режим |
| `multi-user.target` | 3 | Многопользовательский без графики |
| `graphical.target` | 5 | С графической оболочкой |
| `reboot.target` | 6 | Перезагрузка |

1. Текущий target:
   ```bash
   systemctl get-default
   systemctl list-units --type=target
   ```

2. Список юнитов, входящих в target:
   ```bash
   systemctl list-dependencies multi-user.target | head -30
   ```

3. Установить target по умолчанию (без перезагрузки):
   ```bash
   systemctl set-default multi-user.target
   systemctl get-default
   ```
   Верните обратно:
   ```bash
   systemctl set-default graphical.target
   ```

4. Переключиться на target немедленно (например, на rescue):
   ```bash
   systemctl isolate rescue.target
   ```
   > **Осторожно:** в rescue-режиме может потребоваться ввод пароля root. Для возврата — `systemctl isolate graphical.target` или `reboot`.

### 6.2. Зависимости между юнитами

Директивы в `[Unit]`:

| Директива | Действие |
|---|---|
| `Requires=A` | Запустить A; если A упадёт — упадёт и этот юнит |
| `Wants=A` | Запустить A, но не критично при падении |
| `After=A` | Запустить **после** A |
| `Before=A` | Запустить **до** A |
| `Conflicts=A` | Остановить A перед запуском |
| `PartOf=A` | Перезапуск/остановка A → то же для этого юнита |

**Пример:** сервис, который должен запускаться после `network-online.target` и требует PostgreSQL:
```ini
[Unit]
Description=My App
After=network-online.target postgresql.service
Wants=network-online.target
Requires=postgresql.service
```

1. Просмотр зависимостей сервиса:
   ```bash
   systemctl list-dependencies ssh
   systemctl list-dependencies --reverse ssh
   ```

2. Дерево зависимостей:
   ```bash
   systemd-analyze critical-chain ssh
   ```

### 6.3. Анализ времени загрузки

1. Общее время:
   ```bash
   systemd-analyze
   ```

2. По юнитам:
   ```bash
   systemd-analyze blame | head -20
   ```

3. Критическая цепочка:
   ```bash
   systemd-analyze critical-chain
   ```




## Приложение. Шпаргалка команд

| Задача | Команда |
|---|---|
| Запустить | `systemctl start UNIT` |
| Остановить | `systemctl stop UNIT` |
| Перезапустить | `systemctl restart UNIT` |
| Перечитать конфиг | `systemctl reload UNIT` |
| Включить автозапуск | `systemctl enable UNIT` |
| Выключить автозапуск | `systemctl disable UNIT` |
| Запретить запуск | `systemctl mask UNIT` |
| Разрешить запуск | `systemctl unmask UNIT` |
| Статус | `systemctl status UNIT` |
| Список всех юнитов | `systemctl list-units --all` |
| Список юнит-файлов | `systemctl list-unit-files` |
| Упавшие юниты | `systemctl --failed` |
| Зависимости | `systemctl list-dependencies UNIT` |
| Показать юнит-файл | `systemctl cat UNIT` |
| Свойства юнита | `systemctl show UNIT` |
| Текущий target | `systemctl get-default` |
| Сменить target | `systemctl set-default TARGET` |
| Переключиться в target | `systemctl isolate TARGET` |
| Перезагрузить менеджер | `systemctl daemon-reload` |
| Логи юнита | `journalctl -u UNIT` |
| Логи в реальном времени | `journalctl -f` |
| Логи текущей загрузки | `journalctl -b` |
| Только ошибки | `journalctl -p err` |
| Список таймеров | `systemctl list-timers --all` |
| Анализ загрузки | `systemd-analyze blame` |
| Проверка расписания | `systemd-analyze calendar "..."` |

**Расположение юнитов:**
```
/etc/systemd/system/        — пользовательские юниты (высший приоритет)
/usr/lib/systemd/system/    — юниты из пакетов
/run/systemd/system/        — runtime-юниты
/etc/systemd/journald.conf  — настройки журнала
```

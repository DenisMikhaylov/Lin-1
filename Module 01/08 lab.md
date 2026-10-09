# Лабораторная работа: Планирование задач в Debian 12 (cron, anacron, at, systemd timers)

**Тема:** Планирование регулярных задач через `cron`, отложенный запуск через `at`, гарантированный запуск пропущенных задач через `anacron`, интеграция с systemd timers.

**Цель работы:**
- Освоить синтаксис crontab и системных файлов `/etc/crontab`, `/etc/cron.d/`.
- Изучить каталоги `/etc/cron.{hourly,daily,weekly,monthly}/`.
- Научиться ограничивать доступ к cron (`/etc/cron.allow`, `/etc/cron.deny`).
- Освоить `anacron` и понять его отличие от `cron`.
- Научиться планировать одноразовые задачи через `at`.
- Сравнить cron/anacron с systemd timers.
- Разобраться в логировании и отладке задач.

**Оборудование и ПО:**
- Виртуальная машина с Debian 12 (Bookworm).
- Пользователь `root` (пароль `111`), пользователь `user` (пароль `111`).

**Продолжительность:** 2 академических часа (120 минут).

---

## Теоретическая справка (5 мин)

**`cron`** — демон, запускающий задачи по расписанию. Работает постоянно, точность до минуты. Если задача должна была выполниться, но машина была выключена — она **пропускается**.

**`anacron`** — надстройка над cron, гарантирует выполнение задач с периодичностью в днях, неделях, месяцах. Если машина была выключена — задача выполнится после включения. **Не работает с задачами чаще одного раза в день.**

**`at`** — планировщик одноразовых задач («выполнить один раз в указанное время»).

**`systemd timers`** — современная альтернатива cron, использующая юниты systemd. Точность до секунд, интеграция с `journald`.

**Синтаксис crontab:**
```
* * * * * команда
│ │ │ │ │
│ │ │ │ └── день недели (0–7, 0 и 7 = воскресенье; или sun, mon…)
│ │ │ └──── месяц (1–12)
│ │ └────── день месяца (1–31)
│ └──────── час (0–23)
└────────── минута (0–59)
```

**Специальные значения:**
| Запись | Значение |
|---|---|
| `*` | любое значение |
| `,` | список (`1,15,30`) |
| `-` | диапазон (`1-5`) |
| `/` | шаг (`*/15` — каждые 15 минут, `10-50/5`) |
| `@reboot` | при загрузке |
| `@daily` | раз в день в 00:00 |
| `@hourly` | каждый час |
| `@weekly`, `@monthly`, `@yearly` | раз в неделю/месяц/год |

**Расположение:**
```
/var/spool/cron/crontabs/<user>    — пользовательские crontab
/etc/crontab                       — системный crontab
/etc/cron.d/                       — дополнительные системные crontab
/etc/cron.{hourly,daily,weekly,monthly}/  — скрипты-задачи
/etc/cron.allow, /etc/cron.deny    — управление доступом
/etc/anacrontab                    — конфигурация anacron
/var/spool/anacron/                — метки последнего запуска anacron
```

---

## Этап 1. Подготовка (5 мин)

1. Войдите как `root`:
   ```bash
   su -
   # пароль: 111
   ```

2. Убедитесь, что пользователь `user` существует:
   ```bash
   id user || (useradd -m -s /bin/bash user && echo 'user:111' | chpasswd)
   ```

3. Установите необходимые пакеты:
   ```bash
   apt update
   apt install -y cron anacron at logrotate
   ```

4. Убедитесь, что службы запущены:
   ```bash
   systemctl status cron
   systemctl status anacron
   systemctl status atd
   ```

5. Если `atd` не активен — включите:
   ```bash
   systemctl enable --now atd
   ```

6. Создайте рабочий каталог для логов:
   ```bash
   mkdir -p /srv/cron-lab
   chmod 777 /srv/cron-lab
   ```

7. Проверьте часовой пояс и системное время:
   ```bash
   timedatectl
   date
   ```

---

## Этап 2. Пользовательский crontab (25 мин)

### 2.1. Основы работы с `crontab`

1. Просмотрите текущий crontab для `root`:
   ```bash
   crontab -l
   ```
   Если пусто — увидите сообщение «no crontab for root».

2. Откройте редактор:
   ```bash
   crontab -e
   ```
   Выберите редактор (по умолчанию `nano`).

3. Добавьте тестовую задачу: каждую минуту писать дату в лог.
   ```cron
   * * * * * echo "root task: $(date)" >> /srv/cron-lab/root.log
   ```

4. Сохраните и выйдите. Проверьте:
   ```bash
   crontab -l
   ```

5. Подождите 1–2 минуты, затем:
   ```bash
   cat /srv/cron-lab/root.log
   ```
   Убедитесь, что записи появляются каждую минуту.

### 2.2. Создание crontab для другого пользователя

1. От `root` отредактируйте crontab пользователя `user`:
   ```bash
   crontab -u user -e
   ```
   Добавьте:
   ```cron
   * * * * * echo "user task: $(date) by $(whoami)" >> /srv/cron-lab/user.log
   ```

2. Посмотрите список задач пользователя:
   ```bash
   crontab -u user -l
   ```

3. Через 1–2 минуты:
   ```bash
   cat /srv/cron-lab/user.log
   ```

4. Обратите внимание на права файла `/var/spool/cron/crontabs/user`:
   ```bash
   ls -l /var/spool/cron/crontabs/
   ```
   Файлы имеют права `600` и принадлежат пользователю.

### 2.3. Различные форматы расписания

Замените содержимое crontab `root` через `crontab -e` на:
```cron
# Каждую минуту
* * * * * echo "every minute" >> /srv/cron-lab/pattern.log

# Каждые 5 минут
*/5 * * * * echo "every 5 min" >> /srv/cron-lab/pattern.log

# Каждый час в 00 минут
0 * * * * echo "hourly" >> /srv/cron-lab/pattern.log

# По будням в 09:30
30 9 * * 1-5 echo "weekday morning" >> /srv/cron-lab/pattern.log

# Первое число месяца в 00:00
0 0 1 * * echo "monthly first" >> /srv/cron-lab/pattern.log

# 15 января и 15 июля в 12:00
0 12 15 1,7 * echo "biannual" >> /srv/cron-lab/pattern.log

# Специальные псевдонимы
@hourly echo "hourly via @hourly" >> /srv/cron-lab/pattern.log
@daily  echo "daily via @daily" >> /srv/cron-lab/pattern.log
```

Сохраните, посмотрите `crontab -l`. Через несколько минут проверьте содержимое:
```bash
tail -f /srv/cron-lab/pattern.log
# Ctrl+C через 15-20 секунд
```

### 2.4. Использование переменных окружения

1. В crontab можно задавать переменные:
   ```cron
   SHELL=/bin/bash
   PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
   MAILTO=""
   LOGDIR=/srv/cron-lab

   * * * * * echo "with vars" >> $LOGDIR/vars.log
   ```

2. `MAILTO=""` отключает отправку писем с выводом. По умолчанию cron отправляет весь вывод (`stdout`+`stderr`) почтой.

3. `MAILTO="user@example.com"` — отправлять почту на адрес.

4. Установите `MAILTO=""` для остальных задач.

### 2.5. Отладка: типичные ошибки

1. Путь к команде не указан полностью — используйте `which`:
   ```bash
   which echo
   which date
   ```

2. Проверьте, что `cron` использует тот же `PATH`. В логе `cron`:
   ```bash
   grep CRON /var/log/syslog | tail -20
   ```

3. Ошибки в crontab не генерируют предупреждений. Всегда проверяйте лог:
   ```bash
   journalctl -u cron -n 30
   ```

### 2.6. Очистка crontab

Удалите свои задачи:
```bash
crontab -r
crontab -l 2>&1 || echo "Пусто"
crontab -u user -r
crontab -u user -l 2>&1 || echo "Пусто для user"
```

---

## Этап 3. Системный crontab и /etc/cron.d (15 мин)

### 3.1. `/etc/crontab` — системный файл

1. Посмотрите содержимое:
   ```bash
   cat /etc/crontab
   ```

2. Обратите внимание: в системном crontab **перед командой указывается пользователь**:
   ```cron
   # m h dom mon dow user  command
   17 *  *   *   *   root  cd / && run-parts --report /etc/cron.hourly
   25 6  *   *   *   root  test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.daily )
   47 6  *   *   7   root  test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.weekly )
   52 6  1   *   *   root  test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.monthly )
   ```

3. Добавьте свою задачу (от `root`):
   ```bash
   cp /etc/crontab /etc/crontab.bak
   echo '* * * * * root echo "system crontab: $(date)" >> /srv/cron-lab/sys.log' >> /etc/crontab
   tail -3 /etc/crontab
   ```

4. Через 1 минуту:
   ```bash
   cat /srv/cron-lab/sys.log
   ```

### 3.2. `/etc/cron.d/` — дополнительные файлы

1. Создайте файл `/etc/cron.d/lab-task`:
   ```bash
   cat > /etc/cron.d/lab-task <<'EOF'
   # Пример задачи из /etc/cron.d
   SHELL=/bin/bash
   PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
   * * * * * user echo "from cron.d: $(date)" >> /srv/cron-lab/crond.log
   EOF
   ```
   Заметьте: файлы в `/etc/cron.d/` используют тот же синтаксис, что и `/etc/crontab` (с указанием пользователя), но **не имеют shebang и не должны быть исполняемыми**; также имя файла не должно содержать точки (иначе будет проигнорирован).

2. Проверьте права и имя:
   ```bash
   ls -l /etc/cron.d/lab-task
   chmod 644 /etc/cron.d/lab-task
   ```

3. Через 1 минуту:
   ```bash
   cat /srv/cron-lab/crond.log
   ```

### 3.3. Каталоги `/etc/cron.{hourly,daily,weekly,monthly}/`

1. Просмотрите содержимое:
   ```bash
   ls /etc/cron.hourly/ /etc/cron.daily/ /etc/cron.weekly/ /etc/cron.monthly/
   ```

2. Создайте задачу в `/etc/cron.hourly/`:
   ```bash
   cat > /etc/cron.hourly/lab-hourly <<'EOF'
   #!/bin/bash
   echo "hourly task: $(date)" >> /srv/cron-lab/hourly.log
   EOF
   chmod 755 /etc/cron.hourly/lab-hourly
   ```

3. Запустите вручную, чтобы проверить:
   ```bash
   run-parts --test /etc/cron.hourly
   run-parts /etc/cron.hourly
   cat /srv/cron-lab/hourly.log
   ```
   **Важно:** `run-parts` игнорирует файлы с точкой в имени.

---

## Этап 4. Ограничение доступа (10 мин)

### 4.1. `/etc/cron.allow` и `/etc/cron.deny`

1. По умолчанию в Debian — `/etc/cron.deny` существует, `/etc/cron.allow` — нет:
   ```bash
   ls -l /etc/cron.allow /etc/cron.deny 2>&1
   ```

2. Проверьте, может ли `user` создавать crontab:
   ```bash
   su - user -c 'crontab -l 2>&1 || true'
   ```

3. Добавьте `user` в `/etc/cron.deny`:
   ```bash
   echo "user" >> /etc/cron.deny
   su - user -c 'crontab -e' 2>&1 | head -3
   # Должны увидеть "You (user) are not allowed to use this program (crontab)"
   ```

4. Удалите из deny:
   ```bash
   grep -v '^user$' /etc/cron.deny > /etc/cron.deny.new
   mv /etc/cron.deny.new /etc/cron.deny
   ```

5. Создайте `/etc/cron.allow` с `user` и `root`:
   ```bash
   echo -e "user\nroot" > /etc/cron.allow
   su - user -c 'crontab -l' 2>&1 || true    # user разрешён
   su - alice -c 'crontab -l' 2>&1 || echo "alice не в allow — отказано"
   ```
   **Правило:** если существует `/etc/cron.allow`, то он имеет приоритет над `/etc/cron.deny`.

6. Удалите `/etc/cron.allow`, восстановите `/etc/cron.deny`:
   ```bash
   rm /etc/cron.allow
   # Убедитесь, что user не в deny
   grep -q '^user$' /etc/cron.deny && sed -i '/^user$/d' /etc/cron.deny
   ```

---

## Этап 5. Anacron (20 мин)

### 5.1. Что такое anacron и зачем

`cron` пропускает задачи, если машина выключена. `anacron` — гарантирует выполнение задач, которые должны были запуститься раз в день/неделю/месяц, но были пропущены. Проверяется раз в сутки (обычно через `/etc/cron.d/anacron` или systemd-таймер).

**Важно:** `anacron` не работает с задачами чаще одного раза в день.

### 5.2. Конфигурация

1. Посмотрите `/etc/anacrontab`:
   ```bash
   cat /etc/anacrontab
   ```

2. Формат строки:
   ```
   period  delay  job-identifier  command
   ```
   - **period** — периодичность в днях (1 — ежедневно, 7 — еженедельно, 30 — ежемесячно).
   - **delay** — задержка в минутах перед запуском (чтобы не перегружать систему).
   - **job-identifier** — имя задачи (используется для файла метки в `/var/spool/anacron/`).
   - **command** — команда.

3. Загляните в `/var/spool/anacron/`:
   ```bash
   ls -l /var/spool/anacron/
   cat /var/spool/anacron/cron.daily
   ```
   Файл содержит дату последнего успешного запуска задачи.

### 5.3. Добавление своей задачи в anacron

1. Создайте скрипт:
   ```bash
   cat > /srv/cron-lab/daily-task.sh <<'EOF'
   #!/bin/bash
   echo "anacron daily task: $(date)" >> /srv/cron-lab/anacron.log
   EOF
   chmod +x /srv/cron-lab/daily-task.sh
   ```

2. Добавьте строку в `/etc/anacrontab`:
   ```bash
   cp /etc/anacrontab /etc/anacrontab.bak
   echo -e "1\t5\tlab-daily\t/srv/cron-lab/daily-task.sh" >> /etc/anacrontab
   tail -3 /etc/anacrontab
   ```

3. Запустите anacron вручную для проверки:
   ```bash
   anacron -f -n -d
   # -f — force, запустить даже если не просрочено
   # -n — не форкать, выполнять в текущем процессе
   # -d — debug, выводить сообщения
   cat /srv/cron-lab/anacron.log
   ```

4. Проверьте метку:
   ```bash
   cat /var/spool/anacron/lab-daily
   ```

5. Запустите снова без `-f`:
   ```bash
   anacron -n -d 2>&1 | tail -5
   ```
   Задача не запустится, потому что уже выполнена сегодня.

6. Симулируйте «пропуск» — установите метку на дату 3 дня назад:
   ```bash
   echo "20250101" > /var/spool/anacron/lab-daily
   cat /var/spool/anacron/lab-daily
   anacron -n -d 2>&1 | tail -5
   cat /srv/cron-lab/anacron.log
   ```
   Задача выполнится как «пропущенная».

### 5.4. Как anacron вызывается автоматически

1. В Debian `anacron` запускается через systemd-таймер:
   ```bash
   systemctl status anacron.timer
   systemctl cat anacron.timer
   systemctl cat anacron.service
   ```

2. Либо через `/etc/cron.d/anacron` (если пакет cron-anacron):
   ```bash
   cat /etc/cron.d/anacron 2>/dev/null || echo "нет /etc/cron.d/anacron"
   ```

3. Просмотр расписания:
   ```bash
   systemctl list-timers | grep anacron
   ```

### 5.5. Сравнение cron и anacron

| Свойство | cron | anacron |
|---|---|---|
| Частота | от минут | от дня |
| Работа при выключенной машине | пропускает | выполняет после включения |
| Где живёт | демон | утилита, запускаемая по расписанию |
| Точность | до минуты | до дня |
| Использование | веб-серверы, 24/7 | ноутбуки, рабочие станции |

---

## Этап 6. Утилита `at` — одноразовые задачи (10 мин)

### 6.1. Основы

1. Поставьте задачу на выполнение через 2 минуты:
   ```bash
   echo 'echo "at task: $(date)" >> /srv/cron-lab/at.log' | at now + 2 minutes
   ```
   Ожидаемо: `job 1 at ...`

2. Просмотр очереди:
   ```bash
   atq
   at -l
   ```

3. Просмотр содержимого задачи:
   ```bash
   at -c 1 | tail -20
   ```

4. Отмена задачи:
   ```bash
   atrm 1
   atq
   ```

### 6.2. Интерактивный ввод

```bash
at 15:30
> echo "hi" > /srv/cron-lab/at-time.log
> Ctrl+D
```
`at` покажет время выполнения. Для отмены: `atrm <number>`.

### 6.3. Практическое задание

1. Поставьте задачу через 1 минуту:
   ```bash
   echo 'echo "at ran at $(date)" >> /srv/cron-lab/at.log' | at now + 1 minute
   atq
   ```

2. Через 1–2 минуты проверьте:
   ```bash
   cat /srv/cron-lab/at.log
   atq
   ```

### 6.4. Ограничение доступа

Аналогично cron, файлы `/etc/at.allow` и `/etc/at.deny`:
```bash
ls -l /etc/at.allow /etc/at.deny 2>&1
```

### 6.5. Логирование `at`

```bash
journalctl -u atd -n 20
grep atd /var/log/syslog | tail -10
```

---

## Этап 7. systemd timers — современная альтернатива (10 мин)

### 7.1. Простой таймер

1. Создайте сервис `/etc/systemd/system/lab-echo.service`:
   ```bash
   cat > /etc/systemd/system/lab-echo.service <<'EOF'
   [Unit]
   Description=Lab echo timer task

   [Service]
   Type=oneshot
   ExecStart=/bin/bash -c 'date >> /srv/cron-lab/systemd-timer.log'
   EOF
   ```

2. Создайте таймер `/etc/systemd/system/lab-echo.timer`:
   ```bash
   cat > /etc/systemd/system/lab-echo.timer <<'EOF'
   [Unit]
   Description=Run lab-echo every minute

   [Timer]
   OnCalendar=*:0/1
   Persistent=true
   Unit=lab-echo.service

   [Install]
   WantedBy=timers.target
   EOF
   ```

3. Активируйте:
   ```bash
   systemctl daemon-reload
   systemctl enable --now lab-echo.timer
   systemctl list-timers | grep lab-echo
   ```

4. Через 1–2 минуты:
   ```bash
   cat /srv/cron-lab/systemd-timer.log
   journalctl -u lab-echo.service -n 10
   ```

5. `Persistent=true` — аналог anacron: если система была выключена, задача запустится при следующем включении.

### 7.2. Полезные команды

```bash
systemctl list-timers --all
systemctl status lab-echo.timer
systemd-analyze calendar "*:0/1"
systemctl stop lab-echo.timer
systemctl disable lab-echo.timer
```

### 7.3. Сравнение с cron

| Свойство | cron | systemd timers |
|---|---|---|
| Точность | минута | секунда |
| Логи | syslog | journald |
| Зависимости | нет | юниты, targets |
| Persistent | через anacron | `Persistent=true` |
| Управление | crontab | `systemctl` |
| Привязка к пользователю | есть | системные или user-юниты |

---

## Этап 8. Логирование и отладка (10 мин)

### 8.1. Логи cron

1. Просмотр в syslog:
   ```bash
   grep CRON /var/log/syslog | tail -20
   ```

2. Через journald:
   ```bash
   journalctl -u cron -n 30
   journalctl -u cron --since "5 min ago"
   ```

3. Логи anacron:
   ```bash
   journalctl -u anacron -n 30
   grep anacron /var/log/syslog | tail -10
   ```

### 8.2. Типичные проблемы

1. **Команда не найдена** — укажите полный путь:
   ```cron
   * * * * * /usr/bin/date >> /srv/cron-lab/date.log
   ```

2. **Переменные не заданы** — добавьте `PATH` и другие переменные в crontab.

3. **Права на запись** — файл лога должен быть доступен пользователю:
   ```bash
   ls -l /srv/cron-lab/
   ```

4. **Задача не запускается** — проверьте `crontab -l`, `journalctl -u cron`, синтаксис.

5. **Спецсимволы `%`** — в crontab `%` означает перевод строки. Экранируйте `\%` или выносите в скрипт.

### 8.3. Проверка crontab без запуска

`crontab` не имеет встроенного валидатора, но можно использовать сторонние утилиты:
```bash
# Проверка синтаксиса через cron
crontab -l | crontab -    # переустановит crontab; ошибки покажет
```

Проверить сам файл:
```bash
cat /var/spool/cron/crontabs/root
```

---

## Этап 9. Практические сценарии (5 мин)

### 9.1. Ежедневный бэкап /etc

1. Создайте скрипт:
   ```bash
   cat > /usr/local/bin/backup-etc.sh <<'EOF'
   #!/bin/bash
   set -e
   DEST=/srv/cron-lab/backups
   mkdir -p "$DEST"
   tar -czf "$DEST/etc-$(date +%F).tar.gz" /etc
   # Удалить старше 7 дней
   find "$DEST" -name 'etc-*.tar.gz' -mtime +7 -delete
   EOF
   chmod +x /usr/local/bin/backup-etc.sh
   ```

2. Добавьте в `/etc/cron.daily/`:
   ```bash
   cat > /etc/cron.daily/lab-backup-etc <<'EOF'
   #!/bin/bash
   /usr/local/bin/backup-etc.sh
   EOF
   chmod 755 /etc/cron.daily/lab-backup-etc
   ```

3. Запустите для проверки:
   ```bash
   run-parts /etc/cron.daily
   ls -lh /srv/cron-lab/backups/
   ```

### 9.2. Напоминание раз в минуту в лог

```bash
crontab -e
```
```cron
* * * * * echo "$(date): uptime = $(uptime -p)" >> /srv/cron-lab/uptime.log
```

### 9.3. Одноразовый запуск задачи через `at`

```bash
echo 'echo "Одноразовая задача выполнена в $(date)" >> /srv/cron-lab/at-once.log' | at now + 3 minutes
atq
```

### 9.4. Задача через systemd timer

Создайте таймер, который каждые 2 минуты пишет `uptime` в лог:
```bash
cat > /etc/systemd/system/lab-uptime.service <<'EOF'
[Unit]
Description=Log uptime

[Service]
Type=oneshot
ExecStart=/bin/bash -c 'uptime >> /srv/cron-lab/uptime-systemd.log'
EOF

cat > /etc/systemd/system/lab-uptime.timer <<'EOF'
[Unit]
Description=Run every 2 minutes

[Timer]
OnCalendar=*:0/2
Persistent=true
Unit=lab-uptime.service

[Install]
WantedBy=timers.target
EOF

systemctl daemon-reload
systemctl enable --now lab-uptime.timer
systemctl list-timers | grep lab-uptime
```
Проверка через 2–4 минуты:
```bash
cat /srv/cron-lab/uptime-systemd.log
```


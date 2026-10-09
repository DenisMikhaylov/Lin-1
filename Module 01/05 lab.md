# Лабораторная работа: Создание и управление локальными пользователями в Debian 12

**Тема:** Создание, изменение и удаление локальных пользователей и групп. Управление паролями, сроком действия учётных записей, правами `sudo`. Изучение файлов `/etc/passwd`, `/etc/shadow`, `/etc/group`, `/etc/gshadow`.

**Цель работы:**
- Освоить команды `useradd`, `adduser`, `usermod`, `userdel`, `passwd`, `chage`.
- Научиться управлять локальными группами (`groupadd`, `gpasswd`, `groupmod`, `groupdel`).
- Изучить структуру файлов `/etc/passwd`, `/etc/shadow`, `/etc/group`, `/etc/gshadow`.
- Настроить права `sudo` для пользователей и групп.
- Освоить блокировку, разблокировку и удаление учётных записей.
- Управлять сроком действия паролей и учётных записей.
- Настроить каталог `/etc/skel` для новых пользователей.

**Оборудование и ПО:**
- Виртуальная машина с Debian 12 (Bookworm).
- Пользователь `root` (пароль `111`), пользователь `user` (пароль `111`).

**Продолжительность:** 2 академических часа (120 минут).

---

## Теоретическая справка (5 мин)

**Локальный пользователь** — учётная запись, хранящаяся в файлах `/etc/passwd`, `/etc/shadow`, `/etc/group`, `/etc/gshadow`. Отличается от сетевых пользователей (LDAP, NIS), которые в этой работе не рассматриваются.

**Ключевые файлы:**

| Файл | Назначение | Права |
|---|---|---|
| `/etc/passwd` | Список пользователей (без паролей) | `644` |
| `/etc/shadow` | Хэши паролей, срок действия | `640 root:shadow` |
| `/etc/group` | Список групп | `644` |
| `/etc/gshadow` | Пароли групп, администраторы | `640 root:shadow` |

**Формат `/etc/passwd`:** `name:x:UID:GID:GECOS:home:shell`

**Формат `/etc/shadow`:** `name:hash:lastchg:min:max:warn:inactive:expire:reserved`

**Диапазоны UID в Debian:**
- `0` — root.
- `1–99` — системные (создаются пакетами).
- `100–999` — системные (для служб).
- `1000–59999` — обычные пользователи.
- `60000–64999` — зарезервировано.
- `65534` — `nobody`.

**Основные команды:**
- `useradd`, `adduser` — создание пользователя.
- `usermod` — изменение.
- `userdel`, `deluser` — удаление.
- `passwd` — управление паролем.
- `chage` — управление сроком действия.
- `groupadd`, `groupmod`, `groupdel`, `gpasswd` — управление группами.
- `id`, `groups`, `whoami`, `who` — просмотр информации.

---

## Этап 1. Подготовка (5 мин)

1. Войдите как `root`:
   ```bash
   su -
   # пароль: 111
   ```

2. Убедитесь, что базовый пользователь `user` существует:
   ```bash
   id user || (useradd -m -s /bin/bash user && echo 'user:111' | chpasswd)
   id user
   ```

3. Сделайте резервные копии основных файлов (на случай ошибок):
   ```bash
   mkdir -p /root/lab-backup
   cp /etc/passwd /etc/shadow /etc/group /etc/gshadow /root/lab-backup/
   cp /etc/sudoers /root/lab-backup/ 2>/dev/null
   ls -l /root/lab-backup/
   ```

4. Проверьте, что утилиты `adduser`, `chage`, `passwd`, `usermod` доступны:
   ```bash
   which adduser useradd usermod userdel passwd chage groupadd gpasswd su sudo
   ```

---

## Этап 2. Изучение структуры файлов учётных записей (10 мин)

### 2.1. `/etc/passwd`

1. Просмотрите начало файла:
   ```bash
   head -10 /etc/passwd
   ```

2. Разберите одну строку:
   ```bash
   grep '^user:' /etc/passwd
   ```
   Пример:
   ```
   user:x:1000:1000:,,,:/home/user:/bin/bash
   ```
   - `user` — имя
   - `x` — пароль в `/etc/shadow`
   - `1000` — UID
   - `1000` — GID основной группы
   - `,,,` — GECOS (комментарий)
   - `/home/user` — домашний каталог
   - `/bin/bash` — оболочка

3. Найдите пользователей с UID ≥ 1000:
   ```bash
   awk -F: '$3>=1000 && $3<65534 {print $1, $3, $6, $7}' /etc/passwd
   ```

4. Системные пользователи:
   ```bash
   awk -F: '$3<1000 {print $1, $3}' /etc/passwd | head -15
   ```

### 2.2. `/etc/shadow`

1. Только root имеет доступ:
   ```bash
   head -5 /etc/shadow
   grep '^user:' /etc/shadow
   ```

2. Пример строки `user`:
   ```
   user:$y$j9T$...:19700:0:99999:7:::
   ```
   - Поле 2 — хэш пароля (`$y$` — yescrypt, текущий в Debian 12).
   - Поле 3 — дней с 1970-01-01 последнего изменения пароля.
   - Поле 4 — минимум дней до смены пароля.
   - Поле 5 — максимум дней.
   - Поле 6 — предупреждение за N дней.
   - Поле 7 — неактивность после истечения пароля (дней).
   - Поле 8 — дата истечения учётной записи (дней с 1970).
   - Поле 9 — зарезервировано.

### 2.3. `/etc/group` и `/etc/gshadow`

1. Просмотрите:
   ```bash
   head -10 /etc/group
   grep '^user' /etc/group
   ```

2. Формат `/etc/group`: `name:x:GID:members`

3. `/etc/gshadow`:
   ```bash
   head -5 /etc/gshadow
   ```
   Формат: `name:password:admins:members`

### 2.4. Просмотр своей учётной записи

1. Кто я:
   ```bash
   whoami
   id
   ```

2. Мои группы:
   ```bash
   groups
   ```

3. Кто сейчас в системе:
   ```bash
   who
   w
   ```

4. Последние входы:
   ```bash
   last | head -10
   ```

---

## Этап 3. Создание пользователей (25 мин)

### 3.1. `useradd` — низкоуровневая команда

1. Создайте пользователя `alice`:
   ```bash
   useradd alice
   ```

2. Проверьте результат:
   ```bash
   grep '^alice:' /etc/passwd
   grep '^alice:' /etc/shadow
   ls -ld /home/alice 2>/dev/null || echo "домашнего каталога нет"
   ```
   **Важно:** по умолчанию в Debian `useradd` **не создаёт домашний каталог** и **не задаёт пароль** (аккаунт заблокирован).

3. Удалите `alice`, чтобы создать заново с параметрами:
   ```bash
   userdel alice
   ```

4. Создайте `alice` с домашним каталогом и оболочкой:
   ```bash
   useradd -m -s /bin/bash -c "Alice Ivanova" alice
   ls -la /home/alice
   grep '^alice:' /etc/passwd
   ```

5. Задайте пароль:
   ```bash
   passwd alice
   ```
   Введите пароль `111` дважды. (В лабораторной допустимо.)

6. Проверьте вход:
   ```bash
   su - alice -c 'whoami; echo $HOME'
   ```

7. Посмотрите параметры по умолчанию для `useradd`:
   ```bash
   cat /etc/default/useradd
   cat /etc/login.defs | grep -E '^(UID|GID|PASS|CREATE|UMASK)' | head -20
   ```

### 3.2. `adduser` — высокоуровневая обёртка

1. Создайте пользователя `bob`:
   ```bash
   adduser --gecos "Bob Petrov" --disabled-password bob
   ```
   Флаг `--disabled-password` создаст пользователя без пароля.

2. Задайте пароль:
   ```bash
   echo 'bob:111' | chpasswd
   ```

3. Проверьте:
   ```bash
   id bob
   ls -la /home/bob
   ```

4. Создайте пользователя `carol` в одну команду с паролем:
   ```bash
   adduser --gecos "Carol Sidorova" --disabled-password carol
   echo 'carol:111' | chpasswd
   ```

### 3.3. Создание системного пользователя

1. Создайте системного пользователя `svc-backup` без домашнего каталога:
   ```bash
   useradd -r -s /usr/sbin/nologin -c "Backup Service" svc-backup
   ```

2. Проверьте:
   ```bash
   id svc-backup
   grep '^svc-backup:' /etc/passwd
   grep '^svc-backup:' /etc/shadow
   ```
   - UID < 1000.
   - Оболочка `/usr/sbin/nologin` запрещает интерактивный вход.

3. Попробуйте войти:
   ```bash
   su - svc-backup
   ```
   Ожидаемая ошибка: «This account is currently not available.»

### 3.4. Создание пользователя с собственным UID и GID

1. Создайте группу и пользователя с конкретным GID/UID:
   ```bash
   groupadd -g 2000 developers
   useradd -m -u 2000 -g developers -s /bin/bash -c "Dev User" devuser
   echo 'devuser:111' | chpasswd
   ```

2. Проверьте:
   ```bash
   id devuser
   ```

---

## Этап 4. Управление группами (15 мин)

### 4.1. Создание группы и добавление пользователей

1. Создайте группу `admins`:
   ```bash
   groupadd admins
   grep '^admins:' /etc/group
   ```

2. Добавьте `alice` и `bob` в `admins`:
   ```bash
   usermod -aG admins alice
   gpasswd -a bob admins
   ```

3. Проверьте членство:
   ```bash
   groups alice
   groups bob
   grep '^admins:' /etc/group
   ```

4. Удалите `bob` из группы:
   ```bash
   gpasswd -d bob admins
   groups bob
   ```

### 4.2. Замена основной группы

1. Создайте группу `testers`:
   ```bash
   groupadd testers
   ```

2. Сделайте `testers` основной группой для `carol`:
   ```bash
   usermod -g testers carol
   id carol
   ```

3. Верните прежнюю группу (по умолчанию для пользователей Debian — одноимённая группа):
   ```bash
   usermod -g carol carol
   id carol
   ```

### 4.3. Управление администраторами группы (gshadow)

1. Назначьте `alice` администратором группы `admins` (может добавлять/удалять участников без sudo):
   ```bash
   gpasswd -A alice admins
   grep '^admins:' /etc/gshadow
   ```

2. Посмотрите состав:
   ```bash
   gpasswd -M alice,bob admins
   grep '^admins:' /etc/group /etc/gshadow
   ```

### 4.4. Переименование и удаление группы

1. Переименуйте группу `testers` в `qa`:
   ```bash
   groupmod -n qa testers
   grep '^qa:' /etc/group
   ```

2. Удалите группу `qa`:
   ```bash
   groupdel qa
   ```

### 4.5. Переименование пользователя

1. Переименуйте `carol` в `caroline`:
   ```bash
   usermod -l caroline carol
   ```

2. Переименуйте домашний каталог:
   ```bash
   usermod -d /home/caroline -m caroline
   ls -ld /home/caroline
   ```

---

## Этап 5. Управление паролями и сроком действия (20 мин)

### 5.1. `passwd` — смена и блокировка

1. Смените пароль `alice`:
   ```bash
   passwd alice
   ```

2. Блокировка (`!` перед хэшем):
   ```bash
   passwd -l alice
   grep '^alice:' /etc/shadow
   su - alice -c 'whoami'    # должно отказать
   ```

3. Разблокировка:
   ```bash
   passwd -u alice
   su - alice -c 'whoami'    # должно работать
   ```

4. Удалить пароль пользователя (вход без пароля):
   ```bash
   passwd -d bob
   grep '^bob:' /etc/shadow
   ```

5. Заставить сменить пароль при следующем входе:
   ```bash
   passwd -e bob
   chage -l bob
   ```

6. Посмотреть состояние пароля:
   ```bash
   passwd -S alice
   passwd -S bob
   ```

### 5.2. `chage` — управление сроком действия

1. Посмотрите текущие параметры `alice`:
   ```bash
   chage -l alice
   ```

2. Установите: смена обязательна каждые 90 дней, предупреждение за 7 дней:
   ```bash
   chage -M 90 -W 7 alice
   chage -l alice
   ```

3. Минимальный срок между сменами — 1 день:
   ```bash
   chage -m 1 alice
   chage -l alice
   ```

4. Задайте дату истечения учётной записи (например, 31.12.2025):
   ```bash
   chage -E 2025-12-31 alice
   chage -l alice
   ```

5. Разблокируйте пароль:
   ```bash
   chage -I -1 alice
   ```

### 5.3. Политика паролей по умолчанию

1. Отредактируйте `/etc/login.defs` (только для просмотра):
   ```bash
   grep -E '^PASS_(MAX|MIN|WARN)_DAYS' /etc/login.defs
   ```

2. Установите значения для **новых** пользователей (не влияет на существующих):
   ```bash
   sed -i 's/^PASS_MAX_DAYS.*/PASS_MAX_DAYS   99999/' /etc/login.defs
   grep '^PASS_MAX_DAYS' /etc/login.defs
   ```

3. Для политики сложности паролей можно установить `libpam-pwquality`:
   ```bash
   apt install -y libpam-pwquality
   cat /etc/security/pwquality.conf | grep -v '^#' | grep -v '^$'
   ```

### 5.4. Блокировка и истечение срока

1. Заблокировать учётную запись по дате:
   ```bash
   chage -E 0 bob
   chage -l bob
   su - bob    # должно отказать
   ```

2. Вернуть возможность входа:
   ```bash
   chage -E -1 bob
   chage -l bob
   ```

3. Блокировка через `usermod`:
   ```bash
   usermod -L alice      # эквивалент passwd -l
   usermod -U alice      # эквивалент passwd -u
   ```

4. Блокировка через `usermod -e`:
   ```bash
   usermod -e 1 alice    # "завтра"
   usermod -e '' alice   # снять
   ```

---

## Этап 6. Права sudo (15 мин)

### 6.1. Добавление пользователя в sudo-группу

1. Проверьте, есть ли группа `sudo`:
   ```bash
   getent group sudo
   ```

2. Добавьте `alice` в `sudo`:
   ```bash
   usermod -aG sudo alice
   id alice
   ```

3. Проверьте:
   ```bash
   su - alice -c 'sudo -l'
   ```
   Будет запрошен пароль `alice` (`111`).

### 6.2. Прямое редактирование sudoers через `visudo`

1. Создайте отдельный файл в `/etc/sudoers.d/` (лучшая практика):
   ```bash
   visudo -f /etc/sudoers.d/devuser
   ```
   Содержимое:
   ```
   devuser ALL=(ALL:ALL) ALL
   ```
   Сохраните (Ctrl+O, Enter, Ctrl+X).

2. Проверьте синтаксис:
   ```bash
   visudo -c
   ```

3. Проверьте права:
   ```bash
   su - devuser -c 'sudo whoami'
   ```
   Пароль `devuser` — `111`.

### 6.3. Ограниченные привилегии

1. Разрешите `bob` только перезапускать службу:
   ```bash
   visudo -f /etc/sudoers.d/bob
   ```
   ```
   bob ALL=(root) NOPASSWD: /usr/bin/systemctl restart ssh, /usr/bin/systemctl status ssh
   ```

2. Проверьте:
   ```bash
   su - bob -c 'sudo systemctl status ssh'
   su - bob -c 'sudo systemctl restart apache2'   # должно отказать
   ```

3. Посмотрите `sudo -l` от лица `bob`:
   ```bash
   su - bob -c 'sudo -l'
   ```

### 6.4. Группа sudoers с полными правами

1. Создайте группу `wheel`:
   ```bash
   groupadd wheel
   ```

2. Разрешите группе `wheel` все команды без пароля:
   ```bash
   visudo -f /etc/sudoers.d/wheel
   ```
   ```
   %wheel ALL=(ALL:ALL) NOPASSWD: ALL
   ```

3. Добавьте `caroline` в `wheel`:
   ```bash
   usermod -aG wheel caroline
   su - caroline -c 'sudo whoami'
   ```

---

## Этап 7. Каталог `/etc/skel` (5 мин)

1. Добавьте файл приветствия в `/etc/skel`:
   ```bash
   cat > /etc/skel/WELCOME.txt <<'EOF'
Добро пожаловать в систему Debian 12!
Просьба не хранить пароли в открытом виде.
EOF
   ```

2. Добавьте `.bashrc.d`:
   ```bash
   mkdir -p /etc/skel/.config
   echo 'alias ll="ls -alF"' > /etc/skel/.config/aliases
   ```

3. Создайте нового пользователя `dave` и проверьте:
   ```bash
   adduser --gecos "Dave" --disabled-password dave
   echo 'dave:111' | chpasswd
   ls -la /home/dave
   cat /home/dave/WELCOME.txt
   ```

---

## Этап 8. Удаление пользователей (10 мин)

### 8.1. `userdel`

1. Удалите `dave` без домашнего каталога:
   ```bash
   userdel dave
   ls -ld /home/dave    # каталог остался
   ```

2. Удалите `bob` вместе с домашним каталогом:
   ```bash
   userdel -r bob
   ls -ld /home/bob 2>/dev/null || echo "каталог удалён"
   grep '^bob:' /etc/passwd || echo "пользователь удалён"
   ```



---

## Приложение. Шпаргалка команд

### Пользователи

| Задача | Команда |
|---|---|
| Создать пользователя (низкоуровнево) | `useradd -m -s /bin/bash -c "Comment" login` |
| Создать пользователя (высокоуровнево) | `adduser login` |
| Создать системного пользователя | `useradd -r -s /usr/sbin/nologin login` |
| Задать/сменить пароль | `passwd login` |
| Пароль одной командой | `echo 'login:pass' \| chpasswd` |
| Переименовать | `usermod -l newname oldname` |
| Изменить домашний каталог | `usermod -d /new/home -m login` |
| Изменить оболочку | `usermod -s /bin/zsh login` |
| Изменить UID | `usermod -u 2000 login` |
| Сменить основную группу | `usermod -g group login` |
| Добавить в доп. группы | `usermod -aG g1,g2 login` |
| Заменить доп. группы | `usermod -G g1,g2 login` |
| Заблокировать | `usermod -L login` / `passwd -l login` |
| Разблокировать | `usermod -U login` / `passwd -u login` |
| Удалить пользователя | `userdel login` |
| Удалить с домашним каталогом | `userdel -r login` / `deluser --remove-home login` |
| Информация о пользователе | `id login`, `finger login` |
| Список пользователей | `getent passwd`, `cat /etc/passwd` |

### Группы

| Задача | Команда |
|---|---|
| Создать группу | `groupadd -g 2000 group` |
| Переименовать | `groupmod -n new old` |
| Удалить | `groupdel group` |
| Добавить пользователя | `gpasswd -a login group` / `usermod -aG group login` |
| Удалить пользователя | `gpasswd -d login group` |
| Список участников | `getent group group` |
| Администратор группы | `gpasswd -A login group` |
| Заменить состав | `gpasswd -M u1,u2 group` |
| Информация о группах пользователя | `groups login` |

### Пароли и срок действия

| Задача | Команда |
|---|---|
| Статус пароля | `passwd -S login` |
| Удалить пароль | `passwd -d login` |
| Истечь пароль | `passwd -e login` |
| Политика пароля | `chage -l login` |
| Мин. срок | `chage -m 1 login` |
| Макс. срок | `chage -M 90 login` |
| Предупреждение | `chage -W 7 login` |
| Дата истечения | `chage -E 2025-12-31 login` |
| Неактивность | `chage -I 30 login` |

### Просмотр и редактирование

| Задача | Команда |
|---|---|
| Просмотр пользователей | `getent passwd` |
| Просмотр теней | `getent shadow` (только root) |
| Редактировать `/etc/passwd` | `vipw` |
| Редактировать `/etc/shadow` | `vipw -s` |
| Редактировать `/etc/group` | `vigr` |
| Редактировать `/etc/gshadow` | `vigr -s` |
| Проверка sudoers | `visudo -c` |
| Кто в системе | `who`, `w`, `last` |

**Расположение файлов:**
```
/etc/passwd          — список пользователей
/etc/shadow          — пароли и срок действия
/etc/group           — список групп
/etc/gshadow         — пароли групп и администраторы
/etc/login.defs      — политика по умолчанию для новых пользователей
/etc/default/useradd — параметры useradd по умолчанию
/etc/skel/           — шаблон домашнего каталога
/etc/sudoers         — основной файл sudo
/etc/sudoers.d/      — отдельные файлы sudo
```

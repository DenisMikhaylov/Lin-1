# Лабораторная работа: Права доступа в Linux (chmod, chown, chgrp, umask, ACL)

**Тема:** Управление правами доступа в Debian 12. Команды `chmod`, `chown`, `chgrp`, `umask`, специальные биты (SUID, SGID, Sticky), ACL (`setfacl`, `getfacl`).

**Цель работы:**
- Разобраться в модели прав доступа Linux (владелец, группа, остальные).
- Освоить числовую и символьную запись прав.
- Научиться менять владельца и группу файлов (`chown`, `chgrp`).
- Изучить специальные биты: SUID, SGID, Sticky bit.
- Освоить `umask` и права по умолчанию.
- Научиться использовать расширенные ACL (`setfacl`, `getfacl`).

**Оборудование и ПО:**
- Виртуальная машина с Debian 12 (Bookworm).
- Пользователь `root` (пароль `111`), пользователь `user` (пароль `111`).

**Продолжительность:** 2 академических часа (120 минут).

---

## Теоретическая справка (5 мин)

Каждый файл и каталог в Linux имеет **владельца** (UID) и **группу** (GID) и три набора прав:

```
- rwx r-x r--  1 alice developers  4096 Oct 9 12:00 file.txt
│ └┬┘ └┬┘ └┬┘   │  │      │
│  │   │   │    │  │      └── группа
│  │   │   │    │  └───────── владелец
│  │   │   │    └──────────── число жёстких ссылок
│  │   │   └───────────────── права остальных (others)
│  │   └───────────────────── права группы (group)
│  └───────────────────────── права владельца (user)
└──────────────────────────── тип файла: - (файл), d (каталог), l (ссылка), c, b, s, p
```

**Числовая запись:**

| Право | Бит | Значение |
|---|---|---|
| `r` (read) | 4 | чтение |
| `w` (write) | 2 | запись |
| `x` (execute) | 1 | выполнение |
| `-` | 0 | нет права |

Сумма для каждой тройки: `rwx` = 7, `rw-` = 6, `r-x` = 5, `r--` = 4, `-wx` = 3, `-w-` = 2, `--x` = 1, `---` = 0.

**Специальные биты:**
- **SUID** (4xxx) — файл выполняется с правами владельца.
- **SGID** (2xxx) — файл выполняется с правами группы; для каталога — новые файлы получают группу каталога.
- **Sticky bit** (1xxx) — в каталоге удалять файлы может только владелец файла (или владелец каталога).

**Права по умолчанию (umask):** обычно `022` → новые файлы `644`, новые каталоги `755`.

---

## Этап 1. Подготовка (5 мин)

1. Войдите как `root`:
   ```bash
   su -
   # пароль: 111
   ```

2. Убедитесь, что есть пользователи `user`, а также создайте двух тестовых:
   ```bash
   id user  || (useradd -m -s /bin/bash user  && echo 'user:111'  | chpasswd)
   id alice || (useradd -m -s /bin/bash alice && echo 'alice:111' | chpasswd)
   id bob   || (useradd -m -s /bin/bash bob   && echo 'bob:111'   | chpasswd)
   ```

3. Создайте группы:
   ```bash
   groupadd -f developers
   groupadd -f testers
   usermod -aG developers alice
   usermod -aG testers   bob
   usermod -aG developers bob
   ```

4. Создайте рабочий каталог:
   ```bash
   mkdir -p /srv/perm-lab
   cd /srv/perm-lab
   ```

5. Подготовьте несколько тестовых файлов:
   ```bash
   echo "Hello, world" > file1.txt
   echo "Another file"  > file2.txt
   mkdir dir1 dir2
   touch dir1/inner.txt dir2/inner.txt
   echo '#!/bin/bash
   echo "I am running with UID=$UID"' > script.sh
   chmod +x script.sh
   ls -l
   ```

6. Сделайте снимок прав для последующего сравнения:
   ```bash
   ls -lR /srv/perm-lab > /root/perm-before.txt
   ```

---

## Этап 2. Просмотр прав: `ls -l`, `stat` (10 мин)

### 2.1. Просмотр через `ls`

1. Обычный вывод:
   ```bash
   ls -l /srv/perm-lab
   ```

2. Права каталогов:
   ```bash
   ls -ld /srv/perm-lab dir1 dir2
   ```

3. Права числом:
   ```bash
   stat -c '%a %n' file1.txt dir1 script.sh
   ```

4. Полная информация о файле:
   ```bash
   stat file1.txt
   ```

5. Разбор вывода `stat`:
   - `Access: (0644/-rw-r--r--)` — права в числовом и символьном виде.
   - `Uid: (0/root)` — владелец.
   - `Gid: (0/root)` — группа.
   - `Access`, `Modify`, `Change` — времена доступа.

### 2.2. Просмотр через `namei`

Показывает права по всему пути:
```bash
namei -l /srv/perm-lab/file1.txt
```

### 2.3. Что означают права для каталогов

- **`r`** — можно прочитать список файлов (`ls`).
- **`w`** — можно создавать, удалять, переименовывать файлы.
- **`x`** — можно «войти» в каталог (`cd`) и обращаться к файлам по имени.

**Проверка 1:** заберите `x`, оставьте `r`:
```bash
chmod 644 dir1
su - user -c 'ls /srv/perm-lab/dir1' 2>&1 | head -2
su - user -c 'cat /srv/perm-lab/dir1/inner.txt' 2>&1 | head -2
```
Без `x` нельзя обратиться к файлам внутри, даже если знаете имя.

**Проверка 2:** верните `x`:
```bash
chmod 755 dir1
su - user -c 'cat /srv/perm-lab/dir1/inner.txt'
```

---

## Этап 3. `chmod` — изменение прав (25 мин)

### 3.1. Символьная запись

Кому: `u` (user), `g` (group), `o` (others), `a` (all).
Действие: `+` (добавить), `-` (убрать), `=` (установить точно).
Право: `r`, `w`, `x`, `X` (execute только для каталогов и уже исполняемых), `s` (SUID/SGID), `t` (sticky).

1. Добавить право выполнения владельцу:
   ```bash
   chmod u+x file1.txt
   ls -l file1.txt
   ```

2. Убрать чтение у остальных:
   ```bash
   chmod o-r file1.txt
   ls -l file1.txt
   ```

3. Установить права группы: `r-x`:
   ```bash
   chmod g=rx file1.txt
   ls -l file1.txt
   ```

4. Одновременно несколько изменений:
   ```bash
   chmod u=rw,g=r,o= file1.txt
   ls -l file1.txt
   ```

5. Для всех: убрать запись для группы и остальных:
   ```bash
   chmod go-w file1.txt
   ls -l file1.txt
   ```

6. Каскадно на каталог и файлы (`-R`):
   ```bash
   chmod -R u=rwX,g=rX,o= dir1
   ls -lR dir1
   ```
   Обратите внимание: `X` устанавливается на каталоги и на файлы, у которых уже был `x`, но **не** делает обычные файлы исполняемыми.

### 3.2. Числовая запись

1. Установить `rw-r--r--`:
   ```bash
   chmod 644 file2.txt
   ls -l file2.txt
   ```

2. Установить `rwxr-xr-x`:
   ```bash
   chmod 755 script.sh
   ls -l script.sh
   ```

3. Установить `rwx------`:
   ```bash
   chmod 700 dir2
   ls -ld dir2
   ```

4. Установить `rw-rw----`:
   ```bash
   chmod 660 file1.txt
   ls -l file1.txt
   ```

### 3.3. Практика: восстановить исходные права

```bash
chmod 644 file1.txt file2.txt
chmod 755 dir1 dir2
chmod 755 script.sh
ls -l
```

### 3.4. `chmod` с `--reference`

Скопировать права с другого файла:
```bash
chmod --reference=file1.txt file2.txt
ls -l file1.txt file2.txt
```

---

## Этап 4. `chown` и `chgrp` — смена владельца и группы (20 мин)

### 4.1. `chown` — смена владельца

1. Смените владельца файла:
   ```bash
   chown alice file1.txt
   ls -l file1.txt
   ```

2. Смена владельца и группы одновременно:
   ```bash
   chown bob:testers file2.txt
   ls -l file2.txt
   ```

3. Сменить только группу через `chown`:
   ```bash
   chown :developers file1.txt
   ls -l file1.txt
   ```

4. Рекурсивно:
   ```bash
   chown -R alice:developers dir1
   ls -lR dir1
   ```

5. Сохранить группу при смене владельца:
   ```bash
   chown alice file2.txt   # группа остаётся прежней
   ls -l file2.txt
   ```

6. Смена владельца по ссылке (`-h`) или цели (`--dereference`):
   ```bash
   ln -s file1.txt link1
   chown -h bob link1     # меняет владельца самой ссылки
   ls -l link1
   chown --dereference bob link1  # меняет владельца файла-цели
   ls -l file1.txt link1
   ```

### 4.2. `chgrp` — смена группы

1. Смените группу:
   ```bash
   chgrp developers file2.txt
   ls -l file2.txt
   ```

2. Рекурсивно:
   ```bash
   chgrp -R testers dir2
   ls -lR dir2
   ```

### 4.3. Кто может менять владельца?

1. Проверьте от имени пользователя:
   ```bash
   chown user /srv/perm-lab/file1.txt 2>&1 || echo "Отказано — только root может менять владельца"
   ```

2. Пользователь может сменить группу файла на группу, в которой он состоит:
   ```bash
   chown alice:developers /srv/perm-lab/file1.txt 2>&1 || true
   su - alice -c 'chgrp developers /srv/perm-lab/file1.txt' 2>&1 || echo "Нет прав на файл"
   ```

3. Если файл принадлежит `alice` и она в группе `developers` — сработает:
   ```bash
   chown alice /srv/perm-lab/file1.txt
   su - alice -c 'chgrp developers /srv/perm-lab/file1.txt' && ls -l /srv/perm-lab/file1.txt
   ```

---

## Этап 5. `umask` — права по умолчанию (15 мин)

### 5.1. Как работает umask

**`umask`** — маска, которая «вычитается» из базовых прав:
- Для файлов базовые права = `666`.
- Для каталогов базовые права = `777`.
- Итоговые права = `base AND NOT umask`.

Пример: `umask 022` → файлы `644`, каталоги `755`.

### 5.2. Просмотр и установка

1. Посмотрите текущую маску:
   ```bash
   umask
   umask -S
   ```

2. Установите новую маску:
   ```bash
   umask 027
   umask
   ```

3. Создайте файлы и каталоги:
   ```bash
   touch test_mask.txt
   mkdir test_mask_dir
   ls -ld test_mask.txt test_mask_dir
   ```
   Ожидаемо: файл `640`, каталог `750`.

4. Установите `077`:
   ```bash
   umask 077
   touch private.txt
   mkdir private_dir
   ls -ld private.txt private_dir
   ```

5. Верните стандартную маску:
   ```bash
   umask 022
   ```

### 5.3. Постоянная настройка

1. Для системы: `/etc/profile`, `/etc/login.defs` (директива `UMASK`).
2. Для пользователя: `~/.bashrc`, `~/.profile`.

Проверим:
```bash
grep -E '^UMASK' /etc/login.defs
```

3. Установите `UMASK 027` глобально (демонстрация):
   ```bash
   cp /etc/login.defs /etc/login.defs.bak
   sed -i 's/^UMASK.*/UMASK           027/' /etc/login.defs
   grep '^UMASK' /etc/login.defs
   ```

4. Восстановите:
   ```bash
   mv /etc/login.defs.bak /etc/login.defs
   ```

---

## Этап 6. Специальные биты: SUID, SGID, Sticky (20 мин)

### 6.1. SUID (Set User ID)

Позволяет обычному пользователю выполнить файл с правами владельца файла.

1. Посмотрите на классический пример:
   ```bash
   ls -l /usr/bin/passwd
   ```
   Вы увидите: `-rwsr-xr-x`. Буква `s` в позиции владельца = SUID.

2. Создайте свой SUID-файл:
   ```bash
   cat > /srv/perm-lab/uid_demo.sh <<'EOF'
   #!/bin/bash
   echo "Реальный UID: $UID"
   echo "Эффективный UID: $(id -u)"
   EOF
   chmod 755 /srv/perm-lab/uid_demo.sh
   chown root:root /srv/perm-lab/uid_demo.sh
   ```

3. Запустите от `user`:
   ```bash
   su - user -c '/srv/perm-lab/uid_demo.sh'
   ```

4. Установите SUID:
   ```bash
   chmod u+s /srv/perm-lab/uid_demo.sh
   ls -l /srv/perm-lab/uid_demo.sh
   ```

5. Снова запустите от `user`:
   ```bash
   su - user -c '/srv/perm-lab/uid_demo.sh'
   ```
   > **Важно:** для скриптов ядро Linux игнорирует SUID, поэтому эффективный UID останется тем же. SUID реально работает только для скомпилированных бинарников. Символьная демонстрация: права `-rwsr-xr-x` установлены, но ядро не применит их к скрипту.

6. Снимите SUID:
   ```bash
   chmod u-s /srv/perm-lab/uid_demo.sh
   ls -l /srv/perm-lab/uid_demo.sh
   ```

**Числовая запись SUID:** `4755`, `4711` и т.п.

7. Поиск всех SUID-файлов в системе:
   ```bash
   find / -perm -4000 -type f 2>/dev/null | head -20
   ```

### 6.2. SGID (Set Group ID)

1. Для файла — выполняется с правами группы.
2. **Для каталога** — новые файлы наследуют группу каталога, а не группу создателя. Это часто используется в командной работе.

1. Создайте каталог и установите SGID:
   ```bash
   mkdir /srv/perm-lab/team
   chown root:developers /srv/perm-lab/team
   chmod 2775 /srv/perm-lab/team
   ls -ld /srv/perm-lab/team
   ```
   В групповой позиции владельца будет `s`: `drwxrwsr-x`.

2. Создайте файлы от разных пользователей:
   ```bash
   su - alice -c 'touch /srv/perm-lab/team/alice.txt'
   su - bob   -c 'touch /srv/perm-lab/team/bob.txt'
   ls -l /srv/perm-lab/team
   ```
   Все файлы получат группу `developers`, а не личные группы создателей.

3. Поиск SGID-файлов:
   ```bash
   find / -perm -2000 -type f 2>/dev/null | head -10
   ```

4. Поиск SGID-каталогов:
   ```bash
   find / -perm -2000 -type d 2>/dev/null | head -10
   ```

### 6.3. Sticky bit

Актуален для каталогов: удалять и переименовывать файл может только владелец файла, владелец каталога или root. Классический пример — `/tmp`.

1. Проверьте `/tmp`:
   ```bash
   ls -ld /tmp
   ```
   Символ `t` в конце: `drwxrwxrwt`.

2. Создайте общий каталог:
   ```bash
   mkdir /srv/perm-lab/shared
   chmod 1777 /srv/perm-lab/shared
   ls -ld /srv/perm-lab/shared
   ```

3. Создайте файлы от разных пользователей:
   ```bash
   su - alice -c 'echo "alice data" > /srv/perm-lab/shared/alice.txt'
   su - bob   -c 'echo "bob data"   > /srv/perm-lab/shared/bob.txt'
   ls -l /srv/perm-lab/shared
   ```

4. Попробуйте от `bob` удалить файл `alice.txt`:
   ```bash
   su - bob -c 'rm /srv/perm-lab/shared/alice.txt' 2>&1 || echo "Отказано — работает sticky bit"
   ```

5. Снимите sticky bit и повторите:
   ```bash
   chmod 0777 /srv/perm-lab/shared
   ls -ld /srv/perm-lab/shared
   su - bob -c 'rm /srv/perm-lab/shared/alice.txt' 2>&1 && echo "Удалено — sticky bit был снят"
   ```
   Верните sticky bit:
   ```bash
   chmod 1777 /srv/perm-lab/shared
   ```

**Числовая запись:** SUID=4xxx, SGID=2xxx, Sticky=1xxx. Комбинация: `chmod 6755` = SUID+SGID.

### 6.4. Поиск «опасных» прав

1. Файлы, доступные на запись всем:
   ```bash
   find /srv/perm-lab -type f -perm -0002 2>/dev/null
   ```

2. Файлы без владельца (UID больше нет в системе):
   ```bash
   find /srv/perm-lab -nouser -o -nogroup 2>/dev/null
   ```

---

## Этап 7. Расширенные ACL (`getfacl`, `setfacl`) (15 мин)

**ACL** позволяет задавать права не только владельцу/группе/остальным, но и произвольным пользователям и группам.

1. Установите пакет:
   ```bash
   apt install -y acl
   ```

2. Посмотрите ACL файла по умолчанию:
   ```bash
   getfacl /srv/perm-lab/file1.txt
   ```
   Строки с `#` — комментарии, `user::`, `group::`, `other::` — обычные права, они дублируют `chmod`.

3. Разрешите пользователю `bob` читать файл, принадлежащий `alice`:
   ```bash
   chown alice:alice /srv/perm-lab/file1.txt
   chmod 640 /srv/perm-lab/file1.txt
   su - bob -c 'cat /srv/perm-lab/file1.txt' 2>&1 || echo "Отказано"
   setfacl -m u:bob:r /srv/perm-lab/file1.txt
   su - bob -c 'cat /srv/perm-lab/file1.txt'
   ```

4. Проверьте ACL:
   ```bash
   getfacl /srv/perm-lab/file1.txt
   ls -l /srv/perm-lab/file1.txt
   ```
   Обратите внимание на знак `+` в правах после `ls -l` — он указывает на наличие ACL.

5. Дайте право `rwx` всей группе `testers`:
   ```bash
   setfacl -m g:testers:rwx /srv/perm-lab/file1.txt
   getfacl /srv/perm-lab/file1.txt
   ```

6. Удалите отдельную запись:
   ```bash
   setfacl -x u:bob /srv/perm-lab/file1.txt
   getfacl /srv/perm-lab/file1.txt
   ```

7. Удалите все ACL:
   ```bash
   setfacl -b /srv/perm-lab/file1.txt
   ls -l /srv/perm-lab/file1.txt
   ```

8. **Маска ACL** — верхняя граница прав для именованных пользователей/групп:
   ```bash
   setfacl -m u:bob:rwx /srv/perm-lab/file1.txt
   setfacl -m m::r /srv/perm-lab/file1.txt
   getfacl /srv/perm-lab/file1.txt
   ```
   Права `bob` будут урезаны маской.

9. **ACL по умолчанию** для каталога (наследуется новыми файлами):
   ```bash
   mkdir /srv/perm-lab/acl-dir
   setfacl -d -m u:bob:rwx /srv/perm-lab/acl-dir
   getfacl /srv/perm-lab/acl-dir
   touch /srv/perm-lab/acl-dir/new.txt
   getfacl /srv/perm-lab/acl-dir/new.txt
   ```

10. Резервная копия и восстановление ACL:
    ```bash
    getfacl -R /srv/perm-lab > /root/perm-acl-backup.txt
    setfacl --restore=/root/perm-acl-backup.txt
    ```

---

## Этап 8. Практические сценарии (10 мин)

### 8.1. Совместная папка для группы

Задача: папка `/srv/perm-lab/project`, владелец `root`, группа `developers`, все файлы внутри наследуют группу и доступны на чтение/запись участникам группы.

1. Настройка:
   ```bash
   mkdir /srv/perm-lab/project
   chown root:developers /srv/perm-lab/project
   chmod 2770 /srv/perm-lab/project
   ls -ld /srv/perm-lab/project
   ```
   Права: `rwxrws---` — владелец и группа имеют полный доступ, остальные — нет. SGID гарантирует, что все файлы получат группу `developers`.

2. Проверьте от имени `alice` (в группе `developers`):
   ```bash
   su - alice -c 'touch /srv/perm-lab/project/alice.txt && ls -l /srv/perm-lab/project/alice.txt'
   ```

3. От имени `bob` (тоже в группе):
   ```bash
   su - bob -c 'touch /srv/perm-lab/project/bob.txt && ls -l /srv/perm-lab/project/bob.txt'
   ```

4. От имени `user` (не в группе):
   ```bash
   su - user -c 'ls /srv/perm-lab/project' 2>&1 || echo "Отказано — user не в группе developers"
   ```

### 8.2. Приватный каталог

```bash
mkdir /srv/perm-lab/private
chown user:user /srv/perm-lab/private
chmod 700 /srv/perm-lab/private
ls -ld /srv/perm-lab/private
su - user -c 'touch /srv/perm-lab/private/secret.txt'
su - alice -c 'ls /srv/perm-lab/private' 2>&1 || echo "Отказано"
```

### 8.3. Скрипт, доступный только владельцу

```bash
cat > /srv/perm-lab/secret.sh <<'EOF'
#!/bin/bash
echo "Только владелец может это выполнить"
EOF
chmod 700 /srv/perm-lab/secret.sh
chown user:user /srv/perm-lab/secret.sh
ls -l /srv/perm-lab/secret.sh
su - user  -c '/srv/perm-lab/secret.sh'
su - alice -c '/srv/perm-lab/secret.sh' 2>&1 || echo "Отказано"
```

---


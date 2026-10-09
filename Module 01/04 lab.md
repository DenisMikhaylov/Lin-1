# Лабораторная работа: Управление пакетами в Debian 12 с помощью dpkg и apt

**Тема:** Низкоуровневая работа с пакетами (`dpkg`), управление репозиториями, установка/обновление/удаление пакетов (`apt`), работа с зависимостями, кэшем, apt-пиннинг и локальные репозитории.

**Цель работы:**
- Освоить низкоуровневую утилиту `dpkg` для работы с `.deb`-пакетами.
- Изучить работу с `apt`: репозитории, установка, обновление, удаление, очистка.
- Разобраться с зависимостями, конфликтами и `apt-cache`.
- Научиться собирать простой `.deb`-пакет и настраивать локальный репозиторий.
- Освоить `apt-pinning` и работу с `apt` в нестабильных ситуациях.

**Оборудование и ПО:**
- Виртуальная машина с Debian 12 (Bookworm).
- Пользователь `root` (пароль `111`), пользователь `user` (пароль `111`).

**Продолжительность:** 2 академических часа (120 минут).

---

## Теоретическая справка (5 мин)

**`dpkg`** — низкоуровневый менеджер пакетов Debian. Устанавливает, удаляет, конфигурирует `.deb`-пакеты, но **не умеет автоматически разрешать зависимости** и **не работает с репозиториями**.

**`apt`** — высокоуровневая надстройка над `dpkg`, работает с репозиториями, автоматически разрешает зависимости, загружает пакеты из сети.

**Основные понятия:**
- **Пакет `.deb`** — архив с файлами, метаданными (`control`) и скриптами (`preinst`, `postinst`, `prerm`, `postrm`).
- **Репозиторий** — сервер с пакетами и индексом.
- **Зависимости**: `Depends`, `Pre-Depends`, `Recommends`, `Suggests`, `Conflicts`, `Breaks`, `Replaces`, `Provides`.

**Расположение apt-конфигурации:**
```
/etc/apt/sources.list              — основной список репозиториев
/etc/apt/sources.list.d/           — дополнительные списки
/etc/apt/apt.conf.d/               — конфигурация apt
/etc/apt/preferences.d/            — настройки пиннинга
/var/lib/apt/lists/                — кэш индексов
/var/cache/apt/archives/           — кэш .deb-файлов
/var/lib/dpkg/status               — БД установленных пакетов
```

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

3. Проверьте версию Debian и текущую нагрузку:
   ```bash
   cat /etc/os-release
   dpkg --print-architecture
   dpkg --print-foreign-architectures
   ```

4. Сделайте снимок текущего состояния (для отката):
   ```bash
   dpkg --get-selections > /root/pkg-state-before.txt
   ```

---

## Этап 2. Основы `dpkg` (20 мин)

### 2.1. Просмотр установленных пакетов

1. Список всех установленных пакетов:
   ```bash
   dpkg -l | head -30
   ```

2. Поиск пакета по имени:
   ```bash
   dpkg -l | grep bash
   dpkg -l 'bash*'
   ```

3. Проверить, установлен ли конкретный пакет:
   ```bash
   dpkg -s bash
   dpkg -s nano
   ```

4. Показать файлы, установленные пакетом:
   ```bash
   dpkg -L bash | head -20
   ```

5. Определить, какому пакету принадлежит файл:
   ```bash
   dpkg -S /bin/ls
   dpkg -S /usr/bin/apt
   ```

### 2.2. Статусы пакетов

1. Все пакеты в состоянии `ii` (установлены корректно):
   ```bash
   dpkg -l | awk '$1=="ii" {print $2}' | head -10
   ```

2. Пакеты в «некорректном» состоянии (`iU`, `iF`, `rc`):
   ```bash
   dpkg -l | awk '$1 !~ /^ii/ && NR>5 {print}' | head -20
   ```

3. Расшифровка статусов:
   - Первая буква — желаемое состояние (`i` — install, `r` — remove, `p` — purge, `h` — hold).
   - Вторая — текущее (`n` — not installed, `i` — installed, `c` — config-files, `U` — unpacked, `F` — half-configured, `H` — half-installed, `W` — triggers-awaited, `t` — triggers-pending).

### 2.3. Работа с `.deb`-файлом напрямую

1. Скачайте `.deb`-пакет вручную (например, `hello`):
   ```bash
   mkdir -p /root/deb-lab && cd /root/deb-lab
   apt download hello
   ls -lh
   ```

2. Посмотрите метаинформацию без установки:
   ```bash
   dpkg -I hello_*.deb
   ```

3. Посмотрите список файлов внутри пакета:
   ```bash
   dpkg -c hello_*.deb
   ```

4. Установите пакет напрямую:
   ```bash
   dpkg -i hello_*.deb
   hello
   ```

5. Удалите пакет (но оставьте конфиги):
   ```bash
   dpkg -r hello
   ```

6. Удалите пакет полностью с конфигами:
   ```bash
   dpkg -P hello
   ```

### 2.4. Демонстрация проблемы зависимостей

1. Скачайте пакет, у которого много зависимостей (например, `nginx`):
   ```bash
   apt download nginx
   ls
   ```

2. Попробуйте установить через `dpkg`:
   ```bash
   dpkg -i nginx_*.deb
   ```
   Появится ошибка о неудовлетворённых зависимостях.

3. Разрешите зависимости через `apt` (специально):
   ```bash
   apt --fix-broken install
   ```

4. Убедитесь, что `nginx` установлен:
   ```bash
   dpkg -l nginx
   ```

5. Удалите:
   ```bash
   apt remove --purge -y nginx
   apt autoremove -y
   ```


---

## Этап 3. Основы `apt`: репозитории и индексы (20 мин)

### 3.1. Файлы репозиториев

1. Посмотрите текущий `sources.list`:
   ```bash
   cat /etc/apt/sources.list
   ```

2. Дополнительные списки:
   ```bash
   ls /etc/apt/sources.list.d/
   cat /etc/apt/sources.list.d/*.list 2>/dev/null
   cat /etc/apt/sources.list.d/*.sources 2>/dev/null
   ```

3. Формат `.sources` (deb822, новый формат в Debian 12):
   ```bash
   cat /etc/apt/sources.list.d/debian.sources 2>/dev/null || echo "Не найден"
   ```

### 3.2. Обновление индексов и обновление системы

1. Обновить список пакетов:
   ```bash
   apt update
   ```

2. Посмотреть, что можно обновить:
   ```bash
   apt list --upgradable
   ```

3. Обновить систему без удаления пакетов и без установки новых:
   ```bash
   apt upgrade -y
   ```

4. Полное обновление (может удалять/добавлять пакеты):
   ```bash
   apt full-upgrade -y
   ```

5. Только обновить конкретный пакет:
   ```bash
   apt install --only-upgrade bash
   ```

### 3.3. Поиск информации о пакетах

1. Поиск по имени и описанию:
   ```bash
   apt search nginx
   ```

2. Показать метаинформацию:
   ```bash
   apt show nginx
   ```

3. Показать все доступные версии:
   ```bash
   apt list -a nginx
   apt-cache policy nginx
   ```

4. Какие пакеты зависят от `nginx`:
   ```bash
   apt-cache rdepends nginx
   ```

5. Какие зависимости у пакета:
   ```bash
   apt-cache depends nginx
   ```

6. Поиск по установленным файлам:
   ```bash
   apt-file search /usr/bin/nginx 2>/dev/null || echo "apt-file не установлен"
   ```

### 3.4. Установка и удаление

1. Установка пакета:
   ```bash
   apt install -y htop
   ```

2. Установка без установки рекомендованных:
   ```bash
   apt install --no-install-recommends -y tree
   ```

3. Имитация установки (dry-run):
   ```bash
   apt install --simulate -y cowsay
   ```

4. Удаление пакета (с сохранением конфигов):
   ```bash
   apt remove -y htop
   ```

5. Полное удаление:
   ```bash
   apt purge -y tree
   ```

6. Удаление неиспользуемых зависимостей:
   ```bash
   apt autoremove -y
   apt autoclean
   apt clean
   ```

7. Проверка на «битые» зависимости:
   ```bash
   apt check
   ```

---



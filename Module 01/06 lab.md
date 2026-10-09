# Лабораторная работа: Архивация и сжатие данных в Debian 12 (tar, gzip, xz, bzip2)

**Тема:** Работа с архивами и сжатием данных: `tar`, `gzip`, `gunzip`, `bzip2`, `bunzip2`, `xz`, `unxz`, `zip`/`unzip`. Создание, распаковка, просмотр содержимого архивов. Совместное использование `tar` с компрессорами. Разделение и сборка больших архивов.

**Цель работы:**
- Освоить утилиты сжатия `gzip`, `bzip2`, `xz` и их параметры.
- Научиться создавать и распаковывать архивы `tar` с разными компрессорами.
- Освоить просмотр содержимого архива без распаковки.
- Научиться архивировать с сохранением прав, владельцев и символических ссылок.
- Освоить `zip`/`unzip` и сравнить их с `tar`+компрессор.
- Научиться разделять большие архивы и собирать их обратно.
- Сравнить степень сжатия и скорость разных алгоритмов.

**Оборудование и ПО:**
- Виртуальная машина с Debian 12 (Bookworm).
- Пользователь `root` (пароль `111`), пользователь `user` (пароль `111`).

**Продолжительность:** 2 академических часа (120 минут).

---

## Теоретическая справка (5 мин)

**Архивация** — объединение нескольких файлов в один (например, `tar`).
**Сжатие** — уменьшение размера данных за счёт кодирования (например, `gzip`, `bzip2`, `xz`).

**`tar`** — «Tape ARchiver». Умеет архивировать и вызывать внешний компрессор. Сам по себе не сжимает.

**Сравнение компрессоров:**

| Утилита | Расширение | Скорость | Степень сжатия | Особенности |
|---|---|---|---|---|
| `gzip` | `.gz` | высокая | низкая | Универсальный, по умолчанию в `tar -z` |
| `bzip2` | `.bz2` | средняя | средняя | Лучше, чем gzip, но медленнее |
| `xz` | `.xz` | низкая | высокая | Лучшее сжатие, но требует много памяти и CPU |
| `zstd` | `.zst` | очень высокая | высокая | Современный, баланс скорости и сжатия |
| `zip` | `.zip` | средняя | средняя | Совместим с Windows |

**Ключевые ключи `tar`:**
- `-c` — создать архив.
- `-x` — распаковать.
- `-t` — показать содержимое.
- `-f` — имя файла архива.
- `-v` — подробный вывод.
- `-z` — через `gzip`.
- `-j` — через `bzip2`.
- `-J` — через `xz`.
- `--zstd` — через `zstd`.
- `-p` — сохранить права.
- `--numeric-owner` — сохранить числовые UID/GID.
- `-C dir` — сменить каталог.
- `-r`, `-u` — добавить/обновить (только для несжатых архивов).
- `--exclude=PATTERN` — исключить файлы.

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

3. Установите необходимые утилиты (в Debian 12 базовый набор уже есть, но проверим):
   ```bash
   apt update
   apt install -y tar gzip bzip2 xz-utils zip unzip zstd p7zip-full file bc
   ```

4. Проверьте версии:
   ```bash
   tar --version | head -1
   gzip --version | head -1
   bzip2 --help 2>&1 | head -1
   xz --version | head -1
   zip -v | head -2
   ```

5. Создайте рабочий каталог:
   ```bash
   mkdir -p /root/arch-lab && cd /root/arch-lab
   ```

6. Подготовьте тестовый набор файлов:
   ```bash
   mkdir -p /root/arch-lab/data/sub1 /root/arch-lab/data/sub2
   echo "Первый файл" > /root/arch-lab/data/file1.txt
   echo "Второй файл" > /root/arch-lab/data/file2.txt
   for i in $(seq 1 2000); do echo "Строка номер $i с некоторым текстом для сжатия"; done > /root/arch-lab/data/big1.txt
   for i in $(seq 1 2000); do echo "Строка номер $i с некоторым текстом для сжатия"; done > /root/arch-lab/data/big2.txt
   dd if=/dev/urandom of=/root/arch-lab/data/random.bin bs=1M count=5 status=none
   head -c 500000 /dev/zero > /root/arch-lab/data/zeros.bin
   echo "Данные в sub1" > /root/arch-lab/data/sub1/inner1.txt
   echo "Данные в sub2" > /root/arch-lab/data/sub2/inner2.txt
   ln -s file1.txt /root/arch-lab/data/link_to_file1
   chmod 600 /root/arch-lab/data/file1.txt
   ```

7. Проверьте структуру:
   ```bash
   ls -lR /root/arch-lab/data
   du -sh /root/arch-lab/data
   du -sh --apparent-size /root/arch-lab/data
   ```

---

## Этап 2. Утилита `gzip` (15 мин)

### 2.1. Сжатие одного файла

1. Скопируйте файл и сожмите:
   ```bash
   cd /root/arch-lab
   cp data/big1.txt big1.txt
   ls -lh big1.txt
   gzip big1.txt
   ls -lh big1.txt.gz
   ```
   **Важно:** `gzip` **удаляет** исходный файл, заменяя его `.gz`.

2. Сжатие с сохранением оригинала (`-k`):
   ```bash
   cp data/big2.txt big2.txt
   gzip -k big2.txt
   ls -lh big2.txt big2.txt.gz
   ```

3. Просмотр информации:
   ```bash
   gzip -l big1.txt.gz big2.txt.gz
   ```

4. Проверить целостность:
   ```bash
   gzip -t big1.txt.gz && echo "OK"
   ```

### 2.2. Уровни сжатия

1. Создайте файлы одинакового размера с разным уровнем:
   ```bash
   cp data/big1.txt lvl1.txt
   cp data/big1.txt lvl9.txt
   gzip -1 -k lvl1.txt
   gzip -9 -k lvl9.txt
   ls -lh lvl1.txt.gz lvl9.txt.gz
   ```
   Уровень 9 — максимальное сжатие, но медленнее. Уровень 1 — быстро, но меньше сжатие. По умолчанию — 6.

2. Сравните время выполнения на большом файле:
   ```bash
   cp data/random.bin r1.bin
   cp data/random.bin r9.bin
   time gzip -1 -k r1.bin
   time gzip -9 -k r9.bin
   ls -lh r1.bin.gz r9.bin.gz
   ```
   **Наблюдение:** случайные данные почти не сжимаются — это нормально.

### 2.3. Распаковка

1. Распакуйте `big1.txt.gz`:
   ```bash
   gunzip big1.txt.gz
   ls -lh big1.txt
   ```
   Эквивалент: `gzip -d big1.txt.gz`.

2. Распакуйте с сохранением архива:
   ```bash
   gunzip -k big2.txt.gz
   ```

3. Вывод в stdout:
   ```bash
   zcat big1.txt.gz | head -3
   gzip -dc big1.txt.gz | tail -3
   ```

### 2.4. Управление именем и потоковая обработка

1. `gzip -c` — вывод в stdout, не удаляя исходник:
   ```bash
   gzip -c big1.txt > mybig.gz
   ls -lh mybig.gz
   ```

2. Перенаправление через pipe:
   ```bash
   cat big1.txt | gzip > piped.gz
   zcat piped.gz | wc -l
   ```

---

## Этап 3. Утилита `bzip2` (10 мин)

1. Сожмите файл:
   ```bash
   cd /root/arch-lab
   cp data/big1.txt big1_bz.txt
   ls -lh big1_bz.txt
   bzip2 big1_bz.txt
   ls -lh big1_bz.txt.bz2
   ```
   `bzip2` тоже удаляет исходник.

2. Уровни сжатия:
   ```bash
   cp data/big1.txt bz1.txt
   cp data/big1.txt bz9.txt
   bzip2 -1 -k bz1.txt
   bzip2 -9 -k bz9.txt
   ls -lh bz1.txt.bz2 bz9.txt.bz2
   ```

3. Распаковка:
   ```bash
   bunzip2 big1_bz.txt.bz2
   ls -lh big1_bz.txt
   ```

4. Вывод в stdout:
   ```bash
   bzcat bz1.txt.bz2 | head -2
   bunzip2 -c bz9.txt.bz2 | tail -2
   ```

5. Целостность:
   ```bash
   bzip2 -t bz1.txt.bz2 && echo "OK"
   ```

6. Проверьте информацию:
   ```bash
   bzip2 -v big1_bz.txt.bz2   # если файла нет — используйте bz1.txt.bz2
   ```

---

## Этап 4. Утилита `xz` (15 мин)

1. Сжатие:
   ```bash
   cd /root/arch-lab
   cp data/big1.txt big1_xz.txt
   xz big1_xz.txt
   ls -lh big1_xz.txt.xz
   ```

2. Уровни сжатия `-0` … `-9`, `-e` (extreme):
   ```bash
   cp data/big1.txt xz0.txt
   cp data/big1.txt xz6.txt
   cp data/big1.txt xz9e.txt
   time xz -0 -k xz0.txt
   time xz -6 -k xz6.txt
   time xz -9e -k xz9e.txt
   ls -lh xz0.txt.xz xz6.txt.xz xz9e.txt.xz
   ```

3. Распаковка:
   ```bash
   unxz big1_xz.txt.xz
   ls -lh big1_xz.txt
   ```

4. Вывод в stdout:
   ```bash
   xzcat xz0.txt.xz | head -2
   unxz -c xz6.txt.xz | tail -2
   ```

5. Целостность:
   ```bash
   xz -t xz0.txt.xz && echo "OK"
   ```

6. Параллельное сжатие (`-T`) — использует несколько ядер:
   ```bash
   cp data/big1.txt par.txt
   time xz -T0 -k par.txt
   ls -lh par.txt.xz
   ```
   `-T0` = использовать все доступные ядра.

---

## Этап 5. Архиватор `tar` (25 мин)

### 5.1. Создание архива без сжатия

1. Перейдите в каталог с данными:
   ```bash
   cd /root/arch-lab
   tar -cvf data.tar data/
   ls -lh data.tar
   ```

2. Просмотр содержимого:
   ```bash
   tar -tvf data.tar
   ```

3. Обратите внимание на симлинк `link_to_file1`:
   ```bash
   tar -tvf data.tar | grep link
   ```

### 5.2. Создание архива с разными компрессорами

```bash
tar -czvf data.tar.gz data/
tar -cjvf data.tar.bz2 data/
tar -cJvf data.tar.xz data/
tar -c --zstd -vf data.tar.zst data/
ls -lh data.tar*
```

Сравните размеры:
```bash
ls -lh data.tar data.tar.gz data.tar.bz2 data.tar.xz data.tar.zst
```

### 5.3. Автоматическое определение компрессора

`tar` сам определяет метод по расширению при распаковке:
```bash
tar -xvf data.tar.gz -C /tmp/extract_auto
```
Создайте каталог заранее:
```bash
mkdir -p /tmp/extract_auto
tar -xvf data.tar.gz -C /tmp/extract_auto
ls -R /tmp/extract_auto
```

### 5.4. Распаковка разных форматов

```bash
mkdir -p /tmp/e1 /tmp/e2 /tmp/e3 /tmp/e4
tar -xzvf data.tar.gz  -C /tmp/e1
tar -xjvf data.tar.bz2 -C /tmp/e2
tar -xJvf data.tar.xz  -C /tmp/e3
tar -x --zstd -vf data.tar.zst -C /tmp/e4
```

### 5.5. Просмотр без распаковки

```bash
tar -tf  data.tar.xz | head
tar -tvf data.tar.xz | head
tar -tf  data.tar.gz | grep '\.txt$'
```

### 5.6. Сохранение прав и владельцев

1. Проверьте права внутри архива:
   ```bash
   tar -tvf data.tar | grep file1.txt
   ```
   Файл `file1.txt` имеет права `600`. Права сохраняются автоматически.

2. Владельцы сохраняются только при распаковке от `root`:
   ```bash
   # от root владельцы сохраняются автоматически
   # для числовых UID/GID используйте --numeric-owner
   tar --numeric-owner -cvf data_num.tar data/
   ```

3. Для пользователя без root:
   ```bash
   su - user -c 'tar -xvf /root/arch-lab/data.tar -C /tmp 2>&1 | head'
   ```
   Владельцы будут изменены на `user` (если нет соответствующих привилегий).

### 5.7. Исключение файлов

```bash
tar -czvf no_random.tar.gz --exclude='*.bin' --exclude='sub2' data/
tar -tzf no_random.tar.gz
```

### 5.8. Добавление файлов (только для несжатых)

```bash
cp data/file1.txt extra.txt
tar -rvf data.tar extra.txt
tar -tf data.tar | tail -3
```

### 5.9. Обновление в архиве (`-u`)

```bash
# Изменим файл и обновим
echo "Изменение" >> data/file1.txt
tar -uvf data.tar data/file1.txt
tar -tvf data.tar | grep file1.txt
```

### 5.10. Архивация по списку

1. Создайте файл со списком:
   ```bash
   cat > list.txt <<'EOF'
   data/file1.txt
   data/file2.txt
   data/sub1
   EOF
   ```

2. Архивируйте по списку:
   ```bash
   tar -czvf list.tar.gz -T list.txt
   tar -tzf list.tar.gz
   ```

### 5.11. Архив из текущего каталога без путей

```bash
cd /root/arch-lab/data
tar -czvf /root/arch-lab/data_clean.tar.gz .
cd /root/arch-lab
tar -tzf data_clean.tar.gz | head
```

Без точки-пути распакуется в текущий каталог.

---

## Этап 6. `zip` и `unzip` (10 мин)

1. Создайте zip-архив:
   ```bash
   cd /root/arch-lab
   zip -r data.zip data/
   ls -lh data.zip
   ```

2. Просмотр содержимого:
   ```bash
   unzip -l data.zip
   ```

3. Распаковка:
   ```bash
   mkdir -p /tmp/e_zip
   unzip data.zip -d /tmp/e_zip
   ls -R /tmp/e_zip | head
   ```

4. Пароль:
   ```bash
   zip -r -e secret.zip data/file1.txt data/file2.txt
   ```
   Введите пароль `111` дважды. Затем:
   ```bash
   unzip -P 111 secret.zip -d /tmp/e_zip_sec
   ls /tmp/e_zip_sec
   ```

5. Сравнение размера `zip` с `tar.gz`:
   ```bash
   ls -lh data.zip data.tar.gz
   ```

6. Распаковка одного файла:
   ```bash
   unzip data.zip data/file1.txt -d /tmp/one
   ls /tmp/one/data
   ```

---

## Этап 7. Сравнение компрессоров (10 мин)

1. Создайте «реалистичный» файл — например, лог:
   ```bash
   for i in $(seq 1 20000); do
       echo "$(date +%T) INFO Some log line number $i"
   done > logfile.txt
   ls -lh logfile.txt
   ```

2. Сожмите разными утилитами и измерьте время:
   ```bash
   time gzip -k logfile.txt        && mv logfile.txt.gz log_gz
   time bzip2 -k logfile.txt       && mv logfile.txt.bz2 log_bz
   time xz -k logfile.txt          && mv logfile.txt.xz log_xz
   time zstd -k logfile.txt        && mv logfile.txt.zst log_zst
   ```

3. Сравните размеры:
   ```bash
   ls -lh logfile.txt log_gz log_bz log_xz log_zst
   ```

4. Результат оформите таблицей:

   | Метод | Размер | Время | Степень сжатия |
   |---|---|---|---|
   | Без сжатия | … | — | 1× |
   | gzip | … | … | … |
   | bzip2 | … | … | … |
   | xz | … | … | … |
   | zstd | … | … | … |

   Степень сжатия = размер оригинала / размер сжатого файла.

---

## Этап 8. Разделение и сборка больших архивов (10 мин)

### 8.1. Разделение через `split`

1. Создайте большой архив:
   ```bash
   cd /root/arch-lab
   tar -czf big_archive.tar.gz data/
   ls -lh big_archive.tar.gz
   ```

2. Разделите на части по 1 МБ:
   ```bash
   split -b 1M big_archive.tar.gz part_
   ls -lh part_*
   ```

3. Сборка обратно:
   ```bash
   cat part_* > big_archive_rebuilt.tar.gz
   ls -lh big_archive.tar.gz big_archive_rebuilt.tar.gz
   ```

4. Проверьте целостность:
   ```bash
   gzip -t big_archive_rebuilt.tar.gz && echo "OK"
   ```

5. Распакуйте:
   ```bash
   mkdir -p /tmp/e_split
   tar -xzf big_archive_rebuilt.tar.gz -C /tmp/e_split
   diff -r data /tmp/e_split/data && echo "Содержимое совпадает"
   ```

### 8.2. Многоточечная архивация (`-M`)

`tar` умеет писать на несколько носителей, но требует физических устройств. В виртуальной среде проще использовать `split`.

### 8.3. Многотомные архивы `zip`

```bash
zip -r -s 1m multi.zip data/
ls -lh multi.z*
```
Сборка:
```bash
zip -s 0 multi.zip --out single.zip
unzip -l single.zip | head
```

---

## Этап 9. Практические сценарии (5 мин)

### 9.1. Резервная копия `/etc`

```bash
tar -czf /root/arch-lab/etc_backup_$(date +%F).tar.gz /etc
ls -lh /root/arch-lab/etc_backup_*.tar.gz
```

### 9.2. Резервное копирование с исключениями

```bash
tar -czf home_backup.tar.gz \
    --exclude='*.cache' \
    --exclude='.local/share/Trash' \
    /home/user
ls -lh home_backup.tar.gz
```

### 9.3. Инкрементальный бэкап (упрощённо через снимок mtime)

```bash
# Создаём маркер
touch /root/arch-lab/last_backup.marker
sleep 1
# Создаём новые файлы
echo "new" > /root/arch-lab/data/newfile.txt
# Архивируем только новые
tar -czf incremental.tar.gz --newer=/root/arch-lab/last_backup.marker data/
tar -tzf incremental.tar.gz
```
Обратите внимание: попадёт только `newfile.txt`.

### 9.4. Восстановление одного файла из архива

```bash
tar -xzf etc_backup_*.tar.gz -C /tmp/restore etc/hostname
cat /tmp/restore/etc/hostname
```

### 9.5. Проверка архива без извлечения

```bash
tar -tzf home_backup.tar.gz >/dev/null && echo "Архив валиден"
```

---

# Лабораторная работа: Управление дисками, LVM и монтирование в Debian 12

**Тема:** Создание разделов, форматирование, изменение размеров разделов, LVM, ремонт повреждённого диска в LVM, монтирование через `fstab`, `systemd mount` и `automount`, управление файлами на диске.

**Продолжительность:** 2 академических часа (120 минут).

**Оборудование:**
- Виртуальная машина с Debian 12.
- Дополнительные виртуальные диски: `/dev/sdb`, `/dev/sdc`, `/dev/sdd`, `/dev/sde`, `/dev/sdf`, `/dev/sdg` (по 1 ГБ).
- Пользователь `root` (пароль `111`), пользователь `user` (пароль `111`).

> **Важно:** Из работы исключены RAID и утилита `mdadm`. Добавлен этап ремонта повреждённого диска в LVM (миграция данных и замена PV), а также отработка трёх способов монтирования.

---

## Этап 1. Подготовка (10 мин)

1. Войдите в систему как `root`:
   ```bash
   su -
   # пароль: 111
   ```

2. Проверьте наличие пользователя `user`:
   ```bash
   id user
   ```
   Если пользователя нет, создайте его:
   ```bash
   useradd -m -s /bin/bash user
   echo 'user:111' | chpasswd
   ```

3. Посмотрите список дисков:
   ```bash
   lsblk
   fdisk -l
   ```
   Убедитесь, что видны диски `/dev/sdb` … `/dev/sdg`.

4. Установите необходимые пакеты:
   ```bash
   apt update
   apt install -y lvm2 xfsprogs parted
   ```

---

## Этап 2. Создание разделов, форматирование, монтирование (15 мин)

1. Создайте основной раздел на диске `/dev/sdb`:
   ```bash
   fdisk /dev/sdb
   ```
   В интерактивном режиме:
   - `n` → `p` → `1` → Enter → Enter → `w`

2. Отформатируйте раздел в ext4:
   ```bash
   mkfs.ext4 /dev/sdb1
   ```

3. Смонтируйте раздел:
   ```bash
   mkdir -p /mnt/test
   mount /dev/sdb1 /mnt/test
   df -h /mnt/test
   ```

4. Посмотрите UUID раздела:
   ```bash
   blkid /dev/sdb1
   ```

---

## Этап 3. Изменение размеров разделов (15 мин)

### 3.1. Увеличение раздела

1. Создайте раздел на `/dev/sdc` размером 500 МБ:
   ```bash
   parted /dev/sdc mklabel msdos
   parted /dev/sdc mkpart primary ext4 1MiB 500MiB
   mkfs.ext4 /dev/sdc1
   mkdir -p /mnt/test2
   mount /dev/sdc1 /mnt/test2
   df -h /mnt/test2
   umount /mnt/test2
   ```

2. Увеличьте раздел до конца диска:
   ```bash
   parted /dev/sdc resizepart 1 100%
   e2fsck -f /dev/sdc1
   resize2fs /dev/sdc1
   mount /dev/sdc1 /mnt/test2
   df -h /mnt/test2
   ```

### 3.2. Уменьшение раздела

1. Размонтируйте и проверьте:
   ```bash
   umount /mnt/test2
   e2fsck -f /dev/sdc1
   ```

2. Уменьшите ФС до 280 МБ, затем раздел до 300 МБ:
   ```bash
   resize2fs /dev/sdc1 280M
   parted /dev/sdc resizepart 1 300M
   resize2fs /dev/sdc1
   mount /dev/sdc1 /mnt/test2
   df -h /mnt/test2
   ```

> **Примечание:** XFS нельзя уменьшать. Для уменьшения используйте ext4.

---

## Этап 4. LVM: создание, увеличение, уменьшение (30 мин)

### 4.1. Создание PV, VG, LV

1. Создайте физические тома:
   ```bash
   pvcreate /dev/sdd /dev/sde
   pvs
   ```

2. Создайте группу томов `vg_data`:
   ```bash
   vgcreate vg_data /dev/sdd /dev/sde
   vgs
   ```

3. Создайте логический том `lv_data` размером 500 МБ:
   ```bash
   lvcreate -L 500M -n lv_data vg_data
   lvs
   ```

4. Отформатируйте и смонтируйте:
   ```bash
   mkfs.ext4 /dev/vg_data/lv_data
   mkdir -p /mnt/lvm
   mount /dev/vg_data/lv_data /mnt/lvm
   df -h /mnt/lvm
   ```

### 4.2. Увеличение LV

1. Добавьте 200 МБ:
   ```bash
   lvextend -L +200M /dev/vg_data/lv_data
   resize2fs /dev/vg_data/lv_data
   df -h /mnt/lvm
   ```

### 4.3. Уменьшение LV

1. Размонтируйте и проверьте ФС:
   ```bash
   umount /mnt/lvm
   e2fsck -f /dev/vg_data/lv_data
   ```

2. Уменьшите ФС и LV до 300 МБ:
   ```bash
   resize2fs /dev/vg_data/lv_data 300M
   lvreduce -L 300M /dev/vg_data/lv_data
   mount /dev/vg_data/lv_data /mnt/lvm
   df -h /mnt/lvm
   ```

### 4.4. Расширение группы томов

1. Добавьте новый диск `/dev/sdf` в LVM:
   ```bash
   pvcreate /dev/sdf
   vgextend vg_data /dev/sdf
   vgs
   ```

2. Создайте дополнительные логические тома:
   ```bash
   lvcreate -L 200M -n lv_extra vg_data
   lvcreate -L 200M -n lv_auto vg_data
   mkfs.ext4 /dev/vg_data/lv_extra
   mkfs.ext4 /dev/vg_data/lv_auto
   ```

---

## Этап 5. Ремонт повреждённого диска в LVM (15 мин)

**Ситуация:** один из физических томов LVM вышел из строя (например, диск физически повреждён или отключён). Данные нужно перенести на новый диск и заменить вышедший из VG.

### 5.1. Подготовка: имитация выхода диска из строя

1. Посмотрите, какие диски входят в VG:
   ```bash
   pvs
   vgdisplay vg_data
   ```

2. Проверьте, есть ли на каком-либо PV свободное место для миграции:
   ```bash
   pvdisplay
   ```
   Если свободного места нет, добавьте новый диск в VG (см. п. 4.4) или создайте новый VG с двумя-тремя дисками специально для этого упражнения.

3. Сымитируйте поломку диска `/dev/sdd` — «отключите» его от VG:
   ```bash
   vgreduce --removemissing --force vg_data
   ```
   (В реальной ситуации этот шаг выполняется, когда диск уже физически недоступен.)

### 5.2. Перенос данных перед заменой диска (если диск ещё доступен)

Если диск ещё читается, но выходит из строя, данные можно заранее перенести:

1. Переместите все данные с `/dev/sdd` на другой PV группы:
   ```bash
   pvmove /dev/sdd
   ```
   Команда перенесёт все логические экстенты с указанного PV на другие PV в группе.

2. Убедитесь, что на `/dev/sdd` больше нет данных:
   ```bash
   pvs -o+pv_used
   ```
   Столбец `PFree` должен равняться `PSize`.

3. Удалите PV из группы:
   ```bash
   vgreduce vg_data /dev/sdd
   ```

4. Удалите метку PV:
   ```bash
   pvremove /dev/sdd
   ```

### 5.3. Замена диска и восстановление VG

1. Подготовьте новый диск (например, `/dev/sdg`) — разметьте как PV:
   ```bash
   pvcreate /dev/sdg
   ```

2. Добавьте его в группу:
   ```bash
   vgextend vg_data /dev/sdg
   vgs
   ```

3. Проверьте, что все LV по-прежнему доступны:
   ```bash
   lvs
   mount /dev/vg_data/lv_data /mnt/lvm
   df -h /mnt/lvm
   ```

### 5.4. Аварийное восстановление при недоступном диске

Если диск физически недоступен и данных на нём не жалко (или есть резервная копия), используйте:

```bash
vgreduce --removemissing vg_data
```

Флаг `--removemissing` удаляет из VG все PV, которых больше нет в системе, и «выбрасывает» относящиеся к ним LV (если они не зеркалируются).

Для проверки целостности метаданных VG:
```bash
vgck vg_data
```

Для восстановления (при необходимости):
```bash
vgcfgrestore vg_data
```

> **Вывод:** LVM сам по себе не обеспечивает отказоустойчивость как RAID — он лишь упрощает замену и перенос данных (`pvmove`, `vgreduce`, `vgextend`). Для настоящей отказоустойчивости нужен RAID или LVM-RAID (`lvcreate --type raid1`), что выходит за рамки этой работы.

---

## Этап 6. Монтирование: fstab, systemd mount, automount (20 мин)

### 6.1. Монтирование через `/etc/fstab`

1. Добавьте строку в `/etc/fstab`:
   ```bash
   nano /etc/fstab
   ```
   Строка:
   ```
   /dev/vg_data/lv_data /mnt/lvm ext4 defaults 0 2
   ```

2. Проверьте:
   ```bash
   umount /mnt/lvm
   mount -a
   df -h /mnt/lvm
   ```

### 6.2. Монтирование через systemd `.mount`

1. Создайте каталог и юнит:
   ```bash
   mkdir -p /mnt/extra
   nano /etc/systemd/system/mnt-extra.mount
   ```
   Содержимое:
   ```ini
   [Unit]
   Description=Mount extra LV

   [Mount]
   What=/dev/vg_data/lv_extra
   Where=/mnt/extra
   Type=ext4
   Options=defaults

   [Install]
   WantedBy=multi-user.target
   ```

2. Активируйте:
   ```bash
   systemctl daemon-reload
   systemctl enable --now mnt-extra.mount
   systemctl status mnt-extra.mount
   ```

### 6.3. Автомонтирование через systemd `.automount`

1. Создайте каталог и два юнита:
   ```bash
   mkdir -p /mnt/auto
   nano /etc/systemd/system/mnt-auto.mount
   ```
   ```ini
   [Unit]
   Description=Mount auto LV

   [Mount]
   What=/dev/vg_data/lv_auto
   Where=/mnt/auto
   Type=ext4
   Options=defaults
   ```
   ```bash
   nano /etc/systemd/system/mnt-auto.automount
   ```
   ```ini
   [Unit]
   Description=Automount auto LV

   [Automount]
   Where=/mnt/auto

   [Install]
   WantedBy=multi-user.target
   ```

2. Включите только `.automount`:
   ```bash
   systemctl daemon-reload
   systemctl enable --now mnt-auto.automount
   ```

3. Проверьте срабатывание:
   ```bash
   ls /mnt/auto
   findmnt /mnt/auto
   ```

> **Важно:** Не включайте одновременно `mnt-auto.mount` и `mnt-auto.automount`.

---

## Этап 7. Управление файлами на диске (5 мин)

1. Создайте каталог и файл:
   ```bash
   mkdir /mnt/lvm/testdir
   echo "Hello, Linux!" > /mnt/lvm/testdir/file.txt
   chmod 750 /mnt/lvm/testdir
   chown user:user /mnt/lvm/testdir/file.txt
   ls -l /mnt/lvm/testdir
   ```

2. Создайте жёсткую и символическую ссылки:
   ```bash
   ln /mnt/lvm/testdir/file.txt /mnt/lvm/testdir/hardlink
   ln -s /mnt/lvm/testdir/file.txt /mnt/lvm/testdir/symlink
   ls -li /mnt/lvm/testdir
   ```

---

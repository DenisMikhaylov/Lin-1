# Лабораторная работа: Управление дисками, LVM и монтирование в Debian 12

**Тема:** Создание разделов, форматирование, изменение размеров разделов, LVM, монтирование через `fstab`, `systemd mount` и `automount`, управление файлами на диске.

**Продолжительность:** 2 академических часа (120 минут).

**Оборудование:**
- Виртуальная машина с Debian 12.
- Дополнительные виртуальные диски: `/dev/sdb`, `/dev/sdc`, `/dev/sdd`, `/dev/sde`, `/dev/sdf` (по 1 ГБ).
- Пользователь `root` (пароль `111`), пользователь `user` (пароль `111`).

> **Важно:** Из работы исключены RAID и утилита `mdadm`. Для сохранения двухчасовой нагрузки добавлены дополнительные упражнения по LVM: расширение группы томов, создание нескольких логических томов и отработка трёх способов монтирования.

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
   Убедитесь, что видны диски `/dev/sdb` … `/dev/sdf`.

4. Установите необходимые пакеты:
   ```bash
   apt update
   apt install -y lvm2 xfsprogs parted
   ```

---

## Этап 2. Создание разделов, форматирование, монтирование (20 мин)

1. Создайте основной раздел на диске `/dev/sdb`:
   ```bash
   fdisk /dev/sdb
   ```
   В интерактивном режиме:
   - `n` — новый раздел
   - `p` — основной
   - `1` — номер раздела
   - Enter — первый сектор по умолчанию
   - Enter — последний сектор (весь диск)
   - `w` — записать изменения

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

## Этап 3. Изменение размеров разделов (20 мин)

### 3.1. Увеличение раздела

1. Создайте раздел на `/dev/sdc` размером 500 МБ:
   ```bash
   parted /dev/sdc mklabel msdos
   parted /dev/sdc mkpart primary ext4 1MiB 500MiB
   ```

2. Отформатируйте и смонтируйте:
   ```bash
   mkfs.ext4 /dev/sdc1
   mkdir -p /mnt/test2
   mount /dev/sdc1 /mnt/test2
   df -h /mnt/test2
   ```

3. Размонтируйте раздел:
   ```bash
   umount /mnt/test2
   ```

4. Увеличьте раздел до конца диска:
   ```bash
   parted /dev/sdc resizepart 1 100%
   ```

5. Проверьте и увеличьте файловую систему:
   ```bash
   e2fsck -f /dev/sdc1
   resize2fs /dev/sdc1
   ```

6. Смонтируйте обратно и проверьте размер:
   ```bash
   mount /dev/sdc1 /mnt/test2
   df -h /mnt/test2
   ```

### 3.2. Уменьшение раздела

1. Размонтируйте раздел:
   ```bash
   umount /mnt/test2
   ```

2. Проверьте файловую систему:
   ```bash
   e2fsck -f /dev/sdc1
   ```

3. Уменьшите файловую систему до 280 МБ:
   ```bash
   resize2fs /dev/sdc1 280M
   ```

4. Уменьшите раздел до 300 МБ:
   ```bash
   parted /dev/sdc resizepart 1 300M
   ```

5. Расширьте файловую систему до размера раздела:
   ```bash
   resize2fs /dev/sdc1
   ```

6. Смонтируйте и проверьте:
   ```bash
   mount /dev/sdc1 /mnt/test2
   df -h /mnt/test2
   ```

> **Примечание:** XFS нельзя уменьшать. Для уменьшения используйте ext4.

---

## Этап 4. LVM: создание, увеличение, уменьшение (35 мин)

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
   ```

2. Расширьте файловую систему:
   ```bash
   resize2fs /dev/vg_data/lv_data
   df -h /mnt/lvm
   ```

### 4.3. Уменьшение LV

1. Размонтируйте том:
   ```bash
   umount /mnt/lvm
   ```

2. Проверьте файловую систему:
   ```bash
   e2fsck -f /dev/vg_data/lv_data
   ```

3. Уменьшите файловую систему до 300 МБ:
   ```bash
   resize2fs /dev/vg_data/lv_data 300M
   ```

4. Уменьшите логический том:
   ```bash
   lvreduce -L 300M /dev/vg_data/lv_data
   ```

5. Смонтируйте обратно:
   ```bash
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
   lvs
   ```

---

## Этап 5. Монтирование: fstab, systemd mount, automount (25 мин)

### 5.1. Монтирование через `/etc/fstab`

1. Откройте файл:
   ```bash
   nano /etc/fstab
   ```

2. Добавьте строку:
   ```
   /dev/vg_data/lv_data /mnt/lvm ext4 defaults 0 2
   ```

3. Проверьте:
   ```bash
   umount /mnt/lvm
   mount -a
   df -h /mnt/lvm
   ```

### 5.2. Монтирование через systemd `.mount`

1. Создайте каталог:
   ```bash
   mkdir -p /mnt/extra
   ```

2. Создайте файл `/etc/systemd/system/mnt-extra.mount`:
   ```bash
   nano /etc/systemd/system/mnt-extra.mount
   ```

3. Содержимое:
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

4. Активируйте юнит:
   ```bash
   systemctl daemon-reload
   systemctl enable --now mnt-extra.mount
   systemctl status mnt-extra.mount
   df -h /mnt/extra
   ```

### 5.3. Автомонтирование через systemd `.automount`

1. Создайте каталог:
   ```bash
   mkdir -p /mnt/auto
   ```

2. Создайте файл `/etc/systemd/system/mnt-auto.mount`:
   ```bash
   nano /etc/systemd/system/mnt-auto.mount
   ```
   Содержимое:
   ```ini
   [Unit]
   Description=Mount auto LV

   [Mount]
   What=/dev/vg_data/lv_auto
   Where=/mnt/auto
   Type=ext4
   Options=defaults
   ```

3. Создайте файл `/etc/systemd/system/mnt-auto.automount`:
   ```bash
   nano /etc/systemd/system/mnt-auto.automount
   ```
   Содержимое:
   ```ini
   [Unit]
   Description=Automount auto LV

   [Automount]
   Where=/mnt/auto

   [Install]
   WantedBy=multi-user.target
   ```

4. Включите и запустите автомонтирование:
   ```bash
   systemctl daemon-reload
   systemctl enable --now mnt-auto.automount
   systemctl status mnt-auto.automount
   ```

5. Проверьте: при обращении к каталогу том смонтируется автоматически:
   ```bash
   ls /mnt/auto
   findmnt /mnt/auto
   ```

> **Важно:** Не включайте одновременно `mnt-auto.mount` и `mnt-auto.automount`. Для автомонтирования достаточно включить только `.automount`.

---

## Этап 6. Управление файлами на диске (10 мин)

1. Создайте каталог и файл на смонтированном томе `/mnt/lvm`:
   ```bash
   mkdir /mnt/lvm/testdir
   echo "Hello, Linux!" > /mnt/lvm/testdir/file.txt
   ```

2. Измените права доступа:
   ```bash
   chmod 750 /mnt/lvm/testdir
   chown user:user /mnt/lvm/testdir/file.txt
   ls -l /mnt/lvm/testdir
   ```

3. Создайте жёсткую и символическую ссылки:
   ```bash
   ln /mnt/lvm/testdir/file.txt /mnt/lvm/testdir/hardlink
   ln -s /mnt/lvm/testdir/file.txt /mnt/lvm/testdir/symlink
   ls -li /mnt/lvm/testdir
   ```

4. Проверьте содержимое:
   ```bash
   cat /mnt/lvm/testdir/hardlink
   cat /mnt/lvm/testdir/symlink
   ```

---


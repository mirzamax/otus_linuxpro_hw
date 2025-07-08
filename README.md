## Создание RAID5 с разбивкой на разделы и монтированием

### 1. Сброс суперблоков RAID

```bash
sudo mdadm --zero-superblock --force /dev/vd{b,c,d,e,f}
```

Вывод:

```
mdadm: Unrecognised md component device - /dev/vdb
mdadm: Unrecognised md component device - /dev/vdc
mdadm: Unrecognised md component device - /dev/vdd
mdadm: Unrecognised md component device - /dev/vde
mdadm: Couldn't open /dev/vdf for write - not zeroing
```

### 2. Создаем RAID5

```bash
sudo mdadm --create --verbose /dev/md0 -l 5 -n 3 /dev/vdb /dev/vdc /dev/vdd
```

Вывод:

```
mdadm: layout defaults to left-symmetric
mdadm: chunk size defaults to 512K
mdadm: size set to 1046528K
mdadm: Defaulting to version 1.2 metadata
mdadm: array /dev/md0 started.
```

### 3. Проверка состояния RAID

```bash
cat /proc/mdstat
```

Пример вывода:

```
md0 : active raid5 vdd[3] vdc[1] vdb[0]
      2093056 blocks super 1.2 level 5, 512k chunk, algorithm 2 [3/2] [UU_]
      [=============>.......]  recovery = 65.0% (681316/1046528) finish=0.3min speed=15331K/sec
```

### 4. Форсируем ошибку и удаляем диск

```bash
sudo mdadm /dev/md0 --fail /dev/vdd
sudo mdadm /dev/md0 --remove /dev/vdd
```

Вывод:

```
mdadm: set /dev/vdd faulty in /dev/md0
mdadm: hot removed /dev/vdd from /dev/md0
```

### 5. Добавляем заменный диск

```bash
sudo mdadm /dev/md0 --add /dev/vde
```

Вывод:

```
mdadm: added /dev/vde
```

### 6. Проверка состояния RAID после добавления

```bash
sudo mdadm -D /dev/md0
```

Вывод:

```
State : clean, degraded, recovering
Active Devices : 2
Working Devices : 3
Spare Devices : 1
Rebuild Status : 13% complete
```

### 7. Создаем GPT таблицу разделов

```bash
sudo parted -s /dev/md0 mklabel gpt
```

### 8. Разбиваем на 5 разделов

```bash
sudo parted /dev/md0 mkpart primary ext4 0% 20%
sudo parted /dev/md0 mkpart primary ext4 20% 40%
sudo parted /dev/md0 mkpart primary ext4 40% 60%
sudo parted /dev/md0 mkpart primary ext4 60% 80%
sudo parted /dev/md0 mkpart primary ext4 80% 100%
```

Вывод (каждая команда):

```
Information: You may need to update /etc/fstab.
```

### 9. Создаем файловую систему ext4 на каждом разделе

```bash
for i in $(seq 1 5); do sudo mkfs.ext4 /dev/md0p$i; done
```

Вывод (сокращен до одного):

```
Creating filesystem with 104448 4k blocks and 104448 inodes
Filesystem UUID: ...
Creating journal (4096 blocks): done
Writing superblocks and filesystem accounting information: done
```

### 10. Создаем точки монтирования

```bash
sudo mkdir -p /raid/part{1,2,3,4,5}
```

### 11. Монтируем разделы

```bash
for i in $(seq 1 5); do sudo mount /dev/md0p$i /raid/part$i; done
```

### 12. Проверка

```bash
lsblk
```

Вывод:

```
md0p1 ... /raid/part1
md0p2 ... /raid/part2
...
md0p5 ... /raid/part5
```

---

> Готово: RAID5 с разделами и монтированием по каталогам.

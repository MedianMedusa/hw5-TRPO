# ДЗ 5 по технологиям разработки ПО
## Задание
 Надо было по мануалу побаловаться с PV и LV. ТЗ:
<details>
  <summary>Посмотреть ТЗ</summary>  
Нужна система linux с кучей дисков/разделов.
*Можно попробовать на loop, но их нужно исключить из фильтра в файле конфигурации lvm.

Практика с lvm:
Смотрим текущее состояние:
lsblk
lvmdiskscan

Добавляем диск как PV:
pvcreate /dev/sdb
pvdisplay
lvmdiskscan
pvs

Создаём VG на базе PV
vgcreate mai /dev/sdb
vgdisplay -v mai
vgs

Создаём LV на базе VG
lvcreate -l+100%FREE -n first mai
lvdisplay
lvs

Создаём файловую систему, монтируем её и проверяем
mkfs.ext4 /dev/mai/first
mount /dev/mai/first /mnt 
mount

Создаём файл на весь размер точки монтирования
dd if=/dev/zero of=/mnt/test.file bs=1M count=8000 status=progress
df -h

Расширяем LV за счёт нового PV в VG
pvcreate /dev/sdc 
vgextend mai /dev/sdc 
lvextend -l+100%FREE /dev/mai/first
lvdisplay
lvs
df -h

Чуда не произошло, поэтому расширяем файловую систему
resize2fs /dev/mai/first
df -h

Уменьшаем файловую систему и LV
umount /mnt
e2fsck -fy /dev/mai/first
resize2fs /dev/mai/first 1100M
lvreduce /dev/mai/first -L 1100M
e2fsck -fy /dev/mai/first
mount /dev/mai/first /mnt
df -h

Создаём несколько файлов и делаем снимок
touch /mnt/file{1..5}
lvcreate -L 100M -s -n snapsh /dev/mai/first
lvs
lsblk

Удаляем несколько файлов
rm -f /mnt/file{1..3}
ls /mnt

Монтируем снимок и проверяем, что файлы там есть. Отмонтируем.
mkdir /snap
mount /dev/mai/snapsh /snap
ls /snap
umount /snap

Отмонтируем файловую систему и производим слияние. Проверяем, что файлы на месте.
umount /mnt
lvconvert --merge /dev/mai/snapsh
mount /dev/mai/first /mnt
ls /mnt

Добавляем ещё PV, VG и создаём LV-зеркало.
pvcreate /dev/sd{d,e}
vgcreate vgmirror /dev/sd{d,e}
lvcreate -l+80%FREE -m1 -n mirror1 vgmirror

Наблюдаем синхронизацию.
lvs
lsblk

</details>
 
## Инструкция

1. Настраиваем виртуалку - нужно много разделов.
 - В Vagrantfile добавляем:
```
config.vm.provider "virtualbox" do |vb|
     vb.memory = "1024"
     second_disk = "LVMDisk.vmdk"
     vb.customize ['createhd', '--filename', second_disk, '--size', 2 * 1024]
     vb.customize ['storageattach', :id, '--storagectl', 'IDE', '--port', 1, '--device', 0, '--type', 'hdd', '--medium', second_disk]
end
```
 - Создаём нужное количество разделов:
 ```
 sudo -i
 fdisk /dev/sdb
 
 lsblk
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sda      8:0    0    10G  0 disk
└─sda1   8:1    0    10G  0 part /
sdb      8:16   0     2G  0 disk
├─sdb1   8:17   0 409.6M  0 part
├─sdb2   8:18   0 409.1M  0 part
├─sdb3   8:19   0 408.7M  0 part
├─sdb4   8:20   0 409.3M  0 part
└─sdb5   8:21   0   408M  0 part
 ```
 - Устанавливаем что нужно: `yum install lvm2 -y`
 
2. Создание всякого разного.
 - Создаём группу test на основе раздела sdb1:
 ```
 vgcreate test /dev/sdb1
  Physical volume "/dev/sdb1" successfully created.
  Volume group "test" successfully created
 ```
 - Создаём логический раздел first в группе test на 100% объёма:
 ```
 # lvcreate -l+100%FREE -n first test
 # lsblk
NAME           MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sda              8:0    0    10G  0 disk
└─sda1           8:1    0    10G  0 part /
sdb              8:16   0     2G  0 disk
├─sdb1           8:17   0 409.6M  0 part
│ └─test-first 253:0    0   408M  0 lvm
├─sdb2           8:18   0 409.1M  0 part
├─sdb3           8:19   0 408.7M  0 part
├─sdb4           8:20   0 409.3M  0 part
└─sdb5           8:21   0   408M  0 part
```
 - Создаём файловую систему ext4 и монтируем в /mnt:
```
mkfs.ext4 /dev/test/first
mount /dev/test/first /mnt
```
 - Забиваем нулями весь диск: `dd if=/dev/zero of=/mnt/test.file bs=1M count=8000 status=progress` 
  Смотрим, точно ли забилось: `df`
 ```
 Filesystem             1K-blocks    Used Available Use% Mounted on
devtmpfs                  488644       0    488644   0% /dev
tmpfs                     502480       0    502480   0% /dev/shm
tmpfs                     502480   12960    489520   3% /run
tmpfs                     502480       0    502480   0% /sys/fs/cgroup
/dev/sda1               10474496 3344744   7129752  32% /
tmpfs                     100496       0    100496   0% /run/user/1000
/dev/mapper/test-first    396396  392301         0 100% /mnt
```
  Интересует только последняя строчка. Да, диск полный, расширяемся!
3. Расширяем логический раздел за счёт физического в группе:
```
pvcreate /dev/sdb2 
vgextend test /dev/sdb2 
lvextend -l+100%FREE /dev/test/first
lvs
df
```
  Получаем прежний вывод. Расширяем файловую систему - `resize2fs /dev/test/first`. Результат:
```
# df
Filesystem             1K-blocks    Used Available Use% Mounted on
devtmpfs                  488644       0    488644   0% /dev
tmpfs                     502480       0    502480   0% /dev/shm
tmpfs                     502480   12960    489520   3% /run
tmpfs                     502480       0    502480   0% /sys/fs/cgroup
/dev/sda1               10474496 3344756   7129740  32% /
tmpfs                     100496       0    100496   0% /run/user/1000
/dev/mapper/test-first    800995  392527    366774  52% /mnt
```
  Использование 52%, размер в 2 раза больше
4. Уменьшаем раздел.
  - СНАЧАЛА уменьшаем файловую систему, чтобы не потерять данные:
  ```
  umount /mnt
  resize2fs /dev/test/first 600M
  ```
  - Уменьшаем раздел и монтируем его обратно:
  ```
  lvreduce /dev/test/first -L 600M
  mount /dev/test/first /mnt
  ```
    Что с памятью? `df`:
    ```
    Filesystem             1K-blocks    Used Available Use% Mounted on
    devtmpfs                  488644       0    488644   0% /dev
    tmpfs                     502480       0    502480   0% /dev/shm
    tmpfs                     502480   12960    489520   3% /run
    tmpfs                     502480       0    502480   0% /sys/fs/cgroup
    /dev/sda1               10474496 3344760   7129736  32% /
    tmpfs                     100496       0    100496   0% /run/user/1000
    /dev/mapper/test-first    586803  392292    162770  71% /mnt
    ```
5. Снэпшоты.
  - Создаём файлы и раздел снэпшота:
  ```
  touch /mnt/file{1..10}
  rm /mnt/test.file
  lvcreate -L 100M -s -n snap /dev/test/first
  lvs
  ```
  - Удаляем файлы:
  ```
  # rm -f /mnt/file{4..8}
  # ls /mnt
  file1  file10  file2  file3  file9  lost+found
  ```
  - Монтируем бэкап:
  ```
  # mkdir /snap
  # mount /dev/test/snap /snap
  # ls /snap
  file1  file10  file2  file3  file4  file5  file6  file7  file8  file9  lost+found
  ```
  - Восстанавливаем файлы слиянием:
  ```
  # umount /snap
  # umount /mnt
  # lvconvert --merge /dev/test/snap
  # mount /dev/test/first /mnt
  # ls /mnt
  file1  file10  file2  file3  file4  file5  file6  file7  file8  file9  lost+found
  ```
6. LV-зеркало.
  - Создаём PV и VG:
  ```
  pvcreate /dev/sdb{3,4}
  lvcreate -l+80%FREE -m1 -n mirror1 vgmirror
  lvs
  ```
  - `lsblk`:
  ```
  NAME                          MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
  sda                             8:0    0    10G  0 disk
  └─sda1                          8:1    0    10G  0 part /
  sdb                             8:16   0     2G  0 disk
  ├─sdb1                          8:17   0 409.6M  0 part
  │ └─test-first                253:0    0   600M  0 lvm  /mnt
  ├─sdb2                          8:18   0 409.1M  0 part
  │ └─test-first                253:0    0   600M  0 lvm  /mnt
  ├─sdb3                          8:19   0 408.7M  0 part
  │ ├─vgmirror-mirror1_rmeta_0  253:1    0     4M  0 lvm
  │ │ └─vgmirror-mirror1        253:5    0   324M  0 lvm
  │ └─vgmirror-mirror1_rimage_0 253:2    0   324M  0 lvm
  │   └─vgmirror-mirror1        253:5    0   324M  0 lvm
  ├─sdb4                          8:20   0 409.3M  0 part
  │ ├─vgmirror-mirror1_rmeta_1  253:3    0     4M  0 lvm
  │ │ └─vgmirror-mirror1        253:5    0   324M  0 lvm
  │ └─vgmirror-mirror1_rimage_1 253:4    0   324M  0 lvm
  │   └─vgmirror-mirror1        253:5    0   324M  0 lvm
  └─sdb5                          8:21   0   408M  0 part
  ```
> Готово!

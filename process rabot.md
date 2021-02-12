Проверяем структуру 


#Устанавливаем:
#

lsblk
NAME                    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                       8:0    0   40G  0 disk 
├─sda1                    8:1    0    1M  0 part 
├─sda2                    8:2    0    1G  0 part /boot
└─sda3                    8:3    0   39G  0 part 
  ├─VolGroup00-LogVol00 253:0    0 37.5G  0 lvm  /
  └─VolGroup00-LogVol01 253:1    0  1.5G  0 lvm  [SWAP]
sdb                       8:16   0   10G  0 disk 
sdc                       8:32   0    2G  0 disk 
sdd                       8:48   0    1G  0 disk 
sde                       8:64   0    1G  0 disk 

cat /etc/fstab
/dev/mapper/VolGroup00-LogVol00 /                       xfs     defaults        0 0
UUID=570897ca-e759-4c81-90cf-389da6eee4cc /boot                   xfs     defaults        0 0
/dev/mapper/VolGroup00-LogVol01 swap                    swap    defaults        0 0

Для работы с xfs доставляем необходимые пакеты.
yum install nano xfsdump

Особенность xfs в том , что она не предусматривает инструментов для уменьшения раздела.
Поэтому идем таким путем: создаем новую временную lvm, переносим туда весь /, настраиваем загрузчик 
для работы с новым разделом, после удаляем действующую группу lvm, создаем такую же группу с меньшим размером
, копируем всю информацию из временой группы в вновь сосданную группу и меняем загрузчик


1. Подготовим временную группу для / раздела:

    1.1) pvcreate /dev/sdb
    1.2) vgcreate vg_root /dev/sdb
    1.3) lvcreate -n lv_root -l +100%FREE /dev/vg_root

2. Создадим на нем файловую систему и смонтируем его, чтобы перенести туда данные:

    2.1) mkfs.xfs /dev/vg_root/lv_root
    2.2) mount /dev/vg_root/lv_root /mnt

3. Используем связку команд xfsdump/xfsrestore для копирования всеx данные с / раздела в /mnt:

xfsdump -J - /dev/VolGroup00/LogVol00 | xfsrestore -J - /mnt

4. Переконфигурируем grub для того, чтобý при старте перейти в новый /

    4.1) for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt/$i; done
    4.2) chroot /mnt/
    4.3) grub2-mkconfig -o /boot/grub2/grub.cfg

5. Обновим образ initrd:

cd /boot ; for i in `ls initramfs-*img`; do dracut -v $i `echo $i|sed "s/initramfs-//g; s/.img//g"` --force; done

6. Для нормальной работы необходимо конфигурировать /boot/grub2/grub.cfg, приводим его к виду  rd.lvm.lv=vg_root/lv_root

7. Также меняем в файле /etc/fstab монтирование корня и приводим его к виду:


/dev/mapper/vg_root-lv_root /                       xfs     defaults        0 0


8. Выходим из chroot командой exit. Далее reboot.

9. Перезагружаемся успешно с новым рут томом. Проверяем :

lsblk
NAME                    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                       8:0    0   40G  0 disk
├─sda1                    8:1    0    1M  0 part
├─sda2                    8:2    0    1G  0 part /boot
└─sda3                    8:3    0   39G  0 part
  ├─VolGroup00-LogVol01 253:1    0  1.5G  0 lvm  [SWAP]
  └─VolGroup00-LogVol00 253:2    0 37.5G  0 lvm
sdb                       8:16   0   10G  0 disk
└─vg_root-lv_root       253:0    0   10G  0 lvm  /
sdc                       8:32   0    2G  0 disk
sdd                       8:48   0    1G  0 disk
sde                       8:64   0    1G  0 disk


10. Удаляем старую группу и создаем новую с меньшим размером:

    10.1) lvremove /dev/VolGroup00/LogVol00
    10.2) lvcreate -n VolGroup00/LogVol00 -L 8G /dev/VolGroup00

11. Создаем файловую систему, монтируем и копируем туда /:

    11.1) mkfs.xfs /dev/VolGroup00/LogVol00
    11.2) mount /dev/VolGroup00/LogVol00 /mnt
    11.3) xfsdump -J - /dev/vg_root/lv_root | xfsrestore -J - /mnt

12. Монтируем новый /, переконфигурируем grub2, обновляем образ :

    12.1) for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt/$i; done
    12.2) chroot /mnt/
    12.3) grub2-mkconfig -o /boot/grub2/grub.cfg
    12.4) cd /boot ; for i in `ls initramfs-*img`; do dracut -v $i `echo $i|sed "s/initramfs-//g; s/.img//g"` --force; done

Пока не перезагружаемся и не выходим из под chroot - мы сразу создать раздел для /var и перенести его:

13. На свободных дисках создаем массив с дисками в зеркале:

    13.1) pvcreate /dev/sdc /dev/sdd
    13.2) vgcreate vg_var /dev/sdc /dev/sdd
    13.3) lvcreate -L 950M -m1 -n lv_var vg_var

14. Создаем на файловую систему , монтируем  и переносим /var:

    14.1) mkfs.ext4 /dev/vg_var/lv_var
    14.2) mount /dev/vg_var/lv_var /mnt
    14.3) rsync -avHPSAX /var/ /mnt/
    14.4) rm -rf /var/*

15. Монтируем новый var в каталог /var:
    15.1) umount /mnt
    15.2) mount /dev/vg_var/lv_var /var

16. Правим fstab для автоматического монтирования /var:
    echo "`blkid | grep var: | awk '{print $2}'` /var ext4 defaults 0 0" >> /etc/fstab

17. Также меняем в файле /etc/fstab монтирование корня и приводим его к виду:

    /dev/mapper/VolGroup00-LogVol00 /                       xfs     defaults        0 0


18. Перезагружаться в уменьшенный / и удалять временную Volume Group:
    18.1) lvremove /dev/vg_root/lv_root
    18.2) vgremove /dev/vg_root
    18.3) pvremove /dev/sdb

19. Выделяем том под /home :
    19.1) lvcreate -n LogVol_Home -L 2G /dev/VolGroup00
    19.2) mkfs.xfs /dev/VolGroup00/LogVol_Home
    19.3) mount /dev/VolGroup00/LogVol_Home /mnt/
    19.4) rsync -avHPSAX /home/ /mnt/
    19.5) rm -rf /home/*
    19.5) umount /mnt
    19.6) mount /dev/VolGroup00/LogVol_Home /home/

20. Правим fstab длā автоматического монтирования /home:
    echo "`blkid | grep Home | awk '{print $2}'` /home xfs defaults 0 0" >> /etc/fstab

21. Работа со снапшетами:

    21.1) создаем файлы в /home/: touch /home/file{1..10}

    21.2) Снять снапшот: lvcreate -L 100MB -s -n home_snap /dev/VolGroup00/LogVol_Home

    21.3) Удалить часть файлов: rm -f /home/file{6..8}

    21.4) Процесс восстановления со снапшота:

        21.4.1) umount /home
        21.4.2) lvconvert --merge /dev/VolGroup00/home_snap 
        21.4.3) mount /home



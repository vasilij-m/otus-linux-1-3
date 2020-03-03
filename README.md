**1. Уменьшить том под / до 8G**
Для снятия копии раздела / необходимо установить пакет **xfsdump**:
```
[vagrant@lvm ~]$ sudo su
[root@lvm vagrant]# yum install xfsdump -y
```
Выведем список блочных устройств:
```
[root@lvm vagrant]# lsblk
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
```
Для раздела / подготовим временный том на устройстве /dev/sdb, для чего создадим Physical Volume, Volume Group и Logical Volume:
```
[root@lvm vagrant]# pvcreate /dev/sdb
  Physical volume "/dev/sdb" successfully created.
[root@lvm vagrant]# vgcreate vg_root /dev/sdb
  Volume group "vg_root" successfully created
[root@lvm vagrant]# lvcreate -n lv_root -l +100%FREE /dev/vg_root
  Logical volume "lv_root" created.
```
Создадим на логическом томе lv_root файловую систему и смонтируем его для переноса данных:
```
[root@lvm vagrant]# mkfs.xfs /dev/vg_root/lv_root
meta-data=/dev/vg_root/lv_root   isize=512    agcount=4, agsize=655104 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0, sparse=0
data     =                       bsize=4096   blocks=2620416, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
[root@lvm vagrant]# mount /dev/vg_root/lv_root /mnt
[root@lvm vagrant]# lsblk
NAME                    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                       8:0    0   40G  0 disk 
├─sda1                    8:1    0    1M  0 part 
├─sda2                    8:2    0    1G  0 part /boot
└─sda3                    8:3    0   39G  0 part 
  ├─VolGroup00-LogVol00 253:0    0 37.5G  0 lvm  /
  └─VolGroup00-LogVol01 253:1    0  1.5G  0 lvm  [SWAP]
sdb                       8:16   0   10G  0 disk 
└─vg_root-lv_root       253:2    0   10G  0 lvm  /mnt
sdc                       8:32   0    2G  0 disk 
sdd                       8:48   0    1G  0 disk 
sde                       8:64   0    1G  0 disk 
```
Скопируем все данные с раздела / в раздел /mnt:
```
[root@lvm vagrant]# xfsdump -J - /dev/VolGroup00/LogVol00 | xfsrestore -J - /mnt
...
xfsdump: dump complete: 10 seconds elapsed
xfsdump: Dump Status: SUCCESS
xfsrestore: restore complete: 11 seconds elapsed
xfsrestore: Restore Status: SUCCESS
```
Проверим, что содержимое скопировалось:
```
[root@lvm vagrant]# ls /mnt
bin   dev  home  lib64  mnt  proc  run   srv  tmp  vagrant
boot  etc  lib   media  opt  root  sbin  sys  usr  var
```
Теперь нужно обновить конфигурацию grub, чтобы при старте системы перейти в новый /. 
Для этого примонтируем содержимое оригинального раздела / в /mnt, изменим корневой раздел на /mnt с помощью **chroot** и обновим конфиг grub'а:
```
[root@lvm vagrant]# for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt/$i; done
[root@lvm vagrant]# chroot /mnt/
[root@lvm /]# grub2-mkconfig -o /boot/grub2/grub.cfg
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-3.10.0-862.2.3.el7.x86_64
Found initrd image: /boot/initramfs-3.10.0-862.2.3.el7.x86_64.img
Found CentOS Linux release 7.5.1804 (Core)  on /dev/mapper/vg_root-lv_root
done
```
Обновим образ **initramfs** - это временная файловая система, используемая в качестве корневой файловой системы ядром Linux при начальной загрузке (_Initial_ _RAM_ _Disk_). Для обновления (создания) initrd используется утилита **dracut**:
```
[root@lvm /]# cd /boot ; for i in $(ls initramfs-*img); do dracut -v $i $(echo $i | sed "s/initramfs-//g;s/.img//g") --force; done
...
*** Creating image file ***
*** Creating image file done ***
*** Creating initramfs image file '/boot/initramfs-3.10.0-862.2.3.el7.x86_64.img' done ***
```
Для того, чтобы при загрузке был смонтирован нужный **root** необходимо в файле /boot/grub2/grub.cfg заменить rd.lvm.lv=VolGroup00/LogVol00 на rd.lvm.lv=vg_root/lv_root:
```
[root@lvm boot]# sed -i 's/rd\.lvm\.lv=VolGroup00\/LogVol00/rd\.lvm\.lv=vg_root\/lv_root/' /boot/grub2/grub.cfg
```
Теперь необходимо перезагрузиться и убедиться, что мы загрузились с новым корневым разделом:
```
[root@lvm boot]# exit
exit
[root@lvm vagrant]# reboot
...
[vagrant@lvm ~]$ lsblk
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
```
Теперь можно удалить старый LV (LogVol00) и создать новый LV на 8Gb, чтобы вернуть на него корневой раздел.
```
[vagrant@lvm ~]$ sudo su
[root@lvm vagrant]# lvremove /dev/VolGroup00/LogVol00 -y
  Logical volume "LogVol00" successfully removed
[root@lvm vagrant]# lvcreate -n LogVol00 -L 8G /dev/VolGroup00 -y
  Wiping xfs signature on /dev/VolGroup00/LogVol00.
  Logical volume "LogVol00" created.
```
Создадим на новом LV файловую систему xfs и примонтируем её в /mnt:
```
[root@lvm vagrant]# mkfs.xfs /dev/VolGroup00/LogVol00 && mount /dev/VolGroup00/LogVol00 /mnt
meta-data=/dev/VolGroup00/LogVol00 isize=512    agcount=4, agsize=524288 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0, sparse=0
data     =                       bsize=4096   blocks=2097152, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
```
С помощью утилит **xfsdump** и **xfsrestore** скопируем данные с lv_root на LogVol00:
```
[root@lvm vagrant]# xfsdump -J - /dev/vg_root/lv_root | xfsrestore -J - /mnt
...
xfsdump: Dump Status: SUCCESS
xfsrestore: restore complete: 5 seconds elapsed
xfsrestore: Restore Status: SUCCESS
```
Примонтируем содержимое текущего раздела / в /mnt, изменим корневой раздел на /mnt с помощью **chroot**, обновим конфиг grub'а и образ initramfs:
```
root@lvm vagrant]# for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt/$i; done
[root@lvm vagrant]# chroot /mnt/
[root@lvm /]# grub2-mkconfig -o /boot/grub2/grub.cfg
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-3.10.0-862.2.3.el7.x86_64
Found initrd image: /boot/initramfs-3.10.0-862.2.3.el7.x86_64.img
done
[root@lvm /]# cd /boot ; for i in $(ls initramfs-*img); do dracut -v $i $(echo $i | sed "s/initramfs-//g;s/.img//g") --force; done
...
*** Creating image file ***
*** Creating image file done ***
*** Creating initramfs image file '/boot/initramfs-3.10.0-862.2.3.el7.x86_64.img' done ***
```
Не выходя из **chroot**, перенесем **/var** на предварительно выделенный для него том, сделанный в mirror.

**2. Выделить том под /var**
```
[root@lvm boot]# lsblk
NAME                    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                       8:0    0   40G  0 disk 
├─sda1                    8:1    0    1M  0 part 
├─sda2                    8:2    0    1G  0 part /boot
└─sda3                    8:3    0   39G  0 part 
  ├─VolGroup00-LogVol01 253:1    0  1.5G  0 lvm  [SWAP]
  └─VolGroup00-LogVol00 253:2    0    8G  0 lvm  /
sdb                       8:16   0   10G  0 disk 
└─vg_root-lv_root       253:0    0   10G  0 lvm  
sdc                       8:32   0    2G  0 disk 
sdd                       8:48   0    1G  0 disk 
sde                       8:64   0    1G  0 disk 
```
Зеркало под **/var** создадим на устройствах **sdd** и **sde**:
```
[root@lvm boot]# pvcreate /dev/sdd /dev/sde
  Physical volume "/dev/sdd" successfully created.
  Physical volume "/dev/sde" successfully created.
[root@lvm boot]# vgcreate vg_var /dev/sdd /dev/sde
  Volume group "vg_var" successfully created
[root@lvm boot]# lvcreate -n lv_var -m1 -l +100%FREE /dev/vg_var
  Logical volume "lv_var" created.
[root@lvm boot]# lsblk
NAME                     MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                        8:0    0   40G  0 disk 
├─sda1                     8:1    0    1M  0 part 
├─sda2                     8:2    0    1G  0 part /boot
└─sda3                     8:3    0   39G  0 part 
  ├─VolGroup00-LogVol01  253:1    0  1.5G  0 lvm  [SWAP]
  └─VolGroup00-LogVol00  253:2    0    8G  0 lvm  /
sdb                        8:16   0   10G  0 disk 
└─vg_root-lv_root        253:0    0   10G  0 lvm  
sdc                        8:32   0    2G  0 disk 
sdd                        8:48   0    1G  0 disk 
├─vg_var-lv_var_rmeta_0  253:3    0    4M  0 lvm  
│ └─vg_var-lv_var        253:7    0 1016M  0 lvm  
└─vg_var-lv_var_rimage_0 253:4    0 1016M  0 lvm  
  └─vg_var-lv_var        253:7    0 1016M  0 lvm  
sde                        8:64   0    1G  0 disk 
├─vg_var-lv_var_rmeta_1  253:5    0    4M  0 lvm  
│ └─vg_var-lv_var        253:7    0 1016M  0 lvm  
└─vg_var-lv_var_rimage_1 253:6    0 1016M  0 lvm  
  └─vg_var-lv_var        253:7    0 1016M  0 lvm  
```
Создаем на lv_var файловую систему и перемещаем туда **/var**:
```
[root@lvm boot]# mkfs.ext4 /dev/vg_var/lv_var 
mke2fs 1.42.9 (28-Dec-2013)
Filesystem label=
OS type: Linux
Block size=4096 (log=2)
Fragment size=4096 (log=2)
Stride=0 blocks, Stripe width=0 blocks
65024 inodes, 260096 blocks
13004 blocks (5.00%) reserved for the super user
First data block=0
Maximum filesystem blocks=266338304
8 block groups
32768 blocks per group, 32768 fragments per group
8128 inodes per group
Superblock backups stored on blocks: 
	32768, 98304, 163840, 229376

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (4096 blocks): done
Writing superblocks and filesystem accounting information: done
[root@lvm boot]# mount /dev/vg_var/lv_var /mnt/
[root@lvm boot]# cp -aR /var/* /mnt/
```
На всякий случай переместим содержимое старого **/var**:
```
[root@lvm boot]# mkdir /tmp/oldvar && mv /var/* /tmp/oldvar
```
Теперь примонтируем lv_var в /var, не забыв предварительно отмонитровать его из /mnt:
```
[root@lvm boot]# umount /mnt
[root@lvm boot]# mount /dev/vg_var/lv_var /var
```
Для автоматического монтирования /var пропишем монтирование в fstab:
```
[root@lvm boot]# echo "$(blkid | grep var: | awk '{print $2}') /var ext4 defaults 0 0" >> /etc/fstab
```
Теперь можно перезагрузиться под новым (уменьшенным) корневым разделом и удалить временную vg_root:
```
[root@lvm boot]# exit
exit
[root@lvm vagrant]# reboot
...
[vagrant@lvm ~]$ lsblk
NAME                     MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                        8:0    0   40G  0 disk 
├─sda1                     8:1    0    1M  0 part 
├─sda2                     8:2    0    1G  0 part /boot
└─sda3                     8:3    0   39G  0 part 
  ├─VolGroup00-LogVol00  253:0    0    8G  0 lvm  /
  └─VolGroup00-LogVol01  253:1    0  1.5G  0 lvm  [SWAP]
sdb                        8:16   0   10G  0 disk 
└─vg_root-lv_root        253:2    0   10G  0 lvm  
sdc                        8:32   0    2G  0 disk 
sdd                        8:48   0    1G  0 disk 
├─vg_var-lv_var_rmeta_0  253:3    0    4M  0 lvm  
│ └─vg_var-lv_var        253:7    0 1016M  0 lvm  /var
└─vg_var-lv_var_rimage_0 253:4    0 1016M  0 lvm  
  └─vg_var-lv_var        253:7    0 1016M  0 lvm  /var
sde                        8:64   0    1G  0 disk 
├─vg_var-lv_var_rmeta_1  253:5    0    4M  0 lvm  
│ └─vg_var-lv_var        253:7    0 1016M  0 lvm  /var
└─vg_var-lv_var_rimage_1 253:6    0 1016M  0 lvm  
  └─vg_var-lv_var        253:7    0 1016M  0 lvm  /var

[vagrant@lvm ~]$ sudo -i
[root@lvm ~]# lvremove /dev/vg_root/lv_root -y
  Logical volume "lv_root" successfully removed
[root@lvm ~]# vgremove /dev/vg_root -y
  Volume group "vg_root" successfully removed
[root@lvm ~]# pvremove /dev/sdb
  Labels on physical volume "/dev/sdb" successfully wiped.
[root@lvm ~]# lsblk
NAME                     MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                        8:0    0   40G  0 disk 
├─sda1                     8:1    0    1M  0 part 
├─sda2                     8:2    0    1G  0 part /boot
└─sda3                     8:3    0   39G  0 part 
  ├─VolGroup00-LogVol00  253:0    0    8G  0 lvm  /
  └─VolGroup00-LogVol01  253:1    0  1.5G  0 lvm  [SWAP]
sdb                        8:16   0   10G  0 disk 
sdc                        8:32   0    2G  0 disk 
sdd                        8:48   0    1G  0 disk 
├─vg_var-lv_var_rmeta_0  253:3    0    4M  0 lvm  
│ └─vg_var-lv_var        253:7    0 1016M  0 lvm  /var
└─vg_var-lv_var_rimage_0 253:4    0 1016M  0 lvm  
  └─vg_var-lv_var        253:7    0 1016M  0 lvm  /var
sde                        8:64   0    1G  0 disk 
├─vg_var-lv_var_rmeta_1  253:5    0    4M  0 lvm  
│ └─vg_var-lv_var        253:7    0 1016M  0 lvm  /var
└─vg_var-lv_var_rimage_1 253:6    0 1016M  0 lvm  
  └─vg_var-lv_var        253:7    0 1016M  0 lvm  /var
```
**3. Выделить том под /home**
Создадим LV LogVol_Home на VG VolGroup00, отформатируем его в xfs и примонтируем в /mnt, чтобы можно было скопировать данные с оригинального /home:
```
[root@lvm ~]# lvcreate -n LogVol_Home -L 2G /dev/VolGroup00
  Logical volume "LogVol_Home" created.
[root@lvm ~]# mkfs.xfs /dev/VolGroup00/LogVol_Home 
meta-data=/dev/VolGroup00/LogVol_Home isize=512    agcount=4, agsize=131072 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0, sparse=0
data     =                       bsize=4096   blocks=524288, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
[root@lvm ~]# mount /dev/VolGroup00/LogVol_Home /mnt/
```
Сделаем копию данных с /home, удалим файлы из оригинального /home и примонтируем LogVol_Home в /home:
```
[root@lvm ~]# cp -aR /home/* /mnt/
[root@lvm ~]# rm -rf /home/*
[root@lvm ~]# umount /mnt/
[root@lvm ~]# mount /dev/VolGroup00/LogVol_Home /home/
[root@lvm ~]# lsblk
NAME                       MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                          8:0    0   40G  0 disk 
├─sda1                       8:1    0    1M  0 part 
├─sda2                       8:2    0    1G  0 part /boot
└─sda3                       8:3    0   39G  0 part 
  ├─VolGroup00-LogVol00    253:0    0    8G  0 lvm  /
  ├─VolGroup00-LogVol01    253:1    0  1.5G  0 lvm  [SWAP]
  └─VolGroup00-LogVol_Home 253:2    0    2G  0 lvm  /home
sdb                          8:16   0   10G  0 disk 
sdc                          8:32   0    2G  0 disk 
sdd                          8:48   0    1G  0 disk 
├─vg_var-lv_var_rmeta_0    253:3    0    4M  0 lvm  
│ └─vg_var-lv_var          253:7    0 1016M  0 lvm  /var
└─vg_var-lv_var_rimage_0   253:4    0 1016M  0 lvm  
  └─vg_var-lv_var          253:7    0 1016M  0 lvm  /var
sde                          8:64   0    1G  0 disk 
├─vg_var-lv_var_rmeta_1    253:5    0    4M  0 lvm  
│ └─vg_var-lv_var          253:7    0 1016M  0 lvm  /var
└─vg_var-lv_var_rimage_1   253:6    0 1016M  0 lvm  
  └─vg_var-lv_var          253:7    0 1016M  0 lvm  /var
```
Для автоматического монтирования пропишем LogVol_Home в fstab:
```
[root@lvm boot]# echo "$(blkid | grep Home: | awk '{print $2}') /home xfs defaults 0 0" >> /etc/fstab
```
**3. /home - сделать том для снапшотов**
Сгенерируем файлы в /home:
```
[root@lvm ~]# touch /home/file{1..20}
[root@lvm ~]# ll /home/
total 0
-rw-r--r--. 1 root    root     0 Mar  1 16:04 file1
-rw-r--r--. 1 root    root     0 Mar  1 16:04 file10
-rw-r--r--. 1 root    root     0 Mar  1 16:04 file11
-rw-r--r--. 1 root    root     0 Mar  1 16:04 file12
-rw-r--r--. 1 root    root     0 Mar  1 16:04 file13
-rw-r--r--. 1 root    root     0 Mar  1 16:04 file14
-rw-r--r--. 1 root    root     0 Mar  1 16:04 file15
-rw-r--r--. 1 root    root     0 Mar  1 16:04 file16
-rw-r--r--. 1 root    root     0 Mar  1 16:04 file17
-rw-r--r--. 1 root    root     0 Mar  1 16:04 file18
-rw-r--r--. 1 root    root     0 Mar  1 16:04 file19
-rw-r--r--. 1 root    root     0 Mar  1 16:04 file2
-rw-r--r--. 1 root    root     0 Mar  1 16:04 file20
-rw-r--r--. 1 root    root     0 Mar  1 16:04 file3
-rw-r--r--. 1 root    root     0 Mar  1 16:04 file4
-rw-r--r--. 1 root    root     0 Mar  1 16:04 file5
-rw-r--r--. 1 root    root     0 Mar  1 16:04 file6
-rw-r--r--. 1 root    root     0 Mar  1 16:04 file7
-rw-r--r--. 1 root    root     0 Mar  1 16:04 file8
-rw-r--r--. 1 root    root     0 Mar  1 16:04 file9
drwx------. 3 vagrant vagrant 74 May 12  2018 vagrant
```
Снимем снапшот тома LogVol_Home:
```
[root@lvm ~]# lvcreate -L 100MB -s -n home_snap /dev/VolGroup00/LogVol_Home
  Rounding up size to full physical extent 128.00 MiB
  Logical volume "home_snap" created.
[root@lvm ~]# lsblk
NAME                            MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                               8:0    0   40G  0 disk 
├─sda1                            8:1    0    1M  0 part 
├─sda2                            8:2    0    1G  0 part /boot
└─sda3                            8:3    0   39G  0 part 
  ├─VolGroup00-LogVol00         253:0    0    8G  0 lvm  /
  ├─VolGroup00-LogVol01         253:1    0  1.5G  0 lvm  [SWAP]
  ├─VolGroup00-LogVol_Home-real 253:8    0    2G  0 lvm  
  │ ├─VolGroup00-LogVol_Home    253:2    0    2G  0 lvm  /home
  │ └─VolGroup00-home_snap      253:10   0    2G  0 lvm  
  └─VolGroup00-home_snap-cow    253:9    0  128M  0 lvm  
    └─VolGroup00-home_snap      253:10   0    2G  0 lvm  
sdb                               8:16   0   10G  0 disk 
sdc                               8:32   0    2G  0 disk 
sdd                               8:48   0    1G  0 disk 
├─vg_var-lv_var_rmeta_0         253:3    0    4M  0 lvm  
│ └─vg_var-lv_var               253:7    0 1016M  0 lvm  /var
└─vg_var-lv_var_rimage_0        253:4    0 1016M  0 lvm  
  └─vg_var-lv_var               253:7    0 1016M  0 lvm  /var
sde                               8:64   0    1G  0 disk 
├─vg_var-lv_var_rmeta_1         253:5    0    4M  0 lvm  
│ └─vg_var-lv_var               253:7    0 1016M  0 lvm  /var
└─vg_var-lv_var_rimage_1        253:6    0 1016M  0 lvm  
  └─vg_var-lv_var               253:7    0 1016M  0 lvm  /var
```
Удалим часть файлов:
```
[root@lvm ~]# rm -f /home/file{11..20}
[root@lvm ~]# ll /home/
total 0
-rw-r--r--. 1 root    root     0 Mar  1 16:04 file1
-rw-r--r--. 1 root    root     0 Mar  1 16:04 file10
-rw-r--r--. 1 root    root     0 Mar  1 16:04 file2
-rw-r--r--. 1 root    root     0 Mar  1 16:04 file3
-rw-r--r--. 1 root    root     0 Mar  1 16:04 file4
-rw-r--r--. 1 root    root     0 Mar  1 16:04 file5
-rw-r--r--. 1 root    root     0 Mar  1 16:04 file6
-rw-r--r--. 1 root    root     0 Mar  1 16:04 file7
-rw-r--r--. 1 root    root     0 Mar  1 16:04 file8
-rw-r--r--. 1 root    root     0 Mar  1 16:04 file9
drwx------. 3 vagrant vagrant 74 May 12  2018 vagrant
```
Для восстановления /home из снапшота необходимо отмонтировать /home, выполнить восстановление и примонтировать /home снова:
```
[root@lvm ~]# umount /home
[root@lvm ~]# lvconvert --merge /dev/VolGroup00/home_snap
  Merging of volume VolGroup00/home_snap started.
  VolGroup00/LogVol_Home: Merged: 100.00%
[root@lvm ~]# mount /home
[root@lvm ~]# ll /home/
total 0
-rw-r--r--. 1 root    root     0 Mar  1 16:04 file1
-rw-r--r--. 1 root    root     0 Mar  1 16:04 file10
-rw-r--r--. 1 root    root     0 Mar  1 16:04 file11
-rw-r--r--. 1 root    root     0 Mar  1 16:04 file12
-rw-r--r--. 1 root    root     0 Mar  1 16:04 file13
-rw-r--r--. 1 root    root     0 Mar  1 16:04 file14
-rw-r--r--. 1 root    root     0 Mar  1 16:04 file15
-rw-r--r--. 1 root    root     0 Mar  1 16:04 file16
-rw-r--r--. 1 root    root     0 Mar  1 16:04 file17
-rw-r--r--. 1 root    root     0 Mar  1 16:04 file18
-rw-r--r--. 1 root    root     0 Mar  1 16:04 file19
-rw-r--r--. 1 root    root     0 Mar  1 16:04 file2
-rw-r--r--. 1 root    root     0 Mar  1 16:04 file20
-rw-r--r--. 1 root    root     0 Mar  1 16:04 file3
-rw-r--r--. 1 root    root     0 Mar  1 16:04 file4
-rw-r--r--. 1 root    root     0 Mar  1 16:04 file5
-rw-r--r--. 1 root    root     0 Mar  1 16:04 file6
-rw-r--r--. 1 root    root     0 Mar  1 16:04 file7
-rw-r--r--. 1 root    root     0 Mar  1 16:04 file8
-rw-r--r--. 1 root    root     0 Mar  1 16:04 file9
drwx------. 3 vagrant vagrant 74 May 12  2018 vagrant
```
Как видно, восстановление прошло успешно.


**2. Доп. задание: BTRFS**

```
[vagrant@lvm ~]$ lsblk
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
```
Отформатируем в BTRFS 2 устройства: **sdb** и **sdc**, причем данные будем хранить в RAID-0, а метаданные в RAID-1:
``` 
[vagrant@lvm ~]$ sudo -i
[root@lvm ~]# mkfs.btrfs -m raid1 -d raid0 /dev/sdb /dev/sdc
btrfs-progs v4.9.1
See http://btrfs.wiki.kernel.org for more information.

Label:              (null)
UUID:               8a5097df-feb4-4f02-b725-bd7583b4d327
Node size:          16384
Sector size:        4096
Filesystem size:    12.00GiB
Block group profiles:
  Data:             RAID0             1.20GiB
  Metadata:         RAID1             1.00GiB
  System:           RAID1             8.00MiB
SSD detected:       no
Incompat features:  extref, skinny-metadata
Number of devices:  2
Devices:
   ID        SIZE  PATH
    1    10.00GiB  /dev/sdb
    2     2.00GiB  /dev/sdc
root@lvm ~]# btrfs filesystem show /dev/sdb
Label: none  uuid: 8a5097df-feb4-4f02-b725-bd7583b4d327
	Total devices 2 FS bytes used 112.00KiB
	devid    1 size 10.00GiB used 1.61GiB path /dev/sdb
	devid    2 size 2.00GiB used 1.61GiB path /dev/sdc

```
Смонтируем созданную ФС в /mnt и просмотрим информацию о смонтированном разделе:
```
[root@lvm ~]# mkdir /mnt/btrfs
[root@lvm ~]# mount /dev/sdb /mnt/btrfs
[root@lvm ~]# df -h
Filesystem                       Size  Used Avail Use% Mounted on
/dev/mapper/VolGroup00-LogVol00   38G  760M   37G   2% /
devtmpfs                         1.9G     0  1.9G   0% /dev
tmpfs                            1.9G     0  1.9G   0% /dev/shm
tmpfs                            1.9G  8.6M  1.9G   1% /run
tmpfs                            1.9G     0  1.9G   0% /sys/fs/cgroup
/dev/sda2                       1014M   63M  952M   7% /boot
tmpfs                            380M     0  380M   0% /run/user/1000
tmpfs                            380M     0  380M   0% /run/user/0
/dev/sdb                          12G   17M  2.0G   1% /mnt/btrfs

```
Cоздадим subvolume для размещения **/opt**:
```
[root@lvm ~]# btrfs subvolume create /mnt/btrfs/sv-opt
Create subvolume '/mnt/btrfs/sv-opt'
root@lvm ~]# ll /mnt/btrfs/
total 0
drwxr-xr-x. 1 root root 0 Mar  2 22:27 sv-opt
```
Примонтируем сабвольюм **sv-opt** к **/opt**, узнав предварительно его ID:
```
[root@lvm ~]# btrfs subvolume list /mnt/btrfs
ID 261 gen 22 top level 5 path sv-opt
[root@lvm ~]# mount -o subvolid=261 /dev/sdb /opt
```
Создадим в **/opt** файлы, сделаем снапшот:
```
[root@lvm ~]# cd /opt
[root@lvm opt]# touch file{1..10}
[root@lvm opt]# ll
total 0
-rw-r--r--. 1 root root 0 Mar  2 21:45 file1
-rw-r--r--. 1 root root 0 Mar  2 21:45 file10
-rw-r--r--. 1 root root 0 Mar  2 21:45 file2
-rw-r--r--. 1 root root 0 Mar  2 21:45 file3
-rw-r--r--. 1 root root 0 Mar  2 21:45 file4
-rw-r--r--. 1 root root 0 Mar  2 21:45 file5
-rw-r--r--. 1 root root 0 Mar  2 21:45 file6
-rw-r--r--. 1 root root 0 Mar  2 21:45 file7
-rw-r--r--. 1 root root 0 Mar  2 21:45 file8
-rw-r--r--. 1 root root 0 Mar  2 21:45 file9
[root@lvm opt]# btrfs subvolume snapshot /mnt/btrfs/sv-opt/ /mnt/btrfs/sv-backup/
Create a snapshot of '/mnt/btrfs/sv-opt/' in '/mnt/btrfs/sv-backup'
[root@lvm opt]# ll /mnt/btrfs/sv-backup/
total 0
-rw-r--r--. 1 root root 0 Mar  2 22:30 file1
-rw-r--r--. 1 root root 0 Mar  2 22:30 file10
-rw-r--r--. 1 root root 0 Mar  2 22:30 file2
-rw-r--r--. 1 root root 0 Mar  2 22:30 file3
-rw-r--r--. 1 root root 0 Mar  2 22:30 file4
-rw-r--r--. 1 root root 0 Mar  2 22:30 file5
-rw-r--r--. 1 root root 0 Mar  2 22:30 file6
-rw-r--r--. 1 root root 0 Mar  2 22:30 file7
-rw-r--r--. 1 root root 0 Mar  2 22:30 file8
-rw-r--r--. 1 root root 0 Mar  2 22:30 file9
```
Удалим часть файлов из **/opt**:
```
[root@lvm opt]# rm -rf /opt/file{1..5} && ll
total 0
-rw-r--r--. 1 root root 0 Mar  2 21:45 file10
-rw-r--r--. 1 root root 0 Mar  2 21:45 file6
-rw-r--r--. 1 root root 0 Mar  2 21:45 file7
-rw-r--r--. 1 root root 0 Mar  2 21:45 file8
-rw-r--r--. 1 root root 0 Mar  2 21:45 file9
```
Восстановление со снапшота в BTRFS предполагает замену оригинального сабвольюма на его снапшот, т.е. в случае с нашим /opt нужно примонтировать sv-backup вместо sv-opt:
```
[root@lvm opt]# cd ..
[root@lvm /]# umount /opt
[root@lvm /]# btrfs subvolume list /mnt/btrfs
ID 261 gen 28 top level 5 path sv-opt
ID 263 gen 27 top level 5 path sv-backup
[root@lvm /]# mount -o subvolid=263 /dev/sdb /opt
[root@lvm /]# ll /opt
total 0
-rw-r--r--. 1 root root 0 Mar  2 22:30 file1
-rw-r--r--. 1 root root 0 Mar  2 22:30 file10
-rw-r--r--. 1 root root 0 Mar  2 22:30 file2
-rw-r--r--. 1 root root 0 Mar  2 22:30 file3
-rw-r--r--. 1 root root 0 Mar  2 22:30 file4
-rw-r--r--. 1 root root 0 Mar  2 22:30 file5
-rw-r--r--. 1 root root 0 Mar  2 22:30 file6
-rw-r--r--. 1 root root 0 Mar  2 22:30 file7
-rw-r--r--. 1 root root 0 Mar  2 22:30 file8
-rw-r--r--. 1 root root 0 Mar  2 22:30 file9
```
Как видно, все файлы на месте.












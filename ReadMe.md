# Работа с LVM

## Цель:
создание и работа с логическими томами;
## Домашнее задание

1. Уменьшить том под / до 8G
2. Выделить том под /home
3. Для /home - сделать том для снэпшотов
4. прописать монтирование в fstab (попробовать с разными опциями и разными файловыми системами на выбор)
5. Работа со снапшотами:
* сгенерировать файлы в /home/;
* снять снэпшот;
* удалить часть файлов;
* восстановиться со снэпшота;

Задание со звездочкой*
На существующих дисках попробовать поставить btrfs/zfs:
 * с кешем и снэпшотами
 * разметить здесь каталог /opt


## Описание выполнения домашнего задания.

Конфигурация стенда: 
образ (centos/7 1804.2)
 
Vagrantfile [взят отсюда](https://gitlab.com/otus_linux/stands-03-lvm)

### 1. Уменьшить том под / до 8G

Уменьшение будем проводить с использованием временного диска под / , для того чтобы не использовать LiveCD.

#### 1.1 Проверка существующей дисковой системы, ее конфигурация и файловая систем

``` BASH
[root@lvm ~]# lsblk
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

root@lvm ~]# df -Th
Filesystem                      Type      Size  Used Avail Use% Mounted on
/dev/mapper/VolGroup00-LogVol00 xfs        38G  884M   37G   3% /
devtmpfs                        devtmpfs  109M     0  109M   0% /dev
tmpfs                           tmpfs     118M     0  118M   0% /dev/shm
tmpfs                           tmpfs     118M  4.5M  114M   4% /run
tmpfs                           tmpfs     118M     0  118M   0% /sys/fs/cgroup
/dev/sda2                       xfs      1014M   63M  952M   7% /boot
tmpfs                           tmpfs      24M     0   24M   0% /run/user/1000

[root@lvm ~]# pvs
  PV         VG         Fmt  Attr PSize   PFree
  /dev/sda3  VolGroup00 lvm2 a--  <38.97g    0 
[root@lvm ~]# vgs
  VG         #PV #LV #SN Attr   VSize   VFree
  VolGroup00   1   2   0 wz--n- <38.97g    0 
[root@lvm ~]# lvs
  LV       VG         Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  LogVol00 VolGroup00 -wi-ao---- <37.47g                                                    
  LogVol01 VolGroup00 -wi-ao----   1.50g                                                    
[root@lvm ~]# 
```
#### 1.2 Создание новый временный / 

1.2.1 Создаем физический том (PV), группу физических томов (VG), логический том (VL).
Логический том создаем на все свободное пространство.
```
[root@lvm ~]# pvcreate /dev/sdb
  Physical volume "/dev/sdb" successfully created.

[root@lvm ~]# vgcreate vg_root /dev/sdb
  Volume group "vg_root" successfully created

[root@lvm ~]# lvcreate -n lv_root -l +100%FREE /dev/vg_root
  Logical volume "lv_root" created.

```
1.2.2 Создаем файловую систему `xfs` на `lv_root`
```
[root@lvm ~]# mkfs.xfs /dev/vg_root/lv_root
meta-data=/dev/vg_root/lv_root   isize=512    agcount=4, agsize=655104 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0, sparse=0
data     =                       bsize=4096   blocks=2620416, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
[root@lvm ~]# 
```
монтируем ее в `/mnt`

```
[root@lvm ~]# mount /dev/vg_root/lv_root /mnt
[root@lvm ~]# df -Th
Filesystem                      Type      Size  Used Avail Use% Mounted on
/dev/mapper/VolGroup00-LogVol00 xfs        38G  884M   37G   3% /
devtmpfs                        devtmpfs  109M     0  109M   0% /dev
tmpfs                           tmpfs     118M     0  118M   0% /dev/shm
tmpfs                           tmpfs     118M  4.5M  114M   4% /run
tmpfs                           tmpfs     118M     0  118M   0% /sys/fs/cgroup
/dev/sda2                       xfs      1014M   63M  952M   7% /boot
tmpfs                           tmpfs      24M     0   24M   0% /run/user/1000
/dev/mapper/vg_root-lv_root     xfs        10G   33M   10G   1% /mnt
[root@lvm ~]# 

```
копируем данные с `/dev/VolGroup00/LogVol00` в `/mnt` командой 

`xfsdump -J - /dev/VolGroup00/LogVol00 | xfsrestore -J - /mnt`

результат

``` BASH

...
xfsdump: media file size 864457912 bytes
xfsdump: dump size (non-dir files) : 851399832 bytes
xfsdump: dump complete: 24 seconds elapsed
xfsdump: Dump Status: SUCCESS
xfsrestore: restore complete: 25 seconds elapsed
xfsrestore: Restore Status: SUCCESS
[root@lvm ~]# ls -l /mnt
total 12
lrwxrwxrwx.  1 root    root       7 Feb  4 12:45 bin -> usr/bin
drwxr-xr-x.  2 root    root       6 May 12  2018 boot
drwxr-xr-x.  2 root    root       6 May 12  2018 dev
drwxr-xr-x. 77 root    root    8192 Feb  4 12:10 etc
drwxr-xr-x.  3 root    root      21 May 12  2018 home
lrwxrwxrwx.  1 root    root       7 Feb  4 12:45 lib -> usr/lib
lrwxrwxrwx.  1 root    root       9 Feb  4 12:45 lib64 -> usr/lib64
drwxr-xr-x.  2 root    root       6 Apr 11  2018 media
drwxr-xr-x.  2 root    root       6 Apr 11  2018 mnt
drwxr-xr-x.  2 root    root       6 Apr 11  2018 opt
drwxr-xr-x.  2 root    root       6 May 12  2018 proc
dr-xr-x---.  3 root    root     149 Feb  4 12:10 root
drwxr-xr-x.  2 root    root       6 May 12  2018 run
lrwxrwxrwx.  1 root    root       8 Feb  4 12:45 sbin -> usr/sbin
drwxr-xr-x.  2 root    root       6 Apr 11  2018 srv
drwxr-xr-x.  2 root    root       6 May 12  2018 sys
drwxrwxrwt.  8 root    root     193 Feb  4 12:12 tmp
drwxr-xr-x. 13 root    root     155 May 12  2018 usr
drwxrwxr-x.  2 vagrant vagrant   47 Feb  4 12:08 vagrant
drwxr-xr-x. 18 root    root     254 Feb  4 12:09 var

```
1.2.3 Переконфигурируем grub для перехода при старте в новый /
```
[root@lvm ~]# for i in /proc/ /sys/ /dev/ /run/ /boot/ ; do mount --bind $i /mnt/$i ; done
[root@lvm ~]# chroot /mnt/
[root@lvm /]# grub2-mkconfig -o /boot/grub2/grub.cfg
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-3.10.0-862.2.3.el7.x86_64
Found initrd image: /boot/initramfs-3.10.0-862.2.3.el7.x86_64.img
done
[root@lvm /]# 

```
1.2.4 Обновляем initd
```
[root@lvm /]# cd /boot ; for i in `ls initramfs-*img`; do dracut -v $i `echo $i|sed "s/initramfs-//g;s/.img//g"` --force; done
Executing: /sbin/dracut -v initramfs-3.10.0-862.2.3.el7.x86_64.img 3.10.0-862.2.3.el7.x86_64 --force
dracut module 'busybox' will not be installed, because command 'busybox' could not be found!
dracut module 'crypt' will not be installed, because command 'cryptsetup' could not be found!
dracut module 'dmraid' will not be installed, because command 'dmraid' could not be found!
dracut module 'dmsquash-live-ntfs' will not be installed, because command 'ntfs-3g' could not be found!
dracut module 'mdraid' will not be installed, because command 'mdadm' could not be found!
dracut module 'multipath' will not be installed, because command 'multipath' could not be found!
dracut module 'busybox' will not be installed, because command 'busybox' could not be found!
dracut module 'crypt' will not be installed, because command 'cryptsetup' could not be found!
dracut module 'dmraid' will not be installed, because command 'dmraid' could not be found!
dracut module 'dmsquash-live-ntfs' will not be installed, because command 'ntfs-3g' could not be found!
dracut module 'mdraid' will not be installed, because command 'mdadm' could not be found!
dracut module 'multipath' will not be installed, because command 'multipath' could not be found!
*** Including module: bash ***
*** Including module: nss-softokn ***

...

*** Generating early-microcode cpio image contents ***
*** Constructing AuthenticAMD.bin ****
*** No early-microcode cpio image needed ***
*** Store current command line parameters ***
*** Creating image file ***
*** Creating image file done ***
*** Creating initramfs image file '/boot/initramfs-3.10.0-862.2.3.el7.x86_64.img' done ***
[root@lvm boot]# 
```
В файле `/boot/grub2/grub.cfg` меняем значенеие `rd.lvm.lv=VolGroup00/LogVol00` на `rd.lvm.lv=vg_root/lv_root`

1.2.5 Перезагрузкаи проверка что загрузились с временного `/` расположенного на `/dev/mapper/vg_root-lv_root`

```
[root@lvm ~]# lsblk
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
[root@lvm ~]# 

```
#### 1.3 Уменьшение `/` до 8G.

Уменьшаем /dev/VolGroup00/LogVol00 до 8G, 
для этого удаляем LogVol00
```
[root@lvm ~]# lvremove /dev/VolGroup00/LogVol00
Do you really want to remove active logical volume VolGroup00/LogVol00? [y/n]: y
  Logical volume "LogVol00" successfully removed
[root@lvm ~]# lsblk
NAME                    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                       8:0    0   40G  0 disk 
├─sda1                    8:1    0    1M  0 part 
├─sda2                    8:2    0    1G  0 part /boot
└─sda3                    8:3    0   39G  0 part 
  └─VolGroup00-LogVol01 253:1    0  1.5G  0 lvm  [SWAP]
sdb                       8:16   0   10G  0 disk 
└─vg_root-lv_root       253:0    0   10G  0 lvm  /
sdc                       8:32   0    2G  0 disk 
sdd                       8:48   0    1G  0 disk 
sde                       8:64   0    1G  0 disk 
[root@lvm ~]# 

```
и создаем новый 8G
```
[root@lvm ~]# lvcreate -n VolGroup00/LogVolNewRoot -L 8G /dev/VolGroup00
WARNING: xfs signature detected on /dev/VolGroup00/LogVolNewRoot at offset 0. Wipe it? [y/n]: y
  Wiping xfs signature on /dev/VolGroup00/LogVolNewRoot.
  Logical volume "LogVolNewRoot" created.
[root@lvm ~]# lsblk
NAME                         MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                            8:0    0   40G  0 disk 
├─sda1                         8:1    0    1M  0 part 
├─sda2                         8:2    0    1G  0 part /boot
└─sda3                         8:3    0   39G  0 part 
  ├─VolGroup00-LogVol01      253:1    0  1.5G  0 lvm  [SWAP]
  └─VolGroup00-LogVolNewRoot 253:2    0    8G  0 lvm  
sdb                            8:16   0   10G  0 disk 
└─vg_root-lv_root            253:0    0   10G  0 lvm  /
sdc                            8:32   0    2G  0 disk 
sdd                            8:48   0    1G  0 disk 
sde                            8:64   0    1G  0 disk 
[root@lvm ~]# mkfs.xfs /dev/VolGroup00/LogVolNewRoot 
meta-data=/dev/VolGroup00/LogVolNewRoot isize=512    agcount=4, agsize=524288 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0, sparse=0
data     =                       bsize=4096   blocks=2097152, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
[root@lvm ~]# mount /dev/VolGroup00/LogVolNewRoot /mnt

[root@lvm ~]# xfsdump -J - /dev/vg_root/lv_root | xfsrestore -J - /mnt
xfsdump: using file dump (drive_simple) strategy
xfsdump: version 3.1.7 (dump format 3.0)
xfsdump: level 0 dump of lvm:/
xfsdump: dump date: Sun Feb  4 13:57:08 2024
xfsdump: session id: 62ef94ea-7b83-4840-ad7d-446b048b3fcb
xfsdump: session label: ""

...

xfsdump: ending media file
xfsdump: media file size 863129072 bytes
xfsdump: dump size (non-dir files) : 850066640 bytes
xfsdump: dump complete: 37 seconds elapsed
xfsdump: Dump Status: SUCCESS
xfsrestore: restore complete: 37 seconds elapsed
xfsrestore: Restore Status: SUCCESS
[root@lvm ~]# 
```

повторяем действия пунктов 1.2.3 и 1.2.4

В /etc/default/grub меняем значенеие `rd.lvm.lv=VolGroup00/LogVol00` на `rd.lvm.lv=VolGroup00/LogVolNewRoot`.
Это необходимо выполнить т.к. изменилось расположение / на дисковых устройствах.
Команда `grub2-mkconfig -o /boot/grub2/grub.cfg`.
```
[root@lvm ~]# lsblk
NAME                         MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                            8:0    0   40G  0 disk 
├─sda1                         8:1    0    1M  0 part 
├─sda2                         8:2    0    1G  0 part /boot
└─sda3                         8:3    0   39G  0 part 
  ├─VolGroup00-LogVolNewRoot 253:0    0    8G  0 lvm  /
  └─VolGroup00-LogVol01      253:1    0  1.5G  0 lvm  [SWAP]
sdb                            8:16   0   10G  0 disk 
└─vg_root-lv_root            253:2    0   10G  0 lvm  
sdc                            8:32   0    2G  0 disk 
sdd                            8:48   0    1G  0 disk 
sde                            8:64   0    1G  0 disk 
[root@lvm ~]# 

```
Удаляем временный /
```
[root@lvm ~]# lvremove /dev/vg_root/lv_root
Do you really want to remove active logical volume vg_root/lv_root? [y/n]: y
  Logical volume "lv_root" successfully removed
[root@lvm ~]# vgremove /dev/vg_root
  Volume group "vg_root" successfully removed
[root@lvm ~]# pvremove /dev/sdb
  Labels on physical volume "/dev/sdb" successfully wiped.
[root@lvm ~]# 

```
### 2. Выделить том под /home

Выделяем /home

```
[root@lvm ~]# pvcreate /dev/sdb
  Physical volume "/dev/sdb" successfully created.
[root@lvm ~]# vgcreate vg_home /dev/sdb
  Volume group "vg_home" successfully created

[root@lvm ~]# lvcreate -n LogVolHome -L3G /dev/vg_home
WARNING: xfs signature detected on /dev/vg_home/LogVolHome at offset 0. Wipe it? [y/n]: y
  Wiping xfs signature on /dev/vg_home/LogVolHome.
  Logical volume "LogVolHome" created.
[root@lvm ~]# 
[root@lvm ~]# mkfs.xfs /dev/vg_home/LogVolHome 
meta-data=/dev/vg_home/LogVolHome isize=512    agcount=4, agsize=196608 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0, sparse=0
data     =                       bsize=4096   blocks=786432, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
[root@lvm ~]# 

[root@lvm ~]# mount /dev/vg_home/LogVolHome /mnt/
[root@lvm ~]# 
[root@lvm ~]# cp -aR /home/* /mnt/

```

### 3. прописать монтирование в fstab (попробовать с разными опциями и разными файловыми системами на выбор)
```
echo "`blkid | grep Home | awk '{print $2}'` /home xfs defaults 0 0" >> /etc/fstab
```
после этого в /etc/fstab появиться строка 
`UUID="296ba315-cac8-4680-afa9-7c42767155b6" /home xfs defaults 0 0`

### 4. для /home - сделать том для снэпшотов

## Работа со снапшотами:

### 5. сгенерировать файлы в /home/
```
[root@lvm ~]# touch /home/file{1..40}
[root@lvm ~]# ls /home/
file1   file12  file15  file18  file20  file23  file26  file29  file31  file34  file37  file4   file6  file9
file10  file13  file16  file19  file21  file24  file27  file3   file32  file35  file38  file40  file7  vagrant
file11  file14  file17  file2   file22  file25  file28  file30  file33  file36  file39  file5   file8
[root@lvm ~]# 
```
### 6. для /home - сделать том для снэпшотов, снять снэпшот

```
[root@lvm ~]# lvcreate -L 200MB -s -n home_snap /dev/vg_home/LogVolHome
  Logical volume "home_snap" created.
[root@lvm ~]# 
[root@lvm ~]# lvs
  LV            VG         Attr       LSize   Pool Origin     Data%  Meta%  Move Log Cpy%Sync Convert
  LogVol01      VolGroup00 -wi-ao----   1.50g                                                        
  LogVolNewRoot VolGroup00 -wi-ao----   8.00g                                                        
  LogVolHome    vg_home    owi-aos---   3.00g                                                        
  home_snap     vg_home    swi-a-s--- 200.00m      LogVolHome 0.01   
```
проверяем
```
[root@lvm ~]# mount -o nouuid /dev/mapper/vg_home-home_snap /mnt/snapshot
[root@lvm ~]# ls /mnt/snapshot/
file1   file12  file15  file18  file20  file23  file26  file29  file31  file34  file37  file4   file6  file9
file10  file13  file16  file19  file21  file24  file27  file3   file32  file35  file38  file40  file7  vagrant
file11  file14  file17  file2   file22  file25  file28  file30  file33  file36  file39  file5   file8
[root@lvm ~]# 
```
### 7. удалить часть файлов
```
[root@lvm ~]# rm -f /home/file{11..20}
[root@lvm ~]# ls /home/
file1   file2   file22  file24  file26  file28  file3   file31  file33  file35  file37  file39  file40  file6  file8  vagrant
file10  file21  file23  file25  file27  file29  file30  file32  file34  file36  file38  file4   file5   file7  file9
[root@lvm ~]# 

```
### 8. восстанавливаем данные из снэпшота
```
[root@lvm ~]# umount /home/
[root@lvm ~]# lvconvert --merge /dev/vg_home/home_snap 
  Merging of volume vg_home/home_snap started.
  vg_home/LogVolHome: Merged: 100.00%
[root@lvm ~]# mount /home/
[root@lvm ~]# ls /home/
file1   file12  file15  file18  file20  file23  file26  file29  file31  file34  file37  file4   file6  file9
file10  file13  file16  file19  file21  file24  file27  file3   file32  file35  file38  file40  file7  vagrant
file11  file14  file17  file2   file22  file25  file28  file30  file33  file36  file39  file5   file8
[root@lvm ~]# 
```

### 9. выделить том под /var (/var - сделать в mirror)

Создаем зеркало для /var на диска sdd sde
```
[root@lvm ~]# lsblk
NAME                         MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                            8:0    0   40G  0 disk 
├─sda1                         8:1    0    1M  0 part 
├─sda2                         8:2    0    1G  0 part /boot
└─sda3                         8:3    0   39G  0 part 
  ├─VolGroup00-LogVolNewRoot 253:0    0    8G  0 lvm  /
  └─VolGroup00-LogVol01      253:1    0  1.5G  0 lvm  [SWAP]
sdb                            8:16   0   10G  0 disk 
└─vg_home-LogVolHome         253:2    0    3G  0 lvm  /home
sdc                            8:32   0    2G  0 disk 
sdd                            8:48   0    1G  0 disk 
sde                            8:64   0    1G  0 disk 
```
Определяем физический том из 2-х дисков
```
[root@lvm ~]# pvcreate /dev/sdd /dev/sde
  Physical volume "/dev/sdd" successfully created.
  Physical volume "/dev/sde" successfully created.
```
Определяем группу `vg_var` физических томов
```
[root@lvm ~]# vgcreate vg_var /dev/sdd /dev/sde
  Volume group "vg_var" successfully created
```  
 Определяем том `lv_var`
 ```
[root@lvm ~]# lvcreate -L 950M -m1 -n lv_var vg_var
  Rounding up size to full physical extent 952.00 MiB
  Logical volume "lv_var" created.
[root@lvm ~]# 
```
Создаем файловую систему `exp4` на lv_var, затем монтируем в /mnt/
```
[root@lvm ~]# mkfs.ext4 /dev/vg_var/lv_var
mke2fs 1.42.9 (28-Dec-2013)
Filesystem label=
OS type: Linux
Block size=4096 (log=2)
Fragment size=4096 (log=2)
Stride=0 blocks, Stripe width=0 blocks
60928 inodes, 243712 blocks
12185 blocks (5.00%) reserved for the super user
First data block=0
Maximum filesystem blocks=249561088
8 block groups
32768 blocks per group, 32768 fragments per group
7616 inodes per group
Superblock backups stored on blocks: 
	32768, 98304, 163840, 229376

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (4096 blocks): done
Writing superblocks and filesystem accounting information: done

[root@lvm ~]#
[root@lvm ~]# mount /dev/vg_var/lv_var /mnt

```
перемещаем данных с /var в /mnt/
```
[root@lvm ~]# rsync -avHPSAX /var/ /mnt/
...
sent 266,231,003 bytes  received 259,782 bytes  18,378,674.83 bytes/sec
total size is 265,764,479  speedup is 1.00
[root@lvm ~]
[root@lvm ~]# sudo rm -rf /var/*
rm: cannot remove ‘/var/lib/nfs/rpc_pipefs/gssd/clntXX/gssd’: Operation not permitted
rm: cannot remove ‘/var/lib/nfs/rpc_pipefs/gssd/clntXX/info’: Operation not permitted
rm: cannot remove ‘/var/lib/nfs/rpc_pipefs/nfsd’: Operation not permitted
rm: cannot remove ‘/var/lib/nfs/rpc_pipefs/cache’: Operation not permitted
rm: cannot remove ‘/var/lib/nfs/rpc_pipefs/nfsd4_cb’: Operation not permitted
rm: cannot remove ‘/var/lib/nfs/rpc_pipefs/statd’: Operation not permitted
rm: cannot remove ‘/var/lib/nfs/rpc_pipefs/portmap’: Operation not permitted
rm: cannot remove ‘/var/lib/nfs/rpc_pipefs/nfs’: Operation not permitted
rm: cannot remove ‘/var/lib/nfs/rpc_pipefs/mount’: Operation not permitted
rm: cannot remove ‘/var/lib/nfs/rpc_pipefs/lockd’: Operation not permitted
rm: cannot remove ‘/var/tmp’: Device or resource busy
[root@lvm ~]# umount /mnt
[root@lvm ~]# mount /dev/vg_var/lv_var /var
[root@lvm ~]# 
```
В /etc/fstab вносим данные для монтирования /var
```
[root@lvm ~]# echo "`blkid | grep var: | awk '{print $2}'` /var ext4 defaults 0 0" >> /etc/fstab
```
перезагружаем, проверяем монтирование

```
[root@lvm ~]# lsblk
NAME                         MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                            8:0    0   40G  0 disk 
├─sda1                         8:1    0    1M  0 part 
├─sda2                         8:2    0    1G  0 part /boot
└─sda3                         8:3    0   39G  0 part 
  ├─VolGroup00-LogVolNewRoot 253:0    0    8G  0 lvm  /
  └─VolGroup00-LogVol01      253:1    0  1.5G  0 lvm  [SWAP]
sdb                            8:16   0   10G  0 disk 
└─vg_home-LogVolHome         253:5    0    3G  0 lvm  /home
sdc                            8:32   0    2G  0 disk 
sdd                            8:48   0    1G  0 disk 
├─vg_var-lv_var_rmeta_0      253:2    0    4M  0 lvm  
│ └─vg_var-lv_var            253:7    0  952M  0 lvm  /var
└─vg_var-lv_var_rimage_0     253:3    0  952M  0 lvm  
  └─vg_var-lv_var            253:7    0  952M  0 lvm  /var
sde                            8:64   0    1G  0 disk 
├─vg_var-lv_var_rmeta_1      253:4    0    4M  0 lvm  
│ └─vg_var-lv_var            253:7    0  952M  0 lvm  /var
└─vg_var-lv_var_rimage_1     253:6    0  952M  0 lvm  
  └─vg_var-lv_var            253:7    0  952M  0 lvm  /var
[root@lvm ~]# 

[root@lvm ~]# df -Th
Filesystem                           Type      Size  Used Avail Use% Mounted on
/dev/mapper/VolGroup00-LogVolNewRoot xfs       8.0G  625M  7.4G   8% /
devtmpfs                             devtmpfs  110M     0  110M   0% /dev
tmpfs                                tmpfs     118M     0  118M   0% /dev/shm
tmpfs                                tmpfs     118M  4.6M  114M   4% /run
tmpfs                                tmpfs     118M     0  118M   0% /sys/fs/cgroup
/dev/mapper/vg_home-LogVolHome       xfs       3.0G   33M  3.0G   2% /home
/dev/mapper/vg_var-lv_var            ext4      922M  262M  596M  31% /var
/dev/sda2                            xfs      1014M   61M  954M   6% /boot
tmpfs                                tmpfs      24M     0   24M   0% /run/user/1000
[root@lvm ~]#
```
Полный лог находиться в файле [dz4.log](dz4.log)

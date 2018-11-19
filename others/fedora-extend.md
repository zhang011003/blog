# fedora扩容

一直在VirtualBox下使用linux，遇到了磁盘容量不足的情况，几经周折，终于扩容成功了。记录一下扩容的历程

```bash
[zhang011003@localhost ~]$ sudo fdisk /dev/sda
[sudo] password for zhangyu: 

Welcome to fdisk (util-linux 2.32).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.


Command (m for help): m

Help:

  DOS (MBR)
   a   toggle a bootable flag
   b   edit nested BSD disklabel
   c   toggle the dos compatibility flag

  Generic
   d   delete a partition
   F   list free unpartitioned space
   l   list known partition types
   n   add a new partition
   p   print the partition table
   t   change a partition type
   v   verify the partition table
   i   print information about a partition

  Misc
   m   print this menu
   u   change display/entry units
   x   extra functionality (experts only)

  Script
   I   load disk layout from sfdisk script file
   O   dump disk layout to sfdisk script file

  Save & Exit
   w   write table to disk and exit
   q   quit without saving changes

  Create a new label
   g   create a new empty GPT partition table
   G   create a new empty SGI (IRIX) partition table
   o   create a new empty DOS partition table
   s   create a new empty Sun partition table


Command (m for help): p   
Disk /dev/sda: 20 GiB, 21474836480 bytes, 41943040 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x4dd44768

Device     Boot   Start      End  Sectors Size Id Type
/dev/sda1  *       2048  2099199  2097152   1G 83 Linux
/dev/sda2       2099200 20971519 18872320   9G 8e Linux LVM

Command (m for help): n
Partition type
   p   primary (2 primary, 0 extended, 2 free)
   e   extended (container for logical partitions)
Select (default p): p
Partition number (3,4, default 3): 
First sector (20971520-41943039, default 20971520): 
Last sector, +sectors or +size{K,M,G,T,P} (20971520-41943039, default 41943039): 

Created a new partition 3 of type 'Linux' and of size 10 GiB.
Partition #3 contains a ext4 signature.

Do you want to remove the signature? [Y]es/[N]o: y

The signature will be removed by a write command.

Command (m for help): p
Disk /dev/sda: 20 GiB, 21474836480 bytes, 41943040 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x4dd44768

Device     Boot    Start      End  Sectors Size Id Type
/dev/sda1  *        2048  2099199  2097152   1G 83 Linux
/dev/sda2        2099200 20971519 18872320   9G 8e Linux LVM
/dev/sda3       20971520 41943039 20971520  10G 83 Linux

Filesystem/RAID signature on partition 3 will be wiped.


Command (m for help): w
The partition table has been altered.
Syncing disks.

[zhangyu@localhost ~]$ df -h
Filesystem                               Size  Used Avail Use% Mounted on
devtmpfs                                 2.0G     0  2.0G   0% /dev
tmpfs                                    2.0G     0  2.0G   0% /dev/shm
tmpfs                                    2.0G  1.4M  2.0G   1% /run
tmpfs                                    2.0G     0  2.0G   0% /sys/fs/cgroup
/dev/mapper/fedora_localhost--live-root  7.9G  7.2G  204M  98% /
tmpfs                                    2.0G   72K  2.0G   1% /tmp
/dev/sda1                                976M  150M  759M  17% /boot
D_DRIVE                                  932G   43G  889G   5% /media/sf_D_DRIVE
tmpfs                                    395M  8.0M  387M   3% /run/user/1000
/dev/sr0                                  56M   56M     0 100% /run/media/zhangyu/VBox_GAs_5.2.8

[zhangyu@localhost ~]$ sudo fdisk -l
[sudo] password for zhangyu: 
Disk /dev/sda: 20 GiB, 21474836480 bytes, 41943040 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x4dd44768

Device     Boot    Start      End  Sectors Size Id Type
/dev/sda1  *        2048  2099199  2097152   1G 83 Linux
/dev/sda2        2099200 20971519 18872320   9G 8e Linux LVM
/dev/sda3       20971520 41943039 20971520  10G 83 Linux


Disk /dev/mapper/fedora_localhost--live-root: 8 GiB, 8585740288 bytes, 16769024 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/mapper/fedora_localhost--live-swap: 1 GiB, 1073741824 bytes, 2097152 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes

[zhangyu@localhost ~]$ sudo pvcreate /dev/sda3
  Physical volume "/dev/sda3" successfully created.

[zhangyu@localhost ~]$ sudo vgdisplay 
  --- Volume group ---
  VG Name               fedora_localhost-live
  System ID             
  Format                lvm2
  Metadata Areas        1
  Metadata Sequence No  3
  VG Access             read/write
  VG Status             resizable
  MAX LV                0
  Cur LV                2
  Open LV               2
  Max PV                0
  Cur PV                1
  Act PV                1
  VG Size               <9.00 GiB
  PE Size               4.00 MiB
  Total PE              2303
  Alloc PE / Size       2303 / <9.00 GiB
  Free  PE / Size       0 / 0   
  VG UUID               xAtGSL-culJ-bFM1-xZrA-pQzA-Gien-mwYpXY
   

[zhangyu@localhost ~]$ sudo vgextend /dev/mapper/fedora_localhost--live-root /dev/sda3
  Volume group name "fedora_localhost-live/root" has invalid characters.
  Cannot process volume group fedora_localhost-live/root
   
[zhangyu@localhost ~]$ df -h
Filesystem                               Size  Used Avail Use% Mounted on
devtmpfs                                 2.0G     0  2.0G   0% /dev
tmpfs                                    2.0G   32M  1.9G   2% /dev/shm
tmpfs                                    2.0G  1.4M  2.0G   1% /run
tmpfs                                    2.0G     0  2.0G   0% /sys/fs/cgroup
/dev/mapper/fedora_localhost--live-root  7.9G  7.3G  179M  98% /
tmpfs                                    2.0G   72K  2.0G   1% /tmp
/dev/sda1                                976M  150M  759M  17% /boot
D_DRIVE                                  932G   43G  889G   5% /media/sf_D_DRIVE
tmpfs                                    395M   11M  384M   3% /run/user/1000
/dev/sr0                                  56M   56M     0 100% /run/media/zhangyu/VBox_GAs_5.2.8

[zhangyu@localhost ~]$ sudo vgextend /dev/mapper/fedora_localhost--live /dev/sda3
  Volume group "fedora_localhost-live" successfully extended
   
[zhangyu@localhost ~]$ sudo pvdisplay
  --- Physical volume ---
  PV Name               /dev/sda2
  VG Name               fedora_localhost-live
  PV Size               <9.00 GiB / not usable 3.00 MiB
  Allocatable           yes (but full)
  PE Size               4.00 MiB
  Total PE              2303
  Free PE               0
  Allocated PE          2303
  PV UUID               Q2qo9C-eX8C-JOf0-wqom-kFU8-m5kS-wIy7Um
   
  --- Physical volume ---
  PV Name               /dev/sda3
  VG Name               fedora_localhost-live
  PV Size               10.00 GiB / not usable 4.00 MiB
  Allocatable           yes 
  PE Size               4.00 MiB
  Total PE              2559
  Free PE               2559
  Allocated PE          0
  PV UUID               HTk1Nt-J0X5-qwir-pcna-G9x6-MH5P-cqKXyu
   
[zhangyu@localhost ~]$ sudo lvdisplay
  --- Logical volume ---
  LV Path                /dev/fedora_localhost-live/swap
  LV Name                swap
  VG Name                fedora_localhost-live
  LV UUID                12TNT1-gMov-kx7Z-ZqHh-DGDP-OyK3-UAxLsI
  LV Write Access        read/write
  LV Creation host, time localhost-live, 2018-09-27 14:08:28 +0800
  LV Status              available
  # open                 2
  LV Size                1.00 GiB
  Current LE             256
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     256
  Block device           253:1
   
  --- Logical volume ---
  LV Path                /dev/fedora_localhost-live/root
  LV Name                root
  VG Name                fedora_localhost-live
  LV UUID                D2GeGT-p4Gt-LFUK-iop0-lBQ2-kxc9-7oS5Ql
  LV Write Access        read/write
  LV Creation host, time localhost-live, 2018-09-27 14:08:29 +0800
  LV Status              available
  # open                 1
  LV Size                <8.00 GiB
  Current LE             2047
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     256
  Block device           253:0
   
[zhangyu@localhost ~]$ sudo lvextend -L+11G /dev/fedora_localhost-live/root
  Insufficient free space: 2816 extents needed, but only 2559 available

[zhangyu@localhost ~]$ sudo lvextend /dev/fedora_localhost-live/root /dev/sda3
  Size of logical volume fedora_localhost-live/root changed from <8.00 GiB (2047 extents) to 17.99 GiB (4606 extents).
  Logical volume fedora_localhost-live/root successfully resized.
[zhangyu@localhost ~]$ df -h
Filesystem                               Size  Used Avail Use% Mounted on
devtmpfs                                 2.0G     0  2.0G   0% /dev
tmpfs                                    2.0G   42M  1.9G   3% /dev/shm
tmpfs                                    2.0G  1.4M  2.0G   1% /run
tmpfs                                    2.0G     0  2.0G   0% /sys/fs/cgroup
/dev/mapper/fedora_localhost--live-root  7.9G  7.3G  178M  98% /
tmpfs                                    2.0G   72K  2.0G   1% /tmp
/dev/sda1                                976M  150M  759M  17% /boot
D_DRIVE                                  932G   43G  889G   5% /media/sf_D_DRIVE
tmpfs                                    395M   13M  382M   4% /run/user/1000
/dev/sr0                                  56M   56M     0 100% /run/media/zhangyu/VBox_GAs_5.2.8

[zhangyu@localhost ~]$ sudo resize2fs /dev/fedora_localhost-live/root
resize2fs 1.43.8 (1-Jan-2018)
Filesystem at /dev/fedora_localhost-live/root is mounted on /; on-line resizing required
old_desc_blocks = 1, new_desc_blocks = 3
The filesystem on /dev/fedora_localhost-live/root is now 4716544 (4k) blocks long.

[zhangyu@localhost ~]$ df -h
Filesystem                               Size  Used Avail Use% Mounted on
devtmpfs                                 2.0G     0  2.0G   0% /dev
tmpfs                                    2.0G   36M  1.9G   2% /dev/shm
tmpfs                                    2.0G  1.4M  2.0G   1% /run
tmpfs                                    2.0G     0  2.0G   0% /sys/fs/cgroup
/dev/mapper/fedora_localhost--live-root   18G  7.3G  9.7G  43% /
tmpfs                                    2.0G   72K  2.0G   1% /tmp
/dev/sda1                                976M  150M  759M  17% /boot
D_DRIVE                                  932G   43G  889G   5% /media/sf_D_DRIVE
tmpfs                                    395M   13M  382M   4% /run/user/1000
/dev/sr0                                  56M   56M     0 100% /run/media/zhangyu/VBox_GAs_5.2.8
```

 # Домашняя работа "Дисковая подсистема" #

Задачи:  

**-    добавить в Vagrantfile еще дисков;**  
**-    сломать/починить raid;**  
**-    собрать R0/R5/R10 на выбор;**  
**-    прописать собранный рейд в конф, чтобы рейд собирался при загрузке;**  
**-    создать GPT раздел и 5 партиций.**  

Добавляем в Vagrantfile дополнтельные диски:  
Vagrantfile находится в папке /disks, для ручной сборки RAID комментируем раздел SHELL, в котором описано создание RAID'a и монтирование разделов

Поднимаем виртуальную машину, подключаемся к ней и вызываем режим суперпользователя

      $vagrant up  
      $vagrant ssh  
      $sudo -i  

проверяем наличие добавленных дисков в системе  

    [root@localhost ~]# lsscsi  

      [0:0:0:0]    disk    ATA      VBOX HARDDISK    1.0   /dev/sda 
      [3:0:0:0]    disk    ATA      VBOX HARDDISK    1.0   /dev/sdb 
      [4:0:0:0]    disk    ATA      VBOX HARDDISK    1.0   /dev/sdc 
      [5:0:0:0]    disk    ATA      VBOX HARDDISK    1.0   /dev/sdd 
      [6:0:0:0]    disk    ATA      VBOX HARDDISK    1.0   /dev/sde 
      [7:0:0:0]    disk    ATA      VBOX HARDDISK    1.0   /dev/sdf 

устанавливаем mdadm  

    [root@localhost ~]# yum install -y mdadm

обнуляем суперблоки  

    [root@localhost ~]# mdadm --zero-superblock --force /dev/sd{b,c,d,e,f}
      mdadm: Unrecognised md component device - /dev/sdb
      mdadm: Unrecognised md component device - /dev/sdc
      mdadm: Unrecognised md component device - /dev/sdd
      mdadm: Unrecognised md component device - /dev/sde
      mdadm: Unrecognised md component device - /dev/sdf
 *сообщение ошибкой не является, просто уведомление, что диски ранее не использовались для RAID*

удаляем метаданные с дисков  

    [root@localhost ~]# wipefs --all --force /dev/sd{b,c,d,e,f}

создаём RAID5  

    [root@localhost ~]# mdadm --create --verbose /dev/md0 -l 5 -n 5 /dev/sd{b,c,d,e,f}
      mdadm: layout defaults to left-symmetric
      mdadm: layout defaults to left-symmetric
      mdadm: chunk size defaults to 512K
      mdadm: size set to 1046528K
      mdadm: Defaulting to version 1.2 metadata
      mdadm: array /dev/md0 started.

проверяем состояние RAID  

    [root@localhost ~]# cat /proc/mdstat
      Personalities : [raid6] [raid5] [raid4] 
      md0 : active raid5 sdf[5] sde[3] sdd[2] sdc[1] sdb[0]
       4186112 blocks super 1.2 level 5, 512k chunk, algorithm 2 [5/5] [UUUUU]
      
      unused devices: <none>

"выводим из строя" один из дисков:  

    [root@localhost ~]# mdadm /dev/md0 --fail /dev/sdc
      mdadm: set /dev/sdc faulty in /dev/md0

смотрим статус RAID
    [root@localhost ~]# cat /proc/mdstat
      Personalities : [raid6] [raid5] [raid4] 
      md0 : active raid5 sdf[5] sde[3] sdd[2] sdc[1](F) sdb[0]
            4186112 blocks super 1.2 level 5, 512k chunk, algorithm 2 [5/4] [U_UUU]
      
      unused devices: <none>
*диск №2, sdc, показан как (F)*

"Заменим" "неисправный" диск  

    [root@localhost ~]# mdadm /dev/md0 --remove /dev/sdc
      mdadm: hot removed /dev/sdc from /dev/md0

    [root@localhost ~]#cat /proc/mdstat
      Personalities : [raid6] [raid5] [raid4] 
      md0 : active raid5 sdf[5] sde[3] sdd[2] sdb[0]
            4186112 blocks super 1.2 level 5, 512k chunk, algorithm 2 [5/4] [U_UUU]
      
      unused devices: <none>
*не видим упоминаний о диске /dev/sdc*

    [root@localhost ~]# mdadm /dev/md0 --add /dev/sdc
      mdadm: added /dev/sdc

можем наблюдать ребилд RAID (с маленькими дисками проходит очень быстро)
    [root@localhost ~]# cat /proc/mdstat

      Personalities : [raid6] [raid5] [raid4] 
      md0 : active raid5 sdc[6] sdf[5] sde[3] sdd[2] sdb[0]
            4186112 blocks super 1.2 level 5, 512k chunk, algorithm 2 [5/4] [U_UUU]
            [=====>...............]  recovery = 26.4% (277120/1046528) finish=0.1min speed=69280K/sec
      
      unused devices: <none>

создаём конфигурационный файл, чтобы RAID собирался при загрузке системы  

    [root@localhost ~]# mkdir /etc/mdadm
    [root@localhost ~]# cho "DEVICE partitions" > /etc/mdadm/mdadm.conf
    [root@localhost ~]# dadm --detail --scan --verbose | awk '/ARRAY/ {print}' >> /etc/mdadm/mdadm.conf

проверяем получившийся файл  

    [root@localhost ~]# сat /etc/mdadm/mdadm.conf 
      DEVICE partitions
      ARRAY /dev/md0 level=raid5 num-devices=5 metadata=1.2 name=localhost.localdomain:0 UUID=ffc8c5f3:3abffe4c:bcd6c8e1:6defaf0f
*можно перезагрузить машину, убедиться, что RAID поднялся после загрузки*

 размечаем диск:
 - создаём таблицу GPT
    [root@localhost ~]# parted -s /dev/md0 mklabel gpt
 - разделы
    [root@localhost ~]# parted /dev/md0 mkpart primary ext4 0% 20%
    [root@localhost ~]# parted /dev/md0 mkpart primary ext4 20% 40% 
    [root@localhost ~]# parted /dev/md0 mkpart primary ext4 40% 60%  
    [root@localhost ~]# parted /dev/md0 mkpart primary ext4 60% 80%          
    [root@localhost ~]# parted /dev/md0 mkpart primary ext4 80% 100%     

проверяем получившееся  

    [root@localhost ~]#fdisk -l

      ...
      Disk /dev/md0: 4286 MB, 4286578688 bytes, 8372224 sectors
      Units = sectors of 1 * 512 = 512 bytes
      Sector size (logical/physical): 512 bytes / 512 bytes
      I/O size (minimum/optimal): 524288 bytes / 2097152 bytes
      Disk label type: gpt
      Disk identifier: C343BE3C-A44C-458F-A9DC-28ADA488117D


      #         Start          End    Size  Type            Name
       1         4096      1675263    816M  Microsoft basic primary
       2      1675264      3350527    818M  Microsoft basic primary
       3      3350528      5021695    816M  Microsoft basic primary
       4      5021696      6696959    818M  Microsoft basic primary
       5      6696960      8368127    816M  Microsoft basic primary

создаём файловые системы  

    [root@localhost ~]# for i in $(seq 1 5); do sudo mkfs.ext3 /dev/md0p$i; done

 и монтируем их  
 
    [root@localhost ~]# mkdir /media/disk{1,2,3,4,5}
    [root@localhost ~]# for i in $(seq 1 5); do mount /dev/md0p$i /media/disk$i; done

проверяем что получилось 

    [root@localhost ~]# lsblk  

      NAME      MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINT
      sda         8:0    0   40G  0 disk  
      `-sda1      8:1    0   40G  0 part  /
      sdb         8:16   0    1G  0 disk  
      `-md0       9:0    0    4G  0 raid5 
        |-md0p1 259:0    0  816M  0 md    /media/disk1
        |-md0p2 259:1    0  818M  0 md    /media/disk2
        |-md0p3 259:2    0  816M  0 md    /media/disk3
        |-md0p4 259:3    0  818M  0 md    /media/disk4
        `-md0p5 259:4    0  816M  0 md    /media/disk5
      sdc         8:32   0    1G  0 disk  
      `-md0       9:0    0    4G  0 raid5 
        |-md0p1 259:0    0  816M  0 md    /media/disk1
        |-md0p2 259:1    0  818M  0 md    /media/disk2
        |-md0p3 259:2    0  816M  0 md    /media/disk3
        |-md0p4 259:3    0  818M  0 md    /media/disk4
        `-md0p5 259:4    0  816M  0 md    /media/disk5
      sdd         8:48   0    1G  0 disk  
      `-md0       9:0    0    4G  0 raid5 
        |-md0p1 259:0    0  816M  0 md    /media/disk1
        |-md0p2 259:1    0  818M  0 md    /media/disk2
        |-md0p3 259:2    0  816M  0 md    /media/disk3
        |-md0p4 259:3    0  818M  0 md    /media/disk4
        `-md0p5 259:4    0  816M  0 md    /media/disk5
      sde         8:64   0    1G  0 disk  
      `-md0       9:0    0    4G  0 raid5 
        |-md0p1 259:0    0  816M  0 md    /media/disk1
        |-md0p2 259:1    0  818M  0 md    /media/disk2
        |-md0p3 259:2    0  816M  0 md    /media/disk3
        |-md0p4 259:3    0  818M  0 md    /media/disk4
        `-md0p5 259:4    0  816M  0 md    /media/disk5
      sdf         8:80   0    1G  0 disk  
      `-md0       9:0    0    4G  0 raid5 
        |-md0p1 259:0    0  816M  0 md    /media/disk1
        |-md0p2 259:1    0  818M  0 md    /media/disk2
        |-md0p3 259:2    0  816M  0 md    /media/disk3
        |-md0p4 259:3    0  818M  0 md    /media/disk4
        `-md0p5 259:4    0  816M  0 md    /media/disk5

Вывод: RAID работает и автоматически собирается после загрузки системы.

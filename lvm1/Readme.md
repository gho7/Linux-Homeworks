# Домашнее задание #
На виртуальной машине centos/7 -v. 1804.2

1) Уменьшить том под / до 8G
2) Выделить том под /home
3) Выделить том под /var - сделать в mirror
4) /home - сделать том для снапшотов
5) Прописать монтирование в fstab. Попробовать с разными опциями и разными файловыми системами (на выбор)

Работа со снапшотами:
- сгенерить файлы в /home/
- снять снапшот
- удалить часть файлов
- восстановиться со снапшота


В файле Vagrant прописана предварительная установка пакетов: 

        yum install -y mdadm smartmontools hdparm gdisk xfsdump
       
Поднимаем виртуальную машину, подключаемся к ней, переключаемся в sudo:  

    $ vagrant up
    ...
    lvm: 
    lvm: Complete!

    $ vagrant ssh
    [vagrant@lvm ~]$ sudo -i 
    [root@lvm ~]#  
    
Уменьшаем том / до 8G
======================

Создаём, размечаем и монтируем том для / раздела:  

    [root@lvm ~]# pvcreate /dev/sdb
      Physical volume "/dev/sdb" successfully created. 
    
    [root@lvm ~]# vgcreate vg_root /dev/sdb
      Volume group "vg_root" successfully created
      
    [root@lvm ~]# lvcreate -n lv_root -l +100%FREE /dev/vg_root
      Logical volume "lv_root" created.
     
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
      
    [root@lvm ~]# mount /dev/vg_root/lv_root /mnt 
    
С помощью xfsdump/xfsrestore копируем данные с / в /mnt: 
    
    [root@lvm ~]# [root@lvm ~]# xfsdump -J - /dev/VolGroup00/LogVol00 | xfsrestore -J - /mnt
      ...
      xfsdump: dump complete: 15 seconds elapsed
      xfsdump: Dump Status: SUCCESS
      xfsrestore: restore complete: 15 seconds elapsed
      xfsrestore: Restore Status: SUCCESS
      
Убеждаемся, что команда сработала: 
        
        [root@lvm ~]# ll /mnt
            total 12
            lrwxrwxrwx.  1 root    root       7 Dec 21 16:27 bin -> usr/bin
            drwxr-xr-x.  2 root    root       6 May 12  2018 boot
            drwxr-xr-x.  2 root    root       6 May 12  2018 dev
            drwxr-xr-x. 79 root    root    8192 Dec 21 15:53 etc
            drwxr-xr-x.  3 root    root      21 May 12  2018 home
            lrwxrwxrwx.  1 root    root       7 Dec 21 16:27 lib -> usr/lib
            lrwxrwxrwx.  1 root    root       9 Dec 21 16:27 lib64 -> usr/lib64
            drwxr-xr-x.  2 root    root       6 Apr 11  2018 media
            drwxr-xr-x.  2 root    root       6 Apr 11  2018 mnt
            drwxr-xr-x.  2 root    root       6 Apr 11  2018 opt
            drwxr-xr-x.  2 root    root       6 May 12  2018 proc
            dr-xr-x---.  3 root    root     149 Dec 21 15:52 root
            drwxr-xr-x.  2 root    root       6 May 12  2018 run
            lrwxrwxrwx.  1 root    root       8 Dec 21 16:27 sbin -> usr/sbin
            drwxr-xr-x.  2 root    root       6 Apr 11  2018 srv
            drwxr-xr-x.  2 root    root       6 May 12  2018 sys
            drwxrwxrwt.  8 root    root     193 Dec 21 16:21 tmp
            drwxr-xr-x. 13 root    root     155 May 12  2018 usr
            drwxrwxr-x.  2 vagrant vagrant   25 Dec 21 15:36 vagrant
            drwxr-xr-x. 18 root    root     254 Dec 21 15:51 var
            
Вносим изменения в grub, чтобы при старте подхватывался новый раздел / :

    [root@lvm ~]# for i in /proc/ /sys/ /dev/ /run/ /boot/; do mount --bind $i /mnt/$i; done
    [root@lvm ~]# chroot /mnt/
    [root@lvm /]# grub2-mkconfig -o /boot/grub2/grub.cfg
      Generating grub configuration file ...
      Found linux image: /boot/vmlinuz-3.10.0-862.2.3.el7.x86_64
      Found initrd image: /boot/initramfs-3.10.0-862.2.3.el7.x86_64.img
      done
    



    
    

      

     
  
     
    

    
    



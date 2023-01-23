# Домашняя работа ZFS

Поднимаем виртуальную машину, авторизуемся, переходим в root:

	vagrant up
	vagrant ssh
	sudo -i

## Определение алгоритма с наилучшим сжатием

Смотрим список дисков, имеющихся в виртуальной машине:

	[root@localhost ~]# lsblk
		NAME MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
		sda 8:0 0 40G 0 disk
		└─sda1 8:1 0 40G 0 part /
		sdb 8:16 0 512M 0 disk
		sdc 8:32 0 512M 0 disk
		sdd 8:48 0 512M 0 disk
		sde 8:64 0 512M 0 disk
		sdf 8:80 0 512M 0 disk
		sdg 8:96 0 512M 0 disk
		sdh 8:112 0 512M 0 disk
		sdi 8:128 0 512M 0 disk

Создаём несколько пулов для тестирования:

	[root@localhost ~]# zpool create pool1 mirror /dev/sdb /dev/sdc
	[root@localhost ~]# zpool create pool2 mirror /dev/sdd /dev/sde
	[root@localhost ~]# zpool create pool3 mirror /dev/sdf /dev/sdg
	[root@localhost ~]# zpool create pool4 mirror /dev/sdh /dev/sdi

Смотрим, что получилось:

	[root@localhost ~]# zpool list
		NAME    SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
		pool1   480M  91.5K   480M        -         -     0%     0%  1.00x    ONLINE  -
		pool2   480M  91.5K   480M        -         -     0%     0%  1.00x    ONLINE  -
		pool3   480M  91.5K   480M        -         -     0%     0%  1.00x    ONLINE  -
		pool4   480M  91.5K   480M        -         -     0%     0%  1.00x    ONLINE  -


	[root@localhost ~]# zpool status
		  pool: pool1
		 state: ONLINE
		  scan: none requested
		config:

			NAME        STATE     READ WRITE CKSUM
			pool1       ONLINE       0     0     0
			  mirror-0  ONLINE       0     0     0
			    sdb     ONLINE       0     0     0
			    sdc     ONLINE       0     0     0

		errors: No known data errors

		  pool: pool2
		 state: ONLINE
		  scan: none requested
		config:

			NAME        STATE     READ WRITE CKSUM
			pool2       ONLINE       0     0     0
			  mirror-0  ONLINE       0     0     0
			    sdd     ONLINE       0     0     0
			    sde     ONLINE       0     0     0

		errors: No known data errors

		  pool: pool3
		 state: ONLINE
		  scan: none requested
		config:

			NAME        STATE     READ WRITE CKSUM
			pool3       ONLINE       0     0     0
			  mirror-0  ONLINE       0     0     0
			    sdf     ONLINE       0     0     0
			    sdg     ONLINE       0     0     0

		errors: No known data errors

		  pool: pool4
		 state: ONLINE
		  scan: none requested
		config:

			NAME        STATE     READ WRITE CKSUM
			pool4       ONLINE       0     0     0
			  mirror-0  ONLINE       0     0     0
			    sdh     ONLINE       0     0     0
			    sdi     ONLINE       0     0     0

		errors: No known data errors

Определяем различные алгоритмы сжатия для разных пулов. Они применятся для новых файлов.

	[root@localhost ~]# zfs set compression=lzjb pool1
	[root@localhost ~]# zfs set compression=lz4 pool2
	[root@localhost ~]# zfs set compression=gzip-9 pool3
	[root@localhost ~]# zfs set compression=zle pool4

Проверяем:

	[root@localhost ~]# zfs get compression
		NAME   PROPERTY     VALUE     SOURCE
		pool1  compression  lzjb      local
		pool2  compression  lz4       local
		pool3  compression  gzip-9    local
		pool4  compression  zle       local
	
Скачаем в пулы тестовый файл:

	[root@localhost ~]# for i in {1..4}; do wget -P /pool$i https://gutenberg.org/cache/epub/2600/pg2600.converter.log; done
	
		--2023-01-23 07:41:29--  https://gutenberg.org/cache/epub/2600/pg2600.converter.log
		Resolving gutenberg.org (gutenberg.org)... 152.19.134.47, 2610:28:3090:3000:0:bad:cafe:47
		Connecting to gutenberg.org (gutenberg.org)|152.19.134.47|:443... connected.
		HTTP request sent, awaiting response... 200 OK
		Length: 40894017 (39M) [text/plain]
		Saving to: '/pool1/pg2600.converter.log'

		100%[=======================================================>] 40,894,017   659KB/s   in 76s    

		2023-01-23 07:42:47 (525 KB/s) - '/pool1/pg2600.converter.log' saved [40894017/40894017]
		...
		100%[=======================================================>] 40,894,017   533KB/s   in 1m 49s 
		2023-01-23 07:44:37 (366 KB/s) - '/pool2/pg2600.converter.log' saved [40894017/40894017]
		...
		100%[=======================================================>] 40,894,017   422KB/s   in 85s    
		2023-01-23 07:46:03 (471 KB/s) - '/pool3/pg2600.converter.log' saved [40894017/40894017]
		...
		100%[=======================================================>] 40,894,017   399KB/s   in 84s    
		2023-01-23 07:47:28 (475 KB/s) - '/pool4/pg2600.converter.log' saved [40894017/40894017]

Проверяем наличие файла:  

	[root@localhost ~]# ll /pool*
		/pool1:
		total 22037
		-rw-r--r--. 1 root root 40894017 Jan  2 09:19 pg2600.converter.log

		/pool2:
		total 17982
		-rw-r--r--. 1 root root 40894017 Jan  2 09:19 pg2600.converter.log

		/pool3:
		total 10953
		-rw-r--r--. 1 root root 40894017 Jan  2 09:19 pg2600.converter.log

		/pool4:
		total 39963
		-rw-r--r--. 1 root root 40894017 Jan  2 09:19 pg2600.converter.log

Смотрим, сколько места занимает одинаковый файл в системах с разным уровнем сжатия, проверяем compress ratio:  

	[root@localhost ~]# zfs list  
		NAME    USED  AVAIL     REFER  MOUNTPOINT
		pool1  21.6M   330M     21.5M  /pool1
		pool2  17.7M   334M     17.6M  /pool2
		pool3  10.8M   341M     10.7M  /pool3
		pool4  39.1M   313M     39.0M  /pool4
		
	[root@localhost ~]# zfs get compressratio  
		NAME   PROPERTY       VALUE  SOURCE
		pool1  compressratio  1.81x  -
		pool2  compressratio  2.22x  -
		pool3  compressratio  3.64x  -
		pool4  compressratio  1.00x  -

На основании полученных данных можем сделать вывод, что набиолее эффективным алгоритмом сжатия является алгоритм gzip-9 (pool3)

## Определение настроек пула

Скачиваем в домашний каталог тестовый архив и разархивируем:

	[root@localhost ~]# wget -O archive.tar.gz --no-check-certificate 'https://drive.google.com/u/0/uc?id=1KRBNW33QWqbvbVHa3hLJivOAt60yukkg&export=download'  
	
		--2023-01-23 08:25:44--  https://drive.google.com/u/0/uc?id=1KRBNW33QWqbvbVHa3hLJivOAt60yukkg&export=download
		Resolving drive.google.com (drive.google.com)... 142.251.39.14, 2a00:1450:400d:80e::200e
		Connecting to drive.google.com (drive.google.com)|142.251.39.14|:443... connected.
		HTTP request sent, awaiting response... 302 Found
		Location: https://drive.google.com/uc?id=1KRBNW33QWqbvbVHa3hLJivOAt60yukkg&export=download [following]
		--2023-01-23 08:25:45--  https://drive.google.com/uc?id=1KRBNW33QWqbvbVHa3hLJivOAt60yukkg&export=download
		Reusing existing connection to drive.google.com:443.
		HTTP request sent, awaiting response... 303 See Other
		Location: https://doc-0c-bo-docs.googleusercontent.com/docs/securesc/ha0ro937gcuc7l7deffksulhg5h7mbp1/em6dnsoemphrbk1nrdv3uo659msth3ua/1674462300000/16189157874053420687/*/1KRBNW33QWqbvbVHa3hLJivOAt60yukkg?e=download&uuid=dc17c330-104f-415b-9287-b70aa8e0b7e4 [following]
		Warning: wildcards not supported in HTTP.
		--2023-01-23 08:25:49--  https://doc-0c-bo-docs.googleusercontent.com/docs/securesc/ha0ro937gcuc7l7deffksulhg5h7mbp1/em6dnsoemphrbk1nrdv3uo659msth3ua/1674462300000/16189157874053420687/*/1KRBNW33QWqbvbVHa3hLJivOAt60yukkg?e=download&uuid=dc17c330-104f-415b-9287-b70aa8e0b7e4
		Resolving doc-0c-bo-docs.googleusercontent.com (doc-0c-bo-docs.googleusercontent.com)... 172.217.19.97, 2a00:1450:400d:804::2001
		Connecting to doc-0c-bo-docs.googleusercontent.com (doc-0c-bo-docs.googleusercontent.com)|172.217.19.97|:443... connected.
		HTTP request sent, awaiting response... 200 OK
		Length: 7275140 (6.9M) [application/x-gzip]
		Saving to: 'archive.tar.gz'

		100%[=======================================================>] 7,275,140   1.73MB/s   in 4.0s   

		2023-01-23 08:25:55 (1.73 MB/s) - 'archive.tar.gz' saved [7275140/7275140]

	[root@localhost ~]# tar -zxvf archive.tar.gz 
		zpoolexport/
		zpoolexport/filea
		zpoolexport/fileb

Проверяем возможность импорта каталога в пул:
	
	[root@localhost ~]# zpool import -d zpoolexport/
		   pool: otus
		     id: 6554193320433390805
		  state: ONLINE
		 action: The pool can be imported using its name or numeric identifier.
		 config:

			otus                         ONLINE
			  mirror-0                   ONLINE
			    /root/zpoolexport/filea  ONLINE
			    /root/zpoolexport/fileb  ONLINE
Импортируем:

	[root@localhost ~]# zpool status
		  pool: otus
		 state: ONLINE
		  scan: none requested
		config:

			NAME                         STATE     READ WRITE CKSUM
			otus                         ONLINE       0     0     0
			  mirror-0                   ONLINE       0     0     0
			    /root/zpoolexport/filea  ONLINE       0     0     0
			    /root/zpoolexport/fileb  ONLINE       0     0     0

		errors: No known data errors
	...

Запрашиваем все параметры импортированного пула:  

	[root@localhost ~]# zfs get all otus
		NAME  PROPERTY              VALUE                  SOURCE
		otus  type                  filesystem             -
		otus  creation              Fri May 15  4:00 2020  -
		otus  used                  2.04M                  -
		otus  available             350M                   -
		otus  referenced            24K                    -
		otus  compressratio         1.00x                  -
		otus  mounted               yes                    -
		otus  quota                 none                   default
		otus  reservation           none                   default
		otus  recordsize            128K                   local
		otus  mountpoint            /otus                  default
		otus  sharenfs              off                    default
		otus  checksum              sha256                 local
		otus  compression           zle                    local
		otus  atime                 on                     default
		otus  devices               on                     default
		otus  exec                  on                     default
		otus  setuid                on                     default
		otus  readonly              off                    default
		otus  zoned                 off                    default
		otus  snapdir               hidden                 default
		otus  aclinherit            restricted             default
		otus  createtxg             1                      -
		otus  canmount              on                     default
		otus  xattr                 on                     default
		otus  copies                1                      default
		otus  version               5                      -
		otus  utf8only              off                    -
		otus  normalization         none                   -
		otus  casesensitivity       sensitive              -
		otus  vscan                 off                    default
		otus  nbmand                off                    default
		otus  sharesmb              off                    default
		otus  refquota              none                   default
		otus  refreservation        none                   default
		otus  guid                  14592242904030363272   -
		otus  primarycache          all                    default
		otus  secondarycache        all                    default
		otus  usedbysnapshots       0B                     -
		otus  usedbydataset         24K                    -
		otus  usedbychildren        2.01M                  -
		otus  usedbyrefreservation  0B                     -
		otus  logbias               latency                default
		otus  objsetid              54                     -
		otus  dedup                 off                    default
		otus  mlslabel              none                   default
		otus  sync                  standard               default
		otus  dnodesize             legacy                 default
		otus  refcompressratio      1.00x                  -
		otus  written               24K                    -
		otus  logicalused           1020K                  -
		otus  logicalreferenced     12K                    -
		otus  volmode               default                default
		otus  filesystem_limit      none                   default
		otus  snapshot_limit        none                   default
		otus  filesystem_count      none                   default
		otus  snapshot_count        none                   default
		otus  snapdev               hidden                 default
		otus  acltype               off                    default
		otus  context               none                   default
		otus  fscontext             none                   default
		otus  defcontext            none                   default
		otus  rootcontext           none                   default
		otus  relatime              off                    default
		otus  redundant_metadata    all                    default
		otus  overlay               off                    default
		otus  encryption            off                    default
		otus  keylocation           none                   default
		otus  keyformat             none                   default
		otus  pbkdf2iters           0                      default
		otus  special_small_blocks  0                      default

Или уточняем интересующие нас параметры:

свободное место 

	[root@localhost ~]# zfs get available otus
		NAME  PROPERTY   VALUE  SOURCE
		otus  available  350M   -
разрешение на запись  

	[root@localhost ~]# zfs get readonly otus
		NAME  PROPERTY  VALUE   SOURCE
		otus  readonly  off     default

размер блока  

	[root@localhost ~]# zfs get recordsize otus
		NAME  PROPERTY    VALUE    SOURCE
		otus  recordsize  128K     local

тип сжатия (если включен)  

	[root@localhost ~]# zfs get compression otus
		NAME  PROPERTY     VALUE     SOURCE
		otus  compression  zle       local

тип контрольной суммы  

	[root@localhost ~]# zfs get checksum otus
		NAME  PROPERTY  VALUE      SOURCE
		otus  checksum  sha256     local

## Работа со снапшотами

Скачиваем тестовый файл:  

	[root@localhost ~]# wget -O otus_task2.file --no-check-certificate 'https://drive.google.com/u/0/uc?id=1gH8gCL9y7Nd5Ti3IRmplZPF1XjzxeRAG&export=download'
	
		--2023-01-23 09:22:06--  https://drive.google.com/u/0/uc?id=1gH8gCL9y7Nd5Ti3IRmplZPF1XjzxeRAG&export=download
		Resolving drive.google.com (drive.google.com)... 142.251.208.110, 2a00:1450:400d:80c::200e
		Connecting to drive.google.com (drive.google.com)|142.251.208.110|:443... connected.
		HTTP request sent, awaiting response... 302 Found
		Location: https://drive.google.com/uc?id=1gH8gCL9y7Nd5Ti3IRmplZPF1XjzxeRAG&export=download [following]
		--2023-01-23 09:22:07--  https://drive.google.com/uc?id=1gH8gCL9y7Nd5Ti3IRmplZPF1XjzxeRAG&export=download
		Reusing existing connection to drive.google.com:443.
		HTTP request sent, awaiting response... 303 See Other
		Location: https://doc-00-bo-docs.googleusercontent.com/docs/securesc/ha0ro937gcuc7l7deffksulhg5h7mbp1/o8uta487f4d9fc7mv0j520q82cka28kh/1674465675000/16189157874053420687/*/1gH8gCL9y7Nd5Ti3IRmplZPF1XjzxeRAG?e=download&uuid=dacff41b-bad0-4d51-bb81-a14cf9fd13ed [following]
		Warning: wildcards not supported in HTTP.
		--2023-01-23 09:22:11--  https://doc-00-bo-docs.googleusercontent.com/docs/securesc/ha0ro937gcuc7l7deffksulhg5h7mbp1/o8uta487f4d9fc7mv0j520q82cka28kh/1674465675000/16189157874053420687/*/1gH8gCL9y7Nd5Ti3IRmplZPF1XjzxeRAG?e=download&uuid=dacff41b-bad0-4d51-bb81-a14cf9fd13ed
		Resolving doc-00-bo-docs.googleusercontent.com (doc-00-bo-docs.googleusercontent.com)... 142.250.180.193, 2a00:1450:400d:802::2001
		Connecting to doc-00-bo-docs.googleusercontent.com (doc-00-bo-docs.googleusercontent.com)|142.250.180.193|:443... connected.
		HTTP request sent, awaiting response... 200 OK
		Length: 5432736 (5.2M) [application/octet-stream]
		Saving to: 'otus_task2.file'

		100%[=========================================================================================================================================>] 5,432,736   1.58MB/s   in 3.3s   

		2023-01-23 09:22:16 (1.58 MB/s) - 'otus_task2.file' saved [5432736/5432736]
	
Восстанавливаем том из снапшота:  

	[root@localhost ~]# zfs receive otus/test@today < otus_task2.file 

Смотрим, что восстановилось, ищем secret_file и его содержимое:  

	[root@localhost ~]# cd /otus/test/  
	
	[root@localhost test]# ll  
		total 2590
		-rw-r--r--. 1 root    root          0 May 15  2020 10M.file
		-rw-r--r--. 1 root    root     309987 May 15  2020 Limbo.txt
		-rw-r--r--. 1 root    root     509836 May 15  2020 Moby_Dick.txt
		-rw-r--r--. 1 root    root    1209374 May  6  2016 War_and_Peace.txt
		-rw-r--r--. 1 root    root     727040 May 15  2020 cinderella.tar
		-rw-r--r--. 1 root    root         65 May 15  2020 for_examaple.txt
		-rw-r--r--. 1 root    root          0 May 15  2020 homework4.txt
		drwxr-xr-x. 3 vagrant vagrant       4 Dec 18  2017 task1
		-rw-r--r--. 1 root    root     398635 May 15  2020 world.sql

	[root@localhost test]# find /otus/test/ -name "secret_message"
		/otus/test/task1/file_mess/secret_message

	[root@localhost test]# cat /otus/test/task1/file_mess/secret_message 
		https://github.com/sindresorhus/awesome



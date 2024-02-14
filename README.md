# Домашнее задание 5. Практические навыки работы с ZFS
#### Инструкция по запуску стенда
В домашнем каталоге создаем каталог zfs и копируем в него файлы из репозитория.
Поднимаем виртуальную машину под Centos 7 и подключаемся к ней:

vagrant up  
vagrant ssh

####1. Определение алгоритма с наилучшим сжатием

Смотрим список всех дисков, которые есть в виртуальной машине:

[vagrant@zfs ~]$ lsblk  
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT  
sda      8:0    0   40G  0 disk 
`-sda1   8:1    0   40G  0 part /  
sdb      8:16   0  512M  0 disk 
sdc      8:32   0  512M  0 disk 
sdd      8:48   0  512M  0 disk 
sde      8:64   0  512M  0 disk 
sdf      8:80   0  512M  0 disk 
sdg      8:96   0  512M  0 disk 
sdh      8:112  0  512M  0 disk 
sdi      8:128  0  512M  0 disk 

Создаём 4 пула из двух дисков, каждый в режиме RAID 1, и смотрим информацию о пулах:

[vagrant@zfs ~]$ sudo -i  
[root@zfs ~]# zpool create otus1 mirror /dev/sdb /dev/sdc  
[root@zfs ~]# zpool create otus2 mirror /dev/sdd /dev/sde  
[root@zfs ~]# zpool create otus3 mirror /dev/sdf /dev/sdg  
[root@zfs ~]# zpool create otus4 mirror /dev/sdh /dev/sdi  
[root@zfs ~]# zpool list  
NAME    SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT  
otus1   480M  91.5K   480M        -         -     0%     0%  1.00x    ONLINE  -  
otus2   480M  91.5K   480M        -         -     0%     0%  1.00x    ONLINE  -  
otus3   480M  91.5K   480M        -         -     0%     0%  1.00x    ONLINE  -  
otus4   480M  91.5K   480M        -         -     0%     0%  1.00x    ONLINE  -

Добавим разные алгоритмы сжатия в каждую файловую систему и проверим, что все файловые системы имеют разные методы сжатия:

[root@zfs ~]# zfs set compression=lzjb otus1  
[root@zfs ~]# zfs set compression=lz4 otus2  
[root@zfs ~]# zfs set compression=gzip-9 otus3  
[root@zfs ~]# zfs set compression=zle otus4  
[root@zfs ~]# zfs get all | grep compression  
otus1  compression           lzjb                   local  
otus2  compression           lz4                    local  
otus3  compression           gzip-9                 local  
otus4  compression           zle                    local

Скачаем один и тот же текстовый файл во все пулы:

[root@zfs ~]# for i in {1..4}; do wget -P /otus$i https://gutenberg.org/cache/epub/2600/pg2600.converter.log; done
--2024-02-14 10:23:36--  https://gutenberg.org/cache/epub/2600/pg2600.converter.log
Resolving gutenberg.org (gutenberg.org)... 152.19.134.47, 2610:28:3090:3000:0:bad:cafe:47
Connecting to gutenberg.org (gutenberg.org)|152.19.134.47|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 41016061 (39M) [text/plain]
Saving to: '/otus1/pg2600.converter.log'

100%[======================================>] 41,016,061  8.24MB/s   in 5.6s   

2024-02-14 10:23:42 (6.98 MB/s) - '/otus1/pg2600.converter.log' saved [41016061/41016061]

--2024-02-14 10:23:42--  https://gutenberg.org/cache/epub/2600/pg2600.converter.log
Resolving gutenberg.org (gutenberg.org)... 152.19.134.47, 2610:28:3090:3000:0:bad:cafe:47
Connecting to gutenberg.org (gutenberg.org)|152.19.134.47|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 41016061 (39M) [text/plain]
Saving to: '/otus2/pg2600.converter.log'

100%[======================================>] 41,016,061  5.24MB/s   in 6.9s   

2024-02-14 10:23:50 (5.65 MB/s) - '/otus2/pg2600.converter.log' saved [41016061/41016061]

--2024-02-14 10:23:50--  https://gutenberg.org/cache/epub/2600/pg2600.converter.log
Resolving gutenberg.org (gutenberg.org)... 152.19.134.47, 2610:28:3090:3000:0:bad:cafe:47
Connecting to gutenberg.org (gutenberg.org)|152.19.134.47|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 41016061 (39M) [text/plain]
Saving to: '/otus3/pg2600.converter.log'

100%[======================================>] 41,016,061  3.63MB/s   in 9.5s   

2024-02-14 10:24:00 (4.11 MB/s) - '/otus3/pg2600.converter.log' saved [41016061/41016061]

--2024-02-14 10:24:00--  https://gutenberg.org/cache/epub/2600/pg2600.converter.log
Resolving gutenberg.org (gutenberg.org)... 152.19.134.47, 2610:28:3090:3000:0:bad:cafe:47
Connecting to gutenberg.org (gutenberg.org)|152.19.134.47|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 41016061 (39M) [text/plain]
Saving to: '/otus4/pg2600.converter.log'

100%[======================================>] 41,016,061  10.1MB/s   in 5.1s   

2024-02-14 10:24:06 (7.62 MB/s) - '/otus4/pg2600.converter.log' saved [41016061/41016061]

Проверим, что файл был скачан во все пулы:

[root@zfs ~]# ls -l /otus*
/otus1:
total 22067
-rw-r--r--. 1 root root 41016061 Feb  2 08:53 pg2600.converter.log

/otus2:
total 17994
-rw-r--r--. 1 root root 41016061 Feb  2 08:53 pg2600.converter.log

/otus3:
total 10959
-rw-r--r--. 1 root root 41016061 Feb  2 08:53 pg2600.converter.log

/otus4:
total 40091
-rw-r--r--. 1 root root 41016061 Feb  2 08:53 pg2600.converter.log

Видно, что самый оптимальный метод сжатия у нас
используется в пуле otus3 (файл занимает 10959 КБ).
Проверим, сколько места занимает один и тот же файл в разных пулах
и проверим степень сжатия файлов:

[root@zfs ~]# zfs list
NAME    USED  AVAIL     REFER  MOUNTPOINT
otus1  21.6M   330M     21.6M  /otus1
otus2  17.7M   334M     17.6M  /otus2
otus3  10.8M   341M     10.7M  /otus3
otus4  39.2M   313M     39.2M  /otus4
[root@zfs ~]# zfs get all | grep compressratio | grep -v ref
otus1  compressratio         1.81x                  -
otus2  compressratio         2.22x                  -
otus3  compressratio         3.65x                  -
otus4  compressratio         1.00x                  -

Алгоритм gzip-9 самый эффективный по сжатию.

####2. Определение настроек пула

Скачиваем архив в домашний каталог и разархивируем его:

[root@zfs ~]# tar -xzvf zfs_task1.tar.gz
zpoolexport/
zpoolexport/filea
zpoolexport/fileb

Проверим, возможно ли импортировать данный каталог в пул:

[root@zfs ~]# zpool import -d zpoolexport/
   pool: otus
     id: 6554193320433390805
  state: ONLINE
 action: The pool can be imported using its name or numeric identifier.
 config:

	otus                         ONLINE
	  mirror-0                   ONLINE
	    /root/zpoolexport/filea  ONLINE
	    /root/zpoolexport/fileb  ONLINE

Сделаем импорт данного пула к нам в ОС:

[root@zfs ~]# zpool import -d zpoolexport/ otus
[root@zfs ~]# zpool status
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

  pool: otus1
 state: ONLINE
  scan: none requested
config:

	NAME        STATE     READ WRITE CKSUM
	otus1       ONLINE       0     0     0
	  mirror-0  ONLINE       0     0     0
	    sdb     ONLINE       0     0     0
	    sdc     ONLINE       0     0     0

errors: No known data errors

  pool: otus2
 state: ONLINE
  scan: none requested
config:

	NAME        STATE     READ WRITE CKSUM
	otus2       ONLINE       0     0     0
	  mirror-0  ONLINE       0     0     0
	    sdd     ONLINE       0     0     0
	    sde     ONLINE       0     0     0

errors: No known data errors

  pool: otus3
 state: ONLINE
  scan: none requested
config:

	NAME        STATE     READ WRITE CKSUM
	otus3       ONLINE       0     0     0
	  mirror-0  ONLINE       0     0     0
	    sdf     ONLINE       0     0     0
	    sdg     ONLINE       0     0     0

errors: No known data errors

  pool: otus4
 state: ONLINE
  scan: none requested
config:

	NAME        STATE     READ WRITE CKSUM
	otus4       ONLINE       0     0     0
	  mirror-0  ONLINE       0     0     0
	    sdh     ONLINE       0     0     0
	    sdi     ONLINE       0     0     0

errors: No known data errors

Определяем настройки пула:

- размер хранилища:

[root@zfs ~]# zfs get available otus
NAME  PROPERTY   VALUE  SOURCE
otus  available  350M   -
[root@zfs ~]# zfs get used otus
NAME  PROPERTY  VALUE  SOURCE
otus  used      2.04M  -

- тип pool:

[root@zfs ~]# zfs get type otus
NAME  PROPERTY  VALUE       SOURCE
otus  type      filesystem  -
[root@zfs ~]# zfs get readonly otus
NAME  PROPERTY  VALUE   SOURCE
otus  readonly  off     default

- значение recordsize:

[root@zfs ~]# zfs get recordsize otus
NAME  PROPERTY    VALUE    SOURCE
otus  recordsize  128K     local

- какое сжатие используется:

[root@zfs ~]# zfs get compression otus
NAME  PROPERTY     VALUE     SOURCE
otus  compression  zle       local

- какая контрольная сумма используется:

[root@zfs ~]# zfs get checksum otus
NAME  PROPERTY  VALUE      SOURCE
otus  checksum  sha256     local

####3. Работа со снапшотом, поиск сообщения от преподавателя

Скачаем файл, указанный в задании и восстановим файловую систему из снапшота:

[root@zfs ~]# zfs receive otus/test@today < otus_task2.file

Ищем в каталоге /otus/test файл с именем “secret_message”:

[root@zfs ~]# find /otus/test -name "secret_message"
/otus/test/task1/file_mess/secret_message

Смотрим содержимое найденного файла:

[root@zfs ~]# cat /otus/test/task1/file_mess/secret_message
https://otus.ru/lessons/linux-hl/






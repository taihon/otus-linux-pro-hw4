## OTUS Administrator Linux. Professional ДЗ №4: ZFS

Замечания:

- Перед решением стоит отметить, что ссылки на google drive были изменены на docs.google.com, поскольку при попытке скачивания больших файлов с google drive - выдаётся промежуточная страница с результатами антивирусного сканирования, и предложением скачать файл.

### **Задачи:**

1.  Определить алгоритм с наилучшим сжатием.
    Шаги:

    1.  Определить какие алгоритмы сжатия поддерживает zfs (gzip gzip-N, zle lzjb, lz4);
    2.  Создать 4 файловых системы на каждой применить свой алгоритм сжатия;

        Для сжатия использовать либо текстовый файл либо группу файлов:
        скачать файл “Война и мир” и расположить на файловой системе wget -O War_and_Peace.txt http://www.gutenberg.org/ebooks/2600.txt.utf-8, либо скачать файл ядра распаковать и расположить на файловой системе.

    **Решение**

    ```
        [vagrant@zfs vagrant]$ sudo su -
        [root@zfs ~]# readarray -t drives < <(lsblk -o NAME,FSTYPE -dsn|awk '$2 == "" {print $1}')
        [root@zfs ~]# newlen=$(((${#drives[@]}-1) / 2))
        [root@zfs ~]# codecs=("lzjb" "lz4" "gzip" "zle")
        [root@zfs ~]# wget -O war-and-peace.txt http://www.gutenberg.org/ebooks/2600.txt
        .utf-8
        --2023-02-06-- http://www.gutenberg.org/ebooks/2600.txt.utf-8
        Resolving www.gutenberg.org (www.gutenberg.org)... 152.19.134.47, 2610:28:3090:3000:0:bad:cafe:47
        Connecting to www.gutenberg.org (www.gutenberg.org)|152.19.134.47|:80... connected.
        HTTP request sent, awaiting response... 302 Found
        Location: https://www.gutenberg.org/ebooks/2600.txt.utf-8 [following]
        --2023-02-06-- https://www.gutenberg.org/ebooks/2600.txt.utf-8
        Connecting to www.gutenberg.org (www.gutenberg.org)|152.19.134.47|:443... connected.
        HTTP request sent, awaiting response... 302 Found
        Location: https://www.gutenberg.org/cache/epub/2600/pg2600.txt [following]
        --2023-02-06-- https://www.gutenberg.org/cache/epub/2600/pg2600.txt
        Reusing existing connection to www.gutenberg.org:443.
        HTTP request sent, awaiting response... 200 OK
        Length: 3359372 (3.2M) [text/plain]
        Saving to: ‘war-and-peace.txt’
        100%[======================================>] 3,359,372 1.57MB/s in 2.0s

        2023-02-06 (1.57 MB/s) - ‘war-and-peace.txt’ saved [3359372/3359372]

        [root@zfs ~]# for i in `seq 0 $newlen`;do zpool create "otus$(($i+1))" "$drives[$(($i*2))]}" "${drives[$(($i*2+1))]}";zfs set compression="${codecs[$i]}" "otus$(($i+1))";done
        [root@zfs ~]# zpool list
        NAME SIZE ALLOC FREE CKPOINT EXPANDSZ FRAG CAP DEDUP HEALTH ALTROOT
        otus1 960M 147K 960M - - 0% 0% 1.00x ONLINE -
        otus2 960M 147K 960M - - 0% 0% 1.00x ONLINE -
        otus3 960M 147K 960M - - 0% 0% 1.00x ONLINE -
        otus4 960M 147K 960M - - 0% 0% 1.00x ONLINE -
        [root@zfs ~]# zpool get all|grep compression
        otus1 compression lzjb local
        otus2 compression lz4 local
        otus3 compression gzip local
        otus4 compression zle local
        [root@zfs ~]# for i in {1..4};do cp war-and-peace.txt /otus$i;done
        [root@zfs ~]# zfs get all|grep compressratio|grep -v ref
        otus1 compressratio 1.35x -
        otus2 compressratio 1.62x -
        otus3 compressratio 2.63x -
        otus4 compressratio 1.01x -
    ```

    **Результат:**

    - список команд которыми получен результат с их выводами - приведён выше.
    - вывод команды из которой видно какой из алгоритмов лучше. Наглядно видно, что gzip лучше.

2.  Определить настройки pool’a. Шаги:

    1. Загрузить архив с файлами локально.
       https://drive.google.com/open?id=1KRBNW33QWqbvbVHa3hLJivOAt60yukkg
    2. Распаковать.
    3. с помощью команды zfs import собрать pool ZFS;
    4. командами zfs определить настройки:

       1. размер хранилища;
       2. тип pool;
       3. значение recordsize;
       4. какое сжатие используется;
       5. какая контрольная сумма используется.

    **Решение**

    ```
       [root@zfs ~]# wget --no-check-certificate -O archive.tar.gz 'https://docs.google.com/uc?export=download&id=1KRBNW33QWqbvbVHa3hLJivOAt60yukkg' -r -A -e
       WARNING: combining -O with -r or -p will mean that all downloaded content
       will be placed in the single file you specified.

       --2023-02-06-- https://docs.google.com/uc?export=download&id=1KRBNW33QWqbvbVHa3hLJivOAt60yukkg
       Resolving docs.google.com (docs.google.com)... 108.177.119.194, 2a00:1450:4013:c07::c2
       Connecting to docs.google.com (docs.google.com)|108.177.119.194|:443... connected.
       HTTP request sent, awaiting response... 303 See Other
       Location: https://doc-0c-bo-docs.googleusercontent.com/docs/securesc/ha0ro937gcuc7l7deffksulhg5h7mbp1/hceg7ej5usn2ru5o6bfd68dgmd7d52lq/1675670700000/16189157874053420687/*/1KRBNW33QWqbvbVHa3hLJivOAt60yukkg?e=download&uuid=0617dda8-ea5a-4058-b7ea-05d8c99b88cc [following]
       Warning: wildcards not supported in HTTP.
       --2023-02-06-- https://doc-0c-bo-docs.googleusercontent.com/docs/securesc/ha0ro937gcuc7l7deffksulhg5h7mbp1/hceg7ej5usn2ru5o6bfd68dgmd7d52lq/1675670700000/16189157874053420687/*/1KRBNW33QWqbvbVHa3hLJivOAt60yukkg?e=download&uuid=0617dda8-ea5a-4058-b7ea-05d8c99b88cc
       Resolving doc-0c-bo-docs.googleusercontent.com (doc-0c-bo-docs.googleusercontent.com)... 142.250.180.97, 2a00:1450:4008:805::2001
       Connecting to doc-0c-bo-docs.googleusercontent.com (doc-0c-bo-docs.googleusercontent.com)|142.250.180.97|:443... connected.
       HTTP request sent, awaiting response... 200 OK
       Length: 7275140 (6.9M) [application/x-gzip]
       Saving to: ‘archive.tar.gz’
       100%[======================================>] 7,275,140 4.03MB/s in 1.7s
       2023-02-06 (4.03 MB/s) - ‘archive.tar.gz’ saved [7275140/275140]
       FINISHED --2023-02-06--
       Total wall clock time: 7.7s
       Downloaded: 1 files, 6.9M in 1.7s (4.03 MB/s)
       [root@zfs ~]# file archive.tar.gz
       archive.tar.gz: gzip compressed data, from Unix, last modified: Fri May 15 05:00:29 2020
       [root@zfs ~]# tar -xf archive.tar.gz
       [root@zfs ~]# ll
       total 10408
       -rw-------. 1 root root 5570 Apr 30 2020 anaconda-ks.cfg
       -rw-r--r--. 1 root root 7275140 Feb 6 08:05 archive.tar.gz
       -rw-------. 1 root root 5300 Apr 30 2020 original-ks.cfg
       -rw-r--r--. 1 root root 3359372 Feb 2 09:16 war-and-peace.txt
       drwxr-xr-x. 2 root root 32 May 15 2020 zpoolexport
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

       [root@zfs ~]# zpool import -d zpoolexport/ otus
       [root@zfs ~]# zpool list
       NAME SIZE ALLOC FREE CKPOINT EXPANDSZ FRAG CAP DEDUP HEALTH ALTROOT
       otus 480M 2.18M 478M - - 0% 0% 1.00x ONLINE -
       otus1 960M 2.50M 957M - - 0% 0% 1.00x ONLINE -
       otus2 960M 2.11M 958M - - 0% 0% 1.00x ONLINE -
       otus3 960M 1.33M 959M - - 0% 0% 1.00x ONLINE -
       otus4 960M 3.32M 957M - - 0% 0% 1.00x ONLINE -
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
             sda       ONLINE       0     0     0
             sdb       ONLINE       0     0     0

       errors: No known data errors

       pool: otus2
       state: ONLINE
       scan: none requested
       config:

           NAME        STATE     READ WRITE CKSUM
           otus2       ONLINE       0     0     0
             sdc       ONLINE       0     0     0
             sdd       ONLINE       0     0     0

       errors: No known data errors

       pool: otus3
       state: ONLINE
       scan: none requested
       config:

           NAME        STATE     READ WRITE CKSUM
           otus3       ONLINE       0     0     0
             sde       ONLINE       0     0     0
             sdf       ONLINE       0     0     0

       errors: No known data errors

       pool: otus4
       state: ONLINE
       scan: none requested
       config:

           NAME        STATE     READ WRITE CKSUM
           otus4       ONLINE       0     0     0
             sdg       ONLINE       0     0     0
             sdh       ONLINE       0     0     0

       errors: No known data errors
       [root@zfs ~]# zfs get all otus| grep -E 'available|readonly|recordsize|compression|checksum'
       otus available 350M -
       otus recordsize 128K local
       otus checksum sha256 local
       otus compression zle local
       otus readonly off default
    ```

    **Результат**:

    - список команд которыми восстановили pool приведён выше
    - настройки - 350Мб из 480Мб свободного места, размер записи 128К, алгоритм подсчёта контрольных сумм sha256, тип сжатия zle, массив является зеркалом, находится в режиме чтение+запись.

3.  Найти сообщение от преподавателей. Шаги:

    1. Скопировать файл из удаленной директории. https://drive.google.com/file/d/1gH8gCL9y7Nd5Ti3IRmplZPF1XjzxeRAG/view?usp=sharing
       Файл был получен командой zfs send otus/storage@task2 > otus_task2.file
    2. Восстановить файл локально. zfs receive
       найти зашифрованное сообщение в файле secret_message

    **Решение**

    ```
    [root@zfs ~]# wget --no-check-certificate -O otustask2.file 'https://docs.google.com/uc?export=download&=id=1gH8gCL9y7Nd5Ti3IRmplZPF1XjzxeRAG' -r -A -e
    WARNING: combining -O with -r or -p will mean that all downloaded content will be placed in the single file you specified.

    --2023-02-06-- https://docs.google.com/uc?export=download&id=1gH8gCL9y7Nd5Ti3IRmplZPF1XjzxeRAG
    Resolving docs.google.com (docs.google.com)... 108.177.119.194, 2a00:1450:4013:c00::c2
    Connecting to docs.google.com (docs.google.com)|108.177.119.194|:443... connected.
    HTTP request sent, awaiting response... 303 See Other
    Location: https://doc-00-bo-docs.googleusercontent.com/docs/securesc/ha0ro937gcuc7l7deffksulhg5h7mbp1/me38kvfd1mpcdu55eau3hrloodcad7l1/1675670775000/16189157874053420687/*/1gH8gCL9y7Nd5Ti3IRmplZPF1XjzxeRAG?e=download&uuid=b9650fde-7939-432d-8fcb-716fb2cc9d73 [following]
    Warning: wildcards not supported in HTTP.
    --2023-02-06-- https://doc-00-bo-docs.googleusercontent.com/docs/securesc/ha0ro937gcuc7l7deffksulhg5h7mbp1/me38kvfd1mpcdu55eau3hrloodcad7l1/1675670775000/16189157874053420687/*/1gH8gCL9y7Nd5Ti3IRmplZPF1XjzxeRAG?e=download&uuid=b9650fde-7939-432d-8fcb-716fb2cc9d73
    Resolving doc-00-bo-docs.googleusercontent.com (doc-00-bo-docs.googleusercontent.com)... 142.250.180.97, 2a00:1450:4008:805::2001
    Connecting to doc-00-bo-docs.googleusercontent.com (doc-00-bo-docs.googleusercontent.com)|142.250.180.97|:443... connected.
    HTTP request sent, awaiting response... 200 OK
    Length: 5432736 (5.2M) [application/octet-stream]
    Saving to: ‘otustask2.file’
    100%[======================================>] 5,432,736 2.59MB/s in 2.0s

    2023-02-06 (2.59 MB/s) - ‘otustask2.file’ saved [5432736/5432736]

    FINISHED --2023-02-06--
    Total wall clock time: 7.9s
    Downloaded: 1 files, 5.2M in 2.0s (2.59 MB/s)
    [root@zfs ~]# file otustask2.file
    otustask2.file: ZFS shapshot (little-endian machine), version 17, type: ZFS, destination GUID: 70 B1 CE AB 92 00 51 35, name: 'otus/storage@task2'
    [root@zfs ~]# zfs receive otus/test@today < otustask2.file
    [root@zfs ~]# find /otus/test/ -name 'secret_message'
    /otus/test/task1/file_mess/secret_message
    [root@zfs ~]# cat /otus/test/task1/file_mess/secret_message
    https://github.com/sindresorhus/awesome
    ```

    **Результат:**

    - список шагов которыми восстанавливали приведён выше;
    - зашифрованное сообщение - "https://github.com/sindresorhus/awesome"

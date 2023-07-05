# PostgreSQL репликация и резервное копирование.

Задание:

- настроить hot_standby репликацию с использованием слотов;

- настроить правильное резервное копирование (barman).

Цель:

Научиться настраивать репликацию и создавать резервные копии в СУБД PostgreSQL.

## Развертывание стенда с инфраструктурой для PostgreSQL репликации и резервного копирования.

Стэнд состоит из хостовой машины под управлением ОС Ubuntu 20.04 на которой развернут Ansible и трех виртуальных машин `bento/centos-8.4`.

В исходном Vagranfile я изменил Vagrant boxes `centos/stream8` на `bento/centos-8.4` так как `centos/stream8` не доступен хранилище Vagrant.

Виртуальные машины с именами: `node1` (master), `node2` (salve) выполняют роль серверов `PostgreSQL`, на которых будет настроена `hot_standby` репликация. 

Виртуальная машина с именем `barman` выполняет роль сервера резервных копий БД `PostgreSQL`(копии снимаем с сервера `node1`).

Разворачиваем инфраструктуру в Vagrant исключительно через последовательный запуск ролей Ansible.

Все коментарии по каждому блоку указаны в текстах Playbook:

 - `postgre.yml` - главный Playbook, в котором описыны все выполняемые роли;

 - `ansible\install_postgres\tasks\main.yml` - Playbook роли установки СУБД;

 - `ansible\postgres_replication\tasks\main.yml` - Playbook роли настройки репликации;

 - `ansible\install_barman\tasks\main.yml` - Playbook роли установки и настройки 
 резервного копирования;


Каталог `ansible` необходимо поместить в каталог с Vagranfile .

Выполняем установку стенда:

```
vagrant up
```
## Проверяем работу стенда

### Hot_standby репликации

После завершения развертывания стенда, заходим на `node1` сервер 

```
vagrant ssh node1
```
заходим в PostgreSQL и смотрим список БД:  
```
sudo -u postgres psql
```
вывод команды:
```
[root@node1 vagrant]# sudo -u postgres psql
could not change directory to "/home/vagrant": Permission denied
psql (14.8)
Type "help" for help.

postgres=# \l
                                  List of databases
    Name    |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges   
------------+----------+----------+-------------+-------------+-----------------------
 otus_linux | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
 postgres   | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
 template0  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
            |          |          |             |             | postgres=CTc/postgres
 template1  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
            |          |          |             |             | postgres=CTc/postgres
(4 rows)
```
создадим ещё одну БД
```
postgres=#  CREATE DATABASE otus_2023;
CREATE DATABASE
postgres=# \l
                                  List of databases
    Name    |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges   
------------+----------+----------+-------------+-------------+-----------------------
 otus_2023  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
 otus_linux | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
 postgres   | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
 template0  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
            |          |          |             |             | postgres=CTc/postgres
 template1  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
            |          |          |             |             | postgres=CTc/postgres
(5 rows)
```
заходим на `node2` сервер 

```
vagrant ssh node2
```
смотриим список БД `\l`, проверяем как отработала репликация 

```
root@root-ubuntu:/home/roman/Postgres# vagrant ssh node2

This system is built by the Bento project by Chef Software
More information can be found at https://github.com/chef/bento
Last login: Wed Jul  5 04:02:47 2023 from 192.168.57.1
[vagrant@node2 ~]$ sudo -u postgres psql
could not change directory to "/home/vagrant": Permission denied
psql (14.8)
Type "help" for help.

postgres=# \l
                                  List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileg
es   
-----------+----------+----------+-------------+-------------+------------------
-----
 otus_2023  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
 otus_linux | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
 postgres   | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
 template0  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres      
+
            |          |          |             |             | postgres=CTc/post
gres
 template1  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres      
    +
            |          |          |             |             | postgres=CTc/post
gres
(5 rows)
```
### Настройка резервного копирования

заходим на `barman` сервер 

```
vagrant ssh barman
```
проверим работу barman: 

```
bash-4.4$ barman switch-wal node1
The WAL file 000000010000000000000006 has been closed on server 'node1'
bash-4.4$ barman cron
Starting WAL archiving for server node1
bash-4.4$ barman check node1
Server node1:
	PostgreSQL: OK
	superuser or standard user with backup privileges: OK
	PostgreSQL streaming: OK
	wal_level: OK
	replication slot: OK
	directories: OK
	retention policy settings: OK
	backup maximum age: OK (interval provided: 4 days, latest backup age: 7 hours, 52 minutes, 13 seconds)
	backup minimum size: OK (33.5 MiB)
	wal maximum age: OK (no last_wal_maximum_age provided)
	wal size: OK (17.4 KiB)
	compression settings: OK
	failed backups: OK (there are 0 failed backups)
	minimum redundancy requirements: OK (have 1 backups, expected at least 1)
	pg_basebackup: OK
	pg_basebackup compatible: OK
	pg_basebackup supports tablespaces mapping: OK
	systemid coherence: OK
	pg_receivexlog: OK
	pg_receivexlog compatible: OK
	receive-wal running: OK
	archiver errors: OK
```
запускаем создание резервной копии БД сервера `node1`:

```
bash-4.4$ barman backup node1
Starting backup using postgres method for server node1 in /var/lib/barman/node1/base/20230705T120505
Backup start at LSN: 0/7000060 (000000010000000000000007, 00000060)
Starting backup copy via pg_basebackup for 20230705T120505
Copy done (time: less than one second)
Finalising the backup.
Backup size: 42.0 MiB
Backup end at LSN: 0/9000000 (000000010000000000000008, 00000000)
Backup completed (start time: 2023-07-05 12:05:05.075203, elapsed time: less than one second)
Processing xlog segments from streaming for node1
	000000010000000000000007
	000000010000000000000008
```
на хосте node1 в psql смотрим список БД:

```
postgres=# \l
                                  List of databases
    Name    |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges   
------------+----------+----------+-------------+-------------+-----------------------
 otus_2023  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
 otus_linux | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
 postgres   | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
 template0  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
            |          |          |             |             | postgres=CTc/postgres
 template1  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
            |          |          |             |             | postgres=CTc/postgres
(5 rows)
```
удаляем созданные нами БД

```
postgres=# DROP DATABASE otus_2023;
DROP DATABASE
postgres=# DROP DATABASE otus_linux;
DROP DATABASE
postgres=# \l
                                  List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges   
-----------+----------+----------+-------------+-------------+-----------------------
 postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
 template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
           |          |          |             |             | postgres=CTc/postgres
(3 rows)
```

на хосте barman запустим восстановление: 

```
bash-4.4$ barman list-backup node1
node1 20230705T120505 - Wed Jul  5 02:05:05 2023 - Size: 42.0 MiB - WAL Size: 0 B
node1 20230705T041021 - Tue Jul  4 18:10:21 2023 - Size: 33.6 MiB - WAL Size: 49.5 KiB

bash-4.4$ barman recover node1 20230705T120505 /var/lib/pgsql/14/data/ --remote-ssh-comman "ssh postgres@192.168.57.11"
Starting remote restore for server node1 using backup 20230705T120505
Destination directory: /var/lib/pgsql/14/data/
Remote command: ssh postgres@192.168.57.11
Using safe horizon time for smart rsync copy: 2023-07-04 18:10:21.076975-10:00
Copying the base backup.
Copying required WAL segments.
Generating archive status files
Identify dangerous settings in destination directory.

Recovery completed (start time: 2023-07-05 12:12:36.830735+00:00, elapsed time: 5 seconds)
Your PostgreSQL server has been successfully prepared for recovery!
```
На хосте `node1` потребуется перезапустить `postgresql-сервер` и снова проверить список БД

```
[root@node1 vagrant]# systemctl restart postgresql-14.service 
[root@node1 vagrant]# sudo -u postgres psql
could not change directory to "/home/vagrant": Permission denied
psql (14.8)
Type "help" for help.

postgres=# \l
                                  List of databases
    Name    |  Owner   | Encoding |   Collate   |    Ctype    |   Access privileges   
------------+----------+----------+-------------+-------------+-----------------------
 otus_2023  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
 otus_linux | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
 postgres   | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | 
 template0  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
            |          |          |             |             | postgres=CTc/postgres
 template1  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 | =c/postgres          +
            |          |          |             |             | postgres=CTc/postgres
(5 rows)
```
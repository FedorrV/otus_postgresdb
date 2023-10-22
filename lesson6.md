### Смотрим список кластеров PG

```bash
student:~$ sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
12  main    5432 online postgres /var/lib/postgresql/12/main /var/log/postgresql/postgresql-12-main.log
16  main    5433 online postgres /var/lib/postgresql/16/main /var/log/postgresql/postgresql-16-main.log
```

### Создаем тестовую таблицу

```bash
student:~$ sudo -u postgres psql -p 5433
psql (16.0 (Ubuntu 16.0-1.pgdg20.04+1))
Type "help" for help.

postgres=# create table tab (idd integer);
CREATE TABLE
postgres=# \dt
        List of relations
 Schema | Name | Type  |  Owner   
--------+------+-------+----------
 public | tab  | table | postgres
(1 row)
postgres=# insert into tab values (1);
INSERT 0 1
postgres=# insert into tab values (2);
INSERT 0 1
postgres=#
```



### Останавливаем сервер

```bash
student:~$ sudo -u postgres pg_ctlcluster 16 main stop
Warning: stopping the cluster using pg_ctlcluster will mark the systemd unit as failed. Consider using systemctl:
  sudo systemctl stop postgresql@16-main
student:~$ sudo -u postgres pg_ctlcluster 16 main status
pg_ctl: no server running
student:~$ sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
12  main    5432 online postgres /var/lib/postgresql/12/main /var/log/postgresql/postgresql-12-main.log
16  main    5433 down   postgres /var/lib/postgresql/16/main /var/log/postgresql/postgresql-16-main.log
student:~$ 
```

### Преносим файлы бд в новый созданный диск, сервер не запускается, т.к. файлы перенесли

```bash
postgres@student:/mnt$ mv /var/lib/postgresql/16/main /mnt/data
postgres@student:/mnt$ ls
data
postgres@student:/mnt$ cd data/
postgres@student:/mnt/data$ ls
main
postgres@student:/mnt/data$ pg_ctlcluster 16 main start
Error: /var/lib/postgresql/16/main is not accessible or does not exist
postgres@student:/mnt/data$ 

```

## Изеняем конфигурационный файл, сервер успешно запускается

```bash
postgres@student:/mnt/data/16$ nano /etc/postgresql/16/main/postgresql.conf 
postgres@student:/mnt/data/16$ pg_ctlcluster 16 main start
Warning: the cluster will not be running as a systemd service. Consider using systemctl:
  sudo systemctl start postgresql@16-main
postgres@student:/mnt/data/16$ pg_lsclusters 
Ver Cluster Port Status Owner    Data directory              Log file
12  main    5432 down   postgres /var/lib/postgresql/12/main /var/log/postgresql/postgresql-12-main.log
16  main    5433 online postgres /mnt/data/16                /var/log/postgresql/postgresql-16-main.log
postgres@student:/mnt/data/16$ psql -p 5433
psql (16.0 (Ubuntu 16.0-1.pgdg20.04+1))
Type "help" for help.

postgres=# select * from tab;
 idd 
-----
   1
   2
(2 rows)

```

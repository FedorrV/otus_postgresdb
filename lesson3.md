# Установка PostgreSQL 

### Установливаем docker на Ubuntu

```bash
sudo curl -fsSL https://get.docker.com -o get-docker.sh && sudo sh get-docker.sh && rm get-docker.sh && sudo usermod -aG docker $USER && newgrp docker
```

### Создаем docker-сеть

```bash
sudo docker network create pg-net
```

### Запускаем контейнер с Postgres:

```bash
sudo docker run --name pg-server --network pg-net -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres:15
```

### Запускаем отдельный контейнер с клиентом внутри одной сети с БД

```bash
sudo docker run -it --rm --network pg-net --name pg-client postgres:15 psql -h pg-server -U postgres
```

### Смотрим список запущенных контейнеров

```bash
fdudnikov@otuspg:~$ sudo docker ps -a
CONTAINER ID   IMAGE         COMMAND                  CREATED              STATUS              PORTS
                   NAMES
7c506f9bbe46   postgres:15   "docker-entrypoint.s…"   About a minute ago   Up About a minute   5432/tcp
                   pg-client
1a6c728824f9   postgres:15   "docker-entrypoint.s…"   2 minutes ago        Up About a minute   0.0.0.0:5432->5432/tcp, :::5432->5432/tcp   pg-server
```

### Подключяемся к бд извне ВМ

```bash
psql -p 5432 -U postgres -h 158.160.6.133 -d postgres -W
Password:
psql (15.3 (Ubuntu 15.3-1.pgdg22.04+1), server 15.4 (Debian 15.4-1.pgdg120+1))
Type "help" for help.

postgres=# create table aaa (idd integer);
CREATE TABLE
postgres=# insert into aaa values (234);
INSERT 0 1
postgres=# commit;
WARNING:  there is no transaction in progress
COMMIT
postgres=#
```

 ### Заходим внутрь контейнера

```bash
sudo docker exec -it pg-server bash
root@1a6c728824f9:/# psql -U postgres
psql (15.4 (Debian 15.4-1.pgdg120+1))
Type "help" for help.

postgres=# create table tab (idd integer);
CREATE TABLE
postgres=# commit;
WARNING:  there is no transaction in progress
COMMIT
postgres=# insert into tab
postgres-# (1 ,2 ,5);
ERROR:  syntax error at or near "1"
LINE 2: (1 ,2 ,5);
         ^
postgres=# insert into tab
values (1 ,2 ,5);
ERROR:  INSERT has more expressions than target columns
LINE 2: values (1 ,2 ,5);
                   ^
postgres=# insert into tab (idd)
values (1 ,2 ,5);
ERROR:  INSERT has more expressions than target columns
LINE 2: values (1 ,2 ,5);
                   ^
postgres=# insert into tab (idd)
values (1 );
INSERT 0 1
postgres=# insert into tab (idd)
values (2);
INSERT 0 1
postgres=# insert into tab (idd)
values (5);
INSERT 0 1
postgres=# commit;
```

### Рестарт контейнера и проверка, сохранились ли изменения

```bash
fdudnikov@otuspg:~$ sudo docker stop 1a6c728824f9
1a6c728824f9
fdudnikov@otuspg:~$ docker start 1a6c728824f9
1a6c728824f9
fdudnikov@otuspg:~$ psql -h localhost -U postgres -d postgres
Password for user postgres:
psql (15.3 (Ubuntu 15.3-1.pgdg22.04+1), server 15.4 (Debian 15.4-1.pgdg120+1))
Type "help" for help.

postgres=# \dt
        List of relations
 Schema | Name | Type  |  Owner
--------+------+-------+----------
 public | aaa  | table | postgres
 public | tab  | table | postgres
(2 rows)

postgres=#
```
# Цель
Научиться настраивать физическую и логическую репликацию.

# Описание/Пошаговая инструкция выполнения домашнего задания
- Настроить физическую репликацию между двумя нодами: мастер - слейв.
- Настроить каскадную репликацию со слейва на третью ноду.
- Настроить логическую репликацию таблицы с мастер ноды на четвертую ноду.

# Ход выполнения задания
## 1. Настроить физическую репликацию между двумя нодами: мастер - слейв.
Все 4 кластера будут находится на одной машине на разных портах, настройки на всех кластерах аналогичны:
| Кластер | Порт
|---------|-----
| otus1   | 5431
| otus2   | 5432
| otus3   | 5433
| otus4   | 5434

Проверим настройки master:
```bash
sudo -u postgres psql -p 5431 -c "SELECT name,setting FROM pg_settings WHERE name IN ('wal_level', 'max_wal_senders', 'listen_addresses');"
```
|      name        | setting
|------------------|---------
| listen_addresses | *
| max_wal_senders  | 10
| wal_level        | replica


pg_hba.conf:
```
# TYPE  DATABASE        USER            ADDRESS                 METHOD

# "local" is for Unix domain socket connections only
local   all             all                                     peer
# IPv4 local connections:
host    all             all             127.0.0.1/32            ident
# IPv6 local connections:
host    all             all             ::1/128                 ident
# Allow replication connections from localhost, by a user with the
# replication privilege.
local   replication     all                                     peer
host    replication     all             127.0.0.1/32            ident
host    replication     all             ::1/128                 ident
```
Так как кластеры находятся на одной локальной машине для настройки репликаций по socket достаточно записи:
```
local   replication     all                                     peer
```

Остановим второй кластер и очистим данные:
```bash
systemctl stop postgresql@otus2.service
rm -rf /data/pgdata/otus2/*
```

Сделаем бэкап мастера с указанием ключа записи конфигурации для репликации:
```bash
sudo -u postgres pg_basebackup -p 5431 -R -D /data/pgdata/otus2
```

На втором кластере исправим порт в конфиге:
```
port = 5432
```

И запускаем второй кластер:
```bash
systemctl start postgresql@otus2.service
```

Можно посмотреть статус демона, где уже видны процессы настроенной репликации:
```bash
systemctl status postgresql@otus{1,2}.service

● postgresql@otus1.service - PostgreSQL database server
     Loaded: loaded (/usr/lib/systemd/system/postgresql@.service; enabled; preset: disabled)
    Drop-In: /etc/systemd/system/postgresql@otus1.service.d
             └─30-postgresql-setup.conf
     Active: active (running) since Wed 2025-07-02 21:24:51 MSK; 22h ago
   Main PID: 1308 (postgres)
      Tasks: 8 (limit: 12240)
     Memory: 568.8M
        CPU: 25.210s
     CGroup: /system.slice/system-postgresql.slice/postgresql@otus1.service
             ├─ 1308 /usr/bin/postgres -D /data/pgdata/otus1
             ├─ 1309 "postgres: logger "
             ├─ 1310 "postgres: checkpointer "
             ├─ 1311 "postgres: background writer "
             ├─ 1313 "postgres: walwriter "
             ├─ 1314 "postgres: autovacuum launcher "
             ├─ 1315 "postgres: logical replication launcher "
             └─48380 "postgres: walsender postgres [local] streaming 0/10000060"

июл 02 21:24:51 vt-otuspg2 systemd[1]: Starting PostgreSQL database server...
июл 02 21:24:51 vt-otuspg2 postgres[1308]: 2025-07-02 21:24:51.843 MSK [1308] СООБЩЕНИЕ:  передача вывода в протокол процессу сбора протоколов
июл 02 21:24:51 vt-otuspg2 postgres[1308]: 2025-07-02 21:24:51.843 MSK [1308] ПОДСКАЗКА:  В дальнейшем протоколы будут выводиться в каталог "log".
июл 02 21:24:51 vt-otuspg2 systemd[1]: Started PostgreSQL database server.

● postgresql@otus2.service - PostgreSQL database server
     Loaded: loaded (/usr/lib/systemd/system/postgresql@.service; enabled; preset: disabled)
    Drop-In: /etc/systemd/system/postgresql@otus2.service.d
             └─30-postgresql-setup.conf
     Active: active (running) since Thu 2025-07-03 20:14:21 MSK; 58s ago
    Process: 48372 ExecStartPre=/usr/libexec/postgresql-check-db-dir postgresql@otus2 (code=exited, status=0/SUCCESS)
   Main PID: 48374 (postgres)
      Tasks: 6 (limit: 12240)
     Memory: 35.9M
        CPU: 39ms
     CGroup: /system.slice/system-postgresql.slice/postgresql@otus2.service
             ├─48374 /usr/bin/postgres -D /data/pgdata/otus2
             ├─48375 "postgres: logger "
             ├─48376 "postgres: checkpointer "
             ├─48377 "postgres: background writer "
             ├─48378 "postgres: startup recovering 000000010000000000000010"
             └─48379 "postgres: walreceiver streaming 0/10000060"

июл 03 20:14:21 vt-otuspg2 systemd[1]: Starting PostgreSQL database server...
июл 03 20:14:21 vt-otuspg2 postgres[48374]: 2025-07-03 20:14:21.886 MSK [48374] СООБЩЕНИЕ:  передача вывода в протокол процессу сбора протоколов
июл 03 20:14:21 vt-otuspg2 postgres[48374]: 2025-07-03 20:14:21.886 MSK [48374] ПОДСКАЗКА:  В дальнейшем протоколы будут выводиться в каталог "log".
июл 03 20:14:21 vt-otuspg2 systemd[1]: Started PostgreSQL database server.
```

Убедимся в этом, на master:
```bash
sudo -u postgres psql -p 5431 -c "SELECT * FROM pg_stat_replication; SELECT pg_current_wal_lsn();"
```
|  pid  | usesysid | usename  | application_name | client_addr | client_hostname | client_port |         backend_start         | backend_xmin |   state   |  sent_lsn  | write_lsn  | flush_lsn  | replay_lsn | write_lag | flush_lag | replay_lag | sync_priority | sync_state |          reply_time
|-------|----------|----------|------------------|-------------|-----------------|-------------|-------------------------------|--------------|-----------|------------|------------|------------|------------|-----------|-----------|------------|---------------|------------|-------------------------------
| 48380 |       10 | postgres | walreceiver      |             |                 |          -1 | 2025-07-03 20:14:21.931446|03 |              | streaming | 0/10000148 | 0/10000148 | 0/10000148 | 0/10000148 |           |           |            |             0 | async      | 2025-07-03 20:18:51.563284+03

| pg_current_wal_lsn
|--------------------
| 0/10000148

На втором кластере:
```bash
sudo -u postgres psql -p 5432 -c "SELECT * FROM pg_stat_wal_receiver; SELECT pg_last_wal_receive_lsn(); SELECT pg_last_wal_replay_lsn();"
```

|  pid  |  status   | receive_start_lsn | receive_start_tli | written_lsn | flushed_lsn | received_tli |      last_msg_send_time       |     last_msg_receipt_time     | latest_end_lsn |        latest_end_time        | slot_name |     sender_host     | sender_port |                                                                                                                                                                   conninfo                                    
|-------|-----------|-------------------|-------------------|-------------|-------------|--------------|-------------------------------|-------------------------------|----------------|-------------------------------|-----------|---------------------|-------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
| 48379 | streaming | 0/10000000        |                 1 | 0/10000148  | 0/10000148  |            1 | 2025-07-03 20:23:41.665777+03 | 2025-07-03 20:23:41.665805+03 | 0/10000148     | 2025-07-03 20:16:41.507872+03 |           | /var/run/postgresql |        5431 | user=postgres passfile=/var/lib/pgsql/.pgpass channel_binding=prefer dbname=replication port=5431 fallback_application_name=walreceiver sslmode=prefer sslcompression=0 sslcertmode=allow sslsni=1 ssl_min_protocol_version=TLSv1.2 gssencmode=prefer krbsrvname=postgres gssdelegation=0 target_session_attrs=any load_balance_hosts=disable

| pg_last_wal_receive_lsn
|-------------------------
| 0/10000148

| pg_last_wal_replay_lsn
|------------------------
| 0/10000148

И проверим запросами. На мастере:
```bash
sudo -u postgres psql -p 5431
```
```
postgres=# CREATE DATABASE otus_repl;
CREATE DATABASE
postgres=# \c otus_repl
Вы подключены к базе данных "otus_repl" как пользователь "postgres".
otus_repl=# CREATE TABLE data AS SELECT generate_series(1,10) as id, md5(random()::text)::char(10) as val;
SELECT 10
otus_repl=# SELECT * FROM data;
 id |    val
----+------------
  1 | a6deb28ae9
  2 | dc23414f22
  3 | 0d69a1aeb3
  4 | 3f0b6938f2
  5 | 35cddf8644
  6 | 06c63e0fcd
  7 | 999ba99aec
  8 | c48ee2cbeb
  9 | 49309b3767
 10 | b6016c502a
(10 строк)
```

На втором кластере:
```bash
sudo -u postgres psql -p 5432 otus_repl -c "SELECT * FROM data;";
```
| id |    val
|----|------------
|  1 | a6deb28ae9
|  2 | dc23414f22
|  3 | 0d69a1aeb3
|  4 | 3f0b6938f2
|  5 | 35cddf8644
|  6 | 06c63e0fcd
|  7 | 999ba99aec
|  8 | c48ee2cbeb
|  9 | 49309b3767
| 10 | b6016c502a

Все работает, данные реплицируются.

## 2. Настроить каскадную репликацию со слейва на третью ноду.
Порядок действий ровно такой же как на предыдущем этапе. Отличие только в том, что репликацию настраиваем со второго на третий кластер.

Остановим третий кластер и очистим данные:
```bash
systemctl stop postgresql@otus3.service
rm -rf /data/pgdata/otus3/*
```

Сделаем бэкап со второго кластера с указанием ключа записи конфигурации для репликации:
```bash
sudo -u postgres pg_basebackup -p 5432 -R -D /data/pgdata/otus3
```

На третьем кластере исправим порт в конфиге:
```
port = 5433
```

И запускаем третий кластер:
```bash
systemctl start postgresql@otus3.service
```

Посмотрим статусы демонов:
```bash
systemctl status postgresql@otus{1,2,3}.service

● postgresql@otus1.service - PostgreSQL database server
     Loaded: loaded (/usr/lib/systemd/system/postgresql@.service; enabled; preset: disabled)
    Drop-In: /etc/systemd/system/postgresql@otus1.service.d
             └─30-postgresql-setup.conf
     Active: active (running) since Wed 2025-07-02 21:24:51 MSK; 23h ago
   Main PID: 1308 (postgres)
      Tasks: 8 (limit: 12240)
     Memory: 483.7M
        CPU: 25.916s
     CGroup: /system.slice/system-postgresql.slice/postgresql@otus1.service
             ├─ 1308 /usr/bin/postgres -D /data/pgdata/otus1
             ├─ 1309 "postgres: logger "
             ├─ 1310 "postgres: checkpointer "
             ├─ 1311 "postgres: background writer "
             ├─ 1313 "postgres: walwriter "
             ├─ 1314 "postgres: autovacuum launcher "
             ├─ 1315 "postgres: logical replication launcher "
             └─48380 "postgres: walsender postgres [local] streaming 0/1042ABC8"

июл 02 21:24:51 vt-otuspg2 systemd[1]: Starting PostgreSQL database server...
июл 02 21:24:51 vt-otuspg2 postgres[1308]: 2025-07-02 21:24:51.843 MSK [1308] СООБЩЕНИЕ:  передача вывода в протокол процессу сбора протоколов
июл 02 21:24:51 vt-otuspg2 postgres[1308]: 2025-07-02 21:24:51.843 MSK [1308] ПОДСКАЗКА:  В дальнейшем протоколы будут выводиться в каталог "log".
июл 02 21:24:51 vt-otuspg2 systemd[1]: Started PostgreSQL database server.

● postgresql@otus2.service - PostgreSQL database server
     Loaded: loaded (/usr/lib/systemd/system/postgresql@.service; enabled; preset: disabled)
    Drop-In: /etc/systemd/system/postgresql@otus2.service.d
             └─30-postgresql-setup.conf
     Active: active (running) since Thu 2025-07-03 20:14:21 MSK; 41min ago
    Process: 48372 ExecStartPre=/usr/libexec/postgresql-check-db-dir postgresql@otus2 (code=exited, status=0/SUCCESS)
   Main PID: 48374 (postgres)
      Tasks: 7 (limit: 12240)
     Memory: 55.7M
        CPU: 372ms
     CGroup: /system.slice/system-postgresql.slice/postgresql@otus2.service
             ├─48374 /usr/bin/postgres -D /data/pgdata/otus2
             ├─48375 "postgres: logger "
             ├─48376 "postgres: checkpointer "
             ├─48377 "postgres: background writer "
             ├─48378 "postgres: startup recovering 000000010000000000000010"
             ├─48379 "postgres: walreceiver streaming 0/1042ABC8"
             └─49023 "postgres: walsender postgres [local] streaming 0/1042ABC8"

июл 03 20:14:21 vt-otuspg2 systemd[1]: Starting PostgreSQL database server...
июл 03 20:14:21 vt-otuspg2 postgres[48374]: 2025-07-03 20:14:21.886 MSK [48374] СООБЩЕНИЕ:  передача вывода в протокол процессу сбора протоколов
июл 03 20:14:21 vt-otuspg2 postgres[48374]: 2025-07-03 20:14:21.886 MSK [48374] ПОДСКАЗКА:  В дальнейшем протоколы будут выводиться в каталог "log".
июл 03 20:14:21 vt-otuspg2 systemd[1]: Started PostgreSQL database server.

● postgresql@otus3.service - PostgreSQL database server
     Loaded: loaded (/usr/lib/systemd/system/postgresql@.service; enabled; preset: disabled)
    Drop-In: /etc/systemd/system/postgresql@otus3.service.d
             └─30-postgresql-setup.conf
     Active: active (running) since Thu 2025-07-03 20:55:25 MSK; 20s ago
    Process: 49015 ExecStartPre=/usr/libexec/postgresql-check-db-dir postgresql@otus3 (code=exited, status=0/SUCCESS)
   Main PID: 49017 (postgres)
      Tasks: 6 (limit: 12240)
     Memory: 20.5M
        CPU: 27ms
     CGroup: /system.slice/system-postgresql.slice/postgresql@otus3.service
             ├─49017 /usr/bin/postgres -D /data/pgdata/otus3
             ├─49018 "postgres: logger "
             ├─49019 "postgres: checkpointer "
             ├─49020 "postgres: background writer "
             ├─49021 "postgres: startup recovering 000000010000000000000010"
             └─49022 "postgres: walreceiver "

июл 03 20:55:24 vt-otuspg2 systemd[1]: Starting PostgreSQL database server...
июл 03 20:55:24 vt-otuspg2 postgres[49017]: 2025-07-03 20:55:24.943 MSK [49017] СООБЩЕНИЕ:  передача вывода в протокол процессу сбора протоколов
июл 03 20:55:24 vt-otuspg2 postgres[49017]: 2025-07-03 20:55:24.943 MSK [49017] ПОДСКАЗКА:  В дальнейшем протоколы будут выводиться в каталог "log".
июл 03 20:55:25 vt-otuspg2 systemd[1]: Started PostgreSQL database server.
```

И проверим, что репликация работает. На мастере:
```bash
sudo -u postgres psql -p 5431 otus_repl -c "INSERT INTO data VALUES (11,'Проверка');"
```

На третьем кластере:
```bash
sudo -u postgres psql -p 5433 otus_repl -c "SELECT * FROM data WHERE id=11;"
```

| id |    val
|----|------------
| 11 | Проверка

Все работает. Каскадная репликация настроена.

## 3. Настроить логическую репликацию таблицы с мастер ноды на четвертую ноду.
На master задаем парметр "wal_level" и перезапускаем:
```bash
sudo -u postgres psql -p 5431 -c "ALTER SYSTEM SET wal_level = logical;"
systemctl restart postgresql@otus1.service
```

Создаем публикацию:
```bash
sudo -u postgres psql -p 5431 otus_repl -c "CREATE PUBLICATION otus_repl_data_pub FOR TABLE data;"
```
```
                              Публикация otus_repl_data_pub
 Владелец | Все таблицы | Добавления | Изменения | Удаления | Опустошения | Через корень
----------+-------------+------------+-----------+----------+-------------+--------------
 postgres | f           | t          | t         | t        | t           | f
Таблицы:
    "public.data"
```

Подготовим четвертый кластер. Создадим БД, таблицу и задаим параметр "wal_level".
```bash
sudo -u postgres psql -p 5434 -c "CREATE DATABASE otus_repl;"
sudo -u postgres psql -p 5434 otus_repl -c "CREATE TABLE data (id int, val char(10));"
sudo -u postgres psql -p 5434 -c "ALTER SYSTEM SET wal_level = logical;"
systemctl restart postgresql@otus4.service
```

Создаем подписку к master с копированием данных (подключение по socket, параметр host не указываем или указываем путь до сокета):
```bash
sudo -u postgres psql -p 5434 otus_repl -c "CREATE SUBSCRIPTION otus_repl_data_sub \
CONNECTION 'port=5431 user=postgres password=qaz123 dbname=otus_repl' \
PUBLICATION otus_repl_data_pub WITH (copy_data = true);"

ЗАМЕЧАНИЕ:  на сервере публикации создан слот репликации "otus_repl_data_sub"
CREATE SUBSCRIPTION
```

Проверяем:
```bash
systemctl status postgresql@otus4

● postgresql@otus4.service - PostgreSQL database server
     Loaded: loaded (/usr/lib/systemd/system/postgresql@.service; enabled; preset: disabled)
    Drop-In: /etc/systemd/system/postgresql@otus4.service.d
             └─30-postgresql-setup.conf
     Active: active (running) since Thu 2025-07-03 22:49:43 MSK; 3min 29s ago
    Process: 50274 ExecStartPre=/usr/libexec/postgresql-check-db-dir postgresql@otus4 (code=exited, status=0/SUCCESS)
   Main PID: 50276 (postgres)
      Tasks: 8 (limit: 12240)
     Memory: 22.5M
        CPU: 111ms
     CGroup: /system.slice/system-postgresql.slice/postgresql@otus4.service
             ├─50276 /usr/bin/postgres -D /data/pgdata/otus4
             ├─50277 "postgres: logger "
             ├─50278 "postgres: checkpointer "
             ├─50279 "postgres: background writer "
             ├─50281 "postgres: walwriter "
             ├─50282 "postgres: autovacuum launcher "
             ├─50283 "postgres: logical replication launcher "
             └─50342 "postgres: logical replication apply worker for subscription 16396 "

июл 03 22:49:43 vt-otuspg2 systemd[1]: Starting PostgreSQL database server...
июл 03 22:49:43 vt-otuspg2 postgres[50276]: 2025-07-03 22:49:43.118 MSK [50276] СООБЩЕНИЕ:  передача вывода в протокол процессу сбора протоколов
июл 03 22:49:43 vt-otuspg2 postgres[50276]: 2025-07-03 22:49:43.118 MSK [50276] ПОДСКАЗКА:  В дальнейшем протоколы будут выводиться в каталог "log".
июл 03 22:49:43 vt-otuspg2 systemd[1]: Started PostgreSQL database server.
```

На четвертом кластере:
```bash
sudo -u postgres psql -p 5434 otus_repl -c "SELECT * FROM data;"
```
| id |    val
|----|------------
|  1 | a6deb28ae9
|  2 | dc23414f22
|  3 | 0d69a1aeb3
|  4 | 3f0b6938f2
|  5 | 35cddf8644
|  6 | 06c63e0fcd
|  7 | 999ba99aec
|  8 | c48ee2cbeb
|  9 | 49309b3767
| 10 | b6016c502a
| 11 | Проверка

Данные появились. Теперь добавим что-нибудь на master:
```bash
sudo -u postgres psql -p 5431 otus_repl -c "INSERT INTO data VALUES (12,'Логическая');"
```

На четвертом кластере:
```bash
sudo -u postgres psql -p 5434 otus_repl -c "SELECT * FROM data;"
```
| id |    val
|----|------------
|  1 | a6deb28ae9
|  2 | dc23414f22
|  3 | 0d69a1aeb3
|  4 | 3f0b6938f2
|  5 | 35cddf8644
|  6 | 06c63e0fcd
|  7 | 999ba99aec
|  8 | c48ee2cbeb
|  9 | 49309b3767
| 10 | b6016c502a
| 11 | Проверка
| 12 | Логическая

Данные есть, а значит логическая репликация работает.
# Цель
Получить навык создания бекапов и восстановления из них.

# Описание/Пошаговая инструкция выполнения домашнего задания
- Создать бекапы с помощью pg_dump, pg_dumpall и pg_basebackup сравнить скорость создания и возможности.
- Настроить копирование WAL файлов.
- Восстановить базу на другой машине PostgreSQL на заданное время, используя ранее созданные бекапы и WAL файлы.

# Ход выполнения задания
## 1. Создать бекапы с помощью pg_dump, pg_dumpall и pg_basebackup сравнить скорость создания и возможности.

Для экпериментов с бэкапами будем использовать готовую демо-базу [demo-small](https://postgrespro.ru/education/demodb).

### pg_dump

\+ Выдает на консоль или в файл либо SQL-скрипт, либо архив в специальном формате с оглавлением  
\+ Поддерживает параллельное выполнение  
\+ Позволяет ограничить набор выгружаемых объектов (таблицы --table, схемы --schema-only, данные --data-only и т.п.)  
\- По умолчанию не создает tablespace и юзеров  

Создание бэкапа БД "demo":

```bash
time pg_dump -d demo -C -U postgres -p 5431 > /data/pgbackup/otus1_demo_pg_dump.sql

real    0m0,577s
user    0m0,157s
sys     0m0,061s
```

Восстановим бэкап в другом кластере PG. Предварительно необходимо создать БД.

```bash
psql -U postgres -p 5432
psql (16.8)
Введите "help", чтобы получить справку.

postgres=# CREATE DATABASE demo;
CREATE DATABASE
```

```bash
time psql -U postgres -p 5432 demo < /data/pgbackup/otus1_demo_pg_dump.sql
...

real    0m9,040s
user    0m0,073s
sys     0m0,023s
```

### pg_dump (архив)
Создание бэкапа БД "demo", также добавим ключ -C для добавления в копию команды создания базы данных:

```bash
time pg_dump -d demo -C -U postgres -Fc -p 5431 > /data/pgbackup/otus1_demo_pg_dump.gz

real    0m2,383s
user    0m2,343s
sys     0m0,018s
```

Это заняло больше времени из-за сжатия, зато итоговый бэкап занимает меньше дискового пространнства:

```bash
ls -lh /data/pgbackup/
итого 121M
-rw-r--r--. 1 postgres postgres  22M июн 22 12:26 otus1_demo_pg_dump.gz
-rw-r--r--. 1 postgres postgres 100M июн 22 12:06 otus1_demo_pg_dump.sql
```

Восстановим бэкап в другом кластере PG.
```bash
time pg_restore -d demo -U postgres -p 5432 /data/pgbackup/otus1_demo_pg_dump.gz

real    0m6,487s
user    0m0,275s
sys     0m0,020s
```

Интересно, что восстановление заняло меньше времени (повторные попытки показывали до 8 секунд) чем незжатая версия. Возможно это связанно с использованием утилиты pg_restore или при таком объеме БД разница в пределах погрешности.

### pg_dumpall
\+ Сохраняет весь кластер, включая роли и табличные пространства  
\+ Выдает на консоль или в файл SQL-скрипт  
\+ Можно выгрузить только глобальные объекты и воспользоваться pg_dump  
\- Параллельное выполнение не поддерживается  

Создание бэкапа всех БД:
```bash
time pg_dumpall -U postgres -p 5431 > /data/pgbackup/otus1_demo_pg_dump_all.sql

real    0m0,665s
user    0m0,172s
sys     0m0,076s
```

Время создания соизмеримо с вариантом "pg_dump" без жатия, полагаю из-за того что в кластере единственная БД с данными.

Восстановим бэкап в другом кластере PG.

```bash
time psql -U postgres -p 5432 < /data/pgbackup/otus1_demo_pg_dump_all.sql

real    0m7,895s
user    0m0,069s
sys     0m0,019s
```
Опять же - время восстановления в пределах погрешности.

## 2. Настроить копирование WAL файлов.
Создаем каталог для WAL файлов:
```bash
mkdir /data/pgdata/otus1/pg_archive
chmod 700 /data/pgdata/otus1/pg_archive
chown postgres:postgres /data/pgdata/otus1/pg_archive
```

Включаем копирование WAL файлов в PG:
```bash
sudo -u postgres psql -p 5431 -c "ALTER SYSTEM SET archive_mode = 'on';"
sudo -u postgres psql -p 5431 -c "ALTER SYSTEM SET archive_command = 'test ! -f /data/pgdata/otus1/pg_archive/%f && cp %p /data/pgdata/otus1/pg_archive/%f';"
```

Перезапускаем PG и проверяем настройки:
```bash
systemctl restart postgresql@otus1.service
sudo -u postgres psql -p 5431 -c "SELECT name, setting FROM pg_settings WHERE name IN ('archive_mode','archive_command','archive_timeout');"
```

|      name       |                      setting
|-----------------|----------------------------------------------------
| archive_command | test ! -f /data/pgdata/otus1/pg_archive/%f && cp %p /data/pgdata/otus1/pg_archive/%f
| archive_mode    | on
| archive_timeout | 0

## 3. Восстановить базу на другой машине PostgreSQL на заданное время, используя ранее созданные бекапы и WAL файлы.
### Создание бэкапа
Подготовим каталоги для бэкапа:
```bash
mkdir /data/pgbackup/otus1
chmod 700 /data/pgbackup/otus1
chown postgres:postgres /data/pgbackup/otus1
mkdir /data/pgbackup/otus2
chmod 700 /data/pgbackup/otus2
chown postgres:postgres /data/pgbackup/otus2
```

Создадим бэкап с помощью pg_basebackup:
```bash
sudo -u postgres pg_basebackup -p 5431 -v -D /data/pgbackup/otus1
pg_basebackup: начинается базовое резервное копирование, ожидается завершение контрольной точки
pg_basebackup: контрольная точка завершена
pg_basebackup: стартовая точка в журнале предзаписи: 0/20000028 на линии времени 1
pg_basebackup: запуск фонового процесса считывания WAL
pg_basebackup: создан временный слот репликации "pg_basebackup_136265"
pg_basebackup: конечная точка в журнале предзаписи: 0/20000138
pg_basebackup: ожидание завершения потоковой передачи фоновым процессом...
pg_basebackup: сохранение данных на диске...
pg_basebackup: переименование backup_manifest.tmp в backup_manifest
pg_basebackup: базовое резервное копирование завершено
```

Внесем некоторые изменения в БД:
```bash
sudo -u postgres psql -p 5431 demo
```

```
demo=# CREATE TABLE test (c1 TEXT);
CREATE TABLE
demo=# INSERT INTO test VALUES ('Проверка восстановления с использованием WAL');
INSERT 0 1
demo=# SELECT NOW();
             now
------------------------------
 2025-06-29 13:43:51.47634+03
(1 строка)

demo=# INSERT INTO test VALUES ('Этой записи не должно быть после восстановления');
INSERT 0 1
demo=# SELECT * FROM test;
                       c1
-------------------------------------------------
 Проверка восстановления с использованием WAL
 Этой записи не должно быть после восстановления
(2 строки)
```

### Восстановление из бэкапа
Останавливаем кластеры:
```bash
systemctl stop postgresql@otus1.service
systemctl stop postgresql@otus2.service
```

Очищаем каталог воторого кластера:
```bash
rm -rf /data/pgdata/otus2/*
```

Копируем в него полную резервную копию:
```bash
cp -r /data/pgbackup/otus1/* /data/pgdata/otus2/
```

В востановленной версии очищаем каталог pg_wal и переносим последний вариант WAL файлов из первого кластера:
```bash
rm -rf /data/pgdata/otus2/pg_wal/*
cp -r /data/pgdata/otus1/pg_wal/* /data/pgdata/otus2/pg_wal/
```

Поправим конфиги. В "postgresql.conf":
```
port = 5432
restore_command = 'cp /data/pgdata/otus1/pg_archive/%f "%p"'
recovery_target_time = '2025-06-29 13:43:51.47634+03'
```

Так как у нас второй кластер находится на этой же машине, необходимо поправить параметр "archive_command" в "postgresql.auto.conf":
```
archive_command = 'test ! -f /data/pgdata/otus2/pg_archive/%f && cp %p /data/pgdata/otus2/pg_archive/%f'
```

Создаем пустой файл recovery.signal:
```bash
touch /data/pgdata/otus2/recovery.signal
```

Для всего скопированного назаначем права:
```bash
chown -R postgres:postgres /data/pgdata/otus2/*
```

Запускаем кластер:
```bash
systemctl start postgresql@otus2.service
```

Проверяем:
```bash
sudo -u postgres psql -p 5432 demo -c "SELECT * FROM test;"
                      c1
----------------------------------------------
 Проверка восстановления с использованием WAL
(1 строка)
```

В логе:
```
2025-06-29 14:39:38.915 MSK [137001] СООБЩЕНИЕ:  восстановление останавливается перед фиксированием транзакции 1168, время 2025-06-29 13:43:56.432676+03
2025-06-29 14:39:38.915 MSK [137001] СООБЩЕНИЕ:  остановка в конце восстановления
2025-06-29 14:39:38.915 MSK [137001] ПОДСКАЗКА:  Выполните pg_wal_replay_resume() для повышения.
```

Восстановление прошло успешно, можно перевести кластер в рабочий режим:
```bash
sudo -u postgres psql -p 5432 demo -c "select pg_wal_replay_resume();"
 pg_wal_replay_resume
----------------------

(1 строка)
```

И в логе:
```
2025-06-29 14:44:26.523 MSK [136996] СООБЩЕНИЕ:  система БД готова принимать подключения
```

Попробуем что-нибудь записать:
```bash
sudo -u postgres psql -p 5432 demo -c "INSERT INTO test VALUES ('Новая запись после восстановления');"
sudo -u postgres psql -p 5432 demo -c "SELECT * FROM test;"
```

|                      c1
|----------------------------------------------
| Проверка восстановления с использованием WAL
| Новая запись после восстановления
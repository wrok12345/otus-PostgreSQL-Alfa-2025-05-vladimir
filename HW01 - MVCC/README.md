# Цель
Получить навык настройки параметров autovacuum.

# Описание/Пошаговая инструкция выполнения домашнего задания
- Выполнить pgbench -c8 -P 6 -T 60 -U postgres postgres.
- Настроить параметры vacuum/autovacuum.
- Заново протестировать: pgbench -c8 -P 6 -T 60 -U postgres postgres и сравнить результаты.
- Создать таблицу с текстовым полем и заполнить случайными или сгенерированными данным в размере 1млн строк.
- Посмотреть размер файла с таблицей.
- 5 раз обновить все строчки и добавить к каждой строчке любой символ.
- Посмотреть количество мертвых строчек в таблице и когда последний раз приходил автовакуум.
- Отключить Автовакуум на таблице и опять 5 раз обновить все строки.
- Объяснить полученные результаты.

# Ход выполнения задания
1. Выполнить pgbench -c8 -P 6 -T 60 -U postgres postgres.

```
[root@localhost ~]# sudo -u postgres /usr/pgsql-17/bin/pgbench -c8 -P 6 -T 60 benchtest
pgbench (17.5)
starting vacuum...end.
progress: 6.0 s, 967.7 tps, lat 8.250 ms stddev 4.197, 0 failed
progress: 12.0 s, 972.3 tps, lat 8.225 ms stddev 4.085, 0 failed
progress: 18.0 s, 965.2 tps, lat 8.285 ms stddev 4.093, 0 failed
progress: 24.0 s, 962.5 tps, lat 8.310 ms stddev 3.967, 0 failed
progress: 30.0 s, 975.7 tps, lat 8.197 ms stddev 4.000, 0 failed
progress: 36.0 s, 968.5 tps, lat 8.259 ms stddev 4.214, 0 failed
progress: 42.0 s, 963.7 tps, lat 8.298 ms stddev 4.255, 0 failed
progress: 48.0 s, 978.3 tps, lat 8.176 ms stddev 4.032, 0 failed
progress: 54.0 s, 986.5 tps, lat 8.105 ms stddev 3.954, 0 failed
progress: 60.0 s, 987.8 tps, lat 8.098 ms stddev 3.919, 0 failed
transaction type: <builtin: TPC-B (sort of)>      
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 58377  
number of failed transactions: 0 (0.000%)
latency average = 8.220 ms
latency stddev = 4.073 ms
initial connection time = 6.414 ms
tps = 972.888822 (without initial connection time)
```

2.Настроить параметры vacuum/autovacuum.

```
ALTER SYSTEM SET vacuum_cost_page_miss = 5;
ALTER SYSTEM SET vacuum_cost_page_dirty = 10;
ALTER SYSTEM SET autovacuum_naptime = '15s';
ALTER SYSTEM SET autovacuum_vacuum_threshold = 25;
ALTER SYSTEM SET autovacuum_analyze_scale_factor = 0.01;
ALTER SYSTEM SET autovacuum_vacuum_scale_factor = 0.02;
ALTER SYSTEM SET autovacuum_vacuum_insert_scale_factor = 0.05;
```

|                 name                  |  setting   |  context   |                                                   short_desc
|---------------------------------------|------------|------------|-----------------------------------------------------------------------------------------------------------------
| autovacuum                            | on         | sighup     | Запускает подпроцесс автоочистки.
| **autovacuum_analyze_scale_factor**   | **0.01**   | sighup     | Отношение числа добавлений, обновлений или удалений кортежей к reltuples, определяющее потребность в анализе.
| autovacuum_analyze_threshold          | 50         | sighup     | Минимальное число добавлений, изменений или удалений кортежей, вызывающее анализ.
| autovacuum_freeze_max_age             | 200000000  | postmaster | Возраст, при котором необходима автоочистка таблицы для предотвращения зацикливания ID транзакций.
| autovacuum_max_workers                | 3          | postmaster | Задаёт предельное число одновременно выполняющихся рабочих процессов автоочистки.
| autovacuum_multixact_freeze_max_age   | 400000000  | postmaster | Возраст multixact, при котором необходима автоочистка таблицы для предотвращения зацикливания multixact.
| **autovacuum_naptime**                | **15**     | sighup     | Время простоя между запусками автоочистки.
| autovacuum_vacuum_cost_delay          | 2          | sighup     | Задержка очистки для автоочистки (в миллисекундах).
| autovacuum_vacuum_cost_limit          | -1         | sighup     | Суммарная стоимость очистки, при которой нужна передышка, для автоочистки.
| **autovacuum_vacuum_insert_scale_factor** | **0.05**   | sighup     | Отношение числа добавлений кортежей к reltuples, определяющее потребность в очистке.
| autovacuum_vacuum_insert_threshold    | 1000       | sighup     | Минимальное число добавлений кортежей, вызывающее очистку; при -1 такая очистка отключается.
| **autovacuum_vacuum_scale_factor**    | **0.02**   | sighup     | Отношение числа обновлений или удалений кортежей к reltuples, определяющее потребность в очистке.
| **autovacuum_vacuum_threshold**       | **25**     | sighup     | Минимальное число изменений или удалений кортежей, вызывающее очистку.
| autovacuum_work_mem                   | -1         | sighup     | Задаёт предельный объём памяти для каждого рабочего процесса автоочистки.
| log_autovacuum_min_duration           | 600000     | sighup     | Задаёт предельное время выполнения автоочистки, при превышении которого эта операция протоколируется в журнале.
| vacuum_buffer_usage_limit             | 2048       | user       | Задаёт размер пула буферов для операций VACUUM, ANALYZE и автоочистки.
| vacuum_cost_delay                     | 0          | user       | Задержка очистки (в миллисекундах).
| vacuum_cost_limit                     | 200        | user       | Суммарная стоимость очистки, при которой нужна передышка.
| **vacuum_cost_page_dirty**            | **10**     | user       | Стоимость очистки для страницы, которая не была "грязной".
| vacuum_cost_page_hit                  | 1          | user       | Стоимость очистки для страницы, найденной в кеше.
| **vacuum_cost_page_miss**             | **5**      | user       | Стоимость очистки для страницы, не найденной в кеше.
| vacuum_failsafe_age                   | 1600000000 | user       | Возраст, при котором VACUUM должен включить защиту от зацикливания во избежание отказа.
| vacuum_freeze_min_age                 | 50000000   | user       | Минимальный возраст строк таблицы, при котором VACUUM может их заморозить.
| vacuum_freeze_table_age               | 150000000  | user       | Возраст, при котором VACUUM должен сканировать всю таблицу с целью заморозить кортежи.
| vacuum_multixact_failsafe_age         | 1600000000 | user       | Возраст мультитранзакций, при котором VACUUM должен включить защиту от зацикливания во избежание отказа.
| vacuum_multixact_freeze_min_age       | 5000000    | user       | Минимальный возраст, при котором VACUUM будет замораживать MultiXactId в строке таблицы.
| vacuum_multixact_freeze_table_age     | 150000000  | user       | Возраст multixact, при котором VACUUM должен сканировать всю таблицу с целью заморозить кортежи.

3. Заново протестировать: pgbench -c8 -P 6 -T 60 -U postgres postgres и сравнить результаты.

```
[root@localhost ~]# sudo -u postgres /usr/pgsql-17/bin/pgbench -c8 -P 6 -T 60 benchtest
pgbench (17.5)
starting vacuum...end.
progress: 6.0 s, 826.0 tps, lat 9.663 ms stddev 5.744, 0 failed
progress: 12.0 s, 956.0 tps, lat 8.368 ms stddev 4.259, 0 failed
progress: 18.0 s, 974.5 tps, lat 8.205 ms stddev 4.184, 0 failed
progress: 24.0 s, 965.3 tps, lat 8.285 ms stddev 4.254, 0 failed
progress: 30.0 s, 961.0 tps, lat 8.324 ms stddev 4.002, 0 failed
progress: 36.0 s, 950.8 tps, lat 8.411 ms stddev 4.632, 0 failed
progress: 42.0 s, 960.8 tps, lat 8.322 ms stddev 4.309, 0 failed
progress: 48.0 s, 966.2 tps, lat 8.278 ms stddev 4.359, 0 failed
progress: 54.0 s, 947.8 tps, lat 8.438 ms stddev 4.265, 0 failed
progress: 60.0 s, 982.3 tps, lat 8.141 ms stddev 4.337, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 56953
number of failed transactions: 0 (0.000%)
latency average = 8.426 ms
latency stddev = 4.455 ms
initial connection time = 6.513 ms
tps = 949.127196 (without initial connection time)
```

Количество транзакций в секунду уменьшилось на 2.5%. Полагаю из-за более частой сработки autovacuum.

4. Создать таблицу с текстовым полем и заполнить случайными или сгенерированными данным в размере 1млн строк.

```
CREATE DATABASE test;

CREATE TABLE strtable(
  val text
);

INSERT INTO strtable(val) SELECT 'randomtext' FROM generate_series(1,1000000);
```

5. Посмотреть размер файла с таблицей.
```
SELECT pg_size_pretty(pg_total_relation_size('strtable'));
```

| pg_size_pretty 
|----------------
| 42 MB

6. 5 раз обновить все строчки и добавить к каждой строчке любой символ.
```
UPDATE strtable SET val = val || 'q';
UPDATE strtable SET val = val || 'w';
UPDATE strtable SET val = val || 'e';
UPDATE strtable SET val = val || 'r';
UPDATE strtable SET val = val || 't';
```

7. Посмотреть количество мертвых строчек в таблице и когда последний раз приходил автовакуум.
```
SELECT pg_size_pretty(pg_total_relation_size('strtable'));
```

| pg_size_pretty 
|----------------
| 253 MB

```
SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum 
FROM pg_stat_user_TABLEs WHERE relname = 'strtable';
```

| relname  | n_live_tup | n_dead_tup | ratio% |        last_autovacuum        
|----------|------------|------------|--------|-------------------------------
| strtable |    1000000 |          0 |      0 | 2025-05-25 14:51:42.303912+03

8. Отключить Автовакуум на таблице и опять 5 раз обновить все строки.
```
ALTER TABLE strtable SET (autovacuum_enabled = off);

UPDATE strtable SET val = val || 'q';
UPDATE strtable SET val = val || 'w';
UPDATE strtable SET val = val || 'e';
UPDATE strtable SET val = val || 'r';
UPDATE strtable SET val = val || 't';
```

```
SELECT pg_size_pretty(pg_total_relation_size('strtable'));
```

| pg_size_pretty 
|----------------
| 291 MB

```
SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum 
FROM pg_stat_user_TABLEs WHERE relname = 'strtable';
```

| relname  | n_live_tup | n_dead_tup | ratio% |        last_autovacuum        
|----------|------------|------------|--------|-------------------------------
| strtable |    1000000 |    5000000 |    500 | 2025-05-25 14:51:42.303912+03

```
ALTER TABLE strtable SET (autovacuum_enabled = on);
```

9. Объяснить полученные результаты.

- При первом обновлении строк сработал autovacuum (2025-05-25 14:51:42), поэтому "мертвых" строк нет.
- После отключения autovacuum и пятикратного обновления всех строк должны появится "мертвые строки" в количестве `"живых" строк * 5`. Так как при обновлении строки она фактически не удаляется: создается новая, а старая помечается на удаление.
  autovacum не запустился (что подтверждается временем последнего запуска) и не удалил "мертвые" строки.

# Примечание
Интересный момент, что при повторном проведении всей последовательности действий (с пересозданием БД) "живых" и "мертвых" строк оказалось меньше/больше ожидаемого:
| relname  | n_live_tup | n_dead_tup | ratio% |        last_autovacuum        
|----------|------------|------------|--------|-------------------------------
| strtable |     997111 |          0 |      0 | 2025-05-25 15:05:27.336951+03

При втором обновлении с выключенным autovacum:
| relname  | n_live_tup | n_dead_tup | ratio% |        last_autovacuum        
|----------|------------|------------|--------|-------------------------------
| strtable |     997111 |    4999117 |    501 | 2025-05-25 15:05:27.336951+03

И после включения autovacum:
| relname  | n_live_tup | n_dead_tup | ratio% |        last_autovacuum        
|----------|------------|------------|--------|-------------------------------
| strtable |    1001351 |          0 |      0 | 2025-05-25 15:09:12.855482+03

При этом ошибок в данных нет.
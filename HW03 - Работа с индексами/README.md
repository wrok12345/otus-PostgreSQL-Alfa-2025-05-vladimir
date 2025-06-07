# Цель
знать виды индексов, их особенности;
научиться использовать различные виды индексов.

# Описание/Пошаговая инструкция выполнения домашнего задания
- Написать запросы поиска данных без индексов, посмотреть их план запросов.
- Добавить на таблицы индексы с целью оптимизации запросов поиска данных.
- Сравнить новые планы запросов с предыдущими.
- Сравнить применение различных типов индексов

# Ход выполнения задания
## 1. Написать запросы поиска данных без индексов, посмотреть их план запросов.

Для запросов будем использовать готовую демо-базу [demo-big](https://postgrespro.ru/education/demodb).

```
demo=# \d tickets;
                                   Таблица "bookings.tickets"
    Столбец     |          Тип          | Правило сортировки | Допустимость NULL | По умолчанию
----------------+-----------------------+--------------------+-------------------+--------------
 ticket_no      | character(13)         |                    | not null          |
 book_ref       | character(6)          |                    | not null          |
 passenger_id   | character varying(20) |                    | not null          |
 passenger_name | text                  |                    | not null          |
 contact_data   | jsonb                 |                    |                   |
Индексы:
    "tickets_pkey" PRIMARY KEY, btree (ticket_no)
Ограничения внешнего ключа:
    "tickets_book_ref_fkey" FOREIGN KEY (book_ref) REFERENCES bookings(book_ref)
Ссылки извне:
    TABLE "ticket_flights" CONSTRAINT "ticket_flights_ticket_no_fkey" FOREIGN KEY (ticket_no) REFERENCES tickets(ticket_no)
```

Видим наличие индекса на поле "ticket_no". 

Выполним запрос поиска пассажира "ARTEM TIHON".
``` sql
explain (costs, verbose, analyze) SELECT * FROM bookings.tickets WHERE passenger_name = 'ARTEM TIHON';
```

```
                                                            QUERY PLAN
-----------------------------------------------------------------------------------------------------------------------------------
 Gather  (cost=1000.00..72130.78 rows=258 width=104) (actual time=77.253..78.212 rows=0 loops=1)
   Output: ticket_no, book_ref, passenger_id, passenger_name, contact_data
   Workers Planned: 1
   Workers Launched: 1
   ->  Parallel Seq Scan on bookings.tickets  (cost=0.00..71104.98 rows=152 width=104) (actual time=76.246..76.246 rows=0 loops=2)
         Output: ticket_no, book_ref, passenger_id, passenger_name, contact_data
         Filter: (tickets.passenger_name = 'ARTEM TIHON'::text)
         Rows Removed by Filter: 1474928
         Worker 0:  actual time=75.469..75.469 rows=0 loops=1
 Planning Time: 0.182 ms
 Execution Time: 78.226 ms
```

Видим, что происходит последовательное сканирование таблицы ("Seq Scan on bookings.tickets"). Индексы не используются.

## 2. Добавить на таблицы индексы с целью оптимизации запросов поиска данных.
Добавим индекс на поле "contact_data".

``` sql
create index on tickets(passenger_name);
```

```
demo=# \d tickets;
                                   Таблица "bookings.tickets"
    Столбец     |          Тип          | Правило сортировки | Допустимость NULL | По умолчанию
----------------+-----------------------+--------------------+-------------------+--------------
 ticket_no      | character(13)         |                    | not null          |
 book_ref       | character(6)          |                    | not null          |
 passenger_id   | character varying(20) |                    | not null          |
 passenger_name | text                  |                    | not null          |
 contact_data   | jsonb                 |                    |                   |
Индексы:
    "tickets_pkey" PRIMARY KEY, btree (ticket_no)
    "tickets_passenger_name_idx" btree (passenger_name)
Ограничения внешнего ключа:
    "tickets_book_ref_fkey" FOREIGN KEY (book_ref) REFERENCES bookings(book_ref)
Ссылки извне:
    TABLE "ticket_flights" CONSTRAINT "ticket_flights_ticket_no_fkey" FOREIGN KEY (ticket_no) REFERENCES tickets(ticket_no)
```

Появился индекс "tickets_passenger_name_idx" с типом "btree".

## 3. Сравнить новые планы запросов с предыдущими.

Проверим использование индекса при выполнении запроса.

``` sql
analyze tickets;
vacuum tickets;
explain (costs, verbose, analyze) SELECT * FROM bookings.tickets WHERE passenger_name = 'ARTEM TIHON';
```
```
                                                             QUERY PLAN
-------------------------------------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on bookings.tickets  (cost=3.69..325.99 rows=259 width=104) (actual time=0.060..0.060 rows=0 loops=1)
   Output: ticket_no, book_ref, passenger_id, passenger_name, contact_data
   Recheck Cond: (tickets.passenger_name = 'ARTEM TIHON'::text)
   ->  Bitmap Index Scan on tickets_passenger_name_idx  (cost=0.00..3.62 rows=259 width=0) (actual time=0.057..0.057 rows=0 loops=1)
         Index Cond: (tickets.passenger_name = 'ARTEM TIHON'::text)
 Planning Time: 0.213 ms
 Execution Time: 0.075 ms
```

Теперь при выполнении запроса используется индекс "Bitmap Index Scan on tickets_passenger_name_idx". И скорость выполнения запроса значительно возросла: 0.075 ms против 78.226 ms


## 4. Сравнить применение различных типов индексов

Попробуем применить различные типы индексов для поля "contact_data" с типом jsonb.

Создадим индекс и проверим его работу. Будем искать записи в которых присутсвует свойство "email".

``` sql
create index on tickets (contact_data);
```

```
demo=# \d tickets;
                                   Таблица "bookings.tickets"
    Столбец     |          Тип          | Правило сортировки | Допустимость NULL | По умолчанию
----------------+-----------------------+--------------------+-------------------+--------------
 ticket_no      | character(13)         |                    | not null          |
 book_ref       | character(6)          |                    | not null          |
 passenger_id   | character varying(20) |                    | not null          |
 passenger_name | text                  |                    | not null          |
 contact_data   | jsonb                 |                    |                   |
Индексы:
    "tickets_pkey" PRIMARY KEY, btree (ticket_no)
    "tickets_contact_data_idx" btree (contact_data)
    "tickets_passenger_name_idx" btree (passenger_name)
Ограничения внешнего ключа:
    "tickets_book_ref_fkey" FOREIGN KEY (book_ref) REFERENCES bookings(book_ref)
Ссылки извне:
    TABLE "ticket_flights" CONSTRAINT "ticket_flights_ticket_no_fkey" FOREIGN KEY (ticket_no) REFERENCES tickets(ticket_no)
```

Появился индекс "tickets_contact_data_idx" с типом "btree". Этот тип считается не самым подходящим для типа данных "jsonb".

``` sql
analyze tickets;
vacuum tickets;
explain (costs, verbose, analyze) SELECT * FROM bookings.tickets WHERE contact_data ? 'email';
```

```
                                                          QUERY PLAN
------------------------------------------------------------------------------------------------------------------------------
 Seq Scan on bookings.tickets  (cost=0.00..86289.94 rows=1609088 width=104) (actual time=0.007..256.318 rows=1601080 loops=1)
   Output: ticket_no, book_ref, passenger_id, passenger_name, contact_data
   Filter: (tickets.contact_data ? 'email'::text)
   Rows Removed by Filter: 1348777
 Planning Time: 0.942 ms
 Execution Time: 289.140 ms
```

Видим, что при выполнения запроса, индекс вообще не использовался.

Попробуем изменить тип индекса на "gin", более подходящий к данному типу данных.

``` sql
DROP INDEX tickets_contact_data_idx;
create index on tickets USING GIN (contact_data);
```

```
demo=# \d tickets;
                                   Таблица "bookings.tickets"
    Столбец     |          Тип          | Правило сортировки | Допустимость NULL | По умолчанию
----------------+-----------------------+--------------------+-------------------+--------------
 ticket_no      | character(13)         |                    | not null          |
 book_ref       | character(6)          |                    | not null          |
 passenger_id   | character varying(20) |                    | not null          |
 passenger_name | text                  |                    | not null          |
 contact_data   | jsonb                 |                    |                   |
Индексы:
    "tickets_pkey" PRIMARY KEY, btree (ticket_no)
    "tickets_contact_data_idx" gin (contact_data)
    "tickets_passenger_name_idx" btree (passenger_name)
Ограничения внешнего ключа:
    "tickets_book_ref_fkey" FOREIGN KEY (book_ref) REFERENCES bookings(book_ref)
Ссылки извне:
    TABLE "ticket_flights" CONSTRAINT "ticket_flights_ticket_no_fkey" FOREIGN KEY (ticket_no) REFERENCES tickets(ticket_no)
```

Теперь у индекса "tickets_contact_data_idx" тип "gin".

Проверим, как теперь выполняется запрос.

``` sql
analyze tickets;
vacuum tickets;
explain (costs, verbose, analyze) SELECT * FROM bookings.tickets WHERE contact_data ? 'email';
```

```
                                                                    QUERY PLAN
--------------------------------------------------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on bookings.tickets  (cost=9265.01..78789.57 rows=1608765 width=104) (actual time=71.002..303.128 rows=1601080 loops=1)
   Output: ticket_no, book_ref, passenger_id, passenger_name, contact_data
   Recheck Cond: (tickets.contact_data ? 'email'::text)
   Heap Blocks: exact=49415
   ->  Bitmap Index Scan on tickets_contact_data_idx  (cost=0.00..8862.82 rows=1608765 width=0) (actual time=66.288..66.288 rows=1601080 loops=1)
         Index Cond: (tickets.contact_data ? 'email'::text)
 Planning Time: 0.093 ms
 Execution Time: 335.812 ms
```

Теперь индекс используется. Однако, стоит отметить, что хоть поиск по индексу и быстрее, чем последовательное сканирование - общее время выполнения запроса увеличилось.

Можно сделать вывод: что использование индексов не всегда приводит к повышению производительности и требует предварительной проверки на реальных запросов с реальными данными.
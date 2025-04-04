**Лабораторная работа №3: Расширенные возможности и оптимизация 
PostgreSQL на Debian**

Получить опыт в использовании продвинутых функций PostgreSQL (индексы, 
планы запросов, функции и триггеры, базовые приёмы оптимизации). 

# 1. Оптимизация конфигурации PostgreSQL 
### Найти в postgresql.conf параметры, влияющие на производительность: shared_buffers, work_mem, maintenance_work_mem, effective_cache_size. Кратко описать их назначение. Установить значения с учётом объёма оперативной памяти виртуальной машины. Объяснить выбор установленных значений. Перезапустить PostgreSQL (systemctl restart postgresql). Проверить установленные параметры командой SHOW shared_buffers;, SHOW work_mem; и т.д. 

Основные параметры производительности
- shared_buffers - определяет объем памяти, выделяемый PostgreSQL для кэширования данных. Это основной буфер для работы с данными. Рекомендуется: 25–40% ОЗУ.
- work_mem - объем памяти, выделяемый для операций сортировки и хеш-таблиц в рамках одного запроса. Большие значения позволяют выполнять сложные сортировки в памяти. 4MB — дефолт.
- maintenance_work_mem - объем памяти для операций обслуживания БД (VACUUM, CREATE INDEX и др.). от 64MB до 512MB
- effective_cache_size - оценка объема памяти, доступного для кэширования ОС и PostgreSQL. Не выделяет память, но помогает планировщику запросов. Обычно ставится: 60–75% ОЗУ.

           !!!! sudo nano /etc/postgresql/15/main/postgresql.conf !!!!

Для того, чтобы узнать количество оперативной памяти:

          free -h

Изменили:
  - shared_buffers = 512MB
  - work_mem = 16MB
  - maintenance_work_mem = 128MB
  - effective_cache_size = 1.2GB

    sudo systemctl restart postgresql

Для проверки внесенных изменений:

          SHOW shared_buffers;
          SHOW work_mem;
          SHOW maintenance_work_mem;
          SHOW effective_cache_size;

# 2. Создание и анализ индексов 
### Использовать таблицы из предыдущих лабораторных работ или при необходимости добавить новые. При необходимости наполнить их большим количеством строк для тестирования, смотрим generate_series Создать индексы (по одному или нескольким столбцам) на подходящих полях. Сохранить команды CREATE INDEX. Выполнить запросы EXPLAIN и EXPLAIN ANALYZE до и после создания индексов.Сравнить планы (Seq Scan, Index Scan) и время выполнения запросов.

Создали и заполнили данными новую таблицу:

          CREATE TABLE orders (
              id serial PRIMARY KEY,
              order_name text NOT NULL,
              order_date timestamp NOT NULL DEFAULT now(),
              amount integer NOT NULL
          );


          INSERT INTO orders (order_name, order_date, amount)
          SELECT 
              'Тестовый заказ №' || i AS order_name,
              NOW() - (i * INTERVAL '1 day') AS order_date,
              (RANDOM() * 10000)::INT AS amount
          FROM generate_series(1, 50 000) AS i;

          
EXPLAIN (предварительный анализ) и EXPLAIN ANALYZE (фактический анализ)

EXPLAIN SELECT * FROM orders WHERE order_name = 'Тестовый заказ №49999';

EXPLAIN ANALYZE SELECT * FROM orders WHERE order_name = 'Тестовый заказ №49999';



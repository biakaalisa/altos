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

            sudo nano /etc/postgresql/15/main/postgresql.conf

Для того, чтобы узнать количество оперативной памяти:

          free -h

![image](https://github.com/user-attachments/assets/e20623d3-6e59-4aec-a0ae-2211454e60b0)
![image](https://github.com/user-attachments/assets/8495e6e8-8a1f-478f-832c-1c6b57684b82)
![image](https://github.com/user-attachments/assets/727d576b-bc16-47c1-b8cc-f07f5929dfc1)


Изменили:
  - shared_buffers = 512MB
  - work_mem = 16MB
  - maintenance_work_mem = 128MB
  - effective_cache_size = 4GB

    sudo systemctl restart postgresql

Для проверки внесенных изменений:

          SHOW shared_buffers;
          SHOW work_mem;
          SHOW maintenance_work_mem;
          SHOW effective_cache_size;

![image](https://github.com/user-attachments/assets/059921bb-3b53-439d-a5d7-a30dc461f1c2)
![image](https://github.com/user-attachments/assets/62e68a8d-2f73-4b9a-8242-a739369a0ebf)

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
          
![image](https://github.com/user-attachments/assets/d08021bd-b09c-40db-96bf-436966c8ece8)
![image](https://github.com/user-attachments/assets/dbc56554-b571-42f9-81ed-d46e87d0cf72)

          
EXPLAIN (предварительный анализ) и EXPLAIN ANALYZE (фактический анализ)

           EXPLAIN SELECT * FROM orders WHERE order_name = 'Тестовый заказ №49999';
           EXPLAIN ANALYZE SELECT * FROM orders WHERE amount = 2512;

![image](https://github.com/user-attachments/assets/ff377c6a-4666-4264-abd3-0101c2220b57)

           EXPLAIN ANALYZE SELECT * FROM orders WHERE order_name = 'Тестовый заказ №49999';
           EXPLAIN ANALYZE SELECT * FROM orders WHERE amount = 2512;
           
![image](https://github.com/user-attachments/assets/f3a4cab1-9210-4c26-ab50-f1798ebfbfdf)

Индекс — это специальная структура данных, которая позволяет ускорить поиск строк в таблице по определённым столбцам.
Когда происходит запрос вроде:

```SELECT * FROM users WHERE email = 'user@example.com';```

PostgreSQL должен прочитать все строки в таблице, чтобы найти подходящую. Это называется Seq Scan (последовательное сканирование таблицы).

Если создать индекс на поле email, то PostgreSQL быстро найдёт нужную строку, не читая всю таблицу. Такой способ называется Index Scan.
![image](https://github.com/user-attachments/assets/cbabd8f9-1cbe-4801-b78b-3c0340e9460f)

            CREATE INDEX idx_orders_order_name ON orders(order_name);
            CREATE INDEX idx_orders_amount ON orders(amount);
            
![image](https://github.com/user-attachments/assets/6abc5dad-ee87-4b6b-8ee8-db9c859ee1d8)

Проверка и сравнение результатов скорости выполнения запросов с использованием индексов: 

![image](https://github.com/user-attachments/assets/4d0ba391-02d8-4b4e-a476-66aa5c545645)
![image](https://github.com/user-attachments/assets/df843e51-5e57-49e8-9ccb-1935bd656c4b)
![image](https://github.com/user-attachments/assets/7d317730-c539-401c-9b3d-b744f3501e3a)


# 3. Хранимые функции  
### Создать функцию на pgSQL, которая проверяет переданное значение и в зависимости от результата либо вставляет новую запись в таблицу, либо возвращает сообщение об ошибке (например, «Запись добавлена» или «Ошибка отрицательное значение»). Выполнить вызов функции в psql и проверить результат вставки. 

Создадим хранимую функцию которая будет:
- принимать параметры (например, order_name, order_date, amount);
- проверять, что amount ≥ 0;
- если значение корректное — вставлять запись в таблицу orders;
- иначе — возвращать сообщение об ошибке.

                        CREATE OR REPLACE FUNCTION insert_order_check(
                            p_order_name TEXT,
                            p_order_date TIMESTAMP,
                            p_amount INTEGER
                        )
                        RETURNS TEXT
                        LANGUAGE plpgsql
                        AS $$
                        BEGIN
                            IF p_amount < 0 THEN
                                RETURN 'Ошибка: отрицательное значение';
                            ELSE
                                INSERT INTO orders (order_name, order_date, amount)
                                VALUES (p_order_name, p_order_date, p_amount);
                                
                                RETURN 'Запись добавлена';
                            END IF;
                        END;
                        $$;

![image](https://github.com/user-attachments/assets/0e3a0708-bf3b-4adf-8f72-6137b922072d)


            SELECT insert_order_check('Тестовый заказ через функцию', NOW()::timestamp, 1500);
            
            SELECT insert_order_check('Неверный заказ', NOW()::timestamp, -1500);

![image](https://github.com/user-attachments/assets/0221d42c-fe24-42bd-8a67-b8b00d4f62f7)

# 4. Триггеры 
### Создать триггер (BEFORE или AFTER INSERT/UPDATE), который проверяет бизнесправила (например, недопустимость отрицательной цены).При нарушении условий вызвать RAISE EXCEPTION. Вставить данные для проверки срабатывания триггера. 

Создадим триггер который будет:
- срабатывать до вставки или обновления (BEFORE INSERT OR UPDATE);
- проверять бизнес-правило: значение amount не может быть отрицательным;
- если условие нарушено — вызывать RAISE EXCEPTION;
- иначе — разрешать операцию.

Создаем триггурную функцию:

            CREATE OR REPLACE FUNCTION check_amount_before_insert_update()
            RETURNS TRIGGER AS $$
            BEGIN
                IF NEW.amount < 0 THEN
                    RAISE EXCEPTION 'Ошибка: значение amount не может быть отрицательным.';
                END IF;
                RETURN NEW;
            END;
            $$ LANGUAGE plpgsql;

Привязываем созданный триггер к таблице:

            CREATE TRIGGER trg_check_amount
            BEFORE INSERT OR UPDATE ON orders
            FOR EACH ROW
            EXECUTE FUNCTION check_amount_before_insert_update();
            
![image](https://github.com/user-attachments/assets/ababbd54-acdf-4109-a026-9b54adeb1189)

Проверяем:

            INSERT INTO orders (order_name, order_date, amount) VALUES ('Нормальный заказ', NOW(), 1500);
            
            INSERT INTO orders (order_name, order_date, amount) VALUES ('Плохой заказ', NOW(), -1500);

![image](https://github.com/user-attachments/assets/0e448171-0758-4fac-9049-6eb17e44e84b)

# 5. Автоматическая очистка и статистика (VACUUM, ANALYZE) 
### Изучить параметры autovacuum (например, autovacuum_naptime, autovacuum_vacuum_scale_factor).Убедиться, что autovacuum включён (значение on).Выполнить VACUUM ANALYZE для одной или нескольких таблиц.Объяснить назначение VACUUM и ANALYZE (очистка «мёртвых» строк, актуализация статистики). Изучить представления pg_stat_user_tables, pg_stat_all_indexes и другие.Найти информацию о количестве выполненных autovacuum и manual vacuum.

VACUUM:
- Очищает "мёртвые" строки (помеченные как удалённые)
- Освобождает место для повторного использования
- Обновляет карту видимости (visibility map)

ANALYZE:
- Собирает статистику о распределении данных в таблицах
- Используется планировщиком запросов для оптимизации выполнения

Autovacuum - это фоновый процесс PostgreSQL, который автоматически выполняет операции VACUUM и ANALYZE. Вот ключевые параметры:
- autovacuum - главный переключатель (on/off)
- autovacuum_naptime - пауза между проверками (по умолчанию 1 мин)
- autovacuum_vacuum_scale_factor - доля изменённых строк для запуска VACUUM (по умолчанию 0.2 = 20%)
- autovacuum_analyze_scale_factor - аналогично для ANALYZE (по умолчанию 0.1 = 10%)
- autovacuum_vacuum_threshold - минимальное количество изменений для запуска (по умолчанию 50)
- autovacuum_analyze_threshold - аналогично для ANALYZE (по умолчанию 50)

Для проверки статуса Autovacuum необходимо исользовать:

            SHOW autovacuum;

![image](https://github.com/user-attachments/assets/464d5038-ff63-4573-b87c-eaa6ee1873a9)

Для того, тобы вручную запустить VACUUM и ANALYZE:

            VACUUM ANALYZE orders;            // для таблицы orders
            VACUUM ANALYZE;                   // для всей БД 
            
![image](https://github.com/user-attachments/assets/6b9fa492-71ee-4793-921e-92cd48672960)

pg_stat_user_tables - статистика по таблицам:

            SELECT relname, n_dead_tup, last_vacuum, last_autovacuum 
            FROM pg_stat_user_tables;
            
![image](https://github.com/user-attachments/assets/ef9125ec-3c35-438d-bd29-9baad71169d8)

pg_stat_all_indexes - статистика по индексам:

            SELECT indexrelname, idx_scan, idx_tup_read, idx_tup_fetch 
            FROM pg_stat_all_indexes;

![image](https://github.com/user-attachments/assets/10b24734-fa1f-47fb-80e5-68a4da06bc2c)

- indexrelname - имя индекса
- idx_scan - количество сканирований этого индекса
- idx_tup_read - количество прочитанных через индекс записей
- idx_tup_fetch - количество записей, фактически извлечённых из таблицы по этому индексу

Статистика по vacuum:

            SELECT relname, 
                   vacuum_count, 
                   autovacuum_count,
                   analyze_count,
                   autoanalyze_count
            FROM pg_stat_user_tables;
            
![image](https://github.com/user-attachments/assets/ed772201-c121-4b0e-bc5e-819e8381cfcf)

- relname - имя таблицы
- vacuum_count - количество ручных операций VACUUM
- autovacuum_count - количество автоматических операций VACUUM
- analyze_count - количество ручных операций ANALYZE
- autoanalyze_count - количество автоматических операций ANALYZE

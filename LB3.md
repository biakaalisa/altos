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

            sudo nano /etc/postgresql/15/main/postgresql.conf

Для того, чтобы узнать количество оперативной памяти:

          free -h

![image](https://github.com/user-attachments/assets/adcb2a4d-efa8-4ffd-8c1e-302d561b0138)
![image](https://github.com/user-attachments/assets/b94ce53d-a03c-4c33-bb46-fc8818d37805)


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

![image](https://github.com/user-attachments/assets/ebdfa8d8-97bf-41ce-bb92-d17febaa66c4)


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
          
![image](https://github.com/user-attachments/assets/6bca02af-9d3e-4b62-b22d-de8339e29a0b)
![image](https://github.com/user-attachments/assets/b3a571b2-69cf-4fa0-b37d-1d4427ee72b9)

          
EXPLAIN (предварительный анализ) и EXPLAIN ANALYZE (фактический анализ)

           EXPLAIN SELECT * FROM orders WHERE order_name = 'Тестовый заказ №49999';
           EXPLAIN ANALYZE SELECT * FROM orders WHERE amount = 2512;

![image](https://github.com/user-attachments/assets/b0e6266d-175e-41b3-9e28-7604430ef09c)
![image](https://github.com/user-attachments/assets/6a4a08c0-da6a-414d-bf9a-982e1665f8ec)


           EXPLAIN ANALYZE SELECT * FROM orders WHERE order_name = 'Тестовый заказ №49999';
           EXPLAIN ANALYZE SELECT * FROM orders WHERE amount = 2512;
           
![image](https://github.com/user-attachments/assets/6eb33d5f-535c-4b30-826f-15764c696623)
![image](https://github.com/user-attachments/assets/8ea31103-1325-4729-a50e-805a37b30b59)

Индекс — это специальная структура данных, которая позволяет ускорить поиск строк в таблице по определённым столбцам.
Когда происходит запрос вроде:

```SELECT * FROM users WHERE email = 'user@example.com';```

PostgreSQL должен прочитать все строки в таблице, чтобы найти подходящую. Это называется Seq Scan (последовательное сканирование таблицы).

Если создать индекс на поле email, то PostgreSQL быстро найдёт нужную строку, не читая всю таблицу. Такой способ называется Index Scan.
![image](https://github.com/user-attachments/assets/cbabd8f9-1cbe-4801-b78b-3c0340e9460f)

            CREATE INDEX idx_orders_order_name ON orders(order_name);
            CREATE INDEX idx_orders_amount ON orders(amount);
            
![image](https://github.com/user-attachments/assets/d82d6d98-a11b-4be5-b83a-91e648fa8189)

Проверка и сравнение результатов скорости выполнения запросов с использованием индексов: 

![image](https://github.com/user-attachments/assets/1e439a3e-8287-4bbf-b0a2-7e86a11b41be)
![image](https://github.com/user-attachments/assets/e65a84f3-3894-4275-8a09-d85be834e213)
![image](https://github.com/user-attachments/assets/9e9ef9ba-3a8e-41b9-b578-a40d280f60c7)


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

![image](https://github.com/user-attachments/assets/9df0aae2-0b6c-4f3c-8316-886c0711eda3)


            SELECT insert_order_check('Тестовый заказ через функцию', NOW()::timestamp, 1500);
            
            SELECT insert_order_check('Неверный заказ', NOW()::timestamp, -1500);

![image](https://github.com/user-attachments/assets/47c9ca04-3e48-4cbf-a770-b58b5f054b9b)

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

![image](https://github.com/user-attachments/assets/3c9227b8-c610-4513-8246-0e19d3cde2c0)


Привязываем созданный триггер к таблице:

            CREATE TRIGGER trg_check_amount
            BEFORE INSERT OR UPDATE ON orders
            FOR EACH ROW
            EXECUTE FUNCTION check_amount_before_insert_update();

![image](https://github.com/user-attachments/assets/f6a567ce-733f-4a37-bf6d-60fe154b5b4e)

Проверяем:

            INSERT INTO orders (order_name, order_date, amount) VALUES ('Нормальный заказ', NOW(), 1500);
            
            INSERT INTO orders (order_name, order_date, amount) VALUES ('Плохой заказ', NOW(), -1500);

![image](https://github.com/user-attachments/assets/c06e6b25-3360-4b3f-a0db-743589e481cb)

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

![image](https://github.com/user-attachments/assets/81f5a4d3-c63f-4ba3-8b6b-328ab6a0ed4a)

Для того, тобы вручную запустить VACUUM и ANALYZE:

            VACUUM ANALYZE orders;            // для таблицы orders
            VACUUM ANALYZE;                   // для всей БД 
            
![image](https://github.com/user-attachments/assets/80f6ddb9-3d6a-4065-a50d-58b3ef9b0e4c)

pg_stat_user_tables - статистика по таблицам:

            SELECT relname, n_dead_tup, last_vacuum, last_autovacuum 
            FROM pg_stat_user_tables;
            
![image](https://github.com/user-attachments/assets/a44cff10-e3e6-4b5d-b934-1aa1e504d01c)

pg_stat_all_indexes - статистика по индексам:

            SELECT indexrelname, idx_scan, idx_tup_read, idx_tup_fetch 
            FROM pg_stat_all_indexes;

![image](https://github.com/user-attachments/assets/0f418917-5f37-4c9a-b0fc-ddfa6d3c2e61)

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
            
![image](https://github.com/user-attachments/assets/9fcc9bb7-b303-4e8a-a5ad-974ca2cb91e6)

- relname - имя таблицы
- vacuum_count - количество ручных операций VACUUM
- autovacuum_count - количество автоматических операций VACUUM
- analyze_count - количество ручных операций ANALYZE
- autoanalyze_count - количество автоматических операций ANALYZE

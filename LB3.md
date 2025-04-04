![image](https://github.com/user-attachments/assets/bfc2130a-fda5-43e6-8f5e-a27b1a1512d0)![image](https://github.com/user-attachments/assets/aaeb719e-0f95-4e81-a137-dc616f746181)**Лабораторная работа №3: Расширенные возможности и оптимизация 
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


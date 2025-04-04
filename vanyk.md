**Лабораторная работа №2: резервное копирование, восстановление и 
мониторинг в Debian и PostgreSQL** 

Изучить способы резервного копирования баз данных PostgreSQL и восстановления 
их в среде Debian. Освоить базовые инструменты мониторинга системы и сервиса 
PostgreSQL. 

# 1. Утилиты резервного копирования 
### Изучить и сравнить pg_dump и pg_basebackup. Описать сценарии применения каждой утилиты.

pg_dump: Логический бэкап (SQL), отдельных БД/таблиц, медленное восстановление, межверсионная совместимость. Утилита для создания логических дампов отдельных баз данных PostgreSQL. Она извлекает данные в виде SQL-команд, которые можно выполнить для восстановления базы.

pg_basebackup: Физический бэкап (бинарный), всего кластера, быстрое восстановление, только для той же версии. Утилита для создания физических бэкапов всего кластера PostgreSQL (всех баз данных). Она копирует бинарные файлы данных с сервера, создавая полную копию на момент бэкапа.

pg_dump:
 - Бэкап отдельных БД/таблиц
 - Перенос между версиями PostgreSQL
 - Экспорт структуры без данных
 - Малые/средние БД

pg_basebackup:
 - Полный бэкап кластера
 - Настройка репликации
 - Быстрое восстановление (Disaster Recovery)
 - Развертывание идентичных сред

# 2. Создание резервной копии 
### Выполнить полное резервное копирование (dump) базы данных из лаб. №1. Использовать параметры (например, -Fc, -Ft, обязательно посмотреть другие параметры!), объяснить разницу. 

![image](https://github.com/user-attachments/assets/ea0cb578-4423-4589-bde4-8dbf851d0ba4)

Для создания резервной копии использовать команду:

         pg_dump -U postgres -d postgres -Fp -f backup.sql

![image](https://github.com/user-attachments/assets/dd93e1a1-d699-455c-adb2-99d66959d4c4)
![image](https://github.com/user-attachments/assets/a2f1a12d-c7d8-4f51-a770-d436c1eb1a92)

# 3. Частичное (выборочное) резервное копирование 
### Сделать дамп только определённой схемы (например, test_schema, созданной в ЛР №1). Сделать дамп только определённых таблиц из схемы public. Объяснить, в чём отличие от резервного копирования всей базы.    

Для создания резервной копии схемы использовать команду: 

         pg_dump -U postgres -d postgres -n test_schema -Fc -f test_schema.dump

![image](https://github.com/user-attachments/assets/c4cf74cc-edcf-4b45-94a3-66ba97588787)

Для создания резервной копии таблицы использовать команду: 

        pg_dump -U postgres -d postgres -t test_schema.t_t -Fc -f public-test.dump

![image](https://github.com/user-attachments/assets/26b27767-2f04-492e-9c9c-20d4117c358d)


# 4. Восстановление из резервной копии 
### Восстановить базу из резервного файла с помощью pg_restore или утилиты psql. Продемонстрировать процесс и результаты. 

При помощи команды ```CREATE DATABASE``` создали пустую БД

![image](https://github.com/user-attachments/assets/f170f9e0-5b7f-4a11-b38d-332ba18817a4)

Восстановили в нее скопированную ранее БД командой:

          psql -U postgres -d restored_db -f backup.sql

![image](https://github.com/user-attachments/assets/fd950f49-877d-41a3-bb08-7fdaf8b9aacb)
![image](https://github.com/user-attachments/assets/71c0d3f4-29b8-494b-8a96-1ba76cc0feb5)

# 5. Автоматизация бэкапов с помощью cron 
### Настроить планировщик cron на Debian, чтобы ежедневно создавать резервные копии. Указать, куда складываются дампы, и как выполняется ротация. Обязательно понимать, что такое ротация!     

Отредактировали системный crontab файл 

        sudo crontab -e

Добавили строку:

              0 3 * * * /usr/bin/pg_dump -U postgres postgres > /var/lib/postgresql/db_$(date +\%Y\%m\%d).sql && find /var/lib/postgresql -name "*.sql" -mtime +7 -delete


![image](https://github.com/user-attachments/assets/779ed996-7149-49f9-974a-667130797784)
![image](https://github.com/user-attachments/assets/2ec7b3c9-869f-4d2d-afe9-f36ceeb67dc1)
![image](https://github.com/user-attachments/assets/8e66bc36-c6c6-48d0-b7a5-0295c33008bd)

# 6. Мониторинг состояния системы 
### Использовать стандартные инструменты Debian (например, top, htop, iotop) для мониторинга потребления ресурсов PostgreSQL (CPU, RAM, IO). Уметь объяснить все значения выводимых показателей. 

top — это стандартная утилита, которая показывает:

- загрузку процессора (CPU)
- использование памяти (RAM, Swap)
- количество процессов
- список всех процессов с их PID, CPU, памятью и состоянием

      top -u postgres 

![image](https://github.com/user-attachments/assets/a5e39d9b-f1ce-40b4-ac32-f2d2524dfd25)

htop — это улучшенная, более удобная версия top, которая:

- Поддерживает цветной интерфейс
- Позволяет использовать клавиши для управления (стрелки, F1–F10)
- Показывает графики использования CPU и памяти
- Поддерживает поиск и фильтрацию процессов
- Позволяет убивать процессы прямо в интерфейсе

Для использования htop, предварительно его установили ```sudo apt install htop```

    htop

![image](https://github.com/user-attachments/assets/fec86c9b-5f27-4948-83e6-811dbd24cc35)
![image](https://github.com/user-attachments/assets/af20f3bf-4234-4aa4-a28e-3345ac81ec02)

# 7. Мониторинг PostgreSQL 
### Изучить встроенные представления статистики в PostgreSQL (например, pg_stat_activity, pg_stat_database). Показать, как смотреть активные процессы, долгие запросы и т.д. Показать как можно принудительно завершить процесс зависший или слишком тяжелый запрос (необходимо иметь роль суперюзера) 

pg_stat_activity - Это системное представление (view), которое показывает все текущие подключения к PostgreSQL: кто подключён, откуда, что делает, какой запрос выполняется, и сколько он длится.
- Имя пользователя (usename)
- Название базы (datname)
- IP-адрес клиента (client_addr)
- PID процесса (pid)
- Приложение (application_name)

Для получения всей этой информации использовать:

     SELECT pid, usename, datname, client_addr, state, query_start, query
     FROM pg_stat_activity
     WHERE state != 'idle';
     
![image](https://github.com/user-attachments/assets/ab4d399b-c667-44ba-8596-94bb65fb738a)
![image](https://github.com/user-attachments/assets/6ccacc52-b629-48d4-8270-ee1748670dbe)

Для вывода активных процессов использовать:

      SELECT *
      FROM pg_stat_activity
      WHERE state = 'active';

![image](https://github.com/user-attachments/assets/ba19fd54-db76-49e7-8555-a09f8c2a340b)

Для просмотра долговыполняющихся запросов:

     SELECT pid, usename
     FROM pg_stat_activity
     WHERE state = 'active' AND query_start < now() - interval '5 minutes';

![image](https://github.com/user-attachments/assets/e36841d4-2e22-4488-9c10-380e7f3c5db8)


Для "мягкого удаления" долговыполняющегося запросов использовать:

        SELECT pg_cancel_backend(PID);

![image](https://github.com/user-attachments/assets/28de7cf1-1f97-47d7-8cba-a778c05140ab)

Для прсмотра статистики по БД использовать:

        SELECT * FROM pg_stat_database;

![image](https://github.com/user-attachments/assets/18d0deed-ad23-41bb-8872-b5e448f62093)

# 8. Логирование и анализ логов 
### Найти логи PostgreSQL и системные логи Debian (директория /var/log/, файлы syslog, daemon.log). Определить, какие события логгирует СУБД, а какие – ОС. 

В системе не установлен rsyslog, который отвечает за запись логов в файлы вроде syslog, daemon.log, и т.д.

Вместо этого вся система логгирует в журнал systemd через journald, который хранит логи в бинарной базе (/run/log/journal/ или /var/log/journal/), а не в текстовых файлах.
![image](https://github.com/user-attachments/assets/33054732-bafe-42fe-87db-f5959919ef15)
![image](https://github.com/user-attachments/assets/66abe49e-c960-468d-ab1c-31c6522950f1)

Логи postgres просмотреть в файле:
![image](https://github.com/user-attachments/assets/6c5242c6-0c13-4615-b3ad-e3d32263098e)

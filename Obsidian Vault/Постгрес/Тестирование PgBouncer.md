 Тестирование PgBouncer

Чтобы проверить работу PgBouncer воспользуемся утилитой **pgbench**. Она устанавливается вместе с сервером БД и используется для проверки производительности PostgreSQL. Сначала нужно подготовить БД к бенчмарку, для этого запустим команду:

      `pgbench -p 5432 -i -U selectel_user -h localhost selectel_db`

Теперь запустим по очереди два теста: на стандартном порту 5432, и на порту 6432 с пулером. Сымитируем запросы 500 клиентов в 16 потоков, тест будет продолжаться 60 секунд.

      `pgbench -p 5432 -c 500 -j 16 -T 60 -U selectel_user -h localhost selectel_db`

Пока бенчмарк выполняется, выполним команду в соседнем окне:

      `ps aux | grep selectel_user | cat -b`

Увидим, что в ОС одновременно создано 500 физических подключений к БД:

      `<...> 499  postgres    4557  0.1  1.4 245172 28944 ?        Ss   16:45   0:00 postgres: 14/main: selectel_user selectel_db ::1(34508) UPDATE waiting 500  postgres    4558  0.1  1.3 245172 28200 ?        Ss   16:45   0:00 postgres: 14/main: selectel_user selectel_db ::1(34514) UPDATE waiting`

Дождемся результатов теста:

      `number of transactions actually processed: 11716 latency average = 2219.578 ms tps = 225.268041 (without initial connection time)`

Теперь выполним этот же бенчмарк на порту 6432, где запущен PgBouncer:

      `pgbench -p 6432 -c 500 -j 16 -T 60 -U selectel_user -h localhost selectel_db`

Пока выполняется тест, снова проверим количество физических коннектов к БД:

      `ps aux | grep selectel_user | cat -b`

Результат:

      `<...> 19  postgres    1060  1.2  1.5 245044 31716 ?        Ss   16:51   0:00 postgres: 14/main: selectel_user selectel_db ::1(59072) UPDATE waiting 20  postgres    1061  1.4  1.5 245052 31676 ?        Ss   16:51   0:00 postgres: 14/main: selectel_user selectel_db ::1(59086) UPDATE waiting`

Видим, что открыто всего 20 соединений – PgBouncer открыл в 25 раз меньше физических подключений. Дождемся результатов тестов:

      `number of transactions actually processed: 19485 latency average = 1573.937 ms tps = 317.674686 (without initial connection time)`
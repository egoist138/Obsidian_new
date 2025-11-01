## 

Установка и настройка pg_profile

1. Скачиваем последний релиз: [https://github.com/zubkov-andrei/pg_profile/releases](https://github.com/zubkov-andrei/pg_profile/releases)
    
2. Выполняем действия по установке на странице документации. Эти действия надо выполнять на сервере, который собирает статистику со всех серверов.
    

```bash
sudo tar xzf pg_profile--4.6.tar.gz --directory $(pg_config --sharedir)/extension
```

Добавляем расширение `pg_stat_statements` в `postgresql.conf`:

```bash
shared_preload_libraries = 'pg_stat_statements'
```

И устанавливаем все наши расширения для работы:

```sql
CREATE SCHEMA profile;
CREATE EXTENSION dblink;
CREATE EXTENSION pg_stat_statements;
CREATE EXTENSION pg_profile SCHEMA profile;
ALTER EXTENSION pg_profile UPDATE;
```

3. На сервере с которого необходимо собирать статистику надо создать расширение `pg_stat_statements`. Для этого надо добавить его в файл `postgresql.conf`
    

```bash
shared_preload_libraries = 'pg_stat_statements'
```

После чего в `psql` ввести запрос:

```sql
CREATE EXTENSION pg_stat_statements;
```

4. На этом же сервере выполняем следующие запросы:
    

```sql
CREATE USER profile_collector with password 'collector_pwd';
GRANT pg_read_all_stats TO profile_collector;
GRANT EXECUTE ON FUNCTION pg_stat_statements_reset TO profile_collector;
```

5. На сервере `pg_profile` создаем наши сервера, с которых будет собираться статистика:
    

```sql
select profile.create_server('mp-postgree-prod1', 'dbname=postgres port=5432 host=10.122.0.5 user=profile_collector password=V1PMVq4HBI3HeRXnQivq');
select profile.create_server('mp-postgree-prod2', 'dbname=postgres port=5432 host=10.12.16.67 user=profile_collector password=password');
select profile.create_server('mp-postgree-prod3', 'dbname=postgres port=5432 host=10.12.16.101 user=profile_collector password=password');
```

6. Установим количество хранимых семплов для каждого сервера. Делается это на сервере `pg_profile`:
    

```sql
select profile.set_server_max_sample_age('mp-postgree-prod1', 1488);
```

Обычно семплы снимают раз в полчаса, то есть за 1 день будет снято 48 семплов. Таким образом 500 семплов будут хранить информацию о 10 днях. Чтобы сделать хранение за 31 день, надо установить значение 1488.  
7. На сервере `pg_profile` для пользователя `postgres` установим задачу в `cron`:

```bash
*/30 * * * * psql postgres -c 'select * from profile.snapshot()'
10 0 * * * /var/lib/postgresql/report.sh
```

Первая строчка будет каждый полчаса делать снимки статистики.  
Вторая будет создавать отчет. Для этого надо создать файл `report.sh` со следующим содержимым:

```bash
#!/bin/env bash

REPDIR=/var/lib/postgresql/profile_report
SERVERS=$(psql postgres -Aqtc \
    "select server_name from profile.show_servers() where enabled and server_name not in ('local', 'mospgsql5401vm.digital.kfc.ru') order by server_name"
)

for server in ${SERVERS}; do
    mkdir -pv ${REPDIR}/${server}
    psql postgres -Aqtc "select profile.get_report('${server}', tstzrange(now() - interval '1 day',now()))" -o ${REPDIR}/${server}/$(date --date="yesterday" +%F).html
done
```

Отчет будет создаваться каждый день в соответствующих папках, которые видны в скрипте.

## 

Снятие отчета

Если же хотим просто создать отчет, то можно выполнить следующую команду:

```bash
psql postgres -Aqtc "select profile.get_report('mp-postgree-prod1', tstzrange(now() - interval '1 day',now()))" -o /var/lib/postgresql/profile_report/$(date --date="yesterday" +%F).html
```

Потом скачать полученный файл себе с помощью `scp` и проанализировать его.

## 

Анализ

В файле отчета есть много информации.  
Для нас интересны запросы, поэтому мы идем в раздел `SQL query statistics`.  
Там нам представлены следующие таблицы:

- `Top SQL by execution time` - таблица показывает запросы по времени выполнения. Есть общее время в секундах, а так же минимальное, максимальное и среднее в мс. Также можно увидеть количество выполненных запросов. На что нам тут смотреть? На общее время, количество выполнений, среднее и максимальное время. Из этого уже можно будет сделать некоторые выводы.
    
- `Top SQL by executions` - показывает топ запросов по количеству выполнений. Так же есть информация о времени выполнения данных запросов.
    
- `Top SQL by I/O wait time` - показывает информацию и времени чтения и записи блоков на диск и с диска.
    
- `Top SQL by shared blocks fetched` - одна из самых полезных таблиц. Показывает информацию о чтении блоков из буферного кеша. И процент попадания блоков в буферный кеш. Чем меньше процент, тем хуже.
    
- `Top SQL by shared blocks dirtied` - информация о запросах, которые больше всего загрязняют страницы в буферном кеше
    
- `Top SQL by shared blocks written` - информация о пишущих запросах.
    

Когда мы находим проблемный или интересный запрос, мы можем нажать на ссылку и переместиться на текст самого запроса. Этот запрос уже можно выполнить в `psql`:

```sql
explain (analyze, buffers) select...
```

Там уже можно будет получить больше информации. Если `explain` выполнить несколько раз подряд, то получены будут разные результаты, это надо учитывать.
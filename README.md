# Clickhouse_servers
2 Clickhouse servers and ZooKeeper with replicated table and materialized view.

# Инструкция по установке
1. Создаем директорию /home/clickhouse/ куда копируем файл docker-compose.yaml и директорию config
Создаем сеть `sudo docker network clickhouse-net` 

2. Запускаем сервера из директории /home/clickhouse/`sudo docker-compose -f docker-compose.yaml up`

3. Все файы из папки ansible копируем в папку /home/ansible/

4. Генерим ssh ключ с помощью sudo ssh-keygen по умолчанию, без пароля.

5. В файле hosts.txt корректируем IP-адреса и id контейнеров для серверов clickhouse.
`sudo docker network inspect clickhouse-net` - смотрим IP-адреса
`sudo docker ps` - id контейнеров.

6. Из папки /home/ansible/ запускаем скипт настройки доступа по ssh на серверах.
`sudo ./install_serv_con`

7. Из папки /home/ansible/ запускаем скипт настройки окружение на локальном хосте.
`sudo ./preporation_serv`

8. Из папки /home/ansible/ запускаем скипт для завершения настройки доступа по ssh на серверах.
`sudo ./install_serv_con`

9. Проверяем доступ по ssh /home/ansible/sudo ansible serv -m ping

10. Запускаем плейбук для создания базы lab1, двух простых таблиц ruoters и regions,
реплицируемой таблицы web1 на первом сервере в первой реплике и реплицируемой таблицы 
web1 на втором сервере во второй реплике. А также создания для таблицы web1 материализованного представления 
web_view, для выборки данных, суммирующее количество запросв по каждому домену в течении 30 секунд.

11. Подключаемся к первому серверу
`sudo docker run -it --rm --network="clickhouse-net" --link clickhouse-01:clickhouse-server yandex/clickhouse-client --host clickhouse-server`
- заходим в базу lab1 - `use lab1`
- проверяем таблицы - `SHOW TABLES`

12. Вносим тестовые данные в таблицу web1
```
INSERT INTO web1 VALUES (now('Europe/Moscow'), 'http://ya.ru', 'https://ya.ru/?from=tabbar')
INSERT INTO web1 VALUES (now('Europe/Moscow'), 'http://ya.ru', 'https://ya.ru/?from=tabbar222')
INSERT INTO web1 VALUES (now('Europe/Moscow'), 'http://google.com', 'https://google.com/?from=tabbar')
INSERT INTO web1 VALUES (now('Europe/Moscow'), 'http://yandex.ru', 'https://yandex.ru/?from=tabbar')
INSERT INTO web1 VALUES (now('Europe/Moscow'), 'http://mail.ru', 'https://mail.ru/?from=tabbar222')
```
13. Подключаемся ко второму серверу
`sudo docker run -it --rm --network="clickhouse-net" --link clickhouse-02:clickhouse-server yandex/clickhouse-client --host clickhouse-server`
- заходим в базу lab1 - `use lab1`
- проверяем таблицы - `SHOW TABLES`
и убеждаемся что репликация работает
```
clickhouse-02 :) select * from web1

SELECT *
FROM web1
Query id: 8c52922d-f5ae-45a0-9187-4265277aa388
┌────────────────time─┬─domain────────────┬─uri─────────────────────────────┐
│ 2020-11-17 22:42:27 │ http://ya.ru      │ https://ya.ru/?from=tabbar      │
│ 2020-11-17 22:42:47 │ http://ya.ru      │ https://ya.ru/?from=tabbar222   │
│ 2020-11-17 22:42:53 │ http://google.com │ https://google.com/?from=tabbar │
│ 2020-11-17 22:42:58 │ http://yandex.ru  │ https://yandex.ru/?from=tabbar  │
│ 2020-11-17 22:43:03 │ http://mail.ru    │ https://mail.ru/?from=tabbar222 │
└─────────────────────┴───────────────────┴─────────────────────────────────┘

5 rows in set. Elapsed: 0.009 sec. 
```
добавив еще несколько строк в таблицу проверем работу материализованного представления
```
clickhouse-02 :) select * from web_view

SELECT *
FROM web_view
Query id: a7b655e8-5fba-4709-bc26-3c069a5e29fd
┌──────────────period─┬─domain────────────┬─count()─┐
│ 2020-11-17 22:42:00 │ http://ya.ru      │       1 │
│ 2020-11-17 22:42:30 │ http://ya.ru      │       3 │
│ 2020-11-17 22:43:00 │ http://mail.ru    │       1 │
│ 2020-11-17 22:45:00 │ http://ya.ru      │       1 │
│ 2020-11-17 22:45:30 │ http://google.com │       3 │
│ 2020-11-17 22:46:00 │ http://mail.ru    │       2 │
└─────────────────────┴───────────────────┴─────────┘

6 rows in set. Elapsed: 0.006 sec. 
```
или по конкретному домену
```
clickhouse-02 :) select * from web_view where domain='http://ya.ru'

SELECT *
FROM web_view
WHERE domain = 'http://ya.ru'
Query id: 8916b0b1-d46b-4dcb-aac1-47143be00e65
┌──────────────period─┬─domain───────┬─count()─┐
│ 2020-11-17 22:42:00 │ http://ya.ru │       1 │
│ 2020-11-17 22:42:30 │ http://ya.ru │       3 │
│ 2020-11-17 22:45:00 │ http://ya.ru │       1 │
└─────────────────────┴──────────────┴─────────┘

3 rows in set. Elapsed: 0.005 sec.
```
или смотрим перечень доменов имеющихся во всей таблице
```
clickhouse-02 :) select DISTINCT domain from web_view

SELECT DISTINCT domain
FROM web_view
Query id: 893a20f6-9cc5-4d6d-9d7c-19bd852fa609
┌─domain────────────┐
│ http://ya.ru      │
│ http://mail.ru    │
│ http://google.com │
└───────────────────┘

3 rows in set. Elapsed: 0.005 sec. 
```

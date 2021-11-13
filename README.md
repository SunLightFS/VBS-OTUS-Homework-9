# Занятие №9: Репликация

### **Задание №1**
На 1 ВМ создаем таблицы test для записи, test2 для запросов на чтение. Создаем публикацию таблицы test и подписываемся на публикацию таблицы test2 с ВМ №2. На 2 ВМ создаем таблицы test2 для записи, test для запросов на чтение. Создаем публикацию таблицы test2 и подписываемся на публикацию таблицы test1 с ВМ №1.  

#### ***Выполнение***:

После создания ВМ-1 (instance-2) и ВМ-2 (instance-3) и установки на них PostgreSQL 13 на обоих машинах выполнил следующие действия:
- Создал пароль для пользователя postgres:  
```
\password
Enter new password: test123
```
- Установил wal_level в значение "replica":  
*alter system set wal_level = replica;*
- В pg_hba.conf добавил:
```
На ВМ-1:
host    all     all     34.69.10.116/32    scram-sha-256
host    replication     all     34.69.10.116/32    scram-sha-256
На ВМ-2
host    all     all     34.123.153.255/32    scram-sha-256
host    replication     all     34.123.153.255/32    scram-sha-256
```
- В postgresql.conf изменил значение параметра listen_addresses на '\*'
- Перезапустил PostgreSQL
- Создал БД testdb:  
*create database testdb*
- Создал таблицы test и test2:
```
create table test (i int, s varchar);
create table test2 (i int, s varchar);
```
Далее на ВМ-1 создал публикацию таблицы test:  
*create publication test_pub for table test;*  
А на ВМ-2 - публикацию таблицы test2:  
*create publication test2_pub for table test2;*
Затем подписался на эти публикации:
```
ВМ-1 (test_pub) <- ВМ-2 (test_sub) (команда выполняется на ВМ-2):
create subscription test_sub
connection 'host=34.123.153.255 port=5432 user=postgres password=test123 dbname=testdb'
publication test_pub with (copy_data = true);

ВМ-1 (test2_sub) -> ВМ-2 (test2_pub) (команда выполняется на ВМ-1):
create subscription test2_sub
connection 'host=34.69.10.116 port=5432 user=postgres password=test123 dbname=testdb'
publication test2_pub with (copy_data = true);
```

Далее проверил - при вставке строки в test на ВМ-1 эта строка появляется на ВМ-2, а при вставке в таблицу test2 на ВМ-2 строка появляется на ВМ-1.  
Также проверил состояние подписок:  
На ВМ-1:
```
testdb=# SELECT * FROM pg_stat_subscription \gx
-[ RECORD 1 ]---------+------------------------------
subid                 | 16414
subname               | test2_sub
pid                   | 6713
relid                 |
received_lsn          | 0/15F7128
last_msg_send_time    | 2021-11-13 07:47:08.902621+00
last_msg_receipt_time | 2021-11-13 07:47:08.903072+00
latest_end_lsn        | 0/15F7128
latest_end_time       | 2021-11-13 07:47:08.902621+00
```
На ВМ-2:
```
testdb=# SELECT * FROM pg_stat_subscription \gx
-[ RECORD 1 ]---------+------------------------------
subid                 | 16400
subname               | test_sub
pid                   | 6367
relid                 |
received_lsn          | 0/1614890
last_msg_send_time    | 2021-11-13 07:47:16.391629+00
last_msg_receipt_time | 2021-11-13 07:47:16.392587+00
latest_end_lsn        | 0/1614890
latest_end_time       | 2021-11-13 07:47:16.391629+00
```

### **Задание №2**
3 ВМ использовать как реплику для чтения и бэкапов (подписаться на таблицы из ВМ №1 и №2 ). Небольшое описание, того, что получилось.

#### ***Выполнение***:
После создания ВМ-3 (instance-4) и установки на него PostgreSQL 13 выполнил следующие действия:
- Создал пароль для пользователя postgres:  
```
\password
Enter new password: test123
```

- В pg_hba.conf на ВМ-1 и на ВМ-2 добавил:
```
host    all     all     34.121.61.27/32    scram-sha-256
host    replication     all     34.121.61.27/32    scram-sha-256
```
- В postgresql.conf изменил значение параметра listen_addresses на '\*'
- Перезапустил PostgreSQL
- Создал БД testdb:  
*create database testdb*
- Создал таблицы test и test2:
```
create table test (i int, s varchar);
create table test2 (i int, s varchar);
```
Подписался на публикации:
```
ВМ-1 (test_pub) <- ВМ-3 (test_sub2) (команда выполняется на ВМ-3):
create subscription test_sub
connection 'host=34.123.153.255 port=5432 user=postgres password=test123 dbname=testdb'
publication test_pub with (copy_data = true);

ВМ-2 (test2_pub) <- ВМ-3 (test2_sub2) (команда выполняется на ВМ-3):
create subscription test2_sub2
connection 'host=34.69.10.116 port=5432 user=postgres password=test123 dbname=testdb'
publication test2_pub with (copy_data = true);
```

Далее проверил - при вставке строки в test на ВМ-1 эта строка появляется на ВМ-3, и при вставке в таблицу test2 на ВМ-2 строка также появляется на ВМ-3.  
Также проверил состояние подписок на ВМ-3:  
```
testdb=# SELECT * FROM pg_stat_subscription \gx
-[ RECORD 1 ]---------+------------------------------
subid                 | 16415
subname               | test_sub2
pid                   | 7196
relid                 |
received_lsn          | 0/3002F30
last_msg_send_time    | 2021-11-13 08:59:40.038811+00
last_msg_receipt_time | 2021-11-13 08:59:40.039799+00
latest_end_lsn        | 0/3002F30
latest_end_time       | 2021-11-13 08:59:40.038811+00
-[ RECORD 2 ]---------+------------------------------
subid                 | 16416
subname               | test2_sub
pid                   | 7206
relid                 |
received_lsn          | 0/15F7198
last_msg_send_time    | 2021-11-13 08:59:32.802864+00
last_msg_receipt_time | 2021-11-13 08:59:32.803101+00
latest_end_lsn        | 0/15F7198
latest_end_time       | 2021-11-13 08:59:32.802864+00
```

Из проблем - сначала неудачно подписался на изменения (я так понял, проблема возникла из-за одинакового названия у подписок на ВМ-1 и на ВМ-3) и на ВМ-1 каким-то образом пропал слот репликации, связанный с подпиской. Соответственно, подписка не работала и не удалялась, так как при удалении подписки должен удаляться связанный с ней слот репликации, которого нет. В итоге нашлось такое решение:
```
alter subscription test2_sub set (slot_name=NONE);
drop subscription test2_sub;
```
После этого я корректно пересоздал публикации и всё завелось.


### **Задание №3**
3. Реализовать горячее реплицирование для высокой доступности на 4ВМ. Источником должна выступать ВМ №3. Написать с какими проблемами столкнулись.

#### ***Выполнение***:
После создания ВМ-4 (instance-5) и установки на него PostgreSQL 13 выполнил следующие действия:
- В pg_hba.conf на ВМ-3 добавил:
```
host    replication     all             34.133.218.63/32        scram-sha-256
```
- Снял бэкап с базы, находящейся на ВМ-3:  
*pg_basebackup -p 5432 -R -h 34.121.61.27 -D /var/lib/pgsql/13/backups*
- Удалил содержимое каталога data и перенес в него содержимое каталога backups и установил на каталог data и его содержимое права пользователя postgres
- Добавил в postgresql.auto.conf параметр *hot_standby = on*
После запуска postgres на ВМ-4 в него стали попадать записи, вставляемые на ВМ-1 и ВМ-2. Также проверил состояние репликации:
```
На ВМ-3:
postgres=# SELECT * FROM pg_stat_replication \gx
-[ RECORD 1 ]----+------------------------------
pid              | 7888
usesysid         | 10
usename          | postgres
application_name | walreceiver
client_addr      | 34.133.218.63
client_hostname  |
client_port      | 53066
backend_start    | 2021-11-13 09:29:36.595106+00
backend_xmin     |
state            | streaming
sent_lsn         | 0/30003F0
write_lsn        | 0/30003F0
flush_lsn        | 0/30003F0
replay_lsn       | 0/30003F0
write_lag        |
flush_lag        |
replay_lag       |
sync_priority    | 0
sync_state       | async
reply_time       | 2021-11-13 09:43:09.319507+00


На ВМ-4:
testdb=# select * from pg_stat_wal_receiver \gx
-[ RECORD 1 ]---------+-----------------------------------------------
pid                   | 6001
status                | streaming
receive_start_lsn     | 0/3000000
receive_start_tli     | 1
written_lsn           | 0/3000060
flushed_lsn           | 0/3000060
received_tli          | 1
last_msg_send_time    | 2021-11-13 09:29:36.609995+00
last_msg_receipt_time | 2021-11-13 09:29:36.610348+00
latest_end_lsn        | 0/3000060
latest_end_time       | 2021-11-13 09:29:36.609995+00
slot_name             |
sender_host           | 34.121.61.27
sender_port           | 5432
conninfo              | user=postgres password=******** channel_binding=prefer dbname=replication host=34.121.61.27 port=5432 fallback_application_name=walreceiver sslmode=prefer sslcompression=0 ssl_min_protocol_version=TLSv1.2 gssencmode=prefer krbsrvname=postgres target_session_attrs=any
```

# Работа с уровнями изоляции транзакции в PostgreSQL

## 1. Настройка

### Создание виртуальной машины в Yandex Cloud

1. Создал виртуальную машину, следуя инструкции на сайте [yandex cloud](https://cloud.yandex.ru/en/docs/compute/quickstart/quick-create-linux). 

```
Name: kmamaev
CPU: Intel Ice Lake, 2 vCPU
RAM 4gb
SSD 10gb
Operating system: Ubuntu 22.04
Guaranteed vCPU performance: 20%
Preemtible: true
```

2. Сгенерировал новый ssh-ключ. Сделал это по привычке в Git Bash. Скопировал публичный и закинул в авторизацию на виртуальной машине в облаке.

```
Mamaev Konstantin@DESKTOP-31PAPS1 MINGW64 /
$ cd ~/.ssh

Mamaev Konstantin@DESKTOP-31PAPS1 MINGW64 /
$ ssh-keygen -t rsa -b 2048

Mamaev Konstantin@DESKTOP-31PAPS1 MINGW64 ~/.ssh
$ cat kmamaev.pub
```

3. Хотел прокинуть новый ключ в `ssh-agent`, чтобы не указывать его явно, но оказалось, что он у меня выключен. Включил через `powershell` по [инструкции](https://learn.microsoft.com/en-us/windows-server/administration/openssh/openssh_keymanagement).

```
PS C:\> Get-Service ssh-agent | Set-Service -StartupType Automatic
PS C:\> Start-Service ssh-agent
PS C:\> Get-Service ssh-agent

Status   Name               DisplayName
------   ----               -----------
Running  ssh-agent          OpenSSH Authentication Agent
```

4. Зарегистрировал ключ в `ssh-agent` и зашел на виртуальную машину.

```
PS C:\> ssh-add kmamaev
PS C:\> ssh kmamaev@<IP>
kmamaev@kmamaev:~$ 
```

### Настройка PosgtreSQL

1. Установил PostgreSQL по [инструкции](https://www.postgresql.org/download/linux/ubuntu/). Проверил, что кластер работает через `pg_lsclusters`.

```
kmamaev@kmamaev:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
16  main    5432 online postgres /var/lib/postgresql/16/main /var/log/postgresql/postgresql-16-main.log
```

2. Запустил `psql` под пользователем `postgres` и установил пароль, чтобы была возможность под этим пользователем по сети заходить.

```
kmamaev@kmamaev:~$ sudo -u postgres psql
postgres=# \password postgres
postgres=# \q
kmamaev@kmamaev:~$
```

3. Отредактировал конфигурацию кластера `postgresql.conf`, чтобы сервер слушал не только localhost. Заменил `#listen_addresses = 'localhost'` на `listen_addresses = '*'`.

```
kmamaev@kmamaev:~$ cd /etc/postgresql/16/main/
kmamaev@kmamaev:/etc/postgresql/16/main$ sudo nano postgresql.conf
```

4. Отредактировал маску в конфигурации авторизации `pg_hba.conf`, чтобы сервер разрешал авторизацию не только с локальной сети.

```
kmamaev@kmamaev:/etc/postgresql/16/main$ sudo nano pg_hba.conf
```

```
# IPv4 local connections:
host    all             all             127.0.0.1/32            scram-sha-256

# IPv4 local connections:
host    all             all             0.0.0.0/0            scram-sha-256
```

5. Перезапустил кластер.

```
kmamaev@kmamaev:/etc/postgresql/16/main$ sudo pg_ctlcluster 16 main restart
```

6. Успешно подключился через `pgAdmin` и `dbeaver`.

7. Создаем базу данных.

Полезная команда, чтобы очищать экран в `psql`:

```
postgres-# \! clear
```

Интересно, что пока нет точки с запятой, то он не завершает транзакцию. Создаю `test` и вывожу все бд в кластере. 
```
postgres=# create database test
postgres=# ;
postgres-# \list
```

## 2. Практическая часть

Запускаем две сессии. Чтобы проще их было различать обозначил их `postgres1` и `postgres2`.

1. Выключаем `AUTOCOMMIT`.

```
postgres1=# \set AUTOCOMMIT off
postgres1-# \echo :AUTOCOMMIT
off
```

```
postgres2=# \set AUTOCOMMIT off
postgres2-# \echo :AUTOCOMMIT
off
```

2. Создаем таблицу и заполняем данными в первой сессии.

```
postgres1=# create table persons(id serial, first_name text, last_name text);
CREATE TABLE

postgres1=*# insert into persons(first_name, last_name) values ('ivan', 'ivanov'), ('petr', 'petrov');
INSERT 0 2

postgres1=*# commit;
COMMIT
```

Судя по всему `*` в `postgres1=*#` обозначает, что есть открытая транзакция.

3. Вставляем данные в первой сессии, но не коммитим и читаем из второй.

```
postgres1=*# show transaction isolation level;
 transaction_isolation
-----------------------
 read committed
(1 row)

postgres1=# insert into persons(first_name, last_name) values ('sidr', 'sidorov');
INSERT 0 1
```

```
postgres2=# show transaction isolation level;
 transaction_isolation
-----------------------
 read committed
(1 row)

postgres2=*# select * from persons;
 id | first_name | last_name
----+------------+-----------
  1 | ivan       | ivanov
  2 | petr       | petrov
(2 rows)
```

Новые данные `select` не вернул, так как при уровне изоляции `read committed` новые изменения доступны только после коммита новых изменений в других транзакциях, которого в первой сессии не было. Поведение всех уровней изоляции описано в [официальной документации](https://www.postgresql.org/docs/current/transaction-iso.html).

4. Делаем коммит в первой сессии и снова запрашиваем данные во второй.

```
postgres1=*# commit;
COMMIT
```

```
postgres2=*# select * from persons;
 id | first_name | last_name
----+------------+-----------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sidr       | sidorov
(3 rows)

postgres2=*# commit;
COMMIT
```

После коммита изменений в первой сессии `select` вернул новые данные во второй, хотя там транзакция не закрывалась. То есть все, как и описано в документации, данные после коммита стали доступны. Поведение всех уровней изоляции описано в [официальной документации](https://www.postgresql.org/docs/current/transaction-iso.html).

5. Ставим уровень изоляции транзакции `repeatable read`. Вставляем данные в первой сессии, но не коммитим и читаем из второй.

```
postgres1=# set transaction isolation level repeatable read;
SET

postgres1=*# insert into persons(first_name, last_name) values ('bob', 'bobov');
INSERT 0 1
```

```
postgres2=# set transaction isolation level repeatable read;
SET

postgres2=*# select * from persons;
 id | first_name | last_name
----+------------+-----------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sidr       | sidorov
(3 rows)
```

Новые данные `select` не вернул, так как при уровне изоляции `repeatable read` видны только данные, по которым был коммит на начало транзакции. В данном случае коммита не было вообще. Поведение всех уровней изоляции описано в [официальной документации](https://www.postgresql.org/docs/current/transaction-iso.html).

6. Делаем коммит в первой сессии и читаем данные во второй.

```
postgres1=*# commit;
COMMIT
```

```
postgres2=*# select * from persons;
 id | first_name | last_name
----+------------+-----------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sidr       | sidorov
(3 rows)
```

Новые данные `select` не вернул, так как при уровне изоляции `repeatable read` видны только данные, по которым был коммит на начало транзакции. В данном случае коммит был после начала транзакции во второй сессии. Поведение всех уровней изоляции описано в [официальной документации](https://www.postgresql.org/docs/current/transaction-iso.html).

7. Делаем коммит во второй сессии и снова читаем данные.

```
postgres2=*# commit;
COMMIT

postgres2=# select * from persons;
 id | first_name | last_name
----+------------+-----------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sidr       | sidorov
  4 | bob        | bobov
(4 rows)
```

В данном случае коммит был до начала новой транзакции во второй сессии, поэтому данные наконец-то видны. Поведение всех уровней изоляции описано в [официальной документации](https://www.postgresql.org/docs/current/transaction-iso.html).
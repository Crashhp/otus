
# Работа с базами данных, пользователями и правами

### 1. Создадим новую базу данных `rights_db`

```sql
postgres=# create database rights_db;
CREATE DATABASE

postgres=# \l
    Name     |  Owner   | Encoding | Locale Provider |   Collate   |    Ctype    | ICU Locale | ICU Rules |   Access privileges
-------------+----------+----------+-----------------+-------------+-------------+------------+-----------+-----------------------
 postgres    | postgres | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |            |           |
 template0   | postgres | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |            |           | =c/postgres          +
             |          |          |                 |             |             |            |           | postgres=CTc/postgres
 template1   | postgres | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |            |           | =c/postgres          +
             |          |          |                 |             |             |            |           | postgres=CTc/postgres
 rights_db | postgres | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |            |           |

postgres=# \c rights_db
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, compression: off)
You are now connected to database "rights_db" as user "postgres".
```

### 2. Создадим новую схему `rights_schema` в базе данных `rights_db`

```sql
rights_db=# create schema rights_schema;
CREATE SCHEMA
rights_db=# select * from pg_catalog.pg_namespace;
  oid  |      nspname       | nspowner |                            nspacl
-------+--------------------+----------+---------------------------------------------------------------
    99 | pg_toast           |       10 |
    11 | pg_catalog         |       10 | {postgres=UC/postgres,=U/postgres}
  2200 | public             |     6171 | {pg_database_owner=UC/pg_database_owner,=U/pg_database_owner}
 13193 | information_schema |       10 | {postgres=UC/postgres,=U/postgres}
 16426 | rights_schema      |       10 |
(5 rows)
```

### 3. Создадим новую таблицу `rights_table` и заполним ее данными

> **Примечание**: В задании не указывалась схема, в которой создается таблица, я задумался в моменте ошибка это или нет, но решил, что видимо задумано создание в `public` схеме, что позже оказалось верным решением

Убедимся, что по умолчанию таблицы создаются в схеме `public`. Если правильно помню, то `"$user"` обозначает, что если бы была схема по имени пользователя, то он выбрал ее вместо `public`, как схему по умолчанию
```sql
testdb=# SHOW search_path;
   search_path
-----------------
 "$user", public
(1 row)
```
Создадим таблицу и вставим данные
```sql
rights_db=# create table rights_table(id int);
CREATE TABLE

rights_db=# insert into rights_table values(1);
INSERT 0 1

rights_db=# select * from rights_table;
 id
----
  1
(1 row)
```
Посмотрим информацию о созданной таблице. Здесь видно, что таблица действительно создана в сехме `public`
```sql
rights_db=# \d
            List of relations
 Schema |     Name     | Type  |  Owner
--------+--------------+-------+----------
 public | rights_table | table | postgres
```

### 4. Создадим нового пользователя, роль и выдадим нужные права

Официальная [документация](https://www.postgresql.org/docs/current/sql-createrole.html) по созданию роли

```sql
rights_db=# create role rights_readonly;
CREATE ROLE

rights_db=# select rolname from pg_roles where rolname = 'rights_readonly';
     rolname
-----------------
 rights_readonly
(1 row)
```

Официальная [документация](https://www.postgresql.org/docs/current/sql-grant.html) по выдаче прав

```sql
rights_db=#  grant connect on database rights_db to rights_readonly;
GRANT

rights_db=# grant usage on schema rights_schema to rights_readonly;
GRANT

rights_db=# grant select on all tables in schema rights_schema to rights_readonly;
GRANT
```

Официальная [документация](https://www.postgresql.org/docs/current/sql-grant.html) по созданию пользователя

```sql
rights_db=# create user rights_readonly_user with password '123123';
CREATE ROLE

rights_db=# \du rights_readonly_user
           List of roles
      Role name       | Attributes
----------------------+------------
 rights_readonly_user |
```

Официальная [документация](https://www.postgresql.org/docs/current/role-membership.html) по группам пользователей

```sql
rights_db=# grant rights_readonly to rights_readonly_user;
GRANT ROLE
```

### 5. Зайдем под пользователем `rights_readonly_user` и попытаемся считать данные из таблицы `public.rights_table`

Подключение
```bash
kmamaev@kmamaev:~$ sudo psql -h localhost -U rights_readonly_user -d rights_db
Password for user rights_readonly_user:
psql (16.2 (Ubuntu 16.2-1.pgdg22.04+1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, compression: off)
Type "help" for help.
```

```sql
rights_db=> \d
            List of relations
 Schema |     Name     | Type  |  Owner
--------+--------------+-------+----------
 public | rights_table | table | postgres
(1 row)

rights_db=> select * from rights_table;
ERROR:  permission denied for table rights_table
```

Прав нет, потому что оказывается, что у пользователя `rights_readonly_user` нет прав на схему `public`

> **Примечание**: Права можно добавить, если зайти снова под `postgres` в бд `rights_db` и выполнить, например:
> ```sql
> rights_db=# grant all privileges on schema public to rights_readonly;
> GRANT
> 
> rights_db=# grant all privileges on all tables in schema public to rights_readonly;
> GRANT
> ```
> После этого у пользователя `rights_readonly_user` появятся права на чтение таблицы `public.rights_table`. Но в рамках текущих инструкция, эти права я не давал, потому что они ломают немного процесс выполнения последующих пунктов ДЗ. Вот [тут](https://www.postgresql.org/docs/current/ddl-priv.html) еще немного информации про привелегии PUBLIC роли по умолчанию

### 6. Создадим новую таблицу `rights_table` в схеме `rights_schema`

Подключение

```bash
kmamaev@kmamaev:~$ sudo psql -h localhost -U postgres -d rights_db
Password for user postgres:
psql (16.2 (Ubuntu 16.2-1.pgdg22.04+1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, compression: off)
Type "help" for help.
```

Удалим старую таблицу и создадим новую

```sql
rights_db=# drop table rights_table;
DROP TABLE

rights_db=# create table rights_schema.rights_table(id int);
CREATE TABLE

rights_db=# insert into rights_schema.rights_table values(1);
INSERT 0 1

rights_db=# select * from rights_schema.rights_table;
 id
----
  1
(1 row)
```

### 7. Попробуем сделать `select` под пользователем `rights_readonly_user`

Подключение

```bash
kmamaev@kmamaev:~$ sudo psql -h localhost -U rights_readonly_user -d rights_db
Password for user rights_readonly_user:
psql (16.2 (Ubuntu 16.2-1.pgdg22.04+1))
SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, compression: off)
Type "help" for help.
```

```sql
rights_db=> select * from rights_schema.rights_table;
ERROR:  permission denied for table rights_table
```

Прав на просмотр данных таблицы нет, проблема в том, что когда, мы выдавали права, то этой таблицы еще не было. Чтобы получить права, то можно выдать их заново, чтобы они применились на существующий таблицы. Но, как оказалось, есть еще такой [механизм](https://www.postgresql.org/docs/current/sql-alterdefaultprivileges.html). С ним можно выставить права по умолчанию для конкретной группы пользователей

Например (под пользователем `postgres`):
```sql
rights_db=# alter default privileges in schema rights_schema grant select on tables to rights_readonly;
ALTER DEFAULT PRIVILEGES
```

> **Примечание**: Я сначала неправильно понял, думал, что настройки по умолчанию применяются на все таблицы сразу, но потом в документации прочитал, что применяются все таки именно на будущие созданные объекты, не изменяя права текущих

Тогда обновим все таки еще раз все права (под пользователем `postgres`)
```sql
rights_db=# grant select on all tables in schema rights_schema to rights_readonly;
GRANT
```

Пробуем считать данные (под пользователем `rights_readonly_user`). Теперь все работает

```sql
rights_db=> select * from rights_schema.rights_table;
 id
----
  1
(1 row)
```

### 8. Попробуем создать новую таблицу в схеме `public` под пользователем `rights_readonly_user`

```sql
rights_db=> create table rights_public_table(id int);
ERROR:  permission denied for schema public
LINE 1: create table rights_public_table(id int);
```

> **Примечание**: Тут не работает конкретно у меня, потому что на 16ой версии `postgres` нет прав на создание по умолчанию в схеме `public`

Добавим право на создание (под пользователем `postgres`):
```sql
rights_db=# grant create on schema public to rights_readonly;
GRANT
```

Выполним запрос на создание таблицы еще раз (под пользователем `rights_readonly_user`):
```sql
rights_db=> create table rights_public_table(id int);
CREATE TABLE

rights_db=> insert into rights_public_table values (1);
INSERT 0 1

rights_db=> select * from rights_public_table;
 id
----
  1
(1 row)
```

> **Примечание**: К моему удивлению `insert` и `select` сразу работают на этой таблице, если я правильно понял, то дело в том, что теперь пользователь `rights_readonly_user` является не обычным пользователем, а владельцем таблицы `rights_public_table` и поэтому все работает

Удалить права на создание можно также, как и добавляли
```sql
rights_db=# revoke create on schema public from rights_readonly;
REVOKE
```

> **Примечание**: После выполнения ДЗ я посмотрел в шпаргалку и да, оказывается можно было забрать права не у нашей роли на схему `public`, а у системной роли `PUBLIC`. Например:
> ```sql
> rights_db=# revoke create on schema public from public;
> REVOKE
> rights_db=# revoke all on database rights_db from public;
> REVOKE
> ```
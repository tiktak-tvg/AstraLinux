### Роли в СУБД PostgreSQL (Astra Linux Special Edition)
#### 1. Роли в СУБД PostgreSQL
Роль — это центральное понятие в системе управления доступом PostgreSQL. Роль может представлять как отдельного пользователя базы данных, так и группу пользователей. В отличие от многих других СУБД, в PostgreSQL нет отдельного понятия «пользователь» — вместо него используется понятие «роль с атрибутом LOGIN».

Ключевые особенности ролей:

- Роль = пользователь или группа. Любая роль может включать в себя другие роли (быть «групповой ролью») и одновременно может подключаться к базе данных.
- Определяются на уровне кластера. Одна роль может подключаться к разным базам данных и быть владельцем объектов в разных БД.
- Не связаны с пользователями ОС. Роли никак не связаны с именами пользователей операционной системы, хотя некоторые программы (например, psql) берут имя пользователя ОС как имя роли по умолчанию.
- Псевдороль public. Существует специальная псевдороль public, в которую всегда неявно включены все остальные роли.
- Начальная роль. При создании кластера определяется одна начальная роль, имеющая суперпользовательский доступ (обычно postgres).

Атрибуты ролей:

Атрибут	        | Описание
--------------- | ------------------------------------
LOGIN	 | Возможность подключения к серверу
SUPERUSER	 | Права суперпользователя
CREATEDB	 | Возможность создавать базы данных
CREATEROLE	 | Возможность создавать роли
REPLICATION	 | Использование протокола репликации
INHERIT / NOINHERIT	 | Автоматическое наследование прав групповых ролей
CONNECTION LIMIT	 | Ограничение количества одновременных сеансов
VALID UNTIL	 | Ограничение срока действия роли

Атрибуты LOGIN, SUPERUSER, CREATEDB и CREATEROLE можно рассматривать как особые привилегии, но они никогда не наследуются — для их использования необходимо переключиться на роль с помощью SET ROLE.

Создание ролей:

```sql
-- Создать роль-пользователя (с правом входа)
CREATE ROLE username LOGIN PASSWORD 'password';

-- Создать групповую роль (без права входа, как правило)
CREATE ROLE groupname;

-- Создать суперпользователя
CREATE ROLE adminuser SUPERUSER LOGIN PASSWORD 'password';

-- Создать роль с несколькими атрибутами
CREATE ROLE creator CREATEDB CREATEROLE LOGIN;
```
Просмотр информации о ролях:

```sql
SELECT * FROM pg_roles;
\du
```
Изменение и удаление ролей:

```sql
-- Изменение атрибутов
ALTER ROLE username CREATEDB;

-- Переименование
ALTER ROLE oldname RENAME TO newname;

-- Удаление (перед удалением необходимо передать объекты и отозвать привилегии)
REASSIGN OWNED BY username TO newowner;
DROP OWNED BY username;
DROP ROLE username;
```
#### 2. Владельцы объектов БД
Владелец объекта — это роль, которая создала объект (таблицу, индекс, представление и т.д.). Владельцем считается роль, от имени которой был выполнен оператор создания объекта.

Важные особенности:

- Наследование владения. Владельцами объекта считаются также все роли, включённые в роль, создавшую объект.
- Смена владельца. Владелец объекта может быть изменён командой ALTER ... OWNER TO:

```sql
ALTER TABLE table_name OWNER TO new_owner;
ALTER DATABASE dbname OWNER TO new_owner;
```
Требования для смены владельца. Чтобы назначить другой роли права владения объектом, необходимо иметь право SET ROLE для этой роли.

Пример:

```sql
-- Создать таблицу от имени роли appuser
SET ROLE appuser;
CREATE TABLE accounts (id serial, balance numeric);
RESET ROLE;

-- Передать владение другой роли
ALTER TABLE accounts OWNER TO adminuser;
```
#### 3. Членство в роли
Членство в роли (role membership) — это механизм, позволяющий включать роли в другие роли для группового управления правами.

Добавление и удаление членов:

```sql
-- Добавить роль в групповую роль
GRANT group_role TO role1, role2;

-- Удалить роль из групповой роли
REVOKE group_role FROM role1, role2;
```
Особенности членства:

- Членом роли может быть другая групповая роль — в действительности нет различий между групповыми и негрупповыми ролями.
- Замыкание по кругу не допускается. База данных не допускает циклического членства.
- Управление членством роли PUBLIC запрещено.
- Передача права управления. При включении роли в группу можно передать право управления членством:

```sql
GRANT group_role TO role2 WITH ADMIN OPTION;
```
- Теперь role2 может управлять членством в group_role, включая передачу права управления другим ролям.
- Способы использования прав групповой роли:
- Явное переключение через SET ROLE. Каждый член группы может выполнить SET ROLE group_role, чтобы временно «стать» групповой ролью. В этом состоянии сеанс использует полномочия групповой роли, и все создаваемые объекты принадлежат групповой роли.
- Автоматическое наследование (атрибут INHERIT). Роли с атрибутом INHERIT автоматически используют права всех ролей, членами которых они являются. Роли с атрибутом NOINHERIT этого не делают.

Пример наследования:

```sql
CREATE ROLE joe LOGIN INHERIT;
CREATE ROLE admin NOINHERIT;
CREATE ROLE wheel NOINHERIT;

GRANT admin TO joe;
GRANT wheel TO admin;
```
После подключения с ролью joe:
- Сеанс использует права joe и права admin (так как joe наследует права admin).
- Права wheel недоступны, так как членство получено через admin, у которой атрибут NOINHERIT.
- После выполнения SET ROLE admin:
- Сеанс использует только права admin, права joe недоступны.

#### 4. Использование прав групповой роли
Сценарий использования групповых ролей:

Групповые роли позволяют централизованно управлять правами. Вместо того чтобы выдавать привилегии каждому пользователю отдельно, права выдаются групповой роли, а затем пользователи включаются в эту группу.

Пример:

```sql
-- Создать групповую роль для разработчиков
CREATE ROLE developers;

-- Выдать групповой роли права на схему и таблицы
GRANT USAGE ON SCHEMA app TO developers;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA app TO developers;

-- Создать пользователей и включить их в группу
CREATE ROLE dev1 LOGIN PASSWORD 'pass1';
CREATE ROLE dev2 LOGIN PASSWORD 'pass2';
GRANT developers TO dev1, dev2;
```
Теперь dev1 и dev2 автоматически наследуют все права, выданные роли developers (при условии, что у них установлен атрибут INHERIT, который является значением по умолчанию).

Преимущества:

- Упрощение управления правами: при изменении прав группы они автоматически применяются ко всем членам.
- Возможность создавать иерархии ролей.
- Гибкое управление через SET ROLE и INHERIT/NOINHERIT.

#### 5. Предопределённые роли
PostgreSQL предоставляет набор предопределённых ролей, которые дают доступ к часто востребованным, но необщедоступным функциям и данным. Администраторы (включая роли с привилегией CREATEROLE) могут выдавать эти роли пользователям и/или другим ролям с помощью команды GRANT.

Основные предопределённые роли:

Роль	Предоставляемый доступ
Роль	          | Предоставляемый доступ
--------------- | ----------------------------------
pg_read_all_data	 | Чтение всех данных (таблиц, представлений, последовательностей), как если бы роль имела права SELECT на эти объекты и USAGE на все схемы
pg_write_all_data	 | Запись всех данных (INSERT, UPDATE, DELETE), как если бы роль имела соответствующие права
pg_read_all_settings	 | Чтение всех конфигурационных переменных, даже тех, которые обычно видны только суперпользователям
pg_read_all_stats	 | Чтение всех представлений pg_stat_* и использование расширений статистики
pg_stat_scan_tables	 | Выполнение функций мониторинга, которые могут брать блокировки ACCESS SHARE на таблицы
pg_monitor	 | Чтение/выполнение различных представлений и функций мониторинга. Является членом pg_read_all_settings, pg_read_all_stats и pg_stat_scan_tables
pg_database_owner	 | Не имеет прав по умолчанию. Членство состоит неявно из текущего владельца базы данных
pg_signal_backend	 | Отправка сигналов другому backend-процессу для отмены запроса или завершения сеанса
pg_read_server_files	 | Чтение файлов с сервера с помощью COPY и других функций доступа к файлам
pg_write_server_files	 | Запись файлов на сервере
pg_execute_server_program	 | Выполнение программ на сервере базы данных

Использование предопределённых ролей:

```sql
-- Выдать роль мониторинга пользователю
GRANT pg_monitor TO monitor_user;

-- Выдать роль для чтения всех данных
GRANT pg_read_all_data TO analyst_user;

-- Выдать роль для сигналов backend-процессам
GRANT pg_signal_backend TO admin_user;
```
> Важно: роль pg_database_owner не может быть членом какой-либо роли, и никакая роль не может быть её членом.

#### 6. Практическая работа
##### 6.1. Создание суперпользователя
```bash
# Подключиться к PostgreSQL от имени postgres
sudo -u postgres psql
```
```sql
-- Создать суперпользователя
CREATE ROLE dbadmin SUPERUSER LOGIN PASSWORD 'AdminPass123!';

-- Проверить создание
\du dbadmin
```
Ожидаемый результат: роль dbadmin с атрибутами Superuser, Create role, Create DB, Login.

##### 6.2. Создание групповой роли
```sql
-- Создать групповую роль для разработчиков
CREATE ROLE developers;

-- Создать групповую роль для администраторов БД
CREATE ROLE db_operators;

-- Проверить создание
\du
```
Групповые роли не имеют атрибута LOGIN, поэтому не могут подключаться к серверу напрямую.

##### 6.3. Создание ролей для пользователей
```sql
-- Создать роли-пользователи
CREATE ROLE dev_ivanov LOGIN PASSWORD 'IvanovPass1';
CREATE ROLE dev_petrov LOGIN PASSWORD 'PetrovPass2';
CREATE ROLE operator_sidorov LOGIN PASSWORD 'SidorovPass3';

-- Проверить создание
\du
```
##### 6.4. Включение ролей в групповую роль
```sql
-- Включить разработчиков в группу developers
GRANT developers TO dev_ivanov, dev_petrov;

-- Включить оператора в группу db_operators
GRANT db_operators TO operator_sidorov;

-- Проверить членство
\du developers
\du db_operators
```
Ожидаемый результат: в выводе \du developers будут указаны роли dev_ivanov и dev_petrov как члены группы.

##### 6.5. Выдача прав групповым ролям
```sql
-- Подключиться к базе данных
\c appdb

-- Выдать права групповой роли developers
GRANT USAGE ON SCHEMA public TO developers;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO developers;

-- Выдать права групповой роли db_operators
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO db_operators;
GRANT CREATE ON SCHEMA public TO db_operators;
```
##### 6.6. Проверка наследования прав
```sql
-- Проверить права роли dev_ivanov (должны включать права developers)
\dp

-- Проверить права роли operator_sidorov (должны включать права db_operators)
\dp
```
##### 6.7. Использование предопределённой роли
```sql
-- Выдать роль мониторинга оператору
GRANT pg_monitor TO operator_sidorov;

-- Проверить членство
\du operator_sidorov
```
##### 6.8. Полный сценарий (сводка)
```sql
-- От имени postgres
CREATE ROLE dbadmin SUPERUSER LOGIN PASSWORD 'AdminPass123!';
CREATE ROLE developers;
CREATE ROLE db_operators;
CREATE ROLE dev_ivanov LOGIN PASSWORD 'IvanovPass1';
CREATE ROLE dev_petrov LOGIN PASSWORD 'PetrovPass2';
CREATE ROLE operator_sidorov LOGIN PASSWORD 'SidorovPass3';

GRANT developers TO dev_ivanov, dev_petrov;
GRANT db_operators TO operator_sidorov;
GRANT pg_monitor TO operator_sidorov;

-- Подключиться к рабочей базе
\c appdb
GRANT USAGE ON SCHEMA public TO developers;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO developers;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO db_operators;
GRANT CREATE ON SCHEMA public TO db_operators;
```
##### 6.9. Проверка результатов
```sql
-- Список всех ролей
\du

-- Детали по конкретной роли
\du dbadmin
\du developers
\du dev_ivanov

-- Права на объекты
\dp
```
Ожидаемый результат:
- dbadmin — суперпользователь с правами входа.
- developers — групповая роль без права входа, членами которой являются dev_ivanov и dev_petrov.
- db_operators — групповая роль без права входа, членом которой является operator_sidorov.
- operator_sidorov — член db_operators и pg_monitor.

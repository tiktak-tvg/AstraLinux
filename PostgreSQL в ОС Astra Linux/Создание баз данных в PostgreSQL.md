### Создание баз данных в PostgreSQL
#### 1. Шаблоны баз данных
Команда CREATE DATABASE в PostgreSQL работает путём копирования существующей базы данных. По умолчанию копируется стандартная системная база template1, которая и является шаблоном для создания новых баз.

Ключевые особенности шаблонов:

Шаблон	            | Назначение	              | Изменяемость
------------------- | ------------------------- | --------------------
template1	 | Шаблон по умолчанию. Всё, что добавлено в template1, автоматически попадает в новые базы	   | Можно изменять
template0	 | «Чистая» база, содержащая только стандартные объекты. Используется для восстановления дампов и создания БД с другой кодировкой	 | Нельзя изменять после инициализации кластера

Если добавить объекты в template1 (например, установить расширение plperl), они будут скопированы во все новые пользовательские базы без дополнительных действий.

Важно: CREATE DATABASE не копирует права GRANT уровня базы из исходной БД — новая база получает права по умолчанию.

Для создания базы на основе template0:

```sql
CREATE DATABASE dbname TEMPLATE template0;
```
Или из командной строки:

```bash
createdb -T template0 dbname
```
#### 2. Создание БД
Синтаксис команды CREATE DATABASE:

```sql
CREATE DATABASE имя_базы
    [ WITH ]
    [ OWNER [=] имя_владельца ]
    [ TEMPLATE [=] шаблон ]
    [ ENCODING [=] кодировка ]
    [ TABLESPACE [=] табличное_пространство ];
```
Примеры:

```sql
-- Создание базы на основе template1 (по умолчанию)
CREATE DATABASE mydb;

-- Создание базы с указанием владельца
CREATE DATABASE mydb OWNER adminuser;

-- Создание базы на основе template0 с кодировкой UTF8
CREATE DATABASE mydb TEMPLATE template0 ENCODING 'UTF8';
```
#### 3. Управление БД
Управление базами данных осуществляется командой ALTER DATABASE.

Основные формы команды:

Форма               | Описание	       
------------------- | ----------------------
ALTER DATABASE name RENAME TO new_name	 | Переименование базы данных
ALTER DATABASE name OWNER TO new_owner	 | Смена владельца
ALTER DATABASE name SET TABLESPACE new_tablespace	 | Смена табличного пространства по умолчанию
ALTER DATABASE name SET parameter TO value	 | Установка параметра конфигурации
ALTER DATABASE name RESET parameter	 | Сброс параметра
ALTER DATABASE name CONNECTION LIMIT n	 | Ограничение числа подключений

Примеры:

```sql
-- Переименование базы
ALTER DATABASE olddb RENAME TO newdb;

-- Смена владельца
ALTER DATABASE mydb OWNER TO adminuser;

-- Установка параметра для конкретной базы
ALTER DATABASE mydb SET work_mem = '128MB';

-- Ограничение подключений
ALTER DATABASE mydb CONNECTION LIMIT 10;
```
Важно: текущую базу нельзя переименовать — нужно подключиться к другой базе.

#### 4. Схемы в БД
Схема (schema) — это именованное пространство имён внутри базы данных, содержащее таблицы, типы, функции и другие объекты.

Зачем нужны схемы:

- Позволяют нескольким пользователям использовать одну базу без конфликтов имён;
- Организуют объекты в логические группы;
- Изолируют сторонние приложения друг от друга.

Особенности:

- Одно и то же имя объекта может использоваться в разных схемах без конфликта (например, schema1.mytable и myschema.mytable);
- Схемы не могут быть вложенными;
- Каждая новая база содержит схему public по умолчанию.

#### 5. Работа со схемами
Создание схемы:

```sql
CREATE SCHEMA myschema;
```
Создание схемы с указанием владельца:

```sql
CREATE SCHEMA myschema AUTHORIZATION username;
```
Создание объекта в схеме (полное имя):

```sql
CREATE TABLE myschema.mytable (id serial, name text);
```
Удаление пустой схемы:

```sql
DROP SCHEMA myschema;
```
Удаление схемы со всем содержимым:

```sql
DROP SCHEMA myschema CASCADE;
```
Просмотр схем:

```sql
\dn
```
Установка схемы по умолчанию для сессии:

```sql
SET search_path TO myschema, public;
```
#### 6. Каталог PGDATA
PGDATA — это каталог данных кластера, в котором хранятся все файлы конфигурации и данные. Типичное расположение — /var/lib/pgsql/data или /var/lib/postgresql/<версия>/<кластер>.

Структура PGDATA:

Элемент             | Описание	       
------------------- | ----------------------
base/	 | Подкаталоги баз данных (по OID)
global/	 | Общесистемные таблицы (pg_database и др.)
pg_wal/	 | Журналы предзаписи (WAL)
pg_tblspc/	 | Символические ссылки на табличные пространства
postgresql.conf | Основной конфигурационный файл
pg_hba.conf	 | Настройки аутентификации
pg_ident.conf	 | Сопоставление пользователей
PG_VERSION	 | Файл с версией PostgreSQL

#### 7. Табличные пространства
Табличное пространство (tablespace) — это расположение в файловой системе, где PostgreSQL хранит объекты базы данных (таблицы, индексы).

Зачем нужны табличные пространства:

- Распределение дискового ввода-вывода между дисками;
- Размещение тяжёлых объектов на быстрых дисках (SSD);
- Хранение архивных данных на дешёвых носителях.
- Создание табличного пространства:

```bash
# Создать каталог на уровне ОС
sudo mkdir /mnt/pg_tblspc/fastspace
sudo chown postgres:postgres /mnt/pg_tblspc/fastspace
```
```sql
-- Создать табличное пространство в PostgreSQL (только суперпользователь)
CREATE TABLESPACE fastspace LOCATION '/mnt/pg_tblspc/fastspace';
```
Использование табличного пространства при создании объектов:

```sql
CREATE TABLE heavy_table (id serial, data text) TABLESPACE fastspace;
CREATE INDEX heavy_idx ON heavy_table(data) TABLESPACE fastspace;
```
Просмотр табличных пространств:

```sql
SELECT * FROM pg_tablespace;
SELECT spcname, pg_tablespace_location(oid) FROM pg_tablespace;
```
#### 8. Управление ТП
Просмотр существующих табличных пространств:

```sql
\db
```
Перемещение объектов между табличными пространствами:

```sql
-- Перемещение таблицы
ALTER TABLE old_table SET TABLESPACE fastspace;

-- Перемещение индекса
ALTER INDEX old_index SET TABLESPACE fastspace;

-- Перемещение всех таблиц из одного ТП в другой
ALTER TABLE ALL IN TABLESPACE old_space SET TABLESPACE new_space;
```
При перемещении таблицы её индексы не перемещаются автоматически — их нужно переносить отдельно.

Удаление табличного пространства:

```sql
DROP TABLESPACE fastspace;
```
Перед удалением ТП необходимо удалить или переместить все объекты, которые в нём находятся.

#### 9. Практическая работа
##### 9.1. Расширение шаблона template1 и создание БД на основе изменённого шаблона
```bash
# Подключиться к template1 от имени postgres
sudo -u postgres psql -d template1
```
В сессии psql:

```sql
-- Установить расширение в template1
CREATE EXTENSION IF NOT EXISTS pg_trgm;

-- Создать общую схему
CREATE SCHEMA shared_schema;

-- Выйти
\q
```
Создать базу данных на основе изменённого шаблона:

```bash
sudo -u postgres psql -c "CREATE DATABASE based_on_template1;"
```
Проверить, что расширение и схема скопировались:

```bash
sudo -u postgres psql -d based_on_template1 -c "\dx"
sudo -u postgres psql -d based_on_template1 -c "\dn"
```
##### 9.2. Переименование БД
```bash
sudo -u postgres psql -c "ALTER DATABASE based_on_template1 RENAME TO renamed_db;"
```
Проверить:

```bash
sudo -u postgres psql -c "\l"
```
##### 9.3. Изменение параметров БД
```bash
sudo -u postgres psql -c "ALTER DATABASE renamed_db SET work_mem = '64MB';"
sudo -u postgres psql -c "ALTER DATABASE renamed_db CONNECTION LIMIT 20;"
```
Проверить параметры:

```bash
sudo -u postgres psql -d renamed_db -c "SHOW work_mem;"
```
##### 9.4. Определение размера БД
```bash
sudo -u postgres psql -c "SELECT pg_size_pretty(pg_database_size('renamed_db'));"
```
##### 9.5. Работа со схемами
```bash
sudo -u postgres psql -d renamed_db
```
В сессии psql:

```sql
-- Создать схему
CREATE SCHEMA app_schema;

-- Создать таблицу в схеме
CREATE TABLE app_schema.users (id serial PRIMARY KEY, name text);

-- Вставить данные
INSERT INTO app_schema.users (name) VALUES ('Иван'), ('Пётр');

-- Просмотреть схемы
\dn

-- Просмотреть таблицы в схеме
\dt app_schema.*

-- Выйти
\q
```
##### 9.6. Работа с табличными пространствами
Создание ТП:

```bash
sudo mkdir -p /mnt/pg_tblspc/fastspace
sudo chown postgres:postgres /mnt/pg_tblspc/fastspace
```
```bash
sudo -u postgres psql -c "CREATE TABLESPACE fastspace LOCATION '/mnt/pg_tblspc/fastspace';"
```
Создание таблицы в ТП:

```bash
sudo -u postgres psql -d renamed_db -c "CREATE TABLE app_schema.big_data (id serial, payload text) TABLESPACE fastspace;"
```
Просмотр ТП:

```bash
sudo -u postgres psql -c "\db"
```
##### 9.7. Перенос данных и удаление ТП
Перенос таблицы из fastspace в pg_default:

```bash
sudo -u postgres psql -d renamed_db -c "ALTER TABLE app_schema.big_data SET TABLESPACE pg_default;"
```
Проверить, что таблица перемещена:

```bash
sudo -u postgres psql -d renamed_db -c "SELECT relname, reltablespace FROM pg_class WHERE relname = 'big_data';"
```
Удалить ТП (после переноса всех объектов):

```bash
sudo -u postgres psql -c "DROP TABLESPACE fastspace;"
```
Проверить удаление:

```bash
sudo -u postgres psql -c "\db"
```
##### 9.8. Полная проверка результатов
```bash
# Список баз данных
sudo -u postgres psql -c "\l"

# Размер базы
sudo -u postgres psql -c "SELECT pg_size_pretty(pg_database_size('renamed_db'));"

# Схемы в базе
sudo -u postgres psql -d renamed_db -c "\dn"

# Таблицы в схеме
sudo -u postgres psql -d renamed_db -c "\dt app_schema.*"

# Табличные пространства
sudo -u postgres psql -c "\db"
```

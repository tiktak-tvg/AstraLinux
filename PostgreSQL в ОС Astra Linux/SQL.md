### SQL: Функции, процедуры, составные типы в PostgreSQL
#### 1. Функции
Функция — это именованный блок кода, который принимает параметры, выполняет действия и возвращает значение. Функции в PostgreSQL могут быть написаны на разных языках: SQL, PL/pgSQL, Python, C и других.

##### 1.1. Виды функций

Вид                 | 	Описание
------------------- | -------------------------
SQL-функции	 | Содержат один или несколько SQL-запросов
PL/pgSQL-функции	 | Используют процедурное расширение с переменными, циклами, условиями
Функции, возвращающие скаляр	 | Возвращают одно значение (INTEGER, TEXT, NUMERIC и т.д.)
Функции, возвращающие набор	 | Возвращают таблицу (RETURNS TABLE, RETURNS SETOF)
Функции, возвращающие составной тип	 | Возвращают запись с несколькими полями

##### 1.2. Создание простой SQL-функции
```sql
CREATE OR REPLACE FUNCTION get_book_price(p_book_id INTEGER)
RETURNS NUMERIC AS $$
    SELECT price FROM books WHERE book_id = p_book_id;
$$ LANGUAGE SQL;
```
Вызов:

```sql
SELECT get_book_price(1);
```
##### 1.3. Функция на PL/pgSQL
```sql
CREATE OR REPLACE FUNCTION get_customer_discount(p_customer_id INTEGER)
RETURNS NUMERIC AS $$
DECLARE
    v_discount NUMERIC;
    v_total_spent NUMERIC;
BEGIN
    -- Получить сумму покупок клиента
    SELECT COALESCE(SUM(total_amount), 0)
    INTO v_total_spent
    FROM orders
    WHERE customer_id = p_customer_id
      AND status != 'cancelled';

    -- Определить скидку в зависимости от суммы
    IF v_total_spent > 10000 THEN
        v_discount := 15;
    ELSIF v_total_spent > 5000 THEN
        v_discount := 10;
    ELSIF v_total_spent > 1000 THEN
        v_discount := 5;
    ELSE
        v_discount := 0;
    END IF;

    RETURN v_discount;
END;
$$ LANGUAGE plpgsql;
```
Вызов:

```sql
SELECT get_customer_discount(1);
```
##### 1.4. Функция, возвращающая таблицу
```sql
CREATE OR REPLACE FUNCTION get_books_by_category(p_category_name VARCHAR)
RETURNS TABLE (
    book_id INTEGER,
    title VARCHAR,
    price NUMERIC,
    stock_quantity INTEGER
) AS $$
BEGIN
    RETURN QUERY
    SELECT b.book_id, b.title, b.price, COALESCE(s.quantity, 0)
    FROM books b
    LEFT JOIN stock s ON b.book_id = s.book_id
    JOIN categories c ON b.category_id = c.category_id
    WHERE c.name = p_category_name;
END;
$$ LANGUAGE plpgsql;
```
Вызов:

```sql
SELECT * FROM get_books_by_category('Классика');
```
##### 1.5. Функция с параметрами по умолчанию
```sql
CREATE OR REPLACE FUNCTION count_orders(
    p_status VARCHAR DEFAULT 'delivered',
    p_from_date DATE DEFAULT '2000-01-01'
)
RETURNS INTEGER AS $$
DECLARE
    v_count INTEGER;
BEGIN
    SELECT COUNT(*) INTO v_count
    FROM orders
    WHERE status = p_status
      AND order_date >= p_from_date;
    RETURN v_count;
END;
$$ LANGUAGE plpgsql;
```
Вызов:

```sql
SELECT count_orders();                          -- status='delivered'
SELECT count_orders('processing');              -- status='processing'
SELECT count_orders('delivered', '2024-01-01'); -- оба параметра
SELECT count_orders(p_from_date => '2024-01-01'); -- именованный параметр
```
##### 1.6. Функция с переменным числом аргументов (VARIADIC)
```sql
CREATE OR REPLACE FUNCTION sum_prices(VARIADIC p_prices NUMERIC[])
RETURNS NUMERIC AS $$
DECLARE
    v_total NUMERIC := 0;
    v_price NUMERIC;
BEGIN
    FOREACH v_price IN ARRAY p_prices LOOP
        v_total := v_total + v_price;
    END LOOP;
    RETURN v_total;
END;
$$ LANGUAGE plpgsql;
```
Вызов:

```sql
SELECT sum_prices(100.50, 200.75, 300.00);
```
##### 1.7. Удаление функции
```sql
DROP FUNCTION IF EXISTS get_book_price(INTEGER);
DROP FUNCTION IF EXISTS get_books_by_category(VARCHAR);
```
#### 2. Процедуры
Процедура — это именованный блок кода, который выполняет действия, но не возвращает значение (в отличие от функции). Процедуры появились в PostgreSQL 11.

##### 2.1. Отличия процедур от функций

Характеристика      | Функция                   | Процедура
------------------- | ------------------------- | -------------------------
Возврат значения	 | Обязательно	 | Не возвращает
Вызов в SELECT	 | Да	 | Нет
Вызов через CALL	 | Нет	 | Да
Транзакции внутри	 | Нельзя управлять	 | Можно (COMMIT, ROLLBACK)
Использование в триггерах	 | Да	 | Нет

##### 2.2. Создание процедуры
```sql
CREATE OR REPLACE PROCEDURE add_book(
    p_isbn VARCHAR,
    p_title VARCHAR,
    p_publisher_id INTEGER,
    p_category_id INTEGER,
    p_price NUMERIC,
    p_cost_price NUMERIC
)
LANGUAGE plpgsql
AS $$
BEGIN
    INSERT INTO books (isbn, title, publisher_id, category_id, price, cost_price)
    VALUES (p_isbn, p_title, p_publisher_id, p_category_id, p_price, p_cost_price);

    RAISE NOTICE 'Книга "%" добавлена', p_title;
END;
$$;
```
Вызов:

```sql
CALL add_book('978-5-04-1160005', 'Анна Каренина', 1, 2, 900.00, 550.00);
```
##### 2.3. Процедура с управлением транзакциями
```sql
CREATE OR REPLACE PROCEDURE process_order(p_order_id INTEGER)
LANGUAGE plpgsql
AS $$
DECLARE
    v_status VARCHAR;
BEGIN
    -- Получить текущий статус
    SELECT status INTO v_status FROM orders WHERE order_id = p_order_id;

    IF v_status IS NULL THEN
        RAISE EXCEPTION 'Заказ % не найден', p_order_id;
    END IF;

    -- Изменить статус
    UPDATE orders SET status = 'processing' WHERE order_id = p_order_id;

    -- Зафиксировать промежуточный результат
    COMMIT;

    -- Выполнить резервирование на складе
    UPDATE stock s
    SET reserved = reserved + oi.quantity
    FROM order_items oi
    WHERE oi.order_id = p_order_id AND s.book_id = oi.book_id;

    -- Зафиксировать окончательно
    COMMIT;

    RAISE NOTICE 'Заказ % обработан', p_order_id;
EXCEPTION
    WHEN OTHERS THEN
        ROLLBACK;
        RAISE;
END;
$$;
```
Вызов:

```sql
CALL process_order(2);
```
##### 2.4. Процедура с выходными параметрами
```sql
CREATE OR REPLACE PROCEDURE get_book_info(
    p_book_id INTEGER,
    INOUT p_title VARCHAR,
    INOUT p_price NUMERIC,
    INOUT p_stock INTEGER
)
LANGUAGE plpgsql
AS $$
BEGIN
    SELECT b.title, b.price, COALESCE(s.quantity, 0)
    INTO p_title, p_price, p_stock
    FROM books b
    LEFT JOIN stock s ON b.book_id = s.book_id
    WHERE b.book_id = p_book_id;
END;
$$;
```
Вызов:

```sql
CALL get_book_info(1, NULL, NULL, NULL);
```
##### 2.5. Удаление процедуры
```sql
DROP PROCEDURE IF EXISTS add_book(VARCHAR, VARCHAR, INTEGER, INTEGER, NUMERIC, NUMERIC);
DROP PROCEDURE IF EXISTS process_order(INTEGER);
```
#### 3. Составные типы
Составной тип (Composite Type) — это тип данных, который представляет собой структуру из нескольких полей, аналогичную строке таблицы. Составные типы используются для возврата нескольких значений из функций, передачи параметров и организации данных.

##### 3.1. Создание составного типа
```sql
CREATE TYPE book_info AS (
    book_id INTEGER,
    title VARCHAR(300),
    isbn VARCHAR(20),
    price NUMERIC(10,2),
    stock_quantity INTEGER
);
```
##### 3.2. Использование составного типа в функции
```sql
CREATE OR REPLACE FUNCTION get_full_book_info(p_book_id INTEGER)
RETURNS book_info AS $$
DECLARE
    v_result book_info;
BEGIN
    SELECT
        b.book_id,
        b.title,
        b.isbn,
        b.price,
        COALESCE(s.quantity, 0)
    INTO v_result
    FROM books b
    LEFT JOIN stock s ON b.book_id = s.book_id
    WHERE b.book_id = p_book_id;

    RETURN v_result;
END;
$$ LANGUAGE plpgsql;
```
Вызов и доступ к полям:

```sql
-- Получить всю запись
SELECT * FROM get_full_book_info(1);

-- Доступ к отдельным полям
SELECT (get_full_book_info(1)).title;
SELECT (get_full_book_info(1)).price;
```
##### 3.3. Составной тип в таблице
```sql
CREATE TYPE address_type AS (
    city VARCHAR(100),
    street VARCHAR(200),
    postal_code VARCHAR(20)
);

CREATE TABLE customers_v2 (
    customer_id SERIAL PRIMARY KEY,
    last_name VARCHAR(100),
    first_name VARCHAR(100),
    address address_type,
    email VARCHAR(150)
);
```
Вставка данных:

```sql
INSERT INTO customers_v2 (last_name, first_name, address, email)
VALUES (
    'Иванов',
    'Иван',
    ROW('Москва', 'ул. Ленина, д. 1', '101000'),
    'ivanov@example.com'
);

-- Или с явным приведением
INSERT INTO customers_v2 (last_name, first_name, address, email)
VALUES (
    'Петрова',
    'Мария',
    ROW('Санкт-Петербург', 'Невский пр., д. 10', '190000')::address_type,
    'petrova@example.com'
);
```
Запрос к полям составного типа:

```sql
SELECT
    customer_id,
    last_name,
    (address).city,
    (address).street,
    (address).postal_code
FROM customers_v2;
```
##### 3.4. Составной тип на основе таблицы
В PostgreSQL можно использовать имя таблицы как составной тип:

```sql
-- books уже существует как таблица
CREATE OR REPLACE FUNCTION get_book_record(p_book_id INTEGER)
RETURNS books AS $$
DECLARE
    v_book books;
BEGIN
    SELECT * INTO v_book FROM books WHERE book_id = p_book_id;
    RETURN v_book;
END;
$$ LANGUAGE plpgsql;
```
Вызов:

```sql
SELECT * FROM get_book_record(1);
SELECT (get_book_record(1)).title;
SELECT (get_book_record(1)).price;
```
##### 3.5. Составной тип в качестве параметра
```sql
CREATE OR REPLACE FUNCTION add_customer(p_customer customers)
RETURNS INTEGER AS $$
DECLARE
    v_id INTEGER;
BEGIN
    INSERT INTO customers (
        last_name, first_name, email, phone, city, address
    ) VALUES (
        p_customer.last_name,
        p_customer.first_name,
        p_customer.email,
        p_customer.phone,
        p_customer.city,
        p_customer.address
    )
    RETURNING customer_id INTO v_id;

    RETURN v_id;
END;
$$ LANGUAGE plpgsql;
Вызов:

sql
-- Создать запись и передать её как параметр
SELECT add_customer(ROW(
    'Смирнов', 'Пётр', 'smirnov@example.com',
    '+7-900-777-88-99', 'Москва', 'ул. Пушкина, д. 5'
)::customers);
```
##### 3.6. Массив составных типов
```sql
CREATE OR REPLACE FUNCTION get_books_array()
RETURNS book_info[] AS $$
DECLARE
    v_books book_info[];
BEGIN
    SELECT ARRAY_AGG(
        ROW(b.book_id, b.title, b.isbn, b.price, COALESCE(s.quantity, 0))::book_info
    )
    INTO v_books
    FROM books b
    LEFT JOIN stock s ON b.book_id = s.book_id;

    RETURN v_books;
END;
$$ LANGUAGE plpgsql;
```
Вызов:

```sql
-- Получить массив
SELECT get_books_array();

-- Развернуть массив в строки
SELECT * FROM UNNEST(get_books_array());
```
##### 3.7. Удаление составного типа
```sql
DROP TYPE IF EXISTS book_info CASCADE;
DROP TYPE IF EXISTS address_type CASCADE;
```
#### 4. Сравнение и рекомендации

Задача              | Инструмент
------------------- | -------------------------
Вернуть одно значение	 | Функция, возвращающая скаляр
Вернуть несколько полей	 | Функция с RETURNS TABLE или составным типом
Вернуть набор строк	 | Функция с RETURNS TABLE или RETURNS SETOF
Выполнить действие без возврата	 | Процедура
Управлять транзакциями	 | Процедура
Передать структуру данных	 | Составной тип
Хранить структуру в таблице	 | Составной тип

#### 5. Практический пример для «Книжного магазина»
Создание типа для отчёта:

```sql
CREATE TYPE order_report_row AS (
    order_id INTEGER,
    customer_name VARCHAR,
    order_date TIMESTAMP,
    items_count BIGINT,
    total_amount NUMERIC
);
```
Функция генерации отчёта:

```sql
CREATE OR REPLACE FUNCTION get_orders_report(
    p_from_date DATE DEFAULT '2000-01-01',
    p_to_date DATE DEFAULT CURRENT_DATE
)
RETURNS SETOF order_report_row AS $$
BEGIN
    RETURN QUERY
    SELECT
        o.order_id,
        c.last_name || ' ' || c.first_name,
        o.order_date,
        COUNT(oi.order_item_id),
        o.total_amount
    FROM orders o
    JOIN customers c ON o.customer_id = c.customer_id
    LEFT JOIN order_items oi ON o.order_id = oi.order_id
    WHERE o.order_date BETWEEN p_from_date AND p_to_date
      AND o.status != 'cancelled'
    GROUP BY o.order_id, c.last_name, c.first_name, o.order_date, o.total_amount
    ORDER BY o.order_date DESC;
END;
$$ LANGUAGE plpgsql;
```
Вызов:

```sql
SELECT * FROM get_orders_report('2024-01-01', '2024-12-31');
```
Процедура массового обновления цен:

```sql
CREATE OR REPLACE PROCEDURE apply_price_increase(
    p_category_id INTEGER,
    p_percent NUMERIC
)
LANGUAGE plpgsql
AS $$
DECLARE
    v_count INTEGER;
BEGIN
    UPDATE books
    SET price = price * (1 + p_percent / 100),
        cost_price = cost_price * (1 + p_percent / 100)
    WHERE category_id = p_category_id;

    GET DIAGNOSTICS v_count = ROW_COUNT;

    RAISE NOTICE 'Обновлено % книг в категории %', v_count, p_category_id;
    COMMIT;
END;
$$;
```
Вызов:

```sql
CALL apply_price_increase(2, 10);  -- +10% для классики
```
### PL/pgSQL 
Это мощный процедурный язык для PostgreSQL, позволяющий инкапсулировать сложную бизнес-логику непосредственно в базе данных. Вот обзор его основных возможностей.

#### Обзор и конструкции языка
*PL/pgSQL* — это загружаемый процедурный язык для PostgreSQL, который добавляет к SQL управляющие структуры и позволяет выполнять сложные вычисления. Его основная цель — создание функций, процедур и триггеров.

Структура блока: Код на PL/pgSQL организован в блоки следующей структуры:

```sql
[ <<метка>> ]
[ DECLARE
    -- Объявления переменных
]
BEGIN
    -- Исполняемые операторы
[ EXCEPTION
    -- Обработка ошибок
]
END;
```
Ключевые особенности:
- Регистро независимость: Язык не чувствителен к регистру, ключевые слова можно писать в любом регистре.
- Объявление переменных: Все переменные должны быть объявлены в секции DECLARE. Синтаксис: имя [ CONSTANT ] тип [ NOT NULL ] [ DEFAULT | := значение ];.
- Комментарии: Поддерживаются однострочные (--) и блочные (/* ... */) комментарии.

#### Выполнение запросов
- В PL/pgSQL есть несколько способов выполнения SQL-запросов.
- SELECT ... INTO: Используется для присвоения результата запроса (одной строки) переменным или записи. Если запрос не возвращает строк, переменным присваивается NULL. Опция STRICT вызывает ошибку, если запрос возвращает не одну строку.
- PERFORM: Используется, когда результат запроса нужно выполнить, но не сохранять. Это замена SELECT без INTO для вызова функций с побочными эффектами. Специальная переменная FOUND устанавливается в true, если запрос вернул хотя бы одну строку.
- Цикл FOR ... IN: Позволяет итерироваться по результатам запроса, автоматически открывая и закрывая курсор.

#### Курсоры
Курсоры позволяют обрабатывать результат запроса построчно, что полезно для больших выборок.

Типы курсоров: В PL/pgSQL все курсоры имеют тип refcursor. Они могут быть несвязанными (не привязаны к конкретному запросу) или связанными (привязаны к запросу при объявлении).

Объявление:

```sql
curs1 refcursor; -- Несвязанный
curs2 CURSOR FOR SELECT * FROM tenk1; -- Связанный
curs3 CURSOR (key integer) FOR SELECT * FROM tenk1 WHERE unique1 = key; -- Связанный с параметром
```
Использование: Курсор необходимо открыть (OPEN), затем можно получать строки (FETCH) или использовать его в цикле FOR.

#### Динамические команды
- Для выполнения SQL-команд, которые формируются во время выполнения (например, с динамическими именами таблиц), используется оператор EXECUTE.
- Синтаксис: EXECUTE командная_строка [ INTO [STRICT] цель ] [ USING выражение [, ...] ].
- Безопасность: Параметры, передаваемые через USING, безопасны от SQL-инъекций. Для динамических имён таблиц или столбцов их необходимо вставлять в строку запроса, используя функции quote_ident() и quote_literal().

#### Массивы
- PL/pgSQL полностью поддерживает работу с массивами PostgreSQL.
- Объявление: имя_переменной тип_элемента[];.
- Создание: Массивы можно создавать с помощью литералов ('{1,2,3}'), конструктора ARRAY[...] или агрегатной функции array_agg().
- Доступ: Элементы массива доступны по индексу, нумерация начинается с 1. Можно получать срезы: массив[нижний_индекс:верхний_индекс].

#### Обработка ошибок
Для перехвата и обработки ошибок используется блок EXCEPTION в структуре BEGIN ... END.

Синтаксис:

```sql
BEGIN
    -- Код, который может вызвать ошибку
EXCEPTION
    WHEN division_by_zero THEN
        RAISE NOTICE 'Деление на ноль!';
    WHEN OTHERS THEN
        RAISE NOTICE 'Произошла другая ошибка: %', SQLERRM;
END;
```
Специальные переменные: Внутри обработчика доступны SQLSTATE (код ошибки) и SQLERRM (текст сообщения). Можно также использовать GET STACKED DIAGNOSTICS для получения более детальной информации.

Важно: При перехвате ошибки все изменения в базе данных, сделанные внутри блока BEGIN...EXCEPTION, откатываются.

#### Триггеры
- PL/pgSQL позволяет создавать триггерные функции, которые автоматически вызываются при изменениях данных или событиях в БД.
- Создание: Триггерная функция создаётся с типом возврата trigger и не принимает аргументов.
- Специальные переменные: В триггерной функции автоматически доступны переменные NEW (новая строка), OLD (старая строка), TG_OP (тип операции) и другие.
- Возвращаемое значение: Для триггеров BEFORE возврат NULL отменяет операцию для текущей строки, а возврат NEW (возможно, изменённой) позволяет операции продолжиться.

#### Отладка
- Для отладки кода на PL/pgSQL существует несколько подходов.
- RAISE NOTICE: Самый простой способ — вставлять вызовы RAISE NOTICE для вывода значений переменных и отслеживания хода выполнения.
- Расширение plugin_debugger: Это полноценный интерактивный отладчик, позволяющий устанавливать точки останова, выполнять код пошагово и inspectровать переменные. Требует настройки shared_preload_libraries и создания расширения pldbgapi

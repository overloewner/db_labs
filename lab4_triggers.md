# Лабораторная работа №4: Триггеры в PostgreSQL

## Краткая последовательность команд

```sql
-- 1. Создание базы данных
CREATE DATABASE lab4_triggers;
\c lab4_triggers

-- 2. Создание основной таблицы
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    price DECIMAL(10,2) NOT NULL,
    stock INT NOT NULL DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 3. Создание таблицы аудита
CREATE TABLE products_audit (
    audit_id SERIAL PRIMARY KEY,
    operation CHAR(1) NOT NULL,  -- I, U, D
    product_id INT NOT NULL,
    old_data JSONB,
    new_data JSONB,
    changed_by VARCHAR(100),
    changed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 4. Создание триггерной функции BEFORE INSERT
CREATE OR REPLACE FUNCTION validate_product_before_insert()
RETURNS TRIGGER AS $$
BEGIN
    -- Проверка цены
    IF NEW.price <= 0 THEN
        RAISE EXCEPTION 'Цена должна быть положительной';
    END IF;

    -- Проверка количества
    IF NEW.stock < 0 THEN
        RAISE EXCEPTION 'Количество не может быть отрицательным';
    END IF;

    -- Преобразование имени к нормальному виду
    NEW.name := INITCAP(TRIM(NEW.name));

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- 5. Создание BEFORE INSERT триггера
CREATE TRIGGER trg_validate_product_before_insert
    BEFORE INSERT ON products
    FOR EACH ROW
    EXECUTE FUNCTION validate_product_before_insert();

-- 6. Тестирование BEFORE INSERT триггера
INSERT INTO products (name, price, stock)
VALUES ('laptop', 1000.00, 10);  -- name будет преобразовано в "Laptop"

-- Попытка вставить некорректные данные
INSERT INTO products (name, price, stock)
VALUES ('mouse', -50.00, 5);  -- ОШИБКА: цена отрицательная

-- 7. Создание триггерной функции BEFORE UPDATE
CREATE OR REPLACE FUNCTION update_timestamp()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at := CURRENT_TIMESTAMP;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- 8. Создание BEFORE UPDATE триггера
CREATE TRIGGER trg_update_timestamp
    BEFORE UPDATE ON products
    FOR EACH ROW
    EXECUTE FUNCTION update_timestamp();

-- 9. Тестирование BEFORE UPDATE триггера
UPDATE products SET price = 1100.00 WHERE name = 'Laptop';
SELECT * FROM products WHERE name = 'Laptop';  -- updated_at обновлен

-- 10. Создание триггерной функции AFTER для аудита
CREATE OR REPLACE FUNCTION audit_product_changes()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        INSERT INTO products_audit (operation, product_id, new_data, changed_by)
        VALUES ('I', NEW.id, row_to_json(NEW)::jsonb, current_user);
        RETURN NEW;
    ELSIF TG_OP = 'UPDATE' THEN
        INSERT INTO products_audit (operation, product_id, old_data, new_data, changed_by)
        VALUES ('U', NEW.id, row_to_json(OLD)::jsonb, row_to_json(NEW)::jsonb, current_user);
        RETURN NEW;
    ELSIF TG_OP = 'DELETE' THEN
        INSERT INTO products_audit (operation, product_id, old_data, changed_by)
        VALUES ('D', OLD.id, row_to_json(OLD)::jsonb, current_user);
        RETURN OLD;
    END IF;
END;
$$ LANGUAGE plpgsql;

-- 11. Создание AFTER триггеров
CREATE TRIGGER trg_audit_product_insert
    AFTER INSERT ON products
    FOR EACH ROW
    EXECUTE FUNCTION audit_product_changes();

CREATE TRIGGER trg_audit_product_update
    AFTER UPDATE ON products
    FOR EACH ROW
    EXECUTE FUNCTION audit_product_changes();

CREATE TRIGGER trg_audit_product_delete
    AFTER DELETE ON products
    FOR EACH ROW
    EXECUTE FUNCTION audit_product_changes();

-- 12. Тестирование AFTER триггеров
INSERT INTO products (name, price, stock)
VALUES ('Mouse', 25.00, 100);

UPDATE products SET price = 30.00 WHERE name = 'Mouse';

DELETE FROM products WHERE name = 'Mouse';

-- Просмотр аудита
SELECT * FROM products_audit ORDER BY changed_at;

-- 13. Создание таблицы заказов
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    product_id INT REFERENCES products(id),
    quantity INT NOT NULL,
    order_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 14. Триггерная функция для обновления складских остатков
CREATE OR REPLACE FUNCTION update_stock_on_order()
RETURNS TRIGGER AS $$
BEGIN
    -- Проверяем наличие товара на складе
    IF (SELECT stock FROM products WHERE id = NEW.product_id) < NEW.quantity THEN
        RAISE EXCEPTION 'Недостаточно товара на складе';
    END IF;

    -- Уменьшаем количество товара
    UPDATE products
    SET stock = stock - NEW.quantity
    WHERE id = NEW.product_id;

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- 15. Создание триггера для заказов
CREATE TRIGGER trg_update_stock_on_order
    AFTER INSERT ON orders
    FOR EACH ROW
    EXECUTE FUNCTION update_stock_on_order();

-- 16. Тестирование триггера заказов
INSERT INTO orders (product_id, quantity)
VALUES (1, 2);  -- Уменьшит stock на 2

SELECT * FROM products WHERE id = 1;  -- stock уменьшился

-- 17. Создание представления
CREATE VIEW product_summary AS
SELECT id, name, price, stock
FROM products;

-- 18. Триггерная функция INSTEAD OF для представления
CREATE OR REPLACE FUNCTION update_product_via_view()
RETURNS TRIGGER AS $$
BEGIN
    UPDATE products
    SET name = NEW.name,
        price = NEW.price,
        stock = NEW.stock
    WHERE id = NEW.id;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- 19. Создание INSTEAD OF триггера
CREATE TRIGGER trg_update_product_view
    INSTEAD OF UPDATE ON product_summary
    FOR EACH ROW
    EXECUTE FUNCTION update_product_via_view();

-- 20. Тестирование INSTEAD OF триггера
UPDATE product_summary
SET price = 1200.00
WHERE name = 'Laptop';

-- 21. Триггер с условием WHEN
CREATE OR REPLACE FUNCTION log_price_increase()
RETURNS TRIGGER AS $$
BEGIN
    RAISE NOTICE 'Цена товара % увеличена с % на %',
        NEW.name, OLD.price, NEW.price;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_log_price_increase
    BEFORE UPDATE ON products
    FOR EACH ROW
    WHEN (NEW.price > OLD.price)
    EXECUTE FUNCTION log_price_increase();

-- 22. Тестирование условного триггера
UPDATE products SET price = 1300.00 WHERE name = 'Laptop';  -- Триггер сработает
UPDATE products SET price = 1200.00 WHERE name = 'Laptop';  -- Триггер НЕ сработает

-- 23. Операторный триггер (STATEMENT-level)
CREATE OR REPLACE FUNCTION log_table_modification()
RETURNS TRIGGER AS $$
BEGIN
    RAISE NOTICE 'Выполнена операция % на таблице products', TG_OP;
    RETURN NULL;  -- Для STATEMENT триггеров значение игнорируется
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_log_table_modification
    AFTER INSERT OR UPDATE OR DELETE ON products
    FOR EACH STATEMENT
    EXECUTE FUNCTION log_table_modification();

-- 24. Тестирование операторного триггера
INSERT INTO products (name, price, stock)
VALUES ('Keyboard', 75.00, 50), ('Monitor', 300.00, 20);
-- Триггер сработает один раз, хотя вставлено две строки

-- 25. Просмотр триггеров
SELECT
    trigger_name,
    event_manipulation,
    event_object_table,
    action_timing,
    action_orientation
FROM information_schema.triggers
WHERE event_object_table = 'products'
ORDER BY trigger_name;

-- 26. Просмотр триггерных функций
\df+ validate_product_before_insert

-- 27. Отключение триггера
ALTER TABLE products DISABLE TRIGGER trg_validate_product_before_insert;

-- 28. Включение триггера
ALTER TABLE products ENABLE TRIGGER trg_validate_product_before_insert;

-- 29. Удаление триггера
DROP TRIGGER IF EXISTS trg_log_price_increase ON products;

-- 30. Удаление триггерной функции
DROP FUNCTION IF EXISTS log_price_increase();
```

---

## Подробное объяснение команд

### 1-2. Создание тестовой среды

```sql
CREATE DATABASE lab4_triggers;
\c lab4_triggers

CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    price DECIMAL(10,2) NOT NULL,
    stock INT NOT NULL DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**Что делает:** Создает базу данных и таблицу товаров для демонстрации триггеров.

**Поля таблицы:**
- `id` — уникальный идентификатор
- `name` — название товара
- `price` — цена
- `stock` — количество на складе
- `created_at`, `updated_at` — метки времени

---

### 3. Создание таблицы аудита

```sql
CREATE TABLE products_audit (
    audit_id SERIAL PRIMARY KEY,
    operation CHAR(1) NOT NULL,
    product_id INT NOT NULL,
    old_data JSONB,
    new_data JSONB,
    changed_by VARCHAR(100),
    changed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**Что делает:** Создает таблицу для хранения истории изменений.

**Объяснение полей:**
- `audit_id` — уникальный ID записи аудита
- `operation` — тип операции: 'I' (INSERT), 'U' (UPDATE), 'D' (DELETE)
- `product_id` — ID измененного товара
- `old_data` — состояние до изменения (JSONB для гибкости)
- `new_data` — состояние после изменения
- `changed_by` — пользователь, выполнивший изменение
- `changed_at` — время изменения

**Концепция:** Таблицы аудита — распространенная практика для отслеживания всех изменений данных в критичных системах.

---

### 4. Создание триггерной функции BEFORE INSERT

```sql
CREATE OR REPLACE FUNCTION validate_product_before_insert()
RETURNS TRIGGER AS $$
BEGIN
    IF NEW.price <= 0 THEN
        RAISE EXCEPTION 'Цена должна быть положительной';
    END IF;

    IF NEW.stock < 0 THEN
        RAISE EXCEPTION 'Количество не может быть отрицательным';
    END IF;

    NEW.name := INITCAP(TRIM(NEW.name));

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

**Что делает:** Создает функцию для валидации и модификации данных перед вставкой.

**Объяснение компонентов:**

**`RETURNS TRIGGER`** — функция возвращает специальный тип TRIGGER

**`$$...$$`** — разделители тела функции (альтернатива одинарным кавычкам)

**`NEW`** — специальная переменная, содержащая новую версию строки:
- Для INSERT: содержит вставляемую строку
- Для UPDATE: содержит обновленную версию
- Для DELETE: не определена

**`OLD`** — специальная переменная, содержащая старую версию строки:
- Для INSERT: не определена
- Для UPDATE: содержит версию до обновления
- Для DELETE: содержит удаляемую строку

**`RAISE EXCEPTION`** — выбрасывает ошибку и откатывает транзакцию

**`INITCAP()`** — преобразует первую букву каждого слова в верхний регистр

**`TRIM()`** — удаляет пробелы в начале и конце строки

**`RETURN NEW`** — возвращает измененную версию строки, которая будет вставлена

**Концепция:** BEFORE триггеры могут модифицировать данные перед их сохранением, реализуя валидацию и нормализацию на уровне базы данных.

---

### 5. Создание BEFORE INSERT триггера

```sql
CREATE TRIGGER trg_validate_product_before_insert
    BEFORE INSERT ON products
    FOR EACH ROW
    EXECUTE FUNCTION validate_product_before_insert();
```

**Что делает:** Связывает триггерную функцию с таблицей и событием.

**Объяснение компонентов:**
- `CREATE TRIGGER trg_validate_product_before_insert` — имя триггера (префикс `trg_` — соглашение)
- `BEFORE INSERT` — триггер срабатывает ДО вставки
- `ON products` — таблица, на которой создается триггер
- `FOR EACH ROW` — триггер срабатывает для каждой строки
- `EXECUTE FUNCTION validate_product_before_insert()` — вызываемая функция

**Альтернатива: FOR EACH STATEMENT** — триггер срабатывает один раз на весь SQL-оператор, независимо от количества затронутых строк.

**Концепция:** Триггер и триггерная функция — раздельные объекты. Одну функцию можно использовать в нескольких триггерах.

---

### 6. Тестирование BEFORE INSERT триггера

```sql
INSERT INTO products (name, price, stock)
VALUES ('laptop', 1000.00, 10);
```

**Результат:**
```
 id |  name  |  price  | stock |      created_at         |      updated_at
----+--------+---------+-------+-------------------------+-------------------------
  1 | Laptop | 1000.00 |    10 | 2026-01-15 10:30:00.123 | 2026-01-15 10:30:00.123
```

**Объяснение:** Имя 'laptop' было преобразовано в 'Laptop' триггерной функцией.

**Тест на ошибку:**
```sql
INSERT INTO products (name, price, stock)
VALUES ('mouse', -50.00, 5);
```

**Результат:**
```
ERROR:  Цена должна быть положительной
CONTEXT:  PL/pgSQL function validate_product_before_insert() line 3 at RAISE
```

**Концепция:** BEFORE триггеры могут предотвратить выполнение операции, выбросив исключение.

---

### 7-8. Автоматическое обновление метки времени

```sql
CREATE OR REPLACE FUNCTION update_timestamp()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at := CURRENT_TIMESTAMP;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_update_timestamp
    BEFORE UPDATE ON products
    FOR EACH ROW
    EXECUTE FUNCTION update_timestamp();
```

**Что делает:** Автоматически обновляет поле `updated_at` при изменении строки.

**Тест:**
```sql
UPDATE products SET price = 1100.00 WHERE name = 'Laptop';
SELECT updated_at FROM products WHERE name = 'Laptop';
```

**Результат:**
```
      updated_at
-------------------------
 2026-01-15 10:35:42.789
```

**Концепция:** Распространенный паттерн для отслеживания времени последнего изменения записи.

---

### 10-11. Триггерная функция для аудита (AFTER)

```sql
CREATE OR REPLACE FUNCTION audit_product_changes()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        INSERT INTO products_audit (operation, product_id, new_data, changed_by)
        VALUES ('I', NEW.id, row_to_json(NEW)::jsonb, current_user);
        RETURN NEW;
    ELSIF TG_OP = 'UPDATE' THEN
        INSERT INTO products_audit (operation, product_id, old_data, new_data, changed_by)
        VALUES ('U', NEW.id, row_to_json(OLD)::jsonb, row_to_json(NEW)::jsonb, current_user);
        RETURN NEW;
    ELSIF TG_OP = 'DELETE' THEN
        INSERT INTO products_audit (operation, product_id, old_data, changed_by)
        VALUES ('D', OLD.id, row_to_json(OLD)::jsonb, current_user);
        RETURN OLD;
    END IF;
END;
$$ LANGUAGE plpgsql;
```

**Что делает:** Записывает все изменения (INSERT, UPDATE, DELETE) в таблицу аудита.

**Специальные переменные триггера:**
- `TG_OP` — тип операции ('INSERT', 'UPDATE', 'DELETE', 'TRUNCATE')
- `TG_NAME` — имя триггера
- `TG_TABLE_NAME` — имя таблицы
- `TG_TABLE_SCHEMA` — схема таблицы
- `TG_WHEN` — 'BEFORE' или 'AFTER'
- `TG_LEVEL` — 'ROW' или 'STATEMENT'

**Функции преобразования:**
- `row_to_json(record)` — преобразует строку PostgreSQL в JSON
- `::jsonb` — приводит тип к JSONB для эффективного хранения

**`current_user`** — текущий пользователь PostgreSQL

**Концепция:** AFTER триггеры срабатывают после успешного выполнения операции, идеальны для логирования и каскадных изменений.

---

### 12. Тестирование AFTER триггеров

```sql
INSERT INTO products (name, price, stock)
VALUES ('Mouse', 25.00, 100);

UPDATE products SET price = 30.00 WHERE name = 'Mouse';

DELETE FROM products WHERE name = 'Mouse';

SELECT * FROM products_audit ORDER BY changed_at;
```

**Вывод:**
```
 audit_id | operation | product_id | old_data | new_data | changed_by | changed_at
----------+-----------+------------+----------+----------+------------+-------------------------
        1 | I         |          2 |          | {...}    | postgres   | 2026-01-15 10:40:00.123
        2 | U         |          2 | {...}    | {...}    | postgres   | 2026-01-15 10:40:15.456
        3 | D         |          2 | {...}    |          | postgres   | 2026-01-15 10:40:30.789
```

**Объяснение колонок:**
- `operation`: 'I' (INSERT), 'U' (UPDATE), 'D' (DELETE)
- `old_data`, `new_data`: JSONB с полным состоянием строки

**Просмотр JSONB данных:**
```sql
SELECT
    audit_id,
    operation,
    new_data->>'name' AS product_name,
    new_data->>'price' AS new_price,
    old_data->>'price' AS old_price
FROM products_audit
WHERE operation = 'U';
```

**Концепция:** JSONB позволяет гибко хранить исторические данные без изменения схемы таблицы аудита при добавлении новых полей.

---

### 14-15. Каскадное обновление складских остатков

```sql
CREATE OR REPLACE FUNCTION update_stock_on_order()
RETURNS TRIGGER AS $$
BEGIN
    IF (SELECT stock FROM products WHERE id = NEW.product_id) < NEW.quantity THEN
        RAISE EXCEPTION 'Недостаточно товара на складе';
    END IF;

    UPDATE products
    SET stock = stock - NEW.quantity
    WHERE id = NEW.product_id;

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_update_stock_on_order
    AFTER INSERT ON orders
    FOR EACH ROW
    EXECUTE FUNCTION update_stock_on_order();
```

**Что делает:** Автоматически уменьшает количество товара на складе при создании заказа.

**Тест:**
```sql
-- До заказа
SELECT stock FROM products WHERE id = 1;  -- stock = 10

INSERT INTO orders (product_id, quantity) VALUES (1, 2);

-- После заказа
SELECT stock FROM products WHERE id = 1;  -- stock = 8
```

**Попытка заказать больше, чем есть на складе:**
```sql
INSERT INTO orders (product_id, quantity) VALUES (1, 100);
-- ERROR:  Недостаточно товара на складе
```

**Концепция:** Триггеры могут реализовывать сложную бизнес-логику и каскадные изменения в нескольких таблицах, обеспечивая целостность данных.

---

### 18-19. INSTEAD OF триггер для представлений

```sql
CREATE VIEW product_summary AS
SELECT id, name, price, stock
FROM products;

CREATE OR REPLACE FUNCTION update_product_via_view()
RETURNS TRIGGER AS $$
BEGIN
    UPDATE products
    SET name = NEW.name,
        price = NEW.price,
        stock = NEW.stock
    WHERE id = NEW.id;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_update_product_view
    INSTEAD OF UPDATE ON product_summary
    FOR EACH ROW
    EXECUTE FUNCTION update_product_via_view();
```

**Что делает:** Позволяет обновлять данные через представление.

**Объяснение:**
- По умолчанию представления только для чтения
- INSTEAD OF триггер выполняется ВМЕСТО операции над представлением
- Триггер выполняет реальную операцию над базовой таблицей

**Тест:**
```sql
UPDATE product_summary SET price = 1200.00 WHERE name = 'Laptop';
-- Обновляется базовая таблица products
```

**Применение INSTEAD OF:**
- Простые представления: PostgreSQL автоматически делает их обновляемыми
- Сложные представления (JOIN, GROUP BY, DISTINCT): требуют INSTEAD OF триггеров
- Создание упрощенных интерфейсов для сложных схем данных

**Концепция:** INSTEAD OF триггеры применяются только к представлениям и всегда на уровне строк (FOR EACH ROW).

---

### 21. Условный триггер (WHEN)

```sql
CREATE OR REPLACE FUNCTION log_price_increase()
RETURNS TRIGGER AS $$
BEGIN
    RAISE NOTICE 'Цена товара % увеличена с % на %',
        NEW.name, OLD.price, NEW.price;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_log_price_increase
    BEFORE UPDATE ON products
    FOR EACH ROW
    WHEN (NEW.price > OLD.price)
    EXECUTE FUNCTION log_price_increase();
```

**Что делает:** Триггер срабатывает только когда цена увеличивается.

**Объяснение WHEN:**
- `WHEN (condition)` — условие, которое должно быть истинным для срабатывания триггера
- Условие проверяется перед вызовом функции
- Для BEFORE триггеров: проверяется перед вызовом функции
- Для AFTER триггеров: событие запоминается только если условие истинно (оптимизация)

**Тест:**
```sql
UPDATE products SET price = 1300.00 WHERE name = 'Laptop';
-- NOTICE:  Цена товара Laptop увеличена с 1200.00 на 1300.00

UPDATE products SET price = 1200.00 WHERE name = 'Laptop';
-- Триггер не срабатывает (цена уменьшилась)
```

**Концепция:** WHEN позволяет оптимизировать триггеры, избегая ненужных вызовов функций.

---

### 23. Операторный триггер (STATEMENT-level)

```sql
CREATE OR REPLACE FUNCTION log_table_modification()
RETURNS TRIGGER AS $$
BEGIN
    RAISE NOTICE 'Выполнена операция % на таблице products', TG_OP;
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_log_table_modification
    AFTER INSERT OR UPDATE OR DELETE ON products
    FOR EACH STATEMENT
    EXECUTE FUNCTION log_table_modification();
```

**Что делает:** Триггер срабатывает один раз на весь SQL-оператор.

**Отличие FOR EACH STATEMENT от FOR EACH ROW:**

**FOR EACH ROW:**
- Срабатывает для каждой затронутой строки
- Доступны NEW и OLD
- Может изменить данные (в BEFORE)
- Пример: UPDATE products SET price = price * 1.1 (3 строки) → триггер сработает 3 раза

**FOR EACH STATEMENT:**
- Срабатывает один раз на оператор
- NEW и OLD недоступны
- Не может изменить данные
- Тот же UPDATE → триггер сработает 1 раз

**Тест:**
```sql
INSERT INTO products (name, price, stock)
VALUES ('Keyboard', 75.00, 50), ('Monitor', 300.00, 20);
-- NOTICE:  Выполнена операция INSERT на таблице products
-- (только одно сообщение, хотя вставлено 2 строки)
```

**Применение STATEMENT-level триггеров:**
- Логирование на уровне операций
- Проверка прав доступа
- Мониторинг активности
- Предотвращение операций (в BEFORE)

**Концепция:** Операторные триггеры эффективнее для операций, затрагивающих много строк, так как срабатывают один раз.

---

### 25. Просмотр триггеров

```sql
SELECT
    trigger_name,
    event_manipulation,
    event_object_table,
    action_timing,
    action_orientation
FROM information_schema.triggers
WHERE event_object_table = 'products'
ORDER BY trigger_name;
```

**Вывод (пример):**
```
           trigger_name           | event_manipulation | event_object_table | action_timing | action_orientation
----------------------------------+--------------------+--------------------+---------------+--------------------
 trg_audit_product_delete         | DELETE             | products           | AFTER         | ROW
 trg_audit_product_insert         | INSERT             | products           | AFTER         | ROW
 trg_audit_product_update         | UPDATE             | products           | AFTER         | ROW
 trg_log_table_modification       | INSERT             | products           | AFTER         | STATEMENT
 trg_update_timestamp             | UPDATE             | products           | BEFORE        | ROW
 trg_validate_product_before_insert | INSERT           | products           | BEFORE        | ROW
```

**Объяснение колонок:**
- `trigger_name` — имя триггера
- `event_manipulation` — событие (INSERT, UPDATE, DELETE)
- `event_object_table` — таблица
- `action_timing` — момент срабатывания (BEFORE, AFTER, INSTEAD OF)
- `action_orientation` — уровень (ROW, STATEMENT)

**Альтернативный запрос (более подробный):**
```sql
SELECT * FROM pg_trigger WHERE tgrelid = 'products'::regclass;
```

**Концепция:** information_schema — стандартизированный способ получения метаданных, pg_catalog — специфичный для PostgreSQL.

---

### 27-28. Управление триггерами

```sql
-- Отключение триггера
ALTER TABLE products DISABLE TRIGGER trg_validate_product_before_insert;

-- Включение триггера
ALTER TABLE products ENABLE TRIGGER trg_validate_product_before_insert;

-- Отключение всех триггеров на таблице
ALTER TABLE products DISABLE TRIGGER ALL;

-- Включение всех триггеров
ALTER TABLE products ENABLE TRIGGER ALL;

-- Отключение только пользовательских триггеров (не системных)
ALTER TABLE products DISABLE TRIGGER USER;
```

**Применение:**
- Временное отключение для массовой загрузки данных
- Отладка
- Обслуживание

**ВАЖНО:** Отключение триггеров может нарушить целостность данных. Используйте с осторожностью.

---

### 29-30. Удаление триггеров и функций

```sql
DROP TRIGGER IF EXISTS trg_log_price_increase ON products;
DROP FUNCTION IF EXISTS log_price_increase();
```

**Что делает:** Удаляет триггер и его функцию.

**Порядок удаления:**
1. Сначала удалить триггер
2. Затем удалить функцию (если она не используется другими триггерами)

**IF EXISTS** — предотвращает ошибку, если объект не существует.

**Альтернатива (удаление функции со всеми зависимостями):**
```sql
DROP FUNCTION log_price_increase() CASCADE;
-- Автоматически удалит все триггеры, использующие эту функцию
```

---

## Технический концепт: Триггеры в PostgreSQL

### Архитектура триггеров

**Триггер состоит из двух компонентов:**

1. **Определение триггера** (CREATE TRIGGER):
   - Указывает событие (INSERT, UPDATE, DELETE, TRUNCATE)
   - Указывает момент (BEFORE, AFTER, INSTEAD OF)
   - Указывает уровень (ROW, STATEMENT)
   - Связывает с таблицей

2. **Триггерная функция** (CREATE FUNCTION):
   - Содержит логику
   - Может быть переиспользована
   - Должна возвращать TRIGGER
   - Имеет доступ к специальным переменным

### Порядок выполнения триггеров

При выполнении UPDATE products SET price = 100 WHERE id = 1:

1. **BEFORE STATEMENT** триггеры
2. Для каждой строки:
   - **BEFORE ROW** триггеры (в алфавитном порядке имен)
   - Выполнение операции
   - **AFTER ROW** триггеры (в алфавитном порядке имен)
3. **AFTER STATEMENT** триггеры

**ВАЖНО:** Порядок выполнения триггеров одного типа — алфавитный по имени. Используйте префиксы для контроля порядка:
```sql
CREATE TRIGGER a_first_trigger ...
CREATE TRIGGER b_second_trigger ...
CREATE TRIGGER z_last_trigger ...
```

### Возвращаемые значения

**BEFORE ROW триггеры:**
- `RETURN NEW` — операция продолжится с (возможно измененной) строкой
- `RETURN OLD` — для DELETE, использовать исходную версию
- `RETURN NULL` — операция для этой строки отменяется (не откат транзакции!)

**AFTER ROW триггеры:**
- Возвращаемое значение игнорируется, но обязательно
- Обычно `RETURN NEW` или `RETURN OLD`

**STATEMENT триггеры:**
- Возвращаемое значение игнорируется
- Обычно `RETURN NULL`

### Специальные переменные в триггерах

```plpgsql
-- Информация о триггере
TG_NAME         -- имя триггера
TG_WHEN         -- 'BEFORE', 'AFTER', 'INSTEAD OF'
TG_LEVEL        -- 'ROW' или 'STATEMENT'
TG_OP           -- 'INSERT', 'UPDATE', 'DELETE', 'TRUNCATE'
TG_RELID        -- OID таблицы
TG_TABLE_NAME   -- имя таблицы
TG_TABLE_SCHEMA -- схема таблицы
TG_NARGS        -- количество аргументов
TG_ARGV[]       -- массив аргументов

-- Данные строки
NEW             -- новая версия строки (INSERT, UPDATE)
OLD             -- старая версия строки (UPDATE, DELETE)
```

### Производительность триггеров

**Плюсы:**
- Гарантируют целостность данных на уровне БД
- Прозрачны для приложений
- Централизованная логика

**Минусы:**
- Накладные расходы на каждую операцию
- Могут значительно замедлить массовые операции
- Сложность отладки
- "Скрытая" логика, невидимая в SQL-запросах

**Оптимизация:**
- Используйте WHEN для фильтрации ненужных срабатываний
- Используйте STATEMENT-level триггеры где возможно
- Отключайте триггеры при массовой загрузке данных
- Избегайте сложной логики в триггерах

### Ограничения триггеров

**Нельзя создать триггер на:**
- Системные каталоги
- Представления (кроме INSTEAD OF)
- Партиционированные таблицы (только на партиции)

**Нельзя использовать:**
- Команды управления транзакциями (COMMIT, ROLLBACK) внутри триггерной функции
- Прямые возвращаемые значения (вместо RETURN)

**Рекурсия:**
- Триггер может вызвать себя через модификацию той же таблицы
- PostgreSQL не имеет встроенной защиты от бесконечной рекурсии
- Используйте условия для предотвращения рекурсии

### Практические паттерны

**1. Аудит изменений:**
```sql
-- Универсальная функция аудита
CREATE FUNCTION audit_trigger() RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO audit_log (table_name, operation, old_data, new_data, user_name, timestamp)
    VALUES (TG_TABLE_NAME, TG_OP, row_to_json(OLD), row_to_json(NEW), current_user, now());
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

**2. Автоматическая генерация значений:**
```sql
-- Генерация slug из названия
CREATE FUNCTION generate_slug() RETURNS TRIGGER AS $$
BEGIN
    NEW.slug := lower(regexp_replace(NEW.title, '[^a-zA-Z0-9]+', '-', 'g'));
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

**3. Валидация бизнес-правил:**
```sql
-- Проверка диапазона дат
CREATE FUNCTION validate_date_range() RETURNS TRIGGER AS $$
BEGIN
    IF NEW.end_date < NEW.start_date THEN
        RAISE EXCEPTION 'end_date не может быть раньше start_date';
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

**4. Каскадные операции:**
```sql
-- Автоматическое удаление связанных записей
CREATE FUNCTION cascade_delete() RETURNS TRIGGER AS $$
BEGIN
    DELETE FROM related_table WHERE parent_id = OLD.id;
    RETURN OLD;
END;
$$ LANGUAGE plpgsql;
```

**5. Денормализация для производительности:**
```sql
-- Обновление кэшированных счетчиков
CREATE FUNCTION update_post_count() RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        UPDATE users SET post_count = post_count + 1 WHERE id = NEW.user_id;
    ELSIF TG_OP = 'DELETE' THEN
        UPDATE users SET post_count = post_count - 1 WHERE id = OLD.user_id;
    END IF;
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;
```

### Альтернативы триггерам

**Когда НЕ использовать триггеры:**

1. **Простая валидация** → CHECK constraints
2. **Значения по умолчанию** → DEFAULT values
3. **Связи между таблицами** → Foreign keys with CASCADE
4. **Сложная бизнес-логика** → Application code
5. **Асинхронные операции** → Background jobs, очереди

**Правило:** Используйте триггеры для логики, которая ДОЛЖНА выполняться на уровне БД для обеспечения целостности данных.

# Лабораторная работа 4: Триггеры в PostgreSQL

## Задание 1: Система журналирования обращений к таблице (DML-триггеры)

**Что требуется:**
Реализовать в PostgreSQL систему журналирования обращений к таблице. Создать универсальную триггерную функцию, которая будет логировать операции UPDATE, INSERT, DELETE в отдельную таблицу аудита. Система должна учитывать: кто выполнил операцию, время, тип операции, целевую таблицу, схему, старую и новую версии строки. Таблица с логами должна находиться в отдельной схеме. Доступ к таблицам логов должен быть только у администратора (не суперпользователь). Для логов использовать формат JSON. Реализовать цифровую подпись логов (с помощью pgcrypto). Хранить хэши записей для обнаружения подмен.

---

### Шаг 1: Подготовка окружения и расширений

#### Последовательность команд

```sql
-- В psql от имени postgres (суперпользователь)

-- Устанавливаем расширение для криптографии
CREATE EXTENSION IF NOT EXISTS pgcrypto;

-- Создаем отдельную схему для логов аудита
CREATE SCHEMA IF NOT EXISTS audit;

-- Создаем роль администратора (не суперпользователь)
CREATE ROLE tno_admin WITH LOGIN PASSWORD 'secure_password';

-- Даем администратору права на использование схемы audit
GRANT USAGE ON SCHEMA audit TO tno_admin;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA audit TO tno_admin;
GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA audit TO tno_admin;

-- Устанавливаем права по умолчанию для будущих объектов в схеме audit
ALTER DEFAULT PRIVILEGES IN SCHEMA audit GRANT ALL ON TABLES TO tno_admin;
ALTER DEFAULT PRIVILEGES IN SCHEMA audit GRANT ALL ON SEQUENCES TO tno_admin;
```

#### Детальное объяснение каждой команды

1. **`CREATE EXTENSION IF NOT EXISTS pgcrypto;`**
   - Устанавливает расширение pgcrypto в базу данных
   - `pgcrypto` предоставляет криптографические функции: хеширование (MD5, SHA1, SHA256), шифрование, генерацию случайных данных
   - `IF NOT EXISTS` - не вызывает ошибку, если расширение уже установлено
   - Используется для создания цифровых подписей логов

2. **`CREATE SCHEMA IF NOT EXISTS audit;`**
   - Создает отдельную схему (namespace) для хранения таблиц аудита
   - Схема - это логический контейнер для объектов БД (таблицы, функции, представления)
   - Изоляция данных аудита от основных данных
   - `IF NOT EXISTS` - предотвращает ошибку при повторном выполнении

3. **`CREATE ROLE tno_admin WITH LOGIN PASSWORD 'secure_password';`**
   - Создает новую роль (пользователя) с именем tno_admin
   - `WITH LOGIN` - позволяет роли входить в систему (без этого это просто группа)
   - `PASSWORD` - устанавливает пароль для аутентификации
   - Это НЕ суперпользователь - имеет ограниченные права только на схему audit

4. **`GRANT USAGE ON SCHEMA audit TO tno_admin;`**
   - Дает право использовать схему audit (видеть объекты в ней)
   - `USAGE` - базовая привилегия для доступа к схеме
   - Без USAGE пользователь не сможет обращаться к объектам схемы

5. **`GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA audit TO tno_admin;`**
   - Дает все права (SELECT, INSERT, UPDATE, DELETE, TRUNCATE, REFERENCES, TRIGGER) на все существующие таблицы в схеме audit
   - `ALL PRIVILEGES` включает: SELECT, INSERT, UPDATE, DELETE, TRUNCATE, REFERENCES, TRIGGER
   - Применяется только к уже существующим таблицам

6. **`GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA audit TO tno_admin;`**
   - Дает права на последовательности (sequences) в схеме audit
   - Последовательности используются для SERIAL/BIGSERIAL полей (автоинкремент)
   - Необходимо для вставки данных в таблицы с SERIAL полями

7. **`ALTER DEFAULT PRIVILEGES IN SCHEMA audit GRANT ALL ON TABLES TO tno_admin;`**
   - Устанавливает права по умолчанию для БУДУЩИХ таблиц в схеме audit
   - Когда postgres создаст новую таблицу в audit, tno_admin автоматически получит все права
   - Без этой команды пришлось бы явно давать права после создания каждой таблицы

#### Проверка установленных прав

```sql
-- Проверяем список ролей и их атрибуты
\du

-- Проверяем список схем и права на них
\dn+

-- Проверяем права tno_admin на схему audit
SELECT
    nspname AS schema_name,
    pg_catalog.has_schema_privilege('tno_admin', nspname, 'USAGE') AS has_usage,
    pg_catalog.has_schema_privilege('tno_admin', nspname, 'CREATE') AS has_create
FROM pg_catalog.pg_namespace
WHERE nspname = 'audit';
```

#### Объяснение результатов

**Результат \du:**
```
                                   List of roles
 Role name  |                         Attributes                         | Member of
------------+------------------------------------------------------------+-----------
 postgres   | Superuser, Create role, Create DB, Replication, Bypass RLS| {}
 tno_admin  |                                                            | {}
```

**Результат проверки прав:**
```
 schema_name | has_usage | has_create
-------------+-----------+------------
 audit       | t         | f
```

**Объяснение колонок:**
- `has_usage` = t (true) - tno_admin может использовать схему audit
- `has_create` = f (false) - tno_admin не может создавать новые объекты в схеме (это делает postgres)

---

### Шаг 2: Создание таблицы аудита с цифровой подписью

#### Последовательность команд

```sql
-- Создаем таблицу для хранения логов аудита
CREATE TABLE audit.data_audit_log (
    log_id BIGSERIAL PRIMARY KEY,
    table_schema VARCHAR(100) NOT NULL,
    table_name VARCHAR(100) NOT NULL,
    operation VARCHAR(10) NOT NULL,
    old_data JSONB,
    new_data JSONB,
    changed_by VARCHAR(100) NOT NULL,
    changed_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    client_address INET,
    application_name VARCHAR(200),
    transaction_id BIGINT,
    log_hash VARCHAR(64) NOT NULL,
    CONSTRAINT check_operation CHECK (operation IN ('INSERT', 'UPDATE', 'DELETE'))
);

-- Создаем индексы для быстрого поиска
CREATE INDEX idx_audit_table ON audit.data_audit_log(table_schema, table_name);
CREATE INDEX idx_audit_user ON audit.data_audit_log(changed_by);
CREATE INDEX idx_audit_timestamp ON audit.data_audit_log(changed_at);
CREATE INDEX idx_audit_operation ON audit.data_audit_log(operation);

-- Запрещаем UPDATE и DELETE в таблице аудита (только INSERT)
CREATE OR REPLACE FUNCTION audit.protect_audit_log()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'UPDATE' OR TG_OP = 'DELETE' THEN
        RAISE EXCEPTION 'Изменение и удаление логов аудита запрещено!';
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_protect_audit_log
    BEFORE UPDATE OR DELETE ON audit.data_audit_log
    FOR EACH ROW
    EXECUTE FUNCTION audit.protect_audit_log();

-- Комментарии для документирования
COMMENT ON TABLE audit.data_audit_log IS 'Таблица для хранения логов аудита всех изменений данных';
COMMENT ON COLUMN audit.data_audit_log.log_hash IS 'SHA256 хеш для обнаружения подмены записей';
```

#### Детальное объяснение каждой команды

1. **`CREATE TABLE audit.data_audit_log (...)`**
   - Создает таблицу в схеме audit для хранения всех логов
   - `BIGSERIAL PRIMARY KEY` - автоинкрементный идентификатор (64-битный, до 9 квинтиллионов записей)

2. **Поля таблицы:**
   - `log_id` - уникальный идентификатор записи лога
   - `table_schema` - схема таблицы, в которой произошло изменение (VARCHAR(100))
   - `table_name` - имя таблицы, в которой произошло изменение
   - `operation` - тип операции: INSERT, UPDATE или DELETE
   - `old_data` - старое состояние строки в формате JSONB (NULL для INSERT)
   - `new_data` - новое состояние строки в формате JSONB (NULL для DELETE)
   - `changed_by` - имя пользователя, выполнившего операцию (current_user)
   - `changed_at` - метка времени изменения (автоматически устанавливается)
   - `client_address` - IP-адрес клиента (inet_client_addr())
   - `application_name` - имя приложения, выполнившего запрос
   - `transaction_id` - ID транзакции PostgreSQL (txid_current())
   - `log_hash` - SHA256 хеш записи для обнаружения подмен

3. **`CONSTRAINT check_operation CHECK (operation IN ('INSERT', 'UPDATE', 'DELETE'))`**
   - Ограничение CHECK, проверяющее, что operation содержит только допустимые значения
   - Предотвращает вставку некорректных данных на уровне БД

4. **Индексы:**
   - `idx_audit_table` - ускоряет поиск по схеме и таблице (WHERE table_name = '...')
   - `idx_audit_user` - ускоряет поиск по пользователю (WHERE changed_by = '...')
   - `idx_audit_timestamp` - ускоряет поиск и сортировку по времени (ORDER BY changed_at, WHERE changed_at BETWEEN ...)
   - `idx_audit_operation` - ускоряет фильтрацию по типу операции (WHERE operation = 'UPDATE')

5. **`CREATE OR REPLACE FUNCTION audit.protect_audit_log()`**
   - Создает защитную функцию, запрещающую изменение и удаление логов
   - Триггер BEFORE UPDATE OR DELETE вызывает эту функцию
   - `RAISE EXCEPTION` откатывает транзакцию и выбрасывает ошибку
   - Обеспечивает неизменяемость (immutability) логов аудита

6. **`COMMENT ON TABLE/COLUMN`**
   - Добавляет описания к таблице и колонкам
   - Документация хранится в самой БД и видна в psql (\d+ table_name) и GUI-инструментах

#### Проверка структуры таблицы

```sql
-- Просмотр структуры таблицы
\d+ audit.data_audit_log

-- Просмотр индексов
SELECT
    indexname,
    indexdef
FROM pg_indexes
WHERE schemaname = 'audit' AND tablename = 'data_audit_log';
```

---

### Шаг 3: Универсальная триггерная функция для аудита

#### Последовательность команд

```sql
-- Создаем универсальную функцию аудита
CREATE OR REPLACE FUNCTION audit.universal_audit_trigger()
RETURNS TRIGGER AS $$
DECLARE
    v_old_data JSONB;
    v_new_data JSONB;
    v_log_data TEXT;
    v_hash VARCHAR(64);
BEGIN
    -- Формируем JSON для старых и новых данных
    IF TG_OP = 'DELETE' THEN
        v_old_data := row_to_json(OLD)::JSONB;
        v_new_data := NULL;
    ELSIF TG_OP = 'INSERT' THEN
        v_old_data := NULL;
        v_new_data := row_to_json(NEW)::JSONB;
    ELSIF TG_OP = 'UPDATE' THEN
        v_old_data := row_to_json(OLD)::JSONB;
        v_new_data := row_to_json(NEW)::JSONB;
    END IF;

    -- Формируем строку для хеширования
    v_log_data := TG_TABLE_SCHEMA || '.' || TG_TABLE_NAME || '|' ||
                  TG_OP || '|' ||
                  COALESCE(v_old_data::TEXT, 'NULL') || '|' ||
                  COALESCE(v_new_data::TEXT, 'NULL') || '|' ||
                  current_user || '|' ||
                  CURRENT_TIMESTAMP::TEXT || '|' ||
                  txid_current()::TEXT;

    -- Вычисляем SHA256 хеш для цифровой подписи
    v_hash := encode(digest(v_log_data, 'sha256'), 'hex');

    -- Вставляем запись в таблицу аудита
    INSERT INTO audit.data_audit_log (
        table_schema,
        table_name,
        operation,
        old_data,
        new_data,
        changed_by,
        changed_at,
        client_address,
        application_name,
        transaction_id,
        log_hash
    ) VALUES (
        TG_TABLE_SCHEMA,
        TG_TABLE_NAME,
        TG_OP,
        v_old_data,
        v_new_data,
        current_user,
        CURRENT_TIMESTAMP,
        inet_client_addr(),
        current_setting('application_name', true),
        txid_current(),
        v_hash
    );

    -- Возвращаем правильное значение в зависимости от операции
    IF TG_OP = 'DELETE' THEN
        RETURN OLD;
    ELSE
        RETURN NEW;
    END IF;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

-- Комментарий к функции
COMMENT ON FUNCTION audit.universal_audit_trigger() IS 'Универсальная функция аудита для логирования INSERT/UPDATE/DELETE операций';
```

#### Детальное объяснение каждой команды

1. **`DECLARE` секция:**
   - `v_old_data JSONB` - переменная для хранения старого состояния строки
   - `v_new_data JSONB` - переменная для нового состояния
   - `v_log_data TEXT` - строка для формирования хеша
   - `v_hash VARCHAR(64)` - SHA256 хеш (64 шестнадцатеричных символа = 256 бит)

2. **Блок формирования JSON:**
   - `row_to_json(OLD)` - преобразует запись PostgreSQL в JSON объект
   - `::JSONB` - приведение типа к JSONB (бинарный JSON, поддерживает индексацию)
   - Для DELETE: old_data заполнен, new_data = NULL
   - Для INSERT: old_data = NULL, new_data заполнен
   - Для UPDATE: оба заполнены (можно сравнить что изменилось)

3. **Формирование строки для хеширования:**
   ```
   schema.table|OPERATION|old_json|new_json|user|timestamp|txid
   ```
   - `||` - оператор конкатенации строк в PostgreSQL
   - `COALESCE(value, 'NULL')` - заменяет NULL на строку 'NULL'
   - Включает все ключевые данные для гарантии целостности

4. **`encode(digest(v_log_data, 'sha256'), 'hex')`**
   - `digest(data, 'sha256')` - вычисляет SHA256 хеш (возвращает bytea)
   - `encode(bytea, 'hex')` - конвертирует байты в шестнадцатеричную строку
   - SHA256 - криптографическая хеш-функция (256 бит, практически невозможно подделать)

5. **Специальные переменные триггера:**
   - `TG_TABLE_SCHEMA` - схема таблицы, на которой сработал триггер
   - `TG_TABLE_NAME` - имя таблицы
   - `TG_OP` - тип операции ('INSERT', 'UPDATE', 'DELETE')

6. **Системные функции PostgreSQL:**
   - `current_user` - имя текущего пользователя PostgreSQL
   - `CURRENT_TIMESTAMP` - текущая метка времени
   - `inet_client_addr()` - IP-адрес клиента (NULL для локальных подключений)
   - `current_setting('application_name', true)` - имя приложения (из настроек подключения)
   - `txid_current()` - ID текущей транзакции (уникальный 64-битный номер)

7. **`SECURITY DEFINER`**
   - Функция выполняется с правами владельца (postgres), а не вызывающего пользователя
   - Необходимо, чтобы обычные пользователи могли писать в audit.data_audit_log
   - Без этого пользователю нужны были бы права INSERT на таблицу аудита

8. **Возвращаемое значение:**
   - Для DELETE: `RETURN OLD` (так как NEW не определен)
   - Для INSERT/UPDATE: `RETURN NEW`
   - AFTER триггеры игнорируют возвращаемое значение, но его нужно вернуть

#### Концептуальное объяснение

**Цифровая подпись (log_hash):**
Хеш SHA256 служит цифровой подписью записи. Если кто-то попытается изменить данные в таблице аудита напрямую (в обход защитного триггера, например через pg_dump/restore), хеш не совпадет с данными. Можно написать функцию проверки целостности:

```sql
-- Функция для проверки целостности логов
CREATE OR REPLACE FUNCTION audit.verify_log_integrity(p_log_id BIGINT)
RETURNS BOOLEAN AS $$
DECLARE
    v_record RECORD;
    v_computed_hash VARCHAR(64);
    v_log_data TEXT;
BEGIN
    SELECT * INTO v_record FROM audit.data_audit_log WHERE log_id = p_log_id;

    IF NOT FOUND THEN
        RAISE EXCEPTION 'Запись с log_id = % не найдена', p_log_id;
    END IF;

    v_log_data := v_record.table_schema || '.' || v_record.table_name || '|' ||
                  v_record.operation || '|' ||
                  COALESCE(v_record.old_data::TEXT, 'NULL') || '|' ||
                  COALESCE(v_record.new_data::TEXT, 'NULL') || '|' ||
                  v_record.changed_by || '|' ||
                  v_record.changed_at::TEXT || '|' ||
                  v_record.transaction_id::TEXT;

    v_computed_hash := encode(digest(v_log_data, 'sha256'), 'hex');

    RETURN v_computed_hash = v_record.log_hash;
END;
$$ LANGUAGE plpgsql;
```

---

### Шаг 4: Применение триггеров к таблицам курсовой работы

#### Последовательность команд

```sql
-- Создаем триггеры аудита на таблицу policies
CREATE TRIGGER trg_audit_policies_insert
    AFTER INSERT ON insurance_system.policies
    FOR EACH ROW
    EXECUTE FUNCTION audit.universal_audit_trigger();

CREATE TRIGGER trg_audit_policies_update
    AFTER UPDATE ON insurance_system.policies
    FOR EACH ROW
    EXECUTE FUNCTION audit.universal_audit_trigger();

CREATE TRIGGER trg_audit_policies_delete
    AFTER DELETE ON insurance_system.policies
    FOR EACH ROW
    EXECUTE FUNCTION audit.universal_audit_trigger();

-- Создаем триггеры аудита на таблицу payments
CREATE TRIGGER trg_audit_payments_insert
    AFTER INSERT ON insurance_system.payments
    FOR EACH ROW
    EXECUTE FUNCTION audit.universal_audit_trigger();

CREATE TRIGGER trg_audit_payments_update
    AFTER UPDATE ON insurance_system.payments
    FOR EACH ROW
    EXECUTE FUNCTION audit.universal_audit_trigger();

CREATE TRIGGER trg_audit_payments_delete
    AFTER DELETE ON insurance_system.payments
    FOR EACH ROW
    EXECUTE FUNCTION audit.universal_audit_trigger();

-- Создаем триггеры на остальные таблицы курсовой работы
-- (claims, clients, agents - аналогично)
```

#### Детальное объяснение

1. **`AFTER INSERT/UPDATE/DELETE`**
   - Триггеры срабатывают ПОСЛЕ успешного выполнения операции
   - Если операция отменится (ROLLBACK), лог не будет записан
   - AFTER триггеры не могут изменить данные (в отличие от BEFORE)

2. **`FOR EACH ROW`**
   - Триггер срабатывает для каждой затронутой строки
   - Если UPDATE изменит 100 строк, триггер выполнится 100 раз
   - Альтернатива: FOR EACH STATEMENT (один раз на весь оператор)

3. **`EXECUTE FUNCTION audit.universal_audit_trigger()`**
   - Вызывает универсальную функцию аудита
   - Одна функция используется для всех таблиц и всех операций
   - Функция определяет тип операции через TG_OP

#### Тестирование аудита

```sql
-- Выполняем различные операции над данными

-- INSERT
INSERT INTO insurance_system.policies
  (policy_number, client_id, agent_id, policy_type, start_date, end_date, premium, status)
VALUES
  ('POL-TEST-001', 1, 1, 'auto', '2024-01-16', '2025-01-16', 25000.00, 'active');

-- UPDATE
UPDATE insurance_system.policies
SET premium = 27000.00
WHERE policy_number = 'POL-TEST-001';

-- DELETE
DELETE FROM insurance_system.policies
WHERE policy_number = 'POL-TEST-001';

-- Просмотр логов аудита
SELECT
    log_id,
    table_name,
    operation,
    changed_by,
    changed_at,
    new_data->>'policy_number' AS policy_number,
    new_data->>'premium' AS new_premium,
    old_data->>'premium' AS old_premium,
    log_hash
FROM audit.data_audit_log
WHERE table_name = 'policies'
ORDER BY log_id DESC
LIMIT 10;

-- Проверка целостности лога
SELECT audit.verify_log_integrity(1);
```

#### Объяснение результатов

**Результат SELECT из audit.data_audit_log:**
```
 log_id | table_name | operation |  changed_by  |     changed_at      | policy_number | new_premium | old_premium |        log_hash
--------+------------+-----------+--------------+---------------------+---------------+-------------+-------------+------------------
      3 | policies   | DELETE    | postgres     | 2024-01-16 15:30:45 |               |             | 27000.00    | a3f5d8c2e1b...
      2 | policies   | UPDATE    | postgres     | 2024-01-16 15:30:30 | POL-TEST-001  | 27000.00    | 25000.00    | 7b2e4f9a8c1...
      1 | policies   | INSERT    | postgres     | 2024-01-16 15:30:15 | POL-TEST-001  | 25000.00    |             | 9d1c6e3b5a7...
```

**Объяснение колонок:**
- `log_id` - уникальный ID записи лога
- `operation` - тип операции (INSERT/UPDATE/DELETE)
- `changed_by` - пользователь PostgreSQL, выполнивший операцию
- `changed_at` - точное время операции
- `policy_number`, `new_premium`, `old_premium` - извлечены из JSONB через оператор ->>'
- `log_hash` - SHA256 хеш для проверки целостности

**Проверка целостности:**
```
 verify_log_integrity
----------------------
 t
```
Возвращает `t` (true), если хеш соответствует данным.

---

## Задание 2: Событийные триггеры для операций DDL

**Что требуется:**
Создать событийный триггер на создание, изменение, удаление таблиц. Таблица с логами должна находиться в отдельной схеме. Доступ к таблицам логов должен быть только у администратора.

---

### Шаг 1: Создание таблицы для логирования DDL-операций

#### Последовательность команд

```sql
-- Создаем таблицу для хранения логов DDL-операций
CREATE TABLE audit.ddl_audit_log (
    log_id BIGSERIAL PRIMARY KEY,
    event_type VARCHAR(50) NOT NULL,
    object_type VARCHAR(50),
    schema_name VARCHAR(100),
    object_name VARCHAR(200),
    ddl_command TEXT,
    executed_by VARCHAR(100) NOT NULL,
    executed_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    client_address INET,
    application_name VARCHAR(200),
    transaction_id BIGINT,
    is_temp_object BOOLEAN DEFAULT FALSE,
    log_hash VARCHAR(64) NOT NULL
);

-- Индексы для быстрого поиска
CREATE INDEX idx_ddl_event_type ON audit.ddl_audit_log(event_type);
CREATE INDEX idx_ddl_object_name ON audit.ddl_audit_log(object_name);
CREATE INDEX idx_ddl_timestamp ON audit.ddl_audit_log(executed_at);
CREATE INDEX idx_ddl_user ON audit.ddl_audit_log(executed_by);

-- Защита от изменений (аналогично data_audit_log)
CREATE TRIGGER trg_protect_ddl_audit_log
    BEFORE UPDATE OR DELETE ON audit.ddl_audit_log
    FOR EACH ROW
    EXECUTE FUNCTION audit.protect_audit_log();

COMMENT ON TABLE audit.ddl_audit_log IS 'Таблица для хранения логов всех DDL-операций (CREATE/ALTER/DROP)';
```

#### Детальное объяснение каждой команды

1. **Поля таблицы:**
   - `log_id` - уникальный ID записи
   - `event_type` - тип события: 'ddl_command_start', 'ddl_command_end', 'sql_drop'
   - `object_type` - тип объекта: 'table', 'index', 'function', 'view' и т.д.
   - `schema_name` - схема, в которой находится объект
   - `object_name` - имя объекта (таблица, индекс и т.д.)
   - `ddl_command` - полный текст DDL-команды
   - `executed_by` - пользователь, выполнивший команду
   - `executed_at` - время выполнения
   - `client_address` - IP-адрес клиента
   - `application_name` - имя приложения
   - `transaction_id` - ID транзакции
   - `is_temp_object` - флаг временного объекта (TEMPORARY TABLE)
   - `log_hash` - SHA256 хеш для проверки целостности

2. **Индексы:**
   - Оптимизируют поиск по типу события, имени объекта, времени и пользователю

3. **Защитный триггер:**
   - Использует ту же функцию `audit.protect_audit_log()`
   - Запрещает изменение и удаление логов DDL

---

### Шаг 2: Создание функций событийных триггеров

#### Последовательность команд

```sql
-- Функция для события ddl_command_end (после выполнения DDL)
CREATE OR REPLACE FUNCTION audit.log_ddl_command()
RETURNS event_trigger AS $$
DECLARE
    v_obj RECORD;
    v_log_data TEXT;
    v_hash VARCHAR(64);
    v_is_temp BOOLEAN;
BEGIN
    -- Получаем информацию о созданных/измененных объектах
    FOR v_obj IN SELECT * FROM pg_event_trigger_ddl_commands()
    LOOP
        -- Проверяем, является ли объект временным
        v_is_temp := (v_obj.object_identity LIKE '%pg_temp%');

        -- Формируем строку для хеширования
        v_log_data := TG_EVENT || '|' ||
                      v_obj.object_type || '|' ||
                      COALESCE(v_obj.schema_name, 'public') || '|' ||
                      v_obj.object_identity || '|' ||
                      current_query() || '|' ||
                      current_user || '|' ||
                      CURRENT_TIMESTAMP::TEXT || '|' ||
                      txid_current()::TEXT;

        v_hash := encode(digest(v_log_data, 'sha256'), 'hex');

        -- Записываем в таблицу аудита
        INSERT INTO audit.ddl_audit_log (
            event_type,
            object_type,
            schema_name,
            object_name,
            ddl_command,
            executed_by,
            executed_at,
            client_address,
            application_name,
            transaction_id,
            is_temp_object,
            log_hash
        ) VALUES (
            TG_EVENT,
            v_obj.object_type,
            v_obj.schema_name,
            v_obj.object_identity,
            current_query(),
            current_user,
            CURRENT_TIMESTAMP,
            inet_client_addr(),
            current_setting('application_name', true),
            txid_current(),
            v_is_temp,
            v_hash
        );
    END LOOP;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

-- Функция для события sql_drop (перед удалением объектов)
CREATE OR REPLACE FUNCTION audit.log_ddl_drop()
RETURNS event_trigger AS $$
DECLARE
    v_obj RECORD;
    v_log_data TEXT;
    v_hash VARCHAR(64);
    v_is_temp BOOLEAN;
BEGIN
    -- Получаем информацию об удаляемых объектах
    FOR v_obj IN SELECT * FROM pg_event_trigger_dropped_objects()
    LOOP
        v_is_temp := (v_obj.object_identity LIKE '%pg_temp%');

        v_log_data := TG_EVENT || '|' ||
                      v_obj.object_type || '|' ||
                      COALESCE(v_obj.schema_name, 'public') || '|' ||
                      v_obj.object_identity || '|' ||
                      current_query() || '|' ||
                      current_user || '|' ||
                      CURRENT_TIMESTAMP::TEXT || '|' ||
                      txid_current()::TEXT;

        v_hash := encode(digest(v_log_data, 'sha256'), 'hex');

        INSERT INTO audit.ddl_audit_log (
            event_type,
            object_type,
            schema_name,
            object_name,
            ddl_command,
            executed_by,
            executed_at,
            client_address,
            application_name,
            transaction_id,
            is_temp_object,
            log_hash
        ) VALUES (
            TG_EVENT,
            v_obj.object_type,
            v_obj.schema_name,
            v_obj.object_identity,
            current_query(),
            current_user,
            CURRENT_TIMESTAMP,
            inet_client_addr(),
            current_setting('application_name', true),
            txid_current(),
            v_is_temp,
            v_hash
        );
    END LOOP;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

COMMENT ON FUNCTION audit.log_ddl_command() IS 'Функция событийного триггера для логирования CREATE/ALTER команд';
COMMENT ON FUNCTION audit.log_ddl_drop() IS 'Функция событийного триггера для логирования DROP команд';
```

#### Детальное объяснение каждой команды

1. **`RETURNS event_trigger`**
   - Специальный тип возвращаемого значения для событийных триггеров
   - Отличается от обычных триггеров (которые возвращают TRIGGER)

2. **`pg_event_trigger_ddl_commands()`**
   - Функция PostgreSQL, возвращающая информацию о выполненных DDL-командах
   - Доступна только в контексте события `ddl_command_end`
   - Возвращает записи с полями:
     - `classid` - OID системного каталога
     - `objid` - OID объекта
     - `objsubid` - ID подобъекта (0 для таблиц)
     - `command_tag` - тег команды (CREATE TABLE, ALTER TABLE и т.д.)
     - `object_type` - тип объекта ('table', 'index', 'function' и т.д.)
     - `schema_name` - схема объекта
     - `object_identity` - полное имя объекта (schema.table)
     - `in_extension` - является ли объект частью расширения

3. **`pg_event_trigger_dropped_objects()`**
   - Функция для получения информации об удаляемых объектах
   - Доступна только в контексте события `sql_drop`
   - Возвращает аналогичные поля

4. **`current_query()`**
   - Возвращает полный текст текущего SQL-запроса
   - Полезно для аудита - видим точную команду, которая была выполнена
   - Для событийных триггеров это будет DDL-команда (CREATE TABLE, DROP INDEX и т.д.)

5. **`TG_EVENT`**
   - Специальная переменная для событийных триггеров
   - Содержит имя события: 'ddl_command_end', 'sql_drop' и т.д.

6. **Проверка временных объектов:**
   - `v_obj.object_identity LIKE '%pg_temp%'` - проверяет, находится ли объект во временной схеме
   - Временные таблицы создаются в схемах вида `pg_temp_N`
   - Флаг `is_temp_object` позволяет фильтровать временные объекты при анализе

---

### Шаг 3: Создание событийных триггеров

#### Последовательность команд

```sql
-- Событийный триггер для CREATE и ALTER
CREATE EVENT TRIGGER evt_audit_ddl_command
    ON ddl_command_end
    EXECUTE FUNCTION audit.log_ddl_command();

-- Событийный триггер для DROP
CREATE EVENT TRIGGER evt_audit_ddl_drop
    ON sql_drop
    EXECUTE FUNCTION audit.log_ddl_drop();

-- Комментарии
COMMENT ON EVENT TRIGGER evt_audit_ddl_command IS 'Событийный триггер для аудита CREATE/ALTER операций';
COMMENT ON EVENT TRIGGER evt_audit_ddl_drop IS 'Событийный триггер для аудита DROP операций';
```

#### Детальное объяснение

1. **`CREATE EVENT TRIGGER evt_audit_ddl_command`**
   - Создает событийный триггер (глобальный для всей базы данных)
   - Имя триггера: `evt_audit_ddl_command` (префикс evt_ - соглашение)

2. **`ON ddl_command_end`**
   - Триггер срабатывает ПОСЛЕ выполнения DDL-команды
   - Другие события: `ddl_command_start` (до выполнения), `table_rewrite` (при перезаписи таблицы)
   - На `ddl_command_end` объект уже создан/изменен в системных каталогах

3. **`ON sql_drop`**
   - Триггер срабатывает ПЕРЕД удалением объекта
   - Выполняется до того, как объект исчезнет из системных каталогов
   - Позволяет зафиксировать информацию об удаляемом объекте

#### Просмотр событийных триггеров

```sql
-- Просмотр событийных триггеров
SELECT
    evtname AS trigger_name,
    evtevent AS event_name,
    evtenabled AS enabled
FROM pg_event_trigger
ORDER BY evtname;

-- Или через psql
\dET
```

**Результат:**
```
     trigger_name      |   event_name    | enabled
-----------------------+-----------------+---------
 evt_audit_ddl_command | ddl_command_end | O
 evt_audit_ddl_drop    | sql_drop        | O
```

**Объяснение колонки enabled:**
- `O` - Origin (включен всегда)
- `R` - Replica (включен на реплике)
- `A` - Always (включен всегда)
- `D` - Disabled (отключен)

---

### Шаг 4: Тестирование событийных триггеров

#### Последовательность команд

```sql
-- Подключаемся как администратор tno_admin
\c - tno_admin

-- Создаем тестовую таблицу
CREATE TABLE public.test_ddl_audit (
    id SERIAL PRIMARY KEY,
    test_data VARCHAR(100)
);

-- Изменяем структуру таблицы
ALTER TABLE public.test_ddl_audit ADD COLUMN created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP;

-- Создаем индекс
CREATE INDEX idx_test_data ON public.test_ddl_audit(test_data);

-- Создаем временную таблицу (для проверки фильтрации)
CREATE TEMPORARY TABLE temp_test (
    id INT
);

-- Удаляем объекты
DROP INDEX public.idx_test_data;
DROP TABLE public.test_ddl_audit;
DROP TABLE temp_test;

-- Просматриваем логи DDL-операций
SELECT
    log_id,
    event_type,
    object_type,
    object_name,
    executed_by,
    executed_at,
    is_temp_object,
    substring(ddl_command, 1, 50) AS ddl_preview
FROM audit.ddl_audit_log
ORDER BY log_id DESC;

-- Фильтруем только постоянные таблицы (не временные)
SELECT
    object_type,
    object_name,
    executed_by,
    executed_at
FROM audit.ddl_audit_log
WHERE object_type = 'table' AND is_temp_object = FALSE
ORDER BY executed_at DESC;

-- Подсчет операций по типам
SELECT
    event_type,
    object_type,
    COUNT(*) AS operation_count
FROM audit.ddl_audit_log
GROUP BY event_type, object_type
ORDER BY operation_count DESC;
```

#### Объяснение результатов

**Результат SELECT из audit.ddl_audit_log:**
```
 log_id |   event_type    | object_type |        object_name        | executed_by |     executed_at     | is_temp_object |                ddl_preview
--------+-----------------+-------------+---------------------------+-------------+---------------------+----------------+-------------------------------------------
      6 | sql_drop        | table       | temp_test                 | tno_admin   | 2024-01-16 16:15:45 | t              | DROP TABLE temp_test;
      5 | sql_drop        | table       | public.test_ddl_audit     | tno_admin   | 2024-01-16 16:15:40 | f              | DROP TABLE public.test_ddl_audit;
      4 | sql_drop        | index       | public.idx_test_data      | tno_admin   | 2024-01-16 16:15:35 | f              | DROP INDEX public.idx_test_data;
      3 | ddl_command_end | index       | public.idx_test_data      | tno_admin   | 2024-01-16 16:15:20 | f              | CREATE INDEX idx_test_data ON public.te...
      2 | ddl_command_end | table       | public.test_ddl_audit     | tno_admin   | 2024-01-16 16:15:10 | f              | ALTER TABLE public.test_ddl_audit ADD CO...
      1 | ddl_command_end | table       | public.test_ddl_audit     | tno_admin   | 2024-01-16 16:15:00 | f              | CREATE TABLE public.test_ddl_audit (    ...
```

**Объяснение колонок:**
- `event_type` - тип события (ddl_command_end для CREATE/ALTER, sql_drop для DROP)
- `object_type` - тип объекта (table, index, function и т.д.)
- `object_name` - полное имя объекта (включая схему)
- `is_temp_object` - TRUE для временных объектов (temp_test)
- `ddl_preview` - первые 50 символов DDL-команды

**Ответ на вопрос задания:** Да, временные таблицы попадают в аудит, но помечены флагом `is_temp_object = TRUE`, что позволяет их фильтровать при анализе.

---

## Инструкция по выполнению на Windows 10

### Запуск psql и подключение

**Способ 1: Через cmd**
```cmd
# Подключение как postgres (суперпользователь)
psql -U postgres -d your_database

# Подключение как tno_admin
psql -U tno_admin -d your_database
```

**Способ 2: Через PowerShell**
```powershell
# Аналогично cmd
psql -U postgres -d your_database
psql -U tno_admin -d your_database
```

**Способ 3: Использовать pgAdmin 4**
- Query Tool предоставляет удобный GUI для выполнения команд
- Можно переключаться между пользователями в свойствах подключения

### Проверка прав доступа

```sql
-- Подключаемся как tno_admin
\c - tno_admin

-- Проверяем доступ к схеме audit (должно работать)
SELECT COUNT(*) FROM audit.data_audit_log;

-- Проверяем доступ к основным таблицам (должно работать, если есть права)
SELECT COUNT(*) FROM insurance_system.policies;

-- Подключаемся как другой пользователь (если есть)
\c - some_other_user

-- Пытаемся получить доступ к audit (должна быть ошибка)
SELECT COUNT(*) FROM audit.data_audit_log;
-- ERROR:  permission denied for schema audit
```

### Установка расширения pgcrypto

Если pgcrypto не установлен:
```sql
-- От имени postgres
CREATE EXTENSION pgcrypto;

-- Проверка установленных расширений
\dx
```

---

## Концептуальное объяснение системы аудита

### Архитектура системы аудита

**Компоненты:**
1. **Схема audit** - изолированный namespace для всех объектов аудита
2. **Таблицы логов** - data_audit_log (DML), ddl_audit_log (DDL)
3. **Универсальные функции** - переиспользуемые триггерные функции
4. **Триггеры** - привязка функций к таблицам и событиям
5. **Защитные механизмы** - запрет изменения логов, проверка целостности

### Почему JSONB для хранения данных?

**Преимущества JSONB:**
- Гибкость: схема логов не меняется при добавлении колонок в таблицу
- Запросы: поддерживает индексы (GIN) и операторы ->>, ->, @>, ? и др.
- Сжатие: PostgreSQL автоматически сжимает JSONB (TOAST)
- Стандарт: JSON - универсальный формат обмена данными

**Пример запросов к JSONB:**
```sql
-- Извлечение конкретного поля
SELECT new_data->>'premium' AS premium FROM audit.data_audit_log;

-- Проверка наличия ключа
SELECT * FROM audit.data_audit_log WHERE new_data ? 'policy_number';

-- Проверка вложенных значений
SELECT * FROM audit.data_audit_log WHERE new_data @> '{"status": "active"}';

-- Все ключи в JSONB
SELECT jsonb_object_keys(new_data) FROM audit.data_audit_log LIMIT 1;
```

### Зачем нужна цифровая подпись (log_hash)?

**Проблема:** Администратор БД (или злоумышленник с доступом) может изменить данные в таблице аудита напрямую, скрыв следы.

**Решение:** Криптографический хеш SHA256
- Вычисляется от всех ключевых данных записи
- Любое изменение данных приведет к несоответствию хеша
- SHA256 - криптографически стойкая функция (невозможно подобрать данные под хеш)

**Проверка целостности:**
```sql
-- Проверить все логи
SELECT
    log_id,
    audit.verify_log_integrity(log_id) AS is_valid
FROM audit.data_audit_log;

-- Найти поврежденные записи
SELECT log_id, changed_at, changed_by
FROM audit.data_audit_log
WHERE NOT audit.verify_log_integrity(log_id);
```

### SECURITY DEFINER: безопасность vs удобство

**Проблема:** Обычные пользователи должны изменять данные в таблицах (policies, payments), но НЕ должны иметь права INSERT в audit.data_audit_log.

**Решение:** SECURITY DEFINER
- Функция выполняется с правами владельца (postgres), а не вызывающего
- Обычный пользователь может вызвать функцию, и она запишет в audit
- Без SECURITY DEFINER пришлось бы давать всем INSERT на audit (небезопасно)

**Риски SECURITY DEFINER:**
- SQL-инъекции опасны (функция выполняется с повышенными правами)
- Нужна тщательная валидация всех параметров
- В нашем случае безопасно: мы не принимаем параметры от пользователя

### Событийные триггеры vs обычные триггеры

**Обычные триггеры (DML):**
- Привязаны к конкретной таблице
- Срабатывают на INSERT/UPDATE/DELETE/TRUNCATE
- Видят данные строк (OLD, NEW)
- Могут изменить данные (BEFORE)

**Событийные триггеры (DDL):**
- Глобальные для всей базы данных
- Срабатывают на DDL-команды (CREATE, ALTER, DROP)
- Не видят данные строк, видят метаданные объектов
- Не могут изменить структуру, только отменить команду

**Применение событийных триггеров:**
- Аудит изменений структуры БД
- Защита критичных таблиц от удаления
- Автоматизация (например, создание audit-триггеров на новые таблицы)
- Репликация изменений структуры

### Управление триггерами

```sql
-- Отключение всех DML-триггеров на таблице
ALTER TABLE insurance_system.policies DISABLE TRIGGER ALL;

-- Включение триггеров
ALTER TABLE insurance_system.policies ENABLE TRIGGER ALL;

-- Отключение конкретного триггера
ALTER TABLE insurance_system.policies DISABLE TRIGGER trg_audit_policies_insert;

-- Отключение событийного триггера (от имени суперпользователя)
ALTER EVENT TRIGGER evt_audit_ddl_command DISABLE;

-- Включение событийного триггера
ALTER EVENT TRIGGER evt_audit_ddl_command ENABLE;

-- Удаление событийного триггера
DROP EVENT TRIGGER evt_audit_ddl_command;
```

**Когда отключать триггеры:**
- Массовая загрузка данных (миллионы строк)
- Миграция данных между базами
- Обслуживание и восстановление
- **ВАЖНО:** Отключение триггеров аудита = пропущенные логи!

---

## Выводы

В данной лабораторной работе были реализованы:

1. **Система DML-аудита:**
   - Универсальная триггерная функция для всех таблиц
   - Логирование в формате JSONB
   - Цифровая подпись SHA256 для обнаружения подмен
   - Защита от изменения логов
   - Разграничение доступа (только администратор)

2. **Система DDL-аудита:**
   - Событийные триггеры для CREATE/ALTER/DROP
   - Логирование полного текста команд
   - Фильтрация временных объектов
   - Отдельная схема для изоляции данных

3. **Механизмы безопасности:**
   - Изоляция через отдельную схему audit
   - SECURITY DEFINER для контролируемого повышения прав
   - Защитные триггеры против модификации логов
   - Криптографическая проверка целостности

4. **Практическое применение:**
   - Соответствие требованиям аудита (SOX, GDPR, PCI DSS)
   - Forensics: расследование инцидентов безопасности
   - Compliance: доказательство соответствия регуляторным требованиям
   - Change tracking: отслеживание всех изменений данных

Правильно настроенная система аудита - критически важный компонент любой промышленной базы данных, обеспечивающий прозрачность, безопасность и соответствие нормативным требованиям.

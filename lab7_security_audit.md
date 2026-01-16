# Лабораторная работа №7: Аудит безопасности PostgreSQL

## Краткая последовательность команд

```sql
-- 1. Проверка версии PostgreSQL
SELECT version();
SHOW server_version;

-- 2. Проверка установленных расширений
SELECT * FROM pg_available_extensions ORDER BY name;
SELECT extname, extversion, nspname AS schema
FROM pg_extension
JOIN pg_namespace ON pg_extension.extnamespace = pg_namespace.oid;

-- 3. Проверка конфигурации SSL/TLS
SHOW ssl;
SHOW ssl_cert_file;
SHOW ssl_key_file;
SELECT ssl, ssl_version, ssl_cipher FROM pg_stat_ssl;

-- 4. Проверка конфигурации логирования
SHOW log_statement;
SHOW log_min_duration_statement;
SHOW log_connections;
SHOW log_disconnections;
SHOW log_line_prefix;
SHOW log_destination;

-- 5. Аудит суперпользователей
SELECT rolname, rolsuper, rolcreaterole, rolcreatedb, rolcanlogin, rolbypassrls
FROM pg_roles
WHERE rolsuper = true OR rolcreaterole = true OR rolcreatedb = true OR rolbypassrls = true
ORDER BY rolname;

-- 6. Полный список пользователей и их прав
\du+

SELECT
    r.rolname,
    r.rolsuper,
    r.rolinherit,
    r.rolcreaterole,
    r.rolcreatedb,
    r.rolcanlogin,
    r.rolreplication,
    r.rolbypassrls,
    r.rolconnlimit,
    r.rolvaliduntil,
    ARRAY(SELECT b.rolname
          FROM pg_catalog.pg_auth_members m
          JOIN pg_catalog.pg_roles b ON (m.roleid = b.oid)
          WHERE m.member = r.oid) as member_of
FROM pg_catalog.pg_roles r
WHERE r.rolname !~ '^pg_'
ORDER BY r.rolname;

-- 7. Проверка прав доступа по умолчанию
\ddp

SELECT
    pg_catalog.pg_get_userbyid(d.defaclrole) AS "Role",
    n.nspname AS "Schema",
    CASE d.defaclobjtype
        WHEN 'r' THEN 'table'
        WHEN 'S' THEN 'sequence'
        WHEN 'f' THEN 'function'
        WHEN 'T' THEN 'type'
        WHEN 'n' THEN 'schema'
    END AS "Type",
    pg_catalog.array_to_string(d.defaclacl, E'\n') AS "Access privileges"
FROM pg_catalog.pg_default_acl d
LEFT JOIN pg_catalog.pg_namespace n ON n.oid = d.defaclnamespace
ORDER BY 1, 2, 3;

-- 8. Права доступа к таблицам
\dp

SELECT
    schemaname,
    tablename,
    tableowner,
    hasindexes,
    hasrules,
    hastriggers,
    rowsecurity
FROM pg_tables
WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
ORDER BY schemaname, tablename;

-- 9. Проверка Row-Level Security
SELECT
    schemaname,
    tablename,
    rowsecurity,
    CASE WHEN rowsecurity THEN 'Enabled' ELSE 'Disabled' END as rls_status
FROM pg_tables
WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
ORDER BY schemaname, tablename;

SELECT * FROM pg_policies ORDER BY schemaname, tablename;

-- 10. Матрица доступа к объектам
SELECT
    grantee,
    table_schema,
    table_name,
    string_agg(privilege_type, ', ') as privileges
FROM information_schema.table_privileges
WHERE table_schema NOT IN ('pg_catalog', 'information_schema')
GROUP BY grantee, table_schema, table_name
ORDER BY grantee, table_schema, table_name;

-- 11. Проверка прав на передачу привилегий (GRANT OPTION)
SELECT
    grantor,
    grantee,
    table_schema,
    table_name,
    privilege_type,
    is_grantable
FROM information_schema.role_table_grants
WHERE is_grantable = 'YES'
  AND table_schema NOT IN ('pg_catalog', 'information_schema')
ORDER BY grantee, table_schema, table_name;

-- 12. Проверка пользователей без пароля
SELECT rolname
FROM pg_authid
WHERE rolcanlogin = true AND rolpassword IS NULL;

-- 13. Проверка сопоставления пользователей
\deu

SELECT * FROM pg_user_mappings;

-- 14. Проверка динамического SQL в функциях
SELECT
    n.nspname as schema,
    p.proname as function_name,
    pg_get_functiondef(p.oid) as definition
FROM pg_proc p
JOIN pg_namespace n ON p.pronamespace = n.oid
WHERE pg_get_functiondef(p.oid) ILIKE '%EXECUTE%'
  AND n.nspname NOT IN ('pg_catalog', 'information_schema')
ORDER BY n.nspname, p.proname;

-- 15. Проверка опасных функций
SELECT
    n.nspname as schema,
    p.proname as function_name,
    l.lanname as language
FROM pg_proc p
JOIN pg_namespace n ON p.pronamespace = n.oid
JOIN pg_language l ON p.prolang = l.oid
WHERE l.lanname IN ('plpythonu', 'plperlu', 'c')
  AND n.nspname NOT IN ('pg_catalog', 'information_schema')
ORDER BY n.nspname, p.proname;

-- 16. Проверка доступа к системным каталогам
SELECT
    grantee,
    table_schema,
    table_name,
    privilege_type
FROM information_schema.table_privileges
WHERE table_schema IN ('pg_catalog', 'information_schema')
  AND grantee NOT IN ('postgres', 'PUBLIC')
ORDER BY grantee, table_schema, table_name;

-- 17. Проверка активных подключений
SELECT
    pid,
    usename,
    application_name,
    client_addr,
    client_port,
    backend_start,
    state,
    query
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY backend_start;

-- 18. Статистика по аутентификации
SELECT
    datname,
    numbackends,
    xact_commit,
    xact_rollback,
    blks_read,
    blks_hit,
    tup_returned,
    tup_fetched,
    tup_inserted,
    tup_updated,
    tup_deleted
FROM pg_stat_database
WHERE datname NOT IN ('template0', 'template1')
ORDER BY datname;

-- 19. Проверка триггеров (возможные backdoors)
SELECT
    n.nspname as schema,
    t.tgname as trigger_name,
    c.relname as table_name,
    p.proname as function_name,
    pg_get_triggerdef(t.oid) as trigger_def
FROM pg_trigger t
JOIN pg_class c ON t.tgrelid = c.oid
JOIN pg_namespace n ON c.relnamespace = n.oid
LEFT JOIN pg_proc p ON t.tgfoid = p.oid
WHERE n.nspname NOT IN ('pg_catalog', 'information_schema')
  AND NOT t.tgisinternal
ORDER BY n.nspname, c.relname, t.tgname;

-- 20. Проверка правил перезаписи (rules)
SELECT
    schemaname,
    tablename,
    rulename,
    definition
FROM pg_rules
WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
ORDER BY schemaname, tablename;

-- 21. Проверка внешних таблиц и серверов
SELECT
    srvname as server_name,
    srvowner::regrole as owner,
    fdwname as fdw_name,
    srvoptions as options
FROM pg_foreign_server
JOIN pg_foreign_data_wrapper ON srvfdw = pg_foreign_data_wrapper.oid;

SELECT
    ft.relname as foreign_table,
    fs.srvname as server_name,
    n.nspname as schema
FROM pg_class ft
JOIN pg_foreign_table ON ft.oid = ftrelid
JOIN pg_foreign_server fs ON ftserver = fs.oid
JOIN pg_namespace n ON ft.relnamespace = n.oid;

-- 22. Проверка публикаций и подписок (логическая репликация)
SELECT * FROM pg_publication;
SELECT * FROM pg_subscription;

-- 23. Размеры баз данных и таблиц
SELECT
    datname,
    pg_size_pretty(pg_database_size(datname)) as size
FROM pg_database
ORDER BY pg_database_size(datname) DESC;

SELECT
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) as total_size,
    pg_size_pretty(pg_relation_size(schemaname||'.'||tablename)) as table_size,
    pg_size_pretty(pg_indexes_size(schemaname||'.'||tablename)) as indexes_size
FROM pg_tables
WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC
LIMIT 20;

-- 24. Проверка использования pgcrypto для шифрования
SELECT
    table_schema,
    table_name,
    column_name,
    data_type,
    CASE
        WHEN data_type = 'bytea' THEN 'Possibly encrypted'
        ELSE 'Plain text'
    END as encryption_status
FROM information_schema.columns
WHERE table_schema NOT IN ('pg_catalog', 'information_schema')
ORDER BY table_schema, table_name, column_name;

-- 25. Мониторинг долгих запросов
SELECT
    pid,
    now() - query_start as duration,
    usename,
    query
FROM pg_stat_activity
WHERE state != 'idle'
  AND now() - query_start > interval '1 minute'
ORDER BY duration DESC;

-- 26. Проверка блокировок
SELECT
    blocked_locks.pid AS blocked_pid,
    blocked_activity.usename AS blocked_user,
    blocking_locks.pid AS blocking_pid,
    blocking_activity.usename AS blocking_user,
    blocked_activity.query AS blocked_statement,
    blocking_activity.query AS current_statement_in_blocking_process
FROM pg_catalog.pg_locks blocked_locks
JOIN pg_catalog.pg_stat_activity blocked_activity ON blocked_activity.pid = blocked_locks.pid
JOIN pg_catalog.pg_locks blocking_locks
    ON blocking_locks.locktype = blocked_locks.locktype
    AND blocking_locks.database IS NOT DISTINCT FROM blocked_locks.database
    AND blocking_locks.relation IS NOT DISTINCT FROM blocked_locks.relation
    AND blocking_locks.pid != blocked_locks.pid
JOIN pg_catalog.pg_stat_activity blocking_activity ON blocking_activity.pid = blocking_locks.pid
WHERE NOT blocked_locks.granted;

-- 27. Проверка настроек archive_mode
SHOW archive_mode;
SHOW archive_command;
SHOW wal_level;

-- 28. Статистика архивации WAL
SELECT * FROM pg_stat_archiver;

-- 29. Проверка последнего резервного копирования
SELECT
    pg_last_wal_receive_lsn() as receive_lsn,
    pg_last_wal_replay_lsn() as replay_lsn,
    pg_is_in_recovery() as in_recovery;

-- 30. Проверка системных функций с правами SECURITY DEFINER
SELECT
    n.nspname as schema,
    p.proname as function_name,
    pg_get_userbyid(p.proowner) as owner,
    p.prosecdef as security_definer,
    l.lanname as language
FROM pg_proc p
JOIN pg_namespace n ON p.pronamespace = n.oid
JOIN pg_language l ON p.prolang = l.oid
WHERE p.prosecdef = true
  AND n.nspname NOT IN ('pg_catalog', 'information_schema')
ORDER BY n.nspname, p.proname;
```

---

## Подробное объяснение команд и проверок

### 1. Проверка версии PostgreSQL

```sql
SELECT version();
SHOW server_version;
```

**Что проверяем:** Актуальность версии PostgreSQL и наличие последних патчей безопасности.

**Вывод (пример):**
```
                                                    version
----------------------------------------------------------------------------------------------------------------
PostgreSQL 14.10 (Ubuntu 14.10-1.pgdg22.04+1) on x86_64-pc-linux-gnu, compiled by gcc (Ubuntu 11.4.0-1ubuntu1~22.04) 11.4.0, 64-bit
```

**Объяснение:**
- `14.10` — версия PostgreSQL (major.minor)
- Проверить на официальном сайте PostgreSQL последнюю стабильную версию
- Старые версии могут содержать известные уязвимости

**Проверка актуальности:**
```bash
# Сравнить с последней версией
# https://www.postgresql.org/support/versioning/
```

**Рекомендации:**
- ✅ Использовать последнюю minor версию в рамках вашей major версии
- ✅ Планировать обновление до поддерживаемой major версии
- ❌ Избегать EOL (End of Life) версий

**CVE (Критические уязвимости):** Регулярно проверять на https://www.cvedetails.com/product/575/Postgresql-Postgresql.html

---

### 2. Проверка установленных расширений

```sql
SELECT * FROM pg_available_extensions ORDER BY name;

SELECT extname, extversion, nspname AS schema
FROM pg_extension
JOIN pg_namespace ON pg_extension.extnamespace = pg_namespace.oid;
```

**Что проверяем:** Наличие опасных или неиспользуемых расширений, которые расширяют поверхность атаки.

**Вывод (пример):**
```
  extname   | extversion |  schema
------------+------------+-----------
 plpgsql    | 1.0        | pg_catalog
 pg_stat_statements | 1.9 | public
 pgcrypto   | 1.3        | public
```

**Опасные расширения:**

**❌ Высокий риск:**
- `dblink` — позволяет подключаться к другим БД, риск lateral movement
- `file_fdw` — доступ к файловой системе сервера
- `adminpack` — административные функции, доступ к FS
- `plpythonu` (untrusted) — выполнение Python без песочницы
- `plperlu` (untrusted) — выполнение Perl без песочницы

**⚠️ Средний риск:**
- `postgres_fdw` — доступ к удаленным PostgreSQL серверам
- `plpython3u` — untrusted Python
- `xml2` — обработка XML, возможны XXE атаки

**✅ Безопасные:**
- `plpgsql` — стандартный процедурный язык
- `pgcrypto` — криптографические функции
- `pg_stat_statements` — статистика запросов
- `uuid-ossp` — генерация UUID
- `hstore` — key-value хранилище

**Рекомендации:**
```sql
-- Удалить неиспользуемые расширения
DROP EXTENSION IF EXISTS dblink CASCADE;

-- Ограничить использование расширения конкретной схемой
CREATE SCHEMA IF NOT EXISTS extensions;
CREATE EXTENSION pgcrypto SCHEMA extensions;

-- Ограничить права на схему с расширениями
REVOKE ALL ON SCHEMA extensions FROM PUBLIC;
GRANT USAGE ON SCHEMA extensions TO app_user;
```

**Концепция:** Принцип минимальных привилегий — устанавливать только необходимые расширения и ограничивать доступ к ним.

---

### 3. Проверка SSL/TLS конфигурации

```sql
SHOW ssl;
SHOW ssl_cert_file;
SHOW ssl_key_file;

SELECT
    pid,
    usename,
    application_name,
    client_addr,
    ssl,
    ssl_version,
    ssl_cipher
FROM pg_stat_ssl
JOIN pg_stat_activity USING (pid)
WHERE ssl = true;
```

**Что проверяем:** Включено ли шифрование соединений и используют ли его клиенты.

**Вывод (пример):**
```
 ssl |           ssl_cert_file            |          ssl_key_file
-----+------------------------------------+----------------------------------
 on  | /etc/postgresql/ssl/server.crt     | /etc/postgresql/ssl/server.key
```

**Анализ подключений:**
```
  pid  | usename  | application_name | client_addr |  ssl | ssl_version |      ssl_cipher
-------+----------+------------------+-------------+------+-------------+----------------------
 12345 | app_user | myapp            | 10.0.1.5    | t    | TLSv1.3     | TLS_AES_256_GCM_SHA384
 12346 | admin    | psql             | 10.0.1.10   | f    | NULL        | NULL
```

**Проблемы:**
- ❌ `ssl = off` — шифрование отключено
- ❌ Подключения без SSL (ssl = false)
- ❌ Старые протоколы (TLSv1.0, TLSv1.1)
- ❌ Слабые шифры (RC4, DES, 3DES)

**Проверка pg_hba.conf:**
```bash
cat /etc/postgresql/14/main/pg_hba.conf | grep -v "^#" | grep -v "^$"
```

**Рекомендуемая конфигурация:**
```
# pg_hba.conf
# Требовать SSL для всех TCP подключений
hostssl  all  all  0.0.0.0/0  scram-sha-256
hostnossl all all  0.0.0.0/0  reject

# Локальные подключения без SSL (безопасны)
local   all  all                peer
```

**postgresql.conf:**
```ini
ssl = on
ssl_cert_file = '/etc/postgresql/ssl/server.crt'
ssl_key_file = '/etc/postgresql/ssl/server.key'
ssl_ca_file = '/etc/postgresql/ssl/root.crt'
ssl_min_protocol_version = 'TLSv1.2'
ssl_ciphers = 'HIGH:!aNULL:!eNULL:!EXPORT:!DES:!MD5:!PSK:!RC4'
ssl_prefer_server_ciphers = on
```

**Рекомендации:**
- ✅ Всегда использовать SSL для удаленных подключений
- ✅ Минимальная версия TLS 1.2, лучше 1.3
- ✅ Использовать сертификаты от доверенного CA
- ✅ Настроить клиентские сертификаты для привилегированных пользователей

**Концепция:** Шифрование защищает данные при передаче от перехвата (man-in-the-middle атаки).

---

### 4. Проверка конфигурации логирования

```sql
SHOW log_statement;
SHOW log_min_duration_statement;
SHOW log_connections;
SHOW log_disconnections;
SHOW log_line_prefix;
SHOW log_destination;
```

**Что проверяем:** Достаточно ли детальное логирование для аудита и расследования инцидентов.

**Вывод (пример):**
```
 log_statement | log_min_duration_statement | log_connections | log_disconnections
---------------+----------------------------+-----------------+--------------------
 ddl           | 1000                       | on              | on

           log_line_prefix           | log_destination
-------------------------------------+------------------
 %t [%p]: user=%u,db=%d,app=%a,client=%h | stderr
```

**Объяснение параметров:**

**`log_statement`:**
- `none` — не логировать запросы (❌ небезопасно)
- `ddl` — логировать DDL (CREATE, ALTER, DROP) (⚠️ минимум)
- `mod` — логировать DDL + DML (INSERT, UPDATE, DELETE) (✅ рекомендуется)
- `all` — логировать все запросы (⚠️ много логов, но максимальная безопасность)

**`log_min_duration_statement`:**
- `-1` — отключено
- `0` — логировать все запросы
- `1000` — логировать запросы дольше 1 секунды (✅ для production)

**`log_connections` / `log_disconnections`:**
- Логировать все подключения и отключения (✅ обязательно)

**`log_line_prefix`:**
- Формат строки лога
- `%t` — timestamp
- `%p` — process ID
- `%u` — username
- `%d` — database
- `%a` — application_name
- `%h` — client host
- `%i` — command tag
- `%e` — SQL state

**Рекомендуемая конфигурация для безопасности:**
```ini
# Логирование
log_destination = 'stderr,csvlog'
logging_collector = on
log_directory = '/var/log/postgresql'
log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log'
log_rotation_age = 1d
log_rotation_size = 100MB

# Что логировать
log_connections = on
log_disconnections = on
log_duration = off
log_statement = 'mod'  # DDL и DML
log_min_duration_statement = 1000  # > 1 секунды

# Формат
log_line_prefix = '%t [%p]: user=%u,db=%d,app=%a,client=%h '
log_checkpoints = on
log_lock_waits = on

# Безопасность
log_hostname = on
log_error_verbosity = default

# Аудит ошибок
log_min_error_statement = error
```

**Дополнительное логирование с pgAudit:**
```sql
-- Установка pgAudit
CREATE EXTENSION pgaudit;

-- Настройка в postgresql.conf
-- shared_preload_libraries = 'pgaudit'
-- pgaudit.log = 'write, ddl'
-- pgaudit.log_catalog = off
-- pgaudit.log_parameter = on
```

**Анализ логов:**
```bash
# Поиск неудачных попыток аутентификации
grep "authentication failed" /var/log/postgresql/postgresql-*.log

# Поиск SQL-инъекций (подозрительные паттерны)
grep -E "(UNION|OR 1=1|'; DROP|<script>)" /var/log/postgresql/postgresql-*.log

# Поиск DDL команд
grep "CREATE\|ALTER\|DROP" /var/log/postgresql/postgresql-*.log

# Топ пользователей по количеству запросов
awk -F'user=' '{print $2}' /var/log/postgresql/postgresql-*.log | awk -F',' '{print $1}' | sort | uniq -c | sort -rn | head -10
```

**Рекомендации:**
- ✅ Логировать все DDL и DML операции
- ✅ Логировать подключения и отключения
- ✅ Централизованное хранение логов (syslog, ELK, Splunk)
- ✅ Регулярный анализ логов на аномалии
- ✅ Защита логов от модификации (immutable файлы, отправка на SIEM)

**Концепция:** Логирование — основа для обнаружения инцидентов безопасности и forensics расследования.

---

### 5. Аудит суперпользователей и привилегированных ролей

```sql
SELECT
    rolname,
    rolsuper,
    rolcreaterole,
    rolcreatedb,
    rolcanlogin,
    rolbypassrls
FROM pg_roles
WHERE rolsuper = true
   OR rolcreaterole = true
   OR rolcreatedb = true
   OR rolbypassrls = true
ORDER BY rolname;
```

**Что проверяем:** Минимизация количества привилегированных пользователей.

**Вывод (пример):**
```
  rolname   | rolsuper | rolcreaterole | rolcreatedb | rolcanlogin | rolbypassrls
------------+----------+---------------+-------------+-------------+--------------
 postgres   | t        | t             | t           | t           | t
 admin_user | f        | t             | t           | t           | f
 backup_user| f        | f             | f           | t           | f
```

**Анализ привилегий:**

**`rolsuper = true` (Суперпользователь):**
- ✅ Должен быть ТОЛЬКО postgres (системный)
- ❌ Любые другие роли с rolsuper — риск безопасности
- Может обойти все проверки безопасности, изменить любые данные

**`rolcreaterole = true` (Может создавать роли):**
- ⚠️ Может создавать других пользователей и назначать им права
- Ограничить минимальным набором (только DBA)

**`rolcreatedb = true` (Может создавать базы данных):**
- ⚠️ Может создавать новые БД и потреблять ресурсы
- Ограничить по необходимости

**`rolbypassrls = true` (Обходит Row-Level Security):**
- ❌ Может видеть все данные, игнорируя RLS политики
- Опасно для ролей приложений

**Рекомендуемая структура ролей:**

```sql
-- 1. Суперпользователь (только для emergencies)
-- postgres (по умолчанию, не использовать в приложениях!)

-- 2. Административная роль (без superuser)
CREATE ROLE admin_role WITH
    CREATEROLE
    CREATEDB
    NOLOGIN;

CREATE ROLE dba_user WITH
    LOGIN
    PASSWORD 'secure_password'
    IN ROLE admin_role;

-- 3. Роль приложения (минимальные права)
CREATE ROLE app_role WITH
    NOLOGIN;

GRANT CONNECT ON DATABASE mydb TO app_role;
GRANT USAGE ON SCHEMA public TO app_role;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app_role;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO app_role;

-- Автоматическое назначение прав на новые объекты
ALTER DEFAULT PRIVILEGES IN SCHEMA public
    GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO app_role;
ALTER DEFAULT PRIVILEGES IN SCHEMA public
    GRANT USAGE, SELECT ON SEQUENCES TO app_role;

CREATE ROLE app_user WITH
    LOGIN
    PASSWORD 'app_password'
    IN ROLE app_role;

-- 4. Роль только для чтения
CREATE ROLE readonly_role WITH NOLOGIN;
GRANT CONNECT ON DATABASE mydb TO readonly_role;
GRANT USAGE ON SCHEMA public TO readonly_role;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO readonly_role;

CREATE ROLE analyst_user WITH
    LOGIN
    PASSWORD 'analyst_password'
    IN ROLE readonly_role;

-- 5. Роль для резервного копирования
CREATE ROLE backup_role WITH
    REPLICATION
    LOGIN
    PASSWORD 'backup_password';
```

**Проверка иерархии ролей:**
```sql
SELECT
    r.rolname as role,
    ARRAY(
        SELECT b.rolname
        FROM pg_catalog.pg_auth_members m
        JOIN pg_catalog.pg_roles b ON (m.roleid = b.oid)
        WHERE m.member = r.oid
    ) as member_of,
    ARRAY(
        SELECT m.rolname
        FROM pg_catalog.pg_auth_members mb
        JOIN pg_catalog.pg_roles m ON (mb.member = m.oid)
        WHERE mb.roleid = r.oid
    ) as members
FROM pg_catalog.pg_roles r
WHERE r.rolname !~ '^pg_'
ORDER BY r.rolname;
```

**Рекомендации:**
- ✅ Только один суперпользователь (postgres)
- ✅ Использовать иерархию ролей (role inheritance)
- ✅ Приложения подключаются с минимальными правами
- ❌ Никогда не использовать суперпользователя в приложениях
- ✅ Регулярно проводить аудит прав

**Концепция:** Принцип наименьших привилегий (Principle of Least Privilege) — каждая роль должна иметь только те права, которые необходимы для выполнения её функций.

---

### 6-11. Матрица доступа и аудит прав

```sql
-- Права доступа к таблицам
SELECT
    grantee,
    table_schema,
    table_name,
    string_agg(privilege_type, ', ' ORDER BY privilege_type) as privileges
FROM information_schema.table_privileges
WHERE table_schema NOT IN ('pg_catalog', 'information_schema')
GROUP BY grantee, table_schema, table_name
ORDER BY grantee, table_schema, table_name;
```

**Вывод (пример):**
```
   grantee    | table_schema | table_name |          privileges
--------------+--------------+------------+--------------------------------
 app_user     | public       | users      | DELETE, INSERT, SELECT, UPDATE
 app_user     | public       | orders     | INSERT, SELECT
 readonly_user| public       | users      | SELECT
 readonly_user| public       | orders     | SELECT
```

**Анализ матрицы доступа:**

**Проблемы:**
- ❌ PUBLIC имеет права (любой может подключиться)
- ❌ Слишком широкие права (app_user имеет DELETE на всех таблицах)
- ❌ Пользователи имеют прямой доступ к таблицам (обход бизнес-логики)

**Рекомендуемый подход — доступ через функции:**
```sql
-- Отозвать прямой доступ к таблицам
REVOKE ALL ON ALL TABLES IN SCHEMA public FROM app_role;

-- Создать API функции
CREATE OR REPLACE FUNCTION get_user(user_id INT)
RETURNS TABLE(id INT, name VARCHAR, email VARCHAR) AS $$
BEGIN
    -- Бизнес-логика, валидация, логирование
    RETURN QUERY SELECT id, name, email FROM users WHERE id = user_id;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

-- Дать права только на функции
GRANT EXECUTE ON FUNCTION get_user(INT) TO app_role;
```

**Проверка GRANT OPTION (возможность передавать права):**
```sql
SELECT
    grantor,
    grantee,
    table_schema,
    table_name,
    privilege_type,
    is_grantable
FROM information_schema.role_table_grants
WHERE is_grantable = 'YES'
  AND table_schema NOT IN ('pg_catalog', 'information_schema')
ORDER BY grantee, table_schema, table_name;
```

**Если is_grantable = YES:**
- ⚠️ Пользователь может передавать права другим пользователям
- Риск неконтролируемого распространения привилегий

**Рекомендации:**
```sql
-- Отозвать GRANT OPTION
REVOKE GRANT OPTION FOR ALL PRIVILEGES ON users FROM app_user;

-- Или при выдаче прав не использовать WITH GRANT OPTION
GRANT SELECT ON users TO app_user;  -- Без WITH GRANT OPTION
```

**Концепция:** Контроль доступа на основе ролей (RBAC) и минимизация поверхности атаки через ограничение прямого доступа к данным.

---

### 9. Row-Level Security (RLS)

```sql
-- Проверка включения RLS
SELECT
    schemaname,
    tablename,
    rowsecurity,
    CASE WHEN rowsecurity THEN 'Enabled' ELSE 'Disabled' END as rls_status
FROM pg_tables
WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
ORDER BY schemaname, tablename;

-- Просмотр политик RLS
SELECT
    schemaname,
    tablename,
    policyname,
    permissive,
    roles,
    cmd,
    qual,
    with_check
FROM pg_policies
ORDER BY schemaname, tablename, policyname;
```

**Что проверяем:** Используется ли Row-Level Security для защиты чувствительных данных.

**Вывод (пример):**
```
 schemaname | tablename | rowsecurity | rls_status
------------+-----------+-------------+------------
 public     | users     | t           | Enabled
 public     | orders    | t           | Enabled
 public     | products  | f           | Disabled
```

**Пример политик RLS:**
```
 schemaname | tablename | policyname     | permissive | roles       | cmd    | qual
------------+-----------+----------------+------------+-------------+--------+------------------------
 public     | users     | user_isolation | PERMISSIVE | {app_user}  | SELECT | (user_id = current_user_id())
 public     | orders    | user_orders    | PERMISSIVE | {app_user}  | ALL    | (customer_id = current_user_id())
```

**Реализация RLS:**

**Шаг 1: Включить RLS на таблице**
```sql
ALTER TABLE users ENABLE ROW LEVEL SECURITY;
```

**Шаг 2: Создать функцию для получения текущего пользователя**
```sql
CREATE OR REPLACE FUNCTION current_user_id()
RETURNS INTEGER AS $$
BEGIN
    RETURN current_setting('app.user_id')::INTEGER;
EXCEPTION
    WHEN OTHERS THEN
        RETURN NULL;
END;
$$ LANGUAGE plpgsql STABLE;
```

**Шаг 3: Создать политику**
```sql
-- Пользователи видят только свои записи
CREATE POLICY user_isolation ON users
    FOR ALL
    TO app_user
    USING (user_id = current_user_id());

-- Другой вариант: на основе роли
CREATE POLICY manager_access ON employees
    FOR SELECT
    TO manager_role
    USING (department_id IN (
        SELECT department_id
        FROM managers
        WHERE user_id = current_user_id()
    ));
```

**Шаг 4: Установить контекст в приложении**
```python
# В приложении при подключении
connection.execute("SET app.user_id = %s", [authenticated_user_id])

# Теперь все запросы автоматически фильтруются
cursor.execute("SELECT * FROM users")  # Вернёт только записи текущего пользователя
```

**Типы политик:**

**PERMISSIVE (OR logic):**
```sql
CREATE POLICY policy1 ON table USING (condition1);
CREATE POLICY policy2 ON table USING (condition2);
-- Строка видна если condition1 OR condition2
```

**RESTRICTIVE (AND logic):**
```sql
CREATE POLICY policy1 ON table AS RESTRICTIVE USING (condition1);
CREATE POLICY policy2 ON table AS RESTRICTIVE USING (condition2);
-- Строка видна если condition1 AND condition2
```

**Команды (cmd):**
- `ALL` — для всех операций
- `SELECT` — только для чтения
- `INSERT` — только для вставки
- `UPDATE` — только для обновления
- `DELETE` — только для удаления

**USING vs WITH CHECK:**
- `USING` — условие для SELECT, UPDATE, DELETE (какие строки видимы)
- `WITH CHECK` — условие для INSERT, UPDATE (какие строки можно создать/изменить)

**Пример мультитенантности:**
```sql
-- Таблица с tenant_id
CREATE TABLE documents (
    id SERIAL PRIMARY KEY,
    tenant_id INT NOT NULL,
    title VARCHAR(255),
    content TEXT
);

-- Функция для получения tenant
CREATE FUNCTION current_tenant_id()
RETURNS INTEGER AS $$
BEGIN
    RETURN current_setting('app.tenant_id')::INTEGER;
END;
$$ LANGUAGE plpgsql STABLE;

-- Политика изоляции тенантов
ALTER TABLE documents ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON documents
    FOR ALL
    TO app_user
    USING (tenant_id = current_tenant_id())
    WITH CHECK (tenant_id = current_tenant_id());

-- В приложении
SET app.tenant_id = 123;
SELECT * FROM documents;  -- Только документы tenant_id = 123
```

**Обход RLS (для админов):**
```sql
-- Создать роль без RLS ограничений
CREATE ROLE admin_user WITH BYPASSRLS LOGIN PASSWORD 'admin_pass';

-- Или временно отключить для суперпользователя
SET row_security = OFF;
SELECT * FROM users;  -- Видны все строки
SET row_security = ON;
```

**Производительность RLS:**
- ⚠️ RLS добавляет условие к каждому запросу
- ✅ Использовать индексы на полях в USING условии
- ✅ Функции в USING должны быть STABLE или IMMUTABLE
- ⚠️ Сложные подзапросы в USING могут замедлить запросы

**Рекомендации:**
- ✅ Использовать RLS для мультитенантности
- ✅ Применять RLS для чувствительных данных (personal info, финансы)
- ✅ Тестировать производительность с RLS
- ✅ Документировать политики RLS
- ❌ Не полагаться только на RLS (defense in depth)

**Концепция:** Row-Level Security обеспечивает автоматическую фильтрацию данных на уровне БД, предотвращая утечки даже при ошибках в коде приложения.

---

## Итоговый чек-лист аудита безопасности

### ✅ Конфигурация сервера
- [ ] PostgreSQL обновлён до последней стабильной версии
- [ ] Отключены/ограничены опасные расширения (dblink, adminpack, plpythonu)
- [ ] pg_hba.conf настроен с строгим контролем доступа
- [ ] SSL/TLS включён и правильно настроен
- [ ] Логирование включено (log_statement = mod, log_connections = on)
- [ ] Логи защищены и регулярно анализируются

### ✅ Управление правами и ролями
- [ ] Минимум суперпользователей (только postgres)
- [ ] Применён принцип наименьших привилегий
- [ ] Роли структурированы иерархически
- [ ] Нет пользователей без пароля
- [ ] GRANT OPTION используется осторожно
- [ ] Матрица доступа документирована

### ✅ Защита данных
- [ ] Row-Level Security включён для чувствительных таблиц
- [ ] Конфиденциальные поля зашифрованы (pgcrypto)
- [ ] Нет прямого доступа к таблицам (доступ через функции)
- [ ] Доступ к системным каталогам ограничен

### ✅ SQL-код и процедуры
- [ ] Запросы параметризованы (защита от SQL-инъекций)
- [ ] Динамический SQL проверен на безопасность
- [ ] Опасные функции (COPY PROGRAM, dblink_exec) не используются
- [ ] SECURITY DEFINER функции проверены и минимизированы

### ✅ Мониторинг и аудит
- [ ] Активные подключения мониторятся
- [ ] Долгие запросы отслеживаются
- [ ] Блокировки анализируются
- [ ] Триггеры и правила проверены на backdoors
- [ ] pgAudit установлен и настроен

### ✅ Резервное копирование
- [ ] Регулярное резервное копирование настроено
- [ ] Бэкапы зашифрованы
- [ ] Восстановление протестировано
- [ ] WAL архивация настроена для PITR

### ✅ Дополнительные меры
- [ ] Firewall настроен (только необходимые порты)
- [ ] PostgreSQL работает от непривилегированного пользователя
- [ ] Директория данных имеет права 0700
- [ ] Регулярное сканирование на уязвимости
- [ ] План реагирования на инциденты документирован

---

## Автоматизация аудита

**Скрипт для автоматической проверки:**
```bash
#!/bin/bash

echo "=== PostgreSQL Security Audit Report ==="
echo "Generated: $(date)"
echo ""

# 1. Версия
echo "1. PostgreSQL Version:"
psql -U postgres -t -c "SELECT version();"
echo ""

# 2. Суперпользователи
echo "2. Superusers:"
psql -U postgres -t -c "SELECT rolname FROM pg_roles WHERE rolsuper ORDER BY rolname;"
echo ""

# 3. SSL конфигурация
echo "3. SSL Configuration:"
psql -U postgres -t -c "SHOW ssl;"
echo ""

# 4. Подключения без SSL
echo "4. Connections without SSL:"
psql -U postgres -t -c "SELECT usename, client_addr FROM pg_stat_ssl JOIN pg_stat_activity USING (pid) WHERE ssl = false;"
echo ""

# 5. Опасные расширения
echo "5. Installed Extensions:"
psql -U postgres -t -c "SELECT extname FROM pg_extension WHERE extname IN ('dblink', 'file_fdw', 'adminpack', 'plpythonu', 'plperlu') ORDER BY extname;"
echo ""

# 6. Пользователи без пароля
echo "6. Users without password:"
psql -U postgres -t -c "SELECT rolname FROM pg_authid WHERE rolcanlogin = true AND rolpassword IS NULL;"
echo ""

# 7. GRANT OPTION
echo "7. Users with GRANT OPTION:"
psql -U postgres -t -c "SELECT DISTINCT grantee FROM information_schema.role_table_grants WHERE is_grantable = 'YES' AND table_schema NOT IN ('pg_catalog', 'information_schema');"
echo ""

# 8. Динамический SQL в функциях
echo "8. Functions with EXECUTE (dynamic SQL):"
psql -U postgres -t -c "SELECT n.nspname||'.'||p.proname FROM pg_proc p JOIN pg_namespace n ON p.pronamespace = n.oid WHERE pg_get_functiondef(p.oid) ILIKE '%EXECUTE%' AND n.nspname NOT IN ('pg_catalog', 'information_schema');"
echo ""

echo "=== End of Report ==="
```

**Запуск:**
```bash
chmod +x security_audit.sh
./security_audit.sh > audit_report_$(date +%Y%m%d).txt
```

**Концепция:** Регулярный автоматический аудит безопасности помогает выявлять проблемы до того, как они будут эксплуатированы.

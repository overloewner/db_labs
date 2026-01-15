# Лабораторная работа №2: Многоверсионность и блокировки в PostgreSQL

## Краткая последовательность команд

```sql
-- 1. Создание базы данных и подключение
CREATE DATABASE lab2_mvcc;
\c lab2_mvcc

-- 2. Создание таблицы для экспериментов
CREATE TABLE accounts (
    id SERIAL PRIMARY KEY,
    name VARCHAR(50),
    balance DECIMAL(10,2)
);

-- 3. Вставка тестовых данных
INSERT INTO accounts (name, balance) VALUES
    ('Alice', 1000.00),
    ('Bob', 500.00),
    ('Charlie', 750.00);

-- 4. Просмотр служебных полей (xmin, xmax)
SELECT xmin, xmax, * FROM accounts;

-- 5. Обновление записи
UPDATE accounts SET balance = 1100.00 WHERE name = 'Alice';

-- 6. Снова просмотр служебных полей (xmin изменился)
SELECT xmin, xmax, * FROM accounts;

-- 7. Просмотр информации о текущей транзакции
SELECT txid_current(), txid_current_if_assigned();

-- 8. Демонстрация снимков данных (открыть два терминала)
-- Терминал 1:
BEGIN;
SELECT txid_current();
SELECT * FROM accounts WHERE name = 'Bob';

-- Терминал 2 (пока транзакция в Терминале 1 активна):
BEGIN;
UPDATE accounts SET balance = 600.00 WHERE name = 'Bob';
COMMIT;

-- Терминал 1 (продолжение):
SELECT * FROM accounts WHERE name = 'Bob';  -- Видим старое значение!
COMMIT;

-- 9. Просмотр блокировок
SELECT
    locktype,
    database,
    relation::regclass,
    page,
    tuple,
    virtualxid,
    transactionid,
    mode,
    granted
FROM pg_locks
WHERE pid = pg_backend_pid();

-- 10. Демонстрация блокировок на уровне строк
-- Терминал 1:
BEGIN;
SELECT * FROM accounts WHERE name = 'Alice' FOR UPDATE;

-- Терминал 2:
BEGIN;
SELECT * FROM accounts WHERE name = 'Alice' FOR UPDATE;  -- Будет ждать!

-- Терминал 1:
COMMIT;  -- Терминал 2 получит блокировку

-- 11. Просмотр заблокированных запросов
SELECT
    blocked_locks.pid AS blocked_pid,
    blocked_activity.usename AS blocked_user,
    blocking_locks.pid AS blocking_pid,
    blocking_activity.usename AS blocking_user,
    blocked_activity.query AS blocked_statement,
    blocking_activity.query AS blocking_statement
FROM pg_catalog.pg_locks blocked_locks
JOIN pg_catalog.pg_stat_activity blocked_activity ON blocked_activity.pid = blocked_locks.pid
JOIN pg_catalog.pg_locks blocking_locks
    ON blocking_locks.locktype = blocked_locks.locktype
    AND blocking_locks.database IS NOT DISTINCT FROM blocked_locks.database
    AND blocking_locks.relation IS NOT DISTINCT FROM blocked_locks.relation
    AND blocking_locks.page IS NOT DISTINCT FROM blocked_locks.page
    AND blocking_locks.tuple IS NOT DISTINCT FROM blocked_locks.tuple
    AND blocking_locks.virtualxid IS NOT DISTINCT FROM blocked_locks.virtualxid
    AND blocking_locks.transactionid IS NOT DISTINCT FROM blocked_locks.transactionid
    AND blocking_locks.classid IS NOT DISTINCT FROM blocked_locks.classid
    AND blocking_locks.objid IS NOT DISTINCT FROM blocked_locks.objid
    AND blocking_locks.objsubid IS NOT DISTINCT FROM blocked_locks.objsubid
    AND blocking_locks.pid != blocked_locks.pid
JOIN pg_catalog.pg_stat_activity blocking_activity ON blocking_activity.pid = blocking_locks.pid
WHERE NOT blocked_locks.granted;

-- 12. Демонстрация накопления мертвых версий
CREATE TABLE vacuum_test (id INT, data TEXT);
INSERT INTO vacuum_test SELECT generate_series(1, 10000), 'test data';

-- Множественные обновления
UPDATE vacuum_test SET data = 'updated' WHERE id <= 5000;
UPDATE vacuum_test SET data = 'updated again' WHERE id <= 3000;

-- 13. Просмотр статистики мертвых строк
SELECT
    schemaname,
    relname,
    n_live_tup,
    n_dead_tup,
    last_vacuum,
    last_autovacuum
FROM pg_stat_user_tables
WHERE relname = 'vacuum_test';

-- 14. Ручной запуск VACUUM
VACUUM VERBOSE vacuum_test;

-- 15. Повторный просмотр статистики
SELECT
    schemaname,
    relname,
    n_live_tup,
    n_dead_tup,
    last_vacuum,
    last_autovacuum
FROM pg_stat_user_tables
WHERE relname = 'vacuum_test';

-- 16. VACUUM FULL (полная очистка с блокировкой)
VACUUM FULL vacuum_test;

-- 17. Просмотр размера таблицы до и после VACUUM
SELECT pg_size_pretty(pg_total_relation_size('vacuum_test'));
```

---

## Подробное объяснение команд

### 1-3. Создание тестовой среды

```sql
CREATE DATABASE lab2_mvcc;
\c lab2_mvcc

CREATE TABLE accounts (
    id SERIAL PRIMARY KEY,
    name VARCHAR(50),
    balance DECIMAL(10,2)
);

INSERT INTO accounts (name, balance) VALUES
    ('Alice', 1000.00),
    ('Bob', 500.00),
    ('Charlie', 750.00);
```

**Что делает:** Создает базу данных, таблицу счетов и наполняет её тестовыми данными.

**Концепция:** Простая таблица для демонстрации механизмов MVCC и блокировок.

---

### 4. Просмотр служебных полей (xmin, xmax)

```sql
SELECT xmin, xmax, * FROM accounts;
```

**Что делает:** Выводит служебные поля многоверсионности вместе с пользовательскими данными.

**Вывод (пример):**
```
 xmin | xmax | id |  name   | balance
------+------+----+---------+---------
  734 |    0 |  1 | Alice   | 1000.00
  734 |    0 |  2 | Bob     |  500.00
  734 |    0 |  3 | Charlie |  750.00
```

**Объяснение колонок:**
- `xmin` — Transaction ID (XID) транзакции, которая вставила эту версию строки. Все три строки вставлены в одной транзакции (XID 734)
- `xmax` — XID транзакции, которая удалила или обновила эту версию строки. Значение 0 означает, что строка актуальна и не была удалена
- `id`, `name`, `balance` — пользовательские данные

**Концепция MVCC:** Каждая строка содержит информацию о том, какая транзакция её создала (xmin) и какая удалила/обновила (xmax). PostgreSQL использует эти метки для определения видимости строк в разных транзакциях.

---

### 5-6. Обновление и отслеживание версий

```sql
UPDATE accounts SET balance = 1100.00 WHERE name = 'Alice';
SELECT xmin, xmax, * FROM accounts;
```

**Что делает:** Обновляет баланс Alice и показывает изменение служебных полей.

**Вывод после UPDATE:**
```
 xmin | xmax | id |  name   | balance
------+------+----+---------+---------
  735 |    0 |  1 | Alice   | 1100.00
  734 |    0 |  2 | Bob     |  500.00
  734 |    0 |  3 | Charlie |  750.00
```

**Объяснение:**
- Строка Alice теперь имеет `xmin = 735` (новая транзакция создала новую версию)
- Старая версия строки (с `xmin = 734`, `xmax = 735`) физически остается в таблице как "мертвая" версия
- Новая версия имеет `xmax = 0`, показывая, что она актуальна

**Концепция:** При UPDATE PostgreSQL не перезаписывает старую версию строки. Вместо этого создается новая версия, а старая помечается как удаленная (через установку xmax). Это позволяет параллельным транзакциям продолжать видеть старую версию без блокировок.

---

### 7. Просмотр ID текущей транзакции

```sql
SELECT txid_current(), txid_current_if_assigned();
```

**Что делает:** Выводит ID текущей транзакции.

**Вывод (пример):**
```
 txid_current | txid_current_if_assigned
--------------+---------------------------
          736 |                       736
```

**Объяснение функций:**
- `txid_current()` — возвращает XID текущей транзакции, назначая его, если еще не назначен
- `txid_current_if_assigned()` — возвращает XID, только если он уже назначен, иначе NULL

**Концепция:** В PostgreSQL каждая транзакция получает уникальный 32-битный идентификатор (XID). Это монотонно возрастающее число. Read-only транзакции могут не получать XID для оптимизации.

---

### 8. Демонстрация снимков данных

```sql
-- Терминал 1:
BEGIN;
SELECT txid_current();  -- Пусть вернет 740
SELECT * FROM accounts WHERE name = 'Bob';  -- balance = 500.00

-- Терминал 2:
BEGIN;
UPDATE accounts SET balance = 600.00 WHERE name = 'Bob';
COMMIT;

-- Терминал 1:
SELECT * FROM accounts WHERE name = 'Bob';  -- Все еще balance = 500.00!
COMMIT;
```

**Что делает:** Демонстрирует изоляцию транзакций через механизм снимков.

**Объяснение:**
1. Транзакция 1 начинается и создает снимок данных (snapshot)
2. Снимок фиксирует, какие транзакции активны в момент его создания
3. Транзакция 2 обновляет строку и коммитится
4. Транзакция 1 продолжает видеть старую версию строки, потому что её снимок был сделан до коммита транзакции 2

**Концепция снимка данных:** Снимок содержит три компонента:
- `xmin` — самая старая активная транзакция
- `xmax` — следующая транзакция после последней завершенной
- `xip_list` — список активных транзакций в момент создания снимка

Транзакция видит только те версии строк, которые:
- Созданы транзакциями с XID < xmin (и зафиксированы)
- Или созданы текущей транзакцией
- Не удалены транзакциями, видимыми для снимка

**Технический смысл:** Это реализация уровня изоляции READ COMMITTED (по умолчанию в PostgreSQL). Каждый SQL-запрос в транзакции получает свой снимок. На уровне REPEATABLE READ снимок создается один раз в начале транзакции.

---

### 9. Просмотр блокировок

```sql
SELECT
    locktype,
    database,
    relation::regclass,
    page,
    tuple,
    virtualxid,
    transactionid,
    mode,
    granted
FROM pg_locks
WHERE pid = pg_backend_pid();
```

**Что делает:** Запрашивает информацию о всех блокировках, удерживаемых текущей сессией.

**Вывод (пример):**
```
   locktype    | database | relation | page | tuple | virtualxid | transactionid |       mode       | granted
---------------+----------+----------+------+-------+------------+---------------+------------------+---------
 relation      |    16384 | accounts |      |       |            |               | AccessShareLock  | t
 virtualxid    |          |          |      |       | 3/42       |               | ExclusiveLock    | t
 transactionid |          |          |      |       |            |           742 | ExclusiveLock    | t
```

**Объяснение колонок:**
- `locktype` — тип блокировки:
  - `relation` — блокировка таблицы/индекса
  - `tuple` — блокировка конкретной строки
  - `transactionid` — блокировка на ID транзакции (для UPDATE/DELETE)
  - `virtualxid` — виртуальный ID транзакции
  - `page` — блокировка страницы
- `database` — OID базы данных
- `relation` — имя таблицы/индекса (используем `::regclass` для преобразования OID в имя)
- `page`, `tuple` — номер страницы и строки (для построчных блокировок)
- `virtualxid` — виртуальный ID транзакции (используется до назначения реального XID)
- `transactionid` — реальный XID транзакции
- `mode` — режим блокировки:
  - `AccessShareLock` — получается при SELECT (не блокирует другие SELECT)
  - `RowShareLock` — получается при SELECT FOR SHARE
  - `RowExclusiveLock` — получается при INSERT/UPDATE/DELETE
  - `ShareUpdateExclusiveLock` — получается при VACUUM, CREATE INDEX CONCURRENTLY
  - `ShareLock` — получается при CREATE INDEX
  - `ExclusiveLock` — блокирует все, кроме AccessShareLock
  - `AccessExclusiveLock` — получается при DROP TABLE, TRUNCATE, VACUUM FULL
- `granted` — выдана ли блокировка (true) или процесс ожидает (false)

**Концепция:** PostgreSQL использует иерархическую систему блокировок. Блокировки могут конфликтовать или сосуществовать в зависимости от режима. MVCC минимизирует необходимость в блокировках для чтения.

---

### 10. Демонстрация блокировок FOR UPDATE

```sql
-- Терминал 1:
BEGIN;
SELECT * FROM accounts WHERE name = 'Alice' FOR UPDATE;

-- Терминал 2:
BEGIN;
SELECT * FROM accounts WHERE name = 'Alice' FOR UPDATE;  -- Ждет!
```

**Что делает:** Демонстрирует эксклюзивную блокировку строки.

**Объяснение:**
- `FOR UPDATE` захватывает эксклюзивную блокировку на выбранные строки
- Другие транзакции не могут изменить или заблокировать эту строку до освобождения блокировки
- `SELECT` без `FOR UPDATE` не блокируется (благодаря MVCC)

**Режимы блокировок строк:**
- `FOR UPDATE` — эксклюзивная блокировка (для последующего UPDATE/DELETE)
- `FOR NO KEY UPDATE` — менее строгая, разрешает SELECT FOR SHARE
- `FOR SHARE` — разделяемая блокировка (блокирует UPDATE/DELETE, но не другие FOR SHARE)
- `FOR KEY SHARE` — самая слабая, блокирует только изменение ключевых полей

**Концепция:** Блокировки строк необходимы для предотвращения конфликтов при одновременном изменении. PostgreSQL хранит информацию о блокировках строк в заголовках кортежей (tuple headers) и в отдельных структурах для сложных случаев.

---

### 11. Просмотр заблокированных запросов

```sql
SELECT
    blocked_locks.pid AS blocked_pid,
    blocked_activity.usename AS blocked_user,
    blocking_locks.pid AS blocking_pid,
    blocking_activity.usename AS blocking_user,
    blocked_activity.query AS blocked_statement,
    blocking_activity.query AS blocking_statement
FROM pg_catalog.pg_locks blocked_locks
JOIN pg_catalog.pg_stat_activity blocked_activity ON blocked_activity.pid = blocked_locks.pid
JOIN pg_catalog.pg_locks blocking_locks
    ON blocking_locks.locktype = blocked_locks.locktype
    AND blocking_locks.database IS NOT DISTINCT FROM blocked_locks.database
    AND blocking_locks.relation IS NOT DISTINCT FROM blocked_locks.relation
    ...
WHERE NOT blocked_locks.granted;
```

**Что делает:** Находит все заблокированные запросы и показывает, кто их блокирует.

**Вывод (пример):**
```
 blocked_pid | blocked_user | blocking_pid | blocking_user | blocked_statement | blocking_statement
-------------+--------------+--------------+---------------+-------------------+--------------------
       12345 | postgres     |        12344 | postgres      | UPDATE accounts...| UPDATE accounts...
```

**Объяснение колонок:**
- `blocked_pid` — ID процесса, который ждет
- `blocked_user` — пользователь заблокированного процесса
- `blocking_pid` — ID процесса, который держит блокировку
- `blocking_user` — пользователь блокирующего процесса
- `blocked_statement` — SQL-запрос, который ожидает
- `blocking_statement` — SQL-запрос, который держит блокировку

**Концепция:** Этот запрос помогает диагностировать дедлоки и долгие ожидания. В продакшене такие запросы часто оборачивают в функции мониторинга.

---

### 12. Создание таблицы для демонстрации VACUUM

```sql
CREATE TABLE vacuum_test (id INT, data TEXT);
INSERT INTO vacuum_test SELECT generate_series(1, 10000), 'test data';
UPDATE vacuum_test SET data = 'updated' WHERE id <= 5000;
UPDATE vacuum_test SET data = 'updated again' WHERE id <= 3000;
```

**Что делает:** Создает таблицу и выполняет множественные обновления для накопления мертвых версий строк.

**Объяснение:**
- `generate_series(1, 10000)` генерирует последовательность чисел
- Два UPDATE создают 8000 мертвых версий (5000 + 3000)

**Концепция:** Каждый UPDATE создает новую версию строки, оставляя старую как "мертвую" (dead tuple). Эти мертвые версии занимают место и замедляют запросы.

---

### 13. Просмотр статистики мертвых строк

```sql
SELECT
    schemaname,
    relname,
    n_live_tup,
    n_dead_tup,
    last_vacuum,
    last_autovacuum
FROM pg_stat_user_tables
WHERE relname = 'vacuum_test';
```

**Что делает:** Выводит статистику о живых и мертвых строках в таблице.

**Вывод (пример):**
```
 schemaname |   relname   | n_live_tup | n_dead_tup | last_vacuum |   last_autovacuum
------------+-------------+------------+------------+-------------+---------------------
 public     | vacuum_test |      10000 |       8000 |             | 2026-01-15 10:30:15
```

**Объяснение колонок:**
- `schemaname` — имя схемы
- `relname` — имя таблицы
- `n_live_tup` — количество живых строк (актуальных версий)
- `n_dead_tup` — количество мертвых строк (старых версий, которые можно удалить)
- `last_vacuum` — время последнего ручного VACUUM
- `last_autovacuum` — время последнего автоматического VACUUM

**Концепция:** PostgreSQL собирает статистику для принятия решений об автоматическом запуске VACUUM. Когда доля мертвых строк превышает пороговое значение, запускается autovacuum.

---

### 14. Ручной запуск VACUUM

```sql
VACUUM VERBOSE vacuum_test;
```

**Что делает:** Запускает процесс очистки мертвых версий строк.

**Вывод (пример):**
```
INFO:  vacuuming "public.vacuum_test"
INFO:  "vacuum_test": removed 8000 row versions in 89 pages
INFO:  "vacuum_test": found 8000 removable, 10000 nonremovable row versions in 133 out of 133 pages
DETAIL:  0 dead row versions cannot be removed yet.
There were 0 unused item identifiers.
Total free space: 655360 bytes.
```

**Объяснение вывода:**
- `removed 8000 row versions` — удалено 8000 мертвых версий
- `8000 removable, 10000 nonremovable` — могут быть удалены vs должны остаться
- `in 133 out of 133 pages` — обработано страниц
- `Total free space` — освобожденное место (в байтах)

**Что делает VACUUM:**
1. Сканирует таблицу и находит мертвые версии строк
2. Помечает место, занимаемое ими, как свободное для повторного использования
3. Обновляет статистику и visibility map
4. Освобождает место в индексах

**ВАЖНО:** Обычный VACUUM не возвращает место ОС, а делает его доступным для повторного использования внутри таблицы.

---

### 15. Проверка статистики после VACUUM

```sql
SELECT
    schemaname,
    relname,
    n_live_tup,
    n_dead_tup,
    last_vacuum,
    last_autovacuum
FROM pg_stat_user_tables
WHERE relname = 'vacuum_test';
```

**Вывод после VACUUM:**
```
 schemaname |   relname   | n_live_tup | n_dead_tup |      last_vacuum        |   last_autovacuum
------------+-------------+------------+------------+-------------------------+---------------------
 public     | vacuum_test |      10000 |          0 | 2026-01-15 11:00:00     | 2026-01-15 10:30:15
```

**Объяснение:** `n_dead_tup` теперь 0 — все мертвые версии очищены. `last_vacuum` обновился.

---

### 16. VACUUM FULL

```sql
VACUUM FULL vacuum_test;
```

**Что делает:** Выполняет полную очистку с перезаписью таблицы и возвратом места ОС.

**Отличия от обычного VACUUM:**
- Блокирует таблицу эксклюзивно (ACCESS EXCLUSIVE LOCK)
- Перезаписывает всю таблицу, удаляя пустые страницы
- Возвращает дисковое пространство операционной системе
- Требует дополнительного места на диске (размер таблицы)
- Значительно медленнее обычного VACUUM

**Когда использовать:**
- Когда таблица сильно раздулась и нужно вернуть место ОС
- В периоды низкой нагрузки (из-за эксклюзивной блокировки)
- Редко, так как это тяжелая операция

---

### 17. Сравнение размеров

```sql
SELECT pg_size_pretty(pg_total_relation_size('vacuum_test'));
```

**Что делает:** Показывает текущий размер таблицы.

**Вывод (пример до VACUUM FULL):**
```
 pg_size_pretty
----------------
 1064 kB
```

**Вывод (пример после VACUUM FULL):**
```
 pg_size_pretty
----------------
 464 kB
```

**Объяснение:** После VACUUM FULL размер таблицы уменьшился, так как удалены пустые страницы.

---

## Технический концепт: MVCC и блокировки

### Многоверсионность (MVCC)

**Суть:** PostgreSQL не удаляет и не перезаписывает строки при UPDATE/DELETE. Вместо этого создаются новые версии, а старые помечаются как устаревшие.

**Преимущества:**
- Читатели не блокируют писателей, писатели не блокируют читателей
- Высокая конкурентность
- Реализация ACID-свойств без тяжелых блокировок
- Поддержка различных уровней изоляции

**Недостатки:**
- Накопление мертвых версий требует VACUUM
- Дополнительное использование дискового пространства
- Сложность управления старыми транзакциями (transaction wraparound)

### Снимки данных (Snapshots)

**Компоненты снимка:**
- `xmin` — нижняя граница видимости
- `xmax` — верхняя граница видимости
- `xip_list` — список активных транзакций

**Правила видимости:**
Строка видна, если:
- Создана транзакцией < xmin (и зафиксирована)
- Создана текущей транзакцией
- НЕ удалена видимой транзакцией

### Блокировки

**Уровни блокировок:**
1. **Табличные блокировки** — применяются к целым таблицам
2. **Строчные блокировки** — применяются к отдельным строкам
3. **Блокировки страниц** — редко используются
4. **Консультативные блокировки** — определяемые приложением

**Режимы блокировок (от слабой к сильной):**
1. AccessShareLock (SELECT)
2. RowShareLock (SELECT FOR SHARE)
3. RowExclusiveLock (INSERT, UPDATE, DELETE)
4. ShareUpdateExclusiveLock (VACUUM, CREATE INDEX CONCURRENTLY)
5. ShareLock (CREATE INDEX)
6. ShareRowExclusiveLock
7. ExclusiveLock
8. AccessExclusiveLock (DROP TABLE, TRUNCATE, VACUUM FULL)

### VACUUM: очистка и обслуживание

**Типы VACUUM:**

1. **VACUUM** (обычный):
   - Освобождает место для повторного использования
   - Не блокирует чтение/запись
   - Не возвращает место ОС
   - Быстрый и безопасный

2. **VACUUM FULL**:
   - Полностью перезаписывает таблицу
   - Возвращает место ОС
   - Эксклюзивная блокировка
   - Медленный и ресурсоемкий

3. **AUTOVACUUM**:
   - Автоматический фоновый процесс
   - Запускается по достижении порогов
   - Предотвращает раздувание таблиц
   - Критически важен для стабильной работы

**Настройки autovacuum:**
- `autovacuum_vacuum_threshold` — минимальное количество обновлений для запуска
- `autovacuum_vacuum_scale_factor` — доля таблицы, которая должна измениться
- `autovacuum_vacuum_cost_delay` — задержка между операциями (для снижения нагрузки)

### Практическое применение

**Сценарий 1: Банковские транзакции**
- Используются блокировки FOR UPDATE для предотвращения двойного списания
- MVCC обеспечивает изоляцию параллельных транзакций
- Регулярный VACUUM предотвращает раздувание критических таблиц

**Сценарий 2: Веб-приложение с высокой нагрузкой**
- Множество параллельных SELECT не блокируют друг друга благодаря MVCC
- Редкие UPDATE не останавливают чтение
- Autovacuum работает в фоне, поддерживая производительность

**Сценарий 3: Диагностика медленных запросов**
- Проверка pg_locks для поиска блокировок
- Анализ pg_stat_activity для поиска долгих транзакций
- Мониторинг n_dead_tup для выявления проблем с VACUUM

**Сценарий 4: Предотвращение дедлоков**
- Всегда захватывать блокировки в одинаковом порядке
- Использовать минимально необходимые уровни блокировок
- Держать транзакции короткими
- Использовать NOWAIT или lock_timeout для раннего обнаружения

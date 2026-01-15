# Лабораторная работа №3: Транзакции и уровни изоляции в PostgreSQL

## Краткая последовательность команд

```sql
-- 1. Создание базы данных
CREATE DATABASE lab3_isolation;
\c lab3_isolation

-- 2. Создание тестовой таблицы
CREATE TABLE accounts (
    id SERIAL PRIMARY KEY,
    name VARCHAR(50),
    balance DECIMAL(10,2)
);

INSERT INTO accounts (name, balance) VALUES
    ('Alice', 1000.00),
    ('Bob', 500.00);

-- 3. Просмотр текущего уровня изоляции
SHOW default_transaction_isolation;

-- 4. Демонстрация READ COMMITTED (уровень по умолчанию)
-- Терминал 1:
BEGIN;
SELECT * FROM accounts WHERE name = 'Alice';

-- Терминал 2:
BEGIN;
UPDATE accounts SET balance = 1200.00 WHERE name = 'Alice';
COMMIT;

-- Терминал 1:
SELECT * FROM accounts WHERE name = 'Alice';  -- Видим новое значение!
COMMIT;

-- 5. Демонстрация неповторяемого чтения на уровне READ COMMITTED
-- Терминал 1:
BEGIN;
SELECT balance FROM accounts WHERE name = 'Bob';  -- balance = 500

-- Терминал 2:
UPDATE accounts SET balance = 600.00 WHERE name = 'Bob';

-- Терминал 1:
SELECT balance FROM accounts WHERE name = 'Bob';  -- balance = 600 (изменилось!)
COMMIT;

-- 6. Демонстрация фантомного чтения на уровне READ COMMITTED
-- Терминал 1:
BEGIN;
SELECT COUNT(*) FROM accounts;  -- COUNT = 2

-- Терминал 2:
INSERT INTO accounts (name, balance) VALUES ('Charlie', 750.00);

-- Терминал 1:
SELECT COUNT(*) FROM accounts;  -- COUNT = 3 (появилась новая строка!)
COMMIT;

-- 7. Установка уровня изоляции REPEATABLE READ
-- Терминал 1:
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SELECT * FROM accounts;

-- Терминал 2:
BEGIN;
UPDATE accounts SET balance = 1500.00 WHERE name = 'Alice';
COMMIT;

-- Терминал 1:
SELECT * FROM accounts;  -- Видим старые значения!
COMMIT;

-- 8. Попытка обновления на уровне REPEATABLE READ
-- Терминал 1:
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SELECT balance FROM accounts WHERE name = 'Alice';  -- 1000

-- Терминал 2:
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
UPDATE accounts SET balance = balance + 100 WHERE name = 'Alice';
COMMIT;

-- Терминал 1:
UPDATE accounts SET balance = balance + 200 WHERE name = 'Alice';  -- ОШИБКА!
-- ERROR:  could not serialize access due to concurrent update

-- 9. Демонстрация отсутствия фантомного чтения на REPEATABLE READ
-- Терминал 1:
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SELECT COUNT(*) FROM accounts;

-- Терминал 2:
INSERT INTO accounts (name, balance) VALUES ('Dave', 800.00);

-- Терминал 1:
SELECT COUNT(*) FROM accounts;  -- Количество не изменилось!
COMMIT;

-- 10. Установка уровня изоляции SERIALIZABLE
-- Терминал 1:
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
SELECT SUM(balance) FROM accounts;  -- Пусть = 3000

-- Терминал 2:
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
SELECT SUM(balance) FROM accounts;  -- = 3000
INSERT INTO accounts (name, balance) VALUES ('Eve', 400.00);
COMMIT;

-- Терминал 1:
INSERT INTO accounts (name, balance) VALUES ('Frank', 350.00);
COMMIT;  -- ОШИБКА!
-- ERROR:  could not serialize access due to read/write dependencies among transactions

-- 11. Демонстрация успешной сериализации
-- Терминал 1:
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
UPDATE accounts SET balance = balance * 1.1 WHERE name = 'Alice';
COMMIT;

-- Терминал 2:
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
UPDATE accounts SET balance = balance * 1.05 WHERE name = 'Bob';
COMMIT;  -- Успешно (разные строки)

-- 12. Просмотр конфликтов сериализации
SELECT * FROM pg_stat_database WHERE datname = 'lab3_isolation';

-- 13. Установка уровня изоляции для сессии
SET SESSION CHARACTERISTICS AS TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SHOW default_transaction_isolation;

-- 14. Демонстрация ROLLBACK
BEGIN;
UPDATE accounts SET balance = 0 WHERE name = 'Alice';
SELECT * FROM accounts;
ROLLBACK;  -- Отменяем изменения
SELECT * FROM accounts;  -- Данные восстановлены

-- 15. Использование SAVEPOINT
BEGIN;
UPDATE accounts SET balance = balance + 100 WHERE name = 'Alice';
SAVEPOINT my_savepoint;
UPDATE accounts SET balance = balance + 100 WHERE name = 'Bob';
-- Отменяем только последнее изменение
ROLLBACK TO SAVEPOINT my_savepoint;
COMMIT;  -- Сохраняем только изменение для Alice

-- 16. Просмотр активных транзакций
SELECT
    pid,
    usename,
    application_name,
    state,
    query,
    xact_start,
    state_change
FROM pg_stat_activity
WHERE datname = 'lab3_isolation' AND state != 'idle';
```

---

## Подробное объяснение команд

### 1-2. Создание тестовой среды

```sql
CREATE DATABASE lab3_isolation;
\c lab3_isolation

CREATE TABLE accounts (
    id SERIAL PRIMARY KEY,
    name VARCHAR(50),
    balance DECIMAL(10,2)
);

INSERT INTO accounts (name, balance) VALUES
    ('Alice', 1000.00),
    ('Bob', 500.00);
```

**Что делает:** Создает базу данных и таблицу для демонстрации уровней изоляции транзакций.

---

### 3. Просмотр текущего уровня изоляции

```sql
SHOW default_transaction_isolation;
```

**Что делает:** Отображает уровень изоляции по умолчанию для текущей сессии.

**Вывод:**
```
 default_transaction_isolation
-------------------------------
 read committed
```

**Объяснение:** В PostgreSQL уровень изоляции по умолчанию — `read committed`.

**Уровни изоляции в PostgreSQL:**
- `read committed` — чтение зафиксированных данных (по умолчанию)
- `repeatable read` — повторяемое чтение
- `serializable` — сериализуемость
- `read uncommitted` — фактически работает как read committed

**Концепция:** Уровень изоляции определяет, как транзакция видит изменения, сделанные другими параллельными транзакциями.

---

### 4. Демонстрация READ COMMITTED

```sql
-- Терминал 1:
BEGIN;
SELECT * FROM accounts WHERE name = 'Alice';  -- balance = 1000

-- Терминал 2:
BEGIN;
UPDATE accounts SET balance = 1200.00 WHERE name = 'Alice';
COMMIT;

-- Терминал 1:
SELECT * FROM accounts WHERE name = 'Alice';  -- balance = 1200
COMMIT;
```

**Что делает:** Демонстрирует, что на уровне READ COMMITTED транзакция видит изменения, зафиксированные другими транзакциями.

**Поведение READ COMMITTED:**
- Каждый новый запрос в транзакции получает свежий снимок данных
- Видны все изменения, зафиксированные до начала запроса
- Не видны незафиксированные изменения других транзакций
- Предотвращает грязное чтение (dirty read)

**Концепция:** Это самый распространенный уровень изоляции, обеспечивающий баланс между консистентностью и производительностью.

---

### 5. Демонстрация неповторяемого чтения (Non-repeatable Read)

```sql
-- Терминал 1:
BEGIN;
SELECT balance FROM accounts WHERE name = 'Bob';  -- 500

-- Терминал 2:
UPDATE accounts SET balance = 600.00 WHERE name = 'Bob';

-- Терминал 1:
SELECT balance FROM accounts WHERE name = 'Bob';  -- 600 (изменилось!)
COMMIT;
```

**Что делает:** Показывает аномалию неповторяемого чтения на уровне READ COMMITTED.

**Объяснение аномалии:**
- Первый SELECT возвращает balance = 500
- Другая транзакция обновляет и фиксирует изменение
- Второй SELECT в той же транзакции возвращает balance = 600
- Повторное чтение той же строки дает разные результаты

**Почему это происходит:** На уровне READ COMMITTED каждый запрос создает новый снимок данных, поэтому видит новые зафиксированные изменения.

**Когда это проблема:**
- Финансовые расчеты (сумма может измениться между проверкой и списанием)
- Отчеты, требующие консистентного состояния данных
- Бизнес-логика, зависящая от неизменности данных в транзакции

---

### 6. Демонстрация фантомного чтения (Phantom Read)

```sql
-- Терминал 1:
BEGIN;
SELECT COUNT(*) FROM accounts;  -- 2

-- Терминал 2:
INSERT INTO accounts (name, balance) VALUES ('Charlie', 750.00);

-- Терминал 1:
SELECT COUNT(*) FROM accounts;  -- 3
COMMIT;
```

**Что делает:** Показывает аномалию фантомного чтения на уровне READ COMMITTED.

**Объяснение аномалии:**
- Первый запрос возвращает 2 строки
- Другая транзакция вставляет новую строку
- Второй запрос в той же транзакции возвращает 3 строки
- "Фантомная" строка появилась в результате

**Отличие от неповторяемого чтения:**
- Неповторяемое чтение — изменение существующих строк
- Фантомное чтение — появление/исчезновение строк, соответствующих условию

**Когда это проблема:**
- Агрегирующие отчеты (SUM, COUNT, AVG)
- Пагинация результатов
- Проверка бизнес-правил на множестве строк

---

### 7. Уровень изоляции REPEATABLE READ

```sql
-- Терминал 1:
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SELECT * FROM accounts;  -- Видим текущее состояние

-- Терминал 2:
BEGIN;
UPDATE accounts SET balance = 1500.00 WHERE name = 'Alice';
COMMIT;

-- Терминал 1:
SELECT * FROM accounts;  -- Видим старые значения!
COMMIT;
```

**Что делает:** Демонстрирует, что на уровне REPEATABLE READ транзакция видит консистентный снимок данных.

**Вывод в Терминале 1 (оба SELECT):**
```
 id | name  | balance
----+-------+---------
  1 | Alice | 1000.00
  2 | Bob   |  500.00
```

**Поведение REPEATABLE READ:**
- Снимок данных создается один раз в начале транзакции
- Все запросы в транзакции видят одно и то же состояние данных
- Изменения других транзакций невидимы, даже после их фиксации
- Предотвращает неповторяемое и фантомное чтение

**Концепция:** Транзакция работает с "замороженным" состоянием базы данных, что обеспечивает полную консистентность в рамках транзакции.

---

### 8. Конфликт обновлений на REPEATABLE READ

```sql
-- Терминал 1:
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SELECT balance FROM accounts WHERE name = 'Alice';  -- 1000

-- Терминал 2:
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
UPDATE accounts SET balance = balance + 100 WHERE name = 'Alice';
COMMIT;

-- Терминал 1:
UPDATE accounts SET balance = balance + 200 WHERE name = 'Alice';
-- ОШИБКА!
```

**Вывод ошибки:**
```
ERROR:  could not serialize access due to concurrent update
DETAIL:  Concurrent update to the same row.
HINT:  The transaction might succeed if retried.
```

**Что произошло:**
1. Транзакция 1 читает balance = 1000
2. Транзакция 2 обновляет строку (balance = 1100) и коммитится
3. Транзакция 1 пытается обновить ту же строку
4. PostgreSQL обнаруживает конфликт и отменяет транзакцию 1

**Почему это происходит:**
- На уровне REPEATABLE READ транзакция не может обновить строку, которая была изменена после начала транзакции
- Это предотвращает lost updates (потерянные обновления)
- Приложение должно обработать ошибку и повторить транзакцию

**Концепция:** PostgreSQL использует метод first-updater-wins для разрешения конфликтов. Первая транзакция, которая фиксирует изменение, выигрывает; последующие получают ошибку сериализации.

---

### 9. Отсутствие фантомного чтения на REPEATABLE READ

```sql
-- Терминал 1:
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SELECT COUNT(*) FROM accounts;  -- 2

-- Терминал 2:
INSERT INTO accounts (name, balance) VALUES ('Dave', 800.00);

-- Терминал 1:
SELECT COUNT(*) FROM accounts;  -- Все еще 2!
COMMIT;
```

**Что делает:** Показывает, что на уровне REPEATABLE READ фантомное чтение предотвращено.

**Объяснение:**
- Транзакция 1 видит снимок данных на момент своего начала
- Новая строка, вставленная транзакцией 2, невидима для транзакции 1
- Это обеспечивает консистентность агрегирующих запросов

**Концепция:** В PostgreSQL REPEATABLE READ фактически предотвращает все аномалии, кроме аномалий сериализации. Это сильнее, чем требует стандарт SQL.

---

### 10. Уровень изоляции SERIALIZABLE

```sql
-- Терминал 1:
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
SELECT SUM(balance) FROM accounts;  -- 3000

-- Терминал 2:
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
SELECT SUM(balance) FROM accounts;  -- 3000
INSERT INTO accounts (name, balance) VALUES ('Eve', 400.00);
COMMIT;

-- Терминал 1:
INSERT INTO accounts (name, balance) VALUES ('Frank', 350.00);
COMMIT;
-- ERROR:  could not serialize access due to read/write dependencies among transactions
```

**Вывод ошибки:**
```
ERROR:  could not serialize access due to read/write dependencies among transactions
DETAIL:  Reason code: Canceled on identification as a pivot, during commit attempt.
HINT:  The transaction might succeed if retried.
```

**Что произошло:**
1. Обе транзакции читают SUM(balance)
2. Обе вставляют новые строки
3. PostgreSQL обнаруживает, что результат выполнения этих транзакций отличается от их последовательного выполнения
4. Одна из транзакций отменяется с ошибкой сериализации

**Объяснение аномалии сериализации:**
- Если обе транзакции выполнить последовательно, каждая увидела бы изменения предыдущей
- Но при параллельном выполнении каждая работает со своим снимком
- Это создает невозможную при последовательном выполнении ситуацию

**Поведение SERIALIZABLE:**
- Гарантирует, что результат параллельного выполнения транзакций эквивалентен некоторому их последовательному выполнению
- Использует SSI (Serializable Snapshot Isolation) для обнаружения конфликтов
- Самый строгий уровень изоляции
- Требует повторных попыток при ошибках сериализации

**Концепция:** SERIALIZABLE обеспечивает полную изоляцию транзакций, но требует дополнительных вычислительных ресурсов и обработки ошибок сериализации в приложении.

---

### 11. Успешная сериализация

```sql
-- Терминал 1:
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
UPDATE accounts SET balance = balance * 1.1 WHERE name = 'Alice';
COMMIT;  -- Успешно

-- Терминал 2:
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
UPDATE accounts SET balance = balance * 1.05 WHERE name = 'Bob';
COMMIT;  -- Успешно
```

**Что делает:** Показывает, что транзакции, работающие с разными строками, не конфликтуют.

**Объяснение:**
- Транзакции обновляют разные строки
- Нет read-write зависимостей между транзакциями
- Обе могут быть выполнены последовательно в любом порядке с тем же результатом
- Конфликта сериализации нет

**Концепция:** SERIALIZABLE не блокирует все транзакции. Только транзакции с конфликтующими зависимостями чтения-записи получают ошибки.

---

### 12. Просмотр конфликтов сериализации

```sql
SELECT * FROM pg_stat_database WHERE datname = 'lab3_isolation';
```

**Что делает:** Выводит статистику по базе данных, включая конфликты сериализации.

**Вывод (релевантные колонки):**
```
    datname     | xact_commit | xact_rollback | conflicts | deadlocks
----------------+-------------+---------------+-----------+-----------
 lab3_isolation |         125 |            15 |         3 |         0
```

**Объяснение колонок:**
- `datname` — имя базы данных
- `xact_commit` — количество зафиксированных транзакций
- `xact_rollback` — количество откаченных транзакций
- `conflicts` — количество конфликтов (включая сериализацию)
- `deadlocks` — количество обнаруженных дедлоков

**Концепция:** Мониторинг этих метрик помогает оценить, насколько часто возникают конфликты, и решить, подходит ли уровень SERIALIZABLE для приложения.

---

### 13. Установка уровня изоляции для сессии

```sql
SET SESSION CHARACTERISTICS AS TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SHOW default_transaction_isolation;
```

**Что делает:** Устанавливает уровень изоляции по умолчанию для всех транзакций в текущей сессии.

**Вывод:**
```
 default_transaction_isolation
-------------------------------
 repeatable read
```

**Способы установки уровня изоляции:**

1. **Для одной транзакции:**
   ```sql
   BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
   ```

2. **Для сессии:**
   ```sql
   SET SESSION CHARACTERISTICS AS TRANSACTION ISOLATION LEVEL REPEATABLE READ;
   ```

3. **Для всей базы данных:**
   ```sql
   ALTER DATABASE mydb SET default_transaction_isolation = 'repeatable read';
   ```

4. **В postgresql.conf (для всего кластера):**
   ```
   default_transaction_isolation = 'repeatable read'
   ```

**Концепция:** Выбор уровня изоляции зависит от требований приложения к консистентности и производительности.

---

### 14. Демонстрация ROLLBACK

```sql
BEGIN;
UPDATE accounts SET balance = 0 WHERE name = 'Alice';
SELECT * FROM accounts WHERE name = 'Alice';  -- balance = 0
ROLLBACK;
SELECT * FROM accounts WHERE name = 'Alice';  -- balance восстановлен
```

**Что делает:** Показывает откат транзакции, отменяющий все изменения.

**Вывод:**
```
-- После UPDATE внутри транзакции:
 id | name  | balance
----+-------+---------
  1 | Alice |    0.00

-- После ROLLBACK:
 id | name  | balance
----+-------+---------
  1 | Alice | 1000.00
```

**Объяснение:**
- `ROLLBACK` отменяет все изменения, сделанные в транзакции
- Данные возвращаются к состоянию до `BEGIN`
- Освобождаются все блокировки

**Концепция:** ROLLBACK — ключевой механизм обеспечения атомарности транзакций. Либо все изменения фиксируются (COMMIT), либо все отменяются (ROLLBACK).

---

### 15. Использование SAVEPOINT

```sql
BEGIN;
UPDATE accounts SET balance = balance + 100 WHERE name = 'Alice';
SAVEPOINT my_savepoint;
UPDATE accounts SET balance = balance + 100 WHERE name = 'Bob';
ROLLBACK TO SAVEPOINT my_savepoint;  -- Отменяем только последнее изменение
COMMIT;  -- Сохраняем изменение для Alice
```

**Что делает:** Демонстрирует частичный откат транзакции с помощью точек сохранения.

**Результат:**
- Баланс Alice увеличен на 100 (изменение сохранено)
- Баланс Bob не изменился (изменение откачено)

**Команды для работы с SAVEPOINT:**
- `SAVEPOINT name` — создать точку сохранения
- `ROLLBACK TO SAVEPOINT name` — откатиться к точке сохранения
- `RELEASE SAVEPOINT name` — удалить точку сохранения (без отката)

**Концепция:** SAVEPOINT позволяет создавать вложенные транзакции и откатывать только часть изменений. Полезно для обработки ошибок внутри сложных транзакций.

---

### 16. Просмотр активных транзакций

```sql
SELECT
    pid,
    usename,
    application_name,
    state,
    query,
    xact_start,
    state_change
FROM pg_stat_activity
WHERE datname = 'lab3_isolation' AND state != 'idle';
```

**Что делает:** Выводит информацию о всех активных подключениях и транзакциях.

**Вывод (пример):**
```
  pid  | usename  | application_name |        state        |          query           |          xact_start           |         state_change
-------+----------+------------------+---------------------+--------------------------+-------------------------------+-------------------------------
 12345 | postgres | psql             | active              | SELECT * FROM accounts;  | 2026-01-15 10:30:15.123456+00 | 2026-01-15 10:30:17.789012+00
 12346 | postgres | psql             | idle in transaction | <IDLE> in transaction    | 2026-01-15 10:29:45.654321+00 | 2026-01-15 10:30:10.111111+00
```

**Объяснение колонок:**
- `pid` — ID процесса PostgreSQL
- `usename` — имя пользователя
- `application_name` — имя приложения (psql, JDBC, и т.д.)
- `state` — состояние подключения:
  - `active` — выполняется запрос
  - `idle` — ожидает команды
  - `idle in transaction` — транзакция открыта, но не выполняется запрос
  - `idle in transaction (aborted)` — транзакция откачена из-за ошибки
- `query` — текст последнего/текущего запроса
- `xact_start` — время начала текущей транзакции (NULL если нет транзакции)
- `state_change` — время последнего изменения состояния

**Важность мониторинга:**
- **"idle in transaction"** — частая проблема. Открытые транзакции держат блокировки и снимки данных, препятствуя VACUUM
- **Долгие транзакции** — могут вызывать раздувание таблиц и блокировки
- **Активные запросы** — помогают найти медленные операции

**Концепция:** pg_stat_activity — основной инструмент мониторинга активности в PostgreSQL. Критически важен для диагностики проблем с производительностью и блокировками.

---

## Технический концепт: Транзакции и изоляция

### Свойства ACID

PostgreSQL обеспечивает ACID-свойства транзакций:

1. **Atomicity (Атомарность):**
   - Транзакция выполняется полностью или не выполняется вообще
   - COMMIT фиксирует все изменения
   - ROLLBACK отменяет все изменения
   - Реализуется через журнал WAL (Write-Ahead Log)

2. **Consistency (Согласованность):**
   - Транзакция переводит базу из одного консистентного состояния в другое
   - Проверяются ограничения (constraints)
   - Триггеры выполняются в контексте транзакции

3. **Isolation (Изолированность):**
   - Параллельные транзакции не видят промежуточные состояния друг друга
   - Реализуется через MVCC и уровни изоляции
   - Разные уровни изоляции обеспечивают разные гарантии

4. **Durability (Долговечность):**
   - После COMMIT изменения сохраняются даже при сбое
   - Реализуется через fsync и журнал WAL
   - Возможен асинхронный режим для повышения производительности

### Сравнение уровней изоляции

| Уровень изоляции | Грязное чтение | Неповторяемое чтение | Фантомное чтение | Аномалия сериализации | Производительность |
|------------------|----------------|----------------------|------------------|-----------------------|--------------------|
| Read Uncommitted | Нет (в PG)     | Да                   | Да               | Да                    | Высокая            |
| Read Committed   | Нет            | Да                   | Да               | Да                    | Высокая            |
| Repeatable Read  | Нет            | Нет                  | Нет (в PG)       | Да                    | Средняя            |
| Serializable     | Нет            | Нет                  | Нет              | Нет                   | Низкая             |

**Примечание:** В PostgreSQL READ UNCOMMITTED эквивалентен READ COMMITTED, а REPEATABLE READ предотвращает фантомное чтение (сильнее стандарта SQL).

### Механизм SSI (Serializable Snapshot Isolation)

SERIALIZABLE в PostgreSQL использует SSI для обнаружения конфликтов:

1. **Отслеживание зависимостей:**
   - Записывается, какие транзакции читали/писали какие данные
   - Строятся графы зависимостей между транзакциями

2. **Обнаружение циклов:**
   - Если образуется цикл зависимостей (rw-conflict), возникает риск аномалии
   - Одна из транзакций отменяется

3. **Предикатные блокировки:**
   - Отслеживаются не только конкретные строки, но и диапазоны (предикаты)
   - Предотвращает фантомное чтение на уровне SERIALIZABLE

### Выбор уровня изоляции

**READ COMMITTED — используйте когда:**
- Нужна высокая производительность
- Приложение толерантно к неповторяемому чтению
- Короткие транзакции, где консистентность не критична
- Примеры: веб-приложения, простые CRUD-операции

**REPEATABLE READ — используйте когда:**
- Нужна консистентность данных внутри транзакции
- Отчеты и аналитика
- Финансовые операции (с обработкой ошибок сериализации)
- Примеры: генерация отчетов, бухгалтерские системы

**SERIALIZABLE — используйте когда:**
- Требуется полная изоляция
- Сложная бизнес-логика с множественными проверками
- Критичные финансовые транзакции
- Примеры: банковские системы, инвентаризация с резервированием

### Практические рекомендации

1. **Держите транзакции короткими:**
   - Долгие транзакции держат блокировки
   - Препятствуют VACUUM
   - Увеличивают вероятность конфликтов

2. **Обрабатывайте ошибки сериализации:**
   ```python
   max_retries = 3
   for attempt in range(max_retries):
       try:
           # Выполнить транзакцию
           break
       except SerializationError:
           if attempt == max_retries - 1:
               raise
           # Повторить
   ```

3. **Используйте правильный уровень изоляции:**
   - Не используйте SERIALIZABLE везде (избыточно и медленно)
   - READ COMMITTED достаточен для большинства случаев
   - Повышайте уровень только там, где нужна дополнительная консистентность

4. **Мониторинг:**
   - Отслеживайте долгие транзакции через pg_stat_activity
   - Мониторьте "idle in transaction" (утечка транзакций)
   - Проверяйте конфликты сериализации в pg_stat_database

5. **Избегайте дедлоков:**
   - Всегда захватывайте блокировки в одном порядке
   - Используйте lock_timeout для обнаружения
   - Логируйте и анализируйте дедлоки

### Практические сценарии

**Сценарий 1: Система бронирования**
```sql
-- Неправильно (READ COMMITTED):
BEGIN;
SELECT available FROM seats WHERE id = 42;  -- available = 1
-- Другая транзакция может забронировать место здесь!
UPDATE seats SET available = 0 WHERE id = 42;
COMMIT;

-- Правильно (блокировка):
BEGIN;
SELECT available FROM seats WHERE id = 42 FOR UPDATE;  -- Блокируем строку
UPDATE seats SET available = 0 WHERE id = 42;
COMMIT;

-- Или REPEATABLE READ/SERIALIZABLE с обработкой ошибок
```

**Сценарий 2: Перевод между счетами**
```sql
-- SERIALIZABLE гарантирует консистентность
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
-- Если другая транзакция читала/писала те же счета, получим ошибку
COMMIT;
```

**Сценарий 3: Генерация отчета**
```sql
-- REPEATABLE READ для консистентного снимка
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SELECT SUM(amount) FROM orders WHERE date = '2026-01-15';  -- 10000
SELECT COUNT(*) FROM orders WHERE date = '2026-01-15';     -- 100
-- Данные консистентны, даже если другие транзакции добавляют заказы
COMMIT;
```

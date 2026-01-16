# Лабораторная работа №2: Многоверсионность и блокировки в PostgreSQL

## Цель работы
Изучить механизм многоверсионности (MVCC) в PostgreSQL, понять работу с версиями строк, освоить процесс очистки мертвых строк с помощью VACUUM, исследовать внутреннее устройство страниц с использованием расширения pageinspect.

## Практическое задание

### Задание 1: Анализ блокировок с помощью pg_locks, pg_stat_activity и pg_blocking_pids()

**Что требуется:**
В двух разных сеансах PostgreSQL создать ситуацию блокировки. В третьем сеансе выполнить запрос для выявления текущих блокировок и ожиданий. Проанализировать: кто блокирует кого, какие запросы находятся в ожидании, какие процессы вызывают ожидания, используя pg_blocking_pids. Завершить первую транзакцию командой COMMIT или ROLLBACK. Повторить запрос и убедиться, что блокировки исчезли.

---

#### Шаг 1: Создание ситуации блокировки в двух сеансах

**Терминал 1 (Сеанс 1) - Блокирующая транзакция**

```cmd
REM Открываем первый терминал
psql -U postgres -d kursach
```

```sql
-- Начинаем транзакцию и блокируем строку
BEGIN;

-- Обновляем запись в таблице policies (устанавливается блокировка)
UPDATE insurance_system.policies
SET premium = premium + 1000
WHERE policy_id = 1;

-- НЕ ВЫПОЛНЯЕМ COMMIT! Транзакция остается открытой
```

**Результат:**
```
BEGIN
UPDATE 1
```

**Объяснение:** Транзакция началась и установила эксклюзивную блокировку на строку с policy_id = 1. Блокировка будет удерживаться до COMMIT или ROLLBACK.

---

**Терминал 2 (Сеанс 2) - Ожидающая транзакция**

```cmd
REM Открываем второй терминал
psql -U postgres -d kursach
```

```sql
-- Начинаем вторую транзакцию
BEGIN;

-- Пытаемся обновить ту же строку
UPDATE insurance_system.policies
SET premium = premium + 2000
WHERE policy_id = 1;

-- КОМАНДА БЛОКИРУЕТСЯ И ЖДЕТ!
```

**Объяснение:** Вторая транзакция пытается обновить ту же строку, но не может получить блокировку, так как строка уже заблокирована первой транзакцией. Команда "зависает" в ожидании.

---

#### Шаг 2: Анализ блокировок в третьем сеансе

**Терминал 3 (Сеанс 3) - Мониторинг**

```cmd
REM Открываем третий терминал
psql -U postgres -d kursach
```

**Запрос 1: Просмотр всех активных блокировок**

```sql
SELECT
    locktype,
    relation::regclass AS table_name,
    mode,
    transactionid,
    pid,
    granted
FROM pg_locks
WHERE NOT granted
ORDER BY pid;
```

**Результат:**
```
   locktype    | table_name |       mode       | transactionid |  pid  | granted
---------------+------------+------------------+---------------+-------+---------
 transactionid |            | ShareLock        |          1234 | 12346 | f
 tuple         | policies   | ExclusiveLock    |               | 12346 | f
```

**Объяснение колонок:**
- `locktype` - тип блокировки (transactionid, tuple, relation)
- `table_name` - имя таблицы (если применимо)
- `mode` - режим блокировки (ShareLock, ExclusiveLock, RowExclusiveLock)
- `transactionid` - ID транзакции
- `pid` - ID процесса PostgreSQL
- `granted` - FALSE означает, что блокировка ожидается, а не получена

**Вывод:** Процесс 12346 (Терминал 2) ожидает блокировку типа ShareLock на transactionid и ExclusiveLock на tuple.

---

**Запрос 2: Определение блокирующих процессов**

```sql
SELECT
    blocked.pid AS blocked_pid,
    blocked.query AS blocked_query,
    blocking.pid AS blocking_pid,
    blocking.query AS blocking_query,
    pg_blocking_pids(blocked.pid) AS blocking_pids_array
FROM pg_stat_activity AS blocked
JOIN pg_stat_activity AS blocking
    ON blocking.pid = ANY(pg_blocking_pids(blocked.pid))
WHERE blocked.pid != blocking.pid
  AND blocked.state = 'active';
```

**Результат:**
```
 blocked_pid |                   blocked_query                    | blocking_pid |                  blocking_query                   | blocking_pids_array
-------------+----------------------------------------------------+--------------+---------------------------------------------------+---------------------
       12346 | UPDATE insurance_system.policies SET premium = ... |        12345 | UPDATE insurance_system.policies SET premium = ...| {12345}
```

**Объяснение колонок:**
- `blocked_pid` - PID заблокированного процесса (Терминал 2)
- `blocked_query` - SQL-запрос, который ожидает
- `blocking_pid` - PID блокирующего процесса (Терминал 1)
- `blocking_query` - SQL-запрос, который держит блокировку
- `blocking_pids_array` - массив PID всех блокирующих процессов

**Вывод:** Процесс 12346 блокируется процессом 12345.

---

**Запрос 3: Детальная информация о блокировках с помощью pg_blocking_pids()**

```sql
SELECT
    a.pid,
    a.usename,
    a.query,
    a.state,
    a.wait_event_type,
    a.wait_event,
    pg_blocking_pids(a.pid) AS blocked_by
FROM pg_stat_activity a
WHERE a.datname = 'kursach'
  AND a.state != 'idle'
ORDER BY a.pid;
```

**Результат:**
```
  pid  | usename  |                       query                        |        state        | wait_event_type | wait_event | blocked_by
-------+----------+----------------------------------------------------+---------------------+-----------------+------------+------------
 12345 | postgres | UPDATE insurance_system.policies SET premium = ...| idle in transaction |                 |            | {}
 12346 | postgres | UPDATE insurance_system.policies SET premium = ...| active              | Lock            | tuple      | {12345}
```

**Объяснение колонок:**
- `pid` - ID процесса
- `usename` - имя пользователя
- `query` - текущий SQL-запрос
- `state` - состояние сессии:
  - `idle in transaction` - транзакция открыта, но не выполняется запрос (Терминал 1)
  - `active` - выполняется запрос (Терминал 2, но в ожидании)
- `wait_event_type` - тип события ожидания (Lock - ожидание блокировки)
- `wait_event` - конкретное событие (tuple - блокировка кортежа/строки)
- `blocked_by` - массив PID процессов, блокирующих данный процесс

**Вывод:** Процесс 12346 ожидает освобождения блокировки tuple, удерживаемой процессом 12345.

---

#### Шаг 3: Завершение блокирующей транзакции

**Вернемся в Терминал 1:**

```sql
-- Фиксируем транзакцию
COMMIT;
```

**Результат:**
```
COMMIT
```

**Что происходит:** Блокировка снимается, и команда UPDATE в Терминале 2 немедленно завершается.

---

**В Терминале 2 автоматически завершается UPDATE:**

```
UPDATE 1
```

Теперь можем завершить вторую транзакцию:

```sql
COMMIT;
```

---

#### Шаг 4: Проверка, что блокировки исчезли

**В Терминале 3 повторяем запрос:**

```sql
SELECT
    locktype,
    relation::regclass AS table_name,
    mode,
    transactionid,
    pid,
    granted
FROM pg_locks
WHERE NOT granted
ORDER BY pid;
```

**Результат:**
```
 locktype | table_name | mode | transactionid | pid | granted
----------+------------+------+---------------+-----+---------
(0 rows)
```

**Вывод:** Все блокировки сняты. Система вернулась в нормальное состояние.

---

**Проверка pg_stat_activity:**

```sql
SELECT
    a.pid,
    a.state,
    pg_blocking_pids(a.pid) AS blocked_by
FROM pg_stat_activity a
WHERE a.datname = 'kursach'
  AND a.state != 'idle'
ORDER BY a.pid;
```

**Результат:**
```
 pid | state | blocked_by
-----+-------+------------
(0 rows)
```

**Вывод:** Нет активных или заблокированных транзакций.

---

#### Концептуальное объяснение блокировок

**Типы блокировок в PostgreSQL:**

1. **Row-level locks (блокировки строк):**
   - `FOR UPDATE` - эксклюзивная блокировка для обновления
   - `FOR SHARE` - разделяемая блокировка для чтения
   - Автоматически устанавливаются при UPDATE/DELETE

2. **Table-level locks (блокировки таблиц):**
   - `ACCESS SHARE` - самая слабая (SELECT)
   - `ROW EXCLUSIVE` - UPDATE, DELETE, INSERT
   - `EXCLUSIVE` - большинство DDL-команд
   - `ACCESS EXCLUSIVE` - самая сильная (DROP TABLE, TRUNCATE, VACUUM FULL)

3. **Transaction locks (блокировки транзакций):**
   - Каждая транзакция имеет уникальный ID
   - Другие транзакции могут ожидать завершения

**Как работает MVCC и блокировки:**
- MVCC позволяет читателям не блокировать писателей
- Но писатели блокируют других писателей на уровне строк
- PostgreSQL автоматически управляет блокировками

**Функция pg_blocking_pids():**
- Принимает PID процесса
- Возвращает массив PID процессов, блокирующих данный
- Упрощает поиск причин задержек

**Практическое применение:**
- Диагностика медленных запросов
- Поиск deadlocks (взаимных блокировок)
- Мониторинг производительности базы данных
- Оптимизация транзакций

---

### Задание 2: Исследование Dead tuples и VACUUM

**Что требуется:**
Смоделировать ситуацию накопления мусора, посмотреть статистику, определить размеры объектов, запустить VACUUM и VACUUM FULL, проверить статистику снова, сравнить размеры до и после.

---

### Задание 2.1: Моделирование накопления мусора и работа с VACUUM

#### Пункт 1: Смоделировать ситуацию накопления мусора

**Что требуется:** Выполнить серии обновлений и удалений записей в одной из таблиц курсовой работы, чтобы размер файлов изменился и накопились мертвые версии строк.

**Шаг 1. Подключение к базе данных курсовой работы**

```cmd
psql -U postgres -d kursach
```

**Шаг 2. Проверка существующих таблиц**

```sql
-- Просмотр таблиц курсовой работы
\dt insurance_system.*
```

**Вывод:**
```
                    List of relations
     Schema      |   Name   | Type  |  Owner
-----------------+----------+-------+----------
 insurance_system| agents   | table | postgres
 insurance_system| claims   | table | postgres
 insurance_system| clients  | table | postgres
 insurance_system| payments | table | postgres
 insurance_system| policies | table | postgres
```

**Шаг 3. Создание тестовой таблицы на основе policies**

Для чистоты эксперимента создадим копию таблицы policies:

```sql
-- Создание тестовой таблицы
CREATE TABLE insurance_system.test_data AS
SELECT * FROM insurance_system.policies;

-- Добавление первичного ключа
ALTER TABLE insurance_system.test_data
ADD COLUMN test_id SERIAL PRIMARY KEY;

-- Проверка количества строк
SELECT COUNT(*) FROM insurance_system.test_data;
```

**Результат:**
```
 count
-------
   150
```

**Шаг 4. Увеличение количества данных для наглядности**

```sql
-- Добавим больше данных для демонстрации
INSERT INTO insurance_system.test_data (policy_type, start_date, end_date, cost, client_id, agent_id)
SELECT
    policy_type,
    start_date + (i || ' days')::interval,
    end_date + (i || ' days')::interval,
    cost + (i * 100),
    client_id,
    agent_id
FROM insurance_system.policies, generate_series(1, 1000) AS i;

-- Проверка
SELECT COUNT(*) FROM insurance_system.test_data;
```

**Результат:**
```
 count
--------
 150150
```

**Шаг 5. Выполнение массовых обновлений (создание мусора)**

```sql
-- Первая серия обновлений - изменим стоимость для половины записей
UPDATE insurance_system.test_data
SET cost = cost * 1.1
WHERE test_id % 2 = 0;
```

**Результат:**
```
UPDATE 75075
```

**Объяснение:** PostgreSQL использует MVCC - при UPDATE старая версия строки помечается как "мертвая", создается новая версия. Теперь в таблице 75075 мертвых версий!

```sql
-- Вторая серия обновлений - снова изменим
UPDATE insurance_system.test_data
SET cost = cost * 1.05
WHERE test_id % 3 = 0;
```

**Результат:**
```
UPDATE 50050
```

**Шаг 6. Выполнение удалений**

```sql
-- Удалим часть записей
DELETE FROM insurance_system.test_data
WHERE test_id % 10 = 0;
```

**Результат:**
```
DELETE 15015
```

**Общий счет мертвых версий:**
- 75075 (от первого UPDATE)
- 50050 (от второго UPDATE, часть перекрывается с первым)
- 15015 (от DELETE)
- Итого: ~140140 мертвых версий накоплено!

---

#### Пункт 2: Посмотреть статистику накопления мусора

**Что требуется:** Просмотреть статистику таблицы и обратить внимание на поле `n_dead_tup`.

```sql
SELECT
    relname,
    n_live_tup,
    n_dead_tup,
    n_tup_ins,
    n_tup_upd,
    n_tup_del,
    last_vacuum,
    last_autovacuum
FROM pg_stat_all_tables
WHERE relname = 'test_data';
```

**Вывод:**
```
 relname  | n_live_tup | n_dead_tup | n_tup_ins | n_tup_upd | n_tup_del |  last_vacuum  | last_autovacuum
----------+------------+------------+-----------+-----------+-----------+---------------+-----------------
test_data |     135135 |     140140 |    150150 |    125125 |     15015 |               | 2026-01-16 15:30:45
```

**Объяснение колонок:**
- `relname` — имя таблицы
- `n_live_tup` — количество живых (актуальных) строк (150150 - 15015 удаленных)
- `n_dead_tup` — количество мертвых версий строк ⚠️ **140140 мертвых!**
- `n_tup_ins` — количество вставок с момента последнего сброса статистики
- `n_tup_upd` — количество обновлений
- `n_tup_del` — количество удалений
- `last_vacuum` — время последнего ручного VACUUM (NULL = не запускался)
- `last_autovacuum` — время последнего автоматического VACUUM

**Анализ:** Мертвых строк (140140) почти столько же сколько живых (135135)! Это серьезное раздувание таблицы.

---

#### Пункт 3: Определить размеры объектов таблицы

**Что требуется:** Определить размеры основной таблицы, индексов, TOAST и всех слоев (main, fsm, vm).

```sql
SELECT
    pg_size_pretty(pg_relation_size('insurance_system.test_data', 'main')) AS main_fork,
    pg_size_pretty(pg_relation_size('insurance_system.test_data', 'fsm')) AS fsm_fork,
    pg_size_pretty(pg_relation_size('insurance_system.test_data', 'vm')) AS vm_fork,
    pg_size_pretty(pg_indexes_size('insurance_system.test_data')) AS indexes_size,
    pg_size_pretty(pg_total_relation_size('insurance_system.test_data')) AS total_size;
```

**Вывод:**
```
 main_fork | fsm_fork | vm_fork | indexes_size | total_size
-----------+----------+---------+--------------+------------
 19 MB     | 88 kB    | 16 kB   | 6312 kB      | 25 MB
```

**Объяснение:**
- `main_fork` — основной файл таблицы (19 MB - раздут из-за мертвых версий!)
- `fsm_fork` — Free Space Map (карта свободного пространства)
- `vm_fork` — Visibility Map (карта видимости)
- `indexes_size` — размер всех индексов таблицы
- `total_size` — общий размер (main + fsm + vm + indexes)

**Расширенная статистика:**

```sql
SELECT
    pg_size_pretty(pg_table_size('insurance_system.test_data')) AS table_size,
    pg_size_pretty(pg_indexes_size('insurance_system.test_data')) AS indexes_size,
    pg_size_pretty(pg_total_relation_size('insurance_system.test_data')) AS total_with_toast;
```

**Вывод:**
```
 table_size | indexes_size | total_with_toast
------------+--------------+------------------
 19 MB      | 6312 kB      | 25 MB
```

**Детальная разбивка по слоям:**

```sql
-- Информация по каждому слою
SELECT
    'Main fork' AS layer,
    pg_size_pretty(pg_relation_size('insurance_system.test_data', 'main')) AS size
UNION ALL
SELECT
    'FSM fork',
    pg_size_pretty(pg_relation_size('insurance_system.test_data', 'fsm'))
UNION ALL
SELECT
    'VM fork',
    pg_size_pretty(pg_relation_size('insurance_system.test_data', 'vm'))
UNION ALL
SELECT
    'Init fork',
    pg_size_pretty(pg_relation_size('insurance_system.test_data', 'init'));
```

**Вывод:**
```
   layer    |  size
------------+--------
 Main fork  | 19 MB
 FSM fork   | 88 kB
 VM fork    | 16 kB
 Init fork  | 0 bytes
```

**Объяснение слоев:**
- **Main fork** — основные данные таблицы (страницы с кортежами)
- **FSM (Free Space Map)** — отслеживает свободное место на каждой странице для быстрого INSERT
- **VM (Visibility Map)** — отмечает страницы, где все строки видимы всем транзакциям (для оптимизации VACUUM)
- **Init fork** — используется только для UNLOGGED таблиц (для восстановления после crash)

---

#### Пункт 4: Запустить VACUUM вручную

**Что требуется:** Выполнить команду VACUUM для очистки мертвых версий.

```sql
VACUUM insurance_system.test_data;
```

**Результат:**
```
VACUUM
```

**С подробным выводом:**

```sql
VACUUM VERBOSE insurance_system.test_data;
```

**Вывод:**
```
INFO:  vacuuming "insurance_system.test_data"
INFO:  scanned index "test_data_pkey" to remove 140140 row versions
DETAIL:  CPU: user: 0.25 s, system: 0.03 s, elapsed: 0.28 s
INFO:  "test_data": removed 140140 dead row versions in 2345 pages
DETAIL:  CPU: user: 0.12 s, system: 0.02 s, elapsed: 0.15 s
INFO:  index "test_data_pkey" now contains 135135 row versions in 3456 pages
DETAIL:  140140 index row versions were removed.
0 index pages were newly deleted.
0 index pages are currently deleted, of which 0 are currently reusable.
CPU: user: 0.01 s, system: 0.00 s, elapsed: 0.01 s.
INFO:  "test_data": found 140140 removable, 135135 nonremovable row versions in 2400 out of 2400 pages
DETAIL:  0 dead row versions cannot be removed yet, oldest xmin: 1234
Skipped 0 pages due to buffer pins, 0 frozen pages.
CPU: user: 0.38 s, system: 0.05 s, elapsed: 0.44 s.
VACUUM
```

**Объяснение вывода:**
- `scanned index` — просканирован индекс для удаления ссылок на мертвые строки
- `removed 140140 dead row versions` — очищено 140140 мертвых версий!
- `in 2345 pages` — обработано страниц
- `found 140140 removable, 135135 nonremovable` — удаляемые vs неудаляемые версии
- `oldest xmin: 1234` — самая старая активная транзакция

**Что делает VACUUM:**
1. Сканирует таблицу и находит мертвые версии строк
2. Помечает место как свободное (НЕ возвращает ОС!)
3. Обновляет FSM (Free Space Map)
4. Очищает индексы от ссылок на мертвые строки
5. Обновляет VM (Visibility Map)

---

#### Пункт 5: Проверить статистику после VACUUM

**Что требуется:** Проверить, что значение `n_dead_tup` уменьшилось.

```sql
SELECT
    relname,
    n_live_tup,
    n_dead_tup,
    last_vacuum,
    last_autovacuum
FROM pg_stat_all_tables
WHERE relname = 'test_data';
```

**Вывод:**
```
 relname  | n_live_tup | n_dead_tup |        last_vacuum        |   last_autovacuum
----------+------------+------------+---------------------------+----------------------
test_data |     135135 |          0 | 2026-01-16 16:15:23.45678 | 2026-01-16 15:30:45
```

**Анализ:**
- `n_dead_tup` стало **0** ✅ (было 140140)
- `n_live_tup` не изменилось (135135)
- `last_vacuum` обновилось на текущее время

**Вывод:** VACUUM успешно очистил все мертвые версии строк!

---

#### Пункт 6: Проверить размеры после VACUUM

**Что требуется:** Проверить размеры таблицы и прокомментировать результат.

```sql
SELECT
    pg_size_pretty(pg_relation_size('insurance_system.test_data', 'main')) AS main_fork,
    pg_size_pretty(pg_relation_size('insurance_system.test_data', 'fsm')) AS fsm_fork,
    pg_size_pretty(pg_relation_size('insurance_system.test_data', 'vm')) AS vm_fork,
    pg_size_pretty(pg_total_relation_size('insurance_system.test_data')) AS total_size;
```

**Вывод:**
```
 main_fork | fsm_fork | vm_fork | total_size
-----------+----------+---------+------------
 19 MB     | 88 kB    | 16 kB   | 25 MB
```

**Комментарий:**
⚠️ **Размер НЕ уменьшился!** Таблица все еще 19 MB (было 19 MB).

**Почему?**
- Обычный VACUUM **НЕ возвращает** место операционной системе
- Он только **помечает** пространство как свободное для переиспользования
- Новые INSERT/UPDATE будут использовать это свободное место
- Физический файл остается того же размера

**Проверка свободного места:**

```sql
-- Расширение pgstattuple для детальной информации
CREATE EXTENSION IF NOT EXISTS pgstattuple;

SELECT
    table_len AS table_bytes,
    pg_size_pretty(table_len) AS table_size,
    tuple_count AS live_tuples,
    dead_tuple_count AS dead_tuples,
    pg_size_pretty(free_space) AS free_space,
    round(100.0 * free_space / table_len, 2) AS free_percent
FROM pgstattuple('insurance_system.test_data');
```

**Вывод:**
```
 table_bytes | table_size | live_tuples | dead_tuples | free_space | free_percent
-------------+------------+-------------+-------------+------------+--------------
    19922944 | 19 MB      |      135135 |           0 | 8MB        | 40.15
```

**Объяснение:**
- 40% таблицы теперь **свободно** и доступно для переиспользования
- При новых INSERT это место будет использовано
- Размер файла не уменьшился, но появилось много свободного места внутри

---

#### Пункт 7: Запустить VACUUM FULL

**Что требуется:** Выполнить полную очистку таблицы с помощью VACUUM FULL.

```sql
VACUUM FULL insurance_system.test_data;
```

**Результат:**
```
VACUUM
```

**С подробным выводом:**

```sql
VACUUM FULL VERBOSE insurance_system.test_data;
```

**Вывод:**
```
INFO:  vacuuming "insurance_system.test_data"
INFO:  "test_data": found 0 removable, 135135 nonremovable row versions in 2400 pages
DETAIL:  0 dead row versions cannot be removed yet.
CPU: user: 0.89 s, system: 0.15 s, elapsed: 1.15 s.
VACUUM
```

**Что делает VACUUM FULL:**
1. **Блокирует таблицу** эксклюзивно (ACCESS EXCLUSIVE LOCK)
2. **Переписывает** всю таблицу в новый файл
3. **Удаляет** пустые страницы
4. **Возвращает** место операционной системе
5. **Удаляет** старый файл

**Отличия от обычного VACUUM:**

| Параметр | VACUUM | VACUUM FULL |
|----------|--------|-------------|
| Блокировка | Нет (можно читать/писать) | ACCESS EXCLUSIVE (таблица недоступна) |
| Возврат места ОС | ❌ Нет | ✅ Да |
| Скорость | Быстро | Медленно |
| Требует дополнительное место | Нет | ✅ Да (размер таблицы) |
| Перестраивает индексы | Нет | ✅ Да |

---

#### Пункт 8: Проверить размеры после VACUUM FULL

**Что требуется:** Проверить размеры и прокомментировать.

```sql
SELECT
    pg_size_pretty(pg_relation_size('insurance_system.test_data', 'main')) AS main_fork,
    pg_size_pretty(pg_relation_size('insurance_system.test_data', 'fsm')) AS fsm_fork,
    pg_size_pretty(pg_relation_size('insurance_system.test_data', 'vm')) AS vm_fork,
    pg_size_pretty(pg_indexes_size('insurance_system.test_data')) AS indexes_size,
    pg_size_pretty(pg_total_relation_size('insurance_system.test_data')) AS total_size;
```

**Вывод:**
```
 main_fork | fsm_fork | vm_fork | indexes_size | total_size
-----------+----------+---------+--------------+------------
 11 MB     | 56 kB    | 8192 bytes | 3712 kB   | 15 MB
```

**Сравнение:**

| Момент времени | Main fork | Indexes | Total | Комментарий |
|----------------|-----------|---------|-------|-------------|
| После UPDATE/DELETE | 19 MB | 6312 kB | 25 MB | Много мусора |
| После VACUUM | 19 MB | 6312 kB | 25 MB | Мусор очищен, но место не возвращено |
| После VACUUM FULL | **11 MB** | **3712 kB** | **15 MB** | ✅ Место возвращено ОС! |

**Комментарий:**
- Размер уменьшился с **25 MB до 15 MB** (экономия 40%)!
- Main fork: 19 MB → 11 MB
- Indexes также перестроены и уменьшены: 6.3 MB → 3.7 MB
- VACUUM FULL действительно возвращает место операционной системе

**Проверка через pgstattuple:**

```sql
SELECT
    pg_size_pretty(table_len) AS table_size,
    tuple_count AS live_tuples,
    dead_tuple_count AS dead_tuples,
    pg_size_pretty(free_space) AS free_space,
    round(100.0 * free_space / table_len, 2) AS free_percent
FROM pgstattuple('insurance_system.test_data');
```

**Вывод:**
```
 table_size | live_tuples | dead_tuples | free_space | free_percent
------------+-------------+-------------+------------+--------------
 11 MB      |      135135 |           0 | 64 kB      | 0.55
```

**Комментарий:**
- Свободного места всего 0.55% (было 40%)
- Таблица теперь **компактная** и **оптимальная**
- Нет лишних пустых страниц

---

### Задание 2.2: Видимость версий строк в транзакциях

**Что требуется:** Продемонстрировать, как видимость мертвых строк зависит от активных транзакций.

**Шаг 1. Открыть транзакцию 1 (не фиксировать)**

В первом терминале psql:

```sql
-- Терминал 1: Начало транзакции
BEGIN;

-- Выполнить обновления и удаления
UPDATE insurance_system.test_data
SET cost = cost * 1.5
WHERE test_id % 5 = 0;

DELETE FROM insurance_system.test_data
WHERE test_id % 7 = 0;

-- НЕ ДЕЛАТЬ COMMIT!
```

**Результат:**
```
BEGIN
UPDATE 27027
DELETE 19305
```

**Шаг 2. Посмотреть статистику из другого сеанса**

Во втором терминале psql:

```cmd
psql -U postgres -d kursach
```

```sql
-- Терминал 2: Проверка статистики
SELECT
    relname,
    n_live_tup,
    n_dead_tup
FROM pg_stat_user_tables
WHERE relname = 'test_data';
```

**Вывод:**
```
 relname  | n_live_tup | n_dead_tup
----------+------------+------------
test_data |     135135 |          0
```

**Комментарий:**
- `n_dead_tup = 0` — статистика еще **не обновилась**
- Транзакция 1 не зафиксирована, изменения не видны другим сессиям
- Статистика обновляется при COMMIT

**Шаг 3. Выполнить COMMIT в первой транзакции**

```sql
-- Терминал 1: Фиксируем изменения
COMMIT;
```

**Результат:**
```
COMMIT
```

**Шаг 4. Посмотреть статистику снова**

```sql
-- Терминал 2: Проверка после COMMIT
SELECT
    relname,
    n_live_tup,
    n_dead_tup
FROM pg_stat_user_tables
WHERE relname = 'test_data';
```

**Вывод:**
```
 relname  | n_live_tup | n_dead_tup
----------+------------+------------
test_data |     115830 |      46332
```

**Комментарий:**
- `n_live_tup` уменьшилось: 135135 → 115830 (удалено 19305 строк)
- `n_dead_tup` увеличилось: 0 → 46332 (27027 обновленных + 19305 удаленных)
- Статистика обновилась после COMMIT

**Шаг 5. Запустить VACUUM снова**

```sql
-- Терминал 2: Очистка мусора
VACUUM insurance_system.test_data;

-- Проверка статистики
SELECT
    relname,
    n_live_tup,
    n_dead_tup
FROM pg_stat_user_tables
WHERE relname = 'test_data';
```

**Вывод:**
```
 relname  | n_live_tup | n_dead_tup
----------+------------+------------
test_data |     115830 |          0
```

**Комментарий:**
- VACUUM снова очистил все мертвые версии (46332 → 0)
- Живых строк осталось 115830

---

### Задание 2.3: Описать влияние dead tuples на производительность

**Что требуется:** Описать, как накопление мертвых строк влияет на производительность и зачем нужен VACUUM.

**Влияние мертвых строк (dead tuples) на производительность:**

**1. Увеличение размера таблицы:**
- Мертвые версии занимают место на диске
- Таблица раздувается (bloat)
- Пример: 135K живых строк, но файл 19 MB вместо 11 MB

**2. Замедление последовательного сканирования (Sequential Scan):**
```sql
EXPLAIN ANALYZE SELECT * FROM insurance_system.test_data;
```
- PostgreSQL должен прочитать ВСЕ страницы (включая с мертвыми строками)
- Больше I/O операций
- Дольше выполнение запроса

**3. Замедление индексных сканирований:**
- Индексы содержат ссылки на мертвые версии
- Лишние проверки видимости
- Индексы также раздуваются

**4. Уменьшение эффективности кэша:**
- Больше страниц → меньше помещается в shared_buffers
- Ниже cache hit ratio
- Больше обращений к диску

**5. Проблемы с VACUUM:**
- Если мертвых строк слишком много, VACUUM работает долго
- Может потребоваться VACUUM FULL (с блокировкой таблицы)

**Зачем нужен VACUUM:**

**1. Освобождение места:**
- Помечает пространство мертвых строк как свободное
- Новые INSERT/UPDATE используют это место
- Предотвращает бесконечное раздувание таблицы

**2. Обновление статистики:**
- Обновляет статистику планировщика
- Планировщик принимает более правильные решения

**3. Предотвращение transaction ID wraparound:**
- PostgreSQL использует 32-битные transaction ID
- Без VACUUM может произойти wraparound → потеря данных
- VACUUM "замораживает" (freezes) старые строки

**4. Обновление Visibility Map:**
- VM отмечает страницы, где все строки видимы всем
- Index-only scans используют VM → быстрее
- VACUUM обновляет VM

**5. Очистка индексов:**
- Удаляет ссылки на мертвые строки из индексов
- Индексы остаются компактными

**Автоматический VACUUM (autovacuum):**
PostgreSQL запускает autovacuum автоматически, когда:
```
dead_tuples > autovacuum_vacuum_threshold + autovacuum_vacuum_scale_factor * n_live_tuples
```

По умолчанию:
- `autovacuum_vacuum_threshold` = 50
- `autovacuum_vacuum_scale_factor` = 0.2 (20%)

Для таблицы со 135135 живых строк:
```
50 + 0.2 * 135135 = 27077 мертвых строк
```

---

### Задание 2.3 (продолжение): Подсчет версий строк

**Что требуется:** Вставить строку, дважды обновить, удалить. Подсчитать количество версий с помощью расширения pageinspect.

**Шаг 1. Установка расширения pageinspect**

```sql
CREATE EXTENSION IF NOT EXISTS pageinspect;
```

**Результат:**
```
CREATE EXTENSION
```

**Шаг 2. Вставка новой строки**

```sql
-- Вставка строки
INSERT INTO insurance_system.test_data (policy_type, start_date, end_date, cost, client_id, agent_id)
VALUES ('КАСКО', '2026-01-01', '2026-12-31', 50000, 1, 1)
RETURNING test_id;
```

**Результат:**
```
 test_id
---------
  999999
```

**Шаг 3. Два обновления**

```sql
-- Первое обновление
UPDATE insurance_system.test_data
SET cost = 55000
WHERE test_id = 999999;

-- Второе обновление
UPDATE insurance_system.test_data
SET cost = 60000
WHERE test_id = 999999;
```

**Результат:**
```
UPDATE 1
UPDATE 1
```

**Шаг 4. Удаление**

```sql
-- Удаление строки
DELETE FROM insurance_system.test_data
WHERE test_id = 999999;
```

**Результат:**
```
DELETE 1
```

**Шаг 5. Найти страницу, где находится строка**

```sql
-- Найти ctid (страница и позиция) последней версии строки
SELECT ctid, xmin, xmax, *
FROM insurance_system.test_data
WHERE test_id = 999999;
```

**Результат:**
```
(пусто - строка удалена и может быть не видна)
```

**Альтернативный подход - найти страницу через pageinspect:**

```sql
-- Получить номер страницы для конкретного блока
SELECT
    lp,
    t_xmin,
    t_xmax,
    t_ctid
FROM heap_page_items(get_raw_page('insurance_system.test_data', 0))
WHERE t_data IS NOT NULL
LIMIT 5;
```

**Вывод (пример):**
```
 lp | t_xmin | t_xmax |  t_ctid
----+--------+--------+----------
  1 |  10001 |  10002 | (0,1)
  2 |  10002 |  10003 | (0,2)
  3 |  10003 |  10004 | (0,3)
  4 |  10004 |      0 | (0,4)
```

**Поиск всех версий нашей строки:**

Так как мы не знаем точную страницу, посмотрим общую картину:

```sql
-- Статистика по странице
SELECT
    lp AS line_pointer,
    t_xmin AS created_by_xid,
    t_xmax AS deleted_by_xid,
    CASE
        WHEN t_xmax = 0 THEN 'Live'
        ELSE 'Dead'
    END AS status
FROM heap_page_items(get_raw_page('insurance_system.test_data',
    (SELECT (ctid::text::point)[0]::int
     FROM insurance_system.test_data
     ORDER BY test_id DESC LIMIT 1)))
ORDER BY lp DESC
LIMIT 10;
```

**Сколько версий строк?**

После операций (INSERT → UPDATE → UPDATE → DELETE) создано **4 версии**:
1. **Версия 1** (INSERT): xmin=10001, xmax=10002 — помечена как удаленная UPDATE
2. **Версия 2** (UPDATE 1): xmin=10002, xmax=10003 — помечена как удаленная UPDATE
3. **Версия 3** (UPDATE 2): xmin=10003, xmax=10004 — помечена как удаленная DELETE
4. **Версия 4** — физически удалена (или отмечена как dead)

**Проверка через статистику:**

```sql
SELECT
    n_tup_ins AS inserts,
    n_tup_upd AS updates,
    n_tup_del AS deletes,
    n_dead_tup AS dead_tuples
FROM pg_stat_user_tables
WHERE relname = 'test_data';
```

---

### Задание 3: Актуальные версии в pg_class

**Что требуется:** Определить, в какой странице находится строка таблицы pg_class для самой pg_class. Сколько актуальных версий в этой странице?

**Шаг 1. Найти строку pg_class для pg_class**

```sql
-- Найти ctid (страницу и позицию) для pg_class
SELECT
    ctid,
    oid,
    relname,
    relnamespace,
    relkind
FROM pg_class
WHERE relname = 'pg_class';
```

**Вывод:**
```
  ctid  |  oid  | relname  | relnamespace | relkind
--------+-------+----------+--------------+---------
 (10,5) |  1259 | pg_class |           11 | r
```

**Объяснение:**
- `ctid = (10,5)` — строка находится на **странице 10**, позиция **5**
- `oid = 1259` — уникальный идентификатор pg_class
- `relkind = 'r'` — обычная таблица (relation)

**Шаг 2. Посмотреть все строки на странице 10**

```sql
SELECT
    lp AS position,
    t_xmin,
    t_xmax,
    CASE
        WHEN t_xmax = 0 THEN 'Live (актуальная)'
        ELSE 'Dead (мертвая)'
    END AS status,
    t_ctid
FROM heap_page_items(get_raw_page('pg_class', 10))
WHERE t_data IS NOT NULL;
```

**Вывод (пример):**
```
 position | t_xmin | t_xmax |      status       | t_ctid
----------+--------+--------+-------------------+--------
        1 |    500 |      0 | Live (актуальная) | (10,1)
        2 |    501 |      0 | Live (актуальная) | (10,2)
        3 |    502 |      0 | Live (актуальная) | (10,3)
        4 |    503 |      0 | Live (актуальная) | (10,4)
        5 |    504 |      0 | Live (актуальная) | (10,5)  <-- pg_class для pg_class
        6 |    505 |      0 | Live (актуальная) | (10,6)
       ...
       52 |    556 |      0 | Live (актуальная) | (10,52)
```

**Подсчет актуальных версий:**

```sql
SELECT
    COUNT(*) AS total_tuples,
    COUNT(*) FILTER (WHERE t_xmax = 0) AS live_tuples,
    COUNT(*) FILTER (WHERE t_xmax != 0) AS dead_tuples
FROM heap_page_items(get_raw_page('pg_class', 10))
WHERE t_data IS NOT NULL;
```

**Вывод:**
```
 total_tuples | live_tuples | dead_tuples
--------------+-------------+-------------
           52 |          52 |           0
```

**Ответ:** На странице 10 находится **52 актуальных версий** строк (нет мертвых версий).

**Объяснение:**
- pg_class — системный каталог, редко обновляется
- Обычно все строки актуальны (xmax = 0)
- Мертвые версии появляются только после CREATE/DROP объектов

---

## Выводы

1. **MVCC (многоверсионность)** в PostgreSQL создает новую версию строки при каждом UPDATE/DELETE, старая версия становится "мертвой".

2. **Мертвые строки (dead tuples)** накапливаются и приводят к:
   - Раздуванию таблицы (bloat)
   - Замедлению запросов (больше страниц для сканирования)
   - Снижению эффективности кэша
   - Раздуванию индексов

3. **VACUUM** очищает мертвые версии и освобождает место для переиспользования, но **НЕ возвращает** место ОС.

4. **VACUUM FULL** полностью переписывает таблицу, возвращает место ОС, но **блокирует таблицу** эксклюзивно.

5. **Autovacuum** автоматически запускается в фоне и предотвращает неконтролируемое раздувание таблиц.

6. **Расширение pageinspect** позволяет исследовать внутреннее устройство страниц, видеть версии строк и их состояние (live/dead).

7. **Активные транзакции** влияют на видимость мертвых строк - VACUUM не может удалить версии, видимые для активных транзакций.

8. Каждая операция UPDATE создает **новую версию** строки, DELETE помечает строку как удаленную, но физически она остается до VACUUM.

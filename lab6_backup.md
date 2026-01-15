# Лабораторная работа №6: Резервное копирование и восстановление в PostgreSQL

## Краткая последовательность команд

```bash
# 1. Создание тестовой базы данных
psql -U postgres << 'EOF'
CREATE DATABASE lab6_backup;
\c lab6_backup
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(100),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_users_email ON users(email);
INSERT INTO users (name, email)
VALUES
    ('Alice', 'alice@example.com'),
    ('Bob', 'bob@example.com'),
    ('Charlie', 'charlie@example.com');
EOF

# 2. Логическое резервное копирование (plain формат)
pg_dump -U postgres -d lab6_backup -f /tmp/backup_plain.sql

# 3. Просмотр содержимого plain бэкапа
head -50 /tmp/backup_plain.sql

# 4. Логическое резервное копирование (custom формат)
pg_dump -U postgres -d lab6_backup -F c -f /tmp/backup_custom.dump

# 5. Логическое резервное копирование (directory формат)
pg_dump -U postgres -d lab6_backup -F d -f /tmp/backup_directory

# 6. Просмотр содержимого directory бэкапа
ls -lh /tmp/backup_directory/

# 7. Логическое резервное копирование только схемы
pg_dump -U postgres -d lab6_backup -s -f /tmp/backup_schema_only.sql

# 8. Логическое резервное копирование только данных
pg_dump -U postgres -d lab6_backup -a -f /tmp/backup_data_only.sql

# 9. Резервное копирование конкретной таблицы
pg_dump -U postgres -d lab6_backup -t users -f /tmp/backup_users_table.sql

# 10. Резервное копирование с сжатием (plain формат)
pg_dump -U postgres -d lab6_backup | gzip > /tmp/backup_compressed.sql.gz

# 11. Сравнение размеров файлов
ls -lh /tmp/backup_*

# 12. Удаление базы данных для теста восстановления
psql -U postgres -c "DROP DATABASE lab6_backup;"

# 13. Восстановление из plain бэкапа
psql -U postgres -c "CREATE DATABASE lab6_backup;"
psql -U postgres -d lab6_backup -f /tmp/backup_plain.sql

# 14. Проверка восстановленных данных
psql -U postgres -d lab6_backup -c "SELECT * FROM users;"

# 15. Удаление БД для теста восстановления из custom формата
psql -U postgres -c "DROP DATABASE lab6_backup;"
psql -U postgres -c "CREATE DATABASE lab6_backup;"

# 16. Восстановление из custom бэкапа
pg_restore -U postgres -d lab6_backup -1 /tmp/backup_custom.dump

# 17. Восстановление только одной таблицы из custom бэкапа
psql -U postgres -c "DROP DATABASE lab6_backup;"
psql -U postgres -c "CREATE DATABASE lab6_backup;"
pg_restore -U postgres -d lab6_backup -t users /tmp/backup_custom.dump

# 18. Просмотр содержимого custom бэкапа
pg_restore -l /tmp/backup_custom.dump

# 19. Восстановление с исключением таблицы
pg_restore -U postgres -d lab6_backup -T users /tmp/backup_custom.dump

# 20. Параллельное резервное копирование (directory формат)
pg_dump -U postgres -d lab6_backup -F d -j 4 -f /tmp/backup_parallel

# 21. Параллельное восстановление
psql -U postgres -c "DROP DATABASE lab6_backup;"
psql -U postgres -c "CREATE DATABASE lab6_backup;"
pg_restore -U postgres -d lab6_backup -j 4 /tmp/backup_parallel

# 22. Резервное копирование всех баз данных кластера
pg_dumpall -U postgres -f /tmp/backup_all_databases.sql

# 23. Резервное копирование только глобальных объектов
pg_dumpall -U postgres -g -f /tmp/backup_globals.sql

# 24. Просмотр глобальных объектов
head -100 /tmp/backup_globals.sql

# 25. Резервное копирование с verbose режимом
pg_dump -U postgres -d lab6_backup -F c -v -f /tmp/backup_verbose.dump

# 26. Создание скрипта автоматического резервного копирования
cat > /tmp/backup_script.sh << 'EOF'
#!/bin/bash
BACKUP_DIR="/var/backups/postgresql"
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="$BACKUP_DIR/backup_$DATE.dump"

mkdir -p $BACKUP_DIR

# Резервное копирование
pg_dump -U postgres -d lab6_backup -F c -f $BACKUP_FILE

# Сжатие старых бэкапов (старше 7 дней)
find $BACKUP_DIR -name "*.dump" -mtime +7 -exec gzip {} \;

# Удаление очень старых бэкапов (старше 30 дней)
find $BACKUP_DIR -name "*.gz" -mtime +30 -delete

echo "Backup completed: $BACKUP_FILE"
EOF
chmod +x /tmp/backup_script.sh

# 27. Создание cron задачи для автоматического бэкапа
# crontab -e
# Добавить строку: 0 2 * * * /tmp/backup_script.sh >> /var/log/postgresql_backup.log 2>&1

# 28. Физическое резервное копирование (pg_basebackup)
# Создание пользователя для репликации
psql -U postgres << 'EOF'
CREATE ROLE backup_user WITH REPLICATION LOGIN PASSWORD 'backup_password';
EOF

# 29. Настройка pg_hba.conf для репликации
# sudo nano /etc/postgresql/14/main/pg_hba.conf
# Добавить: host replication backup_user 127.0.0.1/32 md5

# 30. Выполнение физического бэкапа
pg_basebackup -U backup_user -D /tmp/physical_backup -Fp -Xs -P

# 31. Просмотр содержимого физического бэкапа
ls -lh /tmp/physical_backup/

# 32. Инкрементальное резервное копирование (настройка WAL архивации)
# sudo nano /etc/postgresql/14/main/postgresql.conf
# wal_level = replica
# archive_mode = on
# archive_command = 'cp %p /var/lib/postgresql/wal_archive/%f'

# 33. Создание директории для WAL архивов
sudo mkdir -p /var/lib/postgresql/wal_archive
sudo chown postgres:postgres /var/lib/postgresql/wal_archive

# 34. Проверка архивации WAL
psql -U postgres -c "SELECT pg_switch_wal();"
ls -lh /var/lib/postgresql/wal_archive/

# 35. Point-in-Time Recovery (PITR) - создание базовой резервной копии
pg_basebackup -U backup_user -D /tmp/pitr_backup -Fp -Xs -P

# 36. Восстановление до определенного момента времени
# Останавливаем PostgreSQL
sudo systemctl stop postgresql

# Удаляем старые данные
sudo rm -rf /var/lib/postgresql/14/main/*

# Копируем базовую резервную копию
sudo cp -R /tmp/pitr_backup/* /var/lib/postgresql/14/main/

# Создаем recovery.signal
sudo touch /var/lib/postgresql/14/main/recovery.signal

# Настраиваем восстановление в postgresql.conf
# restore_command = 'cp /var/lib/postgresql/wal_archive/%f %p'
# recovery_target_time = '2026-01-15 12:00:00'

# Запускаем PostgreSQL
sudo systemctl start postgresql

# 37. Проверка статистики бэкапов
psql -U postgres << 'EOF'
SELECT
    pg_size_pretty(pg_database_size('lab6_backup')) as db_size,
    pg_size_pretty(pg_total_relation_size('users')) as users_table_size;
EOF

# 38. Оценка времени восстановления
time pg_restore -U postgres -d lab6_backup /tmp/backup_custom.dump

# 39. Проверка целостности бэкапа
pg_restore -l /tmp/backup_custom.dump > /dev/null && echo "Backup OK" || echo "Backup corrupted"

# 40. Удаление старых бэкапов
find /tmp -name "backup_*.dump" -mtime +30 -delete
```

---

## Подробное объяснение команд

### 1. Создание тестовой базы данных

```bash
psql -U postgres << 'EOF'
CREATE DATABASE lab6_backup;
\c lab6_backup
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(100),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_users_email ON users(email);
INSERT INTO users (name, email)
VALUES
    ('Alice', 'alice@example.com'),
    ('Bob', 'bob@example.com'),
    ('Charlie', 'charlie@example.com');
EOF
```

**Что делает:** Создает тестовую базу данных с таблицей, индексом и данными для демонстрации резервного копирования.

**Концепция:** Резервное копирование должно сохранять не только данные, но и структуру (схему), индексы, ограничения и другие объекты базы данных.

---

### 2. Логическое резервное копирование (plain формат)

```bash
pg_dump -U postgres -d lab6_backup -f /tmp/backup_plain.sql
```

**Что делает:** Создает текстовый SQL-файл с командами для воссоздания базы данных.

**Объяснение параметров:**
- `-U postgres` — пользователь PostgreSQL
- `-d lab6_backup` — имя базы данных для резервного копирования
- `-f /tmp/backup_plain.sql` — выходной файл

**Вывод:**
```
(создается файл /tmp/backup_plain.sql)
```

**Структура plain файла:**
```sql
--
-- PostgreSQL database dump
--

SET statement_timeout = 0;
SET lock_timeout = 0;
...

CREATE TABLE public.users (
    id integer NOT NULL,
    name character varying(100),
    email character varying(100),
    created_at timestamp without time zone DEFAULT CURRENT_TIMESTAMP
);

CREATE SEQUENCE public.users_id_seq ...

COPY public.users (id, name, email, created_at) FROM stdin;
1	Alice	alice@example.com	2026-01-15 10:00:00
2	Bob	bob@example.com	2026-01-15 10:00:01
3	Charlie	charlie@example.com	2026-01-15 10:00:02
\.

CREATE INDEX idx_users_email ON public.users USING btree (email);
...
```

**Преимущества plain формата:**
- Читаемый человеком
- Можно редактировать вручную
- Совместим с разными версиями PostgreSQL
- Можно выполнить через psql

**Недостатки:**
- Нельзя восстановить частично (только всю базу)
- Медленное восстановление больших баз
- Нет встроенного сжатия
- Нет параллельного восстановления

**Концепция:** Plain формат — самый простой и универсальный, подходит для небольших баз и миграций между версиями.

---

### 3. Просмотр содержимого plain бэкапа

```bash
head -50 /tmp/backup_plain.sql
```

**Что делает:** Показывает первые 50 строк SQL-файла.

**Типичное содержимое:**
- Установка параметров сессии
- Создание схем
- Создание таблиц
- Создание последовательностей (SEQUENCE)
- Вставка данных (COPY)
- Создание индексов
- Создание ограничений (constraints)
- Создание триггеров
- Установка владельцев объектов

**Концепция:** Понимание структуры бэкапа помогает в отладке и ручном редактировании при необходимости.

---

### 4. Логическое резервное копирование (custom формат)

```bash
pg_dump -U postgres -d lab6_backup -F c -f /tmp/backup_custom.dump
```

**Что делает:** Создает бинарный архив с метаданными для выборочного восстановления.

**Объяснение параметров:**
- `-F c` — формат: custom (бинарный архив)

**Преимущества custom формата:**
- Сжатие по умолчанию
- Выборочное восстановление (отдельные таблицы, схемы)
- Параллельное восстановление
- Переупорядочивание объектов при восстановлении
- Меньший размер файла

**Недостатки:**
- Не читаем человеком
- Требует pg_restore для восстановления
- Может быть несовместим между major версиями PostgreSQL

**Размер файлов (пример):**
```bash
ls -lh /tmp/backup_*
# -rw-r--r-- 1 user user 2.3M backup_plain.sql
# -rw-r--r-- 1 user user 890K backup_custom.dump  (сжат)
```

**Концепция:** Custom формат — рекомендуемый для продакшена, обеспечивает гибкость и эффективность.

---

### 5-6. Логическое резервное копирование (directory формат)

```bash
pg_dump -U postgres -d lab6_backup -F d -f /tmp/backup_directory
ls -lh /tmp/backup_directory/
```

**Что делает:** Создает директорию с отдельными файлами для разных компонентов базы данных.

**Объяснение параметров:**
- `-F d` — формат: directory

**Структура directory бэкапа:**
```
/tmp/backup_directory/
├── toc.dat           # Table of Contents (метаданные)
├── 3456.dat.gz       # Данные таблицы users (сжатые)
├── 3457.dat.gz       # Данные другой таблицы
└── ...
```

**Преимущества directory формата:**
- Параллельное резервное копирование (`-j` флаг)
- Параллельное восстановление
- Выборочное восстановление
- Сжатие данных

**Применение:**
- Очень большие базы данных (сотни GB)
- Многопроцессорные системы (ускорение через параллелизм)

**Концепция:** Directory формат — лучший выбор для больших баз данных, где важна скорость.

---

### 7-8. Резервное копирование только схемы или данных

```bash
# Только схема (структура без данных)
pg_dump -U postgres -d lab6_backup -s -f /tmp/backup_schema_only.sql

# Только данные (без структуры)
pg_dump -U postgres -d lab6_backup -a -f /tmp/backup_data_only.sql
```

**Объяснение параметров:**
- `-s` (--schema-only) — только определения объектов (CREATE TABLE, CREATE INDEX, и т.д.)
- `-a` (--data-only) — только данные (INSERT, COPY)

**Применение:**

**Schema-only:**
- Создание копии структуры для разработки
- Миграция схемы между окружениями
- Документирование структуры БД

**Data-only:**
- Обновление данных в существующей базе
- Перенос данных между идентичными схемами
- Тестирование с продакшн данными

**Пример schema-only вывода:**
```sql
CREATE TABLE public.users (...);
CREATE INDEX idx_users_email ON public.users (email);
-- Без COPY/INSERT команд
```

**Пример data-only вывода:**
```sql
-- Без CREATE TABLE
COPY public.users (id, name, email, created_at) FROM stdin;
1	Alice	alice@example.com	2026-01-15 10:00:00
...
```

**Концепция:** Разделение схемы и данных позволяет более гибко управлять процессом развертывания и обновления.

---

### 9. Резервное копирование конкретной таблицы

```bash
pg_dump -U postgres -d lab6_backup -t users -f /tmp/backup_users_table.sql
```

**Объяснение параметров:**
- `-t users` — только таблица users

**Можно указать несколько таблиц:**
```bash
pg_dump -U postgres -d lab6_backup -t users -t orders -f /tmp/backup_two_tables.sql
```

**Или исключить таблицы:**
```bash
pg_dump -U postgres -d lab6_backup -T logs -f /tmp/backup_without_logs.sql
```

**Параметры фильтрации:**
- `-t table` — включить таблицу
- `-T table` — исключить таблицу
- `-n schema` — включить схему
- `-N schema` — исключить схему

**Применение:**
- Резервное копирование критичных таблиц отдельно
- Экспорт данных для анализа
- Выборочное восстановление после ошибки

**Концепция:** Выборочное резервное копирование экономит время и место, особенно для больших баз с редко изменяющимися таблицами.

---

### 10-11. Резервное копирование со сжатием

```bash
# Сжатие через gzip
pg_dump -U postgres -d lab6_backup | gzip > /tmp/backup_compressed.sql.gz

# Или для custom формата (встроенное сжатие)
pg_dump -U postgres -d lab6_backup -F c -Z 9 -f /tmp/backup_max_compression.dump
```

**Объяснение параметров:**
- `-Z 0-9` — уровень сжатия (0 = без сжатия, 9 = максимальное сжатие)
- По умолчанию для custom: `-Z 6` (баланс скорости и размера)

**Сравнение размеров:**
```bash
ls -lh /tmp/backup_*
# -rw-r--r-- 2.3M backup_plain.sql
# -rw-r--r-- 450K backup_compressed.sql.gz
# -rw-r--r-- 890K backup_custom.dump (Z=6)
# -rw-r--r-- 780K backup_max_compression.dump (Z=9)
```

**Восстановление сжатого plain бэкапа:**
```bash
gunzip -c /tmp/backup_compressed.sql.gz | psql -U postgres -d lab6_backup
```

**Концепция:** Сжатие критично для экономии дискового пространства и ускорения передачи по сети, но требует больше CPU.

---

### 12-14. Восстановление из plain бэкапа

```bash
# Удаление базы данных
psql -U postgres -c "DROP DATABASE lab6_backup;"

# Создание пустой базы
psql -U postgres -c "CREATE DATABASE lab6_backup;"

# Восстановление
psql -U postgres -d lab6_backup -f /tmp/backup_plain.sql

# Проверка
psql -U postgres -d lab6_backup -c "SELECT * FROM users;"
```

**Что делает:** Выполняет SQL-команды из файла для воссоздания структуры и данных.

**Вывод при восстановлении:**
```
SET
SET
...
CREATE TABLE
CREATE SEQUENCE
ALTER TABLE
COPY 3
CREATE INDEX
...
```

**Обработка ошибок:**
```bash
# Продолжить при ошибках
psql -U postgres -d lab6_backup -f /tmp/backup_plain.sql --set ON_ERROR_STOP=off

# Один транзакционный блок (откат при любой ошибке)
psql -U postgres -d lab6_backup -f /tmp/backup_plain.sql --single-transaction
```

**Концепция:** Plain формат восстанавливается как обычный SQL-скрипт. Важно обрабатывать ошибки правильно.

---

### 15-16. Восстановление из custom бэкапа

```bash
psql -U postgres -c "DROP DATABASE lab6_backup;"
psql -U postgres -c "CREATE DATABASE lab6_backup;"

# Восстановление в одной транзакции
pg_restore -U postgres -d lab6_backup -1 /tmp/backup_custom.dump
```

**Объяснение параметров:**
- `-d lab6_backup` — целевая база данных
- `-1` (--single-transaction) — выполнить в одной транзакции
- `-e` (--exit-on-error) — остановить при первой ошибке

**Дополнительные опции:**
```bash
# С verbose режимом
pg_restore -U postgres -d lab6_backup -v /tmp/backup_custom.dump

# Без владельцев объектов
pg_restore -U postgres -d lab6_backup -O /tmp/backup_custom.dump

# Без привилегий (GRANT/REVOKE)
pg_restore -U postgres -d lab6_backup -x /tmp/backup_custom.dump

# Только создать SQL-скрипт (не выполнять)
pg_restore -f /tmp/restore_script.sql /tmp/backup_custom.dump
```

**Концепция:** pg_restore предоставляет больше контроля над процессом восстановления, чем psql.

---

### 17. Выборочное восстановление таблицы

```bash
psql -U postgres -c "DROP DATABASE lab6_backup;"
psql -U postgres -c "CREATE DATABASE lab6_backup;"

# Восстановить только таблицу users
pg_restore -U postgres -d lab6_backup -t users /tmp/backup_custom.dump
```

**Другие варианты выборочного восстановления:**
```bash
# Только схема (без данных)
pg_restore -U postgres -d lab6_backup -s /tmp/backup_custom.dump

# Только данные (без схемы)
pg_restore -U postgres -d lab6_backup -a /tmp/backup_custom.dump

# Исключить таблицу
pg_restore -U postgres -d lab6_backup -T logs /tmp/backup_custom.dump

# Восстановить конкретную схему
pg_restore -U postgres -d lab6_backup -n public /tmp/backup_custom.dump
```

**Концепция:** Выборочное восстановление позволяет быстро вернуть конкретные данные без восстановления всей базы.

---

### 18. Просмотр содержимого custom бэкапа

```bash
pg_restore -l /tmp/backup_custom.dump
```

**Вывод (пример):**
```
;
; Archive created at 2026-01-15 10:00:00 UTC
;     dbname: lab6_backup
;     TOC Entries: 15
;     Compression: 6
;     Dump Version: 1.14-0
;     Format: CUSTOM
;     Integer: 4 bytes
;     Offset: 8 bytes
;     Dumped from database version: 14.10
;     Dumped by pg_dump version: 14.10
;

214; 1259 16385 TABLE public users postgres
215; 1259 16384 SEQUENCE public users_id_seq postgres
216; 0 0 SEQUENCE OWNED BY public users_id_seq postgres
3456; 0 16385 TABLE DATA public users postgres
217; 2606 16392 CONSTRAINT public users users_pkey postgres
218; 1259 16393 INDEX public idx_users_email postgres
```

**Объяснение вывода:**
- Номер TOC (Table of Contents)
- OID объекта
- Тип объекта (TABLE, INDEX, CONSTRAINT, и т.д.)
- Имя объекта
- Владелец

**Применение:**
- Изучение структуры бэкапа
- Подготовка custom восстановления (редактирование TOC)
- Диагностика проблем

**Создание списка для выборочного восстановления:**
```bash
pg_restore -l /tmp/backup_custom.dump > /tmp/toc.txt
# Отредактировать toc.txt (закомментировать ненужные объекты)
pg_restore -U postgres -d lab6_backup -L /tmp/toc.txt /tmp/backup_custom.dump
```

**Концепция:** TOC (Table of Contents) — индекс содержимого бэкапа, позволяющий точно контролировать восстановление.

---

### 20-21. Параллельное резервное копирование и восстановление

```bash
# Параллельное резервное копирование (4 процесса)
pg_dump -U postgres -d lab6_backup -F d -j 4 -f /tmp/backup_parallel

# Параллельное восстановление (4 процесса)
psql -U postgres -c "DROP DATABASE lab6_backup;"
psql -U postgres -c "CREATE DATABASE lab6_backup;"
pg_restore -U postgres -d lab6_backup -j 4 /tmp/backup_parallel
```

**Объяснение параметров:**
- `-j 4` — использовать 4 параллельных процесса

**Требования:**
- Формат должен быть directory (`-F d`)
- Количество процессов ≤ количество ядер CPU
- База данных должна иметь несколько больших таблиц для эффективности

**Производительность:**
```
Без параллелизма: 10 минут
С -j 4:          3 минуты (ускорение ~3.3x)
С -j 8:          2 минуты (ускорение ~5x)
```

**Ограничения:**
- Только для directory формата
- Не работает с plain форматом
- Накладные расходы для маленьких баз (< 1GB)

**Концепция:** Параллелизм критичен для больших баз данных, сокращая время резервного копирования/восстановления в разы.

---

### 22-23. Резервное копирование всего кластера

```bash
# Все базы данных + глобальные объекты
pg_dumpall -U postgres -f /tmp/backup_all_databases.sql

# Только глобальные объекты (роли, табличные пространства)
pg_dumpall -U postgres -g -f /tmp/backup_globals.sql
```

**Что включает pg_dumpall:**
- Все базы данных в кластере
- Роли (пользователи и группы)
- Табличные пространства
- Глобальные привилегии

**Объяснение параметров:**
- `-g` (--globals-only) — только роли, табличные пространства, и т.д.

**Содержимое globals файла:**
```sql
CREATE ROLE admin;
ALTER ROLE admin WITH SUPERUSER INHERIT CREATEROLE CREATEDB LOGIN PASSWORD '...';

CREATE ROLE app_user;
ALTER ROLE app_user WITH NOSUPERUSER INHERIT NOCREATEROLE NOCREATEDB LOGIN PASSWORD '...';

CREATE TABLESPACE app_data OWNER postgres LOCATION '/mnt/data';
```

**Восстановление кластера:**
```bash
# Восстановить глобальные объекты
psql -U postgres -f /tmp/backup_globals.sql

# Восстановить все базы
psql -U postgres -f /tmp/backup_all_databases.sql
```

**ВАЖНО:** pg_dumpall создает только plain формат, нельзя использовать custom/directory.

**Концепция:** pg_dumpall необходим для полного восстановления кластера, включая пользователей и их права.

---

### 26. Скрипт автоматического резервного копирования

```bash
#!/bin/bash
BACKUP_DIR="/var/backups/postgresql"
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="$BACKUP_DIR/backup_$DATE.dump"

mkdir -p $BACKUP_DIR

# Резервное копирование
pg_dump -U postgres -d lab6_backup -F c -f $BACKUP_FILE

# Сжатие старых бэкапов (старше 7 дней)
find $BACKUP_DIR -name "*.dump" -mtime +7 -exec gzip {} \;

# Удаление очень старых бэкапов (старше 30 дней)
find $BACKUP_DIR -name "*.gz" -mtime +30 -delete

echo "Backup completed: $BACKUP_FILE"
```

**Что делает:**
1. Создает резервную копию с меткой времени
2. Сжимает бэкапы старше 7 дней
3. Удаляет бэкапы старше 30 дней

**Улучшенная версия с проверками:**
```bash
#!/bin/bash
set -e  # Остановить при ошибке

BACKUP_DIR="/var/backups/postgresql"
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="$BACKUP_DIR/backup_$DATE.dump"
LOG_FILE="/var/log/postgresql_backup.log"

# Функция логирования
log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a $LOG_FILE
}

log "Starting backup..."

# Проверка директории
mkdir -p $BACKUP_DIR

# Резервное копирование
if pg_dump -U postgres -d lab6_backup -F c -f $BACKUP_FILE; then
    log "Backup successful: $BACKUP_FILE"

    # Проверка целостности
    if pg_restore -l $BACKUP_FILE > /dev/null 2>&1; then
        log "Backup integrity verified"

        # Размер бэкапа
        SIZE=$(du -h $BACKUP_FILE | cut -f1)
        log "Backup size: $SIZE"
    else
        log "ERROR: Backup integrity check failed"
        exit 1
    fi
else
    log "ERROR: Backup failed"
    exit 1
fi

# Rotation
log "Running backup rotation..."
find $BACKUP_DIR -name "*.dump" -mtime +7 -exec gzip {} \; -exec log "Compressed old backup: {}" \;
find $BACKUP_DIR -name "*.gz" -mtime +30 -delete -exec log "Deleted old backup: {}" \;

log "Backup process completed"
```

**Настройка cron:**
```bash
# Редактировать crontab
crontab -e

# Добавить строку (ежедневно в 2:00 AM)
0 2 * * * /var/backups/postgresql/backup_script.sh

# Или еженедельно (воскресенье в 3:00 AM)
0 3 * * 0 /var/backups/postgresql/backup_script.sh
```

**Концепция:** Автоматизация резервного копирования с ротацией критична для защиты данных в продакшене.

---

### 28-31. Физическое резервное копирование (pg_basebackup)

```bash
# Создание пользователя для репликации
psql -U postgres << 'EOF'
CREATE ROLE backup_user WITH REPLICATION LOGIN PASSWORD 'backup_password';
EOF

# Настройка pg_hba.conf
# host replication backup_user 127.0.0.1/32 md5

# Перезапуск PostgreSQL
sudo systemctl restart postgresql

# Выполнение физического бэкапа
pg_basebackup -U backup_user -D /tmp/physical_backup -Fp -Xs -P
```

**Объяснение параметров:**
- `-U backup_user` — пользователь с правами репликации
- `-D /tmp/physical_backup` — целевая директория
- `-F p` — формат: plain (директория с файлами)
- `-F t` — формат: tar (архив)
- `-X s` — включить WAL файлы методом stream
- `-X f` — включить WAL файлы методом fetch
- `-P` — показывать прогресс
- `-z` — сжимать (только с `-F t`)

**Методы включения WAL:**
- **stream (-Xs)**: Потоковая передача WAL во время копирования (рекомендуется)
- **fetch (-Xf)**: Получить WAL после копирования (может быть недостаточно для восстановления)
- **none**: Не включать WAL (требуется архивация WAL)

**Содержимое физического бэкапа:**
```bash
ls -lh /tmp/physical_backup/
# base/          # Данные баз данных
# global/        # Глобальные объекты
# pg_wal/        # Журналы транзакций
# postgresql.conf
# pg_hba.conf
# backup_label   # Метаданные бэкапа
```

**Восстановление из физического бэкапа:**
```bash
# Остановить PostgreSQL
sudo systemctl stop postgresql

# Заменить data директорию
sudo rm -rf /var/lib/postgresql/14/main/*
sudo cp -R /tmp/physical_backup/* /var/lib/postgresql/14/main/
sudo chown -R postgres:postgres /var/lib/postgresql/14/main

# Запустить PostgreSQL
sudo systemctl start postgresql
```

**Отличия от логического бэкапа:**
- Копирует все файлы базы данных на уровне ОС
- Быстрее для больших баз (копирование файлов vs SQL)
- Создает точную копию кластера
- Требует одинаковые версии PostgreSQL и архитектуру CPU
- Включает все базы данных (нельзя выборочно)

**Концепция:** Физическое резервное копирование — самый быстрый метод для больших баз, но менее гибкий, чем логическое.

---

### 32-34. Непрерывная архивация WAL

```bash
# Настройка в postgresql.conf
sudo nano /etc/postgresql/14/main/postgresql.conf
```

**Добавить/изменить:**
```ini
wal_level = replica              # Минимальный уровень для архивации
archive_mode = on                # Включить архивацию
archive_command = 'cp %p /var/lib/postgresql/wal_archive/%f'
archive_timeout = 300            # Принудительный переключатель WAL каждые 5 мин
```

**Объяснение параметров:**

**`wal_level`:**
- `minimal` — минимальное логирование (без архивации)
- `replica` — для репликации и архивации (рекомендуется)
- `logical` — для логической репликации

**`archive_mode`:**
- `off` — архивация отключена
- `on` — архивация включена
- `always` — архивация всегда (даже в standby режиме)

**`archive_command`:**
- Команда для копирования WAL файлов
- `%p` — путь к файлу WAL
- `%f` — имя файла WAL

**Примеры archive_command:**
```bash
# Копирование локально
archive_command = 'cp %p /backup/wal/%f'

# Копирование через SSH
archive_command = 'scp %p backup-server:/backup/wal/%f'

# Копирование в AWS S3
archive_command = 'aws s3 cp %p s3://my-bucket/wal/%f'

# С проверкой целостности
archive_command = 'test ! -f /backup/wal/%f && cp %p /backup/wal/%f'
```

**Создание директории:**
```bash
sudo mkdir -p /var/lib/postgresql/wal_archive
sudo chown postgres:postgres /var/lib/postgresql/wal_archive
sudo chmod 700 /var/lib/postgresql/wal_archive
```

**Проверка архивации:**
```bash
# Переключить WAL
psql -U postgres -c "SELECT pg_switch_wal();"

# Проверить архив
ls -lh /var/lib/postgresql/wal_archive/
# -rw------- 1 postgres postgres 16M 00000001000000000000002A
```

**Мониторинг архивации:**
```sql
SELECT
    archived_count,
    last_archived_wal,
    last_archived_time,
    failed_count,
    last_failed_wal,
    last_failed_time
FROM pg_stat_archiver;
```

**Концепция:** WAL архивация позволяет восстановить базу данных до любого момента времени (Point-in-Time Recovery).

---

### 35-36. Point-in-Time Recovery (PITR)

```bash
# 1. Создать базовую резервную копию
pg_basebackup -U backup_user -D /tmp/pitr_backup -Fp -Xs -P

# 2. Выполнить операции в базе
psql -U postgres -d lab6_backup -c "INSERT INTO users (name, email) VALUES ('Eve', 'eve@example.com');"
# Записать время: 2026-01-15 12:00:00

psql -U postgres -d lab6_backup -c "DELETE FROM users WHERE name = 'Eve';"  # Ошибка!
# Время: 2026-01-15 12:05:00

# 3. Остановить PostgreSQL
sudo systemctl stop postgresql

# 4. Заменить data директорию
sudo rm -rf /var/lib/postgresql/14/main/*
sudo cp -R /tmp/pitr_backup/* /var/lib/postgresql/14/main/
sudo chown -R postgres:postgres /var/lib/postgresql/14/main

# 5. Создать recovery.signal
sudo touch /var/lib/postgresql/14/main/recovery.signal

# 6. Настроить восстановление
sudo nano /var/lib/postgresql/14/main/postgresql.conf
```

**Добавить в postgresql.conf:**
```ini
restore_command = 'cp /var/lib/postgresql/wal_archive/%f %p'
recovery_target_time = '2026-01-15 12:00:00'
recovery_target_action = 'promote'
```

**Объяснение параметров:**

**`recovery.signal`:**
- Файл-маркер, указывающий PostgreSQL войти в режим восстановления
- Удаляется автоматически после успешного восстановления

**`restore_command`:**
- Команда для получения WAL файлов из архива
- `%f` — имя файла WAL
- `%p` — путь, куда поместить файл

**`recovery_target_time`:**
- Восстановить до указанного момента времени
- Формат: 'YYYY-MM-DD HH:MM:SS'

**`recovery_target_action`:**
- `pause` — остановиться после достижения цели (для проверки)
- `promote` — автоматически перевести в обычный режим
- `shutdown` — выключить сервер после восстановления

**Другие варианты recovery_target:**
```ini
# Восстановить до конкретной транзакции
recovery_target_xid = '12345'

# Восстановить до конкретного LSN
recovery_target_lsn = '0/15D0128'

# Восстановить до именованной точки восстановления
recovery_target_name = 'before_critical_update'

# Восстановить до конца WAL
recovery_target = 'immediate'
```

**Создание именованной точки восстановления:**
```sql
SELECT pg_create_restore_point('before_critical_update');
```

**Запуск PostgreSQL:**
```bash
sudo systemctl start postgresql
```

**Проверка восстановления:**
```bash
# Проверить логи
sudo tail -f /var/log/postgresql/postgresql-14-main.log

# Проверить данные
psql -U postgres -d lab6_backup -c "SELECT * FROM users;"
# Должна быть строка Eve (до DELETE)
```

**Концепция:** PITR — мощный механизм восстановления до любого момента времени между базовым бэкапом и текущим состоянием. Критично для восстановления после ошибок администратора или приложения.

---

### 37-39. Мониторинг и проверка бэкапов

```sql
-- Размеры базы данных
SELECT
    pg_size_pretty(pg_database_size('lab6_backup')) as db_size,
    pg_size_pretty(pg_total_relation_size('users')) as users_table_size;
```

**Вывод:**
```
 db_size | users_table_size
---------+------------------
 8145 kB | 16 kB
```

**Оценка времени восстановления:**
```bash
time pg_restore -U postgres -d lab6_backup /tmp/backup_custom.dump

# real    0m2.543s
# user    0m0.123s
# sys     0m0.045s
```

**Проверка целостности бэкапа:**
```bash
# Для custom формата
pg_restore -l /tmp/backup_custom.dump > /dev/null && echo "Backup OK" || echo "Backup corrupted"

# Для plain формата (проверить синтаксис)
psql -U postgres -d template1 -f /tmp/backup_plain.sql --set ON_ERROR_STOP=on --dry-run 2>&1 | grep -i error
```

**Автоматическая проверка бэкапов:**
```bash
#!/bin/bash
# Создать временную базу
createdb test_restore_db

# Попытка восстановления
if pg_restore -d test_restore_db /path/to/backup.dump 2>&1 | tee restore.log; then
    echo "Backup is valid"

    # Проверить количество таблиц
    TABLE_COUNT=$(psql -d test_restore_db -t -c "SELECT COUNT(*) FROM information_schema.tables WHERE table_schema = 'public';")
    echo "Tables restored: $TABLE_COUNT"
else
    echo "Backup is corrupted"
    exit 1
fi

# Удалить временную базу
dropdb test_restore_db
```

**Концепция:** Регулярная проверка бэкапов критична. Бэкап бесполезен, если его нельзя восстановить.

---

## Технический концепт: Стратегии резервного копирования

### Правило 3-2-1

**Золотое правило резервного копирования:**
- **3 копии данных**: продакшн + 2 бэкапа
- **2 разных носителя**: диск + облако, или локальный + удаленный диск
- **1 копия вне офиса**: защита от физических катастроф (пожар, затопление)

**Реализация в PostgreSQL:**
1. Первичная копия: pg_basebackup на локальный диск
2. Вторая копия: логический бэкап (pg_dump) на другой диск
3. Третья копия: синхронизация в облако (S3, GCS, Azure Blob)

### Виды резервного копирования

**1. Полное (Full Backup):**
- Копирует все данные
- Самое медленное
- Самое простое восстановление
- Частота: еженедельно/ежемесячно

**2. Инкрементальное (Incremental Backup):**
- Копирует только изменения с последнего бэкапа
- Самое быстрое
- Сложное восстановление (нужна цепочка)
- Частота: ежедневно/ежечасно
- В PostgreSQL: WAL архивация

**3. Дифференциальное (Differential Backup):**
- Копирует изменения с последнего полного бэкапа
- Среднее между полным и инкрементальным
- Восстановление проще, чем инкрементальное
- В PostgreSQL: не поддерживается нативно

### Логическое vs Физическое резервное копирование

**Логическое (pg_dump):**

**Преимущества:**
- Переносимость между версиями PostgreSQL
- Выборочное резервное копирование
- Читаемый формат (plain)
- Независимость от ОС и архитектуры

**Недостатки:**
- Медленнее для больших баз (GB -> TB)
- Большая нагрузка на CPU
- Требует место для экспорта данных

**Применение:**
- Миграции между версиями
- Частичное резервное копирование
- Небольшие и средние базы (< 100GB)

**Физическое (pg_basebackup + WAL):**

**Преимущества:**
- Очень быстрое (копирование файлов)
- Минимальная нагрузка на CPU
- Point-in-Time Recovery
- Подходит для репликации

**Недостатки:**
- Требует одинаковые версии PostgreSQL
- Зависит от ОС и архитектуры
- Копирует весь кластер (нельзя выборочно)

**Применение:**
- Большие базы (> 100GB)
- Критичные системы (требуется PITR)
- Настройка репликации

### Стратегия восстановления (Recovery Time Objective / Recovery Point Objective)

**RTO (Recovery Time Objective):**
- Максимальное допустимое время простоя
- Как быстро нужно восстановить систему

**RPO (Recovery Point Objective):**
- Максимально допустимая потеря данных
- Насколько старые данные допустимы

**Примеры стратегий:**

**1. Критичная система (RTO < 1 час, RPO < 5 минут):**
```
- Streaming репликация (standby сервер)
- WAL архивация каждые 5 минут
- pg_basebackup ежедневно
- Тестирование восстановления еженедельно
```

**2. Важная система (RTO < 4 часа, RPO < 1 час):**
```
- WAL архивация каждые 15 минут
- pg_basebackup ежедневно ночью
- pg_dump еженедельно
- Копирование в облако
```

**3. Некритичная система (RTO < 24 часа, RPO < 24 часа):**
```
- pg_dump ежедневно
- Хранение 30 дней
- Копирование в облако еженедельно
```

### График резервного копирования (пример)

**Ежедневно (2:00 AM):**
```bash
# Инкрементальное - WAL архивация (автоматически)
# Логический бэкап
pg_dump -F c -f /backup/daily/db_$(date +%Y%m%d).dump mydb
```

**Еженедельно (Воскресенье 3:00 AM):**
```bash
# Физический бэкап
pg_basebackup -D /backup/weekly/base_$(date +%Y%m%d)

# Полный логический бэкап всех баз
pg_dumpall -f /backup/weekly/all_$(date +%Y%m%d).sql
```

**Ежемесячно (1-е число 4:00 AM):**
```bash
# Копирование в облако
aws s3 sync /backup s3://my-postgres-backups/
```

**Проверка (Ежедневно 6:00 AM):**
```bash
# Автоматическая проверка последнего бэкапа
/scripts/verify_backup.sh /backup/daily/latest.dump
```

### Инструменты для резервного копирования

**Встроенные:**
- `pg_dump` / `pg_dumpall` — логическое резервное копирование
- `pg_restore` — восстановление
- `pg_basebackup` — физическое резервное копирование

**Сторонние решения:**

**pgBackRest:**
- Продвинутое резервное копирование и архивация
- Инкрементальные и дифференциальные бэкапы
- Параллельное копирование и восстановление
- Поддержка нескольких репозиториев

**Barman (Backup and Recovery Manager):**
- Централизованное управление бэкапами
- Поддержка множества серверов PostgreSQL
- WAL архивация и компрессия
- Мониторинг и алерты

**WAL-G:**
- Современный инструмент на Go
- Поддержка облачных хранилищ (S3, GCS, Azure)
- Инкрементальное резервное копирование
- Delta-бэкапы

**Patroni + pgBackRest:**
- Высокая доступность + продвинутые бэкапы
- Автоматическое переключение при сбое
- Интеграция с Kubernetes

### Тестирование восстановления

**Критически важно:**
- Бэкап бесполезен, если его нельзя восстановить
- Регулярно проверяйте процедуру восстановления
- Документируйте процесс
- Тренируйте команду

**План тестирования:**
1. **Ежемесячно:** Полное восстановление на тестовую среду
2. **Еженедельно:** Проверка целостности бэкапов
3. **Ежеквартально:** Drill (симуляция катастрофы)
4. **Ежегодно:** Полная проверка всего процесса DR (Disaster Recovery)

**Чек-лист восстановления:**
- [ ] Бэкап доступен и читаем
- [ ] Достаточно дискового пространства
- [ ] PostgreSQL версия совместима
- [ ] Права доступа настроены
- [ ] Конфигурация восстановлена
- [ ] Данные целостны
- [ ] Индексы и ограничения работают
- [ ] Производительность нормальная
- [ ] Приложения могут подключиться

### Безопасность бэкапов

**Шифрование:**
```bash
# Шифрование при создании
pg_dump mydb | gpg -e -r admin@example.com > backup.sql.gpg

# Расшифровка при восстановлении
gpg -d backup.sql.gpg | psql mydb
```

**Хранение паролей:**
```bash
# Использовать .pgpass
echo "localhost:5432:*:postgres:password" >> ~/.pgpass
chmod 600 ~/.pgpass

# Или переменные окружения
export PGPASSWORD=mypassword
pg_dump mydb
```

**Контроль доступа:**
- Ограничить доступ к бэкапам (chmod 600)
- Отдельный пользователь для резервного копирования
- Аудит доступа к бэкапам
- Шифрование при передаче (scp, S3 с SSE)

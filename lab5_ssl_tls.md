# Лабораторная работа №5: Настройка SSL/TLS в PostgreSQL

## Краткая последовательность команд

```bash
# 1. Проверка поддержки SSL в PostgreSQL
psql -U postgres -c "SHOW ssl;"

# 2. Создание директории для сертификатов
sudo mkdir -p /etc/postgresql/ssl
cd /etc/postgresql/ssl

# 3. Генерация приватного ключа сервера (RSA 2048 бит)
sudo openssl genrsa -out server.key 2048

# 4. Установка правильных прав доступа на ключ
sudo chmod 600 server.key
sudo chown postgres:postgres server.key

# 5. Создание запроса на подпись сертификата (CSR)
sudo openssl req -new -key server.key -out server.csr \
  -subj "/C=RU/ST=Moscow/L=Moscow/O=MyCompany/OU=IT/CN=localhost"

# 6. Генерация самоподписанного сертификата (действителен 365 дней)
sudo openssl x509 -req -days 365 -in server.csr -signkey server.key -out server.crt

# 7. Установка прав на сертификат
sudo chmod 644 server.crt
sudo chown postgres:postgres server.crt

# 8. Копирование корневого сертификата (для клиентской аутентификации)
sudo cp server.crt root.crt
sudo chmod 644 root.crt

# 9. Поиск postgresql.conf
sudo -u postgres psql -c "SHOW config_file;"

# 10. Редактирование postgresql.conf (включение SSL)
sudo nano /etc/postgresql/14/main/postgresql.conf
# Изменить/добавить строки:
# ssl = on
# ssl_cert_file = '/etc/postgresql/ssl/server.crt'
# ssl_key_file = '/etc/postgresql/ssl/server.key'
# ssl_ca_file = '/etc/postgresql/ssl/root.crt'

# 11. Просмотр текущей конфигурации SSL
psql -U postgres -c "SHOW ssl;"
psql -U postgres -c "SHOW ssl_cert_file;"
psql -U postgres -c "SHOW ssl_key_file;"

# 12. Редактирование pg_hba.conf (требование SSL)
sudo nano /etc/postgresql/14/main/pg_hba.conf
# Добавить строку:
# hostssl  all  all  0.0.0.0/0  md5

# 13. Перезапуск PostgreSQL
sudo systemctl restart postgresql

# 14. Проверка статуса PostgreSQL
sudo systemctl status postgresql

# 15. Подключение через SSL
psql "host=localhost user=postgres dbname=postgres sslmode=require"

# 16. Проверка SSL-соединения внутри psql
psql -U postgres -c "\conninfo"
# Или
psql -U postgres -c "SELECT ssl_is_used();"

# 17. Просмотр информации о SSL-соединении
psql -U postgres << 'EOF'
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
EOF

# 18. Создание клиентского сертификата
sudo openssl genrsa -out client.key 2048
sudo openssl req -new -key client.key -out client.csr \
  -subj "/C=RU/ST=Moscow/L=Moscow/O=MyCompany/OU=IT/CN=postgres"
sudo openssl x509 -req -days 365 -in client.csr \
  -CA root.crt -CAkey server.key -CAcreateserial -out client.crt

# 19. Установка прав на клиентские файлы
sudo chmod 600 client.key
sudo chmod 644 client.crt

# 20. Копирование клиентских сертификатов в домашнюю директорию
mkdir -p ~/.postgresql
cp /etc/postgresql/ssl/client.crt ~/.postgresql/postgresql.crt
cp /etc/postgresql/ssl/client.key ~/.postgresql/postgresql.key
cp /etc/postgresql/ssl/root.crt ~/.postgresql/root.crt
chmod 600 ~/.postgresql/postgresql.key
chmod 644 ~/.postgresql/postgresql.crt
chmod 644 ~/.postgresql/root.crt

# 21. Настройка pg_hba.conf для клиентских сертификатов
sudo nano /etc/postgresql/14/main/pg_hba.conf
# Изменить на:
# hostssl  all  all  0.0.0.0/0  cert

# 22. Перезапуск PostgreSQL
sudo systemctl restart postgresql

# 23. Подключение с клиентским сертификатом
psql "host=localhost user=postgres dbname=postgres sslmode=require"

# 24. Тестирование разных режимов SSL
# sslmode=disable - без SSL
psql "host=localhost user=postgres dbname=postgres sslmode=disable"

# sslmode=allow - попытка SSL, затем без SSL
psql "host=localhost user=postgres dbname=postgres sslmode=allow"

# sslmode=prefer - попытка SSL (по умолчанию)
psql "host=localhost user=postgres dbname=postgres sslmode=prefer"

# sslmode=require - обязательно SSL
psql "host=localhost user=postgres dbname=postgres sslmode=require"

# sslmode=verify-ca - проверка сертификата CA
psql "host=localhost user=postgres dbname=postgres sslmode=verify-ca"

# sslmode=verify-full - полная проверка сертификата
psql "host=localhost user=postgres dbname=postgres sslmode=verify-full"

# 25. Просмотр шифров SSL
psql -U postgres -c "SHOW ssl_ciphers;"

# 26. Настройка минимальной версии TLS
sudo nano /etc/postgresql/14/main/postgresql.conf
# Добавить:
# ssl_min_protocol_version = 'TLSv1.2'
# ssl_max_protocol_version = 'TLSv1.3'

# 27. Мониторинг SSL-соединений
psql -U postgres << 'EOF'
SELECT
    COUNT(*) FILTER (WHERE ssl = true) AS ssl_connections,
    COUNT(*) FILTER (WHERE ssl = false) AS non_ssl_connections,
    COUNT(*) AS total_connections
FROM pg_stat_ssl
JOIN pg_stat_activity USING (pid)
WHERE state = 'active';
EOF

# 28. Проверка срока действия сертификата
openssl x509 -in /etc/postgresql/ssl/server.crt -noout -dates

# 29. Просмотр деталей сертификата
openssl x509 -in /etc/postgresql/ssl/server.crt -noout -text

# 30. Тестирование SSL-соединения через openssl
openssl s_client -connect localhost:5432 -starttls postgres
```

---

## Подробное объяснение команд

### 1. Проверка поддержки SSL

```bash
psql -U postgres -c "SHOW ssl;"
```

**Что делает:** Проверяет, включена ли поддержка SSL в PostgreSQL.

**Вывод (если SSL отключен):**
```
 ssl
-----
 off
```

**Вывод (если SSL включен):**
```
 ssl
-----
 on
```

**Объяснение:** PostgreSQL должен быть скомпилирован с поддержкой OpenSSL. Большинство дистрибутивов включают это по умолчанию.

**Проверка версии OpenSSL:**
```bash
openssl version
# OpenSSL 1.1.1f  31 Mar 2020
```

**Концепция:** SSL/TLS — криптографические протоколы для защиты данных при передаче по сети. TLS (Transport Layer Security) — современная версия протокола SSL (Secure Sockets Layer).

---

### 2-3. Создание приватного ключа сервера

```bash
sudo mkdir -p /etc/postgresql/ssl
cd /etc/postgresql/ssl
sudo openssl genrsa -out server.key 2048
```

**Что делает:** Генерирует RSA-ключ длиной 2048 бит для сервера.

**Объяснение параметров:**
- `genrsa` — команда для генерации RSA-ключа
- `-out server.key` — имя выходного файла
- `2048` — длина ключа в битах (минимум рекомендуемый — 2048, для высокой безопасности — 4096)

**Вывод:**
```
Generating RSA private key, 2048 bit long modulus
.............+++
...........+++
e is 65537 (0x10001)
```

**Концепция:** Приватный ключ — секретная часть криптографической пары. Никогда не должен быть передан клиентам. Используется для расшифровки данных и создания цифровых подписей.

---

### 4. Установка прав доступа на ключ

```bash
sudo chmod 600 server.key
sudo chown postgres:postgres server.key
```

**Что делает:** Устанавливает права доступа на приватный ключ.

**Объяснение:**
- `chmod 600` — только владелец может читать и писать (rw--------)
- `chown postgres:postgres` — владелец — пользователь и группа postgres

**Почему это важно:**
- PostgreSQL откажется запускаться, если права на ключ слишком свободные
- Требование безопасности: приватный ключ не должен быть доступен другим пользователям
- Допустимые права: 0600 (только владелец) или 0640 (владелец + группа)

**Проверка прав:**
```bash
ls -l server.key
# -rw------- 1 postgres postgres 1675 Jan 15 10:00 server.key
```

**Концепция:** Безопасность приватного ключа критична. Если злоумышленник получит доступ к ключу, он сможет расшифровать трафик или выдать себя за сервер.

---

### 5. Создание запроса на подпись сертификата (CSR)

```bash
sudo openssl req -new -key server.key -out server.csr \
  -subj "/C=RU/ST=Moscow/L=Moscow/O=MyCompany/OU=IT/CN=localhost"
```

**Что делает:** Создает запрос на сертификат с метаданными организации.

**Объяснение параметров:**
- `req -new` — создать новый запрос на сертификат
- `-key server.key` — использовать существующий приватный ключ
- `-out server.csr` — файл запроса (Certificate Signing Request)
- `-subj` — информация о субъекте (избегаем интерактивных вопросов)

**Поля субъекта:**
- `C` (Country) — код страны (RU для России)
- `ST` (State) — область/штат
- `L` (Locality) — город
- `O` (Organization) — организация
- `OU` (Organizational Unit) — подразделение
- `CN` (Common Name) — полное доменное имя сервера (КРИТИЧЕСКИ ВАЖНО!)

**ВАЖНО:** CN должен совпадать с именем хоста, к которому подключаются клиенты:
- Для локального тестирования: `CN=localhost`
- Для сервера: `CN=db.example.com`
- Для IP-адреса: использовать SAN (Subject Alternative Name)

**Просмотр CSR:**
```bash
openssl req -in server.csr -noout -text
```

**Концепция:** В продакшене CSR отправляется в центр сертификации (CA), который проверяет данные и выдает подписанный сертификат. Для тестирования используем самоподписанные сертификаты.

---

### 6. Генерация самоподписанного сертификата

```bash
sudo openssl x509 -req -days 365 -in server.csr -signkey server.key -out server.crt
```

**Что делает:** Создает самоподписанный сертификат, действительный 365 дней.

**Объяснение параметров:**
- `x509` — формат сертификата X.509 (стандарт)
- `-req` — использовать CSR в качестве входных данных
- `-days 365` — срок действия сертификата (1 год)
- `-in server.csr` — файл запроса
- `-signkey server.key` — подписать своим же ключом (самоподписанный)
- `-out server.crt` — выходной файл сертификата

**Вывод:**
```
Signature ok
subject=/C=RU/ST=Moscow/L=Moscow/O=MyCompany/OU=IT/CN=localhost
Getting Private key
```

**Отличие от реального сертификата:**
- Самоподписанный: подписан собственным ключом, клиенты не доверяют автоматически
- От CA: подписан доверенным центром сертификации, клиенты доверяют автоматически

**Для продакшена:** Получить сертификат от доверенного CA (Let's Encrypt, DigiCert, и т.д.)

**Концепция:** Сертификат содержит публичный ключ и информацию о владельце, заверенную цифровой подписью. Клиенты используют сертификат для проверки подлинности сервера и шифрования данных.

---

### 7-8. Установка прав на сертификаты

```bash
sudo chmod 644 server.crt
sudo chown postgres:postgres server.crt
sudo cp server.crt root.crt
sudo chmod 644 root.crt
```

**Что делает:** Устанавливает права доступа на публичные сертификаты.

**Объяснение:**
- Сертификаты (`.crt`) — публичные, могут быть доступны для чтения всем
- `644` — владелец может писать, все могут читать (rw-r--r--)
- `root.crt` — корневой сертификат для проверки клиентами

**Концепция:** В отличие от приватного ключа, сертификаты не являются секретными. Они предназначены для распространения.

---

### 9-10. Настройка PostgreSQL для использования SSL

```bash
sudo -u postgres psql -c "SHOW config_file;"
# /etc/postgresql/14/main/postgresql.conf

sudo nano /etc/postgresql/14/main/postgresql.conf
```

**Изменения в postgresql.conf:**
```ini
# Включить SSL
ssl = on

# Пути к файлам SSL
ssl_cert_file = '/etc/postgresql/ssl/server.crt'
ssl_key_file = '/etc/postgresql/ssl/server.key'
ssl_ca_file = '/etc/postgresql/ssl/root.crt'

# Опционально: настройка шифров
ssl_ciphers = 'HIGH:MEDIUM:+3DES:!aNULL'

# Опционально: минимальная версия TLS
ssl_min_protocol_version = 'TLSv1.2'
```

**Объяснение параметров:**

**`ssl = on`** — включает поддержку SSL на сервере

**`ssl_cert_file`** — путь к сертификату сервера

**`ssl_key_file`** — путь к приватному ключу сервера

**`ssl_ca_file`** — путь к корневому сертификату (для проверки клиентских сертификатов)

**`ssl_ciphers`** — набор разрешенных алгоритмов шифрования:
- `HIGH` — высокая криптостойкость (256-бит)
- `MEDIUM` — средняя криптостойкость (128-бит)
- `!aNULL` — исключить анонимные шифры (без аутентификации)
- `!eNULL` — исключить шифры без шифрования

**`ssl_min_protocol_version`** — минимальная версия TLS:
- `TLSv1` — устаревший, небезопасный
- `TLSv1.1` — устаревший
- `TLSv1.2` — рекомендуемый минимум
- `TLSv1.3` — современный, самый безопасный

**Просмотр доступных параметров:**
```sql
SELECT name, setting, short_desc
FROM pg_settings
WHERE name LIKE 'ssl%'
ORDER BY name;
```

**Концепция:** Правильная настройка шифров и протоколов критична для безопасности. Отключение устаревших протоколов (SSLv3, TLSv1.0, TLSv1.1) защищает от известных уязвимостей.

---

### 12. Настройка pg_hba.conf

```bash
sudo nano /etc/postgresql/14/main/pg_hba.conf
```

**Добавить строку:**
```
# TYPE  DATABASE  USER  ADDRESS       METHOD
hostssl all       all   0.0.0.0/0     md5
```

**Объяснение столбцов:**
- `hostssl` — тип подключения (SSL обязателен)
- `all` — все базы данных
- `all` — все пользователи
- `0.0.0.0/0` — все IP-адреса
- `md5` — метод аутентификации (пароль, зашифрованный MD5)

**Типы подключений:**
- `local` — Unix-сокеты (локальные подключения)
- `host` — TCP/IP (с SSL или без)
- `hostssl` — TCP/IP только через SSL
- `hostnossl` — TCP/IP только без SSL

**Методы аутентификации:**
- `trust` — без пароля (небезопасно!)
- `md5` — пароль, зашифрованный MD5
- `scram-sha-256` — современный, безопасный (рекомендуется)
- `cert` — аутентификация по клиентскому сертификату
- `peer` — аутентификация по имени пользователя ОС

**Пример безопасной конфигурации:**
```
# Локальные подключения - без SSL
local   all       all                     scram-sha-256

# Удаленные подключения - обязательно SSL
hostssl all       all   0.0.0.0/0         scram-sha-256

# Запретить подключения без SSL
hostnossl all     all   0.0.0.0/0         reject
```

**Концепция:** pg_hba.conf — файл конфигурации аутентификации клиентов. Правила применяются сверху вниз, первое совпадение используется.

---

### 13-14. Перезапуск PostgreSQL

```bash
sudo systemctl restart postgresql
sudo systemctl status postgresql
```

**Что делает:** Перезапускает PostgreSQL для применения изменений конфигурации.

**Проверка логов при ошибке:**
```bash
sudo journalctl -u postgresql -n 50
# или
sudo tail -f /var/log/postgresql/postgresql-14-main.log
```

**Типичные ошибки:**
- Неправильные права на `server.key` → PostgreSQL откажется запускаться
- Некорректные пути к файлам → SSL не будет работать
- Синтаксическая ошибка в конфигурации → сервер не запустится

**Концепция:** Всегда проверяйте логи после изменения конфигурации. PostgreSQL выводит подробные сообщения об ошибках.

---

### 15-16. Подключение через SSL

```bash
psql "host=localhost user=postgres dbname=postgres sslmode=require"
```

**Формат строки подключения:**
```
psql "host=HOST user=USER dbname=DB sslmode=MODE"
```

**Или через переменные окружения:**
```bash
export PGSSLMODE=require
psql -h localhost -U postgres
```

**Проверка SSL внутри psql:**
```sql
\conninfo
-- You are connected to database "postgres" as user "postgres" on host "localhost" (address "127.0.0.1") at port "5432".
-- SSL connection (protocol: TLSv1.3, cipher: TLS_AES_256_GCM_SHA384, bits: 256, compression: off)

SELECT ssl_is_used();
-- ssl_is_used
-- -------------
--  t
```

**Функция ssl_is_used():**
- Возвращает `true` если текущее соединение использует SSL
- Возвращает `false` для незащищенных соединений

**Концепция:** Всегда проверяйте, что соединение действительно зашифровано, особенно в продакшене.

---

### 17. Мониторинг SSL-соединений

```sql
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

**Вывод (пример):**
```
  pid  | usename  | application_name | client_addr |  ssl | ssl_version |      ssl_cipher
-------+----------+------------------+-------------+------+-------------+------------------------
 12345 | postgres | psql             | 127.0.0.1   | t    | TLSv1.3     | TLS_AES_256_GCM_SHA384
 12346 | app_user | myapp            | 10.0.1.5    | t    | TLSv1.2     | ECDHE-RSA-AES256-SHA384
```

**Объяснение колонок:**
- `pid` — ID процесса
- `usename` — имя пользователя
- `application_name` — имя приложения
- `client_addr` — IP-адрес клиента
- `ssl` — используется ли SSL (true/false)
- `ssl_version` — версия TLS-протокола
- `ssl_cipher` — используемый шифр

**Системные представления:**
- `pg_stat_ssl` — информация о SSL для каждого подключения
- `pg_stat_activity` — общая информация о подключениях

**Запрос для мониторинга:**
```sql
SELECT
    COUNT(*) FILTER (WHERE ssl) AS ssl_connections,
    COUNT(*) FILTER (WHERE NOT ssl) AS non_ssl_connections
FROM pg_stat_ssl;
```

**Концепция:** Регулярный мониторинг SSL-соединений помогает обнаружить клиентов, подключающихся без шифрования.

---

### 18-20. Создание клиентских сертификатов

```bash
# Генерация клиентского ключа
sudo openssl genrsa -out client.key 2048

# Создание CSR для клиента
sudo openssl req -new -key client.key -out client.csr \
  -subj "/C=RU/ST=Moscow/L=Moscow/O=MyCompany/OU=IT/CN=postgres"

# Подписание клиентского сертификата серверным CA
sudo openssl x509 -req -days 365 -in client.csr \
  -CA root.crt -CAkey server.key -CAcreateserial -out client.crt
```

**Что делает:** Создает клиентский сертификат для двусторонней аутентификации.

**Объяснение:**
- CN в клиентском сертификате должен совпадать с именем пользователя PostgreSQL
- Сертификат подписывается корневым CA (в данном случае — тем же серверным ключом)
- `-CAcreateserial` создает файл с серийным номером для отслеживания выданных сертификатов

**Копирование в домашнюю директорию:**
```bash
mkdir -p ~/.postgresql
cp /etc/postgresql/ssl/client.crt ~/.postgresql/postgresql.crt
cp /etc/postgresql/ssl/client.key ~/.postgresql/postgresql.key
cp /etc/postgresql/ssl/root.crt ~/.postgresql/root.crt
chmod 600 ~/.postgresql/postgresql.key
```

**Расположения клиентских файлов (приоритет):**
1. `~/.postgresql/postgresql.crt` и `~/.postgresql/postgresql.key`
2. Указанные в переменных окружения `PGSSLCERT` и `PGSSLKEY`
3. Указанные в строке подключения `sslcert` и `sslkey`

**Концепция:** Двусторонняя аутентификация (mutual TLS) — и сервер, и клиент предъявляют сертификаты. Обеспечивает максимальную безопасность.

---

### 21. Настройка аутентификации по сертификату

```bash
sudo nano /etc/postgresql/14/main/pg_hba.conf
```

**Изменить на:**
```
hostssl  all  all  0.0.0.0/0  cert
```

**Метод `cert`:**
- Аутентификация основана только на клиентском сертификате
- Пароль не требуется
- CN сертификата должен совпадать с именем пользователя PostgreSQL
- Сертификат должен быть подписан доверенным CA (указан в `ssl_ca_file`)

**Комбинированная аутентификация:**
```
# Сертификат + пароль
hostssl  all  all  0.0.0.0/0  cert clientcert=verify-full scram-sha-256
```

**Опции clientcert:**
- `clientcert=verify-full` — требовать клиентский сертификат, проверять его
- `clientcert=verify-ca` — требовать сертификат, проверять только CA
- `clientcert=no-verify` — принимать сертификат, но не проверять

**Концепция:** Аутентификация по сертификату особенно полезна для автоматизированных систем и сервисов, где нельзя безопасно хранить пароли.

---

### 24. Режимы SSL (sslmode)

```bash
# Различные режимы подключения
psql "host=localhost dbname=postgres sslmode=disable"
psql "host=localhost dbname=postgres sslmode=allow"
psql "host=localhost dbname=postgres sslmode=prefer"
psql "host=localhost dbname=postgres sslmode=require"
psql "host=localhost dbname=postgres sslmode=verify-ca"
psql "host=localhost dbname=postgres sslmode=verify-full"
```

**Объяснение режимов:**

**`disable`** — не использовать SSL вообще
- Самый быстрый (нет накладных расходов на шифрование)
- Небезопасный (данные передаются открытым текстом)
- Использовать только для локальных подключений в защищенной сети

**`allow`** — попробовать подключиться без SSL, затем с SSL
- Устаревший режим
- Не рекомендуется (уязвим к атакам понижения версии)

**`prefer`** — попробовать SSL, затем без SSL (по умолчанию)
- Удобный, но не безопасный
- Может быть понижен до незащищенного соединения

**`require`** — обязательно использовать SSL
- Минимальный рекомендуемый уровень для продакшена
- НЕ проверяет подлинность сертификата сервера
- Защищает от прослушивания, но НЕ от атаки man-in-the-middle

**`verify-ca`** — проверить, что сертификат подписан доверенным CA
- Проверяет цепочку сертификатов
- НЕ проверяет имя хоста
- Защищает от самоподписанных сертификатов злоумышленника

**`verify-full`** — полная проверка сертификата
- Проверяет цепочку сертификатов
- Проверяет, что CN или SAN совпадает с именем хоста
- Самый безопасный режим (рекомендуется для продакшена)

**Сравнительная таблица:**

| sslmode      | Шифрование | Проверка сертификата | Проверка имени хоста | Безопасность | Применение |
|--------------|------------|----------------------|----------------------|--------------|------------|
| disable      | Нет        | Нет                  | Нет                  | Нет          | Только для тестирования |
| allow        | Опционально| Нет                  | Нет                  | Низкая       | Не рекомендуется |
| prefer       | Опционально| Нет                  | Нет                  | Низкая       | По умолчанию (не для продакшена) |
| require      | Да         | Нет                  | Нет                  | Средняя      | Минимум для продакшена |
| verify-ca    | Да         | Да                   | Нет                  | Высокая      | Внутренние сети |
| verify-full  | Да         | Да                   | Да                   | Максимальная | Рекомендуется |

**Концепция:** Выбор правильного режима SSL критичен для безопасности. В продакшене всегда используйте `require` или выше.

---

### 26. Настройка версий TLS

```ini
ssl_min_protocol_version = 'TLSv1.2'
ssl_max_protocol_version = 'TLSv1.3'
```

**Что делает:** Ограничивает допустимые версии TLS-протокола.

**Версии TLS:**
- **SSLv2, SSLv3** — устаревшие, имеют критические уязвимости (POODLE), ЗАПРЕЩЕНЫ
- **TLSv1.0** — устаревший, уязвим (BEAST, CRIME), не рекомендуется
- **TLSv1.1** — устаревший, не рекомендуется
- **TLSv1.2** — безопасный, широко поддерживается, рекомендуется как минимум
- **TLSv1.3** — современный, самый безопасный и быстрый, рекомендуется

**Улучшения TLS 1.3:**
- Более быстрое рукопожатие (меньше round-trips)
- Удалены устаревшие алгоритмы
- Улучшенная прямая секретность (Perfect Forward Secrecy)
- Встроенное шифрование рукопожатия

**Проверка поддерживаемых версий:**
```bash
openssl s_client -connect localhost:5432 -starttls postgres -tls1_2
openssl s_client -connect localhost:5432 -starttls postgres -tls1_3
```

**Концепция:** Регулярно обновляйте конфигурацию, отключая устаревшие протоколы для защиты от известных уязвимостей.

---

### 28-29. Проверка сертификата

```bash
# Проверка срока действия
openssl x509 -in /etc/postgresql/ssl/server.crt -noout -dates
```

**Вывод:**
```
notBefore=Jan 15 10:00:00 2026 GMT
notAfter=Jan 15 10:00:00 2027 GMT
```

**Просмотр всех деталей сертификата:**
```bash
openssl x509 -in /etc/postgresql/ssl/server.crt -noout -text
```

**Вывод (фрагмент):**
```
Certificate:
    Data:
        Version: 1 (0x0)
        Serial Number: 12345678901234567890 (0xabcdef123456789)
    Signature Algorithm: sha256WithRSAEncryption
        Issuer: C=RU, ST=Moscow, L=Moscow, O=MyCompany, OU=IT, CN=localhost
        Validity
            Not Before: Jan 15 10:00:00 2026 GMT
            Not After : Jan 15 10:00:00 2027 GMT
        Subject: C=RU, ST=Moscow, L=Moscow, O=MyCompany, OU=IT, CN=localhost
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus: [длинный hex]
                Exponent: 65537 (0x10001)
```

**Автоматизация проверки срока:**
```bash
#!/bin/bash
CERT_FILE="/etc/postgresql/ssl/server.crt"
EXPIRY_DATE=$(openssl x509 -in "$CERT_FILE" -noout -enddate | cut -d= -f2)
EXPIRY_EPOCH=$(date -d "$EXPIRY_DATE" +%s)
NOW_EPOCH=$(date +%s)
DAYS_LEFT=$(( ($EXPIRY_EPOCH - $NOW_EPOCH) / 86400 ))

if [ $DAYS_LEFT -lt 30 ]; then
    echo "WARNING: Certificate expires in $DAYS_LEFT days!"
fi
```

**Концепция:** Истекшие сертификаты вызовут отказ в подключении. Мониторинг и автоматическое обновление сертификатов критичны для непрерывной работы.

---

### 30. Тестирование SSL через OpenSSL

```bash
openssl s_client -connect localhost:5432 -starttls postgres
```

**Что делает:** Устанавливает SSL-соединение с PostgreSQL и показывает детали.

**Вывод (фрагмент):**
```
CONNECTED(00000003)
depth=0 C = RU, ST = Moscow, L = Moscow, O = MyCompany, OU = IT, CN = localhost
verify error:num=18:self signed certificate
verify return:1
---
Certificate chain
 0 s:/C=RU/ST=Moscow/L=Moscow/O=MyCompany/OU=IT/CN=localhost
   i:/C=RU/ST=Moscow/L=Moscow/O=MyCompany/OU=IT/CN=localhost
---
Server certificate
-----BEGIN CERTIFICATE-----
[base64 encoded certificate]
-----END CERTIFICATE-----
subject=/C=RU/ST=Moscow/L=Moscow/O=MyCompany/OU=IT/CN=localhost
issuer=/C=RU/ST=Moscow/L=Moscow/O=MyCompany/OU=IT/CN=localhost
---
No client certificate CA names sent
---
SSL handshake has read 1234 bytes and written 5678 bytes
---
New, TLSv1.3, Cipher is TLS_AES_256_GCM_SHA384
Server public key is 2048 bit
Secure Renegotiation IS NOT supported
Compression: NONE
Expansion: NONE
No ALPN negotiated
SSL-Session:
    Protocol  : TLSv1.3
    Cipher    : TLS_AES_256_GCM_SHA384
    Session-ID: [hex]
    Session-ID-ctx:
    Master-Key: [hex]
    Start Time: 1737000000
    Timeout   : 7200 (sec)
    Verify return code: 18 (self signed certificate)
```

**Ключевые моменты в выводе:**
- `Protocol: TLSv1.3` — используемая версия TLS
- `Cipher: TLS_AES_256_GCM_SHA384` — алгоритм шифрования
- `Verify return code: 18` — самоподписанный сертификат (ожидаемо для теста)
- `Server public key is 2048 bit` — размер ключа

**Концепция:** Этот инструмент помогает диагностировать проблемы с SSL без необходимости подключаться к PostgreSQL.

---

## Технический концепт: SSL/TLS в PostgreSQL

### Как работает SSL/TLS

**Фаза 1: Рукопожатие (Handshake)**

1. **Client Hello:**
   - Клиент отправляет поддерживаемые версии TLS и шифры
   - Генерирует случайное число

2. **Server Hello:**
   - Сервер выбирает версию TLS и шифр
   - Отправляет свой сертификат
   - Генерирует случайное число

3. **Проверка сертификата:**
   - Клиент проверяет подпись сертификата
   - Проверяет срок действия
   - Проверяет CN/SAN (для verify-full)

4. **Обмен ключами:**
   - Клиент генерирует pre-master secret
   - Шифрует его публичным ключом сервера
   - Отправляет серверу

5. **Генерация сессионных ключей:**
   - Обе стороны вычисляют одинаковые ключи из pre-master secret и случайных чисел
   - Используются для симметричного шифрования данных

**Фаза 2: Передача данных**

- Все данные шифруются сессионными ключами
- Используется симметричное шифрование (быстрее ассиметричного)
- Каждый пакет имеет MAC (Message Authentication Code) для целостности

**Фаза 3: Завершение**

- Безопасное закрытие соединения
- Уведомление обеих сторон

### Типы шифров

**Асимметричное шифрование (для обмена ключами):**
- **RSA** — традиционный, широко поддерживается
- **ECDHE (Elliptic Curve Diffie-Hellman Ephemeral)** — современный, обеспечивает Perfect Forward Secrecy

**Симметричное шифрование (для данных):**
- **AES (Advanced Encryption Standard)** — современный стандарт
  - AES-128 — быстрый, достаточно безопасный
  - AES-256 — более безопасный, немного медленнее
- **ChaCha20** — альтернатива AES для мобильных устройств

**Хеш-функции (для целостности):**
- **SHA-256** — современный, безопасный
- **SHA-384** — более сильный
- **MD5** — устаревший, НЕ использовать

**Режимы шифрования:**
- **GCM (Galois/Counter Mode)** — аутентифицированное шифрование, рекомендуется
- **CBC (Cipher Block Chaining)** — старый режим, уязвим к padding oracle атакам

**Рекомендуемые шифры для PostgreSQL:**
```
ssl_ciphers = 'ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256'
```

### Perfect Forward Secrecy (PFS)

**Что это:** Гарантия, что компрометация долгосрочного ключа не позволит расшифровать прошлые сессии.

**Как работает:**
- Для каждой сессии генерируются уникальные ephemeral ключи
- Ключи не сохраняются после завершения сессии
- Даже если серверный ключ украден, старый трафик остается зашифрованным

**Протоколы с PFS:**
- ECDHE (Elliptic Curve Diffie-Hellman Ephemeral)
- DHE (Diffie-Hellman Ephemeral)

**Протоколы БЕЗ PFS:**
- RSA key exchange (старый метод)

**Проверка PFS:**
```bash
openssl s_client -connect localhost:5432 -starttls postgres | grep "Cipher"
# Если содержит ECDHE или DHE — PFS включен
```

### Производительность SSL

**Накладные расходы:**
- **Рукопожатие:** 1-2 дополнительных round-trip (TLS 1.2), 0-1 (TLS 1.3)
- **CPU:** 2-10% нагрузки на шифрование/расшифровку
- **Пропускная способность:** практически не влияет (современные CPU имеют аппаратное ускорение AES)

**Оптимизация:**
- Используйте TLS 1.3 (быстрее рукопожатие)
- Используйте AES-NI (аппаратное ускорение на процессорах Intel/AMD)
- Включите session resumption (переиспользование сессий)
- Connection pooling (меньше рукопожатий)

**Проверка AES-NI:**
```bash
grep aes /proc/cpuinfo
# Если есть — аппаратное ускорение доступно
```

### Безопасность в продакшене

**Чек-лист безопасности:**

1. **Сертификаты:**
   - ✅ Использовать сертификаты от доверенного CA
   - ✅ Мониторить срок действия
   - ✅ Использовать 2048-бит (или 4096-бит) RSA ключи
   - ✅ Хранить приватные ключи с правами 0600

2. **Протоколы:**
   - ✅ Отключить SSLv2, SSLv3, TLSv1.0, TLSv1.1
   - ✅ Использовать минимум TLSv1.2
   - ✅ Предпочитать TLSv1.3

3. **Шифры:**
   - ✅ Использовать только сильные шифры
   - ✅ Отключить NULL, EXPORT, DES, 3DES, RC4
   - ✅ Предпочитать ECDHE (для PFS)
   - ✅ Использовать GCM режим

4. **Клиенты:**
   - ✅ Требовать SSL через pg_hba.conf (hostssl)
   - ✅ Использовать sslmode=verify-full
   - ✅ Проверять CN/SAN сертификата
   - ✅ Использовать клиентские сертификаты для критичных подключений

5. **Мониторинг:**
   - ✅ Логировать неудачные SSL-подключения
   - ✅ Мониторить использование слабых шифров
   - ✅ Alerting на истечение сертификатов

**Пример безопасной конфигурации:**
```ini
# postgresql.conf
ssl = on
ssl_cert_file = '/etc/postgresql/ssl/server.crt'
ssl_key_file = '/etc/postgresql/ssl/server.key'
ssl_ca_file = '/etc/postgresql/ssl/root.crt'
ssl_min_protocol_version = 'TLSv1.2'
ssl_ciphers = 'ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256'
ssl_prefer_server_ciphers = on
```

```
# pg_hba.conf
hostssl  all  all  0.0.0.0/0  scram-sha-256
hostnossl all all  0.0.0.0/0  reject
```

### Практические сценарии

**Сценарий 1: Локальное тестирование**
- Самоподписанный сертификат
- sslmode=require
- Без клиентских сертификатов

**Сценарий 2: Внутренняя корпоративная сеть**
- Сертификат от внутреннего CA
- sslmode=verify-ca
- Клиентские сертификаты для привилегированных пользователей

**Сценарий 3: Публичный облачный сервис**
- Сертификат от Let's Encrypt/DigiCert
- sslmode=verify-full (обязательно!)
- Клиентские сертификаты для всех подключений
- IP whitelisting
- Регулярная ротация сертификатов

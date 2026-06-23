# dbops-project
Исходный репозиторий для выполнения проекта дисциплины "DBOps".

## Шаг 2. Создание базы данных `store`

```sql
CREATE DATABASE store;
```

## Шаг 3. Создание пользователя и выдача прав

Пользователь `saidose` используется автотестами и Flyway-миграциями, поэтому ему
выданы права на работу со схемой `public` и всеми таблицами базы `store`.

```sql
CREATE USER saidose WITH PASSWORD 'Momo19891';
GRANT ALL PRIVILEGES ON DATABASE store TO saidose;
GRANT ALL ON SCHEMA public TO saidose;
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO saidose;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON TABLES TO saidose;
```

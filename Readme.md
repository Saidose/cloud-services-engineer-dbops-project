# dbops-project
Исходный репозиторий для выполнения проекта дисциплины "DBOps"

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

## Шаг 10. Сколько сосисок продано за каждый день предыдущей недели

Выборка из двух столбцов: дата заказа и сумма всех заказанных сосисок за этот день.
Учитываются заказы за предыдущую календарную неделю (пн–вс).

```sql
SELECT o.date_created AS order_date,
       SUM(op.quantity) AS sausages_sold
FROM orders o
JOIN order_product op ON op.order_id = o.id
WHERE o.date_created >= date_trunc('week', current_date)::date - 7
  AND o.date_created <  date_trunc('week', current_date)::date
GROUP BY o.date_created
ORDER BY o.date_created;
```

## Шаг 11. Оптимизация запроса с помощью индексов

Запрос фильтрует заказы по `orders.date_created` и джойнит `order_product` по
`order_id`. Создаём индекс на колонку даты и на внешние ключи `order_product`
(см. `migrations/V004__create_index.sql`):

```sql
CREATE INDEX idx_orders_date_created      ON orders (date_created);
CREATE INDEX idx_order_product_order_id   ON order_product (order_id);
CREATE INDEX idx_order_product_product_id ON order_product (product_id);
```

### До создания индексов

`EXPLAIN (ANALYZE)` — заказы читаются полным `Seq Scan`:

```
 ->  Parallel Seq Scan on orders o
       Filter: ((date_created < ...) AND (date_created >= ...))
       Rows Removed by Filter: 3074263
 ...
 Execution Time: 18511.069 ms
```

### После создания индексов

Фильтр по дате идёт через `Bitmap Index Scan on idx_orders_date_created`:

```
 ->  Parallel Bitmap Heap Scan on orders o
       Recheck Cond: ((date_created >= ...) AND (date_created < ...))
       ->  Bitmap Index Scan on idx_orders_date_created
             Index Cond: ((date_created >= ...) AND (date_created < ...))
 ...
 Execution Time: 8233.205 ms
```

### Вывод

Время выполнения запроса сократилось примерно **с 18 500 мс до 8 200 мс (~2.25×)**.
Полный `Seq Scan` по таблице `orders` (10 млн строк) заменился на
`Bitmap Index Scan` по `idx_orders_date_created`, что и дало основной выигрыш.


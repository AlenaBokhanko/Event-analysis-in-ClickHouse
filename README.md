# Анализ событий в ClickHouse

## Введение
Этот проект посвящен анализу событий пользователей в базе данных ClickHouse. В рамках работы была создана таблица `events`, наполненная случайными данными (1 миллион записей). Затем выполнен анализ данных с использованием SQL-запросов для извлечения ключевых метрик, включая вовлеченность пользователей, распределение событий, доход от покупок и другие важные показатели.

## Структура таблицы `events`

| Поле            | Тип данных  | Описание                                      |
|----------------|------------|-----------------------------------------------|
| event_id       | UUID       | Уникальный идентификатор события             |
| user_id        | UUID       | Уникальный идентификатор пользователя        |
| event_type     | String     | Тип события (`click`, `view`, `purchase`, `addToCart`)    |
| event_timestamp | DateTime  | Временная метка события                      |
| product_id     | UUID       | Уникальный идентификатор продукта            |
| revenue        | Float      | Доход от покупки (заполняется для `purchase`) |

## Генерация данных
При создании таблицы events и наполнении её случайными данными (1 млн строк) были учтены следующие моменты:
1) event_id - уникальное значение (создан 1 млн уникальных event_id);
2) user_id может повторяться, так как один пользователь может участвовать в различных типах событий (event_type) несколько раз;
3) product_id тоже может повторяться, так как разные пользователи могут покупать один и тот же товар;
4) суммы по ‘revenue’ заполнены только там, где event_type = 'purchase', для типов событий 'click', 'view', 'addToCart' сумма по ‘revenue’ = 0.


### SQL-скрипт для создания и заполнения таблицы
```sql
CREATE DATABASE test_database;

USE test_database;
CREATE TABLE events (
    event_id UUID,
    user_id UUID,
    event_type String,
    event_timestamp DateTime,
    product_id UUID,
    revenue Float32 )
ENGINE = MergeTree()
ORDER BY event_timestamp;

-- Генерация 1 млн строк случайных данных
INSERT INTO events
SELECT
generateUUIDv4() AS event_id, -- Уникальный идентификатор события
   arrayElement(
       (SELECT groupArray(generateUUIDv4()) FROM numbers(100000)),
       rand() % 100000 + 1
   ) AS user_id, -- Повторяющиеся user_id
   arrayElement(['click', 'view', 'purchase', 'addToCart'], rand() % 4 + 1) AS event_type,
   now() - INTERVAL rand() % (4 * 30 * 24 * 60 * 60) SECOND AS event_timestamp, -- Данные за последние 4 месяца
   arrayElement(
       (SELECT groupArray(generateUUIDv4()) FROM numbers(50000)),
       rand() % 50000 + 1
   ) AS product_id, -- Повторяющиеся product_id
    IF(event_type = 'purchase', 10 + rand() % (1000 - 10 + 1), 0) AS revenue
FROM numbers(1000000);
```

## SQL-запросы для анализа

### 1. Количество уникальных пользователей и общее количество событий за последний месяц (текущий месяц)
```sql
SELECT 
    COUNT(DISTINCT user_id) AS unique_users,
    COUNT(DISTINCT event_id) AS total_events
FROM df2
WHERE event_timestamp >= '2025-01-01' AND event_timestamp < '2025-02-01';
```

### 2. Распределение типов событий за последний месяц
```sql
SELECT
   event_type,
   COUNT(*) AS event_count
FROM df2
WHERE event_timestamp >= '2025-01-01' AND event_timestamp < '2025-02-01'
GROUP BY event_type
ORDER BY event_count DESC;
```

### 3. Общий доход от покупок и средний доход на пользователя
```sql
SELECT
   SUM(revenue) AS total_revenue,
   SUM(revenue) / COUNT(DISTINCT user_id) AS avg_revenue_per_user
FROM df2
WHERE event_timestamp >= '2025-01-01'
AND event_timestamp < '2025-02-01';
```

### 4. Топ-5 продуктов по доходу за последний месяц
```sql
SELECT
   product_id,
   SUM(revenue) AS total_revenue
FROM df2
WHERE event_type = 'purchase'
AND event_timestamp >= '2025-01-01'
AND event_timestamp < '2025-02-01'
GROUP BY product_id
ORDER BY total_revenue DESC
LIMIT 5;
```

### 5. Материализованное представление для агрегированных данных по дням
```sql
CREATE MATERIALIZED VIEW mv_daily_stats
ENGINE = SummingMergeTree()
ORDER BY event_date
AS 
SELECT
   toDate(event_timestamp) AS event_date,
   COUNT() AS total_events,
   SUMIf(revenue, event_type = 'purchase') AS total_revenue
FROM events
GROUP BY event_date;

INSERT INTO mv_daily_stats
SELECT
   toDate(event_timestamp) AS event_date,
   COUNT() AS total_events,
   SUMIf(revenue, event_type = 'purchase') AS total_revenue
FROM events
GROUP BY event_date;
```

## Оптимизация при росте данных
При увеличении объема данных до 1 миллиарда строк можно применить следующие оптимизации:
- **Использование сжатия данных**: ClickHouse поддерживает сжатие данных на уровне столбцов.
- **Создание индексов**: Индексирование ключевых полей (`event_timestamp`, `event_type`, `user_id`) для ускорения выборок.
- **Партиционирование**: Разделение таблицы по месяцам (`PARTITION BY toYYYYMM(event_timestamp)`) для ускорения запросов.
- **Материализованные представления**: Использование агрегированных данных для быстрого анализа.
- **Предзагрузка данных в кэш**: Настройка кэширования для ускоренного доступа к часто используемым данным.

## Вывод
В рамках проекта была создана и проанализирована таблица событий пользователей в ClickHouse. Проведены вычисления ключевых метрик, таких как количество пользователей, распределение событий и доход. Также рассмотрены способы оптимизации запросов для работы с большими объемами данных.


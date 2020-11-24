# Генерация данных
Иногда требуется использовать сгенерированные данные для построения корректной выборки.
Например вам нужен массив дат на каждй день прошедших двух недель.

## MSSQL
Рекурсивная генерация дат
```sql
DECLARE @start_date date = DATEADD(day, -14, CAST(GETDATE() as date)) -- Выбирает стартовую дату (текущая минус 14 дней)
DECLARE @end_date date= CAST(GETDATE() as date) -- Выбирает конечную дату (текущая)
;WITH dates AS (
    SELECT @start_date AS period_start
    UNION ALL
    SELECT DATEADD(day, 1, period_start)
    FROM dates
    WHERE DATEADD(day, 1, period_start) <= @end_date
)
SELECT * FROM dates
```

## PgSQL
Использование функции `generate_series`
```sql
with dates as (
    select gs::date period_start
    from generate_series(
                 date_trunc('week', now() - interval '2 week'), -- Выбирает стартовую дату (текущая минус 2 недели)
                 date_trunc('week', now()), -- Выбирает конечную дату (текущая)
                 interval '1 day') as gs -- Интервал 1 день
)
select * from dates
```

Результатом работы обоих решений будет таблица
| period_start |
| ------------ |
| 2020-11-09   |
| 2020-11-10   |
| 2020-11-11   |
| 2020-11-12   |
| 2020-11-13   |
| 2020-11-14   |
| 2020-11-15   |
| 2020-11-16   |
| 2020-11-17   |
| 2020-11-18   |
| 2020-11-19   |
| 2020-11-20   |
| 2020-11-21   |
| 2020-11-22   |
| 2020-11-23   |

# Пример использования
Предположим у вас есть сайт - доска объявлений. Все объявления пользователей хранятся в табличке public.offers. Ваша задача - посчитать количество размещенных каждый день объявлений.
Пример таблицы
| id  | user_id | created_date |
| --- | ------- | ------------ |
| 1   | 12      | 2020-11-09   |
| 2   | 22      | 2020-11-09   |
| 3   | 33      | 2020-11-11   |
| 4   | 44      | 2020-11-12   |
| 5   | 55      | 2020-11-12   |

Задачу можно было бы решить простым SQL
```sql
select o.created_date target_date,
       count(*) offers_quantity
from public.offers o
group by o.created_date
```
Но в таком случае вы потеряете информуию о 2020-11-10, так как в этот день не было размещения объявлений на вашем сайте. При построении графика мы увидим разрыв. Нам важно явно указать что 2020-11-10 было 0 размещений
```sql
with dates as (
    select gs::date period_start
    from generate_series(
                 date_trunc('week', now() - interval '2 week'), -- Выбирает стартовую дату (текущая минус 2 недели)
                 date_trunc('week', now()), -- Выбирает конечную дату (текущая)
                 interval '1 day') as gs -- Интервал 1 день
)
select da.period_start target_date,
       count(o.Id) offers_quantity
from dates da
left join public.offers o 
    on da.period_start = o.created_date
group by da.period_start
```
Такой запрос должен вернуть вам таблицу в которой будут данные на каждый день из 14ти сгенерированных через `generate_series` функцию
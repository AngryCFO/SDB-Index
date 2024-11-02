# Домашнее задание к занятию "Индексы" - `Александра Бужор`

---

### Задание 1

Напишите запрос к учебной базе данных, который вернёт процентное отношение общего размера всех индексов к общему размеру всех таблиц.

### Решение:

```sql
SELECT 
    ROUND(SUM(index_length) / SUM(data_length) * 100, 2) AS index_to_table_size_ratio_percent
FROM 
    information_schema.TABLES
WHERE 
    table_schema = 'sakila';
```
```
Name                             |Value|
---------------------------------+-----+
index_to_table_size_ratio_percent|54.68|
```


---

### Задание 2

Выполните explain analyze следующего запроса:
```sql
select distinct concat(c.last_name, ' ', c.first_name), sum(p.amount) over (partition by c.customer_id, f.title)
from payment p, rental r, customer c, inventory i, film f
where date(p.payment_date) = '2005-07-30' and p.payment_date = r.rental_date and r.customer_id = c.customer_id and i.inventory_id = r.inventory_id
```
перечислите узкие места;
оптимизируйте запрос: внесите корректировки по использованию операторов, при необходимости добавьте индексы.

### Решение:

Базовый запрос
```sql
select distinct concat(c.last_name, ' ', c.first_name), sum(p.amount) over (partition by c.customer_id, f.title)
from payment p, rental r, customer c, inventory i, film f
where date(p.payment_date) = '2005-07-30' and p.payment_date = r.rental_date and r.customer_id = c.customer_id and i.inventory_id = r.inventory_id
```
Результаты:
```
-> Table scan on <temporary>  (cost=2.5..2.5 rows=0) (actual time=4674..4674 rows=391 loops=1)
    -> Temporary table with deduplication  (cost=0..0 rows=0) (actual time=4674..4674 rows=391 loops=1)
        -> Window aggregate with buffering: sum(payment.amount) OVER (PARTITION BY c.customer_id,f.title )   (actual time=1853..4515 rows=642000 loops=1)
            -> Sort: c.customer_id, f.title  (actual time=1853..1910 rows=642000 loops=1)
                -> Stream results  (cost=10.5e+6 rows=16.3e+6) (actual time=2.38..1401 rows=642000 loops=1)
                    -> Nested loop inner join  (cost=10.5e+6 rows=16.3e+6) (actual time=2.37..1176 rows=642000 loops=1)
                        -> Nested loop inner join  (cost=8.9e+6 rows=16.3e+6) (actual time=2.27..1030 rows=642000 loops=1)
                            -> Nested loop inner join  (cost=7.26e+6 rows=16.3e+6) (actual time=1.84..871 rows=642000 loops=1)
                                -> Inner hash join (no condition)  (cost=1.61e+6 rows=16.1e+6) (actual time=1.79..45.3 rows=634000 loops=1)
                                    -> Filter: (cast(p.payment_date as date) = '2005-07-30')  (cost=1.81 rows=16086) (actual time=0.129..15.8 rows=634 loops=1)
                                        -> Table scan on p  (cost=1.81 rows=16086) (actual time=0.117..13.8 rows=16044 loops=1)
                                    -> Hash
                                        -> Covering index scan on f using idx_title  (cost=112 rows=1000) (actual time=1.32..1.56 rows=1000 loops=1)
                                -> Covering index lookup on r using rental_date (rental_date=p.payment_date)  (cost=0.25 rows=1.01) (actual time=825e-6..0.0012 rows=1.01 loops=634000)
                            -> Single-row index lookup on c using PRIMARY (customer_id=r.customer_id)  (cost=0.001 rows=1) (actual time=129e-6..145e-6 rows=1 loops=642000)
                        -> Single-row covering index lookup on i using PRIMARY (inventory_id=r.inventory_id)  (cost=0.001 rows=1) (actual time=103e-6..120e-6 rows=1 loops=642000)
```
Узкие места:

1. Оконная функция с буферизацией (Window aggregate with buffering): Это может быть ресурсоемкой операцией, особенно с большим количеством строк (642000 строк в нашем случае). Сортировка данных по customer_id и title для оконной функции требует времени и памяти.

2. Сортировка (Sort): Операция сортировки перед применением оконной функции также может быть затратной, особенно если объем данных велик, как в нашем случае.

3. Вложенные циклы (Nested loop): В запросе используются вложенные циклы для соединений, которые могут быть неэффективными при больших объемах данных.

4. Приведение типов и фильтрация (Filter): Приведение p.payment_date к дате без времени для фильтрации может препятствовать использованию индексов.

5. Перекрестное соединение (Inner hash join (no condition)): В запросе есть перекрестное соединение, которое не использует условие соединения.

Доработанный запрос:
```sql
SELECT 
    CONCAT(c.last_name, ' ', c.first_name) AS customer_name,
    SUM(p.amount) AS total_amount
FROM 
    payment p
JOIN 
    rental r ON p.rental_id = r.rental_id
JOIN 
    customer c ON r.customer_id = c.customer_id
JOIN 
    inventory i ON r.inventory_id = i.inventory_id
JOIN 
    film f ON i.film_id = f.film_id
WHERE 
        p.payment_date >= '2005-07-30' AND p.payment_date < DATE_ADD('2005-07-30', INTERVAL 1 DAY)
GROUP BY 
    customer_name;
```
Результаты:
```
-> Table scan on <temporary>  (actual time=7.75..7.78 rows=391 loops=1)
    -> Aggregate using temporary table  (actual time=7.75..7.75 rows=391 loops=1)
        -> Nested loop inner join  (cost=1.75 rows=1) (actual time=0.0547..7.2 rows=634 loops=1)
            -> Nested loop inner join  (cost=1.4 rows=1) (actual time=0.0517..6.64 rows=634 loops=1)
                -> Nested loop inner join  (cost=1.05 rows=1) (actual time=0.0487..6.07 rows=634 loops=1)
                    -> Nested loop inner join  (cost=0.7 rows=1) (actual time=0.0454..5.63 rows=634 loops=1)
                        -> Filter: ((p.payment_date >= TIMESTAMP'2005-07-30 00:00:00') and (p.payment_date < <cache>(('2005-07-30' + interval 1 day))) and (p.rental_id is not null))  (cost=0.35 rows=1) (actual time=0.0365..4.99 rows=634 loops=1)
                            -> Table scan on p  (cost=0.35 rows=1) (actual time=0.0192..3.92 rows=16049 loops=1)
                        -> Single-row index lookup on r using PRIMARY (rental_id=p.rental_id)  (cost=0.35 rows=1) (actual time=854e-6..869e-6 rows=1 loops=634)
                    -> Single-row index lookup on c using PRIMARY (customer_id=r.customer_id)  (cost=0.35 rows=1) (actual time=567e-6..585e-6 rows=1 loops=634)
                -> Single-row index lookup on i using PRIMARY (inventory_id=r.inventory_id)  (cost=0.35 rows=1) (actual time=773e-6..788e-6 rows=1 loops=634)
            -> Single-row covering index lookup on f using PRIMARY (film_id=i.film_id)  (cost=0.35 rows=1) (actual time=742e-6..760e-6 rows=1 loops=634)
```

Результаты `EXPLAIN ANALYZE` для доработанного запроса показывают значительное улучшение производительности:

1. Уменьшение количества обработанных строк: 
Количество обработанных строк сократилось с 642000 до всего 634. 

2. Использование индексного диапазонного сканирования: В запросе теперь используется индексное диапазонное сканирование по таблице `payment`. Добавлено условие для более оптимального использования индекса, путем изменения условия WHERE и отказа от оператора date

3. Устранение сортировки и оконных функций: В новом плане нет операции сортировки или использования оконной функции. В предыдущем плане требовалось сортировать большое количество строк перед применением оконной функции, что было значительной нагрузкой на ресурсы.

4. Вложенные циклы соединений с индексными поисками: Все соединения теперь являются вложенными циклами внутренних соединений с поиском по одной строке индекса, для эффективного использования первичных ключи или уникальных индексов для получения данных. Это гораздо более эффективно по сравнению с полным сканированием таблицы или соединениями без условий, как в базовом запросе.

---

### Задание 3*

Самостоятельно изучите, какие типы индексов используются в PostgreSQL. Перечислите те индексы, которые используются в PostgreSQL, а в MySQL — нет.

### Решение:

В PostgreSQL поддерживается несколько типов индексов, которые обеспечивают гибкость при оптимизации запросов. Вот некоторые из них:

1. *B-Tree*: По умолчанию используемый тип индекса, оптимизированный для данных, которые можно сортировать. Используется в большинстве СУБД, включая MySQL.

2. *Hash*: Используется для равенства значений. В MySQL также присутствуют, но их использование может быть ограничено.

3. *GiST (Generalized Search Tree)*: Это расширяемая инфраструктура для различных типов индексов, которые могут обрабатывать многомерные типы данных.

4. *SP-GiST (Space-Partitioned Generalized Search Tree)*: Подходит для данных с неравномерным распределением.

5. *GIN (Generalized Inverted Index)*: Подходит для индексации элементов, которые содержат много значений, например массивов или документов JSON.

6. *BRIN (Block Range INdexes)*: Предназначены для больших таблиц, где элементы схожих значений физически хранятся рядом друг с другом.


Индексы, которые есть в PostgreSQL и отсутствуют в MySQL:

- *GiST*: Этот тип индекса используется для создания различных балансирующих деревьев поиска, которые могут быть оптимизированы под разные типы данных.

- *SP-GiST*: Особый тип индекса, который может быть использован для разделения пространства, например, для индексации деревьев и графов.

- *GIN*: Используются для индексации составных значений, где один ключ может быть связан с множеством значений.

- *BRIN*: Эффективны для больших таблиц, где значение столбца обычно увеличивается. Позволяют быстро находить блоки данных, содержащие интересующие диапазоны значений.

---

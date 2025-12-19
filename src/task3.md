## Задание 3

1. Создайте таблицу с большим количеством данных:
    ```sql
    CREATE TABLE test_cluster AS 
    SELECT 
        generate_series(1,1000000) as id,
        CASE WHEN random() < 0.5 THEN 'A' ELSE 'B' END as category,
        md5(random()::text) as data;
    ```

2. Создайте индекс:
    ```sql
    CREATE INDEX test_cluster_cat_idx ON test_cluster(category);
    ```

3. Измерьте производительность до кластеризации:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM test_cluster WHERE category = 'A';
    ```
    
    *План выполнения:*
    ```
    Bitmap Heap Scan on test_cluster  (cost=59.17..7696.73 rows=5000 width=68) (actual time=13.761..110.455 rows=500781 loops=1)
    Recheck Cond: (category = 'A'::text)
    Heap Blocks: exact=8334
    ->  Bitmap Index Scan on test_cluster_cat_idx  (cost=0.00..57.92 rows=5000 width=0) (actual time=12.659..12.659 rows=500781 loops=1)
            Index Cond: (category = 'A'::text)
    Planning Time: 0.490 ms
    Execution Time: 122.485 ms
    ```
    
    *Объясните результат:*

    До кластеризации PostgreSQL использует B-tree индекс по колонке category и выбирает стратегию Bitmap Index Scan + Bitmap Heap Scan, так как условию category = 'A' соответствует примерно половина таблицы.

    Индекс применяется для построения битовой карты подходящих строк, после чего выполняется чтение соответствующих страниц таблицы.
    Однако строки с одинаковым значением category физически
    распределены по большому числу страниц (Heap Blocks: exact=8334), что приводит к значительному количеству обращений к диску и длительному времени выполнения запроса (≈122 мс)

4. Выполните кластеризацию:
    ```sql
    CLUSTER test_cluster USING test_cluster_cat_idx;
    ```
    
    *Результат:*
    ```
    [2025-12-19 03:11:01] workshop.public> CLUSTER test_cluster USING test_cluster_cat_idx
    [2025-12-19 03:11:02] completed in 897 ms
    ```
    Кластеризация физически переупорядочила строки таблицы в соответствии с порядком значений индекса category.

5. Измерьте производительность после кластеризации:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM test_cluster WHERE category = 'A';
    ```
    
    *План выполнения:*
    ```
    Bitmap Heap Scan on test_cluster  (cost=5573.62..20154.71 rows=499767 width=39) (actual time=10.037..45.709 rows=500781 loops=1)
    Recheck Cond: (category = 'A'::text)
    Heap Blocks: exact=4174
    ->  Bitmap Index Scan on test_cluster_cat_idx  (cost=0.00..5448.68 rows=499767 width=0) (actual time=9.556..9.557 rows=500781 loops=1)
            Index Cond: (category = 'A'::text)
    Planning Time: 0.165 ms
    Execution Time: 57.731 ms
    ```
    
    *Объясните результат:*

    После кластеризации PostgreSQL использует тот же тип плана
    (Bitmap Index Scan + Bitmap Heap Scan), однако физическое расположение строк в таблице изменилось.

    Строки с одинаковым значением category = 'A' теперь расположены более компактно, что видно по уменьшению числа читаемых страниц таблицы (Heap Blocks: exact снизилось с 8334 до 4174).

    Это привело к снижению количества случайных обращений к страницам таблицы и почти двукратному ускорению выполнения запроса (≈58 мс вместо ≈122 мс).

6. Сравните производительность до и после кластеризации:
    
    *Сравнение:*

    До кластеризации строки с одинаковым значением category были равномерно распределены по таблице, из-за чего выполнялось большое количество чтений страниц данных.

    После кластеризации физический порядок строк стал соответствовать порядку индекса, что значительно улучшило локальность данных и уменьшило число читаемых блоков.

    Хотя тип плана выполнения остался прежним
    (Bitmap Heap Scan), время выполнения запроса сократилось почти в 2 раза. Это демонстрирует, что кластеризация особенно эффективна для запросов с низкой селективностью, которые регулярно используют один и тот же индекс
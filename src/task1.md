# Задание 1: BRIN индексы и bitmap-сканирование

1. Удалите старую базу данных, если есть:
   ```shell
   docker compose down
   ```

2. Поднимите базу данных из src/docker-compose.yml:
   ```shell
   docker compose down && docker compose up -d
   ```

3. Обновите статистику:
   ```sql
   ANALYZE t_books;
   ```

4. Создайте BRIN индекс по колонке category:
   ```sql
   CREATE INDEX t_books_brin_cat_idx ON t_books USING brin(category);
   ```

5. Найдите книги с NULL значением category:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books WHERE category IS NULL;
   ```
   
   *План выполнения:*
   ```
   Bitmap Heap Scan on t_books  (cost=12.00..16.01 rows=1 width=33) (actual time=0.015..0.016 rows=0 loops=1)
   Recheck Cond: (category IS NULL)
   ->  Bitmap Index Scan on t_books_brin_cat_idx  (cost=0.00..12.00 rows=1 width=0) (actual time=0.013..0.013 rows=0 loops=1)
         Index Cond: (category IS NULL)
   Planning Time: 1.054 ms
   Execution Time: 0.054 ms
   ```
   
   *Объясните результат:*
   BRIN индекс используется только для отсеивания
   диапазонов страниц, где точно нет подходящих значений.
   Так как NULL-значения распределены равномерно и
   селективность условия низкая, планировщик может
   выбрать последовательное сканирование либо Bitmap Scan.
   BRIN индекс не даёт значительного выигрыша в данном случае.


6. Создайте BRIN индекс по автору:
   ```sql
   CREATE INDEX t_books_brin_author_idx ON t_books USING brin(author);
   ```

7. Выполните поиск по категории и автору:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books 
   WHERE category = 'INDEX' AND author = 'SYSTEM';
   ```
   
   *План выполнения:*
   ```
   Bitmap Heap Scan on t_books  (cost=12.16..2330.18 rows=1 width=33) (actual time=14.157..14.158 rows=0 loops=1)
   Recheck Cond: ((category)::text = 'INDEX'::text)
   Rows Removed by Index Recheck: 150000
   Filter: ((author)::text = 'SYSTEM'::text)
   Heap Blocks: lossy=1225
   ->  Bitmap Index Scan on t_books_brin_cat_idx  (cost=0.00..12.16 rows=72868 width=0) (actual time=0.068..0.069 rows=12250 loops=1)
         Index Cond: ((category)::text = 'INDEX'::text)
   Planning Time: 0.177 ms
   Execution Time: 14.182 ms
   ```
   
   *Объясните результат (обратите внимание на bitmap scan):*
   PostgreSQL использует Bitmap Index Scan по BRIN индексу
   на колонке category для предварительного отбора
   диапазонов страниц.
   Условие по author применяется как фильтр на этапе
   Bitmap Heap Scan, так как BRIN индекс по author
   не был выбран планировщиком.
   Это связано с низкой селективностью условия и
   особенностями распределения данных.



8. Получите список уникальных категорий:
   ```sql
   EXPLAIN ANALYZE
   SELECT DISTINCT category 
   FROM t_books 
   ORDER BY category;
   ```
   
   *План выполнения:*
   ```
   Sort  (cost=3100.14..3100.15 rows=6 width=7) (actual time=26.343..26.344 rows=6 loops=1)
   Sort Key: category
   Sort Method: quicksort  Memory: 25kB
   ->  HashAggregate  (cost=3100.00..3100.06 rows=6 width=7) (actual time=26.325..26.327 rows=6 loops=1)
         Group Key: category
         Batches: 1  Memory Usage: 24kB
         ->  Seq Scan on t_books  (cost=0.00..2725.00 rows=150000 width=7) (actual time=0.006..7.703 rows=150000 loops=1)
   Planning Time: 0.164 ms
   Execution Time: 26.400 ms
   ```
   
   *Объясните результат:*
   BRIN индекс не используется, так как запрос требует
   полного просмотра данных и сортировки.
   BRIN не хранит упорядоченные значения строк,
   поэтому PostgreSQL выбирает последовательное сканирование.


9. Подсчитайте книги, где автор начинается на 'S':
   ```sql
   EXPLAIN ANALYZE
   SELECT COUNT(*) 
   FROM t_books 
   WHERE author LIKE 'S%';
   ```
   
   *План выполнения:*
   ```
   Aggregate  (cost=3100.03..3100.05 rows=1 width=8) (actual time=11.400..11.402 rows=1 loops=1)
   ->  Seq Scan on t_books  (cost=0.00..3100.00 rows=14 width=0) (actual time=11.393..11.394 rows=0 loops=1)
         Filter: ((author)::text ~~ 'S%'::text)
         Rows Removed by Filter: 150000
   Planning Time: 0.201 ms
   Execution Time: 11.427 ms
   ```
   
   *Объясните результат:*
   BRIN индекс по author не используется,
   так как оператор LIKE с префиксом не позволяет
   эффективно определить диапазоны страниц.
   Селективность условия низкая, поэтому выполняется
   последовательное сканирование.


10. Создайте индекс для регистронезависимого поиска:
    ```sql
    CREATE INDEX t_books_lower_title_idx ON t_books(LOWER(title));
    ```

11. Подсчитайте книги, начинающиеся на 'O':
    ```sql
    EXPLAIN ANALYZE
    SELECT COUNT(*) 
    FROM t_books 
    WHERE LOWER(title) LIKE 'o%';
    ```
   
   *План выполнения:*
   ```
   Aggregate  (cost=3476.88..3476.89 rows=1 width=8) (actual time=44.174..44.176 rows=1 loops=1)
   ->  Seq Scan on t_books  (cost=0.00..3475.00 rows=750 width=0) (actual time=44.165..44.168 rows=1 loops=1)
         Filter: (lower((title)::text) ~~ 'o%'::text)
         Rows Removed by Filter: 149999
   Planning Time: 0.602 ms
   Execution Time: 44.202 ms
   ```
   
   *Объясните результат:*
   Несмотря на наличие функционального индекса по LOWER(title),
   планировщик выбрал последовательное сканирование.
   Это связано с низкой селективностью условия и тем,
   что ожидаемое количество подходящих строк слишком мало
   для эффективного использования индекса.
   PostgreSQL корректно предпочёл Seq Scan.



12. Удалите созданные индексы:
    ```sql
    DROP INDEX t_books_brin_cat_idx;
    DROP INDEX t_books_brin_author_idx;
    DROP INDEX t_books_lower_title_idx;
    ```

13. Создайте составной BRIN индекс:
    ```sql
    CREATE INDEX t_books_brin_cat_auth_idx ON t_books 
    USING brin(category, author);
    ```

14. Повторите запрос из шага 7:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books 
    WHERE category = 'INDEX' AND author = 'SYSTEM';
    ```
   
   *План выполнения:*
   ```
   Bitmap Heap Scan on t_books  (cost=12.16..2330.18 rows=1 width=33) (actual time=0.813..0.814 rows=0 loops=1)
   Recheck Cond: (((category)::text = 'INDEX'::text) AND ((author)::text = 'SYSTEM'::text))
   Rows Removed by Index Recheck: 8853
   Heap Blocks: lossy=73
   ->  Bitmap Index Scan on t_books_brin_cat_auth_idx  (cost=0.00..12.16 rows=72868 width=0) (actual time=0.020..0.021 rows=730 loops=1)
         Index Cond: (((category)::text = 'INDEX'::text) AND ((author)::text = 'SYSTEM'::text))
   Planning Time: 0.190 ms
   Execution Time: 0.836 ms
   ```
   
   *Объясните результат:*
   Используется составной BRIN индекс, который позволяет
   одновременно учитывать значения category и author
   при отборе диапазонов страниц.
   Это уменьшает количество просматриваемых блоков
   по сравнению с одиночными BRIN индексами.

## Задание 2

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

4. Создайте полнотекстовый индекс:
    ```sql
    CREATE INDEX t_books_fts_idx ON t_books 
    USING GIN (to_tsvector('english', title));
    ```

5. Найдите книги, содержащие слово 'expert':
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books 
    WHERE to_tsvector('english', title) @@ to_tsquery('english', 'expert');
    ```
    
    *План выполнения:*
    ```
    Bitmap Heap Scan on t_books  (cost=21.03..1335.59 rows=750 width=33) (actual time=0.133..0.134 rows=1 loops=1)
     "  Recheck Cond: (to_tsvector('english'::regconfig, (title)::text) @@ '''expert'''::tsquery)"
     Heap Blocks: exact=1
     ->  Bitmap Index Scan on t_books_fts_idx  (cost=0.00..20.84 rows=750 width=0) (actual time=0.125..0.125 rows=1 loops=1)
     "        Index Cond: (to_tsvector('english'::regconfig, (title)::text) @@ '''expert'''::tsquery)"
     Planning Time: 1.163 ms
     Execution Time: 0.180 ms
     ```
    
    *Объясните результат:*
     PostgreSQL использует GIN индекс для полнотекстового поиска.
     Индекс позволяет быстро найти документы, содержащие
     искомую лексему 'expert', без последовательного сканирования
     всей таблицы.
     Bitmap Index Scan применяется для получения множества
     подходящих строк, после чего Bitmap Heap Scan извлекает
     соответствующие данные из таблицы.


6. Удалите индекс:
    ```sql
    DROP INDEX t_books_fts_idx;
    ```

7. Создайте таблицу lookup:
    ```sql
    CREATE TABLE t_lookup (
         item_key VARCHAR(10) NOT NULL,
         item_value VARCHAR(100)
    );
    ```

8. Добавьте первичный ключ:
    ```sql
    ALTER TABLE t_lookup 
    ADD CONSTRAINT t_lookup_pk PRIMARY KEY (item_key);
    ```

9. Заполните данными:
    ```sql
    INSERT INTO t_lookup 
    SELECT 
         LPAD(CAST(generate_series(1, 150000) AS TEXT), 10, '0'),
         'Value_' || generate_series(1, 150000);
    ```

10. Создайте кластеризованную таблицу:
     ```sql
     CREATE TABLE t_lookup_clustered (
          item_key VARCHAR(10) PRIMARY KEY,
          item_value VARCHAR(100)
     );
     ```

11. Заполните её теми же данными:
     ```sql
     INSERT INTO t_lookup_clustered 
     SELECT * FROM t_lookup;
     
     CLUSTER t_lookup_clustered USING t_lookup_clustered_pkey;
     ```

12. Обновите статистику:
     ```sql
     ANALYZE t_lookup;
     ANALYZE t_lookup_clustered;
     ```

13. Выполните поиск по ключу в обычной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup WHERE item_key = '0000000455';
     ```
     
     *План выполнения:*
     ```
     Index Scan using t_lookup_pk on t_lookup  (cost=0.42..8.44 rows=1 width=23) (actual time=0.018..0.019 rows=1 loops=1)
     Index Cond: ((item_key)::text = '0000000455'::text)
     Planning Time: 0.199 ms
     Execution Time: 0.036 ms
     ```
     
     *Объясните результат:*
     Используется B-tree индекс первичного ключа.
     Поиск по точному совпадению ключа выполняется
     за логарифмическое время.
     Так как таблица не кластеризована, доступ к строке
     может потребовать дополнительного обращения к диску.


14. Выполните поиск по ключу в кластеризованной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup_clustered WHERE item_key = '0000000455';
     ```
     
     *План выполнения:*
    ``` 
     Index Scan using t_lookup_clustered_pkey on t_lookup_clustered  (cost=0.42..8.44 rows=1 width=23) (actual time=0.076..0.077 rows=1 loops=1)
     Index Cond: ((item_key)::text = '0000000455'::text)
     Planning Time: 0.134 ms
     Execution Time: 0.249 ms
     ```

     
     *Объясните результат:*

     Используется индекс первичного ключа.
     Таблица физически кластеризована по этому индексу,
     поэтому строки расположены в том же порядке,
     что и индекс.

     В данном конкретном запуске выигрыш по времени
     незначителен, однако кластеризация может снижать
     количество случайных обращений к диску при работе
     с большими объёмами данных.



15. Создайте индекс по значению для обычной таблицы:
     ```sql
     CREATE INDEX t_lookup_value_idx ON t_lookup(item_value);
     ```

16. Создайте индекс по значению для кластеризованной таблицы:
     ```sql
     CREATE INDEX t_lookup_clustered_value_idx 
     ON t_lookup_clustered(item_value);
     ```

17. Выполните поиск по значению в обычной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup WHERE item_value = 'T_BOOKS';
     ```
     
     *План выполнения:*
     ```
     Index Scan using t_lookup_value_idx on t_lookup  (cost=0.42..8.44 rows=1 width=23) (actual time=0.022..0.022 rows=0 loops=1)
     Index Cond: ((item_value)::text = 'T_BOOKS'::text)
     Planning Time: 0.224 ms
     Execution Time: 0.040 ms
     ```
     
     *Объясните результат:*
     Используется B-tree индекс по колонке item_value.
     Индекс позволяет избежать последовательного сканирования
     таблицы.
     Однако, так как таблица не кластеризована по этому индексу,
     доступ к строкам может происходить в произвольном порядке,
     что снижает эффективность кэширования.


18. Выполните поиск по значению в кластеризованной таблице:
     ```sql
     EXPLAIN ANALYZE
     SELECT * FROM t_lookup_clustered WHERE item_value = 'T_BOOKS';
     ```
     
     *План выполнения:*
     ```
     Index Scan using t_lookup_clustered_value_idx on t_lookup_clustered  (cost=0.42..8.44 rows=1 width=23) (actual time=0.027..0.027 rows=0 loops=1)
     Index Cond: ((item_value)::text = 'T_BOOKS'::text)
     Planning Time: 0.322 ms
     Execution Time: 0.058 ms
     ```
     
     *Объясните результат:*
     Так как таблица была ранее кластеризована,
     локальность данных может быть немного лучше,
     однако для индекса по item_value кластеризация
     не является определяющим фактором.


19. Сравните производительность поиска по значению в обычной и кластеризованной таблицах:
     
     *Сравнение:*
     
     Во всех случаях используется B-tree индекс,
     что обеспечивает эффективный доступ к данным.

     Кластеризация влияет на физический порядок строк
     и может улучшать локальность доступа,
     однако в данном эксперименте различия
     во времени выполнения минимальны.

     Эффект кластеризации становится заметнее
     на больших объёмах данных и при последовательных
     чтениях по одному и тому же индексу.


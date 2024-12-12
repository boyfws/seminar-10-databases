# Задание 2: Специальные случаи использования индексов

# Партиционирование и специальные случаи использования индексов

1. Удалите прошлый инстанс PostgreSQL - `docker-compose down` в папке `src` и запустите новый: `docker-compose up -d`.

2. Создайте партиционированную таблицу и заполните её данными:

    ```sql
    -- Создание партиционированной таблицы
    CREATE TABLE t_books_part (
        book_id     INTEGER      NOT NULL,
        title       VARCHAR(100) NOT NULL,
        category    VARCHAR(30),
        author      VARCHAR(100) NOT NULL,
        is_active   BOOLEAN      NOT NULL
    ) PARTITION BY RANGE (book_id);

    -- Создание партиций
    CREATE TABLE t_books_part_1 PARTITION OF t_books_part
        FOR VALUES FROM (MINVALUE) TO (50000);

    CREATE TABLE t_books_part_2 PARTITION OF t_books_part
        FOR VALUES FROM (50000) TO (100000);

    CREATE TABLE t_books_part_3 PARTITION OF t_books_part
        FOR VALUES FROM (100000) TO (MAXVALUE);

    -- Копирование данных из t_books
    INSERT INTO t_books_part 
    SELECT * FROM t_books;
    ```

3. Обновите статистику таблиц:
   ```sql
   ANALYZE t_books;
   ANALYZE t_books_part;
   ```
   
   *Результат:*
   ```
   INFO:  analyzing "public.t_books"
    INFO:  "t_books": scanned 1224 of 1224 pages, containing 150000 live rows and 3 dead rows; 30000 rows in sample, 150000 estimated total rows
    INFO:  analyzing "public.t_books_part" inheritance tree
    INFO:  "t_books_part_1": scanned 408 of 408 pages, containing 49999 live rows and 0 dead rows; 9992 rows in sample, 49999 estimated total rows
    INFO:  "t_books_part_2": scanned 408 of 408 pages, containing 50000 live rows and 0 dead rows; 9992 rows in sample, 50000 estimated total rows
    INFO:  "t_books_part_3": scanned 409 of 409 pages, containing 50001 live rows and 0 dead rows; 10016 rows in sample, 50001 estimated total rows
    INFO:  analyzing "public.t_books_part_1"
    INFO:  "t_books_part_1": scanned 408 of 408 pages, containing 49999 live rows and 0 dead rows; 30000 rows in sample, 49999 estimated total rows
    INFO:  analyzing "public.t_books_part_2"
    INFO:  "t_books_part_2": scanned 408 of 408 pages, containing 50000 live rows and 0 dead rows; 30000 rows in sample, 50000 estimated total rows
    INFO:  analyzing "public.t_books_part_3"
    INFO:  "t_books_part_3": scanned 409 of 409 pages, containing 50001 live rows and 0 dead rows; 30000 rows in sample, 50001 estimated total rows
    ANALYZE
```
4. Выполните запрос для поиска книги с id = 18:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books_part WHERE book_id = 18;
   ```
   
   *План выполнения:*
   ```
   "Seq Scan on t_books_part_1 t_books_part  (cost=0.00..1032.99 rows=1 width=32) (actual time=0.009..2.722 rows=1 loops=1)"
   ```
   
   *Объясните результат:*
   Так как мы сделали партиционирование по book_id мы можем не делать seq scan всей таблицы, а лишь по одной из патриций

5. Выполните поиск по названию книги:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books_part 
   WHERE title = 'Expert PostgreSQL Architecture';
   ```
   
   *План выполнения:*
   ```
    "Append  (cost=0.00..3100.01 rows=3 width=33) (actual time=2.875..8.728 rows=1 loops=1)"
    "->  Seq Scan on t_books_part_1  (cost=0.00..1032.99 rows=1 width=32) (actual time=2.874..2.875 rows=1 loops=1)"
    "Filter: ((title)::text = 'Expert PostgreSQL Architecture'::text)"
    "Rows Removed by Filter: 49998"
    "->  Seq Scan on t_books_part_2  (cost=0.00..1033.00 rows=1 width=33) (actual time=2.925..2.925 rows=0 loops=1)"
    "Filter: ((title)::text = 'Expert PostgreSQL Architecture'::text)"
    "Rows Removed by Filter: 50000"
    "->Seq Scan on t_books_part_3  (cost=0.00..1034.01 rows=1 width=34) (actual time=2.922..2.922 rows=0 loops=1)"
    "Filter: ((title)::text = 'Expert PostgreSQL Architecture'::text)"
    "Rows Removed by Filter: 50001"
    "Planning Time: 0.172 ms"
    "Execution Time: 8.746 ms"
    ```
   
   *Объясните результат:*
   Создание партиции по book_id не влияет на поиск по title, поэтому мы делаем seq scan по всей таблице, проходясь по всей партиции 

6. Создайте партиционированный индекс:
   ```sql
   CREATE INDEX ON t_books_part(title);
   ```
   
   *Результат:*
   ```
   CREATE INDEX
   ```

7. Повторите запрос из шага 5:
   ```sql
   EXPLAIN ANALYZE
   SELECT * FROM t_books_part 
   WHERE title = 'Expert PostgreSQL Architecture';
   ```
   
   *План выполнения:*
    ```
    "Append  (cost=0.29..24.94 rows=3 width=33) (actual time=0.776..2.187 rows=1 loops=1)"
    "->  Index Scan using t_books_part_1_title_idx on t_books_part_1  (cost=0.29..8.31 rows=1 width=32) (actual time=0.775..0.776 rows=1 loops=1)"
    "Index Cond: ((title)::text = 'Expert PostgreSQL Architecture'::text)"
    "->  Index Scan using t_books_part_2_title_idx on t_books_part_2  (cost=0.29..8.31 rows=1 width=33) (actual time=0.727..0.727 rows=0 loops=1)"
    "Index Cond: ((title)::text = 'Expert PostgreSQL Architecture'::text)"
    "->  Index Scan using t_books_part_3_title_idx on t_books_part_3  (cost=0.29..8.31 rows=1 width=34) (actual time=0.680..0.680 rows=0 loops=1)"
    "Index Cond: ((title)::text = 'Expert PostgreSQL Architecture'::text)"
    "Planning Time: 4.327 ms"
    "Execution Time: 2.214 ms"
    ```

   *Объясните результат:*
   Для каждой партиции был построен индекс, теперь мы можем использовать наш индекс для оптимизации поиска, для каждой партиции был построен отдельный индекс и мы должны пройти по всем из них 

8. Удалите созданный индекс:
   ```sql
   DROP INDEX t_books_part_title_idx;
   ```
   
   *Результат:*
   ```
   DROP INDEX
    ```

9. Создайте индекс для каждой партиции:
   ```sql
   CREATE INDEX ON t_books_part_1(title);
   CREATE INDEX ON t_books_part_2(title);
   CREATE INDEX ON t_books_part_3(title);
   ```
   
   *Результат:*
   ```
   CREATE INDEX
    ```

10. Повторите запрос из шага 5:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books_part 
    WHERE title = 'Expert PostgreSQL Architecture';
    ```
    
    *План выполнения:*
    ```
    "Append  (cost=0.29..24.94 rows=3 width=33) (actual time=0.829..2.663 rows=1 loops=1)"
    "->  Index Scan using t_books_part_1_title_idx on t_books_part_1  (cost=0.29..8.31 rows=1 width=32) (actual time=0.828..0.831 rows=1 loops=1)"
    "Index Cond: ((title)::text = 'Expert PostgreSQL Architecture'::text)"
    "->  Index Scan using t_books_part_2_title_idx on t_books_part_2  (cost=0.29..8.31 rows=1 width=33) (actual time=0.679..0.679 rows=0 loops=1)"
    "Index Cond: ((title)::text = 'Expert PostgreSQL Architecture'::text)"
    "->  Index Scan using t_books_part_3_title_idx on t_books_part_3  (cost=0.29..8.31 rows=1 width=34) (actual time=1.146..1.146 rows=0 loops=1)"
    "Index Cond: ((title)::text = 'Expert PostgreSQL Architecture'::text)"
    "Planning Time: 6.922 ms"
    "Execution Time: 2.694 ms"
    ```

    *Объясните результат:*
    Ничего не изменилось, так как индекс для партиционированной таблицы и так строится для каждой партиции отдельно 

11. Удалите созданные индексы:
    ```sql
    DROP INDEX t_books_part_1_title_idx;
    DROP INDEX t_books_part_2_title_idx;
    DROP INDEX t_books_part_3_title_idx;
    ```
    
    *Результат:*
    ```
    DROP INDEX
    ```

12. Создайте обычный индекс по book_id:
    ```sql
    CREATE INDEX t_books_part_idx ON t_books_part(book_id);
    ```
    
    *Результат:*
    ```
    CREATE INDEX
    ```

13. Выполните поиск по book_id:
    ```sql
    EXPLAIN ANALYZE
    SELECT * FROM t_books_part WHERE book_id = 11011;
    ```
    
    *План выполнения:*
    ```
    "Index Scan using t_books_part_1_book_id_idx on t_books_part_1 t_books_part  (cost=0.29..8.31 rows=1 width=32) (actual time=0.027..0.029 rows=1 loops=1)"
    "Index Cond: (book_id = 11011)"
    "Planning Time: 0.388 ms"
    "Execution Time: 0.055 ms"
    ```
    
    *Объясните результат:*
    Мы значем, в какой патриции искать данную строку, а также внутри партиции есть индекс, поэтому мы сначала переходим к индексу патриции в которой находится наше наблюдение, а потом ищем его по индексу внутри партиции 

14. Создайте индекс по полю is_active:
    ```sql
    CREATE INDEX t_books_active_idx ON t_books(is_active);
    ```
    
    *Результат:*
    ```
    CREATE INDEX
    ```

15. Выполните поиск активных книг с отключенным последовательным сканированием:
    ```sql
    SET enable_seqscan = off;
    EXPLAIN ANALYZE
    SELECT * FROM t_books WHERE is_active = true;
    SET enable_seqscan = on;
    ```
    
    *План выполнения:*
    ```
    ```    
    *Объясните результат:*
    Мы не можем использовать индекс, так как у нас бинарное значение и использование индекса неэффективно, а так как сек скан отключен, то мы не можем выполнить данный запрос 

16. Создайте составной индекс:
    ```sql
    CREATE INDEX t_books_author_title_index ON t_books(author, title);
    ```
    
    *Результат:*
    ```
    CREATE INDEX
    ```
17. Найдите максимальное название для каждого автора:
    ```sql
    EXPLAIN ANALYZE
    SELECT author, MAX(title) 
    FROM t_books 
    GROUP BY author;
    ```
    
    *План выполнения:*
    ```
    "HashAggregate  (cost=3474.00..3484.00 rows=1000 width=42) (actual time=53.380..53.484 rows=1003 loops=1)"
    "Group Key: author"
    "Batches: 1  Memory Usage: 193kB"
    "->  Seq Scan on t_books  (cost=0.00..2724.00 rows=150000 width=21) (actual time=0.008..7.554 rows=150000 loops=1)"
    "Planning Time: 0.944 ms"
    "Execution Time: 53.540 ms"
    ```
    
    *Объясните результат:*
    Сначала происходит агрегация по данным, далее происзаодится seq скан для проверки значения title, это происходит так как мы можем использовать индекс только в сортировке по author, но не по title 

18. Выберите первых 10 авторов:
    ```sql
    EXPLAIN ANALYZE
    SELECT DISTINCT author 
    FROM t_books 
    ORDER BY author 
    LIMIT 10;
    ```
    
    *План выполнения:*
    ```
    "Limit  (cost=0.42..56.63 rows=10 width=10) (actual time=4.348..5.922 rows=10 loops=1)"
    "->  Result  (cost=0.42..5621.42 rows=1000 width=10) (actual time=4.346..5.919 rows=10 loops=1)"
    "->  Unique  (cost=0.42..5621.42 rows=1000 width=10) (actual time=4.344..5.914 rows=10 loops=1)"
    "->  Index Only Scan using t_books_author_title_index on t_books  (cost=0.42..5246.42 rows=150000 width=10) (actual time=4.342..5.770 rows=1330 loops=1)"
    "Heap Fetches: 1"
    "Planning Time: 0.082 ms"
    "Execution Time: 5.938 ms"
    ```    
    *Объясните результат:*
    Мы проходимся только по индексу ищя авторов по нему

19. Выполните поиск и сортировку:
    ```sql
    EXPLAIN ANALYZE
    SELECT author, title 
    FROM t_books 
    WHERE author LIKE 'T%'
    ORDER BY author, title;
    ```
    
    *План выполнения:*
    ```
    "Sort  (cost=3099.27..3099.30 rows=14 width=21) (actual time=11.943..11.944 rows=1 loops=1)"
    "Sort Key: author, title"
    "Sort Method: quicksort  Memory: 25kB"
    "->  Seq Scan on t_books  (cost=0.00..3099.00 rows=14 width=21) (actual time=11.935..11.936 rows=1 loops=1)"
    "Filter: ((author)::text ~~ 'T%'::text)"
    "Rows Removed by Filter: 149999"
    "Planning Time: 2.339 ms"
    "Execution Time: 11.963 ms"
    ```
    *Объясните результат:*
    Сам хз
    

20. Добавьте новую книгу:
    ```sql
    INSERT INTO t_books (book_id, title, author, category, is_active)
    VALUES (150001, 'Cookbook', 'Mr. Hide', NULL, true);
    COMMIT;
    ```
    
    *Результат:*
    ```
    WARNING:  there is no transaction in progress
    COMMIT
    ```
21. Создайте индекс по категории:
    ```sql
    CREATE INDEX t_books_cat_idx ON t_books(category);
    ```
    
    *Результат:*
    ```
    CREATE INDEX
    ```

22. Найдите книги без категории:
    ```sql
    EXPLAIN ANALYZE
    SELECT author, title 
    FROM t_books 
    WHERE category IS NULL;
    ```
    
    *План выполнения:*
    ```
    "Index Scan using t_books_cat_idx on t_books  (cost=0.29..8.14 rows=1 width=21) (actual time=0.045..0.046 rows=1 loops=1)"
    "Index Cond: (category IS NULL)"
    "Planning Time: 0.443 ms"
    "Execution Time: 0.066 ms"
    ```
    
    *Объясните результат:*
    Индекс в постгрес хранит NULL как отдельное значение, поэтому мы можем эффективно его использовать для поиска элементов 

23. Создайте частичные индексы:
    ```sql
    DROP INDEX t_books_cat_idx;
    CREATE INDEX t_books_cat_null_idx ON t_books(category) WHERE category IS NULL;
    ```
    
    *Результат:*
    ```
    CREATE INDEX
    ```

24. Повторите запрос из шага 22:
    ```sql
    EXPLAIN ANALYZE
    SELECT author, title 
    FROM t_books 
    WHERE category IS NULL;
    ```
    
    *План выполнения:*
    ```
    "Index Scan using t_books_cat_null_idx on t_books  (cost=0.12..7.97 rows=1 width=21) (actual time=0.017..0.018 rows=1 loops=1)"
    "Planning Time: 0.481 ms"
    "Execution Time: 0.037 ms"
    ```

    *Объясните результат:*
    Все значение удовлетворяющие условию хранятся в индексе, поэтому мы просто сканируем его 

25. Создайте частичный уникальный индекс:
    ```sql
    CREATE UNIQUE INDEX t_books_selective_unique_idx 
    ON t_books(title) 
    WHERE category = 'Science';
    
    -- Протестируйте его
    INSERT INTO t_books (book_id, title, author, category, is_active)
    VALUES (150002, 'Unique Science Book', 'Author 1', 'Science', true);
    
    -- Попробуйте вставить дубликат
    INSERT INTO t_books (book_id, title, author, category, is_active)
    VALUES (150003, 'Unique Science Book', 'Author 2', 'Science', true);
    
    -- Но можно вставить такое же название для другой категории
    INSERT INTO t_books (book_id, title, author, category, is_active)
    VALUES (150004, 'Unique Science Book', 'Author 3', 'History', true);
    ```
    
    *Результат:*
    ```
    ERROR:  Key (title)=(Unique Science Book) already exists.duplicate key value violates unique constraint "t_books_selective_unique_idx" 

    ERROR:  duplicate key value violates unique constraint "t_books_selective_unique_idx"
    SQL state: 23505
    Detail: Key (title)=(Unique Science Book) already exists.
    ```
    
    *Объясните результат:*
    Уникальный индекс накалдывает ограничения на title только для категории Science, поэтому в 3 случае для категории History все будет ок
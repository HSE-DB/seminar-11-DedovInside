# Задание 1: BRIN индексы и bitmap-сканирование

**1.** Удалите старую базу данных, если есть:

```shell
docker compose down
```

**2.** Поднимите базу данных из src/docker-compose.yml:

```shell
docker compose down && docker compose up -d
```

**3.** Обновите статистику:

```sql
ANALYZE t_books;
```

**4.** Создайте BRIN индекс по колонке category:

```sql
CREATE INDEX t_books_brin_cat_idx ON t_books USING brin(category);
```

```md
[2025-12-17 00:21:38] workshop.public> CREATE INDEX t_books_brin_cat_idx ON t_books USING brin(category)[2025-12-17 00:21:38] completed in 26 ms
```

**5.** Найдите книги с NULL значением category:

```sql
EXPLAIN ANALYZE
SELECT * FROM t_books WHERE category IS NULL;
```

*План выполнения:*

```md
Bitmap Heap Scan on t_books  (cost=12.00..16.01 rows=1 width=33) (actual time=0.008..0.009 rows=0 loops=1)
Recheck Cond: (category IS NULL)
->  Bitmap Index Scan on t_books_brin_cat_idx  (cost=0.00..12.00 rows=1 width=0) (actual time=0.006..0.006 rows=0 loops=1)
      Index Cond: (category IS NULL)
Planning Time: 0.246 ms
Execution Time: 0.028 ms
```

*Объясните результат:*

```md
Результат означает, что BRIN-индекс мгновенно отработал и не нашёл ни одной строки с `NULL` в категории.
Bitmap Index Scan по BRIN-индексу почти мгновенно определил, что ни в одном блоке данных нет `NULL` значений в столбце category.

Bitmap Heap Scan не потребовал чтения данных с диска, так как не нашлось блоков для проверки (rows=0).

Время (0.028 мс) блестящее для таблицы в 150К строк.

Почему строк 0? Генерация данных `(ARRAY[...])` не создаёт NULL, а ручные правки только устанавливают конкретные категории. Оценка планировщика (rows=1) - это всего лишь статистическое предположение.
```

**6.** Создайте BRIN индекс по автору:

```sql
CREATE INDEX t_books_brin_author_idx ON t_books USING brin(author);
```

```md
[2025-12-17 00:23:19] workshop.public> CREATE INDEX t_books_brin_author_idx ON t_books USING brin(author)[2025-12-17 00:23:19] completed in 20 ms
```

**7.** Выполните поиск по категории и автору:

```sql
EXPLAIN ANALYZE
SELECT * FROM t_books 
WHERE category = 'INDEX' AND author = 'SYSTEM';
```

*План выполнения:*

```md
Bitmap Heap Scan on t_books  (cost=12.00..16.02 rows=1 width=33) (actual time=10.197..10.198 rows=0 loops=1)
Recheck Cond: ((category)::text = 'INDEX'::text)
Rows Removed by Index Recheck: 150000
Filter: ((author)::text = 'SYSTEM'::text)
Heap Blocks: lossy=1225
->  Bitmap Index Scan on t_books_brin_cat_idx  (cost=0.00..12.00 rows=1 width=0) (actual time=0.086..0.087 rows=12250 loops=1)
      Index Cond: ((category)::text = 'INDEX'::text)
Planning Time: 0.218 ms
Execution Time: 10.221 ms

*Объясните результат (обратите внимание на bitmap scan):*
План запроса неэффективен (10 мс для 0 строк). BRIN-индекс по category отработал, но из-за своей "грубой" природы выбрал 1225 блоков (Heap Blocks: lossy=1225), содержащих ~12250 строк. Затем системе пришлось проверить все 150 000 строк (Rows Removed by Index Recheck: 150000) внутри этих блоков, потому что:
- Условие по author (author = 'SYSTEM') не использовало индекс - индекс t_books_brin_author_idx в плане даже не появился.
- Данные в таблице не отсортированы по author, поэтому BRIN-индекс (который эффективен только для упорядоченных данных) для этого столбца бесполезен.
```

**8.** Получите список уникальных категорий:

```sql
EXPLAIN ANALYZE
SELECT DISTINCT category 
FROM t_books 
ORDER BY category;
```

*План выполнения:*

```md
Sort  (cost=3100.11..3100.12 rows=5 width=7) (actual time=19.142..19.143 rows=6 loops=1)
Sort Key: category
Sort Method: quicksort  Memory: 25kB
->  HashAggregate  (cost=3100.00..3100.05 rows=5 width=7) (actual time=19.116..19.118 rows=6 loops=1)
      Group Key: category
      Batches: 1  Memory Usage: 24kB
      ->  Seq Scan on t_books  (cost=0.00..2725.00 rows=150000 width=7) (actual time=0.006..5.189 rows=150000 loops=1)
Planning Time: 0.080 ms
Execution Time: 19.220 ms 
```

*Объясните результат:*

```md
Чтобы найти все уникальные значения столбца category, системе нужно обработать каждую строку в таблице (150 000 строк). BRIN-индекс не подходит для этой задачи, так как он хранит только сводку по диапазонам блоков (мин/макс), а не полный список значений.
```

**9.** Подсчитайте книги, где автор начинается на 'S':

```sql
EXPLAIN ANALYZE
SELECT COUNT(*) 
FROM t_books 
WHERE author LIKE 'S%';
```

*План выполнения:*

```md
Aggregate  (cost=3100.04..3100.05 rows=1 width=8) (actual time=7.305..7.307 rows=1 loops=1)
->  Seq Scan on t_books  (cost=0.00..3100.00 rows=15 width=0) (actual time=7.301..7.301 rows=0 loops=1)
      Filter: ((author)::text ~~ 'S%'::text)
      Rows Removed by Filter: 150000
Planning Time: 0.310 ms
Execution Time: 7.337 ms
```

*Объясните результат:*

```md
Оптимизатор проигнорировал BRIN-индекс t_books_brin_author_idx и выбрал последовательное сканирование (Seq Scan) всей таблицы из 150 000 строк. Причина в том, что данные в столбце author не упорядочены физически, и BRIN не может эффективно отсечь блоки для условия LIKE 'S%'.

BRIN хранит только минимум и максимум значений для каждого диапазона блоков. Для строковых данных это означает лексикографический диапазон (например, от 'Author_1' до 'Author_999').

- В нашей таблице во всех блоках автор находится в диапазоне примерно ['Author_1', 'Author_999'].

- Условие LIKE 'S%' ищет авторов, начинающихся с буквы S, но 'Author_X' лексикографически начинается с 'A'.

- Поэтому каждый блок попадает в потенциальный диапазон поиска (поскольку 'S' находится между 'A' и 'Z'), и BRIN не может отсечь ни одного блока.

Вывод: BRIN эффективен только тогда, когда данные физически упорядочены так, что искомые значения локализованы в конкретных блоках. Для случайных данных и поиска по префиксу он бесполезен.
```

**10.** Создайте индекс для регистронезависимого поиска:

```sql
CREATE INDEX t_books_lower_title_idx ON t_books(LOWER(title));
```

```md
[2025-12-17 00:51:55] workshop.public> CREATE INDEX t_books_lower_title_idx ON t_books(LOWER(title))
[2025-12-17 00:51:55] completed in 123 ms
```

**11.** Подсчитайте книги, начинающиеся на 'O':

```sql
EXPLAIN ANALYZE
SELECT COUNT(*) 
FROM t_books 
WHERE LOWER(title) LIKE 'o%';
```

*План выполнения:*

```md
Aggregate  (cost=3475.04..3475.05 rows=1 width=8) (actual time=23.533..23.534 rows=1 loops=1)
->  Seq Scan on t_books  (cost=0.00..3475.00 rows=15 width=0) (actual time=23.526..23.528 rows=1 loops=1)
      Filter: (lower((title)::text) ~~ 'o%'::text)
      Rows Removed by Filter: 149999
Planning Time: 0.225 ms
Execution Time: 23.574 ms
```

*Объясните результат:*

```md
План показывает, что созданный индекс t_books_lower_title_idx не использовался. Оптимизатор выбрал полное сканирование таблицы, так как счел это более эффективным для данного конкретного запроса.

- Индекс создан, но не использован (Seq Scan): Хотя индекс на LOWER(title) существует, PostgreSQL решил проигнорировать его и выполнил полное сканирование таблицы. Причина в том, что для поиска по префиксу (LIKE 'o%') индекс может быть неэффективен из-за особенностей распределения данных.

- Данные в таблице: Все названия книг имеют шаблон 'Book_1', 'Book_2', ..., 'Book_150000'. После применения функции LOWER() они превращаются в 'book_1', 'book_2' и т.д. Только одна книга (book_id = 3001, title = 'Oracle Core') после LOWER() начинается на букву 'o' ('oracle core'). Именно её и нашел запрос (rows=1).

- Почему сканирование таблицы?: Оптимизатор оценил, что для поиска одной строки из 150 000 быстрее прочитать всю таблицу (23.5 мс), чем использовать индекс. Индексное сканирование потребовало бы случайных чтений с диска для каждой подходящей записи, что при малом количестве результатов может быть медленнее последовательного чтения.
```

Исправим индекс:

```sql
CREATE INDEX t_books_lower_title_idx ON t_books(LOWER(title) varchar_pattern_ops);
```

```md
`varchar_pattern_ops` создает индекс, специально оптимизированный для поиска по шаблонам (LIKE, ~). Он использует другой порядок сортировки, который лучше подходит для текстовых сравнений с шаблонами.
```

*План выполнения:*

```md
Aggregate  (cost=8.48..8.49 rows=1 width=8) (actual time=0.046..0.054 rows=1 loops=1)
->  Index Scan using t_books_lower_title_idx on t_books  (cost=0.42..8.45 rows=15 width=0) (actual time=0.044..0.051 rows=1 loops=1)
      Index Cond: ((lower((title)::text) ~>=~ 'o'::text) AND (lower((title)::text) ~<~ 'p'::text))
      Filter: (lower((title)::text) ~~ 'o%'::text)
Planning Time: 1.014 ms
Execution Time: 0.116 ms
```

*Объясните результат:*

```md
- Index Scan вместо Seq Scan - запрос теперь выполняется через сканирование индекса, а не всей таблицы. Оптимизатор смог использовать новый индекс благодаря оператору `varchar_pattern_ops`.

- Умное преобразование условия - PostgreSQL преобразовал LIKE 'o%' в диапазонный поиск по индексу.
```

**12.** Удалите созданные индексы:

```sql
DROP INDEX t_books_brin_cat_idx;
DROP INDEX t_books_brin_author_idx;
DROP INDEX t_books_lower_title_idx;
```

```md
[2025-12-17 01:05:29] workshop.public> DROP INDEX t_books_brin_cat_idx
[2025-12-17 01:05:29] completed in 4 ms
[2025-12-17 01:05:29] workshop.public> DROP INDEX t_books_brin_author_idx
[2025-12-17 01:05:29] completed in 3 ms
[2025-12-17 01:05:29] workshop.public> DROP INDEX t_books_lower_title_idx
[2025-12-17 01:05:29] completed in 3 ms
```

**13.** Создайте составной BRIN индекс:

```sql
CREATE INDEX t_books_brin_cat_auth_idx ON t_books 
USING brin(category, author);
```

```md
[2025-12-17 01:06:11] workshop.public> CREATE INDEX t_books_brin_cat_auth_idx ON t_books
                                          USING brin(category, author)
[2025-12-17 01:06:12] completed in 33 ms
```

**14.** Повторите запрос из шага 7:

```sql
EXPLAIN ANALYZE
SELECT * FROM t_books 
WHERE category = 'INDEX' AND author = 'SYSTEM';
```

*План выполнения:*

```md
Bitmap Heap Scan on t_books  (cost=12.15..2305.89 rows=1 width=33) (actual time=0.702..0.703 rows=0 loops=1)
Recheck Cond: (((category)::text = 'INDEX'::text) AND ((author)::text = 'SYSTEM'::text))
Rows Removed by Index Recheck: 8883
Heap Blocks: lossy=73
->  Bitmap Index Scan on t_books_brin_cat_auth_idx  (cost=0.00..12.15 rows=71249 width=0) (actual time=0.130..0.130 rows=730 loops=1)
      Index Cond: (((category)::text = 'INDEX'::text) AND ((author)::text = 'SYSTEM'::text))
Planning Time: 0.558 ms
Execution Time: 0.866 ms
```

*Объясните результат:*

```md
План показывает, что составной BRIN-индекс значительно ускорил запрос (с 10 мс до 0.9 мс), но из-за случайного распределения данных всё равно потребовал проверки 8883 строк в 73 блоках.

- Индекс использован успешно: Составной BRIN-индекс (category, author) был задействован для поиска по обоим полям одновременно (Index Cond: ((category = 'INDEX') AND (author = 'SYSTEM'))).
- Улучшение против одиночного индекса (шаг 7). Составной индекс хранит сводки по комбинации значений. Блоки, где не встречается комбинация ('INDEX', 'SYSTEM'), были сразу отсечены.

BRIN лишь отсек блоки, где эта комбинация точно не встречается, но в отобранных блоках всё равно пришлось перепроверить все строки (операция Recheck Cond)
```

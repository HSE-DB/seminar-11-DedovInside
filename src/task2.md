# Задание 2

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

**4.** Создайте полнотекстовый индекс:

```sql
CREATE INDEX t_books_fts_idx ON t_books 
USING GIN (to_tsvector('english', title));
```

```md
[2025-12-18 14:17:16] workshop.public> CREATE INDEX t_books_fts_idx ON t_books
                                           USING GIN (to_tsvector('english', title))
[2025-12-18 14:17:17] completed in 388 ms
```

**5.** Найдите книги, содержащие слово 'expert':

```sql
EXPLAIN ANALYZE
SELECT * FROM t_books 
WHERE to_tsvector('english', title) @@ to_tsquery('english', 'expert');
```

*План выполнения:*

```md
Bitmap Heap Scan on t_books  (cost=21.03..1336.08 rows=750 width=33) (actual time=0.071..0.072 rows=1 loops=1)
"  Recheck Cond: (to_tsvector('english'::regconfig, (title)::text) @@ '''expert'''::tsquery)"
  Heap Blocks: exact=1
  ->  Bitmap Index Scan on t_books_fts_idx  (cost=0.00..20.84 rows=750 width=0) (actual time=0.047..0.048 rows=1 loops=1)
"        Index Cond: (to_tsvector('english'::regconfig, (title)::text) @@ '''expert'''::tsquery)"
Planning Time: 0.555 ms
Execution Time: 0.113 ms
```

*Объясните результат:*

```md
GIN-индекс t_books_fts_idx был успешно использован для мгновенного поиска слова 'expert' в 150 000 заголовках. Система нашла ровно одну книгу - "Expert PostgreSQL Architecture" (добавленную через UPDATE на book_id = 2025), выполнив запрос всего за 0.113 мс.

- Bitmap Index Scan on t_books_fts_idx - GIN-индекс по to_tsvector('english', title) быстро нашёл все вхождения лексемы 'expert'.

- Heap Blocks: exact=1 - Индекс точно определил один блок данных на диске, где находится искомая строка (в отличие от "lossy" в BRIN).

- Recheck Cond - Стандартный этап для bitmap-сканирования: каждая найденная строка перепроверяется на соответствие условию.

- Разница в оценках - Оптимизатор оценил rows=750, но фактически нашлось rows=1. Это нормально для полнотекстового поиска, так как сложно предсказать частоту слов.
```

**6.** Удалите индекс:

```sql
DROP INDEX t_books_fts_idx;
```

```md
[2025-12-18 14:23:17] workshop.public> DROP INDEX t_books_fts_idx
[2025-12-18 14:23:17] completed in 8 ms
```

**7.** Создайте таблицу lookup:

```sql
CREATE TABLE t_lookup (
     item_key VARCHAR(10) NOT NULL,
     item_value VARCHAR(100)
);
```

```md
[2025-12-18 14:23:45] workshop.public> CREATE TABLE t_lookup (
                                                                 item_key VARCHAR(10) NOT NULL,
                                                                 item_value VARCHAR(100)
                                       )
[2025-12-18 14:23:45] completed in 5 ms
```

**8.** Добавьте первичный ключ:

```sql
ALTER TABLE t_lookup 
ADD CONSTRAINT t_lookup_pk PRIMARY KEY (item_key);
```

```md
[2025-12-18 14:24:14] workshop.public> ALTER TABLE t_lookup
                                           ADD CONSTRAINT t_lookup_pk PRIMARY KEY (item_key)
[2025-12-18 14:24:14] completed in 11 ms
```

**9.** Заполните данными:

```sql
INSERT INTO t_lookup 
SELECT 
     LPAD(CAST(generate_series(1, 150000) AS TEXT), 10, '0'),
     'Value_' || generate_series(1, 150000);
```

```md
[2025-12-18 14:24:39] workshop.public> INSERT INTO t_lookup
                                       SELECT
                                           LPAD(CAST(generate_series(1, 150000) AS TEXT), 10, '0'),
                                           'Value_' || generate_series(1, 150000)
[2025-12-18 14:24:39] 150,000 rows affected in 257 ms
```

**10.** Создайте кластеризованную таблицу:

```sql
CREATE TABLE t_lookup_clustered (
     item_key VARCHAR(10) PRIMARY KEY,
     item_value VARCHAR(100)
);
```

```md
[2025-12-18 14:25:07] workshop.public> CREATE TABLE t_lookup_clustered (
                                                                           item_key VARCHAR(10) PRIMARY KEY,
                                                                           item_value VARCHAR(100)
                                       )
[2025-12-18 14:25:07] completed in 4 ms
```

**11.** Заполните её теми же данными:

```sql
INSERT INTO t_lookup_clustered 
SELECT * FROM t_lookup;

CLUSTER t_lookup_clustered USING t_lookup_clustered_pkey;
```

```md
[2025-12-18 14:25:38] workshop.public> INSERT INTO t_lookup_clustered
                                       SELECT * FROM t_lookup
[2025-12-18 14:25:38] 150,000 rows affected in 232 ms
[2025-12-18 14:25:38] workshop.public> CLUSTER t_lookup_clustered USING t_lookup_clustered_pkey
[2025-12-18 14:25:38] completed in 97 ms
```

**12.** Обновите статистику:

```sql
ANALYZE t_lookup;
ANALYZE t_lookup_clustered;
```

```md
[2025-12-18 14:26:05] workshop.public> ANALYZE t_lookup
[2025-12-18 14:26:05] completed in 41 ms
[2025-12-18 14:26:05] workshop.public> ANALYZE t_lookup_clustered
[2025-12-18 14:26:05] completed in 45 ms
```

**13.** Выполните поиск по ключу в обычной таблице:

```sql
EXPLAIN ANALYZE
SELECT * FROM t_lookup WHERE item_key = '0000000455';
```

*План выполнения:*

```md
Index Scan using t_lookup_pk on t_lookup  (cost=0.42..8.44 rows=1 width=23) (actual time=0.014..0.014 rows=1 loops=1)
  Index Cond: ((item_key)::text = '0000000455'::text)
Planning Time: 0.092 ms
Execution Time: 0.026 ms
```

*Объясните результат:*

```md
Для поиска одной строки по ключу '0000000455' PostgreSQL использовал Index Scan по первичному ключу t_lookup_pk. Система нашла нужную запись за 0.026 мс, выполнив два шага:

- Обращение к индексу - B-дерево индекса быстро нашло физическое расположение строки с данным ключом
- Чтение данных из таблицы - переход по указателю из индекса к конкретному месту в таблице

Быстрое время объясняется:

- Первичный ключ автоматически создаёт B-дерево индекс - оптимальную структуру для поиска по точному совпадению
- Индекс отсортирован по ключу, но данные в таблице разбросаны случайно (не кластеризованы)

Это классический случай - индекс отлично работает для точечных запросов даже при случайном расположении данных на диске
```

**14.** Выполните поиск по ключу в кластеризованной таблице:

```sql
EXPLAIN ANALYZE
SELECT * FROM t_lookup_clustered WHERE item_key = '0000000455';
```

*План выполнения:*

```md
Index Scan using t_lookup_clustered_pkey on t_lookup_clustered  (cost=0.42..8.44 rows=1 width=23) (actual time=0.049..0.050 rows=1 loops=1)
  Index Cond: ((item_key)::text = '0000000455'::text)
Planning Time: 0.121 ms
Execution Time: 0.064 ms
```

*Объясните результат:*

```md
Запрос к кластеризованной таблице выполнился медленнее (0.064 мс против 0.026 мс у обычной), хотя использовал такой же Index Scan. Причина в том, что для поиска одной конкретной записи физическое упорядочивание данных не даёт выгоды - системе всё равно нужно обратиться к индексу, а затем прочитать один блок данных с диска.

Кластеризация (упорядочивание данных на диске по первичному ключу) полезна для других типов запросов:

- Диапазонные запросы (WHERE item_key BETWEEN '00001000' AND '00002000')
- Полное сканирование таблицы с сортировкой по ключу
- Групповые операции с последовательным чтением

Для точечного поиска одной записи B-дерево индекса и так эффективно находит нужную строку, независимо от физического порядка данных в таблице.
```

**15.** Создайте индекс по значению для обычной таблицы:

```sql
CREATE INDEX t_lookup_value_idx ON t_lookup(item_value);
```

```md
[2025-12-18 14:34:54] workshop.public> CREATE INDEX t_lookup_value_idx ON t_lookup(item_value)
[2025-12-18 14:34:54] completed in 153 ms
```

**16.** Создайте индекс по значению для кластеризованной таблицы:

```sql
CREATE INDEX t_lookup_clustered_value_idx 
ON t_lookup_clustered(item_value);
```

```md
[2025-12-18 14:35:24] workshop.public> CREATE INDEX t_lookup_clustered_value_idx
                                           ON t_lookup_clustered(item_value)
[2025-12-18 14:35:25] completed in 150 ms
```

**17.** Выполните поиск по значению в обычной таблице:

```sql
EXPLAIN ANALYZE
SELECT * FROM t_lookup WHERE item_value = 'T_BOOKS';
```

*План выполнения:*

```md
Index Scan using t_lookup_value_idx on t_lookup  (cost=0.42..8.44 rows=1 width=23) (actual time=0.019..0.019 rows=0 loops=1)
Index Cond: ((item_value)::text = 'T_BOOKS'::text)
Planning Time: 0.174 ms
Execution Time: 0.032 ms
```

*Объясните результат:*

```md
Для поиска по неключевому полю item_value система выполнила Index Scan по созданному B-дерево индексу t_lookup_value_idx. Запрос быстро (0.032 мс) определил, что в таблице нет ни одной строки с таким значением (rows=0), так как:

- Все значения в столбце имеют шаблон 'Value_1', 'Value_2', ..., 'Value_150000'
- Строка 'T_BOOKS' отсутствует в данных

Ключевой вывод: Индекс по столбцу item_value работает корректно даже для поиска отсутствующих значений - он позволяет мгновенно это определить без сканирования всей таблицы.
```

**18.** Выполните поиск по значению в кластеризованной таблице:

```sql
EXPLAIN ANALYZE
SELECT * FROM t_lookup_clustered WHERE item_value = 'T_BOOKS';
```

*План выполнения:*

```md
Index Scan using t_lookup_clustered_value_idx on t_lookup_clustered  (cost=0.42..8.44 rows=1 width=23) (actual time=0.018..0.019 rows=0 loops=1)
  Index Cond: ((item_value)::text = 'T_BOOKS'::text)
Planning Time: 0.177 ms
Execution Time: 0.031 ms
```

*Объясните результат:*

```md
При поиске по item_value = 'T_BOOKS' система использовала B-дерево индекс t_lookup_clustered_value_idx, созданный на этом столбце. Запрос выполнился за 0.031 мс (практически так же, как в обычной таблице - 0.032 мс), и также не нашёл ни одной строки (rows=0).

Кластеризация не помогла:

- Кластеризация упорядочила данные только по первичному ключу (item_key), но не по item_value
- Для поиска по item_value используется отдельный индекс, который работает независимо от физического порядка данных в таблице
- Значение 'T_BOOKS' отсутствует в данных (все значения имеют вид 'Value_1', 'Value_2', ...), поэтому поиск через индекс мгновенно даёт результат

Кластеризация таблицы по одному полю (первичному ключу) не влияет на эффективность поиска по другим полям, для которых созданы отдельные индексы.
```

**19.** Сравните производительность поиска по значению в обычной и кластеризованной таблицах:

*Сравнение:*

```md
Сравнение производительности поиска по значению:

1. Время выполнения практически идентично:

- Обычная таблица: 0.032 мс
Кластеризованная таблица: 0.031 мс

Разница в 0.001 мс статистически незначима и находится в пределах погрешности измерений.

2. Одинаковые планы выполнения:

Оба запроса использовали Index Scan по B-дерево индексу, созданному на столбце item_value. Кластеризация таблицы по первичному ключу (item_key) не повлияла на поиск по другому столбцу.

3. Почему нет разницы:

- Кластеризация упорядочила данные только по item_key, но не по item_value
- Для поиска по item_value используется отдельный индекс, который работает одинаково в обоих случаях
- Поскольку искомое значение 'T_BOOKS' отсутствует в данных, оба индекса мгновенно это определили

Вывод: При поиске по неключевому полю (не по тому, по которому выполнена кластеризация) разницы в производительности между обычной и кластеризованной таблицей нет. Кластеризация помогает только при:

- Поиске по первичному ключу (если данные физически упорядочены по нему)
- Диапазонных запросах с последовательным чтением данных
- Полных сканированиях таблицы с сортировкой по кластеризованному полю
```

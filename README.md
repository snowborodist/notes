## Предыстория

За время работы над ДЗ я пришел к следующим выводам относительно использования индексов:

1. Для поиска текста от начала строки (LIKE 'something%') подходят обычные индексы с text_pattern_ops
2. Для поиска по всей строке нужно применять GIN или GIST с оператором класса gin_trgm_ops/gist_trgm_ops
3. GIST - lossy, GIN - вроде как нет.
4. GIN намного быстрее GIST при поиске.
5. GIN медленне строится и соответственно сильнее замедляет вставку данных. Но нам это ок, так как данные в таблице - информация о пользователях, и они не должны часто меняться.
6. Составные индексы бессмысленны, если во WHERE условия с OR.
7. Можно строить упорядоченные индексы для ускорения сортировки.

Возможно, у меня сложилась не совсем корректная картина, но на данный момент она такая..

## Индексы

У сервиса в логах я забрал вот такой запрос:

```sql
SELECT "users_user"."first_name",
       "users_user"."second_name",
       "users_user"."last_name",
       "users_user"."phone_number"
FROM "users_user"
WHERE (UPPER("users_user"."id"::text) = UPPER('zami') OR
       UPPER("users_user"."first_name"::text) LIKE UPPER('%zami%') OR
       UPPER("users_user"."last_name"::text) LIKE UPPER('%zami%') OR
       UPPER("users_user"."phone_number"::text) LIKE UPPER('zami%') OR
       UPPER("users_user"."email"::text) LIKE UPPER('%zami%'))
ORDER BY "users_user"."last_name" ASC;
```

Построил следующие индексы:

```sql
-- для текста по всей строке использовал GIN
CREATE UNIQUE INDEX tpo_idx_users_user_id ON users_user (UPPER(TEXT(id)) text_pattern_ops);
CREATE INDEX tpo_idx_users_user_phone_number ON users_user (UPPER(phone_number) text_pattern_ops);
CREATE INDEX gin_idx_users_user_first_name ON users_user USING GIN (UPPER(first_name) gin_trgm_ops);
CREATE INDEX gin_idx_users_user_last_name ON users_user USING GIN (UPPER(last_name) gin_trgm_ops);
CREATE INDEX gin_idx_users_user_email ON users_user USING GIN (UPPER(email) gin_trgm_ops);
-- для ORDER_BY построил вот такой:
CREATE INDEX IF NOT EXISTS idx_users_user_last_name ON users_user USING btree (last_name ASC);
```

В итоге - грусть и печаль, вот что получается в EXPAIN ALANYZE:
```
Sort  (cost=359142.14..359708.76 rows=226648 width=31) (actual time=6839.573..6840.758 rows=27219 loops=1)
  Sort Key: last_name
  Sort Method: quicksort  Memory: 3049kB
  Buffers: shared hit=129 read=21606
  ->  Bitmap Heap Scan on users_user  (cost=3113.18..338981.69 rows=226648 width=31) (actual time=32.318..6804.047 rows=27219 loops=1)
        Recheck Cond: ((upper((id)::text) = 'IVAN'::text) OR (upper((first_name)::text) ~~ '%IVAN%'::text) OR (upper((last_name)::text) ~~ '%IVAN%'::text) OR (upper((phone_number)::text) ~~ 'IVAN%'::text) OR (upper((email)::text) ~~ '%IVAN%'::text))
        Rows Removed by Index Recheck: 14
        Filter: ((upper((id)::text) = 'IVAN'::text) OR (upper((first_name)::text) ~~ '%IVAN%'::text) OR (upper((last_name)::text) ~~ '%IVAN%'::text) OR (upper((phone_number)::text) ~~ 'IVAN%'::text) OR (upper((email)::text) ~~ '%IVAN%'::text))
        Heap Blocks: exact=21633
        Buffers: shared hit=129 read=21606
        ->  BitmapOr  (cost=3113.18..3113.18 rows=229101 width=0) (actual time=25.024..25.026 rows=0 loops=1)
              Buffers: shared hit=41 read=61
              ->  Bitmap Index Scan on tpo_idx_users_user_id  (cost=0.00..4.44 rows=1 width=0) (actual time=0.006..0.007 rows=0 loops=1)
                    Index Cond: (upper((id)::text) = 'IVAN'::text)
                    Buffers: shared hit=3
              ->  Bitmap Index Scan on gin_idx_users_user_first_name  (cost=0.00..594.00 rows=63200 width=0) (actual time=7.573..7.573 rows=10303 loops=1)
                    Index Cond: (upper((first_name)::text) ~~ '%IVAN%'::text)
                    Buffers: shared hit=9 read=18
              ->  Bitmap Index Scan on gin_idx_users_user_last_name  (cost=0.00..594.00 rows=63200 width=0) (actual time=5.006..5.006 rows=2164 loops=1)
                    Index Cond: (upper((last_name)::text) ~~ '%IVAN%'::text)
                    Buffers: shared hit=8 read=14
              ->  Bitmap Index Scan on tpo_idx_users_user_phone_number  (cost=0.00..1043.43 rows=39500 width=0) (actual time=0.008..0.008 rows=0 loops=1)
                    Index Cond: ((upper((phone_number)::text) ~>=~ 'IVAN'::text) AND (upper((phone_number)::text) ~<~ 'IVAO'::text))
                    Buffers: shared hit=3
              ->  Bitmap Index Scan on gin_idx_users_user_email  (cost=0.00..594.00 rows=63200 width=0) (actual time=12.427..12.427 rows=14849 loops=1)
                    Index Cond: (upper((email)::text) ~~ '%IVAN%'::text)
                    Buffers: shared hit=18 read=29
Planning:
  Buffers: shared hit=3
Planning Time: 0.112 ms
JIT:
  Functions: 6
"  Options: Inlining false, Optimization false, Expressions true, Deforming true"
"  Timing: Generation 0.706 ms, Inlining 0.000 ms, Optimization 0.326 ms, Emission 4.643 ms, Total 5.675 ms"
Execution Time: 6842.334 ms
```

Вроде как идет сканирование по индексам, то основное время тратится на Bitmap Heap Scan (Index Recheck). И оно очень сильно зависит от количества найденных результатов. Тесты на гитлабе, естественно, не проходятся.

Непонятно как быть дальше, и где я ошибся, и какие еще есть варианты...
Нужен какой нибудь намек или идея, как все исправить =)

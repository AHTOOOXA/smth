```SQL
-- impact 60, being called every 5 minutes, avg_dur = 5s, max_dur = 8

 /* get organic search positions of products from campaigns */
with pids as (
    select distinct on (product_id) arrayJoin(products) as product_id
    from ad.campaigns where updated > now()-3600
)
select distinct on(region_id, phrase, product_id) region_id, phrase, product_id, position
from ad.positions_search where dt > now()-3600 and product_id in pids and position<100 order by dt desc
 FORMAT TSVWithNamesAndTypes
```

```
10668 rows in set. Elapsed: 3.509 sec. Processed 52.92 million rows, 2.80 GB (15.08 million rows/s., 798.51 MB/s.)
Peak memory usage: 25.62 MiB.
```

Правильно ли я понимаю что в этом запросе undefined behaviour (не особо страшный наверное) тк DISTINCT ON работает до ORDER BY - то есть могут отрезаться нужные нам самые новые даты

(дока про DISTINCT ON + ORDER BY https://clickhouse.com/docs/ru/sql-reference/statements/select/distinct#distinct-orderby)

```sql
/* К тому же таблица сортируется по возрастанию и партиции только по датам а значит оно достаточно часто может отрезаться */
create table positions_search
(
    dt         DateTime default now(),
    product_id Int32,
    phrase     String,
    region_id  Int32,
    position   Int32
)
    engine = MergeTree PARTITION BY toDate(dt)
        ORDER BY (dt, product_id, phrase, region_id)
        TTL toDate(dt) + toIntervalDay(3)
        SETTINGS index_granularity = 8192, ttl_only_drop_parts = 1;
```

Смог найти тот самый UB, вот пример: (скриншоты могут долго грузиться)
![CleanShot 2024-04-28 at 16.30.51@2x 1.png](CleanShot%202024-04-28%20at%2016.30.51%402x%201.png)
![CleanShot 2024-04-28 at 16.32.43@2x 1.png](CleanShot%202024-04-28%20at%2016.32.43%402x%201.png)

Происходит это видимо когда элементы с одновременно одинаковыми region_id, phrase, product_id попадают в 1 парт и DISTINCT ON отсекает весь хвост (самые свежие даты)

## Моя оптимизация + устранение UB 
 
В среднем раза в 1.5-2 быстрее

```sql
with pids as (select distinct on (product_id) arrayJoin(products) as product_id
              from ad.campaigns
              where updated > now() - 3600)
select region_id, phrase, product_id, position
from ad.positions_search
where (region_id, phrase, product_id, dt) in
      (select region_id, phrase, product_id, max(dt)
       from ad.positions_search
       where dt > now()-3600 and product_id in pids and position < 100
       group by region_id, phrase, product_id)
order by dt desc
    FORMAT TSVWithNamesAndTypes
```

```
9133 rows in set. Elapsed: 0.982 sec. Processed 59.68 million rows, 2.80 GB (60.78 million rows/s., 2.85 GB/s.)
Peak memory usage: 24.94 MiB.
```

## Что то странное происходит!!!
выдача стала больше тк там откуда то дубликаты по (region_id, phrase, product_id) но с разными позициями 

(они из парсинга приходят??? как могут отличаться позциии??? тут уже не разберусь наверное не зная что в целом происходит)

Примеры:
![CleanShot 2024-04-28 at 17.40.56@2x.png](CleanShot%202024-04-28%20at%2017.40.56%402x.png)
![CleanShot 2024-04-28 at 17.48.58@2x.png](CleanShot%202024-04-28%20at%2017.48.58%402x.png)

можно их доотрезать по максимальной позиции наверное.. но уже на консультацию


## Что делать дальше?
1) Возможно стоит указать в регламенте, что DISTINCT ON + ORDER выдают не всегда очевидный результат (да и в целом это не очень практика кажется)
2) Не оч понимаю откуда и для чего этот запрос, тк не вижу апу, но он 
   1) регулярный 
   2) по сути хочет вытаскивать последние по датам ентри из ad.positions_search
      то есть это **last point problem** 
      - здесь есть примем как их решать и sql запросом и in a fancy way через MaterialisedView (22:11) https://youtu.be/j15dvPGzhyE?t=1331&si=d4_RDxDym_dI8HIR

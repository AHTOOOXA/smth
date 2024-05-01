# Что было сделано
Прогнал тесты на каждый подзапрос и дописал комментами
Думаю это лучший понятно доносить и отслеживать как и где кушает перформанс
(куери к базе наверное не так и часто пишутся так что мб и неплохая идея сделать это частью кодстайла)

Здесь разобран 6й по импакту за 24 часа запрос
Прогнозирую что вместо avg dur 8 станет где то <2, запрос будет работать раза в 4 быстрее, кушать раза в 2 меньше памяти
Потенциально засейвим ~1728 секунд (30 минут) КХ в день

# Before
```sql
--initial
--#6,2304,8,22,4,288     <- это ранг, импакт, авгдюр, махдюр, миндюр, каунтер

-- TOTAL PERF
-- 6976 rows in set. Elapsed: 4.626 sec. Processed 90.49 million rows, 1.19 GB (19.56 million rows/s., 257.38 MB/s.)
-- Peak memory usage: 302.74 MiB.


with -- 87311 rows in set. Elapsed: 0.969 sec. Processed 733.62 thousand rows, 65.96 MB (843.96 thousand rows/s., 75.88 MB/s.)
     -- Peak memory usage: 44.65 MiB.
     s as (select distinct on (campaign_id) campaign_id,
                                            managed,
                                            max_price,
                                            target_position,
                                            target_position_max,
                                            min_organic_position,
                                            min_stock,
                                            region,
                                            aggressive,
                                            deposit_daily,
                                            deposit_type,
                                            daily_limit,
                                            paused,
                                            time_active,
                                            min_valuation,
                                            last_valuations_num
           from ad.campaigns_settings
           order by dt desc),
     -- 20670 rows in set. Elapsed: 0.046 sec. Processed 470.73 thousand rows, 41.53 MB (10.24 million rows/s., 903.06 MB/s.)
     -- Peak memory usage: 65.01 MiB.
     c as (select api_key_id,
                  campaign_id,
                  name,
                  campaign_type,
                  status,
                  products,
                  cpm,
                  create_dt,
                  pause_dt
           from ad.campaigns final
           where updated >= today() - 1
             and status in (9, 11)),
     ks as (select campaign_id,
                   main_keyword,
                   main_phrases.phrase          as mp_phrase,
                   main_phrases.target_position as mp_target_position,
                   main_phrases.max_price       as mp_max_price,
                   organic_control
            from ad.keywords_settings final),
     -- 7032 rows in set. Elapsed: 0.012 sec. Processed 155.15 thousand rows, 1.86 MB (13.25 million rows/s., 158.95 MB/s.)
     -- Peak memory usage: 1.02 MiB.
     b as (select distinct on (campaign_id) campaign_id, budget
           from ad.budgets
           where dt > now() - 7200
           order by dt desc),
     -- 73812 rows in set. Elapsed: 4.763 sec. Processed 85.62 million rows, 1.03 GB (17.97 million rows/s., 215.70 MB/s.)
     -- Peak memory usage: 229.61 MiB.
     sv as (select distinct on (campaign_id) campaign_id, price as saved_price from ad.saves order by dt desc),
     -- 9841 rows in set. Elapsed: 0.150 sec. Processed 3.41 million rows, 40.96 MB (22.74 million rows/s., 272.89 MB/s.)
     -- Peak memory usage: 33.76 MiB.
     cs as (select distinct on (campaign_id) campaign_id, main_catalog
            from ad.catalogs_settings
            order by dt desc)
select api_key_id,
       c.campaign_id                         AS campaign_id,
       campaign_type,
       cpm,
       status,
       managed,
       name,
       budget,
       main_keyword,
       main_catalog,
       if(saved_price = 0, cpm, saved_price) as current_price,
       products,
       create_dt,
       pause_dt,
       max_price,
       target_position,
       target_position_max,
       min_organic_position,
       min_stock,
       region                                AS region_id,
       aggressive,
       deposit_daily,
       deposit_type,
       daily_limit,
       paused,
       time_active,
       min_valuation,
       last_valuations_num,
       mp_phrase,
       mp_target_position,
       mp_max_price,
       organic_control
from c
         left join s using (campaign_id)
         left join ks on (c.campaign_id = ks.campaign_id)
         left join b on (c.campaign_id = b.campaign_id)
         left join sv on (c.campaign_id = sv.campaign_id)
         left join cs on (c.campaign_id = cs.campaign_id)
where managed
    FORMAT TSVWithNamesAndTypes
```
# После
## ВАЖНО!!!
По какой то причине выдача становится больше 9725 рядов вместо 6792 (но если вернуть s as.. из изначальной версии то размер выдачи сохраняется, голову ломал так и не понял что там добавляется)
Работает +- одинаково так что снизу закину версию со старым `s as...` которую можно в прод залить по идее уже
Если вдруг кому-то интересно как так получается то приглашаю к обсуждению
```sql
-- my version
--#6,2304,8,22,4,288
-- group by peak memory usage is ~1.5x to 10x+ times less and about 10%+ faster

-- TOTAL PERF
-- 9725 rows in set. Elapsed: 1.670 sec. Processed 91.21 million rows, 1.17 GB (54.60 million rows/s., 700.42 MB/s.)
-- Peak memory usage: 168.18 MiB.


with
    -- 83782 rows in set. Elapsed: 0.039 sec. Processed 733.68 thousand rows, 6.60 MB (18.63 million rows/s., 167.63 MB/s.)
    -- Peak memory usage: 5.48 MiB.
    needed_camps as (select campaign_id, max(dt)
                                      from ad.campaigns_settings
                                      -- IT MAY BE SMART TO CREATE A PROJECTION ON managed
                                      -- managed count: false,  136355
                                      --                true,   597318
                                      where managed
                                      group by campaign_id),
    -- // -- adding 'where campaign_id' in (select campaign_id from needed_camps) makes performance worse for some reason???
    -- 83780 rows in set. Elapsed: 0.877 sec. Processed 1.47 million rows, 72.57 MB (1.67 million rows/s., 82.71 MB/s.)
    -- Peak memory usage: 24.82 MiB.
    s as (select campaign_id,
                 managed,
                 max_price,
                 target_position,
                 target_position_max,
                 min_organic_position,
                 min_stock,
                 region,
                 aggressive,
                 deposit_daily,
                 deposit_type,
                 daily_limit,
                 paused,
                 time_active,
                 min_valuation,
                 last_valuations_num
          from ad.campaigns_settings
          where (campaign_id, dt) in needed_camps),
    -- // s as ... old version
    -- // for some reason only with this s as... query amount of rows stays the same (=6976)
    -- // with my version its =9726
    -- s as (select distinct on (campaign_id) campaign_id,
    --                                             managed,
    --                                             max_price,
    --                                             target_position,
    --                                             target_position_max,
    --                                             min_organic_position,
    --                                             min_stock,
    --                                             region,
    --                                             aggressive,
    --                                             deposit_daily,
    --                                             deposit_type,
    --                                             daily_limit,
    --                                             paused,
    --                                             time_active,
    --                                             min_valuation,
    --                                             last_valuations_num
    --            from ad.campaigns_settings
    --            order by dt desc),
    -- 20670 rows in set. Elapsed: 0.038 sec. Processed 497.96 thousand rows, 44.01 MB (13.04 million rows/s., 1.15 GB/s.)
    -- Peak memory usage: 45.53 MiB.
    c as (select api_key_id,
                 campaign_id,
                 name,
                 campaign_type,
                 status,
                 products,
                 cpm,
                 create_dt,
                 pause_dt
          from ad.campaigns final
               -- IT MAY BE SMART TO CREATE A PROJECTION ON status
               -- status, count
               -- 4,      2340
               -- 7,      218155
               -- 8,      3645
               -- 9,      36807
               -- 11,     62486
              prewhere status in (9, 11)
            and updated >= today() - 1),
    ks as (select campaign_id,
                  main_keyword,
                  main_phrases.phrase          as mp_phrase,
                  main_phrases.target_position as mp_target_position,
                  main_phrases.max_price       as mp_max_price,
                  organic_control
           from ad.keywords_settings final),
    -- 7032 rows in set. Elapsed: 0.011 sec. Processed 155.15 thousand rows, 1.86 MB (14.21 million rows/s., 170.52 MB/s.)
    -- Peak memory usage: 1.11 MiB.
    b as (select campaign_id, argMax(budget, dt) as budget
          from ad.budgets
          where dt > now() - 7200
          group by campaign_id),
    -- 73812 rows in set. Elapsed: 2.339 sec. Processed 85.62 million rows, 1.03 GB (36.60 million rows/s., 439.24 MB/s.)
    -- Peak memory usage: 49.97 MiB.
    sv as (select campaign_id, argMax(price, dt) as saved_price
           from ad.saves
           group by campaign_id),
    -- 9841 rows in set. Elapsed: 0.021 sec. Processed 3.41 million rows, 40.96 MB (160.21 million rows/s., 1.92 GB/s.)
    -- Peak memory usage: 3.66 MiB.
    cs as (select campaign_id, argMax(main_catalog, dt) as main_catalog
           from ad.catalogs_settings
           group by campaign_id)
select api_key_id,
       c.campaign_id                         AS campaign_id,
       campaign_type,
       cpm,
       status,
       managed,
       name,
       budget,
       main_keyword,
       main_catalog,
       if(saved_price = 0, cpm, saved_price) as current_price,
       products,
       create_dt,
       pause_dt,
       max_price,
       target_position,
       target_position_max,
       min_organic_position,
       min_stock,
       region                                AS region_id,
       aggressive,
       deposit_daily,
       deposit_type,
       daily_limit,
       paused,
       time_active,
       min_valuation,
       last_valuations_num,
       mp_phrase,
       mp_target_position,
       mp_max_price,
       organic_control
from c
         left join s using (campaign_id)
         left join ks on (c.campaign_id = ks.campaign_id)
         left join b on (c.campaign_id = b.campaign_id)
         left join sv on (c.campaign_id = sv.campaign_id)
         left join cs on (c.campaign_id = cs.campaign_id)
where managed
    FORMAT TSVWithNamesAndTypes
```

# Версия со старым `s as...`
```sql
-- my version
--#6,2304,8,22,4,288
-- group by peak memory usage is ~1.5x to 10x+ times less and about 10%+ faster

-- TOTAL PERF
-- 6972 rows in set. Elapsed: 1.596 sec. Processed 90.44 million rows, 1.16 GB (56.66 million rows/s., 726.11 MB/s.)
-- Peak memory usage: 160.63 MiB.


with
     -- 87311 rows in set. Elapsed: 0.969 sec. Processed 733.62 thousand rows, 65.96 MB (843.96 thousand rows/s., 75.88 MB/s.)
     -- Peak memory usage: 44.65 MiB.
     s as (select distinct on (campaign_id) campaign_id,
                                            managed,
                                            max_price,
                                            target_position,
                                            target_position_max,
                                            min_organic_position,
                                            min_stock,
                                            region,
                                            aggressive,
                                            deposit_daily,
                                            deposit_type,
                                            daily_limit,
                                            paused,
                                            time_active,
                                            min_valuation,
                                            last_valuations_num
           from ad.campaigns_settings
           order by dt desc),
    c as (select api_key_id,
                 campaign_id,
                 name,
                 campaign_type,
                 status,
                 products,
                 cpm,
                 create_dt,
                 pause_dt
          from ad.campaigns final
               -- IT MAY BE SMART TO CREATE A PROJECTION ON status
               -- status, count
               -- 4,      2340
               -- 7,      218155
               -- 8,      3645
               -- 9,      36807
               -- 11,     62486
              prewhere status in (9, 11)
            and updated >= today() - 1),
    ks as (select campaign_id,
                  main_keyword,
                  main_phrases.phrase          as mp_phrase,
                  main_phrases.target_position as mp_target_position,
                  main_phrases.max_price       as mp_max_price,
                  organic_control
           from ad.keywords_settings final),
    -- 7032 rows in set. Elapsed: 0.011 sec. Processed 155.15 thousand rows, 1.86 MB (14.21 million rows/s., 170.52 MB/s.)
    -- Peak memory usage: 1.11 MiB.
    b as (select campaign_id, argMax(budget, dt) as budget
          from ad.budgets
          where dt > now() - 7200
          group by campaign_id),
    -- 73812 rows in set. Elapsed: 2.339 sec. Processed 85.62 million rows, 1.03 GB (36.60 million rows/s., 439.24 MB/s.)
    -- Peak memory usage: 49.97 MiB.
    sv as (select campaign_id, argMax(price, dt) as saved_price
           from ad.saves
           group by campaign_id),
    -- 9841 rows in set. Elapsed: 0.021 sec. Processed 3.41 million rows, 40.96 MB (160.21 million rows/s., 1.92 GB/s.)
    -- Peak memory usage: 3.66 MiB.
    cs as (select campaign_id, argMax(main_catalog, dt) as main_catalog
           from ad.catalogs_settings
           group by campaign_id)
select api_key_id,
       c.campaign_id                         AS campaign_id,
       campaign_type,
       cpm,
       status,
       managed,
       name,
       budget,
       main_keyword,
       main_catalog,
       if(saved_price = 0, cpm, saved_price) as current_price,
       products,
       create_dt,
       pause_dt,
       max_price,
       target_position,
       target_position_max,
       min_organic_position,
       min_stock,
       region                                AS region_id,
       aggressive,
       deposit_daily,
       deposit_type,
       daily_limit,
       paused,
       time_active,
       min_valuation,
       last_valuations_num,
       mp_phrase,
       mp_target_position,
       mp_max_price,
       organic_control
from c
         left join s using (campaign_id)
         left join ks on (c.campaign_id = ks.campaign_id)
         left join b on (c.campaign_id = b.campaign_id)
         left join sv on (c.campaign_id = sv.campaign_id)
         left join cs on (c.campaign_id = cs.campaign_id)
where managed
    FORMAT TSVWithNamesAndTypes
```

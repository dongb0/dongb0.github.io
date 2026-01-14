---
layout: post
title: "adb pg 实验记录"
subtitle: 
author: "Dongbo"
header-style: text
mathjax: true
hidden: false
tags:
  - 
---


### 连接

```
# adb pg 7.0
psql -h gp-bp1v7cap41o798gsko-master.gpdb.rds.aliyuncs.com -p 5432 -d adb_sampledata_tpch -U batina 

# adv pg 6.x # pswd:batina10+10=100
psql -U batina -d imdb -h gp-bp11q414y016s3950o-master.gpdb.rds.aliyuncs.com

# 新，0328申请
psql -U batina -d imdb -h gp-bp1p6hxq1zr7wx9poo-master.gpdb.rds.aliyuncs.com

# 新，0428
psql -U batina -d batina -h gp-bp1by5b8b5y776735o-master.gpdb.rds.aliyuncs.com


# 0525 建霖
psql -U batina -d batina -h gp-bp17mi6udfadj1151o-master.gpdb.rds.aliyuncs.com

# 0529 智良
psql -U batina -d batina -h gp-bp1eiz58wwmc60q41o-master.gpdb.rds.aliyuncs.com

# 0611 B神
gp-bp1052r6636e45748o-master.gpdb.rds.aliyuncs.com

# 0612 老马
gp-bp10l185r4cd03qn7o-master.gpdb.rds.aliyuncs.com

# 0614 张芝德
gp-2ze6p4i06ywo34139o-master.gpdb.rds.aliyuncs.com

# 实验室服务器
ssh dongbowen@47.98.59.63 -p 9994

# 开启普通物化视图重写的命令
SET enable_matview_query_rewrite TO on;
ALTER MATERIALIZED VIEW {mv_name} SET (enable_query_rewrite=true);

# 关闭inc视图重写 https://help.aliyun.com/document_detail/368342.html
SET enable_incremental_matview_query_rewrite TO off;

# 会话级别开启查询缓存 https://help.aliyun.com/document_detail/399711.html
SET rds_session_use_query_cache = on;

# 查看postgres隔离等级
SELECT current_setting('default_transaction_isolation');

```

测试物化视图重写

CREATE TABLE a (a1 int, a2 int, a3 int);
CREATE TABLE b (b1 int, b2 int, b3 int);
CREATE TABLE c (c1 int, c2 int, c3 int);

INSERT INTO a VALUES(generate_series(1,20), generate_series(1,30), generate_series(1,50));
INSERT INTO b VALUES(generate_series(1,20), generate_series(1,30), generate_series(1,50));
INSERT INTO c VALUES(generate_series(1,20), generate_series(1,30), generate_series(1,50));


EXPLAIN
SELECT a1, b1, c1, count(a1)
FROM a JOIN b ON a1=b1
       JOIN c ON b2=c3
WHERE a1 <=10
GROUP BY a1, b1, c1;

EXPLAIN
SELECT a1, b1
FROM a join b on a1=b1;

EXPLAIN
SELECT a.a1, b1, c1, count(a.a1)
FROM a JOIN b ON a1=b1
       JOIN c ON b2=c3
       JOIN a AS na ON c3=na.a1
WHERE a.a1 <=10
GROUP BY a.a1, b1, c1;

SET enable_matview_query_rewrite TO on;

CREATE INCREMENTAL MATERIALIZED VIEW mv1 AS
SELECT * FROM a JOIN b ON a1=b1 JOIN c ON b2=c3;

CREATE INCREMENTAL MATERIALIZED VIEW mv2 AS
SELECT * FROM a JOIN b ON a1=b1;


```
# 测试candidate之一

SELECT `movie_companies`.`note` AS `production_note`, `title`.`title` AS `movie_title`, `title`.`production_year` AS `movie_year`
FROM `company_type`,
`info_type`,
`movie_companies`,
`movie_info_idx`,
`title`
WHERE `company_type`.`kind` = 'production companies' AND `info_type`.`info` = 'top 250 rank' AND `movie_companies`.`note` NOT LIKE '%(as Metro-Goldwyn-Mayer Pictures)%' AND `company_type`.`id` = `movie_companies`.`company_type_id` AND `title`.`id` = `movie_companies`.`movie_id` AND `title`.`id` = `movie_info_idx`.`movie_id` AND `movie_companies`.`movie_id` = `movie_info_idx`.`movie_id` AND `info_type`.`id` = `movie_info_idx`.`info_type_id`;

# 第一个创建 mv 的语句
CREATE INCREMENTAL MATERIALIZED VIEW IF NOT EXISTS mv_0 as SELECT movie_companies.note, title.title, title.production_year
FROM company_type,
info_type,
movie_companies,
movie_info_idx,
title
WHERE company_type.kind = 'production companies' AND movie_companies.note NOT LIKE '%(as Metro-Goldwyn-Mayer Pictures)%' AND company_type.id = movie_companies.company_type_id AND title.id = movie_companies.movie_id AND title.id = movie_info_idx.movie_id AND movie_companies.movie_id = movie_info_idx.movie_id AND info_type.id = movie_info_idx.info_type_id 


```

### 数据导入


#### IMDB
https://dataverse.harvard.edu/dataset.xhtml?persistentId=doi:10.7910/DVN/2QYZBT
本地 Download目录下的 imdb_pg11 文件

```
# 用的是管理账户batina，所以权限都是有的
pg_restore -U batina -d imdb -h gp-bp1by5b8b5y776735o-master.gpdb.rds.aliyuncs.com  imdb_pg11  --clean --create --verbose
```

```
# 需要 batina 有 create DB 的权限 \du 查看 # 将创建名称为 imdb 的数据库并向其中写入数据
pg_restore -U batina -d batina_base -h huawei-cloud imdb_pg11 --clean --create --verbose
```


### 问题


mv_28 15GB 也太 tm 大了

sql 20 的

CREATE INCREMENTAL MATERIALIZED VIEW mv_28 as 
SELECT keyword.keyword, name.name, title.titleFROM cast_info,keyword,movie_keyword,name,title 
WHERE keyword.id = movie_keyword.keyword_id AND title.id = movie_keyword.movie_id AND title.id = cast_info.movie_id AND cast_info.movie_id = movie_keyword.movie_id AND name.id = cast_info.person_id

预估 rows 有八千万行（ get rows * width 大于阈值，则不创建，这开销也太大了，执行一次半个小时）

SELECT keyword.keyword, name.name, title.title
FROM cast_info,
keyword,
movie_keyword,
name,
title
WHERE keyword.id = movie_keyword.keyword_id AND title.id = movie_keyword.movie_id AND title.id = cast_info.movie_id AND cast_info.movie_id = movie_keyword.movie_id AND name.id = cast_info.person_id


```
// index too big
// 可能是 incremental mv 会创建索引，但是这里两个/三个 string？
// 应该是 incremental mv 的设置问题，下次查查文档

// 太大了，1亿8千万行，24GB,估计出来的代价居然只有800w 900MB
// 哪怕是 select,也才估计1600w
// 是不是需要一些 anylze 来更新一下啊？

SELECT movie_info.info AS info1, movie_info_idx.info AS info2, title.title
FROM cast_info,
info_type,
info_type AS info_type0,
movie_info,
movie_info_idx,
name,
title
WHERE info_type0.info = 'votes' AND name.gender = 'm' AND title.id = movie_info.movie_id AND title.id = movie_info_idx.movie_id AND title.id = cast_info.movie_id AND cast_info.movie_id = movie_info.movie_id AND cast_info.movie_id = movie_info_idx.movie_id AND movie_info.movie_id = movie_info_idx.movie_id AND name.id = cast_info.person_id AND info_type.id = movie_info.info_type_id AND info_type0.id = movie_info_idx.info_type_id;
```

高收益候选（tmp，对60、61、62、64高收益
```
SELECT name.name AS name0
FROM cast_info,
company_name,
keyword,
movie_companies,
movie_keyword,
name,
title
WHERE keyword.keyword = 'character-name-in-title' AND name.id = cast_info.person_id AND cast_info.movie_id = title.id AND title.id = movie_keyword.movie_id AND movie_keyword.keyword_id = keyword.id AND title.id = movie_companies.movie_id AND movie_companies.company_id = company_name.id AND cast_info.movie_id = movie_companies.movie_id AND cast_info.movie_id = movie_keyword.movie_id AND movie_companies.movie_id = movie_keyword.movie_id 
```

以及这样的计划，哪个才是实际要看的cost啊？

这个 index only scan 的cost，算在开销内吗？

                                              QUERY PLAN                                              
------------------------------------------------------------------------------------------------------
 Result  (cost=0.00..0.01 rows=1 width=0)
   InitPlan 1 (returns $0)  (slice2)
     ->  Limit  (cost=0.18..302.53 rows=1 width=15)
           ->  Gather Motion 2:1  (slice1; segments: 2)  (cost=0.18..231866.05 rows=767 width=15)
                 Merge Key: ((mv0.name0)::text)
                 ->  Index Only Scan using mv0_index on mv0  (cost=0.18..231866.05 rows=384 width=15)
                       Index Cond: (name0 IS NOT NULL)
                       Filter: ((name0)::text ~~ 'Z%'::text)
 Optimizer: Postgres query optimizer
(9 rows)

 QUERY PLAN                                                                                                    
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Aggregate  (cost=0.00..7887.40 rows=1 width=16)
   ->  Gather Motion 2:1  (slice7; segments: 2)  (cost=0.00..7887.40 rows=1 width=16)
         ->  Aggregate  (cost=0.00..7887.40 rows=1 width=16)
               ->  Hash Join  (cost=0.00..7879.80 rows=2261965 width=15)
                     Hash Cond: (name.id = cast_info.person_id)


bad mv candidate: mv11
SELECT keyword.keyword, name.name, title.title
FROM cast_info,
keyword,
movie_keyword,
name,
title
WHERE keyword.id = movie_keyword.keyword_id 
AND title.id = movie_keyword.movie_id 
AND title.id = cast_info.movie_id 
AND cast_info.movie_id = movie_keyword.movie_id 
AND name.id = cast_info.person_id 

-- mv27:
SELECT movie_info.info AS info1, movie_info_idx.info AS info2, title.title
FROM cast_info,
info_type,
info_type AS info_type0,
movie_info,
movie_info_idx,
name,
title
WHERE title.id = movie_info.movie_id 
AND title.id = movie_info_idx.movie_id 
AND title.id = cast_info.movie_id 
AND cast_info.movie_id = movie_info.movie_id 
AND cast_info.movie_id = movie_info_idx.movie_id 
AND movie_info.movie_id = movie_info_idx.movie_id 
AND name.id = cast_info.person_id 
AND info_type.id = movie_info.info_type_id 
AND info_type0.id = movie_info_idx.info_type_id  


-- mv28: sql66+sql67+sql68
SELECT movie_info.info AS info1, movie_info_idx.info AS info2, title.title
FROM cast_info,
info_type,
info_type AS info_type0,
movie_info,
movie_info_idx,
name,
title
WHERE info_type0.info = 'votes' 
AND name.gender = 'm' 
AND title.id = movie_info.movie_id 
AND title.id = movie_info_idx.movie_id 
AND title.id = cast_info.movie_id 
AND cast_info.movie_id = movie_info.movie_id 
AND cast_info.movie_id = movie_info_idx.movie_id 
AND movie_info.movie_id = movie_info_idx.movie_id 
AND name.id = cast_info.person_id 
AND info_type.id = movie_info.info_type_id 
AND info_type0.id = movie_info_idx.info_type_id 
-- 这个查询有 1亿8千万 行，24GB大，要跑10分钟，夸张，应该说毫无用处


```
 Aggregate
   ->  Gather Motion 2:1  (slice5; segments: 2)
         ->  Aggregate
               ->  Hash Join
                     Hash Cond: (title.id = movie_info_idx.movie_id)
                     ->  Hash Join
                           Hash Cond: (title.id = movie_companies.movie_id)
                           ->  Seq Scan on title
                           ->  Hash
                                 ->  Redistribute Motion 2:2  (slice2; segments: 2)
                                       Hash Key: movie_companies.movie_id
                                       ->  Nested Loop
                                             Join Filter: true
                                             ->  Broadcast Motion 2:2  (slice1; segments: 2)
                                                   ->  Seq Scan on company_type
                                                         Filter: ((kind)::text = 'production companies'::text)
                                             ->  Index Scan using company_type_id_movie_companies on movie_companies
                                                   Index Cond: (company_type_id = company_type.id)
                                                   Filter: (((note)::text !~~ '%(as Metro-Goldwyn-Mayer Pictures)%'::text) AND (((note)::text ~~ '%(co-production)%'::text) OR ((note)::text ~~ '%(presents)%'::text)))
                     ->  Hash
                           ->  Redistribute Motion 2:2  (slice4; segments: 2)
                                 Hash Key: movie_info_idx.movie_id
                                 ->  Nested Loop
                                       Join Filter: true
                                       ->  Broadcast Motion 2:2  (slice3; segments: 2)
                                             ->  Seq Scan on info_type
                                                   Filter: ((info)::text = 'top 250 rank'::text)
                                       ->  Index Scan using info_type_id_movie_info_idx on movie_info_idx
                                             Index Cond: (info_type_id = info_type.id)
```


```

               ->  Hash Join
                           Hash Cond: (title.id = movie_companies.movie_id)
                           ->  Seq Scan on title
                           ->  Hash
                                 ->  Redistribute Motion 2:2  
                                       Hash Key: movie_companies.movie_id
                                       ->  Nested Loop
                                             ->  Broadcast Motion 2:2  
                                                   ->  Seq Scan on company_type
                                             ->  Index Scan using company_type_id_movie_companies on movie_companies
                                                   Index Cond: (company_type_id = company_type.id)
                                 ->  Nested Loop
                                       Join Filter: true
                                       -> Broadcast Motion 2:2
                                             ->  Seq Scan on info_type
                                                   Filter: ((info)::text = 'top 250 rank'::text)
                                       ->  Index Scan using info_type_id_movie_info_idx on movie_info_idx
                                             Index Cond: (info_type_id = info_type.id)
```
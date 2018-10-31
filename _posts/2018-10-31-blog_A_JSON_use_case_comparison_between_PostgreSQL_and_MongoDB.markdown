---
layout: post
title:  "A JSON use case comparison between PostgreSQL and MongoDB"
date:   2018-10-31 12:14:56 +0200
categories: PostgreSQL MongoDB Performances Benchmarking
author: Fabio Pardi and Wouter van Teijlingen
---



![Mongo_VS_Postgres](https://raw.githubusercontent.com/Portavita/portavita.github.io/master/img/mongo_vs_postgres.jpeg)




Here at [Portavita][portavita] we work with a lot of data in the JSON format and we store it in MongoDB, a open source, non-relational database (NoSQL) born in 2007.


Since version 9.2, also PostgreSQL is able to talk JSON and several progresses have been made over the years making PostgreSQL more and more production ready to serve JSON efficiently.

Very recently, a new project has been kickstarted here at Portavita, and since Portavita uses both PostgreSQL and MongoDB, the following question arose spontaneously: which database is the most suitable to support our use cases?

Me and my colleague Wouter discussed the topic and did some preliminary research trying to compare on query execution speed and disk space consumption.
After all that reading, learning, asking and studying, the conclusion was that we did not have a clue if Mongo was still the best choice to store our JSON data.

That is the reason behind the decision to perform some tests by ourself and start discussing with some numbers at hand.

So here we are sharing our results for your consideration, and in the hope our efforts might turn handy to some other people out there.

If you notice inaccuracy, mistakes, or anything that you want to discuss, please reach out to us. (f.pardi@portavita.eu  and w.van.teijlingen@portavita.eu)

The topic is to us complex and vast, and we are more than willing to listen from you and hopefully learn from it.

# How and what


Now that you know 'why' we felt the need to experiment with both systems ourselves, I will tell you more about what we actually did to benchmark one product versus the other.

We used a test machine running CentOS 6.8 equipped with one SSD disk, 10 GB RAM, and 4 cores. It's the same virtual machine we already used in the past to run tests on several different products (like in the previous blog post when I compared InfluxDB to Postgres).

While not optimal (I would’ve prefered bare metal), this is what we had available for testing. This setup allows read-only benchmarks, since there are no separate set of disks for data and journaling.

Both Mongo and Postgres were installed on the machine, and only one product was running at the time. All tests were repeated over multiple runs, and only the average is reported.

As dataset, we used anonymized data derived from real data. As generic good practice, and as demanded by law in case of medical records, all data is anonymized, and subsequently  made accessible to employees. On top of that, for security reasons, on this paper all the mentioned fields have been altered in the name and in their content, and not relevant ones completely removed.

Both versions of Postgres and Mongo that we’ve used are vanilla versions.

This is the relevant part of postgresql.conf
```
checkpoint_completion_target = 0.9
effective_cache_size = 3GB
work_mem = 200MB
wal_buffers = 16MB
shared_buffers = 5GB
random_page_cost = 1   
```

and mongod.conf

```
storage:
  dbPath: /var/lib/mongo
  journal:
    enabled: true

processManagement:
  fork: true  # fork and run in background
  pidFilePath: /var/run/mongodb/mongod.pid  # location of pidfile

operationProfiling:
  slowOpThresholdMs: 0
```

The dataset comes from an existing project in use with Mongo and is composed of several collections. The data is structured as specified in the [FHIR][FHIR] message exchange format standards.

While this research can be considered as a standalone paper, we are planning to use it as starting point for future studies.

So far we experimented with only typical queries which we use in the real world.

# Postgres: JSON VS JSONB

Even if we use Postgres since a few years already, and I'm the DBA of the company, at the beginning of our talks I was completely green on JSON on Postgres.

The first thing I learned is that 2 representation of JSON are possible within Postgres:

* JSON:   JavaScript Object Notation
* JSONB:  The binary format of JSON

This is what official docs say:


'There are two JSON data types: json and jsonb. They accept almost identical sets of values as input. The major practical difference is one of efficiency.
The json data type stores an exact copy of the input text, which processing functions must reparse on each execution;
while jsonb data is stored in a decomposed binary format that makes it slightly slower to input due to added conversion overhead,
but significantly faster to process, since no reparsing is needed. jsonb also supports indexing, which can be a significant advantage.'


We stored every single collection into a Postgres table. Two schemas are holding the tables: fhir2 and fhir3, which correspond basically to the data exchange format they belong to.

We started experimenting on Postgres 9.6.6, and later on Postgres 10.5 and this is a granular view of the space usage using plain JSON on 9.6.6:

|    Relation      | Schema | Total size |
|---|---| --- |
| m_table         | fhir2    | 36 GB    |
| x_table       | fhir2    | 24 GB    |
| MY_table        | fhir2    | 1413 MB  |
| x_table       | fhir3    | 981 MB   |
| k_table          | fhir3    | 937 MB   |
| q_table     | fhir2    | 827 MB   |
| w_table     | fhir3    | 713 MB   |
| r_table          | fhir2    | 601 MB   |
| w_table     | fhir2    | 499 MB   |
| c_table          | fhir2    | 398 MB   |
| MY_table         | fhir3    | 38 MB    |
| z_table    | fhir3    | 648 kB   |



To convert the table to JSONB we issue the following query:
```
ALTER TABLE fhir3.x_table
  ALTER COLUMN data   
  SET DATA TYPE jsonb
  USING data::jsonb;
```

The table usage goes up to 1201 MB from 981 MB.

This is a complete overview of the space usage in JSONB on Postgres 9.6.6


|    Relation      | Schema | Total size |
|---|---| --- |
| m_table         | fhir2    | 37 GB   |
| x_table       | fhir2    | 27 GB   |
| MY_table         | fhir2    | 1592 MB |
| q_table     | fhir2    | 1013 MB |
| r_table          | fhir2    | 834 MB  |
| w_table     | fhir2    | 635 MB  |
| c_table          | fhir2    | 486 MB  |
| x_table       | fhir3    | 1201 MB |
| k_table          | fhir3    | 1190 MB |
| w_table     | fhir3    | 849 MB  |
| MY_table         | fhir3    | 45 MB   |
| z_table    | fhir3    | 744 kB  |



If we use Postgres 10.5 instead (fhir3 only):

|    Relation      | Schema | Total size |
|---|---| --- |
| x_table       | fhir3  | 1193 MB |
| k_table         | fhir3  | 1168 MB |
| w_table     | fhir3  | 835 MB |
| MY_table         | fhir3  | 45 MB |
| z_table    | fhir3  | 760 KB |

 
So it looks like Postgres 10.5 uses the space slightly more efficiently than 9.6.6

## JSON(B) Conclusions

In our opinion JSONB is the way to go because we need indexes and the additional storage requirement is acceptable in comparison with the benefits.


## Disk Space Comparison Table



| Type of data |    Disk Space (GB) |    Notes  |
| --- | --- | --- |
| Exported JSON files |  106     |                         |     
| MongoDB                |  36       |                        |
| PostgreSQL 9.6.6     |  66        |   JSON format    |
| PostgreSQL 9.6.6        |  71        |   JSONB format     |



# Benchmarking the query

This is a typical query we often run which can also be considered a starting point for other queries.

PostgreSQL Query:
```
 SELECT * FROM fhir3.MY_table, jsonb_array_elements(data -> 'Content_Array') j WHERE data ->> 'MarriageDate' = '2012-03-03' AND j ->> 'Catalog_id' = 'A.B.C.D' AND j ->> 'value' = '3wukr';
```

Same query mongofied:

```
 db.fhir3.MY_table.find({ "MarriageDate": "2012-03-03", "Content_Array" : { "$elemMatch" : { "$and" : [ { "Catalog_id" : "A.B.C.D"} , { "value" : "3wukr"}]}}})
```

Please note that we are benchmarking on a small table, 45 MB in size, holding 57546 records. Is not a big table, but certainly a starting point for further researches.

This is how data is organized:
```
data |
 {"id": "bla-MY_table-4iokxp-1gwxel-4bdkcv",  
  "MarriageDate": "1993-06-26",
 "Content_Array": [{"value": "3wukr", "Catalog_id": "A.B.C.D"},..],
 ..
 ```




## Query performances without indexes on Postgres 9.6


Since there is no PKEY, there are also no indexes predefined.

This is the query:
```
 SELECT * FROM fhir3.MY_table, jsonb_array_elements(data -> 'Content_Array') j WHERE data   @> '{ "MarriageDate":  "2012-03-03" }' AND  j @> '{"Catalog_id": "A.B.C.D", "value":  "3wukr"  }';
```

And its query plan:

```
 Nested Loop  (cost=0.01..25360.33 rows=3913 width=64)
   ->  Seq Scan on MY_table  (cost=0.00..17495.20 rows=3913 width=32)
         Filter: ((data ->> 'MarriageDate'::text) = '2012-03-03'::text)
   ->  Function Scan on jsonb_array_elements j  (cost=0.01..2.00 rows=1 width=32)
       Filter: (((value ->> 'Catalog_id'::text) = 'A.B.C.D'::text) AND ((value ->> 'value'::text) = '3wukr'::text))
```


Cold cache test:

```
count=0 ; while [ $count -lt 5 ] ; do echo 3 > /proc/sys/vm/drop_caches ; sync ; /etc/init.d/postgresql-9.6 restart ; sleep 5 ;  psql -U postgres -h /tmp  MyDatabase -f query.sql ; let count++ ; done
```

Average: 867 ms


Warm cache test. Same test as before, but only running the query over multiple runs. Data is therefore retrieved from RAM.


Average: 31 ms


If we rewrite the query as:

```
 SELECT * FROM fhir3.MY_table WHERE data   @> '{ "MarriageDate":  "2012-03-03" }' AND  data @> '{"Content_Array": [{"Catalog_id": "A.B.C.D", "value": "3wukr"}]}';
```

The query plan becomes:

```
 Seq Scan on MY_table  (cost=0.00..6618.19 rows=1 width=750) (actual time=0.049..31.932 rows=1 loops=1)
   Filter: ((data @> '{"MarriageDate": "2012-03-03"}'::jsonb) AND (data @> '{"Content_Array": [{"value": "3wukr", "Catalog_id": "A.B.C.D"}]}'::jsonb))
   Rows Removed by Filter: 57545
```

Then it can make fully use of future indexes. (since 'jsonb_array_elements cannot make use of indexes)

Cold: 902 ms

Warm: 31 ms





## Without indexes on Postgres 10.5

Same query (and same query plan):

```
 SELECT * FROM fhir3.MY_table, jsonb_array_elements(data -> 'Content_Array') j WHERE data   @> '{ "MarriageDate":  "2012-03-03" }' AND  j @> '{"Catalog_id": "A.B.C.D", "value":  "3wukr"  }';
```


Cold: 730 ms

Warm: 33 ms


With this query instead:
```
SELECT * FROM fhir3.MY_table WHERE data   @> '{ "MarriageDate":  "2012-03-03" }' AND  data @> '{"Content_Array": [{"Catalog_id": "A.B.C.D", "value": "3wukr"}]}';
```

Cold: 711 ms

Warm: 30 ms


We notice that if there is no index defined on our data, on Postgres both queries perform similar.

Since the second query can make use of indexes, we are going to prefear it.


## With indexes on Postgres 9.6.6

Defining an index on the whole data:

```
 CREATE INDEX gin_idx ON fhir3.MY_table USING GIN (data jsonb_path_ops);
```

The very same query:
```
 SELECT * FROM fhir3.MY_table, jsonb_array_elements(data -> 'Content_Array') j WHERE data   @> '{ "MarriageDate":  "2012-03-03" }' AND  j @> '{"Catalog_id": "A.B.C.D", "value":  "3wukr"  }';
```

Produces the resulting query plan:

```
 Nested Loop  (cost=4.45..136.26 rows=58 width=64) (actual time=0.055..0.098 rows=1 loops=1)
   ->  Bitmap Heap Scan on MY_table  (cost=4.45..63.17 rows=58 width=32) (actual time=0.036..0.055 rows=7 loops=1)
         Recheck Cond: (data @> '{"MarriageDate": "2012-03-03"}'::jsonb)
         Heap Blocks: exact=7
         ->  Bitmap Index Scan on gin_idx  (cost=0.00..4.43 rows=58 width=0) (actual time=0.022..0.022 rows=7 loops=1)
               Index Cond: (data @> '{"MarriageDate": "2012-03-03"}'::jsonb)
   ->  Function Scan on jsonb_array_elements j  (cost=0.01..1.25 rows=1 width=32) (actual time=0.005..0.005 rows=0 loops=7)
         Filter: (value @> '{"value": "3wukr", "Catalog_id": "A.B.C.D"}'::jsonb)
         Rows Removed by Filter: 2
```

The timings go down dramatically at:

Cold: 246.7 ms

Warm: 0.75 ms (+ - 0.20 ms)


If we slightly modify the above query to make use of the available operator:
```
 SELECT * FROM fhir3.MY_table WHERE data   @> '{ "MarriageDate":  "2012-03-03" }' AND  data @> '{"Content_Array": [{"Catalog_id": "A.B.C.D", "value": "3wukr"}]}';
```

We can then notice that the plan becomes more efficient:

```
 Bitmap Heap Scan on MY_table  (cost=10.00..11.02 rows=1 width=750) (actual time=0.201..0.201 rows=1 loops=1)
   Recheck Cond: ((data @> '{"MarriageDate": "2012-03-03"}'::jsonb) AND (data @> '{"Content_Array": [{"value": "3wukr", "Catalog_id": "A.B.C.D"}]}'::jsonb))
   Heap Blocks: exact=1
   ->  Bitmap Index Scan on gin_idx  (cost=0.00..10.00 rows=1 width=0) (actual time=0.188..0.188 rows=1 loops=1)
         Index Cond: ((data @> '{"MarriageDate": "2012-03-03"}'::jsonb) AND (data @> '{"Content_Array": [{"value": "3wukr", "Catalog_id": "A.B.C.D"}]}'::jsonb))
```

Cold: 250 ms

Warm: 0.72 ms (+ - 0.09 ms)

Our learning path is slow but constant!.. 


### Notes about possible alternative indexes

Using this index instead (which is default for GIN indexes):

```
 CREATE INDEX gin_idx ON fhir3.MY_table USING GIN (data);
```

Runtimes go up (warm runs: 2x in the first query, 1.5 in the second) but this index also supports the operators @>, ?, ?& and '?|' while jsonb_path_ops only supports @>


The size of the default GIN index is bigger than the jsonb_path_ops: 21MB vs 15MB






## With indexes on Postgres 10.5

```
 CREATE INDEX gin_idx ON fhir3.MY_table USING GIN (data jsonb_path_ops);
```



```
 SELECT * FROM fhir3.MY_table, jsonb_array_elements(data -> 'Content_Array') j WHERE data   @> '{ "MarriageDate":  "2012-03-03" }' AND  j @> '{"Catalog_id": "A.B.C.D", "value":  "3wukr"  }';
```

Cold: 227 ms

Warm: 0.76 ms (+- 0.20 ms)


Using the following query instead, we observe way better timings on Postgres 10.5 when compared to Postgres 9.6.6

```
 SELECT * FROM fhir3.MY_table WHERE data   @> '{ "MarriageDate":  "2012-03-03" }' AND  data @> '{"Content_Array": [{"Catalog_id": "A.B.C.D", "value": "3wukr"}]}';
```

Cold: 121 ms

Warm: 0.544 (+ - 0.02 ms)


Also we tried to use the default GIN index on Postgres 10, and we noticed a similar price to pay as on Postgres 9.6.6.



## Results Summary for PostgreSQL

Our conclusion is that at least in our case, Postgres 10.5 performs better than 9.6.6.
On our dataset the GIN jsonb_path_ops index provides noticeable performance boost in terms of speed.


Here is a summary:

| Query Type  |    Cold Cache (ms)  |    Warm Cache (ms) | Disk Space taken from index (MB) |
| --- | --- | --- | --- |
| PG 9.6.6 No Index                 |   867  | 41     |    -  |
| PG 9.6.6 GIN DEFAULT Index        |    227  |    2.1   |    21 |
| PG 9.6.6 GIN jsonb_path_ops Index |    250  |    0.72  |    15 |
| PG 10.5 No Index                  |    711  | 30       | -  |
| PG 10.5 GIN DEFAULT Index         |   154  |    2.1   |    21 |
| PG 10.5 GIN jsonb_path_ops Index  |    121  |    0.544 | 15 |




### Something worth noticing  (AKA: something we learned)

Similar results to what reported so far, can be obtained by simply placing an index on MarriageDate.

That index is way smaller, aboud 1.5 MB but also indeed restricted to the MarriageDate field, therefore less versatile.

```
 CREATE INDEX gin_idx3 ON fhir3.MY_table USING gin ((data -> 'MarriageDate'::text));
```

```
EXPLAIN SELECT * FROM fhir3.MY_table  WHERE data -> 'MarriageDate' ? '2012-03-03' ;
                               QUERY PLAN                               
------------------------------------------------------------------------
 Bitmap Heap Scan on MY_table  (cost=3.45..62.32 rows=58 width=32)
   Recheck Cond: ((data -> 'MarriageDate'::text) ? '2012-03-03'::text)
   ->  Bitmap Index Scan on gin_idx3  (cost=0.00..3.43 rows=58 width=0)
         Index Cond: ((data -> 'MarriageDate'::text) ? '2012-03-03'::text)
```


Note that the query has been rewritten slightly differently to make sure it uses the desired index.

Was:
```
 SELECT * FROM fhir3.MY_table WHERE data ->> 'MarriageDate' = '2012-03-03' ;
```

Now:

```
 SELECT * FROM fhir3.MY_table WHERE data -> 'MarriageDate' ? '2012-03-03' ;
```

When querying by MarriageDate, there are different ways to obtain the same result. Note that we created one index on 'MarriageDate' and one index on the whole data.

This query is making use of the index on MarriageDate, therefore should be less expensive (because the index is smaller):

```
EXPLAIN SELECT * FROM fhir3.MY_table  WHERE data   -> 'MarriageDate' ?  '2012-03-03'  ;
                               QUERY PLAN                               
------------------------------------------------------------------------
 Bitmap Heap Scan on MY_table  (cost=3.45..62.32 rows=58 width=32)
   Recheck Cond: ((data -> 'MarriageDate'::text) ? '2012-03-03'::text)
   ->  Bitmap Index Scan on gin_idx3  (cost=0.00..3.43 rows=58 width=0)
         Index Cond: ((data -> 'MarriageDate'::text) ? '2012-03-03'::text)
```

while this query is acting on the whole jsonb data:

```
EXPLAIN SELECT * FROM fhir3.MY_table  WHERE data   @> '{ "MarriageDate":  "2012-03-03" }' ;
                              QUERY PLAN                               
-----------------------------------------------------------------------
 Bitmap Heap Scan on MY_table  (cost=4.45..63.17 rows=58 width=32)
   Recheck Cond: (data @> '{"MarriageDate": "2012-03-03"}'::jsonb)
   ->  Bitmap Index Scan on gin_idx  (cost=0.00..4.43 rows=58 width=0)
         Index Cond: (data @> '{"MarriageDate": "2012-03-03"}'::jsonb)
```

The query times for our dataset are similar on both queries.



### Something else worth noticing 

We know that the '->' operator does not work for JSONB if we have a GIN index

```
CREATE INDEX ON fhir3.MY_table USING GIN ((data -> 'MarriageDate'));
```

```
EXPLAIN SELECT * FROM fhir3.MY_table WHERE data ->> 'MarriageDate' > '2012-03-03' ;
                          QUERY PLAN                           
---------------------------------------------------------------
 Seq Scan on MY_table  (cost=0.00..6618.19 rows=288 width=750)
   Filter: ((data ->> 'MarriageDate'::text) = '2012-03-03'::text)
(2 rows)
```

But if we create a B-Tree index...

```
CREATE INDEX ON fhir3.MY_table USING btree ((data ->> 'MarriageDate'));
```

```
EXPLAIN SELECT * FROM fhir3.MY_table WHERE data ->> 'MarriageDate' > '2012-03-03' ;
                                        QUERY PLAN                                        
------------------------------------------------------------------------------------------
 Index Scan using MY_table_expr_idx3 on MY_table  (cost=0.41..6166.10 rows=19182 width=750)
   Index Cond: ((data ->> 'MarriageDate'::text) > '2012-03-03'::text)
(2 rows)
```
Then the index is picked!


Therefore if we need to operate on ranges of dates, we need B-Tree indexes because GIN indexes do not work on date ranges.. And B-Tree indexes perform very slow on JSONB data.

The very same query, rewritten (uses only the MarriageDate index):

 SELECT * FROM fhir3.MY_table WHERE data ->> 'MarriageDate' > '1012-03-03'   AND  data @> '{"Content_Array": [{"Catalog_id": "A.B.C.D", "value": "3wukr"}]}';
 
takes 4 seconds on a cold run!






## Mongo 3.2

The query we ran on Postgres to extract our precious data, becomes on Mongo:

```
 db.fhir3.MY_table.find({ "MarriageDate": "2012-03-03", "Content_Array" : { "$elemMatch" : { "$and" : [ { "Catalog_id" : "A.B.C.D"} , { "value" : "3wukr"}]}}})
```



By default, mongo creates an index on _id. But in a way it is only for internal usage, since nobody is going to query on a field that is not known in advance.

This is how long it takes to query data on Mongo, when no useful index is defined:


Cold: 278 ms

Warm: 70 ms

If we create an index on MarriageDate:


```
 db.fhir3.MY_table.createIndex({"MarriageDate": 1})
```

The resulting query plan confirms that Mongo is using it:

```
Data planner:

                "winningPlan" : {
                        "stage" : "FETCH",
                        "filter" : {
                                "Content_Array" : {
                                        "$elemMatch" : {
                                                "$and" : [
                                                        {
                                                                "Catalog_id" : {
                                                                        "$eq" : "A.B.C.D"
                                                                }
                                                        },
                                                        {
                                                                "value" : {
                                                                       "$eq" : "3wukr"
                                                                }
                                                        }
                                                ]
                                        }
                                }
                        },
                        "inputStage" : {
                                "stage" : "IXSCAN",
                                "keyPattern" : {
                                        "MarriageDate" : 1
                                },
                                "indexName" : "MarriageDate_1",
                                "isMultiKey" : false,
                                "isUnique" : false,
                                "isSparse" : false,
                                "isPartial" : false,
                                "indexVersion" : 1,
                                "direction" : "forward",
                                "indexBounds" : {
                                        "MarriageDate" : [
                                                "[\"2012-03-03\", \"2012-03-03\"]"
                                        ]
                                }
                        }
                },
```

And these are the results:

Cold: 44 ms


Warm: 0ms (as reported by Mongo.. It means 0.something , but logs do not show detailed runtime. Shame.)


The index takes 1MB space, and is a B-Tree index. 


Note: we could also create a very very specific composite index on MarriageDate and Content_Array.value:

```
 db.fhir3.MY_table.createIndex({"MarriageDate": 1, "Content_Array.value": 1})
```

In that case, if both indexes live together (MarriageDate, and the composite one) then the latter is used, and runtime goes down to 30 ms.

But this is a very specific case and to follow this pattern will lead to a linear increase of the total number of indexes when compared to the required queries.

## Newer Mongo DBs

We then decided to upgrade to newer Mongo, since a comparison between Postgres 10.5 (latest) and Mongo 3.2 (released December 2015 and EOL at Sept 2018, last month from writing) sounds quite unfair.

Since an upgrade on Mongo is to be done step by step, version by version, I took my time to also time the queries on all the Mongo versions from 3.2 to latest...

I therefore report here below the timings I got running the same query under the same circumstances:


'Indexed' means the index is placed on MarriageDate

|Mongo version  | Not indexed in ms (cold, warm)| Indexed in ms (cold,warm) |
| --- | --- | --- | 
| Mongo 3.2      |  278, 70    | 44, 0.x   |
| Mongo 3.4.17 |  150, 70    | 45, 0.x   |    
| Mongo 3.6.8   |  152, 67    | 47, 0.x   |    
| Mongo 4.0.3   |  147, 71    | 40, 0.x  |


We definitely see improvements in Mongo in the major releases, mostly in retrieving data in a cold cache scenario.

Best performances are on Mongo 4.0.


## Summary of Postgres VS Mongo

| Database        |      Conditions |  Cold Cache (ms) |   Warm Cache (ms)  |    Data Disk Space    |    Disk Space taken from index (MB) |
| --- | --- | --- | --- | --- | --- |
| Postgres 10.5   |     No Index         |    711   | 30         | 71 GB  |     -                          |
| Postgres 10.5   |     Indexed    |    121   |     0.544     | 71 GB  |  15 MB (on the whole 45MB table)   |
| Mongo 4.0       |     No Index         |    147   |    71         | 36 GB  |     -                          |
| Mongo 4.0       |     Indexed    |     40     |     0.x?     | 36 GB  |     1 MB (on MarriageDate)        |



## Conclusions

Mongo can hold the same amount of data in half the space used by Postgres. This is important to keep in mind if you are holding a big amount of data. Therefore if disk space is a constraint, then Mongo has less restrictive requirements.

Also there are some aspects to take into consideration about disk and RAM. If a table takes more space, more space is needed when loading data in memory.

So the bigger the table/index, the more need for RAM.

About query speed, we can see that if data is loaded in cache/RAM then Postgres is faster than Mongo in returning an output. 

Differently, if data is not in cache, Mongo performs better. 

There is no definitive winner because both provide advantages and disadvantages depending on your use case.

Much depends on the dataset size and machine equipment, together with the data usage pattern which might require more or less indexes to be present.

Does your database entirely fit in RAM? Then Postgres is faster.

Is disk space a big constraint because you are storing a lot of data? Mongo will save you ½ of the space in comparison with Postgres.

Do you have a big database, you are not concerned about disk space and you usually only query a fraction of that data which also fits your RAM? Then go for Postgres.

Of course we are willingly not taking into consideration a very wide range of decisional aspects like the easiness of writing a query, the possibility to use Postgres extensions, the possibility to read from secondary on Postgres, or use sharding (Postgres and Mongo), table partitioning (Postgres) or maybe decouple some fields for our JSON data and insert it on a Postgres column of our JSON table. 

Today’s comparison is only based on a few simple parameters and that’s it.

It is restrictive and not complete? Probably yes. But the problem is complex and if we dive in the specific, then every case you might face would be peculiar.

About how data was indexed, something worth noticing, is that we used a generic index on 'data' for Postgres which takes a lot of space but also is a kind of silver bullet because it covers a lot of use cases (the whole data is in there).

The downside is its size, in our case 1/3 of the table size. (15 MB index on a 45 MB table).

On Mongo we took into consideration only an index on ‘MarriageDate’, which has rather a restrictive usage.






# Bonus Track: jsquery

Our colleague Yeb Havinga from [MGRID][MGRID] pointed us to [jsquery][jsquery], which is a very nice Postgres extension.

A little gem.

There are several advantages in using jsquery: an effective way to search in nested objects and arrays, flexibility in the queries and 'detoast fields one time only' are some of them.

If you are interested in some details, you might find this presentation interesting: http://www.sai.msu.su/%7Emegera/postgres/talks/pgconfeu-2014-jsquery.pdf

The tests here below reported ran over Postgres 10 only. As reference we take the same query we used earlier with the GIN index:

```
 CREATE INDEX gin_idx ON fhir3.MY_table USING GIN (data jsonb_path_ops);
```

```
EXPLAIN SELECT * FROM fhir3.MY_table WHERE data   @> '{ "MarriageDate":  "2012-03-03" }' AND  data @> '{"Content_Array": [{"Catalog_id": "A.B.C.D", "value": "3wukr"}]}';

 Bitmap Heap Scan on MY_table  (cost=10.00..11.02 rows=1 width=32) (actual time=0.341..0.343 rows=1 loops=1)
   Recheck Cond: ((data @> '{"MarriageDate": "2012-03-03"}'::jsonb) AND (data @> '{"Content_Array": [{"value": "3wukr", "Catalog_id": "A.B.C.D"}]}'::jsonb))
   Heap Blocks: exact=1
   ->  Bitmap Index Scan on gin_idx  (cost=0.00..10.00 rows=1 width=0) (actual time=0.318..0.319 rows=1 loops=1)
         Index Cond: ((data @> '{"MarriageDate": "2012-03-03"}'::jsonb) AND (data @> '{"Content_Array": [{"value": "3wukr", "Catalog_id": "A.B.C.D"}]}'::jsonb))
```

JSQuery supports 2 different GIN indexes: jsonb_path_value_ops and jsonb_value_path_ops. Let's see them in action:

```
CREATE INDEX gin_idx_jsquery_value ON fhir3.MY_table USING GIN (data jsonb_path_value_ops);
```

```
EXPLAIN SELECT * FROM fhir3.MY_table WHERE data @@ 'MarriageDate = "2012-03-03" AND Content_Array.#(Catalog_id = "A.B.C.D" AND value="3wukr")';
                                                                           QUERY PLAN                                                                           
----------------------------------------------------------------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on MY_table  (cost=10.45..69.17 rows=58 width=32) (actual time=0.359..0.361 rows=1 loops=1)
   Recheck Cond: (data @@ '("MarriageDate" = "2012-03-03" AND "Content_Array".#("Catalog_id" = "A.B.C.D" AND "value" = "3wukr"))'::jsquery)
   Heap Blocks: exact=1
   ->  Bitmap Index Scan on gin_idx_jsquery_value  (cost=0.00..10.43 rows=58 width=0) (actual time=0.261..0.261 rows=1 loops=1)
         Index Cond: (data @@ '("MarriageDate" = "2012-03-03" AND "Content_Array".#("Catalog_id" = "A.B.C.D" AND "value" = "3wukr"))'::jsquery)
```

Cold Run Average: 193 ms

Warm Run Average: 0.769 ms

```
CREATE INDEX gin_idx_jsquery_path ON fhir3.MY_table USING GIN (data jsonb_value_path_ops);
```

```
 Bitmap Heap Scan on MY_table  (cost=10.45..69.17 rows=58 width=32) (actual time=0.341..0.342 rows=1 loops=1)
   Recheck Cond: (data @@ '("MarriageDate" = "2012-03-03" AND "Content_Array".#("Catalog_id" = "A.B.C.D" AND "value" = "3wukr"))'::jsquery)
   Heap Blocks: exact=1
   ->  Bitmap Index Scan on gin_idx_jsquery_path  (cost=0.00..10.43 rows=58 width=0) (actual time=0.305..0.306 rows=1 loops=1)
         Index Cond: (data @@ '("MarriageDate" = "2012-03-03" AND "Content_Array".#("Catalog_id" = "A.B.C.D" AND "value" = "3wukr"))'::jsquery)
```

If both indexes are present (path_value and value_path), under our circumstances, value_path is preferred. Note that the planning time for the first time the query runs, is rather costly.

Cold: 171 ms

Warm: 0.77 ms


While jsquery does not speed up queries in our scenario, we think it might be very useful and maybe even faster for different and more complicated data usage patterns.






[portavita]: https://www.portavita.com/
[FHIR]: https://en.wikipedia.org/wiki/Fast_Healthcare_Interoperability_Resources
[mgrid]: https://www.mgrid.net/
[jsquery]: https://github.com/postgrespro/jsquery/blob/master/README.md

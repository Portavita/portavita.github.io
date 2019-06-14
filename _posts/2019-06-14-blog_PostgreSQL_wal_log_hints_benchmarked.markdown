---
layout: post
title:  "PostgreSQL: wal_log_hints benchmarked"
date:   2019-06-14 11:31:56 +0200
categories: PostgreSQL Performances Benchmarking
author: Fabio Pardi
---



Last update: 2019-06-14 12:05:56 +0200 - add definition of wal_log_hints


# Intro


I find 'pg_rewind' to be a cool feature that appeared in Postgres 9.5 in the master-replica landscape. It is a good ally in managing master-replica databases.

The manual defines it this way: ```synchronize a PostgreSQL data directory with another data directory that was forked from the first one```.

I am of the opinion that that feature is very handy in some cases, but I will talk about it another day because today I would like to talk about something related to it: wal_log_hints. 

Wal_log_hints is a Postgres configuration parameter that needs to be enabled in order to use pg_rewind. Also it is needed when one wants to use data checksum.

It is defined as: 

```
wal_log_hints (boolean)

    When this parameter is on, the PostgreSQL server writes the entire content of each disk page to WAL during the first modification of that page after a checkpoint, even for non-critical modifications of so-called hint bits.

    If data checksums are enabled, hint bit updates are always WAL-logged and this setting is ignored. You can use this setting to test how much extra WAL-logging would occur if your database had data checksums enabled.

    This parameter can only be set at server start. The default value is off.
```

Basically, to be able to use data checksum or pg_rewind features, more information are needed to be recorded in the WAL files, and that usually comes at a price in terms of performances and resources.

As usual here at [Portavita][portavita] we like to make informed decisions, so I took my time to get some numbers. 

It might not represent a production load, but it should give us some directions.


# The tests

I used our beloved pgbench tool to run the tests. I ran it first to initialize the data, then I ran it again to query. Nothing special here.

I counted the number of produced WAL files after each ran.

As side note, I should mention that checkpoints are called a couple of times over each run and I did not notice much difference in CPU usage over the tests.

## The Setup

Relevant part of the config file:

```
checkpoint_completion_target = 0.9
max_wal_size =  1536MB
```

## wal_log_hints = off 

The first test runs as baseline.

```pgbench -i -s 100```

It produces 76 WAL files

```pgbench -c 4 -s 100 -j 1 -M prepared  -b tpcb-like  -T 60```

Brings the total produced files to 162 



## wal_log_hints = on 

 ```pgbench -i -s 100```

It produces 176 WAL files

```pgbench -c 4 -s 100 -j 1 -M prepared  -b tpcb-like  -T 60```
 
Brings the total produced files to 272 


![Comparing wal_log_hints](https://raw.githubusercontent.com/Portavita/portavita.github.io/master/img/graph_wal_hints_1.png)


It means that wal_log_hints costs to us 67% more space, disk workload, and eventually network when WAL files are archived remotely.
 

That's a lot. It might be for a good cause, but still is a lot.




## There's more..

But... Wait.

In the [last article][last article] we talked about wal_compression, and how cool is that. And how much we can save using it.

So I tried to repeat the tests while wal_compression was enabled.


wal_log_hints = off 

produced 82 WAL files

wal_log_hints = on

produced 85 of them



![With_wal_compression_enabled](https://raw.githubusercontent.com/Portavita/portavita.github.io/master/img/wal_compression_on.png)


# Conclusions

If you are already using wal_compression then to enable wal_log_hints comes for cheap, and that paves the way to pg_rewind and data checksum, both worth exploring.

If you instead are using default settings for wal_compression (off), then to enable wal_log_hints will come at a price and I suggest to you to keep a close eye to performances and disk space.




Ideas or comments? You can find me on [Linkedin](https://www.linkedin.com/in/fabiopardi/)



[last article]: https://portavita.github.io/2019-05-13-blog_about_wal_compression/

[portavita]: https://www.portavita.com/


---
layout: post
title:  "PostgreSQL: Pglogical VS Streaming Replication: What about Bandwidth usage?"
date:   2018-06-23 12:34:56 +0200
categories: Postgres
author: Fabio Pardi
---

Some years ago, all the teams here at  [Portavita][portavita] have been considerably busy for months in planning 'The Migration'.


Our efforts in 'The Migration' can be fairly compared to those of the beautiful [Artic Tern][arctic_tern], also because we (both Portavitians and those birds) migrated over summer time.

But, differently from the Terns, we did a migration 2.0, which means we used brain and keyboard and the result was one funeral and one newborn.

We buried Oracle (very, very, very deep in the digital space) and we celebrated our brand new, shining, super performing and efficiently replicating PostgreSQL server.

Today I would like to talk a bit about replication.

### A few words about replication

PostgreSQL Streaming Replication is made possible by streaming to the standby servers all the changes that happen on the master at disk's blocks level. Therefore you can have one master streaming changes to several standby servers, from which you can read. Most use cases for this setup include the possibility to offload the master database for reads (a lot to discuss here, yes) and to have a High Availability setup for disaster recovery purposes.

This very powerful feature is available on PostgreSQL since version 9.0 and zillions of words have been already spent in the cyberspace on this topic.
More recently, a new replication mechanism has been made possible thanks to the efforts of [2ndQuadrant][2ndQuadrant]. 

I learned the usage of it some 1 year and half ago in London during a 'Replication Backup and Disaster Recovery' course, but only recently I had the possibility to play with it.

What pglogical does differently from Streaming Replication is actually the way it replicates. Master (in pglogical jargon called 'Provider') sends a logical stream of data to Standby ('Subscriber').

Now, let's focus on the word 'logical'. English dictionary defines logical as 'reasonable and based on good judgement', which in a way might fit our pglogical.

But what better fits our 'logical' definition is instead the one in computer science's fields: 'Refers to a user's view of the way data or systems are organized. The opposite of logical is physical'

Basically what pglogical is able to, is to stream only the requested changes, in contrast to streaming replication, where the whole data is streamed to the standby node(s).

This opens up the way to new scenarios where only one database or part of it (a schema, a table or even a column) can be replicated to another node. 

Advantages of using pglogical are multiple, but I do not want to focus on the comparison, since you can read it already elsewhere on internet. If you are interested in knowing more, a good starting point is indeed its official [documentation][pglogical_docs]

### Benchmarking

While trying it out, I thought was interesting to understand how much bandwidth it requires in comparison with what we have now.

I created a basic setup with one Master (hostname: myprovider) and one Standby (hostname: mysubscriber) running on 2 separate VMs equipped with 4 cores, 10GB RAM, on SSD over a CensOS 6.8 running PostgreSQL 9.6.3 and pglogical 2.2 

I then benchmarked performances of streaming replication against pglogical ones. 

### The Setup

#### Streaming Replication

In Streaming Replication nodes are configured as follow:

###### Master

```
wal_level = hot_standby
max_wal_senders = 3
wal_keep_segments = 128
```

###### Standby 

```
wal_keep_segments = 128
hot_standby = on
# host_standby_feedback is off
```

recovery.conf

```
standby_mode = 'on'  #only if is really a standby. If you are recovering a master then comment it out
primary_conninfo = 'host=SOME_IP user=repl password=your_passwd application_name=yourreplication'

```

#### Pglogical

Provider (what is called 'master' on streaming replication ) and Subscriber (the 'standby') are created this way:

###### Provider

```
pglog_db=pgbench
psql $pglog_db  -c "SELECT pglogical.create_node(node_name := 'provider.$pglog_db', dsn := 'host=myprovider port=5432 dbname=$pglog_db');"
psql $pglog_db  -c "SELECT pglogical.replication_set_add_all_tables('default', ARRAY['schema_to_replicate']);"
```

###### Subscriber

```
pglog_db=pgbench
psql $pglog_db -c "SELECT pglogical.create_node(node_name := 'subscriber.$pglog_db', dsn := 'host=mysubscriber port=5432 dbname=$pglog_db');"
psql $pglog_db  -c "SELECT pglogical.create_subscription(subscription_name := 'subscription_to_myprovider_$pglog_db',  synchronize_structure := true, provider_dsn := 'host=myprovider port=5432 dbname=$pglog_db');"
```

##### The Tests

To push data to the database, I used pgbench tool and some homebrew command line scripts, to fit different purposes.

All the tests ran multiple times in order to verify the accuracy of the numbers. Given the simplicity of the test, I received identical outputs for every run, which is always a good thing.

Before each run network interface statistics are cleared, and after the run results are noted down.

##### Run test, run!

Here I'm reporting the set of 3 tests I ran, and their results related to the network interface that connects them to the standby/subscriber

###### pgbench test


I ran pgbench in the following way:

pgbench -c 3 -P 10  -r -T 200 -R 1000 

Which tells pgbench to push 1000 Tuples per Second, therefore is possible to derive the bandwidth usage based on a stable baseline of pushed data.


Streaming Replication:

         RX bytes:41623441 (39.6 MiB)  TX bytes:122636756 (116.9 MiB)
         
         
Logical Replication:

         RX bytes:37705717 (35.9 MiB)  TX bytes:195120378 (186.0 MiB)
         

First conclusion is that logical replication uses more bandwidth to send data to standby. 

Note that 'By default, pgbench tests a scenario that is loosely based on TPC-B, involving five SELECT, UPDATE, and INSERT commands per transaction'.

I think that it might be a kind of real world scenario.

To get more in depth and have more grip over the actions performed I therefore created another couple of tests.



###### INSERT in bulk BEGIN..END  


I pushed 50 Million+1 records:

```
psql -U pgsql pgbench -c 'insert into  my_replicated_schema.my_first_replicated_table values (generate_series(0,50000000));' 

```
The above command will do one big commit, resulting in:

Streaming Replication:

         RX bytes:23058328 (21.9 MiB)  TX bytes:7502525286 (6.9 GiB)

Logical Replication:

         RX bytes:10271325 (9.7 MiB)  TX bytes:2465344586 (2.2 GiB)



Interesting... under this scenario we are actually saving a lot when using pglogical...




###### INSERT and commit records one by one 

count=0 ; while [ $count -lt 100000 ] ; do   psql -c  "insert into my_replicated_schema.my_first_replicated_table VALUES ( $(($count)) )"; ((count++))  ; done



Streaming Replication:

         RX bytes:21279724 (20.2 MiB)  TX bytes:33334914 (31.7 MiB)

Logical Replication:

          RX bytes:18022889 (17.1 MiB)  TX bytes:44468348 (42.4 MiB)


Note that I pushed 10K records and not 50M because inserting one by one costs a lot of time. A lot. 

Here below I will report a comparison, with normalized numbers in order to allow an easy and fair comparison.

##### Summary



Test Type | Pglogical | Streaming Replication
------------ | ------------- | -------------
pgbench -c 3 -P 10 -r -T 200 -R 1000 (MB) | 35.9/186 | 39.6/116.9
INSERT in bulk 50M records (MB) | 9.7/2252 |	21.9/7065
INSERT one by one 50M records (MB) | 855/2120 | 1010/1585 



#### Conclusions and thoughts

I ran the tests with autovacuum=off to have 'clean' results. Since a vacuum causes blocks to be written, wherever your write pattern has updates or deletes, a vacuum will produce written blocks, which in turn will produce network traffic to your standby when using streaming replication. 

The same is not true when using logical replication because every node is in charge for its own content. That means that block changes caused by a vacuum on the Provider node will not generate traffic toward the Subscribers. 

The downside is that the Subscriber will have resources busy in performing its own VACUUM operations. Something definitely to keep in mind.

I think that pglogical is a great invention, a bit more complicate to setup and maintain but depends on your scenario it can solve problems that are unsolvable using streaming replication.
Also the project can probably mature in the future and bring us more features and functions for easier maintenance.

Streaming replication on the other end is dead easy to configure, mature and very robust as solution.

As english say: pick your poison!

 
[gatling]: http://www.gatling.io
[portavita]: https://www.portavita.com/
[arctic_tern]: https://en.wikipedia.org/wiki/Arctic_tern
[2ndQuadrant]: https://www.2ndquadrant.com
[pglogical_docs]: https://www.2ndquadrant.com/en/resources/pglogical/

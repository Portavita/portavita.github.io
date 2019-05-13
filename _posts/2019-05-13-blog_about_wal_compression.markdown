---
layout: post
title:  "About wal_compression"
date:   2019-05-13 17:14:56 +0200
categories: PostgreSQL Performances Benchmarking
author: Fabio Pardi
---


# Spring cleaning time

Sprint over sprint I'm always busy doing a lot of things, like probably most of you out there. 

Professionally, but in life too, I consider as good practice to stop and look behind every now and then.

Where am I? What brought me here? Am I in the right direction? 

As part of this periodic assessment, I also review from scratch the configuration of my beloved databases. 

Configuration file is therefore parsed line by line to see if the settings are matching the current workload and needs. 
I also try out parameters I did not consider in the past or I thought were 'not for us' or 'not for now'. 

It might sound boring to review everything bit by bit, but I can tell you is rewarding and one can learn a lot from that.

Today I want to talk to you about how I saved 50% space and bandwidth, and increased performances turning on one single parameter.


# The magic

It is sometimes mentioned, mostly underrated and frequently ignored. It is a feature introduced in PostgreSQL 9.5 and is called 'wal_compression'. 

PostgreSQL official docs refer to it as:

'''
wal_compression (boolean)

    When this parameter is on, the PostgreSQL server compresses a full page image written to WAL when full_page_writes is on or during a base backup. A compressed page image will be decompressed during WAL replay. The default value is off. Only superusers can change this setting.

    Turning this parameter on can reduce the WAL volume without increasing the risk of unrecoverable data corruption, but at the cost of some extra CPU spent on the compression during WAL logging and on the decompression during WAL replay.
'''


I was intrigued and I decided to give it a try.

Fortunately, I have my friend Mr Jenkins that runs some pgbench test every night, so I have some baseline ready, to compare with. 

After running it, I was immediately flabbergasted by the gain...

# Benchmarking results

## Intro

Here at [Portavita][portavita], we handle medical data and can happen to have a disk or a partition encrypted. 

Some database host in particular, running on a VM, does not have 'aes' flag on the virtual CPU (read: encryption is slow and painful) and, also runs on HDD. Add to it that WAL is not separate from Data (It makes me cry, but..'Take it or leave it' )

## The Results

### VMs on test env

Consider this machine as the worst case scenario, the DBA nightmare setup.

This is what happens when you enabled wal_compression. Pressure on disk is relieved. We have a net increase of performances of 100%. Also worth noticing is that CPU is actually less busy, because what is really keeping the CPU busy 
on this setup is the encryption.


![VM on HDD - encryption - no AES - no separate disks](https://raw.githubusercontent.com/Portavita/portavita.github.io/master/img/pgbench_dev.jpeg)


### Production hosts on SSD

Here instead you can see the benefits on a master-slave setup, running an out of the book recipe on SSDs.

Since I have them available, we can see the real-life data. As you can see, since wal_compression was enabled, less WAL disk activity and consequent network activity (WAL files are archived) at the cost of some CPU cycle.

![Bandwidth usage](https://raw.githubusercontent.com/Portavita/portavita.github.io/master/img/bandwidth_slave_prod.jpeg)
wal_compression has been turned on on the 9th, around 11 AM. Consider that over that link, both streaming and WAL copy happens.

![CPU impact](https://raw.githubusercontent.com/Portavita/portavita.github.io/master/img/cpu_prod_master.jpeg)



### WAL files production

On all the production hosts, WAL files production went down up to 50%.

# Conclusion

With one single setting, we can save on disk space, bandwidth and disk IO. We can therefore make better use of our prcious resources

Something worth considering on every database!


[portavita]: https://www.portavita.com/

---
title: '[PostgreSQL] Bucardo Replication Update is Slow?'
author: saifulmuhajir
type: post
date: 2017-05-15T03:55:20+00:00
url: /postgresql-bucardo-replication-update-is-slow/
categories:
  - PostgreSQL
tags:
  - bucardo
  - database
  - PostgreSQL
  - replication

---
[Bucardo] (https://wiki.postgresql.org/wiki/Bucardo) is one of the replication system for PostgreSQL where it works asynchronous and using triggers to replicate the data. And one of the cool feature is you can have multi masters as well as slaves. Since the replication is working on table level to replicate the data, we can use it to replicate partial data and of course DLL is not supported. There are many more features you can look [here] (https://bucardo.org/wiki/Bucardo).

Recently, I have to setup replication from Amazon RDS on one of our busiest databases to another cloud services provider for some reasons. And [the only way to do this] (https://aws.amazon.com/blogs/aws/rds-postgres-read-replicas/) is using Bucardo.

At first, the replication went flawless. Setting up replication is done with triggers created on the required tables and the data started replicating as expected. The **initial data was only 100GB** and counting fast. There is not much INSERT activity on this database, but there are lots of UPDATEs depending on the hours. Bucardo log shows this on first days:

```
(12794) [Thu May  4 00:41:14 2017] KID (db_sync2) Totals: deletes=2216 inserts=3882 conflicts=0
(12794) [Thu May  4 00:41:22 2017] KID (db_sync2) Totals: deletes=1186 inserts=3663 conflicts=0
(12794) [Thu May  4 00:41:29 2017] KID (db_sync2) Totals: deletes=2511 inserts=3565 conflicts=0
(12794) [Thu May  4 00:41:37 2017] KID (db_sync2) Totals: deletes=3425 inserts=5385 conflicts=0
(12794) [Thu May  4 00:41:38 2017] KID (db_sync2) Totals: deletes=2938 inserts=3675 conflicts=0
```

On the first week the replication went fast enough with **average delay only 30-60 seconds**. The Bucardo slave is actually being used by our microservices application but having delay up to 1 minute is not a problem. A week later tho, the **delay started to increase up to 10 minutes**. No one is complaining or anything, but I am anxious that this will get worse every day and am afraid that the delay will reach 30 minutes and by then people will start screaming.

```
 (22388) [Fri May 12 13:58:58 2017] KID (db_sync2) Totals: deletes=9330 inserts=16061 conflicts=0
 (22388) [Fri May 12 14:05:50 2017] KID (db_sync2) Totals: deletes=9742 inserts=17407 conflicts=0
 (22388) [Fri May 12 14:12:39 2017] KID (db_sync2) Totals: deletes=8631 inserts=16660 conflicts=0
 (22388) [Fri May 12 14:18:45 2017] KID (db_sync2) Totals: deletes=7845 inserts=13834 conflicts=0
 (22388) [Fri May 12 14:25:28 2017] KID (db_sync2) Totals: deletes=8009 inserts=14973 conflicts=0
 ```

I have to admit that I cannot sleep in peace because of this. Every time and everywhere my mind brings me to this problem with no solution anywhere to be found.

So, on Sunday night after putting my son to bed, I opened up my laptop and started to logon to the server and watching the database activity on the master. Since Bucardo 5, **[manual purge on delta tables is not required] (https://bucardo.org/wiki/Bucardo/Cron)** because the process is run automatically with interval 45 seconds. But, I see that the process completed after 3 minutes or so. And this is where I see something must be wrong because if the data is frequently purged the data must be small. If purging small tables is completed >100ms, well, something is wrong.

```
 2017-05-14 03:04:22 LOG:  duration: 113741.467 ms  execute &lt;unnamed&gt;: SELECT bucardo.bucardo_purge_delta('45 seconds')
 2017-05-14 03:12:05 LOG:  duration: 105585.950 ms  execute &lt;unnamed&gt;: SELECT bucardo.bucardo_purge_delta('45 seconds')
 2017-05-14 03:27:05 LOG:  duration: 138461.169 ms  execute &lt;unnamed&gt;: SELECT bucardo.bucardo_purge_delta('45 seconds')
 ```

So, I take a look at the delta tables and see that the table size is growing and one of them reached 1,8GB. Even the track tables are growing bigger.

```
 Schema  │           Name                    │    Size    │
───────────────────────────────────────────────────
 bucardo │ delta_public_booking              │ 699 MB     │ 
 bucardo │ delta_public_booking_detail       │ 175 MB     │ 
 bucardo │ delta_public_booking_histories    │ 1312 kB    │ 
 bucardo │ delta_public_rates                │ 555 MB     │ 
 bucardo │ delta_public_tracking             │ 1875 MB    │ 
 bucardo │ delta_public_tracking_histories   │ 803 MB     │ 
 bucardo │ delta_public_tracking_info_agency │ 87 MB      │ 
 bucardo │ delta_public_tracking_info_tkpd   │ 85 MB      │ 
 bucardo │ track_public_booking              │ 426 MB     │ 
 bucardo │ track_public_booking_detail       │ 78 MB      │ 
 bucardo │ track_public_booking_histories    │ 1480 kB    │ 
 bucardo │ track_public_rates                │ 293 MB     │ 
 bucardo │ track_public_tracking             │ 715 MB     │ 
 bucardo │ track_public_tracking_histories   │ 302 MB     │ 
 bucardo │ track_public_tracking_info_agency │ 76 MB      │ 
 bucardo │ track_public_tracking_info_tkpd   │ 78 MB      │
```

When you purge your tables, frequently, you need to [VACUUM and ANALYZE] (https://www.postgresql.org/docs/current/static/routine-vacuuming.html) your tables frequently as well, if it wasn't being done by AUTOVACUUM. Purging tables means that your data is changing but unless it is being vacuumed and analyzed, Postgres won't know about it.

So, let's do it!

```
 VACUUM ANALYZE bucardo.delta_public_booking;
 VACUUM ANALYZE bucardo.delta_public_booking_detail;
 VACUUM ANALYZE bucardo.track_public_booking;
 VACUUM ANALYZE bucardo.track_public_booking_detail;
```

And so on.

And once the VACUUM ANALYZE processes are done, the updates on Bucardo replication is faster than ever. It is now run every 3 seconds. Yay!

```
 (13073) [Mon May 15 09:44:55 2017] KID (db_sync2) Totals: deletes=905 inserts=1442 conflicts=0
 (13073) [Mon May 15 09:44:58 2017] KID (db_sync2) Totals: deletes=745 inserts=1522 conflicts=0
 (13073) [Mon May 15 09:45:02 2017] KID (db_sync2) Totals: deletes=726 inserts=1122 conflicts=0
```

And the last step is of course changing the AUTOVACUUM threshold and ratio for these tables so that autovacuum will perform what needs to be done.

```
 ALTER TABLE booking SET (autovacuum_vacuum_scale_factor = 0.0);
 ALTER TABLE booking SET (autovacuum_vacuum_threshold = 5000);
 ALTER TABLE booking SET (autovacuum_analyze_scale_factor = 0.0);
 ALTER TABLE booking SET (autovacuum_analyze_threshold = 5000);
```

And now, I can go to sleep.

**UPDATE:** The drawback of my faster-than-ever-replication is CPU usage on the master steadily on 50%. Oh, well.
---
title: '[PostgreSQL] Amazon RDS as A Master: How to Replicate from AWS'
author: saifulmuhajir
type: post
date: 2017-05-30T03:36:33+00:00
url: /postgresql-amazon-rds-as-a-master-how-to-replicate-from-aws/
categories:
  - PostgreSQL
  - Technology
tags:
  - bucardo
  - PostgreSQL
  - replication

---
There are lots of ways to **replicate PostgreSQL database** including _streaming replication, logical replication, trigger-based replication_ and so on. And sometimes you need all of them depending on you and your business needs. I can say that replication options in Postgres is somewhat limitless.

And this time, I am writing how to replicate and use **Amazon RDS as the master** and use another as the slaves.

Why would you do that? Why would anyone bring down their cloud servers? This time, my reason is our data analysts needs to _as the name suggests_ analyze transactions data in their own machine, which is located in our own data center while the database needed is on the cloud. We don't allow them to access production servers because, you know, production is scary stuff. That's what we told them. :grimacing:

Here's the preparation stuff:

  * PostgreSQL servers >= 9.1 in Amazon RDS, this is because we need [session replication role] (https://aws.amazon.com/blogs/aws/rds-postgres-read-replicas/)
  * Server for Bucardo and Postgres destination
  * Bucardo, obviously

That would be all.

[Bucardo] (https://bucardo.org/wiki/Bucardo/Installation) can be installed anywhere and this time I put it in the same server where Postgres destination is located. First, we need to grab Bucardo dependencies consist of some Perl modules including **DBIx::Safe, boolean, DBI, DBD::Pg, Test::Simple.**

```
sudo apt-get install libdbd-pg-perl libdbix-safe-perl postgresql-plperl-9.6 postgresql-contrib-9.6
```

Once the're done, we can continue to grab the source files and install to the system.

```
$ wget http://bucardo.org/downloads/Bucardo-5.4.1.tar.gz
$ tar xvzf Bucardo-5.4.1.tar.gz
$ cd Bucardo-5.4.1
$ perl Makefile.PL
Checking if your kit is complete...
Looks good
Warning: prerequisite DBD::Pg 2.0 not found.
Warning: prerequisite boolean 0 not found.
Writing Makefile for Bucardo
Writing MYMETA.yml and MYMETA.json
```

And now let's perform make:

```
$ make
cp bucardo.schema blib/share/bucardo.schema
cp Bucardo.pm blib/lib/Bucardo.pm
cp bucardo blib/script/bucardo
/usr/bin/perl -MExtUtils::MY -e 'MY->fixin(shift)' -- blib/script/bucardo
Manifying blib/man1/bucardo.1pm
Manifying blib/man3/Bucardo.3pm
```

And... install!

```
$ sudo make install
Installing /usr/local/share/perl/5.18.2/Bucardo.pm
Installing /usr/local/man/man1/bucardo.1pm
Installing /usr/local/man/man3/Bucardo.3pm
Installing /usr/local/bin/bucardo
Installing /usr/local/share/bucardo/bucardo.schema
Appending installation info to /usr/local/lib/perl/5.18.2/perllocal.pod
```

Now we need to make sure that the installation process is successful:

```
$ bucardo --version
bucardo version 5.4.1
```

Viola! But, we need to make sure that Bucardo can run so we will need pid and log directory for it.

```
$ sudo mkdir -p /var/run/bucardo
$ sudo mkdir -p /var/log/bucardo
$ sudo chown postgres:postgres /var/run/bucardo
$ sudo chown postgres:postgres /var/log/bucardo
```

Now that all looks good, we can continue to install the Bucardo database.

Before we proceed, let's create a file so that Bucardo can use it as configuration with following details:

```
dbhost=localhost
dbname=bucardo
dbport=5432
dbuser=superman
```

And put them in .bucardorc file inside user's home directory. This file will tell Bucardo that we'll be using bucardo database in local server with user called 'superman'. Before continuing the process, superman user is needed in the source database (RDS) and destination. So, you have to create them.

```
$ bucardo install
This will install the bucardo database into an existing Postgres cluster.
Postgres must have been compiled with Perl support,
and you must connect as a superuser

Current connection settings:
1. Host:           localhost
2. Port:           5432
3. User:           superman
4. Database:       bucardo
5. PID directory:  /var/run/bucardo
Enter a number to change it, P to proceed, or Q to quit: P

Postgres version is: 9.6
Attempting to create and populate the bucardo database and schema
Database creation is complete

Updated configuration setting "piddir"
Installation is now complete.
If you see errors or need help, please email bucardo-general@bucardo.org

You may want to check over the configuration variables next, by running:
bucardo show all
Change any setting by using: bucardo set foo=bar
```

The database part in connection settings for your environment may be different but that's okay as long as the user can use it. But this time I will change a parameter in Bucardo because the tables I'll be replicating is big enough and quite busy with lots of updates and inserts.

```
$ bucardo show all | grep delta
kid_nodeltarows_sleep     = 0.5
quick_delta_check         = 1

$ bucardo set quick_delta_check=0
Set "quick_delta_check" to "0"
```

This is me changing quick\_delta\_check parameter. This parameter is used for checking delta activity in the tables and skipping it will save me time and keep the replication delay as small as possible.

Everything is now in place and we can continue to set the configuration for databases, tables, syncs, and so on.

```
$ bucardo add db rds_source1 db=source1 host=source1.rds.amazonaws.com port=5432 user=superman
Added database "rds_source1"
$ bucardo add db rds_source2 db=source2 host=source2.rds.amazonaws.com port=5432 user=superman
Added database "rds_source2"
$ bucardo add db target_db db=target1 host=localhost port=5432 user=superman
Added database "target_db"
```

As you can see, I added two sources and one destination for this case because tables needed are located in those two sources and we can verify as follow:

```
$ bucardo list dbs
Database: target_db      Status: active  Conn: psql -U superman -d target1 -h localhost
Database: rds_source1    Status: active  Conn: psql -p 5432 -U superman -d source1 -h source1.rds.amazonaws.com
Database: rds_source2    Status: active  Conn: psql -p 5432 -U superman -d source2 -h source2.rds.amazonaws.com
```

Before we can continue the next process, we need to create sequences and tables structures on target database. So, just dump the schema and execute the result in the target. And we can move on to add them to Bucardo.

```
$ bucardo add sequences public.order_id_seq db=source2 herd=source_group1 --verbose
Added the following tables or sequences:
public.order_id_seq
The following tables or sequences are now part of the relgroup "source_group1":
public.order_id_seq
$ bucardo add sequences public.payment_id_seq db=source2 herd=source_group1 --verbose
Added the following tables or sequences:
public.payment_id_seq
The following tables or sequences are now part of the relgroup "source_group1":
public.payment_id_seq
$ bucardo add sequences public.promos_id_seq db=source1 herd=source_group2 --verbose
Added the following tables or sequences:
public.promos_id_seq
The following tables or sequences are now part of the relgroup "source_group2":
public.promos_id_seq
$ bucardo add tables public.promos db=source1 herd=source_group2 --verbose
Added the following tables or sequences:
public.promos
The following tables or sequences are now part of the relgroup "source_group2":
public.promos
$ bucardo add tables promo_code db=source1 herd=source_group2 --verbose
Added the following tables or sequences:
public.promo_code
The following tables or sequences are now part of the relgroup "source_group2":
public.promo_code
$ bucardo add tables public.order db=source2 herd=source_group1 --verbose
Added the following tables or sequences:
public.order
The following tables or sequences are now part of the relgroup "source_group1":
public.order
$ bucardo add tables public.payment db=source2 herd=source_group1 --verbose
Added the following tables or sequences:
public.payment
The following tables or sequences are now part of the relgroup "source_group1":
public.payment

$ bucardo list tables
1. Table: public.promo_code       DB: source1  PK: id (bigint)
2. Table: public.promos           DB: source1  PK: id (integer)
3. Table: public.order            DB: source2  PK: id (bigint)
4. Table: public.payment          DB: source2  PK: id (bigint)

$ bucardo list sequences
Sequence: public.order_id_seq           DB: source2
Sequence: public.payment_id_seq         DB: source2
Sequence: public.promos_id_seq          DB: source1
```

Since we added _herd_ in our add tables command, the tables and sequences added are now attached to relgroups a.k.a. herds.

```
$ bucardo list relgroups
Relgroup: source_group2  DB: source1  Members: public.promos, public.promo_code_id_seq, public.promo_code
Relgroup: source_group1  DB: source2  Members: public.order, public.order_id_seq, public.payment, public.payment_id_seq
```

This group will be used in sync configuration to tell which relgroups belongs to a sync.

Everything is in place and the last step needed is to create syncs. But, this last step will require [share row exclusive locks on the tables] (https://www.postgresql.org/docs/9.6/static/explicit-locking.html) and will conflict with almost every locks in Postgres except access share and row share. So, to make everything runs smoothly, you may need to stop your production applications to access the database.

```
$ bucardo add sync sync1 relgroup=source_group1 dbs=rds_source1:source,target_db:target onetimecopy=2 --verbose
Added sync "sync1"
Created a new dbgroup named "sync1"
$ bucardo add sync sync2 relgroup=source_group2 dbs=rds_source2:source,target_db:target onetimecopy=2 --verbose
Added sync "sync2"
Created a new dbgroup named "sync2"

$ bucardo list syncs
-- syncs:
Sync "sync1"  Relgroup "source_group1" [Active]
DB group "sync1" target_db:target rds_source1:source
Sync "sync2"  Relgroup "source_group2" [Active]
DB group "sync2" target_db:target rds_source2:source
```

And now that we have everything set, you can start your applications again and the database is back open and running for everyone. All that's left is starting replication so that the tables will be copied and updates will be received by the slave.

```$ bucardo start```

If everything is fine, you will see Bucardo process running on your sources and slave and the status command will print the result.

```
$ bucardo status
PID of Bucardo MCP: 2729
Name        State    Last good    Time    Last I/D    Last bad    Time
==========+========+============+=======+===========+===========+=======
sync1     | Good   | 06:12:03   | 5s    | 5/120     | none      |
sync2     | Good   | 05:02:20   | 30s   | 0/20      | none      |
```

Insert new data on source1:

```
rds_source1 > INSERT into public.order values (10000,'This is order #10000',now(),12);
INSERT 0 1
```

Check Bucardo status:

```
$ bucardo status
PID of Bucardo MCP: 2729
Name        State    Last good    Time    Last I/D    Last bad    Time
==========+========+============+=======+===========+===========+=======
sync1     | Good   | 06:14:07   | 12s   | 0/1       | none      |
```

Verify on the slave:

```
target_db > SELECT * from public.order where id =10000;
id    │      order_text        │        create_time         │ operator_id
======+========================+============================+============
10000 │  This is order #10000  │ 2017-05-22 17:14:04.504965 │          12
```

That's how you replicate PostgreSQL database from Amazon RDS. Eventhough I haven't tested this in Google Cloud environment and other cloud services, as long as session replication set is available on the services, this should be doable.

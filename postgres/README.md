# PostgreSQL (9.6+)

For starters, FreeBSD is the only Unix distribution that packaged PostgreSQL properly.
The very cool thing about it is you can specify the default data directory directly in rc.conf.\
Also, everything lies in $PG\_DATA, even the configuration files.

So forget Debian, and ESPECIALLY forget Centoids! ;-)

Anyway, let's get started...

## Installation
```
pkg install postgresql96-server
```

There!

## Configuration recommendations (for both master and slave)

Basically, we're going to configure all nodes so they can all be master or slave.

In ```/etc/rc.conf```, set the following:

```
postgresql_enable="YES"
postgresql_data="/pgdata"
```

Then, build a ZFS array for the data:

```
zpool create pgdata da1
```

Use ```/usr/local/bin/initdb``` to create the PG base files and template configuration,
and we suggest you simply add the following configuration to ```/pgdata/postgresql.conf```:

```
listen_addresses = '*'
log_destination = 'stderr'
logging_collector = on
log_directory = 'pg_log'
log_filename = 'postgresql-%Y-%m-%d.log'
log_rotation_age = 1d

hot_standby = on
wal_level = hot_standby
archive_mode = on
archive_command = '/var/db/postgres/archive_command.sh "%p" "%f"'
archive_timeout = 60
max_wal_senders = 5
```

Then, ```pg_hba.conf``` has to contain the following:

```
host    replication     all     other_nodes/32        trust
host    all             all     pgpool_ip_address/32  trust
host    all             all     all             md5
```

Finally, make sure user postgres was created, change its shell to bash,\
create ```/pgdata/walarchives```, and change its ownership to postgres:postgres.

## Orchestration scripts (for both master and slave)

Place the following scripts under ~postgres; that way the same deployment can be used for stand-alone as well as master-slave instances.\
Note that ```recovery.conf``` is placed in ~postgres and only copied over to $PG\_DATA when needed (by a slave instance).

### archive\_command.sh
```
#!/bin/sh
fullpath="$1"
walfile="$2"
scp "$fullpath" pg02:/pgdata/walarchives/$walfile > /dev/null 2>&1
exit 0
```

### recovery.conf
```
standby_mode = 'on'
primary_conninfo = 'host=pg02 application_name=pg01'
trigger_file = '/pgdata/pgtrigger'
restore_command = 'cp /pgdata/walarchives/%f %p'
archive_cleanup_command = '/usr/local/bin/pg_archivecleanup /pgdata/walarchives %r'
```

### init\_as\_slave.sh
```
#!/bin/sh
PG_DATA="/pgdata"

pg_ctl -D $PG_DATA stop > /dev/null 2>&1
rm -rf $PG_DATA/*
rm -rf $PG_DATA/walarchives/*
pg_basebackup -D $PG_DATA --host=pg02 --port=5432
cp *.conf $PG_DATA/
```

## Monitoring

Here are a few tricks you can use to monitor the status of your nodes.

### Getting the replication status (from the master node)

```
postgres=# select * from pg_stat_replication ;
-[ RECORD 1 ]----+------------------------------
pid              | 27659
usesysid         | 10
usename          | postgres
application_name | pg02
client_addr      | 10.50.62.22
client_hostname  | 
client_port      | 33651
backend_start    | 2017-04-05 20:35:18.506734+02
backend_xmin     | 
state            | streaming
sent_location    | E/A4000060
write_location   | E/A4000060
flush_location   | E/A4000060
replay_location  | E/A4000060
sync_priority    | 0
sync_state       | async
```

### Monitoring WAL shipping

```
postgres=# select * from pg_stat_archiver ;
-[ RECORD 1 ]------+------------------------------
archived_count     | 3760
last_archived_wal  | 000000010000000E000000AA
last_archived_time | 2017-04-07 07:52:39.141417+02
failed_count       | 0
last_failed_wal    | 
last_failed_time   | 
stats_reset        | 2017-04-03 18:38:05.174681+02
```

## HA

- https://www.slideshare.net/dataloop/zero-downtime-postgres-upgrades
- https://gocardless.com/blog/zero-downtime-postgres-migrations-the-hard-parts/


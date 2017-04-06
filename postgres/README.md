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


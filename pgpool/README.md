# pgpool (3.4.9+)

pgpool for FreeBSD does have a few quirks, but nothing too bad.

## Installation
```
pkg install pgpool
```

## Configuration recommendations

In rc.conf, set the following:

```
pgpool_enable="YES"
pgpool_conf="/usr/local/etc/pgpool.conf"
pgpool_user="root"
```

Ok, here's the first quirk. I know you're not supposed to run anything directly as root, but the rc.d script
seems to fail when starting pgpool as anything other than root.\
I haven't figured that one out yet, but I'm pretty sure it should be easy to work around :)

Now, create a postgres user (or pgpool, whatever), and stick it under /home/postgres.\
The scripts we'll be creating later will have to be there.\
Then, in that directory, create subdirectories for each PostgreSQL instance you'll be using (pg01, pg02, etc.).

Then, put the following configuration in /usr/local/etc/pgpool.conf; note that you must manually find the tokens
and modify their default values.\
And another note: the listen\_addresses token cannot have '*' assigned to it: this is the second quirk I mentioned earlier.
Instead, you must use an IP address, or '0.0.0.0'. This is actually what we're going to do here for simplicity:

```
listen_addresses = '0.0.0.0'
port = 5432
backend_hostname0 = 'pg01'
backend_port0 = 5432
backend_weight0 = 2
backend_data_directory0 = '/home/postgres/pg01'
backend_flag0 = 'ALLOW_TO_FAILOVER'
backend_hostname1 = 'pg02'
backend_port1 = 5432
backend_weight1 = 1
backend_data_directory1 = '/home/postgres/pg02'
backend_flag1 = 'ALLOW_TO_FAILOVER'
replication_mode = off
replicate_select = off
load_balance_mode = on
master_slave_mode = on
master_slave_sub_mode = 'stream'
sr_check_period = 0
sr_check_user = 'postgres'
health_check_period = 5
health_check_timeout = 5
health_check_user = 'postgres'
failover_command = '/home/postgres/failover.sh %d %D %m %M %P %R'
```

Then, create a user/password for PCP (the pgpool administrator account), by adding the following to /usr/local/etc/pcp.conf:

`̀``
postgres:e8a48653851e28c69d0506508fb27fc5
```

Note that you can use any other user name, and that the password is an MD5 hash of "postgres" (which you can change as well... and that's highly recommended ;))

## Orchestration scripts

Place the following scripts under ~postgres:



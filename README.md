
# list ip

IP VM psql client : 10.8.60.205
IP VM SQ1 : 10.8.60.207 (DB Master)
IP VM SQ2 : 10.8.60.209 (DB Standby)
IP VM ES : 10.8.60.205
IP Virtual keepalive : 10.8.60.218


# on node all node

### setup and install postgre

```bash
dnf module install postgresql:12 -y
yum install postgresql-contrib -y
getenforce
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
firewall-cmd --add-port=5432/{tcp,udp} --permanent && firewall-cmd --reload
getenforce

```

### setup in postgre1

```bash
su - postgres
echo 'redhat123' > /var/lib/pgsql/.super_password
/usr/bin/initdb --pgdata=/var/lib/pgsql/data --encoding=UTF8 --username=postgres --pwfile=/var/lib/pgsql/.super_password
mkdir /var/lib/pgsql/archivedir && chown -R postgres:postgres /var/lib/pgsql/archivedir
vim /var/lib/pgsql/data/postgresql.conf

...
listen_addresses = '*'
wal_level = replica
wal_log_hints = on
archive_mode = on
archive_command = 'test ! -f /var/lib/pgsql/archivedir/%f && cp %p /var/lib/pgsql/archivedir/%f'
max_wal_senders = 10
wal_keep_segments = 10
...

vim /var/lib/pgsql/data/pg_hba.conf

...
# for sonar
local   all             all            							md5
host    all             all             127.0.0.1/32            md5
host    all             all             ::1/128                 md5
# for ha
host    replication     all				10.8.60.209/24			trust 
host    all       		all				10.8.60.205/24			trust
...

systemctl enable --now postgresql.service
```

### setup replication stream in postgre1

```bash
su - postgres
psql -U postgres -d postgres
CREATE USER replman REPLICATION LOGIN CONNECTION LIMIT 5 ENCRYPTED PASSWORD 'redhat123';
SELECT * FROM pg_create_physical_replication_slot('pg_replslot_001');
select slot_name,plugin,slot_type,temporary,active from pg_replication_slots;
\q

vim /var/lib/pgsql/data/pg_hba.conf
...
host    replication    replman    		10.8.60.218/24    		md5
...

systemctl reload postgresql.service
```

### setup replication stream in postgre2
```bash
mkdir /var/lib/pgsql/archivedir && chown -R postgres:postgres /var/lib/pgsql/archivedir
su - postgres
pg_basebackup -D /var/lib/pgsql/data -U replman -h 10.8.60.207 -p 5432 -X stream -F p -c fast -W -R -v
vim /var/lib/pgsql/data/postgresql.conf

...
listen_addresses = '*'
wal_level = replica
archive_mode = on
archive_command = 'test ! -f /var/lib/pgsql/archivedir/%f && cp %p /var/lib/pgsql/archivedir/%f'
restore_command = 'cp /var/lib/pgsql/archivedir/%f %p' 
archive_cleanup_command = 'pg_archivecleanup /var/lib/pgsql/archivedir/ %r'
max_wal_senders = 10
wal_keep_segments = 10
hot_standby = on
...

systemctl enable --now postgresql.service
psql -U postgres -c "SELECT * FROM pg_create_physical_replication_slot('pg_replslot_001');"
psql -U postgres -c "select slot_name,plugin,slot_type,temporary,active from pg_replication_slots;"
```




### check replication stream in postgre1

```bash
pg_controldata -D $PGDATA |grep  cluster
psql -U postgres -x -c "select * from pg_stat_replication;"
psql -U postgres -c "select slot_name,plugin,slot_type,temporary,active from pg_replication_slots;"
```

### create database pgmonitor for monitoring keepalived replication status (on master node / postgre1)

```bash
su - postgres
psql -U postgres
create role pgmonitor superuser nocreatedb nocreaterole noinherit login encrypted password 'redhat123';
create database pgmonitor with template template0 encoding 'UTF8' owner pgmonitor;
\c pgmonitor pgmonitor
create schema pgmonitor;
create table cluster_status (id int unique default 1, last_alive timestamp(0) without time zone);

CREATE FUNCTION cannt_delete ()
    RETURNS trigger
    LANGUAGE plpgsql AS $$
    BEGIN
    RAISE EXCEPTION 'You can not delete!';
    END; $$;

CREATE TRIGGER cannt_delete BEFORE DELETE ON cluster_status FOR EACH ROW EXECUTE PROCEDURE cannt_delete();
CREATE TRIGGER cannt_truncate BEFORE TRUNCATE ON cluster_status FOR STATEMENT EXECUTE PROCEDURE cannt_delete();

insert into cluster_status values (1, now());
\q

vim /var/lib/pgsql/data/pg_hba.conf
...
host    pgmonitor      pgmonitor  127.0.0.1/32      md5
host    pgmonitor      pgmonitor  10.8.60.209/32    md5
host    pgmonitor      pgmonitor  10.8.60.207/32    md5
host    pgmonitor      pgmonitor  10.8.60.218/32    md5
...

systemctl reload postgresql.service
```

### Test create table and insert in postgre1 
```bash
su - postgres
createdb testingdb
psql -d testingdb -U postgres -c "CREATE TABLE test_table(x integer)"
psql -d testingdb -U postgres -c "INSERT INTO test_table(x) SELECT y FROM generate_series(1, 2) a(y)"
psql -d testingdb -U postgres -c "SELECT count(*) from test_table"
```

### check table in postgre2
```bash
su - postgres
psql -d testingdb -U postgres -c "SELECT count(*) from test_table"
```

### install and configure keepalived (execute on postgre1 & postgre2)

```bash
yum install keepalived -y
mkdir -p /var/log/keepalived

cat << EOF >> /etc/rsyslog.conf
# Keepalived log setting:
local1.*    /var/log/keepalived/keepalived.log
EOF

systemctl restart rsyslog.service
systemctl disable keepalived.service

```

### configure keepalived in postgre1
```bash
mkdir -p /etc/keepalived/scripts
mv /etc/keepalived/keepalived.conf /etc/keepalived/keepalived.conf.backup
vim /etc/keepalived/keepalived.conf
...
! Configuration File for keepalived
global_defs {
    vrrp_no_swap
    checker_no_swap
    script_user root root
    router_id PG_HA_ROUTER1
}

vrrp_script check_pg_status {
    script "/etc/keepalived/scripts/pg_checkstatus.sh"
    interval 3			# exec ph_checkstatus.sh per 3 second
    fall 1                  # require 1 failures for KO
    user root root
}

vrrp_instance VRRP_INST1 {
    state MASTER
    interface ens18 #change this to your device ethernet
    virtual_router_id 99
    priority 200
    nopreempt # for allow  maintain the master role
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass redhat123
    }
    unicast_src_ip 10.8.60.207 # Private IP address of master
    unicast_peer {
        10.8.60.209          # Private IP address of the backup
    }
    track_script {
        check_pg_status
    }

    virtual_ipaddress {
        10.8.60.218/24 dev ens18 label ens18:1
    }

    notify_master "/etc/keepalived/scripts/pg_failover.sh"
    notify_fault "/etc/keepalived/scripts/pg_fault.sh"
    notify_stop "/etc/keepalived/scripts/pg_fault.sh"
}
...

vim /etc/keepalived/scripts/pg_checkstatus.sh

...
#!/usr/bin/env bash

export PGPORT=5432
export PGUSER=pgmonitor
export PGDBNAME=pgmonitor
export PGDATA=/var/lib/pgsql/data

LOGFILE=/var/log/keepalived/keepalived_pg.log

SQL1='SELECT pg_is_in_recovery from pg_is_in_recovery();'
SQL2='update cluster_status set last_alive = now() where id = 1;'
#SQL3='SELECT 1;'

DB_ROLE=`echo $SQL1  |psql -d $PGDBNAME -U $PGUSER -At -w`

if [ $DB_ROLE == 't' ] ; then
    echo -e `date +"%F %T"` "`basename $0`: [INFO] PostgreSQL is running in STANDBY mode." >> $LOGFILE
        exit 0
elif [ $DB_ROLE == 'f' ]; then
    echo -e `date +"%F %T"` "`basename $0`: [INFO] PostgreSQL is running in PRIMARY mode." >> $LOGFILE
fi

# If current server is in STANDBY mode, then exit. Otherwise, update the cluster_status table. 

#echo $SQL3 |psql -p $PGPORT -d $PGDBNAME -U $PGUSER -At -w &> /dev/null
echo $SQL2 |psql -p $PGPORT -d $PGDBNAME -U $PGUSER -At -w
if [ $? -ne 0 ] ;then
    echo -e `date +"%F %T"` "`basename $0`: [ERR] Cannot update 'cluster_status' table, is PostgreSQL running?" >> $LOGFILE
    exit 1
#else
#    echo -e `date +"%F %T"` "`basename $0`: [INFO] Table 'cluster_status' is successfully updated." >> $LOGFILE
#    exit 0
fi
...

vim /etc/keepalived/scripts/pg_failover.sh

...
#!/usr/bin/env bash

export PGPORT=5432
export PGUSER=pgmonitor
export PG_OS_USER=postgres
export PGDBNAME=pgmonitor
export PGDATA=/var/lib/pgsql/data
export master_ip="10.8.60.207"
export slave_ip="10.8.60.209"
export pg_ctl="/usr/bin/pg_ctl"
export pg_home="/var/lib/pgsql/data"


LOGFILE=/var/log/keepalived/keepalived_pg.log

PROMOTE_COMMAND="pg_ctl promote -D $PGDATA"

echo -e `date +"%F %T"` "`basename $0`: [INFO] Checking if there needs a promote operation... " >> $LOGFILE

SQL1="select pg_is_in_recovery from pg_is_in_recovery();"
SQL2="select last_alive from cluster_status where now()-last_alive > interval '60';"

DB_ROLE=`echo $SQL1 |psql -At -p $PGPORT -U $PGUSER -d $PGDBNAME -w`

# If current node is in PRIMARY mode, then do nothing. Otherwise, promote it to PRIMARY mode.
if [ $DB_ROLE == 'f' ]; then
    echo -e `date +"%F %T"` "`basename $0`: [WARN] PostgreSQL is running in PRIMARY mode." >> $LOGFILE
    exit 0
elif [ $DB_ROLE == 't' ]; then
    echo -e `date +"%F %T"` "`basename $0`: [WARN] PostgreSQL is running in STANDBY mode, ready to promote..." >> $LOGFILE
fi

# script condition failover from primary to standby
(echo >/dev/tcp/"$master_ip"/5432) &>/dev/null && echo -e `date +"%F %T"` "PRIMARY is RUNNING no need to failover , ALL IS OKAY " >> $LOGFILE && exit 0 || echo "PROCESSING TO PROMOTED" >> $LOGFILE

# Prerequisites for an PRIMARY and STANDBY switching:
#   1 Current node must be in STANDBY mode.
#   2 Delay-time between PRIMARY and STANDBY nodes must be as small as possible(60s).

STANDBY_DELAY=`echo $SQL2 |psql -At -p $PGPORT -U $PGUSER -d $PGDBNAME -w`
if [ -z $STANDBY_DELAY ] && [ $DB_ROLE == 't' ]; then
        echo -e `date +"%F %T"` "`basename $0`: [WARN] Promote the current node to PRIMARY mode..." >> $LOGFILE
        su - $PG_OS_USER -c "$PROMOTE_COMMAND"
        if [ $? -eq 0 ]; then
                echo -e `date +"%F %T"` "`basename $0`: [INFO] Promote the current node to PRIMARY mode success." >> $LOGFILE
                exit 0
        else
                echo -e `date +"%F %T"` "`basename $0`: [ERR] Promote the current node to PRIMARY failed, check logfile for details." >> $LOGFILE
                exit 0
        fi
else
    echo -e `date +"%F %T"` "`basename $0`: [WARN] Stream replication delay time is about: $STANDBY_DELAY""s." >> $LOGFILE
    echo -e `date +"%F %T"` "`basename $0`: [ERR] STANDBY node is too far behind the MASTER node, you must repair the cluster manually" >> $LOGFILE
    exit 1
fi

...

vim /etc/keepalived/scripts/pg_fault.sh

...
#!/usr/bin/env bash

export PGPORT=5432
export PGUSER=pgmonitor
export PG_OS_USER=postgres
export PGDBNAME=pgmonitor
export PGDATA=/var/lib/pgsql/data

LOGFILE=/var/log/keepalived/keepalived_pg.log

echo -e "`date +%F\ %T`" "`basename $0`: [WARN] Keepalived failed, Checking if need to shutdown PostgreSQL... " >> $LOGFILE

SQL1="select pg_is_in_recovery from pg_is_in_recovery();"
DB_ROLE=`echo $SQL1 |psql -At -p $PGPORT -U $PGUSER -d $PGDBNAME -w`

# If the keepalived daemon on primary entering FAULT state, then shutdown the postgresql server.
# If the keepalived daemon on standby entering FAULT state, then , do nothing.
if [ $DB_ROLE == 'f' ]; then
    echo -e "`date +%F\ %T`" "`basename $0`: [WARN] PostgreSQL is running in PRIMARY mode, shutting down... " >> $LOGFILE
    systemctl stop postgresql.service
    if [ $? -eq 0 ]; then
        echo -e "`date +%F\ %T`" "`basename $0`: [INFO] Shutdown PostgreSQL success." >> $LOGFILE
    else
        echo -e "`date +%F\ %T`" "`basename $0`: [ERR] Shutdown PostgreSQL failed, check logfile for details." >> $LOGFILE
    fi
elif [ $DB_ROLE == 't' ]; then
    echo -e "`date +%F\ %T`" "`basename $0`: [INFO] PostgreSQL is running in STANDBY mode, not recommended to shutdown." >> $LOGFILE
fi

...


```

### configure keepalived in postgre2
```bash
mkdir -p /etc/keepalived/scripts
mv /etc/keepalived/keepalived.conf /etc/keepalived/keepalived.conf.backup
vim /etc/keepalived/keepalived.conf
...
! Configuration File for keepalived
global_defs {
    vrrp_no_swap
    checker_no_swap
    script_user root root
    router_id PG_HA_ROUTER1
}

vrrp_script check_pg_status {
    script "/etc/keepalived/scripts/pg_checkstatus.sh"
    interval 3          # exec ph_checkstatus.sh per 3 second
    fall 1                  # require 1 failures for KO
    user root root
}

vrrp_instance VRRP_INST1 {
    state BACKUP
    interface ens18 #change this to your device ethernet
    virtual_router_id 99
    priority 100
    nopreempt # for allow  maintain the master role
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass redhat123
    }
    unicast_src_ip 10.8.60.209          # Private IP address of the backup
    unicast_peer {
        10.8.60.207           # Private IP address of master
    }
    track_script {
        check_pg_status
    }

    virtual_ipaddress {
        10.8.60.218/24 dev ens18 label ens18:1
    }

    notify_master "/etc/keepalived/scripts/pg_failover.sh"
    notify_fault "/etc/keepalived/scripts/pg_fault.sh"
    notify_stop "/etc/keepalived/scripts/pg_fault.sh"
}
...

```
>> for the script same like postgres1


### start keepalived on postgre1
```bash
chmod 700 /etc/keepalived/scripts/*.sh
systemctl start keepalived.service
tail -f /var/log/keepalived/keepalived.log
```

### after keepalived daemon success starting , then start keepalived on postgre2
```bash
chmod 700 /etc/keepalived/scripts/*.sh
systemctl start keepalived.service
tail -f /var/log/keepalived/keepalived.log
```

# Maintenance Recover old master to master again

### Start machine postgre1 again and follow this command
```bash
systemctl stop keepalived.service
systemctl stop postgresql.service
su - postgres
pg_rewind --target-pgdata=/var/lib/pgsql/data --source-server="host=10.8.60.209 port=5432 user=postgres password=redhat123"
```

### Stop service standby postgres2

```bash
systemctl stop keepalived.service
systemctl stop postgresql.service
su - postgres
mv /var/lib/pgsql/data /var/lib/pgsql/data_old
mkdir -p /var/lib/pgsql/data
```

### Start service postgre1

```bash
systemctl start postgresql.service
systemctl status postgresql.service
```

### Start service postgre2 and configure standby again
```bash
su - postgres
pg_basebackup -D /var/lib/pgsql/data -U replman -h 10.8.60.207 -p 5432 -X stream -F p -c fast -W -R -v
su - root
chmod -R 700 /var/lib/pgsql
systemctl start postgresql.service
systemctl status postgresql.service
```

### Start service keepalived postgre1
```bash
systemctl start keepalived.service
systemctl status keepalived.service
```

### Start service keepalived postgre2
```bash
systemctl start keepalived.service
systemctl status keepalived.service
```
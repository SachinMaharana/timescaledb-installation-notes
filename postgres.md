```
https://docs.timescale.com/latest/getting-started/installation/rhel-centos/installation-yum


// All Servers

sudo su -

yum update -y && yum install -y vim

vim /etc/yum.repos.d/CentOS-Base.repo
 
exclude=postgresql*

in [base] and [updates] section

yum install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm

yum install -y postgresql12-server

/usr/pgsql-12/bin/postgresql-12-setup initdb

systemctl enable postgresql-12

systemctl restart postgresql-12

systemctl status postgresql-12


sudo tee /etc/yum.repos.d/timescale_timescaledb.repo <<EOL
[timescale_timescaledb]
name=timescale_timescaledb
baseurl=https://packagecloud.io/timescale/timescaledb/el/7/\$basearch
repo_gpgcheck=1
gpgcheck=0
enabled=1
gpgkey=https://packagecloud.io/timescale/timescaledb/gpgkey
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt
metadata_expire=300
EOL

sudo yum update -y

sudo yum install -y timescaledb-postgresql-12

sudo timescaledb-tune --pg-config=/usr/pgsql-12/bin/pg_config

systemctl restart postgresql-12


su - postgres

psql

CREATE database tutorial;

\c tutorial

CREATE EXTENSION IF NOT EXISTS timescaledb;

psql -U postgres -h localhost -d tutorial // fails

sudo -u postgres psql

ALTER USER postgres PASSWORD 'postgres';

vim /var/lib/pgsql/12/data/pg_hba.conf
With md5, trust: 

host    all             all             127.0.0.1/32            md5         
# IPv6 local connections:                                                       
host    all             all             ::1/128                 trust 

systemctl restart postgresql-12

psql -U postgres -h localhost -d tutorial


su - postgres
psql -c "ALTER SYSTEM SET listen_addresses TO '*';"

psql

SET password_encryption = 'scram-sha-256';

```
--- 


```
// Primary Server

CREATE ROLE repuser WITH REPLICATION PASSWORD 'reppassword' LOGIN;

vim /var/lib/pgsql/12/data/pg_hba.conf

host    replication     repuser    172.31.30.115/32   scram-sha-256

systemctl restart postgresql-12

vim /var/lib/pgsql/12/data/postgresql.conf

listen_addresses = '*'
wal_level = replica
max_wal_senders = 1
max_replication_slots = 1
synchronous_commit = off

systemctl restart postgresql-12
```

```
// Secondary Servers

systemctl stop postgresql-12.service

su - postgres

cp -R /var/lib/pgsql/12/data /var/lib/pgsql/12/data_orig

rm -rf /var/lib/pgsql/12/data/*

pg_basebackup -h 172.31.17.73 -D /var/lib/pgsql/12/data -U repuser -P -v  -R -X stream -C -S replica_2_slot

ls -l /var/lib/pgsql/12/data/

sudo chown -Rf postgres:postgres /var/lib/pgsql/12/data/

hot_standby = on
wal_level = replica
max_wal_senders = 2
max_replication_slots = 2
synchronous_commit = off

systemctl restart postgresql-12
```

```
// Primary
su - postgres

psql

select * from pg_stat_replication; 

SELECT client_addr, state FROM pg_stat_replication;

psql -c "\x" -c "SELECT * FROM pg_stat_replication;"

// standby and master
psql -c "\x" -c "SELECT * FROM pg_stat_wal_receiver;" in standby
psql -c "\x" -c "SELECT * FROM pg_stat_replication;" in master


less /var/lib/pgsql/12/data/log/postgresql-Wed.log


// Primary

CREATE database tuts;

\c tuts

CREATE EXTENSION IF NOT EXISTS timescaledb;

CREATE TABLE conditions (
  time        TIMESTAMPTZ       NOT NULL,
  location    TEXT              NOT NULL,
  temperature DOUBLE PRECISION  NULL,
  humidity    DOUBLE PRECISION  NULL
);

SELECT create_hypertable('conditions', 'time');


INSERT INTO conditions(time, location, temperature, humidity)
  VALUES (NOW(), 'office', 70.0, 50.0);

SELECT * FROM conditions ORDER BY time DESC LIMIT 100;


// Secondary

psql -U postgres -h localhost -d tuts

SELECT * FROM conditions ORDER BY time DESC LIMIT 100;

```





**the open source tool repmgr (Replication Manager) and how to set up and configure it for automatic failover in PostgreSQL.**

1. Install PostgreSQL
2. Install repmgr
3. Configure PostgreSQL
4. Create users
5. Configure pg_hba.conf
6. Configure the repmgr file
7. Register the primary server
8. Build/clone the standby server
9. Register the standby server
10. Start repmgrd daemon process



Step 1: Install PostgreSQL
Create two clusters/servers with the PostgreSQL installation. You can follow the PostgreSQL instructions at the link below for installation using PostgreSQL’s PGDG repo package. For the sake of naming conventions, we will consider master and standby as two servers.
#ubuntu

`wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -`

```
echo "deb http://apt.postgresql.org/pub/repos/apt/ `lsb_release -cs`-pgdg main" |sudo tee  /etc/apt/sources.list.d/pgdg.list
sudo apt update
sudo apt -y install postgresql-12 postgresql-client-12
systemctl enable --now postgresql-12
systemctl start postgresql-12
systemctl status postgresql-12
```

#centos

```
wget https://download.postgresql.org/pub/repos/yum/9.6/redhat/rhel-7-x86_64/pgdg-centos96-9.6-3.noarch.rpm
yum install postgresql12-server postgresql12-contrib
```

#Initialize the Database

` /usr/pgsql-12/bin/postgresql12-setup initdb`


Step 2: Install repmgr
You will need to install repmgr on the master as well as standby.

You can use the [link](https://repmgr.org/docs/4.0/installation-packages.html) to run on different versions of Linux


Step 3: Configure PostgreSQL
On the primary server, a PostgreSQL instance must be initialized and running. The following replication settings may need to be adjusted:

```
max_wal_senders = 10 
wal_keep_segments = 64
max_replication_slots = 10 
wal_level = 'hot_standby' or 'replica' or 'logical' 
hot_standby = on 
archive_mode = on 
archive_command = '/bin/true' 
shared_preload_libraries = 'repmgr'
```


Step 4: Create users
Create a dedicated PostgreSQL superuser account and a database for the repmgr metadata:

```
create user repmgr; 
create database repmgr with owner repmgr;
ALTER USER repmgr WITH SUPERUSER;
```

Step 5: Configure pg_hba.conf
Ensure the repmgr user has appropriate permissions in pg_hba.conf and can connect in replication mode; pg_hba.conf should contain entries similar to the following:

```
local      replication      repmgr                         trust
host       replication      repmgr        127.0.0.1/32     trust
host       replication      repmgr        <ip/netmask>     trust
local       repmgr          repmgr                         trust
host        repmgr          repmgr        127.0.0.1/32     trust
host        repmgr          repmgr        <ip/netmask>     trust
```
Note: Adjust above settings according to your network configurations.



Step 6: Configure the repmgr file
Create a repmgr.conf on the master server with the following entries:

```
node_id=1
node_name=node1
conninfo='host=node1 user=repmgr dbname=repmgr connect_timeout=2'
data_directory='/var/lib/pgsql/12/data/'
se_replication_slots = yes
log_file = '/var/log/postgresql/repmgr.log'
reconnect_attempts = 3
reconnect_interval = 1
failover=automatic
promote_command='/usr/pgsql-12/bin/repmgr standby promote -f /var/lib/pgsql/repmgr.conf –log-to-file'
follow_command='/usr/pgsql-12/bin/repmgr standby follow -f /var/lib/pgsql/repmgr.conf --log-to-file –upstream-node-id=%n'
```




Step 7: Register the primary server
Register the primary server with repmgr:

`repmgr  primary register`

Then check the status of the cluster:

`repmgr  cluster show`




Step 8: Build/clone the standby server
Create the repmgr.conf file on standby server:

```
node_id=2
node_name=node2
conninfo='host=node2 user=repmgr dbname=repmgr connect_timeout=2'
data_directory='/var/lib/pgsql/12/data'
se_replication_slots = yes
log_file = '/var/log/postgresql/repmgr.log'
reconnect_attempts = 3
reconnect_interval = 1
failover=automatic
promote_command='/usr/pgsql-12/bin/repmgr standby promote -f /var/lib/pgsql/repmgr.conf --log-to-file'
follow_command='/usr/pgsql-12/bin/repmgr standby follow -f /var/lib/pgsql/repmgr.conf --log-to-file –upstream-node-id=%n'
```


We can now perform the dry run and test if our configuration is correct:

`repmgr h <ipmaster> -U repmgr -d repmgr standby clone –dry-run`

If there is no problem, start cloning:

`repmgr h <ipmaster> -U repmgr -d repmgr standby clone`



Step 9: Register the standby server
Register the standby server with repmgr:

`repmgr standby register`



Step 10: Start repmgrd daemon process
To enable the automatic failover, we now need to start the repmgrd daemon process on master slave:

`/usr/pgsql-12/bin/repmgrd -f /var/lib/pgsql/repmgr.conf`

We can also check the events for the cluster:

`repmgr  cluster event`




stop 11: recovery online
To get started, first stop the main database and reattach it to the cluster as a ready-made database according to the following instructions.


```
systemctl stop postgresql


repmgr -h <ipmaster2> -U repmgr -d repmgr  standby clone --force


systemctl start postgresql


repmgr  standby register --force


repmgr  cluster show

note:Make sure the engine is running ‘/tmp/repmgrd.pid’


systemctl enable repmgrd
systemctl start repmgrd
systemctl status repmgrd
```

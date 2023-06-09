# Steps to Create InnoDB Clustet

### Change root password on all nodes
```
cat /var/lib/mysql/error.log | grep password

mysql -uroot -h::1 -p -e "alter user root@'localhost' identified by 'Root_123'";
```
### Tune /etc/my.cnf and restart
```
[mysqld]
datadir=/var/lib/mysql
binlog-format=ROW
log-bin=/var/lib/mysql/bin
port=3306
server_id=10
socket=/var/lib/mysql/mysqld.sock
log-error=/var/lib/mysql/error.log
enforce_gtid_consistency = ON
gtid_mode = ON
log_slave_updates = ON

# Tuning
innodb_buffer_pool_size=6442450944
innodb_buffer_pool_instances=4		
innodb_redo_log_capacity=3221225472
innodb_use_fdatasync=on
innodb_flush_neighbors=1
innodb_io_capacity=4000
innodb_io_capacity_max=4000
innodb_lru_scan_depth=1000
innodb_read_io_threads=2
innodb_write_io_threads=2
innodb_checksum_algorithm=strict_crc32
binlog_row_image=minimal
innodb_log_compressed_pages=off
sort_buffer_size=8M		
table_open_cache=2048
innodb_open_files=2048
open_files_limit=100000 
max_prepared_stmt_count=1048576
innodb_change_buffer_max_size=25
innodb_adaptive_hash_index=off
innodb_doublewrite=0
innodb_thread_sleep_delay=500
innodb_spin_wait_delay=2
innodb_spin_wait_pause_multiplier=10
max_delayed_threads=0
delayed_queue_size=1
delayed_insert_timeout=1
delayed_insert_limit=1
thread_cache_size=1000
innodb_lock_wait_timeout=10

log_bin_trust_function_creators=on

# Group Replication related
# group_replication_paxos_single_leader=on
sql_generate_invisible_primary_key=on

# thread pool
plugin-load-add=thread_pool.so
thread_pool_size=8			 # 2x cpu core
thread_pool_max_transactions_limit=400	 # 100x cpu core 
thread_pool_dedicated_listeners=on
thread_pool_stall_limit=200		 # stall limit is 2 sec
thread_pool_prio_kickup_timer=2000	 # move to high prio after 2 sec
```
Restart
```
sudo systemctl restart mysqld
```

### Fix /etc/hosts

### Run configure-instance on all nodes
```
mysqlsh -- dba configure-instance { --host=127.0.0.1 --port=3306 --user=root --instance-password=Root_123 } --clusterAdmin=gradmin --clusterAdminPassword='grpass' --interactive=false --restart=true
```
### Create Cluster
On node-1
```
mysqlsh gradmin:grpass@localhost:3306 -- dba createCluster mycluster

mysqlsh gradmin:grpass@localhost:3306 -- cluster status
```
### Add node 
on node-1
```
mysqlsh gradmin:grpass@localhost:3306 -- cluster add-instance gradmin:grpass@<node-2>:3306 --recoveryMethod=clone
mysqlsh gradmin:grpass@localhost:3306 -- cluster add-instance gradmin:grpass@<node-3>:3306 --recoveryMethod=clone

mysqlsh gradmin:grpass@localhost:3306 -- cluster status
```
### Bootstrap Router
On MySQL Router
```
mysqlrouter --bootstrap gradmin:grpass@<node-1>:3306 --directory=router

router/start.sh

mysqlsh gradmin:grpass@localhost:6446 --sql -e "select @@hostname"
mysqlsh gradmin:grpass@localhost:6447 --sql -e "select @@hostname"
mysqlsh gradmin:grpass@localhost:6447 --sql -e "select @@hostname"
```
### Change Primary
on any DB node
```
mysqlsh gradmin:grpass@localhost:3306 -- cluster setPrimaryInstance <db-2>:3307

mysqlsh gradmin:grpass@localhost:3306 -- cluster status
```
On MySQL Router
```
mysqlsh gradmin:grpass@localhost:6446 --sql -e "select @@hostname"
mysqlsh gradmin:grpass@localhost:6447 --sql -e "select @@hostname"
mysqlsh gradmin:grpass@localhost:6447 --sql -e "select @@hostname"
```

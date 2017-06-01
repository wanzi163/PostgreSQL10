# postgresql replication zero setup

## descrption

postgresql version: 10beta1

> refer to document link https://cloud.google.com/community/tutorials/setting-up-postgres-hot-standby

## create replication user

`$ createuser -U <role_name> repuser -P -c 5 --replication`

the role_name is the role you are currently using, i use denny, and the official procedure use postgres

## set the replication access permission on master node

```
# in the pg_hba.conf
host  replication  replicator <ip of the standby node>   md5
```

of course if the standby node is on the same host of the master node, the default configuration is enough.

## create archive directory

just create a directory to hold the WAL archive file, there will be a archive_command in the conf file that will copy WAL file to the directory.

## change master node configuration

```
listen_address = "*"
wal_level = hot_standby
wal_keep_segments = 8
max_wal_senders = 3
archive_mode = on
achive_mode = archive_command = 'cp %p /home/denny/project/postgres/instance/archive/%f'
```

## restart master server

./pg_ctl -D <pgdata_path> stop



## basebackup files from master to standby server

```
pg_basebackup -h <master_ip> -D <pgdata_path_standby> -U repuser -v -P --wal-method=stream
```

## change setting for standby server

```
hot_standby = on
# change the port number if the standby is located in the same server with master
port = 5430
```

## create and edit the recovery.conf file

```
standby_mode = on
primary_conninfo = 'host=127.0.0.1 port=5432 user=repuser passwd=rep123'
```

## start the standby server

```
./pg_ctl -D <pgdata_path_backup> -l logfile_backup start
2017-06-01 16:29:09.544 CST [62912] LOG:  listening on IPv4 address "0.0.0.0", port 5430
2017-06-01 16:29:09.544 CST [62912] LOG:  listening on IPv6 address "::", port 5430
2017-06-01 16:29:09.606 CST [62912] LOG:  listening on Unix socket "/tmp/.s.PGSQL.5430"
2017-06-01 16:29:09.685 CST [62913] LOG:  database system was shut down in recovery at 2017-06-01 16:29:06 CST
2017-06-01 16:29:09.685 CST [62913] LOG:  entering standby mode
2017-06-01 16:29:09.722 CST [62913] LOG:  redo starts at 0/2000060
2017-06-01 16:29:09.722 CST [62913] LOG:  consistent recovery state reached at 0/3000000
2017-06-01 16:29:09.722 CST [62912] LOG:  database system is ready to accept read only connections
2017-06-01 16:29:09.728 CST [62917] LOG:  started streaming WAL from primary at 0/3000000 on timeline 1
```


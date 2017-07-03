# pg_basebackup 分析

## 概要

## pg_basebackup命令对应的源码文件

### 命令客户端通过sendQuery函数发送命令给pg服务进程

BASE_BACKUP LABLE ‘pg_basebackup base backup’ "progress"

上面的`pg_basebackup base backup`实际上是一个默认的label，同时也会表明这是一个排他的backup执行过程（在后面的章节会说明什么是排他）

## basebackup服务端处理过程

### 命令解析

### 服务函数入口

postgres.c::PostgresMain()=>处理Q请求=>walsender.c::replication_command()=>parser()解析命令，得到一个T_BaseBackupCmd类型的node=>basebackup.c::SendBaseBackup()=>basebackup.c::perform_base_backup()=>basebackup.c::do_pg_start_backup();拷贝数据;basebackup.c::do_pg_stop_backup()

### 备份执行过程

1. 执行do_pg_start_backup，来做一个数据确认检查点，未来在备端进行精确的数据恢复起始点。
   首先做这个操作的目的是，因为在向备端传送文件的过程中，主端一直都在进行着请求处理，也就是一直在做着数据文件和一些状态文件的修改，因为文件拷贝的时间可能会比较长，文件也是一个一个的拷贝到备端，到达备端的文件很可能就是不一致的数据状态，这是一个很难处理的状态，所以未来能够在恢复的时候找到一个大家一一致的数据状态点，在backup执行之前，要做一个checkpoint，这个checkpoint数据会被写入到一个教backup_label文件，这个文件也会传送到备端，给备端做数据恢复起点，也就是说，备端会认为这个起点就是一个数据一致的起点，就会认为后面的数据都是不可信的，要通过xlog的redo进行覆盖，对于之前的数据都可以作为可信的数据保留。

 再做basebackup的时候，lable文件名为backup_lable，文件内容有：

 如果输入的labelfile的名称没有提供的话，那么就认为是排他的备份，如果指定了文件名，那么就不是排他的。

 ```
 各个字段的定义
 - START WAL LOCATION: xx/xx (xlog filename) <= controlfile中的checkpointCopy.redo，即当前checkpoint的redo起始位置。
 - CHECKPOINT LOCATION: xx/xx <= controlfile中的checkPoint，即当前最新的一个checkpoint的LSN
 - BACKUP METHOD: pg_start_backup or streamed <=如果是排他的，那么就是pg_start_backup，否者就是streamed，如果是直接使用pg_basebackup命令的话，那么就是一个排他的过程，因为这个命令会制定一个默认的label名字，从而决定是一个排他的过程。
 - BACKUP FROM: standby or master
 - START TIME: %Y-%m-%d %H:%M:%S %Z
 - LABLE: label name
 ```

 ```
 文件的真是数据样例
 START WAL LOCATION: 0/A000060 (file 00000001000000000000000A)
 CHECKPOINT LOCATION: 0/A000098
 BACKUP METHOD: streamed
 BACKUP FROM: master
 START TIME: 2017-06-12 17:08:33 CST
 LABEL: pg_basebackup base backup
 ```

2. 上一步的checkpoint的redo位置和时间线发送给备端
   通过pg内部的消息队列发送给备端。

3. 把所有数据目录下的文件发送到备端
   实际上是通过使用tar的协议通过消息管道发送给备端，所谓tar的方式就是根据把目录中的所有文件做成tar的格式，即先统计文件大小，文件路径名称信息，然后附带着文件内容一起发送给备端。

4. 执行do_pg_stop_backup
   实际上就是在primary节点上面删除掉label文件，但是需要注意的是，这个时候备机上面已经把这个文件拷贝过去了，备机是存在这个文件的，等到备机做了启动之后，这个label文件在备机上面会被修改名字为.old后缀的文件。

5. 获取backup label文件中指定的那个xlog文件 
   创建xlog stream数据同步进程，获取上面创建的checkpoint点的redo指向的那个xlog文件，并且通过pg的replication slot机制把指定的那个xlog文件传输过来。

## 备注

### sendDIR()函数解析
对数据目录的第一层的所有文件进行排查，用tar的格式传输到备端。
需要注意的是，对于几个目录，实际上只是拷贝其目录，不拷贝里面的文件，比如pg_wal，pg_notify等，因为这些目录将会用来复制权限关系，而里面的文件主要都是动态生成的，不需要拷贝。
其他的文件都会拷贝到备端，目录的话都会做递归拷贝，比如base和global目录下的所有子目录，子目录下的所有文件都会被拷贝到备端。





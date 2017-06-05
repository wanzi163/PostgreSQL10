# pg_basebackup 分析

## 概要

## pg_basebackup命令对应的源码文件

### 命令客户端通过sendQuery函数发送命令给pg服务进程

BASE_BACKUP LABLE pg_basebackup "progress"

## basebackup服务端处理过程

### 命令解析

### 服务函数入口

### 备份执行过程

首先执行do_pg_start_backup来生成一个lable文件， lable文件内容：

- START WAL LOCATION:  xx/xx (xlog filename)
- CHECKPOINT LOCATION: xx/xx
- BACKUP METHOD: pg_start_backup or streamed
- BACKUP FROM: standby or master
- START TIME: %Y-%m-%d %H:%M:%S %Z
- LABLE: lable name
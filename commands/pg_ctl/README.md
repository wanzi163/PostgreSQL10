# 命令说明
此命令是PostgreSQL进行一般启动，重启等运维操作的主要命令入口，通过参数来指定希望PostgreSQL执行何等操作，
支持的操作有：启动，重启，重新加载，杀死进程，升级，查看状态，本文会对这集中操作做一个比较详细的说明。

# 命令支持的参数

- --help -h 打印帮助信息
- --version -V 打印数据库版本信息
- --log -l <日志文件目录> 指定日志文件
- --mode -m <模式符号>  s/smart 指定的是SMART_MODE，f/fast指定的是FAST_MODE（默认），i/immediate指定的是IMMEDIATE_MODE，但是这些mode到底是什么用途，需要在其他文章中进行介绍。
- --pgdata -D 这个参数是最关键的参数，指定我们初始化的数据的目录，这个是一个必须的参数
- --options -o 选项列表，可以支持多个这样的参数，最后会把所有的选项用空格连起来
- --silent -s 安静模式，一般指不打印任何的日志
- --timeout -t 等待系统启动完成的时间，默认的时间是60秒
- --core-files -c 允许生成core文件
- --wait -w 命令是否要等到数据库完成启动，然后再退出，如果后台服务没有启动完成，那么就一直等在那里，默认是nowait
- --no-wait -W 同上



# 支持的主要命令操作说明
上面章节的参数属于调控参数，执行的操作要在pg_ctl命令的最后进行指定，主要有如下的几个命令，init,start,stop,reload,restart,kill,promote,接下来的部分就是针对于这些操作进行说明。

## do_init()
命令举例
`pg_ctl -D /mnt/study/pg/data -l /mnt/study/log/logfile init`

调用initdb，然后把自己的命令行指定的参数传递给initdb，比如-D参数参数指定的数据目录

## do_status()

`pg_ctl -D /mnt/study/pg/data -l /mnt/study/log/logfile status`

查看数据目录下的postmaster.pid文件，获取第一行的进程id(pid)，然后用kill(pid, 0)来判断进程是否是alive，如果是的话，那么打印进程还活着，读取postmaster.opts文件，读取每一行（实际上只有一行），打印的是启动postgresql的时候使用的命令行（包括所有的实用的参数）

## do_start()

`pg_ctl -D /mnt/study/pg/data -l /mnt/study/log/logfile start`

首先会检查是否已经有了一个正在用此数据目录来运行的数据库运行实例，检查凡事就是看数据录下是否已经有了postmaster.pid文件，并且里面记录和一个合法的进程id，如果是的话，那么就会认为是可能还有一个数据库实例再运行。但是不管如何，都会尝试启动，后续的启动可能会因为端口号被占用而失败，但是这里实际上并不会让其失败，因为此环节不方便去检查当前实例所使用的端口号是否在系统中已经被使用了，这种方法也是一种平衡，因为没有必要在这里做这么复杂的检查，后续的失败也很快就会在启动的一开始就会发生。

找到postgres这个程序所在的路径，设置好coredump文件的属性，这里只是设置core文件是否允许的设定，自身对coredump是否生成的设定，因为操作系统层面还会有一个设定。

启动一个子进程来执行postgres程序。然后再主进程中监控自己的子进程，这是典型的启动-监控进程的方式。

启动方式分为两种情况，第一是需要输出日志到文件（指定了-l/--log参数的情况），第二是输出到标准输出的情况。

pg_ctl提供的日志重定向到文件的功能，即提供--log/-l 参数，如果不提供此参数，那么就会把所有的日志打印到标准输出。其中日志输出到文件，实用的是追加写入的方式，且当前没有看到日志文件的分片功能。

使用postgres可执行文件启动postgresql的init过程：

postgres <-D 数据目录> <其他参数> < /dev/null   对于最后为何使用一个/dev/null作为postgres的标准输入，我当前不是很清楚。

最后判断是否参数中设置了等待启动结束（指定了--wait/-w参数），如果没有的话，那么直接打印“server starting”，然后退出，否者，等待在那里。

## do_stop()
`pg_ctl -D /mnt/study/pg/data -l /mnt/study/log/logfile stop`
获取postgresq的linit进程的pid，方法是在数据目录下的postmaster.pid文件中找到init进程的进程id。使用kill函数和命令行指定的signal去发送信号给init进程，具体使用哪一种信号，是在命令行的--mode/-m参数中提供的，映射关系请参考do_restart()章节

如果使用了等待的参数（使用了--wait/-w参数），那么会一直的检查postmaster.pid文件，如果这个文件存在，那么就认为postgresql还没有结束，如果postmaster.pid文件不存在了，那么就认为postgresql停掉了。

如果执行命令的时候有backup正在进行（数据目录下有一个back_label文件），那么就会一直等待，等到backup的pg_stop_backup()函数的执行。然后再继续后续的信号发送操作。

## do_kill()

`pg_ctl -D /mnt/study/pg/data -l /mnt/study/log/logfile kill <信号名>`

是kill函数对init进程进行发送signal的操作，要是用的signal也要在命令行提供, 此参数就是跟随者kill操作参数的后面。
可选的信号名有, HUP,INT,QUIT,ABRT,KILL,TERM,USR1,USR2。

## do_restart()

`pg_ctl -D /mnt/study/pg/data -l /mnt/study/log/logfile restart`

实际上整个过程就是kill掉init进程，然后调用do_start()来启动数据库，可以知道的是，所有进程都会经历删除和重新创建的过程。

需要注意的是是哟个kill的时候传入的信号量是不是kill命令传入的那些，可能是一个默认值SIGINT，又或者是指定了--mode/-m参数的时候，指定的模式对应的sig，如果是smart模式，那么sig就是SIGTERM，如果是fast模式，那么sig就是SIGINT，如果是immedia模式，那么sig就是SIGQUIT。

## do_reload()

`pg_ctl -D /mnt/study/pg/data -l /mnt/study/log/logfile reload`

实际上就是给init进程发送了SIGHUP系统信号，和restart不同的是，reload忽略掉--mode/-m参数提供的任何信号相关的信息，而且不会做wait操作，忽略掉--wait和--do_wait参数。

## do_promote()

`pg_ctl -D /mnt/study/pg/data -l /mnt/study/log/logfile promote`

在数据目录下生成一个promote文件（空文件），然后想init进程发送一个SIGUSR1的信号，等待系统的状态变成DB_IN_PRODUCTION（数据来源于pg_control文件）


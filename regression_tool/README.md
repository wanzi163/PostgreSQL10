# 1    概要

PostgreSQL自身拥有一个自动化测试方案，整个自动化的流程是通过Makefile机制来驱动单用例工具执行的。单个用例来讲，PostgreSQL提供了两种工具来规划单例测试流程和机制，一种是测试简单场景，一种是测试需要多个session内的操作相互交叉执行的隔离测试。整个自动化测试采用串行执行的方式，每一个测试用例都会完成，数据初始化，数据库实例启动和关闭，测试用例执行等一套完整的操作。

因为每一次测试都会重复创建和利用相同的数据目录，所以整个自动化测试流程使用的是遇到错误即中断执行的机制，方便开发人员来对当前的环境进行排查。

# 2    单用例工具

## 2.1  概要

PostgreSQL 提供了两种自动化单例测试工具，

- 简单sql执行工具，源码位于`./src/test/regress/pg_regress.c`，编译之后生成的可执行文件为`./src/test/regress/pg_regress`。
- 交叉隔离测试工具：源码位于`./src/test/isolation/isolation_main.c`和`./src/test/isolation/isolationtester.c`，编译之后生成两个文件，一个是pg_isolation_regress，一个是isolationtester，前者负责单例测试流程（包括启动，环境设置，开启测试，结果比对等），后者解析数据文件，实现交叉执行。

 单例执行流程整体来讲分为两大的部分，一个是临时数据库软件安装，一个是用例执行。

## 2.2  临时数据库软件安装

每一次单例执行的时候都会安装一次数据库可执行文件，安装的方式是在调用用例目录下的Makefile文件的install目标，此目标实际上会运行PostgreSQL的源码目录下的make install过程，并且设置--prefix=一个临时目录，临时目录一般就在源码所在目录的tmp_install目录下（此目录每次单例运行都会重复使用）。make的命令为：

```shell
make -C ../..'/用例目录DESTDIR=/mnt/pg/code DESTDIR=/tmp_install install>>'/mnt/pg/code'/tmp_install/log/install.log
```

安装之后的效果：

如果数据库源码位于：/mnt/pg/code，那么临时数据库安装地址为：`/mnt/pg/code/tmp_install/mnt/pg/installation `

```shell
denny@cacu:/mnt/pg/code/tmp_install/mnt/pg/installation$ ls
bin  include  lib  share
```

## 2.3  单例执行工具pg_regress

单例工具的使用方法为（一般情况下的调用方式，使用常用参数）：

```shell
PATH="/mnt/pg/code/tmp_install/mnt/pg/installation/bin:PATH"LD_LIBRARY_PATH="/mnt/pg/code/tmp_install/mnt/pg/installation/lib:LD_LIBRARY_PATH"../../src/test/regress/pg_regress --inputdir=. --temp-instance=./tmp_check--bindir= --encoding=UTF8 --no-locale  --dbname=contrib_regressionunaccent
```

命令前半段是设置环境变量PATH和LD_LIBRARY_PATH到临时数据库安装目录，这样临时目录下的数据可执行文件和库才会被检测到。

工具支持两种执行模式，一种是SINGLE模式，一种是SCHEDULE模式，下面的流程主要是拿SINGLE模式举例。SCHEDULE模式跟SINGLE模式在流程上面是一致的，只是用例组织上面有一些区别，会在下后面章节单独说明。

### 2.3.1 工具执行流程

一般执行流程如下图

```shell
============== creating temporary instance            ==============
============== initializing database system           ==============
============== starting postmaster                    ==============
running on port 57739 with PID 19686
============== creating database "contrib_regression" ==============
CREATE DATABASE
ALTER DATABASE
============== running regression test queries        ==============
test commit_timestamp         ... ok
============== shutting down postmaster               ==============
============== removing temporary instance            ==============

=====================
 All 1 tests passed. 
=====================
```

从屏幕打印的过程可以发现，单例执行流程，可以分为这么几步：创建临时数据实例，初始化数据库，启动数据库测试库，运行用例，停掉数据库，删除掉临时数据库实例，打印测试报告。另外对于错误的情况，不会执行删除临时数据库实例的操作。

1. 创建临时数据库实例

   创建临时数据库数据目录，临时目录是由pg_regress命令的—temp_instance参数指定的。本例中使用了./tmp_instance目录。另外又创建了用来保存日志的目录，默认情况下在当前目录下的log目录。

2. 初始化数据库

   使用initdb -D ./tmp_instance/data –noclean –nosync –debug–no-locale >./log/initdb.log 2>&1 初始化数据文件，商行面得而临时数据文件目录是通过pg_regress的tmp_instance参数指定的。

   另外还在配置文件中增添了几个配置项如下：

   ```shell
   log_autovacuum_min_duration = 0
   log_checkpoints = on
   log_lock_waits = on
   log_temp_files = 128kB
   max_prepared_transactions = 2
   ```

   由于此过程一般情况下执行速度比较快，过程简单，基本上不会出现问题。所以此过成在当前进程中使用阻塞式的system系统调用来执行。

3. 启动数据库测试库

   通过上面初始化步骤来初始化的数据库数据文件，以此来启动数据库进程实例，使用的命令是postgres -D ./tmp_instance/data -F -c “listen_address=” -k >./log/postmaster.log 2>&1

   由于此过程涉及数据库启动，过程比较复杂，且容易由于配置导致启动失败或者超时，所以另起了独立异步进程来运行，并在父进程中对数据库子进程进行监控，设置超时时间，使用psql进行连接试探，超时时间为60秒。psql试探失败之后有两种可能，一种是发现数据库进程消失，那么就会立即失败。如果数据库进程还在，那么就会暂停一小段时间之后继续试探，直到超时。超时之后，如果数据库还是无法连接，那么就会kill掉数据库进程。一旦psql试探成功，立即往下执行，标记数据库启动成功。

   需要注意的是临时数据库启动所使用的端口是在(49152-65535)之间，具体使用哪一个是由数据库版本决定的。当然也可以在工具的命令行里面使用—port参数来指定端口。在用例执行屏幕输出中会输出当前正在使用的端口号：`running on port 57739 with PID 19686`

4. 启动数据库测试

   按照之前说明，启动测试的方式有两种，一种是在命令行直接指定要测试的用例（可以指定多个用例名称），另一种就是使用schedule数据文件（稍后章节会对schedule方式做额外说明），不管用哪种用例组织模式，对于一个用例本身，执行过程都是一样的，下面只讨论单个用例的执行情况。

   执行过程需要准备好三个关键数据：

   ```shell
   输入sql文件：./sql/用例名称.sql
   输出out文件:./results/用例名称.out
   结果比对文件：./expected/用例名称.out
   ```

   执行过程比较简单，即使用psql -X -a -q -d 数据库名 <sql输入文件 > ./results/用例名.out，作用是把sql文件中的命令输入到数据库中，执行结果会打印到屏幕上面，并且会重定向到输出文件（未来做结果比对的文件）。此处同样使用独立异步进程的方式运行，原因是如果命令行中如果指定多个测试实例，那么这么多个测试实例都可以并行运行，主进程会监控并且等待所有的测试实例进程结束，然后进行后续的操作。

   跟数据库的交互操作结束后，会把之前实例运行所得到的屏幕数据文件，跟之前保存的期待的结果数据文件进行比对。实际上使用的是linux自带的diff命令来完成两个文件的差值比对。这里支持了多版本预期结果数据的功能，即版本化预期结果文件，使用文件名加上.[index]后缀，比如用例名称.out.1，用例名称.out.2，一直最大支持index到9，如果和默认的预期接过文件比较发现不一致，那么会继续比较这些不同版本的预期接过文件，找出差别最小的一个来，作为最终的比较差异结果展示。

   这种预期接过文件的版本化管理正是应对不同pg版本下的自动化测试的一种方案。

5. 停掉数据库

   运行pg_ctl stop -D ./tmp_instance -s -m fast命令来快速把临时的postgresql实例停掉。此过程使用阻塞式的system系统调用来完成。

6. 删除掉临时数据库实例

   使用rmtree系统调用来整个其清除掉临时数据库数据文件的目录。

7. 打印测试报告

   如果都成功了，打印All %dtests passed.

   如果只有要ignore的用例失败了，那么打印%d of %dtests passed, %d failed test(s) ignored. 

   如果只有不能ignore的用例失败了，打印%d of %d tests failed.

   如果上面两种类型的失败用例都有，那么打印%d of %d tests failed, %d of thesefailures ignored.

   如果有错误发生（就是数据结果文件比对发现有差别）指出差别的diff文件和数据库运行的日志文件的位置。

   `上面ignore的数据是在schedule文件中存在的一种数据配置，命令行指定用例的时候不会遇到`

在程序执行完成要退出的时候，判断是否有失败的用例，如果有，那么命令返回给调用者的返回值就是1，如果全部都成功了，那么就会返回0,这个操作也是非常关键的一个处理，方便调用者对执行结果做一个最直接和简单的判断，然后执行调用者后续的相关逻辑。

 以上说明的是一个成功例子的情况，下面就是一个失败的例子，注意最下面会打印出失败用例的diff文件和日志文件，方便开发人员进行问题排查。

```shell
============== creating temporary instance            ==============
============== initializing database system           ==============
============== starting postmaster                    ==============
running on port 57739 with PID 22290
============== creating database "contrib_regression" ==============
CREATE DATABASE
ALTER DATABASE
============== running regression test queries        ==============
test commit_timestamp         ... FAILED
============== shutting down postmaster               ==============

======================
 1 of 1 tests failed. 
======================

The differences that caused some tests to fail can be viewed in the
file "/mnt/pg/code/src/test/modules/commit_ts/regression.diffs".  A copy of the test summary that you see
above is saved in the file "/mnt/pg/code/src/test/modules/commit_ts/regression.out".

../../../../src/makefiles/pgxs.mk:280: recipe for target 'check' failed
make: *** [check] Error 1
```



### 2.3.2 用例的schedule文件和并发执行

在命令行里面指定了—schedule参数，那么工具就会进入SCHEDULE模式，此模式需要用读取—schedule指定的数据文件。文件格式如下：

```shell
# ----------
# Another group of parallel tests
# ----------
test: create_aggregate create_function_3 create_cast constraints triggers inherit create_table_like typed_table vacuum drop_if_exists updatable_views rolenames roleattributes

# ----------
# sanity_check does a vacuum, affecting the sort order of SELECT *
# results. So it should not run parallel to other tests.
# ----------
test: sanity_check

# ----------
# Believe it or not, select creates a table, subsequent
# tests need.
# ----------
test: errors
test: select
ignore: random

# ----------
# Another group of parallel tests
# ----------
test: select_into select_distinct select_distinct_on select_implicit select_having subselect union case join aggregates transactions random portals arrays btree_index hash_index update namespace prepared_xacts delete
```

数据文件中拥有两种标签，test和ignore，但是实现了三种效果，单独执行，并行执行，忽略执行。

- test：需要并行执行的用例集合，程序限制最多并行的用例数是100个。
- ignore：需要忽略不执行的用例。

从上图可以看出，所有不用并发执行的用例都单独放在了一个test标签中，所有可以并行执行的用例都根据类型分组放到了相应的test标签中。

程序扫描数据文件，不是整个文件分析完成之后再执行用例，而是逐行解析的时候，遇到test一行，就会执行当前一行指定的用例集合。

并发执行集合中的所有用例，采用每一个用例创建一个独立进程的方式，每一个进程中按照SINGLE模式的方式来执行psql命令，读取相应的sql输入，并且截获屏幕输出到相应的输出文件。连续创建多个独立进程之后，调用wait系统调用，统一处理所有进程的状态，等待所有用例结束后，为每一个用例做结果比对，并且在屏幕打印用例测试结果。

### 2.3.3 设计思想亮点

- 调用PostgreSQL的管理命令psql来执行命令，这样做简单高效，不用开发使用libpq的代码，维护各种状态。
- 使用psql可以支持更多的除了sql语句之外的命令，比如vacuum等管理命令。
- 启用独立进程来调用psql，在主进程控制子进程的方式能够提供更好的体验。
- 支持并发执行用例，并行和串行混合执行（SCHEDULE模式）
- 结果比对方式比较简单，直接调用系统的diff命令。
- 支持多版本的预估数据，比对的时候自动选择差异最小的版本。

## 2.4  单例执行工具pg_isolation_regress

相对于pg_regress工具，pg_isolation_regress是用来做多个session之间的数据隔离性的测试，多个session之间不同的语句操作相同的资源的时候，会有一些机制来控制数据的可见范围和操作方式，即给定几个session中都有各自的执行语句序列，那么需要穿插所有session中的步骤来判断数据是否是按照PostgreSQL设计的数据隔离可见性来运行的。pg_isolation_regress内部实用的是一个isolationtest的工具（pg_regress用的是psql），这个工具是通过工具编译目录下的一个isolationtest.c源码编译而来，此程序用来解析一种类型的用例格式，并且连接到数据库，发送用例中提供的sql语句。isolationtest的特点就是他提供了多个session并行处理的机制（配置文件中可以配置多个session），并且支持环境的预制。

```shell
PATH="/mnt/pg/code/tmp_install/mnt/pg/installation/bin:$PATH"LD_LIBRARY_PATH="/mnt/pg/code/tmp_install/mnt/pg/installation/lib:$LD_LIBRARY_PATH"./pg_isolation_regress --temp-instance=./tmp_check --inputdir=. --bindir=  --schedule=./isolation_schedul
```

### 2.4.1 数据文件的格式（spec后缀文件）

```shell
setup
{
  CREATE TABLE foo (a int PRIMARY KEY, b text);
  CREATE TABLE bar (a int NOT NULL REFERENCES foo);
  INSERT INTO foo VALUES (42);
}

teardown
{
  DROP TABLE foo, bar;
}

session "s1"
setup           { BEGIN; }
step "ins"      { INSERT INTO bar VALUES (42); }
step "com"      { COMMIT; }

session "s2"
step "upd"      { UPDATE foo SET b = 'Hello World'; 
```

整个文件有三种章节组成，SETUP，TEARDOWN， SESSION，PERMUTATION.

- SETUP：在所有的session开始之前的全局层面运行的SQL语句，通常是用来准备预制数据的。
- TERDOWN：在所有的session执行结束之后执行的SQL语句，通常是用来清理预制数据的。
- SESSION：用例的SQL语句实体，内部也包含一个setup语句，在每一个session的执行环境的开始来执行。
- PERMUTATION：定义本次自动化分批次执行次序。每一个PERMUTATION中的步骤可以来自于不同的SESSION，指定这些操作的交叉执行次序，从而得到想要测试的多session之间的语句隔离场景。不指定PERMUTATION的时候，工具会根据提供的session中的所有步骤，穷尽所有交叉次序的可能，每一种可能的次序都会自动生成一个隐含的PERMUTATION。

### 2.4.2 工具执行流程

程序会使用语法分析器分析整个spec文档， 构建出SETUP，TEARDOWN，SESSION，PERMUTATION四个主要的数据结构。然后根据spec中的session数量，预先创建相应的数据库连接数（一个session对应一个连接），实际上创建的连接数要比session数量大1，因为多出来的一个链接用来做管理命令和预制数据得的处理（个人认为这个设计也是比较不错）。

接下来就是把连接指定给session，并且把session内部的每一步都指定了所要执行的连接上面。轮询所有的step，在其指定的链接上面开始运行step对应的sql语句。

轮询PERMUTATION配置批次，找到每一个批次中要进行执行的步骤序列（每个步骤可能来自于不同的session，这样达到交叉执行的效果），如果不进行指定PERMUTATION的配置信息的话，那么工具默认会学习出一堆批次，这些批次来穷尽所有步骤实现交叉的可能性，通常两个session中个自有三个步骤的话，那么会穷尽出20个批次（要保证步骤在各自的session内部还是传行的效果）。

```shell
# SESSION(“s1”) 拥有的step序列为：rx1, wy1, c1
# SESSION(“s2”) 拥有的step序列为：ry2, wx2, c2
# 那么会自动产生出如下的序列批次
starting permutation: rx1 wy1 c1 ry2 wx2 c2
starting permutation: rx1 wy1 ry2 c1 wx2 c2
starting permutation: rx1 wy1 ry2 wx2 c1 c2
starting permutation: rx1 wy1 ry2 wx2 c2 c1
starting permutation: rx1 ry2 wy1 c1 wx2 c2
starting permutation: rx1 ry2 wy1 wx2 c1 c2
starting permutation: rx1 ry2 wy1 wx2 c2 c1
starting permutation: rx1 ry2 wx2 wy1 c1 c2
starting permutation: rx1 ry2 wx2 wy1 c2 c1
starting permutation: rx1 ry2 wx2 c2 wy1 c1
starting permutation: ry2 rx1 wy1 c1 wx2 c2
starting permutation: ry2 rx1 wy1 wx2 c1 c2
starting permutation: ry2 rx1 wy1 wx2 c2 c1
starting permutation: ry2 rx1 wx2 wy1 c1 c2
starting permutation: ry2 rx1 wx2 wy1 c2 c1
starting permutation: ry2 rx1 wx2 c2 wy1 c1
starting permutation: ry2 wx2 rx1 wy1 c1 c2
starting permutation: ry2 wx2 rx1 wy1 c2 c1
starting permutation: ry2 wx2 rx1 c2 wy1 c1
starting permutation: ry2 wx2 c2 rx1 wy1 c1
```

由于程序只有主进程一个线程，所以需要充分考虑被某一个step的操作block的情况，从而阻塞了所有的后续执行。在发送一个step的sql命令之后，工具会就此等待数据库服务器的回复消息，判断socket里面的是否有回复的消息，如果有消息了，那么就会读取出来，然后继续查看是否消息已经全部读取完毕，如果还没有（比如数据库还是busy状态），那么就会再次继续等待，可能会出现长时间无法等待到消息的情况，那么可能会被堵塞在这个地方，工具不会让这个阻塞永远的hang住，所以设置了一个超时时间为10秒，然后打印`step xx_stepName_xx： xx_sql__xx <waiting…>`。超时之后直接执行下一个排序的step。这里会存在一个问题，如果下一个step也是这个要给这个已经拥塞的session发送的命令，那么肯定还是会block，从代码注释来看，通常这个是不会发生的，因为一个session被block，是应该由其他的session的操作来解锁的，但是从实际来看，step设计失误，数据库自身出现问题等原因导致这个问题还是很有可能的。所以代码针对于此，认为是此测试整体失败，进行了全面的清理工作，发送cancel当前正在执行的任务的命令，最后调用定义的TEARDOWN操作。考虑已经发生上一个session被block的情况，下一条来自另外session的语句会是设计来解锁的操作，如果没有起到解锁的作用，直接再次尝试下一个操作，并且尝试是否解锁，依次下去，除非解锁成功，否者不应该再有来自于被block的session的step要执行，这是工具设计的要求。工具的另外一个设计局限就是虽然可以支持多余两个session，但是只能支持一个session被block的情况，即不能设计和承受多余一个session同时被锁的场景，否者工具会被永远的block。

当前批次中所有step都执行完成后，要检查是否还有未被解锁的step，如果还有，那么会等待10秒，如果最后还是没有解锁，那么只能宣告测试失败。

因为每一次检测block的session都要等满了10秒，执行过程中发生block的时候，用例执行的时间还是很长的（特别是意外block 的情况下，会执行多次block检测）。

批次执行的最后一步就是要teardown，首先执行所有session里面的teardown操作，然后执行全局的teardown操作。需要注意的是，每一个批次都要执行完整的setup和teardown操作，包括全局的和每个session各自的部分。所以这个teardown操作一定要充分的清除掉在setup环节创建的资源，以致不能让下一次setup的时候发现资源的冲突导致失败。

所有批次都串行执行完成后，整个单例测试宣告结束。

### 2.4.3 设计思想亮点

- 单个用例中引入了多个session的处理，使用libpq来管理多个session的状态。pg_regress工具只是用psql来进行处理，实际上都是一个session里面的操作。
- 更细致的控制每一个session中每一个步骤的执行，还包括每一个session内部的初始化，清理等细节的控制。
- 能够测试两个或者多个session内不同步骤之间的交叉执行状况。

- 能够自动穷尽所有给懂的多个session内各个步骤的交叉执行状态，覆盖度达到100%

- 能够测试session被另一个session阻塞，并且被后续语句解锁的场景的复杂用例。

### 2.4.4 批次的穷尽算法

函数名为run_all_permutations_recurse，通过使用两个外部的辅助数据来控制，辅助数据控制的是每一个session内已经固定了多少个step在结果中（实际上就是有前几个step已经放在排序结果中了）。整个函数采用了比较难懂的递归加循环的算法，不过算法的中心思想就是：按照被定义的顺序，每一次递归单元都是想尝试固定前一个session中的一部分step，然后再对后面的进行穷尽操作，直到每一个step都被固定过一次。举一个例子：

`session1 中有step：rx1 wy1 c1， session2中有step：ry2 wx2 c2`

那么首先固定所有session1里面的所有step（那么session2中的所有step肯定就是直接写在后面了）得到一种可能：rx1, wy1, c1  ry2, wx2, c2, 然后在选择固定rx1, wy1, 然后让c1跟session2中所有step进行一次穷尽操作，即固定ry2在前面一次，固定ry2和wx2一次，ry2,wx2,c2一次，一共有三种可能。以此类推下去下一次应该是固定rx1在前面，然后让wy1和c1一起跟session2的所有step来一次穷尽。

最后穷尽的结果是：

```shell
rx1 wy1 c1 ry2 wx2 c2
rx1 wy1 ry2 c1 wx2 c2
rx1 wy1 ry2 wx2 c1 c2
rx1 wy1 ry2 wx2 c2 c1
rx1 ry2 wy1 c1 wx2 c2
rx1 ry2 wy1 wx2 c1 c2
rx1 ry2 wy1 wx2 c2 c1
rx1 ry2 wx2 wy1 c1 c2
rx1 ry2 wx2 wy1 c2 c1
rx1 ry2 wx2 c2 wy1 c1
ry2 rx1 wy1 c1 wx2 c2
ry2 rx1 wy1 wx2 c1 c2
ry2 rx1 wy1 wx2 c2 c1
ry2 rx1 wx2 wy1 c1 c2
ry2 rx1 wx2 wy1 c2 c1
ry2 rx1 wx2 c2 wy1 c1
ry2 wx2 rx1 wy1 c1 c2
ry2 wx2 rx1 wy1 c2 c1
ry2 wx2 rx1 c2 wy1 c1
ry2 wx2 c2 rx1 wy1 c1 
```

### 2.4.5 检测backend是否被block的方式

向管理session发送prepare命令，携带backend号。

此检测只是发生在等待回复消息超时的情况下。目的是检测当前的超时是由于数据库服务整体出了问题，还是只是当前某一个session由于被block导致的超时。因为工具在一开始就多申请了一个数据库的连接用来做管理进程，所以在一个链接中发现超时的时候，向这个管理连接发送消息，看看是否会获得回复，如果有回复，那么说明数据库服务还在，是由于当前所操作的这个session被某种锁block了。

向管理连接发送的消息为：

```c
PQexecPrepared(conns[0], “isolationtester_waiting”, 1, 
               &backend_pids[step->session + 1], NULL, NULL, 0);
```

因为在一开始工具初始化的时候已经做了一个针对于isolationtester_waiting的prepare，

实际上执行的对应的sql语句是：

```sql
SELECT 1FROM pg_locks holder, pg_locks waiter 
WHERE NOT waiter.granted ANDwaiter.pid = $1 
AND holder.granted
AND holder.pid <> $1 AND holder.pid IN (");
```

传入的参数是后台处理相关session的进程id，查看这个处理进程是否被锁。如果被锁，那么就会返回相应的记录回来。

# 3    全用例自动化流程

## 3.1  概要

使用Makefile的机制驱动全流程自动化。包括自动化用例范围，每一个用例的衔接，单用例中临时数据库软件的安装和后续的用例执行。

## 3.2  Makefile全流程驱动

```makefile
define _create_recursive_target
.PHONY: $(1)-$(2)-recurse
$(1): $(1)-$(2)-recurse
$(1)-$(2)-recurse: $(if $(filter check, $(3)), temp-install)
        $$(MAKE) -C $(2) $(3)
endef
```

```makefile
recurse = $(foreach target,$(if $1,$1,$(standard_targets)),$(foreach subdir,$(if $2,$2,$(SUBDIRS)),$(eval $(call _create_recursive_target,$(target),$(subdir),$(if $3,$3,$(target))))))
```

```makefile
SUBDIRS = perl regress isolation modules
```

```makefile
recurse_alldirs_targets := $(filter-out installcheck install, $(standard_targets))
installable_dirs := $(filter-out modules, $(SUBDIRS))

$(call recurse,$(recurse_alldirs_targets))
$(call recurse,installcheck, $(installable_dirs))
$(call recurse,install, $(installable_dirs))
```

上面的代码片段来自于几个文件，但是最终都被include到了一起，通过几个层次的嵌套会学习出如下的我们会使用到的一个规则:

```makefile
# 对每一个子目录都会有一个类似的规则，只是目录名不同而已
.PHONY check-<目录名>-recurse
check: check-<目录名>-recurse
check-**目录名-recurse: temp-install
    $$(MAKE) -C <目录名> check
```

在./src/test目录下运行make check，就会实际上运行几个子目录下的make check

## 3.3  模块和使用的单例工具对照

如下两个模块使用了pg_isolation_regress的工具

`isolation`, `modules/brin`

如下几个模块使用了pg_regress的工具

`regress`, `modules/commit_ts`, `modules/dummy_seclabel`,`modules/test_ddl_deparse`, `test_parser`, `test_rls_hooks`, `test_shm_mq`

在上面各自的目录下的Makefile中，在check这个编译目标中，都明文的调用相应的单例工具。


# 索引

[toc]

## 索引概念

* 覆盖索引
* 前缀索引
* 索引下推

## 锁分类

* ### 全局锁

  > 应用范围： 逻辑备份，
  >
  > 全局读锁添加方式（innoDB建议使用b加锁方式):
  >
  > 1. Flush tables with read lock (FTWRL)
  >
  >    命令行的方式，添加全局读锁，缺点无法进行更新相关操作
  >
  > 2. mysqldump 使用参数–single-transaction 的时候，导数据之前就会启动一个事务，来确保拿到一致性视图。而由于 MVCC 的支持， 这个过程中数据是可以正常更新的。
  >
  > 3. **set global readonly=true** (可能被当做主从判断， 异常处理机制上不会释放)

* ### 表级锁

  > 分类: 
  >
  > 1. 表锁
  >    语法： **lock tables ... read/write**
  >
  > 2. 元数据锁 （**MDL** 、**metadata lock**）
  >
  >    修改表字段，导致数据库崩溃，因为加锁（MDL）后其他读写都在阻塞，重试
  >
  >    解决办法： 如果在这个指定的等待时间里面能够拿到 MDL 写锁最好，拿不到 也不要阻塞后面的业务语句
  >
  >    MariaDB 已经合并了 AliSQL 的这个功能，所以这两个开源分支目前都支持 DDL NOWAIT/WAIT n 这个语法。
  >
  >    1 ALTER TABLE tbl_name NOWAIT add column ... 
  >
  >    2 ALTER TABLE tbl_name WAIT N add column ...

* ### 行级锁

  > 两个事务，执行导致死锁解决方案:
  >
  > 1. 设置innodb_lock_wait_timeout ，默认50s 等待超时
  > 2. 设置 innodb_deadlock_detect ， 主动死锁检测

## 前缀索引

### 		怎么给字符串字段加索引?

> 通过区分度，选择合适长度添加索引

###         区分度不高，或者区分高索引字段过长怎么办?

> 比如身份证号码， ，一共 18 位，其中前 6 位是地址码，所以同一个县的人的身份 证号前 6 位一般会是相同的。按照我们前面说的方法，可能你需要创建长度为 12 以上的前缀索引，才能够满足区分度要 求。
>
> * 倒序存储
>   select field_list from t where id_card = reverse('input_id_card_string');
>
> * 增加hash字段
>
>   每次插入新记录的时候，都同时用 crc32() 这个函数得到校验码填到这个新字段。由于 校验码可能存在冲突，也就是说两个不同的身份证号通过 crc32() 函数得到的结果可能是相同的，所以你的查询语句 where 部分要判断 id_card 的值是否精确相同。
>
>    alter table t add id_card_crc int unsigned, add index(id_card_crc);
>
>   select field_list from t where id_card_crc=crc32('input_id_card_string') and id_card =  ''

### 		 前缀索引优劣

> 会引起覆盖索引失效， 因为前缀索引无法直接使用当前字段，需要通过主键索引查询相关字段

1. 直接创建完整索引，这样可能比较占用空间;

2. 创建前缀索引，节省空间，但会增加查询扫描次数，并且不能使用覆盖索引;

3. 倒序存储，再创建前缀索引，用于绕过字符串本身前缀的区分度不够的问题;

4. 创建 hash 字段索引，查询性能稳定，有额外的存储和计算消耗，跟第三种方式一样，

   都不支持范围扫描。

## 需要频繁统计count(*)怎么做？

* 使用Redis缓存计数 （导致计数，和数据不一致)
* 使用计数表，通过事务控制一致性

## 幻读

* 在可重复读隔离级别下，普通的查询是**快照读**，是不会看到别的事务插入的数据的。因 此，幻读在“当前读”下才会出现。
* 幻读仅专指“新插入的行”。修改的记录被看到不算“幻读”

## MySQL是如何保证主备一致的

### 主备基本原理

> 1. 在备库 B 上通过 change master 命令，设置主库 A 的 IP、端口、用户名、密码，以及 要从哪个位置开始请求 binlog，这个位置包含文件名和日志偏移量。
> 2. 在备库 B 上执行 start slave 命令，这时候备库会启动两个线程，就是图中的 io_thread 和 sql_thread。其中 io_thread 负责与主库建立连接。
> 3. 主库 A 校验完用户名、密码后，开始按照备库 B 传过来的位置，从本地读取 binlog， 发给 B。
> 4. 备库 B 拿到 binlog 后，写到本地文件，称为中转日志(relay log)。
> 5. sql_thread 读取中转日志，解析出日志里的命令，并执行。

### Binlog三种格式

* statement (默认)

  > 每一条会修改数据的sql都会记录到master的bin-log中。slave在复制的时候sql进程会解析成和原来master端执行过的相同的sql来再次执行
  >
  > 优点：statement level下的优点首先就是解决了row level下的缺点，不需要记录每一行数据的变化，减少bin-log日志量，节约IO，提高性能，因为它只需要在Master上锁执行的语句的细节，以及执行语句的上下文的信息。
  >
  > 缺点：由于只记录语句，所以，在statement level下 已经发现了有不少情况会造成MySQL的复制出现问题，主要是修改数据的时候使用了某些定的函数或者功能的时候会出现。

* row 行模式

  > 日志中会记录每一行数据被修改的形式，然后在slave端再对相同的数据进行修改
  >
  > 优点：在row level模式下，bin-log中可以不记录执行的sql语句的上下文相关的信息，仅仅只需要记录那一条被修改。所以rowlevel的日志内容会非常清楚的记录下每一行数据修改的细节。不会出现某些特定的情况下的存储过程或function，以及trigger的调用和触发无法被正确复制的问题
  >
  > 缺点：row level，所有的执行的语句当记录到日志中的时候，都将以每行记录的修改来记录，会产生大量的日志内容。

* Mixed 

  > 因为有些 statement 格式的 binlog 可能会导致主备不一致，所以要使用 row 格式。
  >
  > 但 row 格式的缺点是，很占空间。比如你用一个 delete 语句删掉 10 万行数据，用 statement 的话就是一个 SQL 语句被记录到 binlog 中，占用几十个字节的空间。但如 果用 row 格式的 binlog，就要把这 10 万条记录都写到 binlog 中。这样做，不仅会占 用更大的空间，同时写 binlog 也要耗费 IO 资源，影响执行速度。
  >
  > 所以，MySQL 就取了个折中方案，也就是有了 mixed 格式的 binlog。mixed 格式的意 思是，MySQL 自己会判断这条 SQL 语句是否可能引起主备不一致，如果有可能，就用 row 格式，否则就用 statement 格式。

## Binlog写入机制

> binlog 的写入逻辑比较简单:事务执行过程中，先把日志写到 binlog cache，事务 提交的时候，再把 binlog cache 写到 binlog 文件中。
> 一个事务的 binlog 是不能被拆开的，因此不论这个事务多大，也要确保一次性写入。这就 涉及到了 binlog cache 的保存问题。
> 系统给 binlog cache 分配了一片内存，每个线程一个，参数 binlog_cache_size 用于控制 单个线程内 binlog cache 所占内存的大小。如果超过了这个参数规定的大小，就要暂存到 磁盘。
> 事务提交的时候，执行器把 binlog cache 里的完整事务写入到 binlog 中，并清空 binlog cache。状态如图 1 所示。
> 可以看到，每个线程有自己 binlog cache，但是共用同一份 binlog 文件。
> 图中的 write，指的就是指把日志写入到文件系统的 page cache，并没有把数据持久化 到磁盘，所以速度比较快。
>
> 图中的 fsync，才是将数据持久化到磁盘的操作。一般情况下，我们认为 fsync 才占磁盘 的 IOPS。
> write 和 fsync 的时机，是由参数 sync_binlog 控制的:
>
> 1. sync_binlog=0 的时候，表示每次提交事务都只 write，不 fsync;
>
> 2. sync_binlog=1 的时候，表示每次提交事务都会执行 fsync;
>
> 3. sync_binlog=N(N>1) 的时候，表示每次提交事务都 write，但累积 N 个事务后才
>
>    fsync。
>
> 因此，在出现 IO 瓶颈的场景里，将 sync_binlog 设置成一个比较大的值，可以提升性能。 在实际的业务场景中，考虑到丢失日志量的可控性，一般不建议将这个参数设成 0，比较常 见的是将其设置为 100~1000 中的某个数值。
>
> 但是，将 sync_binlog 设置为 N，对应的风险是:如果主机发生异常重启，会丢失最近 N 个事务的 binlog 日志。

![](https://tva1.sinaimg.cn/large/007S8ZIlgy1gijhqxj52qj30vq0ks0xd.jpg)

## Mysql是如何保证高可用的?

* **可靠性优先策略**

  

* **可用性优先策略**

## Mysql读写分离过期读解决方案

* **强制走主库方案**

* Sleep方案

* **判断主备无延迟方案**

  ![](https://tva1.sinaimg.cn/large/007S8ZIlgy1gikmq3oov0j30w20agwz8.jpg)

  > 1. show slave status 结果里的 seconds_behind_master 参数的值，可以用来衡量主备延迟时间的长短。
  >
  > 所以**第一种确保主备无延迟的方法是，**每次从库执行查询请求前，先判断 seconds_behind_master 是否已经等于 0。如果还不等于 0 ，那就必须等到这个参数变为 0 才能执行查询请求。
  >
  > 2. 对比位点确保主备无延迟
  >
  >    Master_Log_File 和 Read_Master_Log_Pos，表示的是读到的主库的最新位点; Relay_Master_Log_File 和 Exec_Master_Log_Pos，表示的是备库执行的最新位点。
  >
  > 3. 对比 GTID 集合确保主备无延迟
  >    Auto_Position=1 ，表示这对主备关系使用了 GTID 协议。 
  >    Retrieved_Gtid_Set，是备库收到的所有日志的 GTID 集合; 
  >    Executed_Gtid_Set，是备库所有已经执行完成的 GTID 集合。
  >
  >    如果这两个集合相同，也表示备库接收到的日志都已经同步完成。


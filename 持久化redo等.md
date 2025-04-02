## 1.redo log 概述

redo log 是 innodb 的持久化日志，使用的是 WAL（write-Ahead Logging）的机制，即将数据写入磁盘前先记录修改操作日志，redo log 记录的是数据页的修改。用于崩溃恢复，并通过顺序写和组提交来提高写入性能。

## 2.日志文件形式

redolog 是循环写的日志，多个日志文件可以看做头尾相连的一个整体，mysql 用 checkpoint 和 write pos 来记录当前写入的位置和已经将数据刷盘的位置，ib_logfile 就是 redolog 日志文件
![图 1](images/2025-04-02-6b5a9ffa47081d965c3a1d2ddc24a90e2606cf6aaa4354516f4099a26d414171.png)

## 3.写入流程

### 3.1总体写入流程

由于 mysql 同时还拥有 binlog 日志，所以 redo log 日志跟 binlog 日志还需要保持一致，mysql 使用两阶段提交来保证一致性。具体如下：

![图 2](images/2025-04-02-52c98ed282bdf3727762569247ea93ca441bd454f4862c8453ef15bec66d96b0.png)

如果在写入 binlog 前崩溃，事务视为位提交，redo log 丢弃 prepare 的数据，如果在 binlog 写入后提交，mysql 会恢复时会将 prepare 的日志修改为 commit，以保证一致性。

> **XID**：从崩溃恢复流程来看，redo log 和 bin log 需要建立某种关联。 不然无法找到同一个事务对应的日志。这关联就是 XID。<br>
> mysql 在内存中维护变量 global_query_id，每次执行语句将这个赋值给 Query_id，如果这个语句是事务的第一条语句，那么同时会把 Query_id 赋值给 Xid。

### 3.2详细写入流程
#### 3.2.1 基础概念
**redo log buffer**：用来缓存redo log日志，一个事物可能有多次数据修改，每次修改都要生成日志，这时日志不会直接写盘，而是保存在缓存中。等待提交或者其他时机刷到磁盘

**write**：将数据写入到操作系统的page cache中

**fsync**：从page cache写入到磁盘的操作

**innodb_flush_log_at_trx_commit**：0表示事物提交时只把redo log留在redo log buffer中；1表示刷到磁盘中，2表示刷到page cache

**刷盘时机**：
>1、后台线程每秒一次将redo log buffer刷到磁盘，没提交的也刷
>2、redo log buffer占用空间到达innodb_log_buffer_size一半时（write操作）
>3、其他事务提交的时候顺带将这个事物的redo log buffer持久化到磁盘

#### 3.2.2 实际刷盘流程

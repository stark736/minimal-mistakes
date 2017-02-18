---
title: PostgreSQL 性能优化之 synchronous_commit
date: '2015-09-20 15:18:26'
---

上周在排查性能问题时, 我们小组发现PostgreSQL在执行UPDATE/INSERT操作需要花费的时间远远超过预期. 初步怀疑是阿里云的PostgreSQL服务不成熟导致. 于是, 使用内网搭建的PostgreSQL服务进行了测试, 结果如下:

创建测试表tb_emp1

```sql
CREATE TABLE tb_emp1
(
    id Serial PRIMARY KEY,
    name VARCHAR(25),
    data JSON
);
```

只插入4条测试数据

```sql
INSERT INTO tb_emp1 (name, data) VALUES
('test1', '[100,101]'),
('test2', '[100,101,500]'),
('test3', '[200,50,1]'),
('test4', '[200,50,3]');
```

测试UPDATE操作

```sql
UPDATE tb_emp1 SET data = '[1,3,4]' WHERE name = 'test3';
UPDATE 1
Time: 9.059 ms
```

重复测试

```sql
UPDATE tb_emp1 SET data = '[1,3,4]' WHERE name = 'test3';
UPDATE 1
Time: 13.496 ms
```

以上结果表明, 阿里云的稳定性问题是错误的推断, 内网同样存在问题. 于是继续探寻真相, 由于是UPDATE操作的执行效率, 我们决定在PostgreSQL的配置项中搜寻与写操作相关的配置项. 发现了`fsync`这个配置项, 根据官方文档的说法:

> If this parameter is on, the PostgreSQL server will try to make sure that updates are physically written to disk, by issuing fsync() system calls or various equivalent methods (see wal\_sync\_method). This ensures that the database cluster can recover to a consistent state after an operating system or hardware crash.

> While turning off fsync is often a performance benefit, this can result in unrecoverable data corruption in the event of a power failure or system crash. Thus it is only advisable to turn off fsync if you can easily recreate your entire database from external data.

`fsync`参数值为`on`时, 更新操作会被同步地写入磁盘, 在系统或者硬件崩溃后, 保持集群中的数据一致性. 设置该参数值为`off`, 可以提高性能, 但相应会产生了数据一致性的风险. 看来, 这不是解决这个问题的最佳方案.

于是, 我们继续寻找. 这时, `synchronous_commit`进入了我们的眼帘, 以下是该参数的官方描述:

> Specifies whether transaction commit will wait for WAL records to be written to disk before the command returns a "success" indication to the client. Valid values are on, local, and off. The default, and safe, value is on. When off, there can be a delay between when success is reported to the client and when the transaction is really guaranteed to be safe against a server crash. (The maximum delay is three times wal\_writer\_delay.) **Unlike fsync, setting this parameter to off does not create any risk of database inconsistency: an operating system or database crash might result in some recent allegedly-committed transactions being lost, but the database state will be just the same as if those transactions had been aborted cleanly.** So, turning synchronous_commit off can be a useful alternative when performance is more important than exact certainty about the durability of a transaction. For more discussion see Section 29.3.

`synchronous_commit`参数表示是否在等待记录被写入磁盘后才向客户端发送成功的响应, 该参数默认值为`on`. 设置该参数为`off`意味着更新操作会立刻返回成功响应, 而没有直接被写入磁盘. 当系统或者硬件崩溃时, 会丢失一段时间的数据, 该时间由`wal_writer_delay`参数决定. 然而, 与`fsync`不同, 该参数不会产生数据一致性的风险. 设置该参数值为`off`能提高写操作的响应速度.

于是, 我们决定进行内网测试:

关闭该参数后的UPDAE操作

```sql
UPDATE tb_emp1 SET data = '[1,3,4]' WHERE name = 'test3';
UPDATE 1
Time: 0.362 ms
```

再次验证

```sql
UPDATE tb_emp1 SET data = '[1,3,4]' WHERE name = 'test3';
UPDATE 1
Time: 0.382 ms
```

结果可以看出性能提升明显. 同样, 该方案并非完美, 但问题在特定的场景下, 应该使用符合现实的解决方案. 综合评估后, 我们认为当前环境下, 性能上带来的提升的价值高于宕机时短暂时间内的数据丢失所带来的隐患.

so just do it.
# MySQL

Created by: Guo YuJie
Created time: October 28, 2023 12:31 PM
Tags:  Document

/opt/homebrew/opt/mysql/bin/mysqld_safe --datadir\=/opt/homebrew/var/mysql

# 👀 一条语句的执行过程

> update t set b = 200 where id = 2
> 
1. 客户端 发出 **更新语句，**并向 MySQL 服务端建立连接
2. MySQL 连接器负责与客户端建立连接，获取权限，位置和管理连接
3. MySQL 服务端拿到一个查询请求，会先到查询缓存中查看，是否有缓存（8.0版本之后废弃了查询缓存），如果执行过这个缓存，则会**将执行语句与查询结果以K-V形式存在内存**中，直接返回结果。如果没有查询到缓存，则会开始执行语句，分析器会先做词法分析，识别出关键字和表名，再做语法分析，判断输入语句是否合法。
4. 经过分析器，MySQL 优化器会选择使用哪个索引（如果有多个表，会选择表的连接顺序）
5. 最后一个阶段是执行器会调用引擎的查询接口去执行语句
6. 事务开始，写 **undo log** ，记录上一版本数据，并更新记录的回滚指针和事务ID
7. 执行器会取 id = 2 这一行 。id 是主键，引擎直接用B+树搜索到这一行
    1. 如果 id = 2 这一行所在的数据也本来就在**内存**中，就直接返回给执行器更新
        1. 如果记录不在内存，接下来会判断单索引是否是**唯一索引**
        2. 如果是唯一索引，就只能把数据页从磁盘中读入到内存中，返回给执行器
8. 执行器拿到引擎给的行数据，把这个值加上1，比如原来是N，现在就是N+1，得到新的一行数据，再调用引擎接口写入这行新数据；
9. 引擎将这行数据更新到内存中，同时将这个更新操作记录到 **redo log**里面；
10. 执行器生成这个操作的 **binlog** ；
11. 执行器调用引擎的提交事务接口；
12. 事务的两阶段提交：commit的prepare阶段：引擎把刚刚写入的redo log刷盘；
13. 事务的两阶段提交：commit的commit阶段：引擎binlog刷盘。

![Untitled](MySQL%209f9b1877c5204079bcee93ab197a7a4b_Untitled.png)

---

# ****
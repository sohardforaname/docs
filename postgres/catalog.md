# Postgresql的catalog设计与数据管理

## 管理数据字典的模式

Postgresql中使用表空间，模式来作为一个数据库中对象的管理工具。其中，表空间是指一个磁盘上的目录，而模式则是在数据库下的一个个对象的集合，其中，information_schema和pg_catalog
是实现数据字典（catalog）管理的两个重要模式。

## 管理数据的方式

Postgresql会给每个对象分配一个oid，这个oid就是一个对象的唯一标识符，对象的oid可以在pg_class这个系统表中获取，pg_class还记录了这个对象的一系列信息，包括对象类型等。我们可以查阅文档
来了解每个系统表的字段及其含义。[文档链接](https://www.postgresql.org/docs/current/catalog-pg-class.html)

除此之外，还有一些系统表：
pg_class;
pg_database;
pg_proc;
pg_namespace;
pg_depend;
pg_type;

但是系统表之间的关系本身也错综复杂，尤其是我们需要添加过滤条件的时候，因此我们也有必要总结一下各个系统表之间的关联，然后使用join的on条件来实现一些复杂过滤。

TODO: 系统表关联图

## 系统表的读写优化

在parse, analyze, execute(DDL, DML)的时候，需要经常访问系统表，因此系统表的优化将会在这些过程中起到重要作用，PG提供了一些接口来自动使用合适的索引访问系统表，并且如果涉及系统表操作，还可以自动更新索引。
此外，PG中系统表也有缓存机制来加速查找。但是缓存的失效更新机制也成为一个设计问题。这些接口分别是CatalogTupleInsert/Update/Delete函数，调用这些函数会自动更新索引。而session级别也会使用称为Syscache的缓冲机制。

Syscache是一个系统级别的缓存表，结构由哈希表实现，在文件syscache.c中定义。里面的函数分成四类：Init, Search, Get, Release。其中重要的函数就是Search。

由Search函数的接口：SearchCatCache(SysCache[cacheId], key1, key2, key3, key4); 我们可以发现，缓存中最多支持4个key，由于使用的是哈希表进行缓存记录，因此只能支持等值查找，同时由于是使用datumCopy函数记录的key，因此不支持NULL值作为key。如果找不到，则打开系统表查找并写入到缓存中。如果不满足上面的条件，也使用 不使用缓存 的接口，打开系统表获取结果。

Release接口则是用户断开连接时回收缓存。

对于存在变长字段的系统表，PG使用CATALOG_VARLEN宏来区分定长区和变长区，定长区可以直接使用memcpy赋值，而变长区必须使用SysCacheGetAttr函数，传入tupledesc获取。这样的好处就是，一般情况下用户获取定长字段，如oid等的情况比获取变长字段更加普遍。因此设计了使用memcpy来加速，对于一些比较少见的场景，则使用较慢的函数获取。

## 缓存的更新与失效

Syscache借助的哈希表没有多版本机制，由于不同的session中维护了不同的Syscache。
1. 随着并发执行进行，这些Syscache就有可能出现不一致，使用什么机制可以让这些Syscache保持一致性并且不会破坏ACID呢？
2. 缓存的失效会触发读取文件并更新缓存，如果多次查询不存在的数据，将产生大量打到物理表上的查询。

Postgres使用了共享内存的方式实现这个过程。在事务进行中，会分层次写入修改记录。首先在调用了系统表的更新函数时，会记录更改的tuple，并实时更改并立即生效本session的Syscache。
在主事务提交时，CommandCounterIncrement()执行，再写入到共享内存中，在此之后其他的session启动新的事务时，事务会从共享内存中拉取更新并应用更新完成同步。这就保证了Syscache中的记录总是已经提交的数据；
并且在事务开启时，每个开启的事务的系统表都将会有所有已经提交的系统表更改数据，保证了一致性。
为了防止多次查找同一个不存在的数据导致多次访问物理表，缓存中还会针对这种查询建立一个不存在标记的缓存。

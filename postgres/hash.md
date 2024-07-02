# Postgresql的hash索引数据结构

## 索引结构定义

Postgres的Hash索引使用了一种叫做线性哈希的哈希表结构，这样的哈希表结构支持动态增长桶的大小以增加桶的容量。
线性哈希的增删查改算法如下：

线性哈希解读见：![点我](https://xubo123.github.io/2017/12/10/%E7%BA%BF%E6%80%A7%E5%93%88%E5%B8%8C/)

Postgres中一个完整的hash index分成四种页：元数据页（总是位于0号页面），索引数据页，位图页和溢出页。
其中，元数据页保存了哈希索引中的桶数和元组数，溢出页信息和桶的计算信息，详见hash.h:HashMetaPageData结构体；桶页用于保存实际的hash索引数据元组；
位图页用于跟踪溢出页；溢出页则是为超出一页的桶额外分配的空间。

## 插入

- 构造index tuple，这个是老生常谈了，但是注意的是，index tuple中是记录了key值的hash值而不是完整的key，因为考虑到key可能会很长，记录完整的key可能会导致一个hash index tuple超出一页。
当在查找时遇到了相同的hash值的key时，可以通过回表比较key是否相等，并执行可见性检查。
- 执行hash index的_do_insert函数。
  - 获取meta page页面但是不需要上锁；
  - 检查可串行化冲突（TODO）
  - 如果bucket在分裂状态，则完成分裂并重新尝试插入
  - 如果桶中有死元组，执行一遍页内vacuum，然后再尝试检查空间，如果仍不足，分配溢出页
  - 找到空闲页了，上写锁，设置临界区标记
  - 写入数据页，数据页保证按照hash值二分排序，找到对应的位置插入即可。并保证hash值依旧有序
  - 为meta page上写锁，ntup += 1，并把这两个页的修改写入WAL，WAL顺序是先meta后数据页。
  - 释放锁检查负载因子决定是否扩展

## 构造

hash index不能像btree index一样自底向上构造，它只能选择循环执行insert的方式插入。

## 查找

hash index上执行查找主要是执行hash.c:hashgettuple函数，这个函数的实现又依赖于_hash_first和_hash_next两个函数。

_hash_first函数：

# Postgresql的hash索引数据结构

## 索引结构定义

Postgres的Hash索引使用了一种叫做线性哈希的哈希表结构，这样的哈希表结构支持动态增长桶的大小以增加桶的容量。
线性哈希的增删查改算法如下：

线性哈希解读见：[点我](https://xubo123.github.io/2017/12/10/%E7%BA%BF%E6%80%A7%E5%93%88%E5%B8%8C/)

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

## 分裂

由于查找同样依赖于分裂算法，我们先说分裂过程。
_hash_expandtable函数：

这个函数是split的入口函数。

- 获取meta页的写锁，检查hash index是否仍需要split，因为可能在等锁时另一个线程已经完成了split。
- 我们总是让new_bucket = max_bucket + 1; old_bucket = new_bucket & lowmask
- 尝试为old_bucket获取清理锁（即exclusive_lock和pin == 1）
- 如果当前这个页面的状态是split状态（TODO：需要说明什么时候进入分裂状态什么时候结束）
  - 记录下当前meta页上的max_bucket，high_mask和low_mask。
  - 释放待分裂的桶页和元数据页的锁，调用finish_split函数完成上次分裂
    - 为分裂的目标桶建立哈希表，为元数据页和桶页上读锁，记录已经被复制到新桶的index元组
    - 重新获取正在分裂的桶的清理锁，如果失败则销毁哈希表返回，等待下次split时再尝试这个流程。
    - 重新获取目标桶的清理锁，如果失败则销毁哈希表返回，同上
    - 调用_hash_splitbucket完成分裂的数据操作。
  - 重新尝试分裂
  
_hash_splitbucket函数。

- 为目标桶上谓词锁
- 扫描源桶的所有未死亡元组，如果已经在哈希表中存在，则直接跳过这个元组。
- 遇到需要移动的元组
  - 复制一个index元组，并将它放到内存分配的数组中
  - 若数组中的元组放进页中页刚好放满或已经遍历完了这一页，则执行实际的插入操作并写WAL，然后增加一个溢出页
- 给源桶和目标桶的首页上锁，为了防止死锁，一定是先加源桶的锁）
- 执行标记调整，为源桶和目标桶分别取消being_split和being_populated标记，为源桶新增need_split_cleanup标记。
- 尝试一次清理源桶，并解全部的锁

## 查找

hash index上执行查找主要是执行hash.c:hashgettuple函数，这个函数的实现又依赖于_hash_first和_hash_next两个函数。

_hash_first函数：

这个函数主要的用处是初始化scan和找到第一个哈希值相同的元组。相当于是创建迭代器并且找到第一个元素的过程，同理，
_hash_next函数就是迭代器中的next方法。只不过这里把hasNext方法和next方法合在一起了。

- 获取hash key，并通过hash key计算出对应的hash桶所在的buffer，为这个buffer上谓词锁，上谓词锁是防止出现插入行为触发hash桶的分裂；
- 如果当前的桶处于填充状态（being populated），说明它是某一个正在分裂的桶的目标桶，我们需要跳过复制过来的元组。
  - 为了速度更快，我们总是先尝试使用缓存的metapage
  - 使用哈希函数计算后的hashkey获取了当前的hash桶号，调用函数_hash_getbucketbuf_from_hashkey获取bucket
    - 计算出block，如果这个bucket没有在split状态，则直接返回这个block的page
    - 否则出现了split，此时我们强制从磁盘更新metapage，然后重新计算block和对应的page并返回
  - 此时我们获取这个处于填充状态的桶，并上锁了，处于未完成split的状态。由于上锁导致split停止数据复制，
    因此存在一部分应该放在这个桶的数据会放在原来的桶。
  - 我们释放这个处于填充状态的桶的锁，然后尝试以先源桶（being split）后目标桶（being populated）的顺序加锁。
    并记录下这个split状态和populated状态的桶号，然后释放锁。但是保持住pin，防止物理清理操作。
- 如果是backward scan，则我们必须找到最后一个page，如果这个bucket处于被填充的状态，我们就需要跳到填充操作对应的源桶
  因为这里会有数据将会被移动到目标桶。
- 调用_hash_readpage完成迭代器初始值设置。

_hash_next函数：
按照读取顺序读取这个page的全部的元素，然后根据direction切换到前面的页或者是后面的页即可。
由于在前面的读取过程中，由于数据页变动或者是别的情况（TODO），导致出现了失效元组，我们需要在切换页之前，为这些失效元组标记为死元组。

## 清理

清理与删除的函数的入口都是在hashbulkdelete。解析下这个函数

hashbulkdelete函数：

- 获取缓存的metapage，记录ntups和max_buckets
- 扫描每一个桶
  - 如果这个桶处于need_cleanup状态，我们就不能删掉这些元组，因为finish_split需要用到它们完成split。
  - 标记split_cleanup = true，并刷新metapage，因为可能在这段时间中出现了split。
  - 调用hashbucketcleanup函数
    - 如果出现了split，要记录下split的目标桶
    - 对这个桶上的每一个page，扫描它的指向的堆元组是否需要删除，判断删除的条件是：callback函数返回true或者是已经被移动到split的目标桶
    - 把需要删除的元组记录到数组中
    - 执行删除操作并写入到WAL中，保持桶的第一页的pin，找到下一页并释放当前页的锁，为下一页加锁重复进行删除
    - 如果标记了split_cleanup，需要去除split_cleanup标记并写入WAL
    - 尝试获取清理锁并对bucket page重新排布以回收溢出页，调用的是_hash_squeezebucket函数。
- 为metapage上写锁并检查是否在delete的时候发生了split，如果发生了split则释放metapage的锁，并刷新缓存。然后重新尝试扫描每一个桶执行清理
- 更新metapage的信息并写入WAL中，返回统计信息结构体。

_hash_squeezebucket函数：
找到当前桶链的最后一个页，因为溢出页一般来说都是放在桶页链表的最后。然后用双指针的算法，把后面的page的index tuple移动到前面的weite page中，
过程类似于先进行了批量插入后再批量删除，故不再赘述，每清理完一个页都会把数据页更改和元数据页的更改写入WAL，然后释放这个溢出页。

## 并发控制

TODO

## 异常处理

TODO
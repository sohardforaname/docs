# Postgresql的nbtree索引数据结构

## 索引结构定义

B-linked tree：
一个N阶平衡树，无论是叶子节点还是内部节点都有链接右兄弟的节点，在节点满的时候会把一部分数据转移到兄弟节点中，仅两个节点都满了才会触发分裂。
其余情况和B-tree相同，即data存key值，为它指向的页的最小值（或者最大值），只有在叶子节点上，才会存储所有的key。

一个B-linked tree的page组成分为：
0号page存储index的元数据。剩余的page存储索引数据。

## 插入

入口函数：nbtree.c:btinsert;
### 执行流
- 构造scankey，从heap tuple中获取对应的列然后从中构造出index tuple
- 通过_bt_search函数找到查询路径上的stack，这个stack表示查询的路径的page，在每次访问page时，都会尝试一次_bt_moveright，防止出现了页分裂。
  - 通过_bt_binsearch函数执行页内二分查找，调用_bt_compare函数
- 通过stack找到对应的page，在每次访问更深处的叶子时，都会先加叶子的读锁然后再释放父亲的读锁。
- 进行唯一性检查
  - 查找页中的offset，对这个offset，如果是访问到了页中的元组并且未标记dead，则使用SNAPSHOT_DIRTY回表检查
    - 回表检查时，有可能这个tuple是一个HOT链，此时需要检查整个HOT链上是否有仍然可见的tuple而不是只检查索引对应的tuple。
  - 对CHECK_UNIQUE，检查活跃事务并返回事务ID，对CHECK_UNIQUE_PARTIAL，标记is_unique = false，并返回invalid事务号。
  - offset还有可能是页尾，此时需要检查下一页的第一个元组
  - 如果有活跃事务号，需要在该事务结束后重新_bt_search，因为它可能会改变索引结构。
- 通过 _bt_findinsertloc函数在当前找到的页面内找一个合适插入索引项的位置
- 通过 _bt_insertonpg 执行插入逻辑：
  - 如果page还有空闲空间，直接执行插入page操作，并写WAL
  - 如果page满，则通过_bt_split函数进行节点的分裂，完成分裂之后通过_bt_insert_parent()将旧page的high-key插入到父节点。然后重新尝试插入
- 释放页面

### 优化

- 插入后的page会缓存等待下次插入，如果有别的事务在使用这个缓存，则另一个事务会走正常的插入流程。
- 使用fastroot，而不需要获取meta中的实际的root。

## 分裂机制

### 叶子节点分裂

叶子节点分裂比较简单，通过_bt_findsplitloc函数找到分裂点，然后插入数据到新的右兄弟节点中。此时需要把左边旧节点的high-key提升至父亲节点中。
分裂的过程首先是创建一个左节点临时页，然后把原来的节点的index tuple调整到临时左节点和新的右节点，然后临时左节点的数据再复制回原左节点中，记录xlog。

### 分支节点分裂

类似叶子节点，但是需要标记左孩子为BTP_INCOMPLETE_SPLIT，如果是root节点，则需要额外更新meta的信息。

## 查找

查找机制比较重要，是索引中最主要的操作。构建scankey的过程忽略，直接从_bt_search说起。

首先是正常流程，沿着树查找，如果在路径上发现了BTP_INCOMPLETE_SPLIT，则说明路径上它的父节点没完成分裂。如果是插入模式，则调用_bt_finish_split完成分裂操作，然后决定是否需要moveright，最后返回stack路径。

在一个分支节点中查询的流程：通过二分查找即可，依赖于key->nextkey决定是否是严格大于。

在查找时保证在当前节点加读锁，如果需要finish split则升级为写锁，在当前节点完成后，先加孩子节点的读锁后释放本节点的锁。

在叶子节点中的流程就比较复杂了，在查找时，首先与分支节点类似，使用二分查找找到对应的index tuple，然后尝试回表找到这个tuple或者是HOT链，如果已经不可见则需要在_bt_killitems函数中标记这个index tuple。
考虑非唯一索引，索引中会有多个index值相同的heap tuple，我们在二分找到第一个tuple后，仍需执行顺序查找，此时就有可能跨页。在跨页时，我们就需要先给在当前页中的index tuple标记dead，
然后获取下一个叶子页面并获取锁。然后对这个叶子重复操作。

针对_bt_killitems函数中标记这个index tuple时，即使获取了读锁，在这个读锁获取前也可能发生了页分裂，此时数据依旧不会不一致，因为索引上没有mvcc数据，这些索引元组是否被标记取决于heap tuple，
只要heap tuple具有一致性，索引也如此。

## 删除

索引没有删除，通过lazy vacuum来统一删除，触发vacuum可能是空间不够时或者是在删除索引，表vacuum时。由于vacuum的过程主要是在page上进行的，因此我们只需要关注这个过程。

首先在vacuum page时，我们必须保证：收集好了所有的dead tuple，pin == 1, 获取写锁；清理过程即首先从heap page中找到对应的

如果这些dead tuple比较少时，直接删除也只会产生少量的空洞，排序offset将带来大量代价，因此可以直接删除并设置unused，否则整理page。
整理page的过程即对offset排序，然后计算出非dead的元组的位置并移动即可。索引的删除需要记录到WAL中。

## 清理

索引的清理工作就是调用每一个页做vacuumpage的过程，首先我们需要知道哪些index tuple需要删除，因此我们使用lazy_scan_heap函数获取死元组，然后把page中对应的index tuple标记dead。
然后再对这些dead元组执行上文的删除操作中的lazy vacuum。最后回到堆表中删除元组，这里不能先删除堆表再删除索引，因为可能会有别的事务重新获取了被删除的元组的ctid。

## 并发控制

在多个事务同时操作时，在以下几个地方将可能出现并发问题：
在查找时，由于在stack中我们在找到子节点时会释放当前节点的锁，因此可能出现了后续在遍历路径时出现分裂，因此需要moveright。
如果在插入时遇到的分裂，当前节点将持有右边的新页的写锁，然后还需要获取原来的右节点的写锁更新链表节点并写WAL，写完后立即释放后者，
获取父亲节点的写锁。并释放右边的新页的写锁。high-key写入父节点后释放父节点和原左边节点的写锁。

## 异常处理

索引上没有MVCC信息，因此采用原地更新的机制，此时如果出现了宕机，由于新写入的page需要写xlog，则可以直接通过xlog恢复。
如果出现了分裂操作但是分裂没完成，首先新的右节点全程在xlog保护中，因此右节点不会出现数据不一致问题，只需要考虑左节点和父节点。
若左节点未完成数据调整，重做即可。若已经完成，则说明也已经写入xlog。叶子节点的数据不丢失，则说明索引已经写入完成，把左节点设置为BTP_IMCOMPLETE_SPLIT，
在下一次search时调整父节点的指针即可。



            




            
            



            

        

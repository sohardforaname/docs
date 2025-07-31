## ARIES错误恢复算法

ARIES是一种数据库用于保证错误后恢复到数据一致性状态的算法。
- 它是steal策略（可以将未提交的事务的脏页写回磁盘）和no-force策略（事务提交之后不需要马上写回磁盘）
- 需要写WAL日志，在错误恢复时根据日志执行Redo和Undo操作
- 几乎所有DBMS都使用了ARIES及其变种作为错误恢复算法。
### 实现
ARIES使用WAL来记录事务的所有操作。在事务提交时，WAL需要同步刷入磁盘中。在WAL中，日志的序号唯一且递增，使用LSN（Log Sequence Number记录）。同时，在BufferPool中的Page，磁盘的Page和数据库全局参数中也有一些LSN的记录用于算法执行。分别是
- flushedLSN：被刷到磁盘的WAL中的最后一个LSN
- pageLSN：页面中的最后一个修改LSN
- recLSN：使得页面变脏第一个修改LSN，Redo阶段将从这个LSN开始对该页重做。
- lastLSN：每个事务id的最后一个记录的LSN
- masterRecord：磁盘中的WAL最后一个检查点的ckpt-end的LSN

在正常运行时，事务每一个操作都会往WAL中写入一条记录（LSN, page, cur_val, next_val）；并更新pageLSN和该事务的lastLSN。
- 当事务commit时：写入一个commit的日志到WAL中，WAL刷出磁盘。更新flushedLSN，然后经过一些时间，在WAL中写入一个txn-end的日志，标志着以后不再会有该txn的日志出现。在任意时间点，只有pageLSN <= flushedLSN的page才能刷出磁盘。因为使得这个page上的变脏的所有修改的日志都被刷出磁盘了。
- 当事务abort时：写入一个abort日志到WAL中，此时需要进行Undo操作，引入CLR日志来进行Undo操作，一条CLR日志中包含了（LSN, cur_val, prev_val, undoNext），并且在每一条日志中，记录该事务的上一条日志的LSN。CLR需要写入到WAL中，但是不需要同步下刷出。abort完成之后，写入txn-end日志。同时CLR日志不需要做Undo操作。

当发生错误时，DBMS将会尝试执行Redo操作和Undo操作使数据库恢复到一致性状态。此时，数据库需要读取全部的WAL日志进行恢复操作，这会消耗大量时间，因此引入Fuzzy Checkpoint算法（不停机检查点）。

Fuzzy Checkpoint算法的引入，需要在WAL中增加两种Log类型：ckpt-begin和ckpt-end，在DBMS正常运行状态中需要维护ATT（Active Txn Table）和DPT（Dirty Page Table）。
- ATT：事务在begin时写入记录（txn_id, status），在txn-end时删除记录，其中status有R：running；C：committed；U：undo。
- DPT：脏页表，页面在被修改后写入记录（page_id, recLSN），在被刷出磁盘时移除记录。

DBMS每隔一段时间执行检查点操作，往WAL中写入ckpt-begin日志，然后复制一份ATT和DPT的副本（因为不停机，在检查点执行时允许事务操作）。在ckpt-begin之后begin的txn及其操作不会记录到ATT和DPT中。然后在ckpt-end日志中写入ATT和DPT的记录。此时，我们在Redo时就不需要从头开始，只需要找到最后一个检查点，找到检查点中DPT的所有页中最小的recLSN，然后从这里开始重做所有记录即可。
### 恢复
- Analyze阶段
首先找到最后一个检查点，获取ATT与DPT和ckpt-begin后所有开始begin的txn。加入ATT中；找到DPT中page的最早的recLSN作为Redo阶段的起始LSN；和ATT中所有事务最早的LSN（oldestLSN）。recLSN和oldestLSN不一定小于ckpt-begin的LSN，因为有可能在检查开始时事务开启了。
接下来从recLSN开始：
	- 遇到begin，加入txn_id到ATT中，状态是U。
		- 如果遇到了txn-end，移除txn_id。
		- 如果遇到了commit，状态改成C。
	- 遇到了修改记录，在DPT中加入page_id。
		- 将当前的LSN写入到DPT中对应page的recLSN。
- Redo阶段
找到Redo的起始LSN。正向执行每一条修改操作和CLR操作，并将curLSN写入到pageLSN中。
	- 如果page不在DPT或者是curLSN < 页面的recLSN，则忽略。
	- 如果均满足，则读取页面。
		- 判断pageLSN < curLSN时，说明在磁盘的页还没有应用该更改，我们就需要应用，并更新pageLSN为curLSN。
		- 否则说明已经应用更改并落盘，这一条更改可以忽略。
	- 如果遇到了commit日志，则写入txn-end日志并将txn_id移除ATT。
- Undo阶段
此时ATT中剩下的记录都是状态为U的txn_id，逆向对这些txn的所有修改记录回滚并写CLR日志到WAL，并commit即可。显然这里就需要通过事务的lastLSN找到UNDO操作的起始点

经过这些操作之后，数据将恢复到一致性状态。
### 恢复时故障
- Analyze阶段又发生了故障
无事发生，analyze阶段不改数据
- Redo阶段又发生了故障
重做都是读磁盘的WAL，将数据写入到内存的page中，具有幂等性，下次恢复时重做一遍即可。
- Undo阶段又发生了故障
CLR操作的存在保证了Undo的幂等性和可重启性，即使在Undo时出现了故障，因为写入到WAL中的CLR操作都是一致的，因此保证了Undo操作的幂等性。

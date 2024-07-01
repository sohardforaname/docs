# Postgresql的事务实现与元组的增删改

## 事务实现的数据结构

Postgresql中的事务由一个自增不重复的事务号唯一确定，事务号较大的事务一定比事务号较小的事务更晚开始执行。
一般来说，事务的实现和存储引擎的实现紧密耦合，因此我们不得不结合Postgresql的存储引擎介绍事务的实现机制。

Postgresql中用于实现事务ACID特性的模块有锁，日志，MVCC，此外还有进程管理器。其中这些管理器又由事务管理器统一管理。

事务本身是由多个状态组成的过程，所以事务管理器本身也是一个状态机，通过内部或外部的输入进行状态转换。一个事务的状态如下所示：
```C++
/*
 *	transaction states - transaction state from server perspective
 */
typedef enum TransState
{
	TRANS_DEFAULT,				/* idle */
	TRANS_START,				/* transaction starting */
	TRANS_INPROGRESS,			/* inside a valid transaction */
	TRANS_COMMIT,				/* commit in progress */
	TRANS_ABORT,				/* abort in progress */
	TRANS_PREPARE,				/* prepare in progress */
} TransState;

/*
 *	transaction block states - transaction state of client queries
 *
 * Note: the subtransaction states are used only for non-topmost
 * transactions; the others appear only in the topmost transaction.
 */
typedef enum TBlockState
{
	/* not-in-transaction-block states */
	TBLOCK_DEFAULT,				/* idle */
	TBLOCK_STARTED,				/* running single-query transaction */

	/* transaction block states */
	TBLOCK_BEGIN,				/* starting transaction block */
	TBLOCK_INPROGRESS,			/* live transaction */
	TBLOCK_IMPLICIT_INPROGRESS, /* live transaction after implicit BEGIN */
	TBLOCK_PARALLEL_INPROGRESS, /* live transaction inside parallel worker */
	TBLOCK_END,					/* COMMIT received */
	TBLOCK_ABORT,				/* failed xact, awaiting ROLLBACK */
	TBLOCK_ABORT_END,			/* failed xact, ROLLBACK received */
	TBLOCK_ABORT_PENDING,		/* live xact, ROLLBACK received */
	TBLOCK_PREPARE,				/* live xact, PREPARE received */

	/* subtransaction states */
	TBLOCK_SUBBEGIN,			/* starting a subtransaction */
	TBLOCK_SUBINPROGRESS,		/* live subtransaction */
	TBLOCK_SUBRELEASE,			/* RELEASE received */
	TBLOCK_SUBCOMMIT,			/* COMMIT received while TBLOCK_SUBINPROGRESS */
	TBLOCK_SUBABORT,			/* failed subxact, awaiting ROLLBACK */
	TBLOCK_SUBABORT_END,		/* failed subxact, ROLLBACK received */
	TBLOCK_SUBABORT_PENDING,	/* live subxact, ROLLBACK received */
	TBLOCK_SUBRESTART,			/* live subxact, ROLLBACK TO received */
	TBLOCK_SUBABORT_RESTART,	/* failed subxact, ROLLBACK TO received */
} TBlockState;
```

当然，在内核中，事务只有几种情况，如第一个枚举类型。第二个枚举类型里面的是指当前这个事务的状态针对于查询而言的。同时，我们顺便介绍一下表示一个事务的数据结构。

```C++
typedef struct TransactionStateData
{
	FullTransactionId fullTransactionId;	/* my FullTransactionId */
	SubTransactionId subTransactionId;	/* my subxact ID */
	char	   *name;			/* savepoint name, if any */
	int			savepointLevel; /* savepoint level */
	TransState	state;			/* low-level state */
	TBlockState blockState;		/* high-level state */
	int			nestingLevel;	/* transaction nesting depth */
	int			gucNestLevel;	/* GUC context nesting depth */
	MemoryContext curTransactionContext;	/* my xact-lifetime context */
	ResourceOwner curTransactionOwner;	/* my query resources */
	TransactionId *childXids;	/* subcommitted child XIDs, in XID order */
	int			nChildXids;		/* # of subcommitted child XIDs */
	int			maxChildXids;	/* allocated size of childXids[] */
	Oid			prevUser;		/* previous CurrentUserId setting */
	int			prevSecContext; /* previous SecurityRestrictionContext */
	bool		prevXactReadOnly;	/* entry-time xact r/o state */
	bool		startedInRecovery;	/* did we start in recovery? */
	bool		didLogXid;		/* has xid been included in WAL record? */
	int			parallelModeLevel;	/* Enter/ExitParallelMode counter */
	bool		parallelChildXact;	/* is any parent transaction parallel? */
	bool		chain;			/* start a new block after this one */
	bool		topXidLogged;	/* for a subxact: is top-level XID logged? */
	struct TransactionStateData *parent;	/* back link to parent */
} TransactionStateData;

typedef TransactionStateData *TransactionState;
```

由上面的结构可以看出来，事务的high-level state就是相对于查询的事务状态，low-level state就是相对于内核本身的事务状态，这种分层的机制既可以明白地展示事务的状态，
且在不同的顶层状态下，对于事务本身而言，状态与执行的转移函数可能是一致的，所以底层状态机实际上是对顶层状态和转移的归类，这样子就使得设计简单并屏蔽了底层的细节。

## 事务的状态转换

事务的状态转移由一个个转换函数驱动，比如说开启事务就是一个状态转移，由idle状态转换到start状态（见枚举一）。因此，我们可以顺着找到转移函数的实现：xact.c中的BeginTransactionBlock。
打开这个函数，你将会看到事务的high状态机执行了状态转移。然后在这个状态转移函数中，调用了底层的状态转移接口进行底层状态转移并实际执行事务操作。

类似的函数还有（不考虑子事务）：
顶层：

```C++
StartTransactionCommand
CommitTransactionCommand
AbortCurrentTransaction
BeginTransactionBlock
EndTransactionBlock
UserAbortTransactionBlock
```
底层
```C++
StartTransaction
CommitTransaction
CleanupTransaction
AbortTransaction
```

## 事务如何发挥作用

在Postgresql中，由上文提到，每一个事务都会分配一个事务号（transaction id，简称xid），由于其自增唯一，因此事务的id顺序决定了事务的开始顺序。
我们就可以在元组的头标注xmin和xmax表示插入该元组和删除该元组的事务ID。同时维护一个全局xmax来为vacuum提供信息，确认哪些元组可以被物理删除。同时xmin和xmax在元组中以32位存储，
然后在heap的headData中记录了base实现对32位系统的兼容。

在数据库运行中，我们还会记录全局各个事务的状态，来进一步判断元组可见性（比如说一个后开始的事务，只能看到它开始之前的已经提交的事务的元组，因此只通过xmin不能判断可见性，还需要用xmin找到事务本身的状态），
因此我们还需要事务xid与事务状态的映射，这个映射表可能很大（导致内存的局部性差），因此我们使用clog文件和SLRU算法来实现缓存机制。同时，由于在恢复时也需要事务的状态决定提交或者是回滚，此时也需要clog文件。

为了支持MVCC机制，在MVCC中，我们需要知道某一个状态下的事务是否已经提交（当前读可以用clog，但是快照读则不能），因此需要知道在某个“版本”之前的事务提交状态，因此我们需要csn，即提交顺序id。
提交顺序id也是一个自增的数字，在一个事务提交后，它就往前推进一位，建立事务号到提交顺序的映射，而csnlog就是管理这样的映射的文件，当我们需要某个版本的可见性时，则直接查找xmin检查元组是否已提交，或者查找xmax检查元组是否已经删除。

## 事务id的自增性

在写写冲突出现时（MVCC只能解决读写的冲突），必须通过锁机制或者是强制让一个事务回滚来实现写的顺序性（等待或回滚，取决于系统的设计），或者是出现死锁时解锁。事务的id的自增顺序性就很好地解决了这一点，我们可以选择让新事务回滚后重新执行或者是让新事物等待老事务执行等方案解决冲突，或者是在出现死锁时回滚新事务等。

## 实际场景中事务的作用

### 插入

插入的行为最为简单，插入的事务会在事务开启时设置事务号，然后在插入元组时，设置xmin为xid，xmax为-1（表示未被删除），然后提交事务，记录clog为提交状态，记录csnlog即可，若回滚了，则需要记录clog为回滚状态，删除已经插入的行。

### 删除

删除行为也是同理，设置事务号，然后找到对应的元组，设置xmax，提交事务即可，若需要回滚，则需要恢复xmax，然后回滚，clog和csnlog的状态变化类似插入。

### 更新

更新行为可以等价于删除后插入，因此可以在一个事务中先执行一次删除操作再执行一次插入操作，把原元组的xmax和新元组的xmin都记录成当前事务的xid即可。

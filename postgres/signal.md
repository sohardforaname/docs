# Postgresql的信号处理机制

## linux中的信号机制

信号在linux系统中常用于通信使用，接收到信号的进程会被打断并执行信号处理函数，然后再恢复当前进程的执行。但是在实际应用时，信号也有一些特性需要我们加以了解。
- 信号的处理函数
接收信号之后的处理函数是可以自定义的，使用signal函数就可以默认设置这个自定义处理函数，并且它总是返回上一个信号处理函数，后设置的处理函数总是覆盖前一个设置的函数。
```C++
sighandler_t signal(int signum, sighandler_t handler);
int sigaction(int sig, const struct sigaction* act, struct sigaction* oldact);
```
注：更推荐使用sigaction，因为不同的unix/Linux系统中signal函数的行为不一定一致。
- 信号的屏蔽
```C++
int sigprocmask(int how, const sigset_t *_Nullable restrict set,
                                  sigset_t *_Nullable restrictoldset);
int pthread_sigmask(int how, const sigset_t *set, sigset_t *oldset
```
通过这两个函数，可以分别对某个进程或者线程屏蔽某个信号，使其在接收到信号时不会触发中断。尽可能不要在多线程环境中使用sigprocmask，因为这是未定义的。
- 信号发送
```C++
kill函数
pthread_kill函数
```
- 坑点
1）阻塞执行的系统调用也有可能会被信号中断，因此在出现错误时，需要考虑是否由于中断引起（实际上在库中已经实现了这些操作，但是我们也需要注意）
2）作为信号触发后执行的信号处理函数，必须是可重入的（即它的执行不会对外部的环境造成干扰，如使用静态数据结构或者访问全局变量的）函数（看起来有点像是FP中纯函数的概念？），这样子避免了在程序执行期间，中断到来，处理函数修改了同一处变量或者是数据结构导致的程序行为异常现象。

- 信号的阻塞
在执行对应信号的处理函数时，如果再收到信号，则信号会被阻塞，进入pending状态。被阻塞的信号在处理函数解决后会进行一次投送尝试，使得信号处理函数再执行一次。
for example：
```C++
static int my_conter;
static void my_handle(int sig) {
    my_conter++;
    sleep(10);
}

int main(int argc,char* argv[]) {
    my_conter = 0;
	signal(SIGINT, my_handle);
    sleep(10);
    printf("%d\n", my_conter);
}

/* ***** */

static int a;
static int b;
static void func1(int sig) {
    a++;
    sleep(10);   /*可以被SIGINT之外的信号中断*/
}
static void func2(int sig) {
    b++;
    sleep(10);  /*可以被SIGTERM之外的信号中断*/
}

int main(int argc,char* argv[]) {
    a = 0;
    b = 0;
	signal(SIGINT, func1);
    signal(SIGTERM, func2);
    sleep(10);
    printf("a=%d, b=%d\n", a, b);
}

```
例子1中如果在信号处理函数sleep时再触发SIGINT，这些信号会被阻塞直到处理函数执行完成后尝试一次投送。
例子2中如果在信号处理函数sleep时再触发SIGINT和SIGTERM，如果两次是同一信号则类似于例子1，否则，处理函数中的sleep可以被不同的信号类型打断并执行对应的新的信号处理函数。

## Postgres如何使用信号

### PG_TRY，PG_CATCH

PG的异常处理机制使用信号触发函数的机制实现。在PG_TRY()中设置sigsetjmp函数，然后出错时使用siglongjmp函数。如下例子：
```C++
void sig_usr1(int sig) {
    ...
    siglongjmp(exception_jmpbuf, 0);
}

#define PG_TRY() if (sigsetjmp() == 0) {
#define PG_CATCH() } else {
#define PG_END() }
```
使用siglongjmp时信号将执行sigsetjmp返回非0，执行catch模块的代码。

### 进程通信
```C++
static void handle_pm_pmsignal_signal(SIGNAL_ARGS);             // handle child process signal
static void handle_pm_child_exit_signal(SIGNAL_ARGS);           // child exited, handle resource
static void handle_pm_reload_request_signal(SIGNAL_ARGS);       // handle reload conf
static void handle_pm_shutdown_request_signal(SIGNAL_ARGS);     // handle stop postmaster process

void StatementCancelHandler(SIGNAL_ARGS);       // handle cancel query
void die(SIGNAL_ARGS);                          // handle postgres exit
void procsignal_sigusr1_handler(SIGNAL_ARGS);   // handle postgres signal
```

### 中断处理结合事务

在postgres收到SIGTERM信号时，它将执行die函数结束事务清除资源，如何结束事务呢？直接使用elog函数是不可行的，因为AbortTxn函数将使用全局变量，这会引发上文提到的异常行为，同时在临界区时中断并abort事务，error会升级成panic使数据库直接宕机。

因此我们必须手动设计一种中断等待机制，使得程序可以决定中断在哪里可以被执行，简单说就是在中断处理函数中仅设置flag，在离开临界区后在CHECK_INTERRUPT等函数中确认是否有pending的信号，如果有再手动调用实际的中断处理函数。

### 进程之间通信

在postmaster，postgres和worker中进行通信，PG使用了SIGUSR1信号，具体来说，PG的通信机制使用了共享内存和信号，它在sendProcSignal函数中会先写入到procSignalSlot中，然后再使用SIGUSR1通知对应的进程去访问procSignalSlot获取消息。
由上文提到，在PG中并不是每个时刻都可以进行中断，因此这样的设计可以由接受信号的进程自行角色处理这个信号的时机。

### Postgres的启停

PG中的进程只能通过Postmaster进程fork，因此比较显然的策略是，在fork前屏蔽所有信号，fork时子进程会继承主进程的屏蔽字，因此自然是可以屏蔽的所有信号的，然后执行信号安装函数再解除各自的所有信号屏蔽字，此时，所有未决的信号将会重新被投送。
在停机时，PG的机制则比较复杂了。Postmaster将最先收到停机信号，然后它将会设置标志位PM_STOP_BACKEND，使用SIGTERM结束所有子进程。然后进入PM_WAIT_BACKEND阶段继续阻塞，子进程清理完整后会通过发送SIGCHILD信号，此时Postmaster设置状态清理子进程资源，随后更新BackendList，当所有的子进程清理完之后，Postmaster进入PM_SHUTDOWN状态。

## openGauss如何使用信号

众所周知，笔者目前还是在给OG做开发，所以写点OG的解析。

相比PG，OG改造成了多线程环境，在多线程环境下，使用信号将变得更为复杂。主要的原因是对于一个进程，同一个信号只能有一个信号处理函数。因此对于不同的线程，我们需要一种方式调度到不同的信号处理函数。OG的实现是使用一个固定的线程sig_monitor处理信号，其他线程屏蔽除了SIGUSR2外的任何信号，当信号到来时，即使这些信号被屏蔽，sig_moniter也可以使用sigwait函数接受信号，然后放进信号队列中，然后发送SIGUSR2信号唤醒主线程，由主线程通过SIGUSR1信号
唤醒工作线程，然后工作线程前往队列中获取外部发过来的信号（例如SIGTERM，SIGINT）。

### 信号模拟机制

openGauss中使用信号模拟机制来实现多种信号的传输，基于上述描述，假如现在从外部来了SIGINT信号，OG的gs_signal_send就会把SIGINT装入Postmaster的信号槽中，这就相当于是OS中信号队列，然后发送SIGUSR2唤醒Postmaster线程（我们将这个线程视为主线程），然后Postmaster线程就可以从它的信号槽中获取到SIGINT信号并执行对应的回调函数。
这就是信号模拟机制。通过多级信号模拟机制，OG可以方便的将信号传递到Postmaster线程和工作线程中。需要注意的是，在OG的信号处理函数执行前会阻塞SIGUSR2信号，换句话说，执行信号处理函数时不会触发新的中断。

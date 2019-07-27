# 实验内容
- 阅读理解task_struct数据结构http://codelab.shiyanlou.com/xref/linux-3.18.6/include/linux/sched.h#1235；

- 分析fork函数对应的内核处理过程sys\_clone，理解创建一个新进程如何创建和修改task\_struct 的数据结构；

- 使用gdb跟踪分析一个fork系统调用内核处理函数sys\_clone ，验证您对Linux系统创建一个新进程的理解,推荐在实验楼Linux虚拟机环境下完成实验。

- 特别关注新进程是从哪里开始执行的？为什么从哪里能顺利执行下去？即执行起点与内核堆栈如何保证一致。

## 2.启动MenuOS

打开shell终端，执行以下命令：

```
cd LinuxKernel
rm -rf menu
git clone https://github.com/mengning/menu.git
cd menu
mv test_fork.c test.c
make rootfs
```

![这里写图片描述](http://img.blog.csdn.net/20170403141500899?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXE0NzA4Njk4NTI=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

## 3.调试MenuOS

通过增加-s -S启动参数打开调试模式

```
qemu -kernel linux-3.18.6/arch/x86/boot/bzImage -initrd rootfs.img -s -S
```

打开gdb进行远程调试

```
gdb

file linux-3.18.6/vmlinux

target remote:1234
```

![这里写图片描述](http://img.blog.csdn.net/20170403141601078?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXE0NzA4Njk4NTI=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

设置断点

```
b sys_clone

b do_fork

b dup_task_struct

b copy_process

b copy_thread

b ret_from_fork
```

![这里写图片描述](http://img.blog.csdn.net/20170403141637422?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXE0NzA4Njk4NTI=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)


在系统启动过程走了些弯路，设置断点的这几个函数都是系统创建进程所必须的关键部分，所以启动过程中do\_fork,copy_process，copy_thread不断的多次出现，我只好暂时使断点失效，才让menuos顺利启动到命令提示符。disable breakpoints 1 2 3 4 5 6.
执行fork命令，停在了断点SyS\_clone处，单步执行，定在了断点do_fork处。经过几行代码，

![这里写图片描述](http://img.blog.csdn.net/20170403141826860?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXE0NzA4Njk4NTI=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

p = copy\_process(clone_flags, stack_start,stack_size,chile_tidptr, NULL,trace);
停在断点copy_process.在执行几行代码，如下：
p = dup\_task_struct(current);
继续执行，断点arch_dup_task_struct停住。
连续n命令后可以看到子进程的初始化过程：

 ![这里写图片描述](http://img.blog.csdn.net/20170403142141392?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXE0NzA4Njk4NTI=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

程序执行断在copy_thread后如下：

![这里写图片描述](http://img.blog.csdn.net/20170403142215410?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXE0NzA4Njk4NTI=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)


执行finish,及continue命令，进入了子进程执行的起点ret_from_fork.

![这里写图片描述](http://img.blog.csdn.net/20170403142300737?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvcXE0NzA4Njk4NTI=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast) 

执行c命令，主要过程基本结束。


# 代码分析

## 1.进程描述

进程描述符（task_struct）

用来描述进程的数据结构，可以理解为进程的属性。比如进程的状态、进程的标识（PID）等，都被封装在了进程描述符这个数据结构中，该数据结构被定义为task_struct

进程控制块（PCB）

是操作系统核心中一种数据结构，主要表示进程状态。

## 2.fork一个子进程的代码

```
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
int main(int argc, char * argv[])
{
    int pid;
    /* fork another process */
    pid = fork();
    if (pid < 0) 
    { 
        /* error occurred */
        fprintf(stderr,"Fork Failed!");
        exit(-1);
    } 
    else if (pid == 0) 
    {
        /* child process */
        printf("This is Child Process!\n");
    } 
    else 
    { 
        /* parent process  */
        printf("This is Parent Process!\n");
        /* parent will wait for the child to complete*/
        wait(NULL);
        printf("Child Complete!\n");
    }
}
```

## 3.进程创建

### 大致流程

fork 通过0x80中断（系统调用）来陷入内核，由系统提供的相应系统调用来完成进程的创建。

fork.c

```
//fork
#ifdef __ARCH_WANT_SYS_FORK
SYSCALL_DEFINE0(fork)
{
#ifdef CONFIG_MMU
    return do_fork(SIGCHLD, 0, 0, NULL, NULL);
#else
    /* can not support in nommu mode */
    return -EINVAL;
#endif
}
#endif

//vfork
#ifdef __ARCH_WANT_SYS_VFORK
SYSCALL_DEFINE0(vfork)
{
    return do_fork(CLONE_VFORK | CLONE_VM | SIGCHLD, 0,
            0, NULL, NULL);
}
#endif

//clone
#ifdef __ARCH_WANT_SYS_CLONE
#ifdef CONFIG_CLONE_BACKWARDS
SYSCALL_DEFINE5(clone, unsigned long, clone_flags, unsigned long, newsp,
         int __user *, parent_tidptr,
         int, tls_val,
         int __user *, child_tidptr)
#elif defined(CONFIG_CLONE_BACKWARDS2)
SYSCALL_DEFINE5(clone, unsigned long, newsp, unsigned long, clone_flags,
         int __user *, parent_tidptr,
         int __user *, child_tidptr,
         int, tls_val)
#elif defined(CONFIG_CLONE_BACKWARDS3)
SYSCALL_DEFINE6(clone, unsigned long, clone_flags, unsigned long, newsp,
        int, stack_size,
        int __user *, parent_tidptr,
        int __user *, child_tidptr,
        int, tls_val)
#else
SYSCALL_DEFINE5(clone, unsigned long, clone_flags, unsigned long, newsp,
         int __user *, parent_tidptr,
         int __user *, child_tidptr,
         int, tls_val)
#endif
{
    return do_fork(clone_flags, newsp, 0, parent_tidptr, child_tidptr);
}
#endif
```

通过看上边的代码，我们可以清楚的看到，不论是使用 fork 还是 vfork 来创建进程，最终都是通过 do\_fork() 方法来实现的。接下来我们可以追踪到 do\_fork()的代码（部分代码，经过精简）：

```
long do_fork(unsigned long clone_flags,
          unsigned long stack_start,
          unsigned long stack_size,
          int __user *parent_tidptr,
          int __user *child_tidptr)
{
        //创建进程描述符指针
        struct task_struct *p;

        //……

        //复制进程描述符，copy_process()的返回值是一个 task_struct 指针。
        p = copy_process(clone_flags, stack_start, stack_size,
             child_tidptr, NULL, trace);

        if (!IS_ERR(p)) {
            struct completion vfork;
            struct pid *pid;

            trace_sched_process_fork(current, p);

            //得到新创建的进程描述符中的pid
            pid = get_task_pid(p, PIDTYPE_PID);
            nr = pid_vnr(pid);

            if (clone_flags & CLONE_PARENT_SETTID)
                put_user(nr, parent_tidptr);

            //如果调用的 vfork()方法，初始化 vfork 完成处理信息。
            if (clone_flags & CLONE_VFORK) {
                p->vfork_done = &vfork;
                init_completion(&vfork);
                get_task_struct(p);
            }

            //将子进程加入到调度器中，为其分配 CPU，准备执行
            wake_up_new_task(p);

            //fork 完成，子进程即将开始运行
            if (unlikely(trace))
                ptrace_event_pid(trace, pid);

            //如果是 vfork，将父进程加入至等待队列，等待子进程完成
            if (clone_flags & CLONE_VFORK) {
                if (!wait_for_vfork_done(p, &vfork))
                    ptrace_event_pid(PTRACE_EVENT_VFORK_DONE, pid);
            }

            put_pid(pid);
        } else {
            nr = PTR_ERR(p);
        }
        return nr;
}
```

### do_fork 流程

1. 调用 copy_process 为子进程复制出一份进程信息

2. 如果是 vfork 初始化完成处理信息

3. 调用 wake_up_new_task 将子进程加入调度器，为之分配 CPU

4. 如果是 vfork，父进程等待子进程完成 exec 替换自己的地址空间


### copy_process 流程

```
static struct task_struct *copy_process(unsigned long clone_flags,
                    unsigned long stack_start,
                    unsigned long stack_size,
                    int __user *child_tidptr,
                    struct pid *pid,
                    int trace)
{
    int retval;

    //创建进程描述符指针
    struct task_struct *p;

    //……

    //复制当前的 task_struct
    p = dup_task_struct(current);

    //……

    //初始化互斥变量   
    rt_mutex_init_task(p);

    //检查进程数是否超过限制，由操作系统定义
    if (atomic_read(&p->real_cred->user->processes) >=
            task_rlimit(p, RLIMIT_NPROC)) {
        if (p->real_cred->user != INIT_USER &&
            !capable(CAP_SYS_RESOURCE) && !capable(CAP_SYS_ADMIN))
            goto bad_fork_free;
    }

    //……

    //检查进程数是否超过 max_threads 由内存大小决定
    if (nr_threads >= max_threads)
        goto bad_fork_cleanup_count;

    //……

    //初始化自旋锁
    spin_lock_init(&p->alloc_lock);
    //初始化挂起信号
    init_sigpending(&p->pending);
    //初始化 CPU 定时器
    posix_cpu_timers_init(p);


    //……

    //初始化进程数据结构，并把进程状态设置为 TASK_RUNNING
    retval = sched_fork(clone_flags, p);

    //复制所有进程信息，包括文件系统、信号处理函数、信号、内存管理等
    if (retval)
        goto bad_fork_cleanup_policy;

    retval = perf_event_init_task(p);
    if (retval)
        goto bad_fork_cleanup_policy;
    retval = audit_alloc(p);
    if (retval)
        goto bad_fork_cleanup_perf;
    /* copy all the process information */
    shm_init_task(p);
    retval = copy_semundo(clone_flags, p);
    if (retval)
        goto bad_fork_cleanup_audit;
    retval = copy_files(clone_flags, p);
    if (retval)
        goto bad_fork_cleanup_semundo;
    retval = copy_fs(clone_flags, p);
    if (retval)
        goto bad_fork_cleanup_files;
    retval = copy_sighand(clone_flags, p);
    if (retval)
        goto bad_fork_cleanup_fs;
    retval = copy_signal(clone_flags, p);
    if (retval)
        goto bad_fork_cleanup_sighand;
    retval = copy_mm(clone_flags, p);
    if (retval)
        goto bad_fork_cleanup_signal;
    retval = copy_namespaces(clone_flags, p);
    if (retval)
        goto bad_fork_cleanup_mm;
    retval = copy_io(clone_flags, p);

    //初始化子进程内核栈
    retval = copy_thread(clone_flags, stack_start, stack_size, p);

    //为新进程分配新的 pid
    if (pid != &init_struct_pid) {
        retval = -ENOMEM;
        pid = alloc_pid(p->nsproxy->pid_ns_for_children);
        if (!pid)
            goto bad_fork_cleanup_io;
    }

    //设置子进程 pid 
    p->pid = pid_nr(pid);


    //……


    //返回结构体 p
    return p;
}
```

追踪copy_process 代码（部分）

1. 调用 dup\_task_struct 复制当前的 task_struct

2. 检查进程数是否超过限制

3. 初始化自旋锁、挂起信号、CPU 定时器等

4. 调用 sched_fork 初始化进程数据结构，并把进程状态设置为 TASK_RUNNING

5. 复制所有进程信息，包括文件系统、信号处理函数、信号、内存管理等

6. 调用 copy_thread 初始化子进程内核栈

7. 为新进程分配并设置新的pid


### dup\_task_struct 流程

```
static struct task_struct *dup_task_struct(struct task_struct *orig)
{
    struct task_struct *tsk;
    struct thread_info *ti;
    int node = tsk_fork_get_node(orig);
    int err;

    //分配一个 task_struct 节点
    tsk = alloc_task_struct_node(node);
    if (!tsk)
        return NULL;

    //分配一个 thread_info 节点，包含进程的内核栈，ti 为栈底
    ti = alloc_thread_info_node(tsk, node);
    if (!ti)
        goto free_tsk;

    //将栈底的值赋给新节点的栈
    tsk->stack = ti;

    //……

    return tsk;

}
```



1. 调用alloc\_task_struct_node分配一个 task_struct 节点

2. 调用alloc_thread_info_node分配一个 thread_info 节点，其实是分配了一个thread_union联合体,将栈底返回给 ti
	```
	union thread_union {
	   struct thread_info thread_info;
	  unsigned long stack[THREAD_SIZE/sizeof(long)];
	};
	```
3. 最后将栈底的值 ti 赋值给新节点的栈
最终执行完dup\_task_struct之后，子进程除了tsk->stack指针不同之外，全部都一样！


### sched_fork 流程

core.c

```
sched_forkcorint sched_fork(unsigned long clone_flags, struct task_struct *p)
{
    unsigned long flags;
    int cpu = get_cpu();

    __sched_fork(clone_flags, p);

    //将子进程状态设置为 TASK_RUNNING
    p->state = TASK_RUNNING;

    //……

    //为子进程分配 CPU
    set_task_cpu(p, cpu);

    put_cpu();
    return 0;
}
```



可以看到sched\_fork大致完成了两项重要工作：
1. 将子进程状态设置为 TASK_RUNNING
2. 为其分配 CPU


### copy_thread 流程
```
int copy_thread(unsigned long clone_flags, unsigned long sp,
    unsigned long arg, struct task_struct *p)
{
    //获取寄存器信息
    struct pt_regs *childregs = task_pt_regs(p);
    struct task_struct *tsk;
    int err;

    p->thread.sp = (unsigned long) childregs;
    p->thread.sp0 = (unsigned long) (childregs+1);
    memset(p->thread.ptrace_bps, 0, sizeof(p->thread.ptrace_bps));

    if (unlikely(p->flags & PF_KTHREAD)) {
        //内核线程
        memset(childregs, 0, sizeof(struct pt_regs));
        p->thread.ip = (unsigned long) ret_from_kernel_thread;
        task_user_gs(p) = __KERNEL_STACK_CANARY;
        childregs->ds = __USER_DS;
        childregs->es = __USER_DS;
        childregs->fs = __KERNEL_PERCPU;
        childregs->bx = sp; /* function */
        childregs->bp = arg;
        childregs->orig_ax = -1;
        childregs->cs = __KERNEL_CS | get_kernel_rpl();
        childregs->flags = X86_EFLAGS_IF | X86_EFLAGS_FIXED;
        p->thread.io_bitmap_ptr = NULL;
        return 0;
    }

    //将当前寄存器信息复制给子进程
    *childregs = *current_pt_regs();

    //子进程 eax 置 0，因此fork 在子进程返回0
    childregs->ax = 0;
    if (sp)
        childregs->sp = sp;

    //子进程ip 设置为ret_from_fork，因此子进程从ret_from_fork开始执行
    p->thread.ip = (unsigned long) ret_from_fork;

    //……

    return err;
}
```
copy_thread 这段代码为我们解释了两个相当重要的问题！
1. 为什么 fork 在子进程中返回0，原因是childregs->ax = 0;这段代码将子进程的 eax 赋值为0
2. p->thread.ip = (unsigned long) ret_from_fork;将子进程的 ip 设置为 ret_form_fork 的首地址，因此子进程是从 ret_from_fork 开始执行的

# 总结

创建一个新进程在内核中的执行过程大致如下:
1. 使用系统调用Sys\_clone(或fork,vfork)系统调用创建一个新进程，而且都是通过调用do_fork来实现进程的创建；
2. Linux通过复制父进程PCB的task_struct来创建一个新进程，要给新进程分配一个新的内核堆栈;
3. 要修改复制过来的进程数据，比如pid、进程链表等等执行copy_process和copy_thread
4. p->thread.sp = (unsigned long) childregs; //调度到子进程时的内核栈顶
5. p->thread.ip = (unsigned long) ret_from_fork; //调度到子进程时的第一条指令地址





# 第六节 进程的描述和进程的创建

### 本周的主要内容：

1. 如何描述一个进程：进程描述符的数据结构；
2. 如何创建一个进程：内核是如何执行的，以及新创建的进程从哪里开始执行；
3. 使用gdb跟踪新进程的创建过程。

## 进程的描述

操作系统三大功能：

- 进程管理（最核心最基础）
- 内存管理
- 文件系统

#### 进程描述符task_struct数据结构

- task _ struct：为了管理进程，内核必须对每个进程进行清晰的描述，进程描述符提供了内核所需了解的进程信息。struct task_struct数据结构很庞大。

- 进程的状态：Linux进程的状态（就绪态、运行态、阻塞态）与操作系统原理中的描述的进程状态有所不同，比如就绪状态和运行状态都是TASK_RUNNING，当一个进程处于TASK_RUNNING的时候是可运行的，至于有没有运行其中的区别就在于有没有获得CPU的使用权。

- 进程的标示pid：用来标示进程

- 进程描述符task_struct数据结构

  ```
    struct task_struct {
    volatile long state;    /* 进程的运行状态-1 unrunnable, 0 runnable, >0 stopped */
    void *stack;    /*指定了进程的内核堆栈*/
    atomic_t usage;
    unsigned int flags; /* 每一个进程的标识符 */
    unsigned int ptrace;
  
    int on_rq;/运行队列和进程调度相关/
  ```

- 进程调度的链表

  所有进程链表struct list_head tasks;

  ```
    struct list_head{
        struct list_head *next,*prev;
    };
  ```

  内核的双向循环链表的实现方法：一个更简略的双向循环链表。

![img](https://images2015.cnblogs.com/blog/744942/201603/744942-20160330213808441-151745625.jpg)

- 进程的地址空间内存管理相关

  ```
    struct mm_struct *mm, *active_mm;
  ```

- 进程标示（pid）

  ```
    pid_t pid;
    pid_t tgid;
  ```

- 进程父子关系

  程序创建的进程具有父子关系，在编程时往往需要引用这样的父子关系。进程描述符中有几个域用来表示这样的关系

![img](https://images2015.cnblogs.com/blog/744942/201603/744942-20160330213823004-2032407843.jpg)

```
    struct task_struct __rcu *real_parent; /* real parent process */
    struct task_struct __rcu *parent; /* recipient of SIGCHLD, wait4() reports */

    struct list_head children;  /* list of my children */
    struct list_head sibling;   /* linkage in my parent's children list */
    struct task_struct *group_leader;   /* threadgroup leader */
```

- 调试

  ```
    struct list_head ptraced;
    struct list_head ptrace_entry;
  ```

- 当前任务相关的CPU状态

  ```
    struct thread_struct thread;
  ```

  Linux为每个进程分配一个8KB大小的内存区域，用于存放该进程两个不同的数据结构：Thread_info和进程的内核堆栈

## 进程的创建

#### 进程的创建

start _ kernel代码中的rest _ init创建两个内核线程，kernel _ init和kthreadd。kernel _ init将用户态进程init启动,是所有用户态进程的祖先，kthreadd是所有内核线程的祖先。

在命令行下创建进程和启动内核的原理是大致相同的，复制一份0号进程描述符，然后根据进程需要将pid等数据结构修改，就完成了进程的创建。

#### fork一个进程的用户态代码

```
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
int main(int argc, char * argv[])
{
    int pid;
    /* fork another process */
    pid = fork();//fork是在用户态用于创建一个子进程的系统调用
    if (pid < 0) 
    { 
        /* error occurred */
        fprintf(stderr,"Fork Failed!");
        exit(-1);
    } 
    else if (pid == 0) 
    {
        /* child process */
        printf("This is Child Process!\n");
    } 
    else 
    {  
        /* parent process  */
        printf("This is Parent Process!\n");
        /* parent will wait for the child to complete*/
        wait(NULL);
        printf("Child Complete!\n");
    }
}
```

fork系统调用在父进程和子进程各返回一次，在子进程中返回的pid是0，父进程中的返回值子进程的pid，所以程序中的else if(pid==0)和else都会被执行，它的背后有两个进程，可以利用这一点在用户态创建子进程。

#### 创建一个新进程在内核中的执行过程

系统调用再回顾：见第四节博客<http://www.cnblogs.com/July0207/p/5277774.html>

fork、vfork和clone三个系统调用都可以创建一个新进程，而且都是通过调用do_fork来实现进程的创建

Linux通过复制父进程来创建一个新进程，大部分信息都是相同的，少数信息需要修改，否则就会发生混乱。例如：pid、进程链表、内核堆栈、进程执行上下文thread等，并设置好fork返回的下一条指令（esp和eip）。由此得出进程创建的一个大致框架：

- 复制一个PCB——task_struct

  ```
    p = dup_task_struct(current);//复制进程的PCB
  
    int __weak arch_dup_task_struct(struct task_struct *dst,struct task_struct *src)
    {
        *dst = *src;//通过赋值实现复制
        return 0;
    }
  ```

- 给新进程分配一个新的内核堆栈

  ```
    ti = alloc_thread_info_node(tsk, node);
    tsk->stack = ti;
    setup_thread_stack(tsk, orig); //这里只是复制thread_info，而非复制内核堆栈
  ```

- 修改复制过来的进程数据，比如pid、进程链表等（见copy_process内部）。

  ```
    /*copy_thread in copy_process*/
    /*拷贝内核堆栈数据和指定新进程的第一条指令地址*/
    *childregs = *current_pt_regs(); //复制内核堆栈，只复制了SAVE_ALL相关的部分
    childregs->ax = 0; //子进程的fork返回0的原因
  
    p->thread.sp = (unsigned long) childregs; //调度到子进程时的内核栈顶
    p->thread.ip = (unsigned long) ret_from_fork; //调度到子进程时的第一条指令地址
  ```

#### 创建的新进程开始启动的位置

int指令和SAVE_ALL压到内核栈的内容：系统调用压栈的相关数据结构，系统调用号，寄存器参数。

当子进程获得CPU的控制权开始运行的时候，ret _ form _ fork可以将后面的堆栈出栈，从iret返回到用户态，从而切换到子进程的用户空间。

## 实验——使用gdb跟踪创建新进程的过程

1. 将menu目录删除，利用git命令克隆一个新的menu目录。

   ```
    rm menu -rf
    git clone https://github.com/mengning/menu.git
   ```

2. 用test_fork.c将test.c覆盖，然后重新编译footfs

   ```
    mv test_fork.c test.c
    make rootfs
   ```

   可以看到fork函数的运行结果：

![img](https://images2015.cnblogs.com/blog/744942/201603/744942-20160330213901160-936689595.jpg)

1. gdb准备调试，加载符号表并设置端口。

   ```
    file linux-3.18.6/vmlinux
    target remote:1234
   ```

2. 设置断点：

![img](https://images2015.cnblogs.com/blog/744942/201603/744942-20160330214120723-1176671566.jpg)

1. 单步执行：

![img](https://images2015.cnblogs.com/blog/744942/201603/744942-20160330213934410-1348415283.jpg)

（由于这里报错原因不明，没有办法进行单步调试，后续的实验在实验楼环境下完成。）

（1）使用命令c继续执行，可看到执行到do_fork处

![img](https://images2015.cnblogs.com/blog/744942/201603/744942-20160330213948269-86906066.jpg)

（2）单步执行到copy_process

![img](https://images2015.cnblogs.com/blog/744942/201603/744942-20160330214008519-288576751.jpg)

（3）之后进入dup _ task _ struct

![img](https://images2015.cnblogs.com/blog/744942/201603/744942-20160330214136504-1565659528.jpg)

（4）继续执行可到copy_thread

![img](https://images2015.cnblogs.com/blog/744942/201603/744942-20160330214209754-1864889611.jpg)

（5）之后可跟踪到ret _ form _ fork，不能继续跟踪执行，结束调试。

总结：通过使用gdb跟踪创建新进程的过程，可以看出创建一个新进程的过程如下：

- 首先调用do_fork函数（函数功能是fork一个新的进程）；
- do _ fork中的copy _ process函数复制父进程的相关信息
- dup _ task _ struct()复制进程的PCB
- thread相关代码给新进程分配一个新的内核堆栈，copy _ thread复制内核堆栈，只复制SAVE _ ALL相关的部分
- 设置sp调度到子进程时的内核栈顶，ip转到子进程时的第一条指令地址
- 当子进程获得CPU的控制权开始运行的时候，ret _ form _ fork可以将后面的堆栈出栈，从iret返回到用户态，从而切换到子进程的用户空间，完成新进程的创建。

# 参考资料

http://www.jianshu.com/p/a89f622e64ea

http://blog.csdn.net/renwotao2009/article/details/51472435

http://blog.csdn.net/sinat_34144680/article/details/51016244

http://blog.sina.com.cn/s/blog_6a0236970102vh6w.html

http://m.blog.csdn.net/article/details?id=51043739

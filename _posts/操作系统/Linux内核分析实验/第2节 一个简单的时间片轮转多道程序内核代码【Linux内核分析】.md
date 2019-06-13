# 实验要求

完成一个简单的时间片轮转多道程序内核代码。

分析进程的启动和进程的切换机制

通过本讲的学习和实验，我们知道操作系统的核心功能就是：进程调度和中断机制，通过与硬件的配合实现多任务处理，再加上上层应用软件的支持，最终变成可以使用户可以很容易操作的计算机系统。  

运行之前

![](img/3818001488814305536-wm.png)



![](img/3818001488814426432-wm.png)





之后：

[这儿](https://github.com/mengning/mykernel)共拷贝了三个文件，将其放置在实验环境中的LinuxKernel/linux-3.9.4/mykernel中。

> mypcb.h
> mymain.c 
> myinterrupt.c

# 代码分析

### mypcb.h

```c
#define MAX_TASK_NUM 10 //最大进程数
#define KERNEL_STACK_SIZE 1024*8  //进程堆栈大小
#define PRIORITY_MAX 30 //priority range from 0 to 30

/* CPU-specific state of this task */
struct Thread {
    unsigned long	ip;//point to cpu run address，用于保存eip
    unsigned long	sp;//point to the thread stack's top address， 用于保存esp
};

typedef struct PCB{ // process control block用于表示一个进程,定义了进程管理相关的数据结构
    int pid; // pcb id 
    volatile long state;	/* -1 unrunnable, 0 runnable, >0 stopped */
    char stack[KERNEL_STACK_SIZE];// each pcb stack size is 1024*8
    
    /* CPU-specific state of this task */
    struct Thread thread;
    unsigned long	task_entry;//the task execute entry memory address，进程入口地址
    struct PCB *next;//pcb is a circular linked list
    unsigned long priority;// task priority ////////
}tPCB;

void my_schedule(void); //调用了my_schedule，表示调度器
```

mypcb.h定义了

- 最大进程数 MAX_TASK_NUM 
即有MAX_TASK_NUM个进程参与内核的时间片轮转调度。
- 内核进程栈的大小 KERNEL_STACK_SIZE
即每一个进程可以使用的堆栈大小。
- 线程Thread
包括指令指针ip和栈顶指针sp。
- 进程控制块PCB
包括进程编号、进程状态state、进程堆栈、线程、任务实体、下一个进程指针
- 调度函数my_schedule
用于模拟进程执行一段时间后切换到其他进程继续执行的过程


### mymain.c

```c
#include <linux/types.h>
#include <linux/string.h>
#include <linux/ctype.h>
#include <linux/tty.h>
#include <linux/vmalloc.h>
#include "mypcb.h"

tPCB task[MAX_TASK_NUM];
tPCB * my_current_task = NULL;
volatile int my_need_sched = 0;//用来判断是否需要调度的标识

void my_process(void);

void __init my_start_kernel(void){
    int pid = 0;
    int i;
    
    /* Initialize process 0 （初始化0号进程）*/
    task[pid].pid = pid;
    task[pid].state = 0; /* -1 unrunnable, 0 runnable, >0 stopped */
    task[pid].task_entry = task[pid].thread.ip = (unsigned long)my_process;//定义0号进程的入口：myprocess
    task[pid].thread.sp = (unsigned long)&task[pid].stack[KERNEL_STACK_SIZE-1]; //定义栈基质，由于栈向下扩展，基址为栈顶地址
    task[pid].next = &task[pid];//由于0号进程初始化时只有这一个进程，所以next指向自己
    
    /*fork more process （创建更多其他的进程）*/
    for(i=1;i<MAX_TASK_NUM;i++){
        memcpy(&task[i],&task[0],sizeof(tPCB)); //开辟内存空间
        task[i].pid = i;
        task[i].state = -1;
        task[i].thread.sp = (unsigned long)&task[i].stack[KERNEL_STACK_SIZE-1];
        //每个进程都有自己的堆栈，把创建好的新进程放到进程列表的尾部，这样就完成了创建
        task[i].next = task[i-1].next;
        task[i-1].next = &task[i];
    }
    
    /* start process 0 by task[0] */
    pid = 0;
    my_current_task = &task[pid];
    asm volatile(
        //%0表示参数thread.ip，%1表示参数thread.sp。
        "movl %1,%%esp\n\t"     /* set task[pid].thread.sp to esp 把参数thread.sp放到esp中*/
        "pushl %1\n\t"          /* push ebp 由于当前栈是空的，esp与ebp指向相同，所以等价于push ebp*/
        "pushl %0\n\t"          /* push task[pid].thread.ip */
        "ret\n\t"               /* pop task[pid].thread.ip to eip */
        "popl %%ebp\n\t"
        : 
        : "c" (task[pid].thread.ip),"d" (task[pid].thread.sp)   
        /* input c or d mean %ecx/%edx*/
    );
}
/*  	%0表示参数thread.ip，%1表示参数thread.sp。
        movl %1,%%esp表示把参数thread.sp放到esp中；
        接下来push %1，又因为当前栈为空，esp=ebp，所以等价于push ebp；
        然后push thread.ip；ret等价于pop thread.ip；最后pop ebp 		 */ 

void my_process(void){ //定义所有进程的工作,if语句表示循环1000万次才有机会判断是否需要调度。
    int i = 0;
    while(1){
        i++;
        if(i%10000000 == 0){
            printk(KERN_NOTICE "this is process %d -\n",my_current_task->pid);
            if(my_need_sched == 1){
                my_need_sched = 0;
                my_schedule();
            }
            printk(KERN_NOTICE "this is process %d +\n",my_current_task->pid);
        }     
    }
}
```
mymain.c中，主要是进程的初始化并负责启动进程，做了三件事

1. 初始化一个进程
为其分配进程编号、进程状态state、进程堆栈、线程、任务实体等，并将其next指针指向自己。
2. 初始化更多的进程
根据第一个进程的部分资源，包括内存拷贝函数的运用，将0号进程的信息进行了复制，修改pid等信息。
3. 设置当前进程
因为是初始化，所以当前进程就决定给0号进程了，通过执行嵌入式汇编代码，开始执行mykernel内核。

重点分析一下嵌入式汇编代码：
```assembly
asm volatile(
        "movl %1,%%esp\n\t"     /* set task[pid].thread.sp to esp */
        "pushl %1\n\t"          /* push ebp */
        "pushl %0\n\t"          /* push task[pid].thread.ip */
        "ret\n\t"               /* pop task[pid].thread.ip to eip */
        "popl %%ebp\n\t"
        : 
        : "c" (task[pid].thread.ip),"d" (task[pid].thread.sp)   /* input c or d mean %ecx/%edx*/
    );
```

首先解释一下含有冒号的部分，因为这一部分我之前不太明白。"c"就是ecx，内容为task[pid].thread.ip，"d"就是edx，内容为task[pid].thread.sp。从冒号开始，以逗号分隔开，从0开始，每一个寄存器有一个标号，即ecx的标号为%0，edx的标号为%1。

接着对上面的汇编代码进行分析，从第二行开始，第二行将esp置为标号%1，即将**esp指向当前进程的栈顶**；第三行将标号%1压栈，因为上一句就是当前进程的esp被压栈，此时栈为空，当前ebp被压栈，esp下移一位；第四行将标号%0压栈，即当前进程的指令指针eip压栈，第五行将eip指向当前进程指针，开始执行当前进程。

然后实现了my_process函数，这是一个死循环，每个进程都是执行此函数。每10000000次，打印当前进程的pid，全局变量my_need_sched，通过对my_need_sched进行判断，若为1，则通知正在执行的进程执行调度程序，然后打印调度后的进程pid。


### myinterrupt.c
```c
#include <linux/types.h>
#include <linux/string.h>
#include <linux/ctype.h>
#include <linux/tty.h>
#include <linux/vmalloc.h>
#include "mypcb.h"

//extern 引用其他模块定义变量
extern tPCB task[MAX_TASK_NUM]; 
extern tPCB * my_current_task;
extern volatile int my_need_sched; //volatile 全局可见
volatile int time_count = 0;

/*
* Called by timer interrupt.
* it runs in the name of current running process,
* so it use kernel stack of current running process
*/
/*  用于设置时间片的大小，时间片用完时设置调度标志。
    当时钟中断发生1000次，并且my_need_sched!=1时，把my_need_sched赋为1。
    当进程发现my_need_sched=1时，就会执行my_schedule。  */
void my_timer_handler(void){//用于设置时间片的大小，时间片用完时设置调度标志。
#if 1
    if(time_count%1000 == 0 && my_need_sched != 1){
        printk(KERN_NOTICE ">>>my_timer_handler here<<<\n");
        my_need_sched = 1;
    } 
    time_count ++ ;  
#endif
    return;     
}

void my_schedule(void){
    tPCB * next;
    tPCB * prev;

    if(my_current_task == NULL //task为空，即发生错误时返回
   		 || my_current_task->next == NULL)
    {
        return;
    }
    printk(KERN_NOTICE ">>>my_schedule<<<\n");
    /* schedule */
    next = my_current_task->next;//把当前进程的下一个进程赋给next
    prev = my_current_task;//当前进程为prev
    if(next->state == 0)/* -1 unrunnable, 0 runnable, >0 stopped */
    {
        /* switch to next process */
        /*如果下一个进程的状态是正在执行的话，就运用if语句中的代码表示的方法来切换进程*/
        asm volatile(   
            "pushl %%ebp\n\t"       /* save ebp 保存当前进程的ebp*/
            "movl %%esp,%0\n\t"     /* save esp 把当前进程的esp赋给%0（指的是thread.sp），即保存当前进程的esp*/
            "movl %2,%%esp\n\t"     /* restore  esp 把%2（指下一个进程的sp）放入esp中*/
            "movl $1f,%1\n\t"       /* save eip $1f是接下来的标号“1：”的位置，把eip保存下来*/ 
            "pushl %3\n\t"          /*把下一个进程eip压栈*/
            "ret\n\t"               /* restore  eip 下一个进程开始执行*/
            "1:\t"                  /* next process start here */
            "popl %%ebp\n\t"
            : "=m" (prev->thread.sp),"=m" (prev->thread.ip)
            : "m" (next->thread.sp),"m" (next->thread.ip)
        ); 
        my_current_task = next; 
        printk(KERN_NOTICE ">>>switch %d to %d<<<\n",prev->pid,next->pid);      
    }
    else/*  与上一段代码不同的是如果下一个进程为新进程时，就运用else中的这一段代码。
        首先将这个进程置为运行时状态，将这个进程作为当前正在执行的进程。    */
    {
        next->state = 0;
        my_current_task = next;
        printk(KERN_NOTICE ">>>switch %d to %d<<<\n",prev->pid,next->pid);
        /* switch to new process */
        asm volatile(   
            "pushl %%ebp\n\t"       /* save ebp */
            "movl %%esp,%0\n\t"     /* save esp */
            "movl %2,%%esp\n\t"     /* restore  esp */
            "movl %2,%%ebp\n\t"     /* restore  ebp */
            "movl $1f,%1\n\t"       /* save eip */  
            "pushl %3\n\t"          /*把当前进程的入口保存起来*/
            "ret\n\t"               /* restore  eip */
            : "=m" (prev->thread.sp),"=m" (prev->thread.ip)
            : "m" (next->thread.sp),"m" (next->thread.ip)
        );          
    }   
    return; 
}
```

### 定时中断函数
实现比较简单，就是实现定时器中断，每1000下进行my_need_sched的检查，如果不为1，则置其为1使其进程调度。

my_schedule函数具体实现了进程的切换。声明了两个指针，prev和next，分别指向当前进程和下一个进程。进程切换时分两种情况，当next_state ==0 时，即下一个进程正在执行。

在my_schedule函数中，完成进程的切换。进程的切换分两种情况，一种情况是下一个进程没有被调度过，另外一种情况是下一个进程被调度过，可以通过下一个进程的state知道其状态。进程切换依然是通过内联汇编代码实现，无非是保存旧进程的eip和堆栈，将新进程的eip和堆栈的值存入对应的寄存器中。

```c
/* switch to next process */
asm volatile(	
    "pushl %%ebp\n\t" 	    /* save ebp */
    "movl %%esp,%0\n\t" 	/* save esp */
    "movl %2,%%esp\n\t"     /* restore  esp */
    "movl $1f,%1\n\t"       /* save eip */	
    "pushl %3\n\t" 
    "ret\n\t" 	            /* restore  eip */
    "1:\t"                  /* next process start here */
    "popl %%ebp\n\t"
    : "=m" (prev->thread.sp),"=m" (prev->thread.ip)
    : "m" (next->thread.sp),"m" (next->thread.ip)
```
上边代码实现了进程的切换，第三行将当前的ebp保存压栈，然后将当前的esp存入内存中（prev-&gt;thread.sp）；第四行将ebp指向要切换进程的esp；第五行保存当前的eip到pre-&gt;thread.ip；第六行将下一个进程的eip压栈；第七行ret实现将栈顶，即刚刚压栈的eip弹出，赋给eip，程序开始从此处执行（即要切换的进程），完成了进程切换。

当下一个进程状态不为0时，即表示还未执行，此时esp等与ebp，其余部分和第一种情况相同。

## 环境配置

- Set up this platform
  - sudo apt-get install qemu # install QEMU
  - sudo ln -s /usr/bin/qemu-system-i386 /usr/bin/qemu
  - wget <https://www.kernel.org/pub/linux/kernel/v3.x/linux-3.9.4.tar.xz> # download [Linux Kernel 3.9.4 source code](https://www.kernel.org/pub/linux/kernel/v3.x/linux-3.9.4.tar.xz)
  - wget <https://raw.github.com/mengning/mykernel/master/mykernel_for_linux3.9.4sc.patch> # download [mykernel_for_linux3.9.4sc.patch](https://raw.github.com/mengning/mykernel/master/mykernel_for_linux3.9.4sc.patch)
  - xz -d linux-3.9.4.tar.xz
  - tar -xvf linux-3.9.4.tar
  - cd linux-3.9.4
  - patch -p1 < ../mykernel_for_linux3.9.4sc.patch
  - make allnoconfig
  - make
  - qemu -kernel arch/x86/boot/bzImage  #从qemu窗口中您可以看到my_start_kernel在执行，同时my_timer_handler时钟中断处理程序周期性执行。
  - cd mykernel 您可以看到qemu窗口输出的内容的代码mymain.c和myinterrupt.c
  - 当前有一个CPU执行C代码的上下文环境，同时具有中断处理程序的上下文环境，我们初始化好了系统环境。
  - 您只要在mymain.c基础上继续写进程描述PCB和进程链表管理等代码，在myinterrupt.c的基础上完成进程切换代码，一个可运行的小OS kernel就完成了。
  - start to write your own OS kernel,enjoy it!
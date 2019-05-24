# 第四部分: 工具

# 1. GCC

## 1.1 预处理

输出预处理结果到文件。

```
$ gcc -E main.c -o main.i
```

保留文件头注释。

```
$ gcc -C -E main.c -o main.i
```

参数 -Dname 定义宏 (源文件中不能定义该宏)，-Uname 取消 GCC 设置中定义的宏。

```
$ tail -n 10 main.c

int main(int argc, char* argv[])
{
    #if __MY__
    printf("a");
    #else
    printf("b");
    #endif
    return EXIT_SUCCESS;
}

$ gcc -E main.c -D__MY__ | tail -n 10

int main(int argc, char* argv[])
{
    printf("a");
    return 0;
}
```

-Idirectory 设置头文件(.h)的搜索路径。

```
$ gcc -g -I./lib -I/usr/local/include/cbase main.c mylib.c
```

查看依赖文件。

```
$ gcc -M -I./lib main.c
$ gcc -MM -I./lib main.c # 忽略标准库
```

## 1.2 汇编

我们可以将 C 源代码编译成汇编语言 (.s)。

```
$ gcc -S main.c
$ head -n 20 main.s
    .file "main.c"
    .section .rodata
.LC0:
    .string "Hello, World!"
    .text
.globl main
    .type main, @function
main:
    pushl %ebp
    movl %esp, %ebp
    andl $-16, %esp
    subl $16, %esp
    movl $.LC0, (%esp)
    call test
    movl $0, %eax
    leave
    ret
    .size main, .-main
    .ident "GCC: (Ubuntu 4.4.1-4ubuntu9) 4.4.1"
    .section .note.GNU-stack,"",@progbits
```

使用 -fverbose-asm 参数可以获取变量注释。如果需要指定汇编格式，可以使用 "-masm=intel"参数。

## 1.3 链接

参数 -c 仅生成目标文件 (.o)，然后需要调用链接器 (link) 将多个目标文件链接成单一可执行文件。

```
$ gcc -g -c main.c mylib.c
```

参数 -l 链接其他库，比如 -lpthread 链接 libpthread.so。或指定 -static 参数进行静态链接。我们还可以直接指定链接库 (.so, .a) 完整路径。

```
$ gcc -g -o test main.c ./libmy.so ./libtest.a

$ ldd ./test
    linux-gate.so.1 => (0xb7860000)
    ./libmy.so (0xb785b000)
    libc.so.6 => /lib/tls/i686/cmov/libc.so.6 (0xb7710000)
    /lib/ld-linux.so.2 (0xb7861000)
```

另外一种做法就是用 -L 指定库搜索路径。

```
$ gcc -g -o test -L/usr/local/lib -lgdsl main.c

$ ldd ./test
    linux-gate.so.1 => (0xb77b6000)
    libgdsl.so.1 => /usr/local/lib/libgdsl.so.1 (0xb779b000)
    libc.so.6 => /lib/tls/i686/cmov/libc.so.6 (0xb7656000)
    /lib/ld-linux.so.2 (0xb77b7000)
```

## 1.4 动态库

使用 "-fPIC -shared" 参数生成动态库。

```
$ gcc -fPIC -c -O2 mylib.c

$ gcc -shared -o libmy.so mylib.o

$ nm libmy.so
... ...
00000348 T _init
00002010 b completed.6990
00002014 b dtor_idx.6992
... ...
0000047c T test
```

静态库则需要借助 ar 工具将多个目标文件 (.o) 打包。

```
c$ gcc -c mylib.c

$ ar rs libmy.a mylib.o
ar: creating libmy.a
```

## 1.5 优化

参数 -O0 关闭优化 (默认)；-O1 (或 -O) 让可执行文件更小，速度更快；-O2 采用几乎所有的优化手段。

```
$ gcc -O2 -o test main.c mylib.c
```

## 1.6 调试

参数 -g 在对象文件 (.o) 和执行文件中生成符号表和源代码行号信息，以便使用 gdb 等工具进行调试。

```
$ gcc -g -o test main.c mylib.c

$ readelf -S test
There are 38 section headers, starting at offset 0x18a8:

Section Headers:
    [Nr] Name Type Addr Off Size ES Flg Lk Inf Al
    ... ...
    [27] .debug_aranges PROGBITS 00000000 001060 000060 00 0 0 8
    [28] .debug_pubnames PROGBITS 00000000 0010c0 00005b 00 0 0 1
    [29] .debug_info PROGBITS 00000000 00111b 000272 00 0 0 1
    [30] .debug_abbrev PROGBITS 00000000 00138d 00014b 00 0 0 1
    [31] .debug_line PROGBITS 00000000 0014d8 0000f1 00 0 0 1
    [32] .debug_frame PROGBITS 00000000 0015cc 000058 00 0 0 4
    [33] .debug_str PROGBITS 00000000 001624 0000d5 01 MS 0 0 1
    [34] .debug_loc PROGBITS 00000000 0016f9 000058 00 0 0 1
    ... ...
```

参数 -pg 会在程序中添加性能分析 (profiling) 函数，用于统计程序中最耗费时间的函数。程序执行后，统计信息保存在 gmon.out 文件中，可以用 gprof 命令查看结果。

```
$ gcc -g -pg main.c mylib.c
```

# 2. GDB

作为内置和最常用的调试器，GDB 显然有着无可辩驳的地位。熟练使用 GDB，就好像所有 Linux 下的开发人员建议你用 VIM 一样，是个很 "奇怪" 的情节。

测试用源代码。

```
#include <stdio.h>
int test(int a, int b)
{
    int c = a + b;
    return c;
}

int main(int argc, char* argv[])
{
    int a = 0x1000;
    int b = 0x2000;
    int c = test(a, b);
    printf("%d\n", c);

    printf("Hello, World!\n");
    return 0;
}
```

编译命令 (注意使用 -g 参数生成调试符号)：

```
$ gcc -g -o hello hello.c
```

开始调试：

```
$ gdb hello

GNU gdb 6.8-debian
Copyright (C) 2008 Free Software Foundation, Inc.
This GDB was configured as "i486-linux-gnu"...

(gdb)
```

## 2.1 源码

在调试过程中查看源代码是必须的。list (缩写 l) 支持多种方式查看源码。

```
(gdb) l # 显示源代码

2
3 int test(int a, int b)
4 {
5       int c = a + b;
6       return c;
7 }
8
9 int main(int argc, char* argv[])
10 {
11      int a = 0x1000;

(gdb) l # 继续显示

12      int b = 0x2000;
13      int c = test(a, b);
14      printf("%d\n", c);
15
16      printf("Hello, World!\n");
17      return 0;
18 }

(gdb) l 3, 10 # 显示特定范围的源代码

3 int test(int a, int b)
4 {
5       int c = a + b;
6       return c;
7 }
8
9 int main(int argc, char* argv[])
10 {

(gdb) l main # 显示特定函数源代码
5       int c = a + b;
6       return c;
7 }
8
9 int main(int argc, char* argv[])
10 {
11      int a = 0x1000;
12      int b = 0x2000;
13      int c = test(a, b);
14      printf("%d\n", c);
```

可以用如下命令修改源代码显示行数。

```
(gdb) set listsize 50
```

## 2.2 断点

可以使用函数名或者源代码行号设置断点。

```
(gdb) b main # 设置函数断点
Breakpoint 1 at 0x804841b: file hello.c, line 11.

(gdb) b 13 # 设置源代码行断点
Breakpoint 2 at 0x8048429: file hello.c, line 13.

(gdb) b # 将下一行设置为断点 (循环、递归等调试很有用)
Breakpoint 5 at 0x8048422: file hello.c, line 12.

(gdb) tbreak main # 设置临时断点 (中断后失效)
Breakpoint 1 at 0x804841b: file hello.c, line 11.

(gdb) info breakpoints # 查看所有断点

Num Type Disp Enb Address What
2 breakpoint keep y 0x0804841b in main at hello.c:11
3 breakpoint keep y 0x080483fa in test at hello.c:5

(gdb) d 3 # delete: 删除断点 (还可以用范围 "d 1-3"，无参数时删除全部断点)
(gdb) disable 2 # 禁用断点 (还可以用范围 "disable 1-3")
(gdb) enable 2 # 启用断点 (还可以用范围 "enable 1-3")
(gdb) ignore 2 1 # 忽略 2 号中断 1 次
```

当然少不了条件式中断。

```
(gdb) b test if a == 10
Breakpoint 4 at 0x80483fa: file hello.c, line 5.

(gdb) info breakpoints

Num Type Disp Enb Address What
4   breakpoint keep y 0x080483fa in test at hello.c:5
    stop only if a == 10
```

可以用 condition 修改条件，注意表达式不包含 if。

```
(gdb) condition 4 a == 30

(gdb) info breakpoints

Num Type Disp Enb Address What
2   breakpoint keep y 0x0804841b in main at hello.c:11
    ignore next 1 hits
4   breakpoint keep y 0x080483fa in test at hello.c:5
    stop only if a == 30
```

## 2.3 执行

通常情况下，我们会先设置 main 入口断点。

```
(gdb) b main
Breakpoint 1 at 0x804841b: file hello.c, line 11.

(gdb) r # 开始执行 (Run)
Starting program: /home/yuhen/Learn.c/hello
Breakpoint 1, main () at hello.c:11
11 int a = 0x1000;

(gdb) n # 单步执行 (不跟踪到函数内部, Step Over)
12 int b = 0x2000;

(gdb) n
13 int c = test(a, b);

(gdb) s # 单步执行 (跟踪到函数内部, Step In)
test (a=4096, b=8192) at hello.c:5
5 int c = a + b;

(gdb) finish # 继续执行直到当前函数结束 (Step Out)

Run till exit from #0 test (a=4096, b=8192) at hello.c:5
0x0804843b in main () at hello.c:13
13 int c = test(a, b);
Value returned is $1 = 12288

(gdb) c # Continue: 继续执行，直到下一个断点。

Continuing.
12288
Hello, World!

Program exited normally.
```

## 2.4 堆栈

查看调用堆栈无疑是调试过程中非常重要的事情。

```
(gdb) where # 查看调用堆栈 (相同作用的命令还有 info s 和 bt)

#0 test (a=4096, b=8192) at hello.c:5
#1 0x0804843b in main () at hello.c:13

(gdb) frame # 查看当前堆栈帧，还可显示当前代码

#0 test (a=4096, b=8192) at hello.c:5
5 int c = a + b;

(gdb) info frame # 获取当前堆栈帧更详细的信息

Stack level 0, frame at 0xbfad3290:
    eip = 0x80483fa in test (hello.c:5); saved eip 0x804843b
    called by frame at 0xbfad32c0
    source language c.
    Arglist at 0xbfad3288, args: a=4096, b=8192
    Locals at 0xbfad3288, Previous frame's sp is 0xbfad3290
    Saved registers:
    ebp at 0xbfad3288, eip at 0xbfad328c
```

可以用 frame 修改当前堆栈帧，然后查看其详细信息。

```
(gdb) frame 1
#1 0x0804843b in main () at hello.c:13
13 int c = test(a, b);

(gdb) info frame

Stack level 1, frame at 0xbfad32c0:
    eip = 0x804843b in main (hello.c:13); saved eip 0xb7e59775
    caller of frame at 0xbfad3290
    source language c.
    Arglist at 0xbfad32b8, args:
    Locals at 0xbfad32b8, Previous frame's sp at 0xbfad32b4
    Saved registers:
        ebp at 0xbfad32b8, eip at 0xbfad32bc
```

## 2.5 变量和参数

```
(gdb) info locals # 显示局部变量
c = 0

(gdb) info args # 显示函数参数(自变量)
a = 4096
b = 8192
```

我们同样可以切换 frame，然后查看不同堆栈帧的信息。

```
(gdb) p a # print 命令可显示局部变量和参数值
$2 = 4096

(gdb) p/x a # 十六进制输出
$10 = 0x1000

(gdb) p a + b # 还可以进行表达式计算
$5 = 12288
```

x 命令内存输出格式:

- d: 十进制
- u: 十进制无符号
- x: 十六进制
- o: 八进制
- t: 二进制
- c: 字符
- set variable 可用来修改变量值。

```
(gdb) set variable a=100

(gdb) info args
a = 100
b = 8192
```

## 2.6 内存及寄存器

x 命令可以显示指定地址的内存数据。

```
格式: x/nfu [address]
```

- n: 显示内存单位 (组或者行)。
- f: 格式 (除了 print 格式外，还有 字符串 s 和 汇编 i)。
- u: 内存单位 (b: 1字节; h: 2字节; w: 4字节; g: 8字节)。

```
(gdb) x/8w 0x0804843b # 按四字节(w)显示 8 组内存数据

0x804843b <main+49>: 0x8bf04589 0x4489f045 0x04c70424 0x04853024
0x804844b <main+65>: 0xfecbe808 0x04c7ffff 0x04853424 0xfecfe808

(gdb) x/8i 0x0804843b # 显示 8 行汇编指令

0x804843b <main+49>: mov DWORD PTR [ebp-0x10],eax
0x804843e <main+52>: mov eax,DWORD PTR [ebp-0x10]
0x8048441 <main+55>: mov DWORD PTR [esp+0x4],eax
0x8048445 <main+59>: mov DWORD PTR [esp],0x8048530
0x804844c <main+66>: call 0x804831c <printf@plt>
0x8048451 <main+71>: mov DWORD PTR [esp],0x8048534
0x8048458 <main+78>: call 0x804832c <puts@plt>
0x804845d <main+83>: mov eax,0x0

(gdb) x/s 0x08048530 # 显示字符串

0x8048530: "%d\n"
```

除了通过 "info frame" 查看寄存器值外，还可以用如下指令。

```
(gdb) info registers # 显示所有寄存器数据

eax 0x1000 4096
ecx 0xbfad32d0 -1079168304
edx 0x1 1
ebx 0xb7fa1ff4 -1208344588
esp 0xbfad3278 0xbfad3278
ebp 0xbfad3288 0xbfad3288
esi 0x8048480 134513792
edi 0x8048340 134513472
eip 0x80483fa 0x80483fa <test+6>
eflags 0x286 [ PF SF IF ]
cs 0x73 115
ss 0x7b 123
ds 0x7b 123
es 0x7b 123
fs 0x0 0
gs 0x33 51

(gdb) p $eax # 显示单个寄存器数据
$11 = 4096
```

## 2.7 反汇编

我对 AT&T 汇编不是很熟悉，还是设置成 intel 格式的好。

```
(gdb) set disassembly-flavor intel # 设置反汇编格式
(gdb) disass main # 反汇编函数

Dump of assembler code for function main:
0x0804840a <main+0>: lea ecx,[esp+0x4]
0x0804840e <main+4>: and esp,0xfffffff0
0x08048411 <main+7>: push DWORD PTR [ecx-0x4]
0x08048414 <main+10>: push ebp
0x08048415 <main+11>: mov ebp,esp
0x08048417 <main+13>: push ecx
0x08048418 <main+14>: sub esp,0x24
0x0804841b <main+17>: mov DWORD PTR [ebp-0x8],0x1000
0x08048422 <main+24>: mov DWORD PTR [ebp-0xc],0x2000
0x08048429 <main+31>: mov eax,DWORD PTR [ebp-0xc]
0x0804842c <main+34>: mov DWORD PTR [esp+0x4],eax
0x08048430 <main+38>: mov eax,DWORD PTR [ebp-0x8]
0x08048433 <main+41>: mov DWORD PTR [esp],eax
0x08048436 <main+44>: call 0x80483f4 <test>
0x0804843b <main+49>: mov DWORD PTR [ebp-0x10],eax
0x0804843e <main+52>: mov eax,DWORD PTR [ebp-0x10]
0x08048441 <main+55>: mov DWORD PTR [esp+0x4],eax
0x08048445 <main+59>: mov DWORD PTR [esp],0x8048530
0x0804844c <main+66>: call 0x804831c <printf@plt>
0x08048451 <main+71>: mov DWORD PTR [esp],0x8048534
0x08048458 <main+78>: call 0x804832c <puts@plt>
0x0804845d <main+83>: mov eax,0x0
0x08048462 <main+88>: add esp,0x24
0x08048465 <main+91>: pop ecx
0x08048466 <main+92>: pop ebp
0x08048467 <main+93>: lea esp,[ecx-0x4]
0x0804846a <main+96>: ret
End of assembler dump.
```

可以用 "b *address" 设置汇编断点，然后用 si 和 ni 进行汇编级单步执行，这对于分析指针和寻址非常有用。

## 2.8 进程

查看进程相关信息，尤其是 maps 内存数据是非常有用的。

```
(gdb) help info proc stat

Show /proc process information about any running process.
Specify any process id, or use the program being debugged by default.
Specify any of the following keywords for detailed info:
    mappings -- list of mapped memory regions.
    stat -- list a bunch of random process info.
    status -- list a different bunch of random process info.
    all -- list all available /proc info.

(gdb) info proc mappings !# 相当于 cat /proc/{pid}/maps

process 22561
cmdline = '/home/yuhen/Learn.c/hello'
cwd = '/home/yuhen/Learn.c'
exe = '/home/yuhen/Learn.c/hello'
Mapped address spaces:

    Start Addr End Addr Size Offset objfile
    0x8048000 0x8049000 0x1000 0 /home/yuhen/Learn.c/hello
    0x8049000 0x804a000 0x1000 0 /home/yuhen/Learn.c/hello
    0x804a000 0x804b000 0x1000 0x1000 /home/yuhen/Learn.c/hello
    0x8a33000 0x8a54000 0x21000 0x8a33000 [heap]
    0xb7565000 0xb7f67000 0xa02000 0xb7565000
    0xb7f67000 0xb80c3000 0x15c000 0 /lib/tls/i686/cmov/libc-2.9.so
    0xb80c3000 0xb80c4000 0x1000 0x15c000 /lib/tls/i686/cmov/libc-2.9.so
    0xb80c4000 0xb80c6000 0x2000 0x15c000 /lib/tls/i686/cmov/libc-2.9.so
    0xb80c6000 0xb80c7000 0x1000 0x15e000 /lib/tls/i686/cmov/libc-2.9.so
    0xb80c7000 0xb80ca000 0x3000 0xb80c7000
    0xb80d7000 0xb80d9000 0x2000 0xb80d7000
    0xb80d9000 0xb80da000 0x1000 0xb80d9000 [vdso]
    0xb80da000 0xb80f6000 0x1c000 0 /lib/ld-2.9.so
    0xb80f6000 0xb80f7000 0x1000 0x1b000 /lib/ld-2.9.so
    0xb80f7000 0xb80f8000 0x1000 0x1c000 /lib/ld-2.9.so
    0xbfee2000 0xbfef7000 0x15000 0xbffeb000 [stack]
```

## 2.9 线程

可以在 pthread_create 处设置断点，当线程创建时会生成提示信息。

```
(gdb) c

Continuing.
[New Thread 0xb7e78b70 (LWP 2933)]

(gdb) info threads # 查看所有线程列表

* 2 Thread 0xb7e78b70 (LWP 2933) test (arg=0x804b008) at main.c:24
1 Thread 0xb7e796c0 (LWP 2932) 0xb7fe2430 in __kernel_vsyscall ()

(gdb) where # 显示当前线程调用堆栈

#0 test (arg=0x804b008) at main.c:24
#1 0xb7fc580e in start_thread (arg=0xb7e78b70) at pthread_create.c:300
#2 0xb7f478de in clone () at ../sysdeps/unix/sysv/linux/i386/clone.S:130

(gdb) thread 1 # 切换线程

[Switching to thread 1 (Thread 0xb7e796c0 (LWP 2932))]#0 0xb7fe2430 in __kernel_vsyscall ()

(gdb) where # 查看切换后线程调用堆栈

#0 0xb7fe2430 in __kernel_vsyscall ()
#1 0xb7fc694d in pthread_join (threadid=3085405040, thread_return=0xbffff744) at pthread_join.c:89
#2 0x08048828 in main (argc=1, argv=0xbffff804) at main.c:36
```

## 2.10 其他

调试子进程。

```
(gdb) set follow-fork-mode child
```

临时进入 Shell 执行命令，Exit 返回。

```
(gdb) shell
```

调试时直接调用函数。

```
(gdb) call test("abc")
```

使用 "--tui" 参数，可以在终端窗口上部显示一个源代码查看窗。

```
$ gdb --tui hello
```

查看命令帮助。

```
(gdb) help b
```

最后就是退出命令。

```
(gdb) q
```

和 Linux Base Shell 习惯一样，对于记不住的命令，可以在输入前几个字母后按 Tab 补全。

## 2.11 Core Dump

在 Windows 下我们已经习惯了用 Windbg 之类的工具调试 dump 文件，从而分析并排除程序运行时错误。在 Linux 下我们同样可以完成类似的工作 —— Core Dump。

我们先看看相关的设置。

```
$ ulimit -a

core file size (blocks, -c) 0
data seg size (kbytes, -d) unlimited
scheduling priority (-e) 20
file size (blocks, -f) unlimited
pending signals (-i) 16382
max locked memory (kbytes, -l) 64
max memory size (kbytes, -m) unlimited
open files (-n) 1024
pipe size (512 bytes, -p) 8
POSIX message queues (bytes, -q) 819200
real-time priority (-r) 0
stack size (kbytes, -s) 8192
cpu time (seconds, -t) unlimited
max user processes (-u) unlimited
virtual memory (kbytes, -v) unlimited
file locks (-x) unlimited
```

"core file size (blocks, -c) 0" 意味着在程序崩溃时不会生成 core dump 文件，我们需要修改一下设置。如果你和我一样懒得修改配置文件，那么就输入下面这样命令吧。

```
$ sudo sh -c "ulimit -c unlimited; ./test" # test 是可执行文件名。
```

等等…… 我们还是先准备个测试目标。

```
#include <stdio.h>
#include <stdlib.h>
void test()
{
    char* s = "abc";
    *s = 'x';
}

int main(int argc, char** argv)
{
    test();
    return (EXIT_SUCCESS);
}
```

很显然，我们在 test 里面写了一个不该写的东东，这无疑会很严重。生成可执行文件后，执行上面的命令。

```
$ sudo sh -c "ulimit -c unlimited; ./test"

Segmentation fault (core dumped)

$ ls -l

total 96
-rw------- 1 root root 167936 2010-01-06 13:30 core
-rwxr-xr-x 1 yuhen yuhen 9166 2010-01-06 13:16 test
```

这个 core 文件就是被系统 dump 出来的，我们分析目标就是它了。

```
$ sudo gdb test core

GNU gdb (GDB) 7.0-ubuntu
Copyright (C) 2009 Free Software Foundation, Inc.

Reading symbols from .../dist/Debug/test...done.

warning: Can't read pathname for load map: Input/output error.
Reading symbols from /lib/tls/i686/cmov/libpthread.so.0... ...done.
(no debugging symbols found)...done.
Loaded symbols for /lib/tls/i686/cmov/libpthread.so.0
Reading symbols from /lib/tls/i686/cmov/libc.so.6... ...done.
(no debugging symbols found)...done.
Loaded symbols for /lib/tls/i686/cmov/libc.so.6
Reading symbols from /lib/ld-linux.so.2... ...done.
(no debugging symbols found)...done.
Loaded symbols for /lib/ld-linux.so.2

Core was generated by `./test'.
Program terminated with signal 11, Segmentation fault.
#0 0x080483f4 in test () at main.c:16

warning: Source file is more recent than executable.
16 *s = 'x';
```

最后这几行提示已经告诉我们错误的原因和代码位置，接下来如何调试就是 gdb 的技巧了，可以先输入 where 看看调用堆栈。

```
(gdb) where

#0 0x080483f4 in test () at main.c:16
#1 0x08048401 in main (argc=1, argv=0xbfd53e44) at main.c:22

(gdb) p s
$1 = 0x80484d0 "abc"

(gdb) info files

Symbols from ".../dist/Debug/test".
Local core dump file:

Local exec file:
    `.../dist/Debug/test', file type elf32-i386.
    Entry point: 0x8048330
    0x08048134 - 0x08048147 is .interp
    ... ...
    0x08048330 - 0x080484ac is .text
    0x080484ac - 0x080484c8 is .fini
    0x080484c8 - 0x080484d4 is .rodata
```

很显然 abc 属于 .rodata，严禁调戏。

附：如果你调试的是 Release (-O2) 版本，而且删除(strip)了符号表，那还是老老实实数汇编代码吧。可见用 Debug 版本试运行是很重要滴！！！

# 5. Scons

Scons 采用 Python 编写，用来替换 GNU Make 的自动化编译构建工具。相比 Makefile 和类似的老古董，scons 更智能，更简单。

## 5.1 脚本

在项目目录下创建名为 SConstruct (或 Sconstruct、 sconstruct) 的文件，作用类似 Makefile。实质上就是 py 源文件。

简单样本:

```
Program("test", ["main.c"])
```

常用命令：

```
$ scons!! ! ! ! ! # 构建，输出详细信息。
scons: Reading SConscript files ...
scons: done reading SConscript files.
scons: Building targets ...
gcc -o main.o -c main.c
gcc -o test main.o
scons: done building targets.

$ scons -c! ! ! ! ! # 清理，类似 make clean。
scons: Reading SConscript files ...
scons: done reading SConscript files.
scons: Cleaning targets ...
Removed main.o
Removed test
scons: done cleaning targets.

$ scons -Q! ! ! ! ! # 构建，简化信息输出。
gcc -o main.o -c main.c
gcc -o test main.o

$ scons -i! ! ! ! ! # 忽略错误，继续执行。
$ scons -n! ! ! ! ! # 输出要执行的命令，但并不真的执行。
$ scons -s! ! ! ! ! # 安静执行，不输出任何非错误信息。
$ scons -j 2! ! ! ! ! # 并行构建。
```

如需调试，建议插入 "import pdb; pdb.set_trace()"，命令行参数 "--debug=pdb" 并不好用。可用 SConscript(path/filename) 包含其他设置文件 (或列表)，按惯例命名为 SConscript。

## 5.2 环境

影响 scons 执行的环境 (Environment ) 因素包括：

- External：外部环境。执行 scons 时的操作系统环境变量，可以用 os.environ 访问。
- Construction: 构建环境，用来控制实际的编译行为。
- Execution: 执行环境，用于设置相关工具所需设置。比如 PATH 可执行搜索路径。

简单程序，可直接使用默认构建环境实例。

```
env = DefaultEnvironment(CCFLAGS = "-g")! ! # 返回默认构建环境实例，并设置参数。
Program("test", ["main.c"])! ! ! ! # 相当于 env.Program()
```

输出:

```
gcc -o main.o -c -g main.c
gcc -o test main.o
```

如需多个构建环境，可用 Environment 函数创建。同一环境可编译多个目标，比如用相同设置编译静态库和目标执行程序。

```
env = Environment(CCFLAGS = "-O3")
env.Library("my", ["test.c"], srcdir = "lib")
env.Program("test", ["main.c"], LIBS = ["my"], LIBPATH = ["."])
```

输出:

```
gcc -o lib/test.o -c -O3 lib/test.c
ar rc libmy.a lib/test.o
ranlib libmy.a

gcc -o main.o -c -O3 main.c
gcc -o test main.o -L. -lmy
```

常用环境参数:

- CC: 编译器，默认 "gcc"。
- CCFLAGS: 编译参数。
- CPPDEFINES: 宏定义。
- CPPPATH：头文件搜索路径。
- LIBPATH：库文件搜索路径。
- LIBS: 需要链接的库名称。
- 除直接提供键值参数外，还可用名为 parse_flags 的特殊参数一次性提供，它会被 ParseFlags 方法自动分解。

```
env = Environment(parse_flags = "-Ilib -L.")
print env["CPPPATH"], env["LIBPATH"]
```

输出:

```
['lib'] ['.']
```

调用 Dictionary 方法返回环境参数字典，或直接用 Dump 方法返回 Pretty-Print 字符串。

```
print env.Dictionary(); ! print env.Dictionary("LIBS", "CPPPATH")
print env.Dump(); ! ! print env.Dump("LIBS")
```

用 "ENV" 键访问执行环境字典。系统不会自动拷贝外部环境变量，需自行设置。

```
import os
env = DefaultEnvironment(ENV = os.environ)
print env["ENV"]["PATH"]
```

## 5.3 方法

### 5.3.1 编译

同一构建环境，可用相关方法编译多个目标。无需关心这些方法调用顺序，系统会自动处理依赖关系，安排构建顺序。

- Program: 创建可执行程序 (ELF、.exe)。
- Library, StaticLibrary: 创建静态库 (.a, .lib)。
- SharedLibrary: 创建动态库 (.so, .dylib, .dll)。
- Object: 创建目标文件 (.o)。
- 如果没有构建环境实例，那么这些函数将使用默认环境实例。

用首个位置参数指定目标文件名 (不包括扩展名)，或用 target、source 指定命名参数。source 是单个源文件名 (包含扩展名) 或列表。

```
Program("test1", "main.c")
Program("test2", ["main.c", "lib/test.c"])! ! # 列表
Program("test3", Split("main.c lib/test.c"))!! # 分解成列表
Program("test4", "main.c lib/test.c".split())! # 分解成列表
```

Glob 用通配符匹配多个文件，还可用 srcdir 指定源码目录简化文件名列表。为方法单独提供环境参数仅影响该方法，不会修改环境对象。

```
Library("my", "test.c", srcdir = "lib")
Program("test2", Glob("*.c"), LIBS = ["my"], LIBPATH = ["."], CPPPATH = "lib")
```

输出:

```
gcc -o lib/test.o -c lib/test.c
ar rc libmy.a lib/test.o
ranlib libmy.a

gcc -o main.o -c -Ilib main.c
gcc -o test2 main.o -L. -lmy
```

创建共享库。

```
SharedLibrary("my", "test.c", srcdir = "lib")
Program("test", Glob("*.c"), LIBS = ["my"], LIBPATH = ["."], CPPPATH = "lib")
```

输出:

```
gcc -o lib/test.os -c -fPIC lib/test.c
gcc -o libmy.dylib -dynamiclib lib/test.os

gcc -o main.o -c -Ilib main.c
gcc -o test main.o -L. -lmy
```

编译方法返回列表，第一元素是目标文件全名。

```
print env.Library("my", "test.c", srcdir = "lib")
```

输出:

```
['libmy.a']
```

### 5.3.2 参数

Append: 追加参数数据。

```
env = Environment(X = "a")
env.Append(X = "b")! ! ! # "a" + "b"。
env.Append(X = ["c"]) !! # 如果原参数或新值是列表，那么 [] + []。
print env["X"]
```

输出:

```
['ab', 'c']
```

AppendUnique: 判断要追加的数据是否已经存在。delete_existing 参数删除原数据，然后添加到列表尾部。原参数值必须是列表。

```
env = Environment(X = ["a", "b", "c"])
env.AppendUnique(X = "d")
env.AppendUnique(1, X = "b")
print env["X"]
```

输出:

```
['a', 'c', 'd', 'b']
```

Prepend, PrependUnique: 将值添加到头部。

```
env = Environment(X = ["a", "b", "c"])
env.Prepend(X = "d")
print env["X"]
```

输出:

```
['d', 'a', 'b', 'c']
```

AppendENVPath, PrependENVPath: 向执行环境追加路径，去重。

```
env = Environment()
print env["ENV"]["PATH"]

env.AppendENVPath("PATH", "./lib")
env.AppendENVPath("PATH", "./lib")
print env["ENV"]["PATH"]
```

输出:

```
/opt/bin:/usr/bin:/bin:/usr/sbin:/sbin:/usr/local/bin
/opt/bin:/usr/bin:/bin:/usr/sbin:/sbin:/usr/local/bin:./lib
```

Replace: 替换参数。如目标不存在，新增。

```
env = Environment(CCFLAGS = ["-g"])
env.Replace(CCFLAGS = "-O3")
print env["CCFLAGS"]
```

输出:

```
-O3
```

SetDefault: 和 Python dict.setdefault 作用相同，仅在目标键不存在时添加。

```
env = Environment(CCFLAGS = "-g")
env.SetDefault(CCFLAGS = "-O3")
env.SetDefault(LIBS = ["m", "pthread"])
print env["CCFLAGS"], env["LIBS"]
```

输出:

```
-g ['m', 'pthread']
```

MergeFlags: 合并参数字典，去重。

```
env = Environment(CCFLAGS = ["option"], CPPATH = ["/usr/local/include"])
env.MergeFlags({"CCFLAGS" : "-O3" })
env.MergeFlags("-I/usr/opt/include -O3 -I/usr/local/include")
print env['CCFLAGS'], env["CPPPATH"]
```

输出:

```
['option', '-O3'] ['/usr/opt/include', '/usr/local/include']
```

ParseFlags: 分解参数。

```
env = Environment()
d = env.ParseFlags("-I/opt/include -L/opt/lib -lfoo")
env.MergeFlags(d)
print d
print env["CPPPATH"], env["LIBS"], env["LIBPATH"]
```

输出:

```
{'LIBPATH': ['/opt/lib'], 'LIBS': ['foo'], ..., 'CPPPATH': ['/opt/include']}
['/opt/include'] ['foo'] ['/opt/lib']
```

### 5.3.3 其他

Clone: 环境对象深度复制，可指定覆盖参数。

```
env = Environment(CCFLAGS = ["-g"], LIBS = ["m", "pthread"])
env2 = env.Clone(CCFLAGS = "-O3")
print env2["CCFLAGS"], env2["LIBS"]
```

输出:

```
-O3 ['m', 'pthread']
```

NoClean: 指示 "scons -c" 不要清理这些文件。

```
my = Library("my", "test.c", srcdir = "lib")
test = Program("test", "main.c")

NoClean(test, my) ! ! ! ! ! # 也可直接使用文件名，注意是 libmy.a。
```

subst: 展开所有环境参数。

```
print env["CCCOM"]
print env.subst("$CCCOM")
```

输出:

```
'$CC -o $TARGET -c $CFLAGS $CCFLAGS $_CCCOMCOM $SOURCES'
'gcc -o -c -O3'
```

各方法详细信息可参考 "man scons" 或 在线手册。

## 5.4 依赖

当依赖文件发生变更时，需重新编译目标程序。可使用 Decider 决定变更探测方式，可选项包括：

- MD5: 默认设置，根据文件内容进行判断。
- timestamp-newer: 如果源文件比目标文件新，则表示发生变更。
- timestamp-match: 检查源文件修改时间和上次编译时是否相同。
- MD5-timestamp: 记录内容变化，但只有源文件修改时间变化时变更。
- 用 touch 更新某个源文件修改时间，即便文件内容没有变化，timestamp-newer 也会让 scons 重新编译该目标文件。

```
env.Decider("timestamp-newer")
env.Program("test", "main.c")
```

某些时候，scons 无法探测到依赖关系，那么可以用 Depends 显式指定依赖。

```
env.Decider("timestamp-newer")

test = env.Program("test", "main.c")
env.Depends(test, ["lib/test.h"])
```

Ignore 忽略依赖关系，Require 指定编译顺序。下例中，指示在编译 my 前必须先构建 test，即便它们之间没有任何依赖关系。

```
my = env.Library("my", "test.c", srcdir = "lib")
test = env.Program("test", "main.c")
env.Requires(my, test)
```

AlwaysBuild 指示目标总是被编译。不管依赖项是否变更，这个目标总是会被重新构建。

```
my = env.Library("my", "test.c", srcdir = "lib")
env.AlwaysBuild(my)
```

## 5.5 命令行

scons 提供了三种不同的命令行参数：

- Options: 以一个或两个 "-" 开始的参数，通常是系统参数，可扩展。
- Variables: 以键值对方式出现。
- Targets: 需要编译的目标。

### 5.5.1 Variables

所有键值都保存在 ARGUMENTS 字典中，可用 Help 函数添加帮助信息。

```
vars = Variables(None, ARGUMENTS)
vars.Add('RELEASE', 'Set to 1 to build for release', 0)

env = Environment(variables = vars)
Help(vars.GenerateHelpText(env))

if not GetOption("help"):
    print ARGUMENTS
    print ARGUMENTS.get("RELEASE", "0")
```

输出:

```
$ scons -Q -h

RELEASE: Set to 1 to build for release
    default: 0
    actual: 0

Use scons -H for help about command-line options.

$ scons -Q RELEASE=1
{'RELEASE': '1'}
1

$ scons -Q
{}
0
```

另有 BoolVariable、EnumVariable、ListVariable、PathVariable 等函数对参数做进一步处理。

### 5.5.2 Targets

Program、Library 等编译目标文件名，可通过 COMMAND_LINE_TARGETS 列表获取。

```
print COMMAND_LINE_TARGETS

Library("my", "lib/test.c")

env = Environment()
env.Program("test", "main.c")
```

输出:

```
$ scons -Q test
['test']
gcc -o main.o -c main.c
gcc -o test main.o

$ scons -Q libmy.a
['libmy.a']
gcc -o lib/test.o -c lib/test.c
ar rc libmy.a lib/test.o
ranlib libmy.a

$ scons -Q -c test libmy.a
['test', 'libmy.a']
Removed main.o
Removed test
Removed lib/test.o
Removed libmy.a
```

除非用 Default 函数指定默认目标，否则 scons 会构建所有目标。多次调用 Default 的结果会被合并，保存在 DEFAULT_TARGETS 列表中。

```
my = Library("my", "lib/test.c")
test = Program("test", "main.c")
Default(my)! ! ! ! ! # 可指定多个目标，比如 Default(my, test)。
```

输出:

```
$ scons -Q
gcc -o lib/test.o -c lib/test.c
ar rc libmy.a lib/test.o
ranlib libmy.a
```

就算指定了默认目标，我们依然可以用 "scons -Q ." 来构建所有目标，清理亦同。

附: scons 还有 Install、InstallAs、Alias、Package 等方法用来处理安装和打包，详细信息可参考官方手册。

[SCons User Guide](http://www.scons.org/doc/production/HTML/scons-user/index.html) [Man page of SCons](http://www.scons.org/doc/production/HTML/scons-man.html)


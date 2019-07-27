# 1. GCC

## 1.1 预处理

输出预处理结果到文件。

```c
$ gcc -E main.c -o main.i
```

保留文件头注释。

```c
$ gcc -C -E main.c -o main.i
```

参数 -Dname 定义宏 (源文件中不能定义该宏)，-Uname 取消 GCC 设置中定义的宏。

```c
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

```c
$ gcc -g -I./lib -I/usr/local/include/cbase main.c mylib.c
```

查看依赖文件。

```c
$ gcc -M -I./lib main.c
$ gcc -MM -I./lib main.c # 忽略标准库
```

## 1.2 汇编

我们可以将 C 源代码编译成汇编语言 (.s)。

```c
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

```c
$ gcc -g -c main.c mylib.c
```

参数 -l 链接其他库，比如 -lpthread 链接 libpthread.so。或指定 -static 参数进行静态链接。我们还可以直接指定链接库 (.so, .a) 完整路径。

```c
$ gcc -g -o test main.c ./libmy.so ./libtest.a

$ ldd ./test
    linux-gate.so.1 => (0xb7860000)
    ./libmy.so (0xb785b000)
    libc.so.6 => /lib/tls/i686/cmov/libc.so.6 (0xb7710000)
    /lib/ld-linux.so.2 (0xb7861000)
```

另外一种做法就是用 -L 指定库搜索路径。

```c
$ gcc -g -o test -L/usr/local/lib -lgdsl main.c

$ ldd ./test
    linux-gate.so.1 => (0xb77b6000)
    libgdsl.so.1 => /usr/local/lib/libgdsl.so.1 (0xb779b000)
    libc.so.6 => /lib/tls/i686/cmov/libc.so.6 (0xb7656000)
    /lib/ld-linux.so.2 (0xb77b7000)
```

## 1.4 动态库

使用 "-fPIC -shared" 参数生成动态库。

```c
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

```c
c$ gcc -c mylib.c

$ ar rs libmy.a mylib.o
ar: creating libmy.a
```

## 1.5 优化

参数 -O0 关闭优化 (默认)；-O1 (或 -O) 让可执行文件更小，速度更快；-O2 采用几乎所有的优化手段。

```c
$ gcc -O2 -o test main.c mylib.c
```

## 1.6 调试

参数 -g 在对象文件 (.o) 和执行文件中生成符号表和源代码行号信息，以便使用 gdb 等工具进行调试。

```c
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

```c
$ gcc -g -pg main.c mylib.c
```

## 1.7 其他编译选项分析

常用选项

-I dir ：在头文件的搜索路径列表中添加dir目录，当用户希望添加放置在非默认位置的头文件时可以通过该选项来指定。

库选项

静态库:一系列的目标文件的归档文件,在编译时会搜索静态库提取出其所需要的目标文件并将其复制到该程序可执行的二进制文件中。

动态库：在程序编译时并不会被链接到目标代码中，而是在程序运行时才会被载入。

警告和出错选项

优化选项

"-On":n是一个代表优化级别的整数,n的取值范围对应的优化效果并不完全相同,比较典型的范围是从0变化到2或者3。

“-O”主要进行线程跳转和延迟退栈。 一般在程序即将发行的时候才考虑对其优化。

体系结构相关选项

gcc有超过100个的可用选项，主要包括总体选项、告警和出错选项、优化选项和体系结构相关选项。以下只对部分常用的选项进行解释：

-c 只是编译不链接（仅把源代码编译为目标代码），生成目标文件“.o”。
-E 只进行预编译，不做其他处理。当这个选项被使用时, 预处理器的输出被送到标准输出而不是储存在文件里。
-S 只是编译不汇编，生成汇编代码。（在为C代码产生了汇编语言文件后停止编译。）gcc产生的汇编语言文件的缺省扩展名是.s 。
-o file 把输出文件输出到file里（即：为将产生的可执行文件用指定的文件名。）

-O 对源代码进行基本优化（主要进行跳转和延迟退栈两种优化）。这些优化在大多数情况下都会使程序执行的更快。
-O2 产生尽可能小和尽可能快的代码。 如-O2，-O3，-On（n常为0--3）；
-O2 除了完成-O1的优化之外，还进行一些额外的调整工作，如指令调整等。
-O3 则包括循环展开和其他一些与处理特性相关的优化工作。
选项将使编译的速度比使用 -O 时慢，但通常产生的代码执行速度会更快。如：

$ gcc test.c -O3
$ gcc -O3 test.c
$ gcc -o tt test.c -O2
$ gcc -O2 -o tt test.c

-g 在可执行程序中包含标准调试信息（产生能被GNU调试器使用的调试信息以便调试你的程序。）
-pg 在编译好的程序里加入额外的代码。运行程序时, 产生gprof用的剖析信息以显示你的程序的耗时情况。
-v 打印出编译器内部编译各过程的命令行信息和编译器的版本
-I 指定头文件目录。可以用相对路径，比如头文件在当前目录，可以用-I.来指定。注意：这里是大写的i
-L 指定库文件目录。可以用相对路径，比如库文件在当前目录，可以用-L.来指定。
-l 指定程序要链接的库，-l参数紧接着就是库名。在这里解释过把库文件名的头lib和尾.so去掉就是库名。注意：这里是小写的Ｌ
-include 用来包含头文件。一般情况下包含头文件都在源码里用＃include <xxx.h>实现，故-include参数很少用。
-static 链接静态库
-llibrary 连接名为 library 的库文件
-Wall 打印出gcc提供的警告信息
-w     关闭所有警告信息
-v      列出所有编译步骤


## 1.8 常见错误分析

- **"xxxx.h: No such file or directory"**

关于-I，-L和-l的几点补充解释：
    /usr/include目录一般是不用指定的，gcc知道去那里找，但是如果头文件不在/usr/include目录的时候，需要用-I参数告诉gcc，比如头文件放在/myinclude目录里，那编译时加上-I/myinclude，如果不加你会得到一个"xxxx.h: No such file or directory"的错误。如果你指定了-I参数，gcc会先查找这个目录，然后才查找默认路径（/usr/include）。

- **“/usr/bin/ld: cannot find -lxxx”**

比如我们自已要用到一个第三方提供的库名字叫libtest.so，那么我们只要把libtest.so拷贝到 /usr/lib里，编译时加上-ltest参数，我们就能用上libtest.so库了（当然要用libtest.so库里的函数，我们还需要与 libtest.so配套的头文件）。

放在/lib和/usr/lib和/usr/local/lib里的库直接用-l参数就能链接了，但如果库文件没放在这三个目录里，而是放在其他目录里，这时我们只用-l参数的话，链接还是会出错，出错信息大概是：“/usr/bin/ld: cannot find -lxxx”，也就是链接程序ld在那3个目录里找不到libxxx.so，这时另外一个参数-L就派上用场了，比如常用的X11的库，它放在/usr/X11R6/lib目录下，我们编译时就要用-L/usr/X11R6/lib -lX11参数，-L参数跟着的是库文件所在的目录名。再比如我们把libtest.so放在/aaa/bbb/ccc目录下，那链接参数就是-L/aaa/bbb/ccc -ltest。



另外，大部分libxxxx.so只是一个链接，以RH9为例，比如libm.so它链接到/lib/libm.so.x，/lib/libm.so.6 又链接到/lib/libm-2.3.2.so，如果没有这样的链接，还是会出错，因为ld只会找libxxxx.so，所以如果你要用到xxxx库，而只有libxxxx.so.x或者libxxxx-x.x.x.so，做一个链接就可以了ln -s libxxxx-x.x.x.so libxxxx.so。手工写链接参数是很麻烦的，很多库开发包提供了生成链接参数的程序，名字一般叫xxxx-config，具体介绍可以[点这里](http://blog.csdn.net/zhuxiaoyang2000/article/details/5575194)





# 2. GDB

作为内置和最常用的调试器，GDB 显然有着无可辩驳的地位。熟练使用 GDB，就好像所有 Linux 下的开发人员建议你用 VIM 一样，是个很 "奇怪" 的情节。

测试用源代码。

```c
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

```c
$ gcc -g -o hello hello.c
```

开始调试：

```c
$ gdb hello

GNU gdb 6.8-debian
Copyright (C) 2008 Free Software Foundation, Inc.
This GDB was configured as "i486-linux-gnu"...

(gdb)
```

## 2.1 源码

在调试过程中查看源代码是必须的。list (缩写 l) 支持多种方式查看源码。

```c
(gdb) l # 显示源代码

(gdb) l # 继续显示

(gdb) l 3, 10 # 显示特定范围的源代码

(gdb) l main # 显示特定函数源代码

(gdb) set listsize 50 //可以用如下命令修改源代码显示行数。
```

## 2.2 断点

可以使用函数名或者源代码行号设置断点。

```c
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

```c
(gdb) b test if a == 10
Breakpoint 4 at 0x80483fa: file hello.c, line 5.

(gdb) info breakpoints

Num Type Disp Enb Address What
4   breakpoint keep y 0x080483fa in test at hello.c:5
    stop only if a == 10
```

可以用 condition 修改条件，注意表达式不包含 if。

```c
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

```c
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

```c
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

```c
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

```c
(gdb) info locals # 显示局部变量
c = 0

(gdb) info args # 显示函数参数(自变量)
a = 4096
b = 8192
```

我们同样可以切换 frame，然后查看不同堆栈帧的信息。

```c
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

```c
(gdb) set variable a=100

(gdb) info args
a = 100
b = 8192
```

## 2.6 内存及寄存器

x 命令可以显示指定地址的内存数据。

```c
格式: x/nfu [address]
```

- n: 显示内存单位 (组或者行)。
- f: 格式 (除了 print 格式外，还有 字符串 s 和 汇编 i)。
- u: 内存单位 (b: 1字节; h: 2字节; w: 4字节; g: 8字节)。

```c
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

```c
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

```c
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

```c
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

```c
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

```shell
(gdb) set follow-fork-mode child
```

临时进入 Shell 执行命令，Exit 返回。

```c
(gdb) shell
```

调试时直接调用函数。

```c
(gdb) call test("abc")
```

使用 "--tui" 参数，可以在终端窗口上部显示一个源代码查看窗。

```
$ gdb --tui hello
```

查看命令帮助。

```c
(gdb) help b
```

最后就是退出命令。

```c
(gdb) q
```

和 Linux Base Shell 习惯一样，对于记不住的命令，可以在输入前几个字母后按 Tab 补全。

## 2.11 Core Dump

在 Windows 下我们已经习惯了用 Windbg 之类的工具调试 dump 文件，从而分析并排除程序运行时错误。在 Linux 下我们同样可以完成类似的工作 —— Core Dump。

我们先看看相关的设置。

```c
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

```c
$ sudo sh -c "ulimit -c unlimited; ./test" # test 是可执行文件名。
```

等等…… 我们还是先准备个测试目标。

```c
#include <stdio.h>
#include <stdlib.h>
void test(){
    char* s = "abc";
    *s = 'x';
}

int main(int argc, char** argv){
    test();
    return (EXIT_SUCCESS);
}
```

很显然，我们在 test 里面写了一个不该写的东东，这无疑会很严重。生成可执行文件后，执行上面的命令。

```c
$ sudo sh -c "ulimit -c unlimited; ./test"

Segmentation fault (core dumped)

$ ls -l

total 96
-rw------- 1 root root 167936 2010-01-06 13:30 core
-rwxr-xr-x 1 yuhen yuhen 9166 2010-01-06 13:16 test
```

这个 core 文件就是被系统 dump 出来的，我们分析目标就是它了。

```c
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

```c
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


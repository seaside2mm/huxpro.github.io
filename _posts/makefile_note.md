# 4. Make

一个完整的 Makefile 通常由 "显式规则"、"隐式规则"、"变量定义"、"指示符"、"注释" 五部分组成。

- 显式规则: 描述了在何种情况下如何更新一个或多个目标文件。
- 隐式规则: make 默认创建目标文件的规则。(可重写)
- 变量定义: 类似 shell 变量或 C 宏，用一个简短名称代表一段文本。
- 指示符: 包括包含(include)、条件执行、宏定义(多行变量)等内容。
- 注释: 字符 "#" 后的内容被当作注释。

(1) 在工作目录按 "GNUmakefile、makefile、Makefile (推荐)" 顺序查找执行，或 -f 指定。 (2) 如果不在 make 命令行显式指定目标规则名，则默认使用第一个有效规则。 (3) Makefile 中 You can't use 'macro parameter character #' in math mode$"。 (4) 可以使用 \ 换行 (注释行也可以使用)，但其后不能有空格，新行同样必须以 Tab 开头和缩进。

注意: 本文中提到的目标文件通常是 ".o"，类似的还有源文件 (.c)、头文件 (.h) 等。

## 4.1 规则

规则组成方式:

```
target...: prerequisites...
    command
    ...
```

- target: 目标。
- prerequisites: 依赖列表。文件名列表 (空格分隔，通常是 ".o, .c, .h"，可使用通配符)。
- command: 命令行。shell 命令或程序，且必须以 TAB 开头 (最容易犯的错误)。

没有命令行的规则只能指示依赖关系，没有依赖项的规则指示 "如何" 构建目标，而非 "何时" 构建。

目标的依赖列表可以通过 GCC -MM 参数获得。

规则处理方式:

- 目标文件不存在，使用其规则 (显式或隐式规则) 创建。
- 目标文件存在，但如果任何一个依赖文件比目标文件修改时间 "新"，则重新创建目标文件。
- 目标文件存在，且比所有依赖文件 "新"，则什么都不做。

### 4.1.1 隐式规则

当我们不编写显式规则时，隐式规则就会生效。当然我们可以修改隐式规则的命令。

```
%.o: %.c
    $(CC) $(CFLAGS) -o $@ -c $<
```

未定义规则或者不包含命令的规则都会使用隐式规则。

```
# 隐式规则
%.o: %.c
    @echo $<
    @echo $^
    $(CC) $(CFLAGS) -o $@ -c $<

all: test.o main.o
    $(CC) $(CFLAGS) $(LDFLAGS) -o $(OUT) $^

main.o: test.o test.h
```

输出:

```
$ make

./lib/test.c
./lib/test.c
gcc -Wall -g -std=c99 -I./lib -I./src -o test.o -c ./lib/test.c

./src/main.c
./src/main.c test.o ./lib/test.h
gcc -Wall -g -std=c99 -I./lib -I./src -o main.o -c ./src/main.c

gcc -Wall -g -std=c99 -I./lib -I./src -lpthread -o test test.o main.o
```

test.o 规则不存在，使用隐式规则。main.o 没有命令，使用隐式规则的同时，还会合并依赖列表。

可以有多个隐式规则，比如：

```
%.o: %.c
    ...
%o: %c %h
    ...
```

### 4.1.2 模式规则

在隐式规则前添加特定的目标，就形成了模式规则。

```
test.o main.o: %.o: %.c
    $(CC) $(CFLAGS) -o $@ -c $<
```

### 4.1.3 搜索路径

在实际项目中我们通常将源码文件分散在多个目录中，将这些路径写入 Makefile 会很麻烦，此时可以考虑用 VPATH 变量指定搜索路径。

```
all: lib/test.o src/main.o
    $(CC) $(CFLAGS) $(LDFLAGS) -o $(OUT) $^
```

改写成 VPATH 方式后，要调整项目目录就简单多了。

```
# 依赖目标搜索路径
VPATH = ./src:./lib

# 隐式规则
%.o:%.c
    -@echo "source file: $<"
    $(CC) $(CFLAGS) -o $@ -c $<

all:test.o main.o
    $(CC) $(CFLAGS) $(LDFLAGS) -o $(OUT) $^
```

执行:

```
$ make

source file: ./lib/test.c
gcc -Wall -g -std=c99 -I./lib -I./src -o test.o -c ./lib/test.c

source file: ./src/main.c
gcc -Wall -g -std=c99 -I./lib -I./src -o main.o -c ./src/main.c

gcc -Wall -g -std=c99 -I./lib -I./src -lpthread -o test test.o main.o
```

还可使用 make 关键字 vpath。比 VPATH 变量更灵活，甚至可以单独为某个文件定义路径。

```
vpath %.c ./src:./lib # 定义匹配模式(%匹配任意个字符)和搜索路径。
vpath %.c # 取消该模式
vpath # 取消所有模式
```

相同的匹配模式可以定义多次，make 会按照定义顺序搜索这多个定义的路径。

```
vpath %.c ./src
vpath %.c ./lib
vpath %.h ./lib

```

VPATH 和 vpath 定义的搜索路径仅对 makefile 规则有效，对 gcc/g++ 命令行无效，比如不能用它定义命令行头文件搜索路径参数。

### 4.1.4 伪目标

当我们为了执行命令而非创建目标文件时，就会使用伪目标了，比如 clean。伪目标总是被执行。

```
clean:
    -rm *.o
.PHONY: clean
```

使用 "-" 前缀可以忽略命令错误，".PHONY" 的作用是避免和当前目录下的文件名冲突 (可能引发隐式规则)。

## 4.2 命令

每条命令都在一个独立 shell 环境中执行，如希望在同一 shell 执行，可以用 ";" 将命令写在一行。

```
test:
    cd test; cp test test.bak
```

提示: 可以用 "\" 换行，如此更美观一些。

默认情况下，多行命令会顺序执行。但如果命令出错，默认会终止后续执行。可以添加 "-" 前缀来忽略命令错误。另外还可以添加 "@" 来避免显示命令行本身。

```
all: test.o main.o
    @echo "build ..."
    @$(CC) $(CFLAGS) $(LDFLAGS) -o $(OUT) $^
```

执行其他规则:

```
all: test.o main.o
    $(MAKE) info
    @$(CC) $(CFLAGS) $(LDFLAGS) -o $(OUT) $^

info:
    @echo "build..."
```

## 4.3 变量

Makefile 支持类似 shell 的变量功能，相当于 C 宏，本质上就是文本替换。 变量名区分大小写。变量名建议使用字母、数字和下划线组成。引用方式 (var)或{var}。引用未定义变量时，输出空。

### 4.3.1 变量定义

首先注意的是 "=" 和 ":=" 的区别。

- = : 递归展开变量，仅在目标展开时才会替换，也就是说它可以引用在后面定义的变量。
- := : 直接展开变量，在定义时就直接展开，它无法后置引用。

```
A = "a: $(C)"
B := "b: $(C)"
C = "haha..."

all:
    @echo $A
    @echo $B

```

输出:

```
$ make

a: haha...
b:

```

由于 B 定义时 C 尚未定义，所以直接展开的结果就是空。修改一下，再看。

```
C = "none..."
A = "a: $(C)"
B := "b: $(C)"
C = "haha..."

all:
    @echo $A
    @echo $B

```

输出:

```
$ make

a: haha...
b: none...
```

可见 A 和 B 的展开时机的区别。

除了使用 "="、":=" 外，还可以用 "define ... endef" 定义多行变量 (宏，递归展开，只需在调用时添加 @ 即可)。

```
define help
    echo ""
    echo " make release : Build release version."
    echo " make clean : Clean templ files."
    echo ""
endef

debug:
    @echo "Build debug version..."
    @$(help)
    @$(MAKE) $(OUT) DEBUG=1

release:
    @echo "Build release version..."
    @$(help)
    @$(MAKE) clean $(OUT)
```

### 4.3.2 操作符

"?=" 表示变量为空或未定义时才进行赋值操作。

```
A ?= "a"

A ?= "A"
B = "B"

all:
    @echo $A
    @echo $B
```

输出:

```
$ make

a
B
```

"+=" 追加变量值。注意变量展开时机。

```
A = "$B"
A += "..."
B = "haha"

all:
    @echo $A
```

输出:

```
$ make
haha ...
```

### 4.3.3 替换引用

使用 "$(VAR:A=B)" 可以将变量 VAR 中所有以 A 结尾的单词替换成以 B 结尾。

```
A = "a.o b.o c.o"

all:
    @echo $(A:o=c)
```

输出:

```
$ make
a.c b.c c.o

```

### 4.3.4 命令行变量

命令行变量会替换 Makefile 中定义的变量值，除非使用 override。

```
A = "aaa"
override B = "bbb"
C += "ccc"
override D += "ddd"

all:
    @echo $A
    @echo $B
    @echo $C
    @echo $D

```

执行:

```
$ make A="111" B="222" C="333" D="444"

111
bbb
333
444 ddd

```

我们注意到追加方式在使用 override 后才和命令行变量合并。

### 4.3.5 目标变量

仅在某个特定目标中生效，相当于局部变量。

```
test1: A = "abc"

test1:
    @echo "test1" $A

test2:
    @echo "test2" $A

```

输出:

```
$ make test1 test2

test1 abc
test2

```

还可以定义模式变量。

```
test%: A = "abc"

test1:
    @echo "test1" $A

test2:
    @echo "test2" $A

```

输出:

```
$ make test1 test2

test1 abc
test2 abc

```

### 4.3.6 自动化变量

- $? : 比目标新的依赖项。
- $@ : 目标名称。
- $< : 第一个依赖项名称 (搜索后路径)。
- $^ : 所有依赖项 (搜索后路径，排除重复项)。

### 4.3.7 通配符

在变量定义中使用通配符则需要借助 wildcard。

```
FILES = $(wildcard *.o)

all:
    @echo $(FILES)

```

### 4.3.8 环境变量

和 shell 一样，可以使用 "export VAR" 将变量设定为环境变量，以便让命令和递归调用的 make 命令能接收到参数。

例如: 使用 GCC C_INCLUDE_PATH 环境变量来代替 -I 参数。

```
C_INCLUDE_PATH := ./lib:/usr/include:/usr/local/include
export C_INCLUDE_PATH

```

## 4.4 条件

没有条件判断是不行滴。

```
CFLAGS = -Wall -std=c99 $(INC_PATHS)
ifdef DEBUG
    CFLAGS += -g
else
    CFLAGS += -O3
endif

```

类似的还有: ifeq、ifneq、ifndef 格式: ifeq (ARG1, ARG2) 或 ifeq "ARG1" "ARG2"

```
# DEBUG == 1
ifeq "$(DEBUG)" "1"
    ...
else
    ...
endif

# DEBUG 不为空
ifneq ($(DEBUG), )
    ...
else
    ...
endif

```

实际上，我们可以用 if 函数来代替。相当于编程语言中的三元表达式 "?:"。

```
CFLAGS = -Wall $(if $(DEBUG), -g, -O3) -std=c99 $(INC_PATHS)

```

## 4.5 函数

*nix 下的 "配置" 都有点 "脚本语言" 的感觉。

make 支持函数的使用，调用方法 "(functionargs)"或"{function args}"。多个参数之间用"," (多余的空格可能会成为参数的一部分)。

例如: 将 "Hello, World!" 替换成 "Hello, GNU Make!"。

```
A = Hello, World!
all:
    @echo $(subst World, GUN Make, $(A))

```

注意: 字符串没有用引号包含起来，如果字符串中有引号字符，使用 "\" 转义。

### 4.5.1 foreach

这个 foreach 很好，执行结果输出 "[1] [2] [3]"。

```
A = 1 2 3
all:
    @echo $(foreach x,$(A),[$(x)])

```

### 4.5.2 call

我们还可以自定义一个函数，其实就是用一个变量来代替复杂的表达式，比如对上面例子的改写。

```
A = x y z
func = $(foreach x, $(1), [$(x)])
all:
    @echo $(call func, $(A))
    @echo $(call func, 1 2 3)

```

传递的参数分别是 "(1),(2) ..."。

用 define 可以定义一个更复杂一点的多行函数。

```
A = x y z
define func
    echo "$(2): $(1) -> $(foreach x, $(1), [$(x)])"
endef

all:
    @$(call func, $(A), char)
    @$(call func, 1 2 3, num)

```

输出:

```
$ make

char: x y z -> [x] [y] [z]
num: 1 2 3 -> [1] [2] [3]

```

### 4.5.3 eval

eval 函数的作用是动态生成 Makefile 内容。

```
define func
    $(1) = $(1)...
endef

$(eval $(call func, A))
$(eval $(call func, B))

all:
    @echo $(A) $(B)

```

上面例子的执行结果实际上是 "动态" 定义了两个变量而已。当然，借用 foreach 可以更紧凑一些。

```
$(foreach x, A B, $(eval $(call func, $(x))))

```

### 4.5.4 shell

执行 shell 命令，这个非常实用。

```
A = $(shell uname)
all:
    @echo $(A)

```

更多的函数列表和详细信息请参考相关文档。

## 4.6 包含

include 指令会读取其他的 Makefile 文件内容，并在当前位置展开。通常使用 ".mk" 作为扩展名，支持文件名通配符，支持相对和绝对路径。

## 4.7 执行

Makefile 常用目标名:

- all: 默认目标。
- clean: 清理项目文件的伪目标。
- install: 安装(拷贝)编译成功的项目文件。
- tar: 创建源码压缩包。
- dist: 创建待发布的源码压缩包。
- tags: 创建 VIM 使用的 CTAGS 文件。
- make 常用命令参数:
- -n: 显示待执行的命令，但不执行。
- -t: 更新目标文件时间戳，也就是说就算依赖项被修改，也不更新目标文件。
- -k: 出错时，继续执行。
- -B: 不检查依赖列表，强制更新目标。
- -C: 执行 make 前，进入特定目录。让我们可以在非 Makefile 目录下执行 make 命令。
- -e: 使用系统环境变量覆盖同名变量。
- -i: 忽略命令错误。相当于 "-" 前缀。
- -I: 指定 include 包含文件搜索目录。
- -p: 显示所有 Makefile 和 make 的相关参数信息。
- -s: 不显示执行的命令行。相当于 "@" 前缀。

顺序执行多个目标:

```
$ make clean debug
```

# 
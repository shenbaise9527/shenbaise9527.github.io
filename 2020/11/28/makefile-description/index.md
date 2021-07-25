# makefile语法说明


## CFLAGS
 表示用于C编译器的选项.
 指定头文件的路径.
 ``` go
 INCLUDES := -I./
 INCLUDES += -I/usr/include
 INCLUDES += -I/usr/local/include
 INCLUDES += -I../../../3rd/curl-7.65.0/include
 INCLUDES += -I../../../3rd/mimetic-0.9.8/include
 CFLAGS := -m64 -std=c++11 -g -Wall -O3 $(INCLUDES)
 ```

## CXXFLAGS
 表示用于C++编译器的选项,基本同CFLAGS

## LDFLAGS
 编译器会用到的一些优化参数,也可指定库文件的位置,告诉链接器从哪里寻找库文件.
 ``` go
 LDFLAGS := -L/usr/lib
 LDFLAGS += -L/usr/local/lib
 LDFLAGS += -L/usr/local/ssl/lib
 LDFLAGS += -L../../../3rd/curl-7.65.0/lib
 LDFLAGS += -L../../../3rd/mimetic-0.9.8/lib
 ```

## LIBS
 告诉链接器需要链接哪些文件.
 ``` go
 LIBS = -lmimetic -lcurl -lrt -static-libgcc -static-libstdc++
 ```

## include
 语法: include <filename>

## wildcard

# 隐晦规则
 1. make看到一个.o文件,就会自动把对应的.c文件加在依赖关系中,并且cc -c xx.c也会被推导.

# 伪目标
.PHONY : clean
clean :
    -rm xx ${objects}
表示clean是个伪目标,在rm前面加个小减号表示当某些文件出现问题时跳过,继续往下执行.

# 工作方式
1. 读入所有的Makefile.
2. 读入被include的其它Makefile.
3. 初始化文件中的变量.
4. 推导隐晦规则,并分析所有规则.
5. 为所有的目标文件创建依赖关系链.
6. 根据依赖关系,决定哪些目标要重新生成.
7. 执行生成命令.
   
# 书写规则
1. Makefile中只应该有一个最终目标,其它目标都是被连带出来的.
2. 规则语法,targets是目标,prerequisites表示目标所依赖的文件或目标,command表示生成目标文件所需要执行的命令.
``` go
targets : prerequisites
    command
```
3. 在规则中使用通配符,支持三种(* ? [...]),*表示任意长度的字符串,转义字符为'/'
``` go
clean :
    rm -f *.o
//　上面表示删除任意已.o结尾的文件.

// 需要注意用在变量中.
objects = *.o
// objects的值就是"*.o",并不会被展开,若想让objects的值是所有.o文件的集合，如下书写
objects := $(wildcard *.o)
```
4. 文件搜寻
   VPATH: Makefile中的特殊变量,指定文件搜寻目录.
   ``` go
   VPATH = src : ../headers
   // make会自动去src和../headers目录搜寻依赖文件.
   ```
   vpath: make的关键字,按照某种模式去搜寻目录,多个目录以:分隔.
   ``` go
   vpath <pattern> <directories>
   // 为符合pattern模式的文件指定搜索目录directories.
   vpath <pattern>
   // 清除pattern模式的搜寻目录.
   vpath
   //　清除所有已被设置好的搜寻目录.
   vpath *.h ../headers
   // 在../headers目录下搜索所有以.h结尾的文件.
   ```

# 自动变量
1. "$@",表示目前规则中所有目标的集合,主要用于有多个目标的规则中.
``` go
bigoutput littleoutput : text.g
    generate text.g -$(substr output,,$@) > $@
//　上述规则等价于
bigoutput : text.g
    generate text.g -big > bigoutput
littleoutput : text.g
    generate text.g -little > littleoutput
// $@表示目标的集合,就像一个数组(2个元素bigoutput、littleoutput),$@依次取出目标并执行命令.
```
2. "$<",表示所有的依赖目标集,是一个一个取出来的.
3. "$%",仅当目标是函数库文件时(.a或.lib),表示规则中的目标成员名.
   如果目标"foo.a(bar.o)",那"$%"就是bar.o,"$@"就是foo.a
4. "$?",所有比目标新的依赖目标的集合,以空格分隔.
5. "$^",所有的依赖目标的集合,以空格分隔,会去除重复的.
6. "$*",表示目标模式中"%"及其之前的部分.如果目标是"dir/a.foo.b",并且目标的模式是"a.%.b",那么值就是"dir/a.foo"

# 静态模式
更容易定义多目标的规则.
``` go
// 语法.
<targets ...> : <target-pattern> : <prereq-patterns ...>
    <command>
// targets: 定义了一系列的目标文件,可以有通配符.是目标的一个集合.
// target-pattern: 指明了targets的模式,也就是目标集的模式.
// prereq-patterns: 目标的依赖模式.对target-pattern形成的模式再进行一次依赖目标的定义.

objects = foo.o bar.o
all : $(objects)
$(objects) : %.o : %.c
    $(CC) -c $(CFLAGS) $< -o $@
// 目标从objects中获取,%.o表明所有以".o"结尾的目标,也就是"foo.o和bar.o",这也是变量$ojbects集合的模式
// 而依赖模式"%.c"则取模式"%.o"的"%",即"foo和bar",并为其加上.c后缀,则依赖的目标是"foo.c和bar.c"
// $<为自动化变量,表示所有的依赖目标集,即"foo.c和bar.c"
// $@为自动化变量,表示目标集.
//　上面规则等价于:
foo.o : foo.c
    $(CC) -c $(CFLAGS) foo.c -o foo.o
bar.o : bar.c
    $(CC) -c $(CFLAGS) bar.c -o bar.o

//　例子:
files = foo.elc bar.o lose.o
$(filter %.o,$(files)) : %.o : %.c
    $(CC) -c $(CFLAGS) $< -o $@
// filter为过滤函数.
```

# 变量
1. 操作符"=",右侧变量的值可以不用提前定义.
2. 操作符":=",右侧变量的值必须在这之前定义.
3. 操作符"?=",如果变量之前没有被定义过,那变量的值就是右侧的值;否则什么也不做.
4. 变量值的替换,$(var:a=b),把变量var中所有以"a"子串结尾的"a"替换成"b"子串,结尾指空格或结束符
5. 操作符"+=",给变量追加值.

# 条件表达式
1. ifeq
2. ifneq
3. ifdef
4. ifndef

# 函数
1. $(subst <from>,<to>,<text>),把字符串text中的"from"替换为"to"
2. $(patsubst <pattern>,<replacement>,<text>),查找字符串text中是否有符合模式"pattern",如果匹配则以"replacement"替换.
   "replacement"中也可以包含"%",指"pattern"中那个"%"所代表的子串.
3. $(stip <string>),去掉字符串string中开头和结尾的空字符.
4. $(findstring <find>,<in>),在字符串"in"中查找"find",如果找到返回"find",否则返回空字符串.
5. $(filter <pattern...>,<text>),以"pattern"模式过滤"text"字符串中的单词,保留符合"pattern"的单词.可以有多个模式.
   ``` go
   sources := foo.c bar.c baz.s ugh.h
   foo : $(sources)
        cc $(filter %.c %.s,$(sources)) -o foo
   // filter返回的是foo.c bar.c baz.s,有２个模式%.c和%.s
   ```
6. $(filter-out <pattern...>,<text>),反过滤函数.
7. $(sort <list>),排序函数,给"list"中的单词升序的方式排序.会去掉相同的单词.
8. $(word <n>,<text>),取字符串"text"中的第n个单词(从1开始).
9. $(wordlist <s>,<e>,<text>),取单词串函数,从字符串"text"中取从<s>到<e>的单词串,s和e是一个数字.
10. $(words <text>),单词个数统计函数.
11. $(firstword <text>),首单词函数.
12. $(dir <names...>),取目录函数,从文件名序列中取出目录部分.
13. $(notdir <names...>),取文件函数,从文件名序列中取出非目录部分.
14. $(suffix <names...>),取后缀函数,从文件名序列中取出各个文件名的后缀.
15. $(basename <names...>),取前缀函数.
16. $(addsuffix <suffix>,<names...>),加后缀函数.
17. $(addprefix <prefix>,<names...>),加前缀函数.
18. $(join <list1>,<list2>),连接函数,把"list2"中的单词对应地加到"list1"的单词后面.
    ``` go
    $(join aaa bbb,111 222 333)->"aaa111 bbb222 333"
    ```
19. $(foreach <var>,<list>,<text>),把参数list中的单词逐一取出放到参数var中,然后再执行<text>所包含的表达式.
    \``` go
    names := a b c d
    files := $(foreach n,$(names),$(n).o)
    // files的值就是a.o b.o c.o d.o
    ```
20. $(if <condition>,<then-part>,<else-part>),else是可选的.
21. $(call <expression>,<param1>,<param2>,<param3>...),用参数依次取代"expression"中的变量.
22. $(origin <variable>),变量"variable"是从哪来的.
    undefined: 表示该变量未定义.
    default: 表示是默认定义的.
    environment: 表示是环境变量.
    file: 表示该变量定义在Makefile中
    command line: 表示是被命令行定义的
    override: 表示是被override指示符重新定义的
    automatic:　表示是自动化变量

# 隐含规则
1. 编译c程序的隐含规则,<n>.o会自动推导为<n>.c,命令是$(CC) -c $(CPPFLAGS) $(CFLAGS)
2. 编译C++的隐含规则,<n>.o会自动推导为<n>.cc,命令是$(CXX) -c $(CPPFLAGS) $(CFLAGS)
3. 链接Object文件的隐含规则,$(CC) $(LDFLAGS) <n>.o $(LOADLIBES) $(LDLIBS)
## 变量
1. CC,C语言编译程序,默认命令是"cc"
2. CXX,C++语言编译程序,默认命令是"g++"
3. RM,删除文件命令,默认命令是"rm -f"
4. CFLAGS,C语言编译器参数
5. CXXFLAGS,C++语言编译器参数
6. CPPFLAGS,C预处理器参数
7. LDFLAGS,链接器参数,如"ld"

# 模式规则
使用模式规则定义一个隐含规则.
## 介绍
模式规则中,至少在规则的目标定义中要包含"%",否则就是一般的规则.目标中的"%"定义表示对文件名的匹配.
例子: %.o : %.c; <command>
## 示例
``` go
%.o : %.c 
    $(CC) -c $(CFLAGS) $(CPPFLAGS) $< -o $@
// $< 表示依赖目标
// $@ 表示目标
```
一个完整的例子:
``` go
SOURCE := $(wildcard *.cpp)
OBJS := $(patsubst %.cpp,%.o,$(SOURCE))
EXENAME := mail_agent
TARGET := $(EXENAME)
CC := g++
 
INCLUDES := -I./
INCLUDES += -I/usr/include
INCLUDES += -I/usr/local/include
INCLUDES += -I../../../3rd/curl-7.65.0/include
INCLUDES += -I../../../3rd/mimetic-0.9.8/include
 
LDFLAGS := -L/usr/lib
LDFLAGS += -L/usr/local/lib
LDFLAGS += -L/usr/local/ssl/lib
LDFLAGS += -L../../../3rd/curl-7.65.0/lib
LDFLAGS += -L../../../3rd/mimetic-0.9.8/lib
 
LIBS = -lmimetic -lcurl -lrt -static-libgcc -static-libstdc++
CFLAGS := -m64 -std=c++11 -g -Wall -O3 $(INCLUDES)
CXXFLAGS := $(CFLAGS)
 
$(TARGET) : $(OBJS)
    $(CC) $(CXXFLAGS) -o $@ $(OBJS) $(LDFLAGS) $(LIBS)
 
.PHONY : clean
clean:
    -rm $(OBJS) $(TARGET)
```


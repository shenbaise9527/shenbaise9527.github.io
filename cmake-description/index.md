# CMake语法说明


## 预定义变量
* PROJECT_SOURCE_DIR 工程的根目录
* PROJECT_BINARY_DIR 运行cmake命令的目录,通常是${PROJECT_SOURCE_DIR}/build
* CMAKE_INCLUDE_PATH 环境变量,非CMake变量
* CMAKE_LIBRARY_PATH 环境变量
* CMAKE_CURRENCT_SOURCE_DIR 当前处理的CMakeLists.txt所在的路径
* CMAKE_CURRENT_BINARY_DIR target编译目录
  * 使用ADD_SUBDIRECTORY(src bin)可以更改此变量的值
  * 使用SET(EXECUTABLE_OUTPUT_PATH <新路径>)并不会对此变量有影响,只改变最终目标文件的存储路径
* CMAKE_CURRENT_LIST_FILE 输出调用这个变量的CMakeLists.txt的完整路径
* CMAKE_CURRENT_LIST_LINE 输出这个变量所在的行
* CMAKE_MODULE_PATH 定义自己的cmake模块所在的路径
* EXECUTABLE_OUTPUT_PATH 重新定义目标二进制可执行文件的存储路径
* LIBRARY_OUTPUT_PATH 重新定义链接库的存储路径
* PROJECT_NAME 返回通过PROJECT指令定义的项目名称
* CMAKE_ALLOW_LOOSE_LOOP_CONSTRUCTS  用来控制

## 系统信息
* CMAKE_MAJOR_VERSION cmake的主版本号,如2.8.6中的2
* CMAKE_MINOR_VERSION cmake的次版本号,如2.8.6中的8
* CMAKE_PATCH_VERSION cmake的补丁等级,如2.8.6中的6
* CMAKE_SYSTEM 系统名称,如Linux-2.6.22
* CMAKE_SYSTEM_NAME 不包含版本的系统名,如Linux
* CMAKE_SYSTEM_VERSION 系统版本,如2.6.22
* CMAKE_SYSTEM_PROCESSOR 处理器名称,如i386
* UNIX 在所有的类UNIX平台为TRUE,包括OS x和cygwin
* WIN32 在所有的win32平台为TRUE,包括cygwin

## 常用命令
* PROJECT 指定工程名称,PROJECT(projectname)
* SET 定义变量,SET(VAR [VALUE]),可以定义多个value,空格分隔
* MESSAGE 向终端输出用户定义的信息或变量的值,MESSAGE([SEND_ERROR|STAUTS|FATAL_ERROR] "display")
  * SEND_ERROR 产生错误,生成过程被跳过
  * STATUS 输出前缀为--的信息
  * FATAL_ERROR 立即终止所有cmake过程
* ADD_EXECUTABLE 生成可执行文件,ADD_EXECUTABLE(bin_file_name SRC_LIST)
* ADD_LIBRARY 生成动态库或静态库,ADD_LIBRARY(libname [SHARED|STATIC|MODULE] [EXCLUED_FROM_ALL] SRC_LIST)
  * SHARED 动态库
  * STATIC 静态库
  * MODULE 在使用dyld的系统有效,否则等同于SHARED
  * EXCLUED_FROM_ALL 表示该库不会被默认构建
* SET_TARGET_PROPERTIES 设置输出的名称,设置动态库的版本和API的版本
* CMAKE_MINIMUN_REQUIRED 声明CMake的版本要求
* ADD_SUBDIRECTORY 添加子目录,ADD_SUBDIRECTORY(dir [binary_dir][EXCLUDE_FROM_ALL])
  * binary_dir 指定中间二进制和目标二进制文件的存储位置
  * EXCLUDE_FROM_ALL 将这个目录中编译过程中排除
* INCLUDE_DIRECTORIES 向工程添加多个特定的头文件搜索路径,路径之间用空格分隔
* LINK_DIRECTORIES 添加非标准的共享库搜索路径
* TARGET_LINK_LIBRARIES 为target添加需要链接的共享库
* ADD_DEFINITIONS 向C/C++编译器添加-D定义,参数之间用空格分隔
* ADD_DEPENDENCIES 定义target依赖的其它target,确保target在构建之前,其依赖的target已构建完毕
* AUX_SOURCE_DERICTORY 发现一个目录下的所有源代码文件并将列表存储在一个变量中
* EXEC_PROGRAM 用于在指定目录运行某个程序(默认为当前CMakeLists.txt目录)
* INCLUDE 用来载入CMakeLists.txt或预定义的cmake模块
* FIND_FILE 查找文件,FIND_FILE(<VAR> name path1 path2 ...),VAR表示找到的文件全路径,包括文件名
* FIND_LIBRARY 查找库
* FIND_PATH 查找路径
* FIND_FILE 
* IF 语法
  > IF (expression) 判断条件是否为真
  > IF (not exp) 与上面相反
  > IF (var1 and var2) 判断2个条件是否都为真
  > IF (var1 or var2) 判断2个条件是否至少有1个为真
  > IF (COMMAND cmd) 判断cmd是否为命令并可调用
  > IF (EXISTS dir) 判断dir目录是否存在
  > IF (EXISTS file) 判断file文件是否存在
  > IF (file1 IS_NEWER_THAN file2) 当file1比file2新,或file1/file2中有一个不存在时为真,使用全路径
  > IF (IS_DIRECTORY dir) 当dir时路径时为真
  > IF (DEFINED var) 若var被定义,为真
  > IF (var MATCHES regex) 当变量var匹配正则表达式regex时,为真

    ``` go
    IF ("hello" MATCHES "ell")
        MESSAGE("true")
    ENDIF ("hello" MATCHES "ell")
    ```
* WHILE 
* FOREACH
  列表格式
  
  ``` go 
  FOREACH(loop_var arg1 arg2 ...)
    COMMAND1(ARGS ...)
    COMMAND2(ARGS ...)
  ENDFOREACH(loop_var)
  
  AUX_SOURCE_DERICTORY(. SRC_LIST)
  FOREACH(F ${SRC_LIST})
    MESSAGE(${F})
  ENDFOREACH(F)
  ```
  范围格式
  
  ``` go
  FOREACH(loop_var RANGE total)
    COMMAND1(ARGS ...)
    COMMAND2(ARGS ...)
  ENDFOREACH(loop_var)
  
  FOREACH(VAR RANGE 10)
    MESSAGE(${VAR})
  ENDFOREACH(VAR)
  ```
  范围和步进格式
  
  ``` go
  FOREACH(loop_var RANGE start stop [step])
    COMMAND1(ARGS ...)
    COMMAND2(ARGS ...)
  ENDFOREACH(loop_var)
  
  FOREACH(A RANGE 5 15 3)
    MESSAGE(${A})
  ENDFOREACH(A)
  ```

## 开关选项
* BUILD_SHARED_LIBS 控制默认的库编译方式.若未设置,使用ADD_LIBRARY时又没有指定库类型,默认编译生成的都是静态库
* CMAKE_C_FLAGS 设置C编译选项
* CMAKE_CXX_FLAGS 设置C++编译选项

## 添加子文件夹
``` go
# 设置查找目录
set(plugins_dir ${CMAKE_CURRENT_LIST_DIR}/plugins/)

# 运行脚本查找对应的子目录,并存放到变量dirs中
execute_process(
    COMMAND sh ${CMAKE_CURRENT_LIST_DIR}/findplugin.sh ${plugins_dir}
    OUTPUT_VARIABLE dirs)

# 把字符串变量转换为列表RPLACE_LIST
string(REPLACE "\n" ";" RPLACE_LIST ${dirs})

# 循环,把每个plugin加入到编译中
foreach (miapi ${RPLACE_LIST})
    ADD_SUBDIRECTORY(${plugins_dr}${miapi} ${CMAKE_BINARY_DIR}/${miapi})
endforeach(miapi)
```

## add_custom_command
增加定制化的构建规则到构建系统中,有两种使用方式
### 增加一个定制化命令来产生一个输出
语法格式
``` go
ADD_CUSTOM_COMMAND(OUTPUT output1 [output2 ...]
    COMMAND command1 [ARGS] [arg1 ...]
    [COMMAND command2 [ARGS] [arg2 ...]...]
    [MAIN_DEPENDENCY depend]
    [DEPENDS [depends ...]]
    [IMPLICIT_DEPENDS <lang1> depend1 ...]
    [WORKING_DIRECTORY dir]
    [COMMENT comment] [VERBATIM] [APPEND])
# 不要同时在多个相互独立的目标中执行上述命令产生相同的文件,主要是为了防止产生冲突.
# 如果有多条命令,会按顺序执行.
# ARGS是为了向后兼容,使用过程中可以忽略
# MAIN_DEPENDENCY完全是可选的,是针对VS给出的一个建议

# 例子,copy复制文件.
ADD_CUSTOM_COMMAND(OUTPUT ${dst_sql_xml}
    COMMAND ${CMAKE_COMMAND} -E copy ${src_sql_xml} ${dst_sql_xml}
    COMMENT "copy ${src_sql_xml} \nto ${dst_sql_xml}"
    DEPENDS ${src_sql_xml})
ADD_CUSTOM_TARGET(syncxml ALL DEPENDS ${dst_sql_xml})

# 例子,copy_directory复制文件夹.
ADD_CUSTOM_COMMAND(OUTPUT ${dst_sql_xml}
    COMMAND ${CMAKE_COMMAND} -E copy_directory ${src_sql_xml} ${dst_sql_xml}
    COMMENT "copy_directory ${src_sql_xml} \nto ${dst_sql_xml}"
    DEPENDS ${src_sql_xml})
ADD_CUSTOM_TARGET(syncxml ALL DEPENDS ${dst_sql_xml})
```
### 标记在什么时候执行命令:编译前、编译后、链接前
语法格式
``` go
ADD_CUSTOM_COMMAND(TARGET target
    PRE_BUILD | PRE_LINK | POST_BUILD
    COMMAND command1 [ARGS] [args1 ...]
    [COMMAND command2 [ARGS] [args2 ...]...]
    [WORKINGDIRECTORY dir]
    [COMMENT comment] [VERBATIM])
# PRE_BUILD 命令将会在其它依赖项执行前执行,只被VS7及之后的版本支持,其它会将其等同于PRE_LINK
# PRE_LINK 命令将会在其它依赖项执行完后执行
# POST_BUILD 命令将会在目标构建完后执行
# 如果指定了WORKINGDIRECTORY,命令将会在指定目录运行
# 如果指定了COMMENT,命令执行前会把COMMENT的内容当做信息输出
# 如果指定了APPEND,COMMANDS和DEPENDS的值会追加到第一个指定的命令中
# 如果指定了APPEND,COMMENT、WORKINGDIRECTORY和MAIN_DEPENDENCY将会被忽略
# 如果指定了VERBATIM,所传递的命令参数会被适当地转义
# 如果指定命令的输出不是创建一个存储在磁盘上的文件,需使用SET_SOURCE_FILE_PROPERTIES把它标记为SYMBOLIC
# 如果COMMAND指定了一个可执行的目标(由ADD_EXECUTABLE创建),则会
# DEPENDS指定了该命令依赖的文件
```
### add_custom_target
使用该命令增加一个没有输出的目标,使得它总是被构建
``` go
ADD_CUSTOM_TARGET(Name [ALL]
    COMMAND command1 [ARGS] [args1 ...]
    [DEPENDS depend depend ...]
    [WORKINGDIRECTORY dir]
    [COMMENT comment] [VERBATIM]
    [SOURCES src1 [src2 ...]])
# 该目标没有输出,总是被认为过期的
# 如果指定了ALL,表明目标会被添加到默认的构建目标,使得它每次都会被运行
# 具体可以参见上面的例子
```

## FAQ
1. 设置条件编译
``` go
option(DEBUG_mode "ON for debug or OFF for release" ON)
IF(DEBUG_mode)
    add_definitions(-DDEBUG)
ENDIF()
```
2. 根据OS指定编译选项
IF(WIN32) IF(APPLE) IF(UNIX)


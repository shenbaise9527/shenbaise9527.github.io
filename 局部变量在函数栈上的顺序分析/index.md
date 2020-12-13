# 局部变量在函数栈上的顺序分析


## 问题来源
源于如下代码:
``` C
int main(int argc, char* argv[]){
    int i = 0;
    int arr[3] = {0};
    for(; i<=3; i++){
        arr[i] = 0;
        printf("hello world\n");
    }
    return 0;
}
```
这段代码的运行结果并非是打印三行“hello word”，而是会无限打印“hello word”，这是为什么？

当看到这段后，脑海中最直观的感受：
> 函数栈上的地址是从大到小的，局部变量i先入栈，数组arr后入栈。假设arr的地址为0x0，则arr[0]的地址为0x0，arr[1]的地址为0x4，arr[2]的地址为0x8，i的地址为0xc。所以在循环中访问到arr[3]时的地址正好对应到变量i上，把i的值修改为0，导致出现死循环。

需要进行验证下，首先在vs2013上测试，发现和想象的完全不一样。
只打印了四行“hello word”，然后程序崩溃，显示“Run-Time Check Failure #2 - Stack around the variable 'arr' was corrupted”。
然后修改下代码把变量i和arr的地址都打印出来，可以看到变量i的地址比arr的地址小，这是怎么回事？怎么参数入栈的顺序和变量声明的顺序不一样了？

接着又在x86-64位centos6上测试，采用“gcc demo.c -o demo”编译，运行发现和想象的是一样的，无限打印“hello world”。

问题：
> 1.参数入栈的顺序和变量声明的顺序怎么不一致？
> 2.在linux下表现怎么和windows不一样？

## 问题分析
以前从来没关注过局部变量的声明顺序和入栈顺序之前的关系，首先来理理为什么在windows下不一致，而在linux下是一致的。
在linux下使用的编译器是GCC，主要参考的是[GCC 中的编译器堆栈保护技术](https://www.ibm.com/developerworks/cn/linux/l-cn-gccstack/index.html)这篇文章，文中提到了GCC中三个与堆栈保护有关的选项。
> 1. -fstack-protector，启用堆栈保护，不过只为局部变量中含有 char 数组的函数插入保护代码
> 2. -fstack-protector-all，启用堆栈保护，为所有函数插入保护代码
> 3. -fno-stack-protector，禁用堆栈保护

在linux下重新编译下，gcc demo.c -fstack-protector -o demo，发现运行时只打印了四次“hello world”，且变量i的地址也是比arr的地址小，linux下的现象已和windows一样了。
> 编译时如果开启了优化选项(O2或O3)，会默认启用堆栈保护。

当启用堆栈保护后，局部变量的顺序被重新组织了。这样做的目的主要是为了防止溢出攻击，具体可以详读上面提到的那篇文章。
局部变量的顺序的重组的规则如下，主要适用于GCC编译器，VS的处理不一样(主要参考[C语言局部变量在内存栈中的顺序](https://blog.csdn.net/qq_19406483/article/details/77511447?utm_source=blogxgwz3)这篇文章,但结论有所修正)：
> 1. 内存由高到低优先分配给占位8字节、4字节、2字节、1字节的数据类型
> 2. 同总占位的类型按定义变量的先后顺序内存地址会增加
> 3. 在规则2前提下，定义数组不会和同总数据类型混占内存

数据类型占位说明(64位机器下)： 
8字节：double、long long int、long int(该类型在64位windows下位4字节，在linux x86/ppc下都是8字节)
4字节：int、float、unsigned int 
2字节：short 、unsigned short 
1字节：char 、unsigned char 
例如,分别定义下列变量，内存地址中由高到低分别为： double < int < short < char 

参考如下代码：
``` C
int main(int argc, const char *argv[])
{
    char char_a;
    short short_a;
    int int_a;
    float float_a;
    double double_a;
    unsigned int uint_a;
    long int lint_a;
    long long int dlint_a;

    printf("  &char_a : %p\n",&char_a);
    printf(" &short_a : %p\n",&short_a);
    printf("   &int_a : %p\n",&int_a);
    printf(" &float_a : %p\n",&float_a);
    printf("&double_a : %p\n",&double_a);
    printf(" &unsigned_int_a : %p\n",&uint_a);
    printf("     &long_int_a : %p\n",&lint_a);
    printf("&long_long_int_a : %p\n",&dlint_a);
    
    int i = 0;
    printf("address-index: %p\n", &i);
    int arr[3] = {0};
    for (; i < 3; i++) 
    {
        arr[i] = 0;
        printf("address-arr[%d]: %p\n", i, &(arr[i]));
    }
    return 0;
}

```

启用堆栈保护选项，结果如下：
``` C
&char_a : 0x7fff14d57fa5   --在最低的低地址位
&short_a : 0x7fff14d57fa6
&int_a : 0x7fff14d57fa8  --int_a、float_a、uint_a、i 4个同大小的变量地址在一起(与声明顺序相反)
&float_a : 0x7fff14d57fac
&double_a : 0x7fff14d57fb8 --double_a、lint_a、llint_a 3个同大小的变量地址在一起(与声明顺序相反)
&uint_a : 0x7fff14d57fb0
&lint_a : 0x7fff14d57fc0
&llint_a : 0x7fff14d57fc8
address-i: 0x7fff14d57fb4
address-arr[0]: 0x7fff14d57fd0  --数组地址在高位上
address-arr[1]: 0x7fff14d57fd4
address-arr[2]: 0x7fff14d57fd8
```
从以上结果可以验证上面的规则。针对规则三，对于int类型的数组地址在高位，非数组变量在低位。double类型的数组也是类似的(注意上面的提到的文章是相反的结论)

## 问题总结
1. 堆栈保护技术，主要是编译器为了防止溢出攻击而发展出来的技术，GCC有相应的编译选项可以开启和关闭。
2. 局部变量的入栈规则，不启用堆栈保护技术时，按照声明的顺序入栈。启用堆栈保护技术时，是按照三个规则来进行重组的。
3. GCC默认是没有开启堆栈保护选项的，但如果启用了优化选项，堆栈保护选项会自动启用。

## 问题扩展
如果数组arr的长度是4，如下代码，在64位x86上还能无限打印“hello world”吗？(gcc demo.c -o demo编译)
``` C
int main(int argc, char* argv[]){
    int i = 0;
    int arr[4] = {0};
    for(; i<=4; i++){
        arr[i] = 0;
        printf("hello world\n");
    }
    return 0;
}
```
是不会无限打印的，这涉及到8字节对齐的问题。
数组arr[4]刚好满足8字节对齐，在栈中i和arr是不会连续存放的(暂不清楚缘由)，所以越界是不会访问到i的。
数组arr[3]是不满足8字节对齐，把变量i放到一起刚好满足，编译器就会把i和arr存放到一起。

如果数组arr的长度是5或7列，结果又是如何的？




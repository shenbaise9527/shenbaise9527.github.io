# golang数据类型


主要包含基本的内建类型(布尔类型、数值类型和字符串类型)和复合类型(array、slice、map、channel、function、struct、interface)

## 基本数据类型

### 布尔类型
类型标记为bool，值为true/false，零值为false，**值类型，可定义为常量**
``` go
// 变量定义的几种方式
var bflag bool
bflag = true

var bflag1 bool = true
var bflag2 = true

// 短变量声明,只能用于函数内部
bflag3 := true
```

### 整数类型
类型标记为**int/uint、int8/uint8、int16/uint16、int32/uint32、int64/uint64、byte、rune**，零值为0，**值类型，可定义为常量**
|标记符|说明|
|:----|:---|
|int/uint|有符号/无符号整数,依赖于CPU平台机器字大小,32或64bit|
|int8/uint8|有符号/无符号整数,8bit|
|int16/uint16|有符号/无符号整数,16bit|
|int32/uint32|有符号/无符号整数,32bit|
|int64/uint64|有符号/无符号整数,64bit|
|byte|等价于uint8,一般用于强调数值是一个原始的数据而不是一个小的整数|
|rune|等价于int32,表示一个Unicode码点|

其中有符号整数采用2的补码形式表示，也就是最高bit位用来表示符号位，一个n-bit的有符号数的值域是从$-2^{n-1}$到$2^{n-1}-1$。无符号整数的所有bit位都用于表示非负数，值域是0到$2^n-1$。例如，int8类型整数的值域是从-128到127，而uint8类型整数的值域是从0到255

rune专门用来存储Unicode编码的单个字符，有5种表示方式：
1. 该rune字面量所对应的字符，比如'a'、'-'，这个字符必须是Unicode编码规范所支持的
2. 使用“\x”为前导后跟2位十六进制数，表示宽度为1字节
3. 使用“\”为前导后跟3位八进制数，表示的范围与上一个表示法相同
4. 使用“\u”为前导后跟4位十六进制数，表示宽度为2字节的值
5. 使用“\U”为前导后跟8位十六进制数，表示宽度为4字节的值
**还支持一类特殊的字符序列----转义符**

Go语言中关于算术运算、逻辑运算和比较运算的二元运算符，它们按照优先级递减的顺序排列：
``` go
*      /      %      <<       >>     &       &^
+      -      |      ^
==     !=     <      <=       >      >=
&&
||
```

整数的bit位操作符
``` go
&      位运算 AND
|      位运算 OR
^      位运算 XOR
&^     位清空 (AND NOT)
<<     左移
>>     右移
```
> **注意：** ++/- -只能后置，且是语句不是表达式，不能进行赋值，即i++是合法，++i和j=i++都是非法的

### 浮点数类型
类型标记为**float32/float64**，零值为0，**值类型，可定义为常量**

浮点数的范围极限值可以在math包找到。常量math.MaxFloat32表示float32能表示的最大数值，大约是 3.4e38；对应的math.MaxFloat64常量大约是1.8e308。它们分别能表示的最小值近似为1.4e-45和4.9e-324。

一个float32类型的浮点数可以提供大约6个十进制数的精度，而float64则可以提供约15个十进制数的精度；通常应该优先使用float64类型，因为float32类型的累计计算误差很容易扩散，并且float32能精确表示的正整数并不是很大（**注意：因为float32的有效bit位只有23个，其它的bit位用于指数和符号；当整数大于23bit能表达的范围时，float32的表示将出现误差**）。

### 复数类型
类型标记为**complex64/complex128**，**值类型，可定义为常量**

### 字符串类型
类型标记为**string**，零值为""，**值类型，可定义为常量**

一个字符串是一个不可改变的字节序列
```go
str := "hello"
str[0] = 'x' // 非法，字符串是只读的
```

字符串在Go语言内存模型中用一个2字长的数据结构表示。它包含一个指向字符串存储数据的指针和一个长度数据。因为string类型是不可变的，对于多字符串共享同一个存储数据是安全的。切分操作会得到一个新的2字长结构字符串，但是指向同一个字节序列，切分时不涉及内存分配或复制操作。

### 常量特别说明
常量只能是布尔类型、整数类型、浮点数类型、复数类型、字符串

常量生成器**itoa**，常量声明可以使用iota常量生成器初始化，它用于生成一组以相似规则初始化的常量，但是不用每行都写一遍初始化表达式

无类型常量：
Go语言的常量有个不同寻常之处。虽然一个常量可以有任意有一个确定的基础类型，例如int或float64，或者是类似time.Duration这样命名的基础类型，但是许多常量并没有一个明确的基础类型。编译器为这些没有明确的基础类型的数字常量提供比基础类型更高精度的算术运算；你可以认为至少有256bit的运算精度。这里有六种未明确类型的常量类型，分别是**无类型的布尔型、无类型的整数、无类型的字符、无类型的浮点数、无类型的复数、无类型的字符串**。
通过延迟明确常量的具体类型，无类型的常量不仅可以提供更高的运算精度，而且可以直接用于更多的表达式而不需要显式的类型转换。
```go
// 不需要类型转换
var x float32 = math.Pi
var y float64 = math.Pi
var z complex128 = math.Pi

// 需要类型转换,Pi64定义了具体类型
const Pi64 float64 = math.Pi
var x float32 = float32(Pi64)
var y float64 = Pi64
var z complex128 = complex128(Pi64)
```

## 复合数据类型
### 数组(array)
固定长度的特定类型元素组成的序列，**值类型**
```go
// 定义
var a [3]int
b := [...]int{1,2,3}
c := [3]int{1,2,3}

// 含有100个元素的数组，最后一个元素被初始化为-1，其余的为0
d := [...]int{99: -1}

// 支持切片操作
```

### 切片(slice)
变长的序列，序列中每个元素都有相同的类型，一个slice类型一般写作[]T，其中T代表slice中元素的类型。
**零值为nil，引用类型**

元素的底层存储结构为数组，一个slice由三个部分构成：指针、长度和容量。指针指向第一个slice元素对应的底层数组元素的地址，要注意的是slice的第一个元素并不一定就是数组的第一个元素。长度指目前slice中已有元素的数目；长度不能超过容量，容量指目前slice最多能存放的元素个数。内置的len和cap函数分别返回slice的长度和容量。

和数组不同的是，slice之间不能比较，因此我们不能使用==操作符来判断两个slice是否含有全部相等元素。不过标准库提供了高度优化的bytes.Equal函数来判断两个字节型slice是否相等（[]byte），但是对于其他类型的slice，我们必须自己展开每个元素进行比较。

一个零值的slice等于nil。一个nil值的slice并没有底层数组。如果你需要测试一个slice是否是空的，使用len(s) == 0来判断，而不应该用s == nil来判断
```go
var s []int    // len(s) == 0, s == nil
s = nil        // len(s) == 0, s == nil
s = []int(nil) // len(s) == 0, s == nil
s = []int{}    // len(s) == 0, s != nil
```

内置的make函数创建一个指定元素类型、长度和容量的slice。容量部分可以省略，在这种情况下，容量将等于长度。
```go
make([]T, len)
make([]T, len, cap)
```

当调用内置的append函数向slice追加元素时，如果元素数量超过容量，会引发扩容操作，此时slice的指针所指向的数组会发生变更
```go {.line-numbers}
package main

import (
	"fmt"
)

func main() {
	// intslice为nil
	var intslice []int64
	
	// intslice的长度和容量为4
	intslice = append(intslice, 11, 22, 33, 44)
	fmt.Println(len(intslice), cap(intslice)) // output:4 4
	
	// intslice会扩充，长度和容量变为8
	intslice = append(intslice, 55, 66, 77, 88)
	fmt.Println(len(intslice), cap(intslice)) // output:8 8
	
	// 通过切片操作赋值给is1，此时is1和intslice底层指向同一个数组
	is1 := intslice[1:3:5]
	fmt.Println(len(is1), cap(is1)) // output: 2 4
	fmt.Println(is1) // output: [22 33]
	
	// is1追加一个元素，长度未超过容量，不会引起扩容，此时修改is1中的元素会影响intslice
	is1 = append(is1, 99)
	fmt.Println(len(is1), cap(is1)) // output: 3 4
	fmt.Println(is1) // output: [22 33 99]
	// intslice[3]也被修改为99了
	fmt.Println(intslice) // output: [11 22 33 99 55 66 77 88]
	
	// 继续追加元素，超过了容量，引起扩容，is1和intslice此时底层指向不同的数组，对is1的操作不会影响intslice
	is1 = append(is1, 990, 991, 992)
	fmt.Println(len(is1), cap(is1)) // output: 6 8
	fmt.Println(is1) // output: [22 33 99 990 991 992]
	// intslice并未被修改
	fmt.Println(intslice) // output: [11 22 33 99 55 66 77 88]
}
```

遍历切片时可以直接采用下标也可以采用for-range的方式
```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	ss := []int32{1, 2, 3, 4, 5}

	// 针对切片的第一种遍历方式,直接采用下标访问
	ssLen := len(ss)
	for i := 0; i < ssLen; i++ {
		fmt.Println(ss[i])
	}

	fmt.Println("------")

	// 针对切片的第二种遍历方式,使用for-range,也是采用下标访问
	for i := range ss {
		fmt.Println(ss[i])
	}

	fmt.Println("------")
	var wg sync.WaitGroup
	wg.Add(len(ss))
	// 针对切片的第三种遍历方式,也是使用for-range,但采用索引+值的方式,下文中索引使用了_(忽略该参数)
	// 需要注意value是个局部变量,生命周期归属这个for循环
	// 迭代时只会改变value所对应的值,value本身只会被声明一次,即在整个for循环内其地址是不会改变的
	for _, value := range ss {
		/* 注意和下面的go func方式进行比较
		   此种方式在https://goplay.space/上一直输出的是5(即切片中的最后一个元素)
		   但在本地windows下用vscode输出的结果是变化的,有时全部输出5,有时输出4和5...
		   此种方式存在竞态
		   使用go tool vet 可以进行检测：loop variable value captured by func literal
		*/
		go func() {
			// 此种方式在vscode上会直接显示告警 loop variable value captured by func literal
			fmt.Println(value)
			wg.Done()
		}()
	}

	wg.Wait()
	fmt.Println("------")

	wg.Add(len(ss))
	for _, value := range ss {
		// 此种方式会把切片中的元素打印一遍
		go func(i int32) {
			fmt.Println(i)
			wg.Done()
		}(value)
	}

	wg.Wait()
	fmt.Println("done")
}
```

### 字典(map)
map在底层是用哈希表实现的，哈希表是一种巧妙并且实用的数据结构。它是一个无序的key/value对的集合，其中所有的key都是不同的，然后通过给定的key可以在常数时间复杂度内检索、更新或删除对应的value。

**零值为nil，引用类型**

一个map就是一个哈希表的引用，map类型可以写为map[K]V，其中K和V分别对应key和value。map中所有的key都有相同的类型，所有的value也有着相同的类型，但是key和value之间可以是不同的数据类型。其中K对应的key必须是支持==比较运算符的数据类型，所以map可以通过测试key是否相等来判断是否已经存在。虽然浮点数类型也是支持相等运算符比较的，但是将浮点数用做key类型则是一个坏的想法。

可以通过内置函数make或字面值创建map
```go
// 通过内置make来创建
ages := make(map[string]int)

// 通过字面值创建,并初始化了2个元素
ages := map[string]int{
    "alice":   31,
    "charlie": 34,
}

// 通过key的下标进行访问对应的value
ages["alice"] = 32
fmt.Println(ages["alice"]) // "32"

// 通过key来访问value时，若key不存在，也不会报错，而是会返回value对应的零值
fmt.Println(ages["bob"]) // "0",bob并不存在于ages中,返回value的零值(0)
age := ages["bob"]  // age=0

// 若key存在则ok为true;若key不存在则ok为false。可通过这种方式来判断key是不是存在
// 判断一个元素是否存在必须采用此种方式，不能通过比较零值
age, ok := ages["bob"]

// 通过内置delete函数来删除元素
delete(ages, "alice")

// 空的map
ages := map[string]int{}
```
> **注意：**向一个nil值的map存入元素将导致一个panic异常

map中的元素并不是一个变量，因此我们不能对map的元素进行取址操作。禁止对map元素取址的原因是map可能随着元素数量的增长而重新分配更大的内存空间，从而可能导致之前的地址无效。
```go
_ = &ages["bob"] // compile error: cannot take address of map element
```

遍历map中全部的key/value对，可以使用range风格的for循环实现，和之前的slice遍历语法类似
```go
// name和age对应map中的key、value，迭代顺序是不确定的
for name, age := range ages {
    fmt.Printf("%s\t%d\n", name, age)
}
```

map的迭代顺序是不确定的，并且不同的哈希函数实现可能导致不同的遍历顺序。在实践中，遍历的顺序是随机的，每一次遍历的顺序都不相同。这是故意的，每次都使用随机的遍历顺序可以强制要求程序不会依赖具体的哈希函数实现。

hash结构中直接使用的Bucket数组，而不是Bucket*指针的数据，是一段连续的内存空间。

每个bucket中存放最多8个key/value对, 如果多于8个，那么会申请一个新的bucket，并将它与之前的bucket链起来(**称为溢出链overflow**)，溢出链的Bucket的空间是使用mallocgc分配的。

hash结构采用的可扩展哈希的算法。由hash值mod当前hash表大小决定某一个值属于哪个桶，而hash表大小是2的指数(2^B)。每次扩容，会增大到上次大小的两倍。结构体中有一个buckets和一个oldbuckets是用来实现增量扩容的。正常情况下直接使用buckets，而oldbuckets为空。如果当前哈希表正在扩容中，则oldbuckets不为空，并且buckets大小是oldbuckets大小的两倍。

按key的类型采用相应的hash算法得到key的hash值。将hash值的低位当作hmap结构体中buckets数组的index，找到key所在的bucket。将hash的高8位存储在了bucket的tophash中。**注意，这里高8位不是用来当作key/value在bucket内部的offset的，而是作为一个主键，在查找时对tophash数组的每一项进行顺序匹配的。**先比较hash值高位与bucket的tophash[i]是否相等，如果相等则再比较bucket的第i个的key与所给的key是否相等。如果相等，则返回其对应的value，反之，在overflow buckets中按照上述方法继续寻找。

> **注意：**Bucket中key/value的放置顺序，是将keys放在一起，values放在一起，为什么不将key和对应的value放在一起呢？如果那么做，存储结构将变成key1/value1/key2/value2… 设想如果是这样的一个map[int64]int8，考虑到字节对齐，会浪费很多存储空间。不得不说通过上述的一个小细节，可以看出Go在设计上的深思熟虑。

Go语言使用的是增量扩容。假设扩容之前容量为X，扩容之后容量为Y，对于某个哈希值hash，一般情况下(hash mod X)不等于(hash mod Y)，所以扩容之后要重新计算每一项在哈希表中的新位置。当hash表扩容之后，需要将那些旧的pair重新哈希到新的table上(源代码中称之为evacuate)， 这个工作并没有在扩容之后一次性完成，而是逐步的完成（在insert和remove时每次搬移1-2个pair），主要是为了缩短map容器的响应时间，避免扩容时阻塞(本质上还是将总的扩容时间分摊到了每一次哈希操作上面)。
> **注意：**如果key或value小于128字节，则它们的值是直接使用的bucket作为存储的。否则bucket中存储的是指向实际key/value数据的指针，

### 通道(channel)
类型标记为chan，它在栈上只是一个指针，实际的数据都是由指针所指向的堆上面。**零值为nil，引用类型**

一个channel是一个通信机制，它可以让一个goroutine通过它给另一个goroutine发送值信息。每个channel都有一个特殊的类型，也就是channels可发送数据的类型。一个可以发送int类型数据的channel一般写为chan int。
```go
// 通过内置make函数创建chan
ch := make(chan int)

// 先声明，再创建
var ch chan int
ch = make(chan int)

// 通过内置close函数关闭chan，此操作不是必须的，当没有被引用时，GC会回收
close(ch)
```

channel创建时默认时双向的，但Go语言也提供了单向的channel，分别表示用于只发送或只接收的channel。类型**chan<- int**表示一个只发送int的channel，只能发送不能接收。相反，类型**<-chan int**表示一个只接收int的channel，只能接收不能发送。（箭头<-和关键字chan的相对位置表明了channel的方向）这种限制将在编译期检测。

Go语言提供了无缓冲的channels和带缓冲的channels，在使用make创建时看是否提供了第二个参数。
```go
// 无缓冲channel
ch := make(chan int)

// 带缓冲channel,容量为100
ch := make(chan int, 100)

// 读channel时可忽略读取到的值
<-ch
```
- 读或写一个nil的channel的操作会永远阻塞
- 读一个已关闭的channel会立刻返回一个channel元素类型的零值
- 写一个已关闭的channel会导致panic
- 无缓冲channel是同步的，若发送者和接受者不是同时存在，则读或写将被阻塞
- 有缓冲channel是异步的，当容量为满时写将被阻塞，当容量为空时读将被阻塞

select-case 多路复用
```go
package main

import (
	"fmt"
	"os"
	"time"
)

func main() {
	abort := make(chan struct{})
	go func() {
		os.Stdin.Read(make([]byte, 1))
		abort <- struct{}{}
	}()

	tick := time.NewTicker(1 * time.Second)
	fmt.Println("Commencing countdown. Please return to abort.")
	for countdown := 10; countdown > 0; countdown-- {
		fmt.Println(countdown)
		/* case后必须是channel变量的操作
		   若不存在default分支且所有channel都未触发时，select将阻塞
		   若存在default分支且所有channel都未触发时，会立即执行default分支
		   若只有1个channel触发时，会立即执行对应的case分支代码
		   若同时多个channel触发时，会随机选择执行其中一个对应的case分支代码
		*/
		select {
		case <-abort:
			fmt.Println("Lauch aborted!")
			return
		case <-tick.C:
		}
	}

	tick.Stop()
	close(abort)
	fmt.Println("Lauching.")
}

// 读取channel时可返回2个参数，ok表示是否读取成功，当fileSize被关闭时会立即返回false
fileSize := make(chan int64)
case size, ok := <-fileSize
```
for-range 迭代，可循环获取channel上的数据，若channel上没有数据会被阻塞。当channel被close后for-range结束循环迭代

### 函数(function)
类型标记为func，函数声明包括函数名、形式参数列表、返回值列表（可省略）以及函数体。**零值为nil，引用类型**
```go
func name(parameter-list) (result-list) {
    body
}

// 当涉及多参数返回时，需要注意返回值类型是必须的，返回值命名是可选的（所有返回值要么都有命名，要么都没有）
// 命名的返回值可以在函数内部作为变量来使用
func name(parameter-list) (bool, error) {
	body
}

func name(parameter-list) (result bool, err error) {
	body
}

// 以下2种声明是错误的
func name(parameter-list) (result bool, error) {
	body
}

func name(parameter-list) (bool, err error) {
	body
}
```

函数的类型被称为函数的标识符。如果两个函数形式参数列表和返回值列表中的变量类型一一对应，那么这两个函数被认为有相同的类型和标识符。形参和返回值的变量名不影响函数标识符也不影响它们是否可以以省略参数类型的形式表示。

拥有函数名的函数只能在包级语法块中被声明，通过函数字面量（function literal），我们可绕过这一限制，在任何表达式中表示一个函数值。函数字面量的语法和函数声明相似，区别在于func关键字后没有函数名。函数值字面量是一种表达式，它的值被成为匿名函数（anonymous function）。
```go
// squares返回一个匿名函数。
// 该匿名函数每次被调用时都会返回下一个数的平方。
func squares() func() int {
    var x int
    return func() int {
        x++
        return x * x
    }
}

func main() {
    f := squares()
    fmt.Println(f()) // "1"
    fmt.Println(f()) // "4"
    fmt.Println(f()) // "9"
    fmt.Println(f()) // "16"
}
```

> **注意：**Go语言没有默认参数值，也没有任何方法可以通过参数名指定形参，因此形参和返回值的变量名对于函数调用者而言没有意义

### 方法
在函数声明时，在其名字之前放上一个变量，即是一个方法。这个附加的参数会将该函数附加到这种类型上，即相当于为这种类型定义了一个独占的方法。
```go
import "math"

// 结构体
type Point struct{ X, Y float64 }

// 函数
func Distance(p, q Point) float64 {
    return math.Hypot(q.X-p.X, q.Y-p.Y)
}

// Point结构的一个方法,
func (p Point) Distance(q Point) float64 {
    return math.Hypot(q.X-p.X, q.Y-p.Y)
}
```
上面的代码里那个附加的参数p，叫做方法的接收器(receiver)，早期的面向对象语言留下的遗产将调用一个方法称为“向一个对象发送消息”。可以任意的选择接收器的名字。

在方法调用过程中，接收器参数一般会在方法名之前出现。
```go
// 定义Point类型的变量
p := Point{1, 2}
q := Point{3, 4}

// 通过变量调用方法
r := p.Distance(q)
```

也可以采用指针来声明方法，如下：
```go
// 方法的接收器类型是*Point
func (p *Point) ScaleBy(factor float64) {
    p.X *= factor
    p.Y *= factor
}
```

- 不管你的method的receiver是指针类型还是非指针类型，都是可以通过指针/非指针类型进行调用的，编译器会帮你做类型转换。
- 在声明一个method的receiver该是指针还是非指针类型时，你需要考虑两方面的因素，第一方面是这个对象本身是不是特别大，如果声明为非指针变量时，调用会产生一次拷贝；第二方面是如果你用指针类型作为receiver，那么你一定要注意，这种指针类型指向的始终是一块内存地址，就算你对其进行了拷贝。
- 不能通过一个无法取到地址的接收器来调用指针方法，比如临时变量的内存地址就无法获取得到。Point{1, 2}.ScaleBy(2)是非法的，编译会报错

### 结构体(struct)
类型标记为struct，**零值为结构中各成员变量所对应的零值，值类型**
结构体是一种聚合的数据类型，是由零个或多个任意类型的值聚合成的实体。每个值称为结构体的成员。
```go
// 定义了结构体Employee
type Employee struct {
    ID        int
    Name      string
    Address   string
    DoB       time.Time
    Position  string
    Salary    int
    ManagerID int
}

// 声明变量，类型为Employee
var dilbert Employee
```
dilbert结构体变量的成员可以通过点操作符访问，比如dilbert.Name和dilbert.DoB。因为dilbert是一个变量，它所有的成员也同样是变量。
```go
// 直接对每个成员赋值
dilbert.Salary -= 5000

// 对成员取地址，然后通过指针访问
position := &dilbert.Position
*position = "Senior " + *position

// 点操作符也可以和指向结构体的指针一起工作：
var employeeOfTheMonth *Employee = &dilbert
employeeOfTheMonth.Position += " (proactive team player)"
```
> **注意：**结构体成员名字是以大写字母开头的，那么该成员就是导出的；这是Go语言导出规则决定的。一个结构体可能同时包含导出和未导出的成员。

结构体值也可以用结构体字面值表示，结构体字面值可以指定每个成员的值
```go
// 严格按照结构体定义的成员顺序
type Point struct{ X, Y int }
p := Point{1, 2}

// 以成员名字和相应的值来初始化，可以包含部分或全部的成员，成员出现的顺序不重要
p := Point{X: 1, Y: 2} // 初始化全部成员
anim := gif.GIF{LoopCount: nframes} // 初始化其中一个成员，其它成员默认为对应的零值

// 以上2种方式不能混用
```

如果结构体的全部成员都是可以比较的，那么结构体也是可以比较的，那样的话两个结构体将可以使用= =或!=运算符进行比较。相等比较运算符==将比较两个结构体的每个成员

Go语言提供的不同寻常的结构体嵌入机制，让一个命名的结构体包含另一个结构体类型的匿名成员，这样就可以通过简单的点运算符x.f来访问匿名成员链中嵌套的x.d.e.f成员。
```go
type Point struct {
    X, Y int
}

type Circle struct {
    Point // 匿名嵌套
    Radius int
}

type Wheel struct {
    Circle // 匿名嵌套
    Spokes int
}

// 基于匿名嵌入的特性，可以直接访问叶子属性而不需要给出完整的路径
var w Wheel
w.X = 8            // 等价于w.Circle.Point.X = 8
w.Y = 8            // 等价于w.Circle.Point.Y = 8
w.Radius = 5       // 等价于w.Circle.Radius = 5
w.Spokes = 20

// 结构体字面值并没有简短表示匿名成员的语法， 因此下面的语句都不能编译通过
w = Wheel{8, 8, 5, 20}                       // compile error: unknown fields
w = Wheel{X: 8, Y: 8, Radius: 5, Spokes: 20} // compile error: unknown fields

// 正确的做法
w = Wheel{
    Circle: Circle{
        Point:  Point{X: 8, Y: 8},
        Radius: 5,
    },
    Spokes: 20, // NOTE: 逗号是必须要的
}
```

因为匿名成员也有一个隐式的名字，因此不能同时包含两个类型相同的匿名成员，这会导致名字冲突。同时，因为成员的名字是由其类型隐式地决定的，所有匿名成员也有可见性的规则约束。在上面的例子中，Point和Circle匿名成员都是导出的。即使它们不导出（比如改成小写字母开头的point和circle），我们依然可以用简短形式访问匿名成员嵌套的成员。

但是在包外部，因为circle和point没有导出不能访问它们的成员，因此简短的匿名成员访问语法也是禁止的。

目前描述的匿名成员特性只是对访问嵌套成员的点运算符提供了简短的语法糖。匿名成员并不要求是结构体类型，其实任何命名的类型都可以作为结构体的匿名成员。

**简短的点运算符语法可以用于选择匿名成员嵌套的成员，也可以用于访问它们的方法。实际上，外层的结构体不仅仅是获得了匿名成员类型的所有成员，而且也获得了该类型导出的全部的方法。这个机制可以用于将一个有简单行为的对象组合成有复杂行为的对象。组合是Go语言中面向对象编程的核心**

### 接口(interface)
接口类型是一种抽象的类型，类型标记为interface，**零值为nil，引用类型**

接口类型具体描述了一系列方法的集合，一个实现了这些方法的具体类型是这个接口类型的实例。

**依赖于接口而不是实现，优先使用组合而不是继承，这是程序抽象的基本原则。**但是长久以来以C++为代表的“面向对象”语言曲解了这些原则，让人们走入了误区。为什么要将方法和数据绑死？为什么要有多重继承这么变态的设计？面向对象中最强调的应该是对象间的消息传递，却为什么被演绎成了封装继承和多态。面向对象是否实现程序程序抽象的合理途径，又或者是因为它存在我们就认为它合理了。历史原因，中间出现了太多的错误。不管怎么样，Go的interface给我们打开了一扇新的窗。
> 关于C++和面向对象的发展可以参考下孟岩的文章--[function/bind的救赎](https://blog.csdn.net/myan/article/details/5928531)

interface实际上就是一个结构体，包含两个成员。其中一个成员是指向具体数据的指针，另一个成员中包含了类型信息。
> **注意：**空接口和带方法的接口底层结构略有不同，带方法的接口除了包含类型信息还需要包含具体类型中已实现的方法

一个不包含任何值的nil接口和一个刚好包含nil指针的接口值是不同的。这个细微区别产生了一个容易绊倒每个Go程序员的陷阱。具体可参考官方文档，[国内链接](https://golang.google.cn/doc/faq#nil_error) [国外链接](http://golang.org/doc/faq#nil_error)
```go
package main

import (
	"fmt"
)

type Student struct {
	Name  string
}

func main() {
	var s *Student
	var it interface{}
	// 此时it为nil.output: nil
	if it == nil {
		fmt.Println("nil")
	} else {
		fmt.Println("non nil")
	}

	// 赋值后it本身不为nil了,其类型字段指向了*Student,其值指向的是nil.output: non nil
	it = s
	if it == nil {
		fmt.Println("nil")
	} else {
		fmt.Println("non nil")
	}
}

// 引用官方文档的例子，无论条件怎么变化始终会返回一个non-nil的error
func returnsError() error {
	var p *MyError = nil
	if bad() {
		p = ErrBad
	}
	return p // Will always return a non-nil error.
}
```

空的interface可以被当作任意类型来使用，它使得Go语言拥有了一定的动态性，但却又不损失静态语言在类型安全方面拥有的编译时检查的优势。


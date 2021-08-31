# golang字符串解析


## 字符编码
&ensp;&ensp;&ensp;&ensp;计算机世界中只能识别二进制,所有的信息最终都会表示成一个二进制的字符串,每一个二进制位有0和1两种状态.

&ensp;&ensp;&ensp;&ensp;当在计算机中存储字符'A'时,是如何存储的?在读取时又是如何还原为字符'A'的列?比如可以把'A'存储为`0100 0001`,然后在读取时把`0100 0001`还原为'A',`0100 0001`与'A'的对应关系应该是唯一的,这种唯一映射规则就是编码.通过这种编码规则可以把字符映射到唯一的一种状态(二进制字符串).

&ensp;&ensp;&ensp;&ensp;而最早出现的编码规则是ASCII码,在ASCII编码规则中,字符'A'对应的是`0100 0001`.

### ASCII码
&ensp;&ensp;&ensp;&ensp;在上世纪60年代,美国制定了一套字符编码,对英语字符与二进制位之间的关系,做了统一规定,这就是ASCII编码.

&ensp;&ensp;&ensp;&ensp;ASCII码规定了128个字符的编码,比如空格`SPACE`是32(二进制`00100000`),字母'A'是65(二进制`0100 0001`).这128个字符(包括32个不能打印出来的控制符号),只占用一个字节的后7位,最前面一位统一为0.

### UNICODE
&ensp;&ensp;&ensp;&ensp;英语用128个符号编码足够了,但世界上还有很多其它语言,128是远远不够的,如在法语中,字母上方有注音符号,它就无法用ASCII码表示.后来ISO组织制定了一些扩展规则,利用字节中闲置的最高位编入新的符号.如法语中`é`的编码为130(二进制`10000010`),这样可以表示最多256个符号.

&ensp;&ensp;&ensp;&ensp;至于亚洲国家的文字,使用的符号就更多了,汉字就多达10万左右.使用一个字节是远远不够的,必须使用多个字节来表示一个符号.比如后来的`GB2312`、`BIG5`、`GBK`编码规则就是定义的多个字节来表述中文符号.

&ensp;&ensp;&ensp;&ensp;不同的国家/组织采用不同的编码规则,导致世界上存在很多编码规则,同一个二进制串可以被解释成不同的符号.因此,在打开一个文件时就必须知道它的编码方式,否则用错误的编码方式去解读,就会出现乱码.以前的邮件经常出现乱码,就是因为发件人和收件人使用的编码方式不一样.

&ensp;&ensp;&ensp;&ensp;就需要有一种编码,将世界上所有的符号都纳入其中,每一个符号都给予独一无二的编码,那么乱码就会消失.这就是UNICODE,现在的规模可以容纳100多万个符号.每个符号的编码不一样,具体的符号对应表,可以查询[unicode.org](https://home.unicode.org/)或专门的[汉字对应表](http://www.chi2ko.com/tool/CJK.htm)

&ensp;&ensp;&ensp;&ensp;Unicode是指一张表,里面包含了可能出现的所有字符,每个字符对应一个数字,这个数字称为码点(Code Point),如字符'H'的码点为72(十进制),字符'李'的码点为26446(十进制).Unicode表包含了1114112个码点，即从000000(十六进制) - 10FFFF(十六进制).地球上所有字符都可以在Unicode表中找到对应的唯一码点.[点击这里](https://unicode-table.com/cn/),可查询字符对应的码点.

&ensp;&ensp;&ensp;&ensp;**需要注意,Unicode只是个符号集,它只规定了符号的二进制代码,却没有规定这个二进制代码应该如何存储**

&ensp;&ensp;&ensp;&ensp;比如汉字`周`的Unicode码点是`5468`,用二进制表示为`101 0100 0110 1000`,总共有15位,这个符号至少需要两个字节.表示其它更大的符号,可能需要3个字节或4个字节.这里引申出两个严重的问题:
1. 计算机怎么知道两个字节表示一个符号,而不是分别表示两个符号?
2. 英文字母只需要一个字节就可以表示,如果Unicode统一规定,每个符号用三个字节或四个字节表示,那每个英文字母前面必然有二到三个字节都是0,这对于存储是极大的浪费.

### UTF-8
&ensp;&ensp;&ensp;&ensp;互联网的普及,强烈要求出现一种统一的编码方式.`UTF-8`就是在互联网上使用最广的一种Unicode的实现方式.其它实现方式还包括`UTF-16`(字符用两个字节或四个字节表示)和`UTF-32`(字符用四个字节表示),不过在互联网上基本不用.

&ensp;&ensp;&ensp;&ensp;**注意:`UTF-8`是Unicode的实现方式之一**

&ensp;&ensp;&ensp;&ensp;`UTF-8`最大的一个特点: 就是它是一种变长的编码方式.它可以使用1~4个字节表示一个符号,根据不同的符号而变化字节长度.

&ensp;&ensp;&ensp;&ensp;`UTF-8`的编码规则很简单,只有二条:
1. 对于单字节的符号,字节的第一位设为0,后面7位为这个符号的Unicode码.因此对于英语字母,`UTF-8`编码和`ASCII码`是相同的.
2. 对于n字节的符号(n > 1),第一个字节的前n位都设为1,第n+1位设为0,后面字节的前两位一律设为10.剩下的没有提及的二进制位,全部为这个符号的`Unicode码`.

&ensp;&ensp;&ensp;&ensp;规则如下:
|Unicode符号范围(十六进制)|UTF-8编码方式(二进制)|
|:--|:--|
|0000 0000-0000 007F(0-127)|0xxxxxxx|
|0000 0080-0000 07FF(128-2047)|110xxxxx 10xxxxxx|
|0000 0800-0000 FFFF(2048-65535)|1110xxxx 10xxxxxx 10xxxxxx|
|0001 0000-0010 FFFF(65536-111411)|11110xxx 10xxxxxx 10xxxxxx 10xxxxxx|

&ensp;&ensp;&ensp;&ensp;根据上表,解读`UTF-8`编码非常简单.如果一个字节的第一位是0,则这个字节单独就是一个字符;如果第一位是1,则连续有多少个1,就表示当前字符占用多少个字节.

&ensp;&ensp;&ensp;&ensp;下面以汉字`周`为例,Unicode码点为`5468`(101 0100 0110 1000),根据上表,`5468`处在第三行的范围内(`0000 0800-0000 FFFF`),因此需要三个字节,即格式为`1110xxxx 10xxxxxx 10xxxxxx`,然后,从`周`的最后一个二进制位开始,依次从后向前填入格式中的`x`,多出的位补0.这样`周`的`UTF-8`编码为`1110 0101 10 010001 10 101000`,转换为十六进制就是`E5 91A8`

&ensp;&ensp;&ensp;&ensp;如下代码,就会打印出`e591a8`
``` go
name := "周"
fmt.Printf("%x", name)
```

### BOM头
&ensp;&ensp;&ensp;&ensp;对于`UTF-16`和`UTF-32`编码方式,是采用多字节编码,计算机就需要知道其顺序,如字符`A`的码点是65(十进制),十六进制为41,根据UTF-16来编码时会使用两个字节,但计算机是以字节为单位来存储的,那这两个字节应该表示为`0x0041`还是表示为`0x4100`?这就引出了字节序的问题,需要依赖BOM(`Byte Order Mark`)机制来解决.

&ensp;&ensp;&ensp;&ensp;若为`0x0041`表示采用了大端序(Big endian),而为`0x4100`表示采用了小端序(Little endian).

&ensp;&ensp;&ensp;&ensp;在UCS(Universal Multiple-Octet Coded Character Set,属于ISO组织,与unicode基本保持一致)编码中有一个叫做`Zero Width No-Break Space`(零宽无间断间隔)的字符,它的编码是`FEFF`.规范建议在传输字节流之前,先传输该字符,若收到`FEFF`就表示是`Big endian`的,若收到`FFFE`就表示是`Little endian`.因此该字符又被称为BOM.

&ensp;&ensp;&ensp;&ensp;注意字节顺序这个概念对`UTF-8`来说是没有意义的,因为在`UTF-8`编码中,其自身已经带了控制信息,如`1110xxxx 10xxxxxx 10xxxxxx 10xxxxxx`,其中1110就起到了控制作用,所以不需要额外的BOM机制.

&ensp;&ensp;&ensp;&ensp;但windows在保存UTF-8编码的文件时,会在文件开始的地方自动插入BOM,即三个字符`0xEF 0xBB 0xBF`(字符`Zero Width No-Break Space`的`UTF-8`编码是 EF BB BF).windows是利用BOM来标记文本文件的编码方式.

## string

### 原理
**内建类型**

代码在`src/builtin/builtin.go`
``` go
// string is the set of all strings of 8-bit bytes, conventionally but not
// necessarily representing UTF-8-encoded text. A string may be empty, but
// not nil. Values of string type are immutable.
type string string
```
`string`是8字节的集合,通常但并不一定是utf8编码的文本.可以为空,但不能为nil.且不可修改.

**底层结构**

代码在`src/runtime/string.go`里面
``` go
type stringStruct struct {
	str unsafe.Pointer
	len int
}
```
* `str`: 指针,指向存储实际字符串的地址.
* `len`: 字符串的长度,在代码中可以使用函数`len()`来获取该字段的值,**注意:是指实际的字节数,而不是字符数**.

可以采用如下方式来输出字符串底层结构字段的值
``` go
type stringStruct struct {
	str unsafe.Pointer
	len int
}

func main() {
	s := "hello world"
	fmt.Println(*(*stringStruct)(unsafe.Pointer(&s)))
}
```

输出结果为：`{0x4bbad0 11}`

`runtime.stringStruct`是非导出的,不能直接在外部使用,所有在代码中定义一个`stringStruct`结构,与`runtime.stringStruct`的字段保持一样.

**构建**

先根据字符串构建`stringStruct`,再转换为`string`.代码在`src/runtime/string.go`里面
``` go
//go:nosplit
func gostringnocopy(str *byte) string { // 根据字符串地址构建string
	ss := stringStruct{str: unsafe.Pointer(str), len: findnull(str)} // 先构造stringStruct
	s := *(*string)(unsafe.Pointer(&ss)) // 转换成string
	return s
}
```

### 索引和遍历
在使用`for-range`循环来遍历字符串时需要注意,`for-range`对多字节编码有特殊的支持.
``` go
func main() {
  s := "hi 高盈"

  for index, c := range s {
    fmt.Printf("%d %c\n", index, c)
  }
}
```
上面代码输出：
``` go
0 h
1 i
2  
3 高
6 盈
```
可以看出：***遍历时是按照字符来循环的(如果有多字节字符,就会导致索引不连续).***

### string和[]byte类型转换

<div id="standard"></div>

#### 标准转换
`[]byte`的底层结构如下:
``` go
type slice struct {
	array unsafe.Pointer
	len   int
	cap   int
}
```

* `string` -> `[]byte`: 语法为`[]byte(str)`.
* `[]byte` -> `string`: 语法为`string(bs)`.

`string`转`[]byte`的实现在`src/runtime/string.go`中
``` go
// The constant is known to the compiler.
// There is no fundamental theory behind this number.
const tmpStringBufSize = 32

type tmpBuf [tmpStringBufSize]byte

func stringtoslicebyte(buf *tmpBuf, s string) []byte {
	var b []byte
	if buf != nil && len(s) <= len(buf) {
		*buf = tmpBuf{}
		b = buf[:len(s)]
	} else {
		b = rawbyteslice(len(s)) // 当长度超过了32,需要新申请一块内存.
	}
	copy(b, s)
	return b
}

// rawbyteslice allocates a new byte slice. The byte slice is not zeroed.
func rawbyteslice(size int) (b []byte) {
	cap := roundupsize(uintptr(size))
	p := mallocgc(cap, nil, false)
	if cap != uintptr(size) {
		memclrNoHeapPointers(add(p, uintptr(size)), cap-uintptr(size))
	}

	*(*slice)(unsafe.Pointer(&b)) = slice{p, size, int(cap)}
	return
}
```
注意: 当`s`的长度大于32时,需要调用`mallocgc`分配一块新的内存来存放`[]byte`.在转换时,如果字符串比较长(字节数超过32),标准转换方式会有一次内存分配的操作.

`[]byte`转`string`的实现也在`src/runtime/string.go`中
``` go
// slicebytetostring converts a byte slice to a string.
// It is inserted by the compiler into generated code.
// ptr is a pointer to the first element of the slice;
// n is the length of the slice.
// Buf is a fixed-size buffer for the result,
// it is not nil if the result does not escape.
func slicebytetostring(buf *tmpBuf, ptr *byte, n int) (str string) {
	if n == 0 {
		// Turns out to be a relatively common case.
		// Consider that you want to parse out data between parens in "foo()bar",
		// you find the indices and convert the subslice to string.
		return ""
	}
	if raceenabled {
		racereadrangepc(unsafe.Pointer(ptr),
			uintptr(n),
			getcallerpc(),
			funcPC(slicebytetostring))
	}
	if msanenabled {
		msanread(unsafe.Pointer(ptr), uintptr(n))
	}
	if n == 1 {
		p := unsafe.Pointer(&staticuint64s[*ptr]) // 当长度为1时,使用静态数组staticuint64s,其保存了从0x00->0xFF之间的数字.
		if sys.BigEndian {
			p = add(p, 7)
		}
		stringStructOf(&str).str = p
		stringStructOf(&str).len = 1
		return
	}

	var p unsafe.Pointer
	if buf != nil && n <= len(buf) {
		p = unsafe.Pointer(buf)
	} else {
		p = mallocgc(uintptr(n), nil, false) // 长度超过32时,需要新申请一块内存.
	}
	stringStructOf(&str).str = p
	stringStructOf(&str).len = n
	memmove(p, unsafe.Pointer(ptr), uintptr(n))
	return
}
```

#### 强转换

`string`和`[]byte`的底层结构是很类似的,可以通过`unsafe`和`reflect`包来强制转换
``` go
func String2Bytes(str string) []byte {
	sh := (*reflect.StringHeader)(unsafe.Pointer(&str))
	bh := reflect.SliceHeader{
		Data: sh.Data,
		Len:  sh.Len,
		Cap:  sh.Len,
	}

	return *(*[]byte)(unsafe.Pointer(&bh))
}

func Bytes2String(b []byte) string {
	return *(*string)(unsafe.Pointer(&b))
}
```

#### 性能比较
``` go
import "testing"

func Benchmark_NormalBytes2String(b *testing.B) {
	x := []byte("Going International Group")
	for i := 0; i < b.N; i++ {
		_ = string(x)
	}
}

func Benchmark_Bytes2String(b *testing.B) {
	x := []byte("Going International Group")
	for i := 0; i < b.N; i++ {
		_ = Bytes2String(x)
	}
}

func Benchmark_NormalString2Bytes(b *testing.B) {
	x := "Going International Group"
	for i := 0; i < b.N; i++ {
		_ = []byte(x)
	}
}

func Benchmark_String2Bytes(b *testing.B) {
	x := "Going International Group"
	for i := 0; i < b.N; i++ {
		_ = String2Bytes(x)
	}
}

func Benchmark_NormalBytes2String1(b *testing.B) {
	x := []byte("Going International Group, Going International Group, Going International Group, Going International Group, Going International Group")
	for i := 0; i < b.N; i++ {
		_ = string(x)
	}
}

func Benchmark_Bytes2String1(b *testing.B) {
	x := []byte("Going International Group, Going International Group, Going International Group, Going International Group, Going International Group")
	for i := 0; i < b.N; i++ {
		_ = Bytes2String(x)
	}
}

func Benchmark_NormalString2Bytes1(b *testing.B) {
	x := "Going International Group, Going International Group, Going International Group, Going International Group, Going International Group"
	for i := 0; i < b.N; i++ {
		_ = []byte(x)
	}
}

func Benchmark_String2Bytes1(b *testing.B) {
	x := "Going International Group, Going International Group, Going International Group, Going International Group, Going International Group"
	for i := 0; i < b.N; i++ {
		_ = String2Bytes(x)
	}
}
```

运行结果:
``` bash
$ go test -bench=. -benchmem
goos: linux
goarch: amd64
pkg: string
cpu: Intel(R) Core(TM) i7-8700 CPU @ 3.20GHz
Benchmark_NormalBytes2String-6    	254173088	         4.320 ns/op	       0 B/op	       0 allocs/op
Benchmark_Bytes2String-6          	1000000000	         0.2536 ns/op	       0 B/op	       0 allocs/op
Benchmark_NormalString2Bytes-6    	163714125	         7.257 ns/op	       0 B/op	       0 allocs/op
Benchmark_String2Bytes-6          	1000000000	         0.2647 ns/op	       0 B/op	       0 allocs/op
Benchmark_NormalBytes2String1-6   	18545620	        57.28 ns/op	      96 B/op	       1 allocs/op
Benchmark_Bytes2String1-6         	1000000000	         0.3686 ns/op	       0 B/op	       0 allocs/op
Benchmark_NormalString2Bytes1-6   	14502368	        70.26 ns/op	      96 B/op	       1 allocs/op
Benchmark_String2Bytes1-6         	1000000000	         0.2787 ns/op	       0 B/op	       0 allocs/op
PASS
ok  	string	7.124s
```
注意: 当字符串比较长时,`string`和`[]byte`的标准转换会额外进行一次内存分配([参考上面的标准转换](#standard)).

#### 标准转换的实现
如下代码:
``` go
package main
 
import "fmt"
 
func main() {
    str := "steven"
    b := []byte(str)
 
    str1 := string(b)
    fmt.Println(str1)
}
```
使用指令`go tool compile -N -l -S main.go`,可以看到`string`转换为`[]byte`是调用的`runtme.stringtoslicebyte`,`[]byte`转换为`string`是调用的`runtime.slicebytetostring`
``` bash
$ go tool compile -N -l -S main.go
"".main STEXT size=382 args=0x0 locals=0xc8 funcid=0x0
	0x0000 00000 (main.go:5)	TEXT	"".main(SB), ABIInternal, $200-0
	0x0000 00000 (main.go:5)	MOVQ	(TLS), CX
	0x0009 00009 (main.go:5)	LEAQ	-72(SP), AX
	0x000e 00014 (main.go:5)	CMPQ	AX, 16(CX)
	0x0012 00018 (main.go:5)	PCDATA	$0, $-2
	0x0012 00018 (main.go:5)	JLS	372
	0x0018 00024 (main.go:5)	PCDATA	$0, $-1
	0x0018 00024 (main.go:5)	SUBQ	$200, SP
	0x001f 00031 (main.go:5)	MOVQ	BP, 192(SP)
	0x0027 00039 (main.go:5)	LEAQ	192(SP), BP
	0x002f 00047 (main.go:5)	FUNCDATA	$0, gclocals·69c1753bd5f81501d95132d08af04464(SB)
	0x002f 00047 (main.go:5)	FUNCDATA	$1, gclocals·6cb97e7811aee91aade906c749dc574d(SB)
	0x002f 00047 (main.go:5)	FUNCDATA	$2, "".main.stkobj(SB)
	0x002f 00047 (main.go:6)	LEAQ	go.string."steven"(SB), AX
	0x0036 00054 (main.go:6)	MOVQ	AX, "".str+112(SP)
	0x003b 00059 (main.go:6)	MOVQ	$6, "".str+120(SP)
	0x0044 00068 (main.go:7)	LEAQ	""..autotmp_4+48(SP), CX
	0x0049 00073 (main.go:7)	MOVQ	CX, (SP)
	0x004d 00077 (main.go:7)	MOVQ	AX, 8(SP)
	0x0052 00082 (main.go:7)	MOVQ	$6, 16(SP)
	0x005b 00091 (main.go:7)	PCDATA	$1, $0
	0x005b 00091 (main.go:7)	NOP
	0x0060 00096 (main.go:7)	CALL	runtime.stringtoslicebyte(SB)
	0x0065 00101 (main.go:7)	MOVQ	24(SP), AX
	0x006a 00106 (main.go:7)	MOVQ	32(SP), CX
	0x006f 00111 (main.go:7)	MOVQ	40(SP), DX
	0x0074 00116 (main.go:7)	MOVQ	AX, "".b+144(SP)
	0x007c 00124 (main.go:7)	MOVQ	CX, "".b+152(SP)
	0x0084 00132 (main.go:7)	MOVQ	DX, "".b+160(SP)
	0x008c 00140 (main.go:9)	MOVQ	$0, (SP)
	0x0094 00148 (main.go:9)	MOVQ	AX, 8(SP)
	0x0099 00153 (main.go:9)	MOVQ	CX, 16(SP)
	0x009e 00158 (main.go:9)	NOP
	0x00a0 00160 (main.go:9)	CALL	runtime.slicebytetostring(SB)
```

**runtime.stringtoslicebyte**

代码在`src\runtime\string.go`里.
``` go
// The constant is known to the compiler.
// There is no fundamental theory behind this number.
const tmpStringBufSize = 32

type tmpBuf [tmpStringBufSize]byte

func stringtoslicebyte(buf *tmpBuf, s string) []byte {
	var b []byte
	if buf != nil && len(s) <= len(buf) {
		*buf = tmpBuf{}
		b = buf[:len(s)]
	} else {
		b = rawbyteslice(len(s))
	}
	copy(b, s)
	return b
}

// rawbyteslice allocates a new byte slice. The byte slice is not zeroed.
func rawbyteslice(size int) (b []byte) {
	cap := roundupsize(uintptr(size))
	p := mallocgc(cap, nil, false)
	if cap != uintptr(size) {
		memclrNoHeapPointers(add(p, uintptr(size)), cap-uintptr(size))
	}

	*(*slice)(unsafe.Pointer(&b)) = slice{p, size, int(cap)}
	return
}
```
预先定义了一个长度为32的数组,如果字符串的长度超过了这个数组的长度,说明`[]byte`不够用了,需要重新分配内存.`32`是阈值,只有超过了才会进行内存分配.

最后通过调用`copy`方法实现`string`到`[]byte`的拷贝,具体实现在`src/runtime/slice.go`中的`slicestringcopy`方法.
``` go
func slicestringcopy(toPtr *byte, toLen int, fm string) int {
	if len(fm) == 0 || toLen == 0 {
		return 0
	}

	n := len(fm)
	if toLen < n {
		n = toLen
	}

	if raceenabled {
		callerpc := getcallerpc()
		pc := funcPC(slicestringcopy)
		racewriterangepc(unsafe.Pointer(toPtr), uintptr(n), callerpc, pc)
	}
	if msanenabled {
		msanwrite(unsafe.Pointer(toPtr), uintptr(n))
	}

	memmove(unsafe.Pointer(toPtr), stringStructOf(&fm).str, uintptr(n))
	return n
}
```
核心思路是:将string的底层数组从头部复制n个到[]byte对应的底层数组中.

**runtime.slicebytetostring**

代码在`src\runtime\string.go`里.
``` go
// slicebytetostring converts a byte slice to a string.
// It is inserted by the compiler into generated code.
// ptr is a pointer to the first element of the slice;
// n is the length of the slice.
// Buf is a fixed-size buffer for the result,
// it is not nil if the result does not escape.
func slicebytetostring(buf *tmpBuf, ptr *byte, n int) (str string) {
	if n == 0 {
		// Turns out to be a relatively common case.
		// Consider that you want to parse out data between parens in "foo()bar",
		// you find the indices and convert the subslice to string.
		return ""
	}
	if raceenabled {
		racereadrangepc(unsafe.Pointer(ptr),
			uintptr(n),
			getcallerpc(),
			funcPC(slicebytetostring))
	}
	if msanenabled {
		msanread(unsafe.Pointer(ptr), uintptr(n))
	}
	if n == 1 {
		p := unsafe.Pointer(&staticuint64s[*ptr])
		if sys.BigEndian {
			p = add(p, 7)
		}
		stringStructOf(&str).str = p
		stringStructOf(&str).len = 1
		return
	}

	var p unsafe.Pointer
	if buf != nil && n <= len(buf) {
		p = unsafe.Pointer(buf)
	} else {
		p = mallocgc(uintptr(n), nil, false)
	}
	stringStructOf(&str).str = p
	stringStructOf(&str).len = n
	memmove(p, unsafe.Pointer(ptr), uintptr(n))
	return
}
```
会根据`[]byte`的长度来决定是否重新分配内存,最后通过`memmove`拷贝数组到字符串中.

#### 如何取舍
非必要推荐使用标准转换方式,标准的才是安全的.除非是特别的高性能场景,才应该考虑强转换方式,但强转换是不安全的,使用上要特别注意.
``` go
func String2Bytes(str string) []byte {
	sh := (*reflect.StringHeader)(unsafe.Pointer(&str))
	bh := reflect.SliceHeader{
		Data: sh.Data,
		Len:  sh.Len,
		Cap:  sh.Len,
	}

	return *(*[]byte)(unsafe.Pointer(&bh))
}

func main() {
	str1 := "steven"
	b := String2Bytes(str1)
	b[0] = 'H'
}
```

运行会报错,如下,这种错误是程序直接发生了严重错误,使用`defer+recover`也是无法捕获的.`string`的底层数据是不允许更改的,这种方式相当于更改了`string`的内容,就会出现问题导致整个程序`dump`.
``` bash
unexpected fault address 0x49d1f4
fatal error: fault
[signal SIGSEGV: segmentation violation code=0x2 addr=0x49d1f4 pc=0x47dfdc]

goroutine 1 [running]:
runtime.throw(0x49d054, 0x5)
	/opt/go/src/runtime/panic.go:1117 +0x72 fp=0xc000072f08 sp=0xc000072ed8 pc=0x431032
runtime.sigpanic()
	/opt/go/src/runtime/signal_unix.go:741 +0x268 fp=0xc000072f40 sp=0xc000072f08 pc=0x445ba8
main.main()
	/home/steven/go/module/test/strtobyte/main.go:22 +0x5c fp=0xc000072f88 sp=0xc000072f40 pc=0x47dfdc
runtime.main()
	/opt/go/src/runtime/proc.go:225 +0x256 fp=0xc000072fe0 sp=0xc000072f88 pc=0x433876
runtime.goexit()
	/opt/go/src/runtime/asm_amd64.s:1371 +0x1 fp=0xc000072fe8 sp=0xc000072fe0 pc=0x461d21
```

#### 编译器优化

出于性能上的考虑,有些场景编译器做了优化不涉及到内存拷贝,主要是`[]byte`临时转换为`string`的场景.
* map查找: `m[string(b)]`
* 字符串拼接: `"<" + string(b) + ">"`
* 字符串比较: `string(b) == "foo"`
以上string只是临时使用,期间切片不会发生变化,所以可以进行优化,共享底层存储空间.

### 字符串拼接

#### +拼接
最常用最简单的就是通过`+`来拼接字符串
``` go
func StringPlus() string {
	var s string
	s += "steven" + " zhou"
	s += " works" + " at"
	s += " Going International Group"

	return s
}
```
使用指令`go tool compile -N -l -S main.go`来查看具体的实现.
``` bash
$ go tool compile -N -l -S main.go 
"".StringPlus STEXT size=269 args=0x10 locals=0x50 funcid=0x0
	0x0000 00000 (main.go:7)	TEXT	"".StringPlus(SB), ABIInternal, $80-16
	0x0000 00000 (main.go:7)	MOVQ	(TLS), CX
	0x0009 00009 (main.go:7)	CMPQ	SP, 16(CX)
	0x000d 00013 (main.go:7)	PCDATA	$0, $-2
	0x000d 00013 (main.go:7)	JLS	259
	0x0013 00019 (main.go:7)	PCDATA	$0, $-1
	0x0013 00019 (main.go:7)	SUBQ	$80, SP
	0x0017 00023 (main.go:7)	MOVQ	BP, 72(SP)
	0x001c 00028 (main.go:7)	LEAQ	72(SP), BP
	0x0021 00033 (main.go:7)	FUNCDATA	$0, gclocals·2a5305abe05176240e61b8620e19a815(SB)
	0x0021 00033 (main.go:7)	FUNCDATA	$1, gclocals·f207267fbf96a0178e8758c6e3e0ce28(SB)
	0x0021 00033 (main.go:7)	XORPS	X0, X0
	0x0024 00036 (main.go:7)	MOVUPS	X0, "".~r0+88(SP)
	0x0029 00041 (main.go:8)	XORPS	X0, X0
	0x002c 00044 (main.go:8)	MOVUPS	X0, "".s+56(SP)
	0x0031 00049 (main.go:9)	XORPS	X0, X0
	0x0034 00052 (main.go:9)	MOVUPS	X0, (SP)
	0x0038 00056 (main.go:9)	MOVQ	$0, 16(SP)
	0x0041 00065 (main.go:9)	LEAQ	go.string."steven zhou"(SB), AX
	0x0048 00072 (main.go:9)	MOVQ	AX, 24(SP)
	0x004d 00077 (main.go:9)	MOVQ	$11, 32(SP)
	0x0056 00086 (main.go:9)	PCDATA	$1, $0
	0x0056 00086 (main.go:9)	CALL	runtime.concatstring2(SB)
```
需要注意`"steven" + " zhou"`直接被合并为`"steven zhou"`,编译器已经做了优化.

`+`的具体实现是`runtime.concatstring2(SB)`函数.代码在`src/runtime/string.go`里
``` go
// The constant is known to the compiler.
// There is no fundamental theory behind this number.
const tmpStringBufSize = 32

type tmpBuf [tmpStringBufSize]byte

// concatstrings implements a Go string concatenation x+y+z+...
// The operands are passed in the slice a.
// If buf != nil, the compiler has determined that the result does not
// escape the calling function, so the string data can be stored in buf
// if small enough.
func concatstrings(buf *tmpBuf, a []string) string {
	idx := 0
	l := 0
	count := 0
	for i, x := range a {
		n := len(x)
		if n == 0 {
			continue
		}
		if l+n < l {
			throw("string concatenation too long")
		}
		l += n
		count++
		idx = i
	}
	if count == 0 {
		return ""
	}

	// If there is just one string and either it is not on the stack
	// or our result does not escape the calling frame (buf != nil),
	// then we can return that string directly.
	if count == 1 && (buf != nil || !stringDataOnStack(a[idx])) {
		return a[idx]
	}
	s, b := rawstringtmp(buf, l)
	for _, x := range a {
		copy(b, x)
		b = b[len(x):]
	}
	return s
}

func concatstring2(buf *tmpBuf, a [2]string) string {
	return concatstrings(buf, a[:])
}

func concatstring3(buf *tmpBuf, a [3]string) string {
	return concatstrings(buf, a[:])
}

func concatstring4(buf *tmpBuf, a [4]string) string {
	return concatstrings(buf, a[:])
}

func concatstring5(buf *tmpBuf, a [5]string) string {
	return concatstrings(buf, a[:])
}

// stringDataOnStack reports whether the string's data is
// stored on the current goroutine's stack.
func stringDataOnStack(s string) bool {
	ptr := uintptr(stringStructOf(&s).str)
	stk := getg().stack
	return stk.lo <= ptr && ptr < stk.hi
}

func rawstringtmp(buf *tmpBuf, l int) (s string, b []byte) {
	if buf != nil && l <= len(buf) {
		b = buf[:l]
		s = slicebytetostringtmp(&b[0], len(b))
	} else {
		s, b = rawstring(l)
	}
	return
}
```
内置了`concatstring2`到`concatstring5`,但实际上最终都是调用`concatstring`函数.
1. 先计算所有字符串的累加长度.
2. 如果有效的字符串个数为0,直接返回"".
3. 如果只有一个字符串且该字符串不在当前栈空间内或返回结果的字符串没有逃逸,则直接返回该字符串本身.
4. 如果buf不为nil且累计长度没有超过32,直接使用该数组空间.否则就重新分配一块内存.
5. 依次循环拷贝字符串.

#### fmt拼接
使用`fmt.Sprint`的系列函数来进行拼接,然后返回拼接的字符串.
``` go
func StringFmt() string {
	return fmt.Sprint("steven", " zhou", " works", " at", " Going International Group")
}
```

#### join拼接
使用`strings.Join`函数进行拼接,接受一个字符串数组,转换为一个拼接好的字符串.
``` go
func StringJoin() string {
	s := []string{"steven", " zhou", " works", " at", " Going International Group"}
	return strings.Join(s, "")
}
```

#### buffer拼接
使用`bytes.Buffer`来进行拼接,该结构体不止可以拼接字符串,还可以是`byte`,`rune`等,且实现了`io.Writer`接口.
``` go
func StringBuffer() string {
	var b bytes.Buffer
	b.WriteString("steven")
	b.WriteString(" zhou")
	b.WriteString(" works")
	b.WriteString(" at")
	b.WriteString(" Going International Group")
	return b.String()
}
```

#### builder拼接
使用`strings.Builder`来进行拼接,使用方式和`bytes.Buffer`类似.`1.10`版本提供的,主要是为了改善`bytes.Buffer`的拼接性能.
``` go
func StringBuilder() string {
	var b strings.Builder
	b.WriteString("steven")
	b.WriteString(" zhou")
	b.WriteString(" works")
	b.WriteString(" at")
	b.WriteString(" Going International Group")
	return b.String()
}
```

#### 性能分析
测试代码:
``` go
func Benchmark_Plus(b *testing.B) {
	for i := 0; i < b.N; i++ {
		StringPlus()
	}
}

func Benchmark_Fmt(b *testing.B) {
	for i := 0; i < b.N; i++ {
		StringFmt()
	}
}

func Benchmark_Join(b *testing.B) {
	for i := 0; i < b.N; i++ {
		StringJoin()
	}
}

func Benchmark_Buffer(b *testing.B) {
	for i := 0; i < b.N; i++ {
		StringBuffer()
	}
}

func Benchmark_Builder(b *testing.B) {
	for i := 0; i < b.N; i++ {
		StringBuilder()
	}
}
```
性能测试结果:
``` bash
$ go test -bench=. -benchmem                        
goos: linux
goarch: amd64
pkg: strjoin
cpu: Intel(R) Core(TM) i7-8700 CPU @ 3.20GHz
Benchmark_Plus-6      	 9745922	       120.5 ns/op	      72 B/op	       2 allocs/op
Benchmark_Fmt-6       	 7163672	       192.0 ns/op	      48 B/op	       1 allocs/op
Benchmark_Join-6      	13101292	        91.44 ns/op	      48 B/op	       1 allocs/op
Benchmark_Buffer-6    	10681605	       121.6 ns/op	     112 B/op	       2 allocs/op
Benchmark_Builder-6   	 7664673	       167.2 ns/op	     120 B/op	       4 allocs/op
PASS
ok  	strjoin	7.030s
```
`strings.Join`耗时最短,而`fmt.Sprint`耗时最长,性能最差.这种情况下`bytes.Buffer`和`strings.Builder`好像没什么优势.

关注下`+`方式,多次使用了`+`,但总共只分配了2次内存,在前面也已经分析过,编译器做了优化,直接合并了字符串.

把代码改造下,使用更多的字符串来拼接.
``` go
func StringPlus(ss []string) string {
	var s string
	l := len(ss)
	for i := 0; i < l; i++ {
		s += ss[i]
	}

	return s
}

func StringFmt(ss []interface{}) string {
	return fmt.Sprint(ss...)
}

func StringJoin(ss []string) string {
	return strings.Join(ss, "")
}

func StringBuffer(ss []string) string {
	var b bytes.Buffer
	l := len(ss)
	for i := 0; i < l; i++ {
		b.WriteString(ss[i])
	}
	return b.String()
}

func StringBuilder(ss []string) string {
	var b strings.Builder
	l := len(ss)
	for i := 0; i < l; i++ {
		b.WriteString(ss[i])
	}
	return b.String()
}

const content = "steven zhou works at Going International Group"
func initStrings(N int) []string {
	ss := make([]string, N)
	for i := 0; i < N; i++ {
		ss[i] = content
	}

	return ss
}

func initInterface(N int) []interface{} {
	ss := make([]interface{}, N)
	for i := 0; i < N; i++ {
		ss[i] = content
	}

	return ss
}
```

测试代码:
``` go
func Benchmark_Plus(b *testing.B) {
	ss := initStrings(10)
	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		StringPlus(ss)
	}
}

func Benchmark_Fmt(b *testing.B) {
	ss := initInterface(10)
	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		StringFmt(ss)
	}
}

func Benchmark_Join(b *testing.B) {
	ss := initStrings(10)
	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		StringJoin(ss)
	}
}

func Benchmark_Buffer(b *testing.B) {
	ss := initStrings(10)
	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		StringBuffer(ss)
	}
}

func Benchmark_Builder(b *testing.B) {
	ss := initStrings(10)
	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		StringBuilder(ss)
	}
}
```
测试结果:
``` bash
$ go test -bench=. -benchmem
goos: linux
goarch: amd64
pkg: strjoin
cpu: Intel(R) Core(TM) i7-8700 CPU @ 3.20GHz
Benchmark_Plus-6      	  835051	      1479 ns/op	    2592 B/op	       9 allocs/op
Benchmark_Fmt-6       	 2368869	       532.0 ns/op	     480 B/op	       1 allocs/op
Benchmark_Join-6      	 3631810	       321.2 ns/op	     480 B/op	       1 allocs/op
Benchmark_Buffer-6    	 1183435	      1165 ns/op	    2032 B/op	       5 allocs/op
Benchmark_Builder-6   	 1484762	       771.1 ns/op	    1488 B/op	       5 allocs/op
PASS
ok  	strjoin	9.385s
```
随着字符串数量加多,`+`拼接时每次都会涉及内存分配次数,导致耗时越长,10个字符串拼接供分配9次.

把字符串个数修改为100,继续测试:
``` bash
$ go test -bench=. -benchmem
goos: linux
goarch: amd64
pkg: strjoin
cpu: Intel(R) Core(TM) i7-8700 CPU @ 3.20GHz
Benchmark_Plus-6      	   10507	    104382 ns/op	  249441 B/op	      99 allocs/op
Benchmark_Fmt-6       	  256449	      5048 ns/op	    4865 B/op	       1 allocs/op
Benchmark_Join-6      	  333220	      3143 ns/op	    4864 B/op	       1 allocs/op
Benchmark_Buffer-6    	  118428	      8802 ns/op	   20496 B/op	       8 allocs/op
Benchmark_Builder-6   	  163093	      7220 ns/op	   16080 B/op	      10 allocs/op
PASS
ok  	strjoin	6.813s
```
`+`拼接共分配内存99次,性能越来越差,依然是`strings.Join`的表现最好.

把字符串个数修改为1000,继续测试:
``` bash
$ go test -bench=. -benchmem
goos: linux
goarch: amd64
pkg: strjoin
cpu: Intel(R) Core(TM) i7-8700 CPU @ 3.20GHz
Benchmark_Plus-6      	     132	   7778708 ns/op	24993274 B/op	    1000 allocs/op
Benchmark_Fmt-6       	   24487	     44290 ns/op	   49189 B/op	       1 allocs/op
Benchmark_Join-6      	   44905	     28059 ns/op	   49152 B/op	       1 allocs/op
Benchmark_Buffer-6    	   19155	     53682 ns/op	  165136 B/op	      11 allocs/op
Benchmark_Builder-6   	   15176	     76206 ns/op	  228304 B/op	      19 allocs/op
PASS
ok  	strjoin	8.737s
```
`strings.Builder`共分配了19次内存,`bytes.Buffer`共分配了11次内存,`strings.Join`和`fmt.Sprint`都只分配了1次内存,性能表现也是和内存分配次数有直接关系.

要想优化`strings.Builder`和`bytes.Buffer`的性能,必定要减少其内存分配次数.

这两个结构的底层都是采用`[]byte`来存储数据的,如下:
``` go
// A Builder is used to efficiently build a string using Write methods.
// It minimizes memory copying. The zero value is ready to use.
// Do not copy a non-zero Builder.
type Builder struct {
	addr *Builder // of receiver, to detect copies by value
	buf  []byte
}

// A Buffer is a variable-sized buffer of bytes with Read and Write methods.
// The zero value for Buffer is an empty buffer ready to use.
type Buffer struct {
	buf      []byte // contents are the bytes buf[off : len(buf)]
	off      int    // read at &buf[off], write at &buf[len(buf)]
	lastRead readOp // last read operation, so that Unread* can work correctly.
}
```
当底层slice容量不够,就会引发扩容,则就会涉及内存分配,如果提前预设`[]byte`的容量列?

修改函数,先调用`Grow`方法,预设容量.
``` go
func StringBuffer(ss []string) string {
	var b bytes.Buffer
	b.Grow(1000*len(content))
	l := len(ss)
	for i := 0; i < l; i++ {
		b.WriteString(ss[i])
	}
	return b.String()
}

func StringBuilder(ss []string) string {
	var b strings.Builder
	b.Grow(1000*len(content))
	l := len(ss)
	for i := 0; i < l; i++ {
		b.WriteString(ss[i])
	}
	return b.String()
}
```
再运行测试:
``` bash
$ go test -bench=. -benchmem
goos: linux
goarch: amd64
pkg: strjoin
cpu: Intel(R) Core(TM) i7-8700 CPU @ 3.20GHz
Benchmark_Fmt-6       	   28171	     44338 ns/op	   49202 B/op	       1 allocs/op
Benchmark_Join-6      	   40654	     29231 ns/op	   49152 B/op	       1 allocs/op
Benchmark_Buffer-6    	   38374	     35796 ns/op	   98304 B/op	       2 allocs/op
Benchmark_Builder-6   	   59942	     21109 ns/op	   49152 B/op	       1 allocs/op
PASS
ok  	strjoin	6.365s
```
`strings.Builder`只分配1次内存,性能表现最好,而`bytes.Buffer`分配了2次内存,其底层容量已经足够,为什么还有2次内存分配?

分析`String()`的源码:
``` go
// String returns the contents of the unread portion of the buffer
// as a string. If the Buffer is a nil pointer, it returns "<nil>".
//
// To build strings more efficiently, see the strings.Builder type.
func (b *Buffer) String() string {
	if b == nil {
		// Special case, useful in debugging.
		return "<nil>"
	}
	return string(b.buf[b.off:]) // 采用标准转换方式,把[]byte转换为string,会涉及一次内存分配.
}

// String returns the accumulated string.
func (b *Builder) String() string {
	return *(*string)(unsafe.Pointer(&b.buf)) // 采用强制转换方式,共用底层存储,不涉及内存分配.
}
```

**结论**
1. `+`拼接适用于短小的、常量字符串(明确的,非变量),因为编译器会优化.
2. `strings.Join`的性能一直很坚挺,当有现成的字符串数组或切片需要拼接时,优先使用该方法.但其灵活度有限.
3. `fmt.Sprint`系列主要是在格式化不同类型的数据方面有很大的灵活性.
4. `bytes.Buffer`不推荐使用.
5. 在使用`strings.Builder`时可以先预设容量,这样其灵活性和性能会有比较大的优势.

### 乱码和不可打印字符
如果字符串中出现不合法的utf8编码,打印时对于每个不合法的编码字节都会输出一个特定的符号�：
``` go
func main() {
  s := "高盈"
  fmt.Println(s[:5])
}
```
上面代码输出：`高�`,由于"盈"编码有三个字节,`s[:5]`只取前两个字节,无法组成一个合法的UTF8字符,故输出该特定符号.

还需要注意不可打印字符,两个字符串打印输出的内容相同,但两个字符串就是不相等:
``` go
func main() {
  b1 := []byte{0xEF, 0xBB, 0xBF, 72, 101, 108, 108, 111}
  b2 := []byte{72, 101, 108, 108, 111}

  s1 := string(b1)
  s2 := string(b2)

  fmt.Println(s1)
  fmt.Println(s2)
  fmt.Println(s1 == s2)
}
```
上面代码输出:
``` go
hello
hello
false
```
字符串比较是会比较长度和每个字节的,虽然打印出来是相同的,但b1是带了BOM头的(`0xEFBBBF`就是BOM头),b2没有带,这种一般会出现在从文件读取数据,一定要注意文件是否带了BOM头.



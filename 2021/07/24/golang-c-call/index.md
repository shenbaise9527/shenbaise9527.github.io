# Golang与C语言相互调用


主要是描述Go语言调用易盛行情API的动态库过程中，一些需要注意的地方。

## 类型对应关系
|C类型|调用方法|Go类型|字节数|
|:--|:--|:--|:--|:--|
|char|C.char|byte|1|
|signed char|C.schar|int8|1|
|unsigned char|C.uchar|uint8|1|
|short int|C.short|int16|2|
|short unsigned int|C.ushort|uint16|2|
|int|C.int|int|4|
|unsigned int|C.uint|uint32|4|
|long int|C.long|int32 or int64|4|
|long unsigned int|C.ulong|uint32 or uint64|4|
|long long int|C.longlong|int64|8|
|long long unsigned int|C.ulonglong|uint64|8|
|float|C.float|float32|4|
|double|C.double|float64|8|
|wchar_t|C.wchar_t||2|
|void *|unsafe.Pointer|||

## 导出函数
采用C++风格编译时,函数名会加上修饰符,会导致CGO查找不到对应的函数.所以必须采用C风格编译,这样函数名才会保持不变.

如下,就是采用C风格编译.
``` C++
#ifdef __cplusplus
extern "C"{
#endif

int Login(char* authCode, char* logPath, char* ip, int port, char* userName, char* passwd);
......

#ifdef __cplusplus
}
#endif
```

使用命令`nm libesunny.so`来查看动态库的符号表,可以看到函数`Login`的符号信息为`0000000000001fb0 T Login`,与名字是一样.

## 字符串
在C中用结尾带'\0'的字符数组来表示字符串的,而在Golang中string类型是原生类型,因此两种语言互操作时是需要进行字符串类型转换的.

通过`C.String`函数可以将Golang中的`string`类型转换为C的字符串类型,再传给C函数使用.
``` go
// Golang中的字符串.
s := "hello cgo\n"
// 转换为C的字符串.
cs := C.String(s)
// 传给C函数Print使用.
C.Print(cs)
```

需要注意,转换后的`cs`并不能由Golang的GC所管理,必须手动释放`cs`所占用的内存,即必须显示调用`C.free`.
``` go
// Golang中的字符串.
s := "hello cgo\n"
// 转换为C的字符串.
cs := C.String(s)
// 传给C函数Print使用.
C.Print(cs)

// 显示释放cs的内存.
C.free(cs)
```

## C中struct的定义
**第一种方式**

定义如下结构体.
``` C++
// 品种编码结构.
struct ESunnyCommodity {
    char* ExchangeNo;               ///< 交易所编码
    char  CommodityType;            ///< 品种类型
    char* CommodityNo;              ///< 品种编号
};
```

若在其它结构体或函数中引用了上述`ESunnyCommodity`结构体,则必须显示申明为`struct`.
``` C++
// 品种信息.
struct ESunnyCommodityInfo {
    struct ESunnyCommodity Commodity;            ///< 品种
    char*                  CommodityName;        ///< 品种名称,GBK编码格式
    char*                  CommodityEngName;     ///< 品种英文名称
    double                 ContractSize;         ///< 每手乘数
    double                 CommodityTickSize;    ///< 最小变动价位
    int                    CommodityDenominator; ///< 报价分母
    char                   CmbDirect;            ///< 组合方向
    int                    CommodityContractLen; ///< 品种合约年限
    char                   IsDST;                ///< 是否夏令时,'Y'为是,'N'为否
    struct ESunnyCommodity RelateCommodity1;     ///< 关联品种1
    struct ESunnyCommodity RelateCommodity2;     ///< 关联品种2
};

int QryCommodity(struct ESunnyCommodityInfo** info, int* len);
```

在上述引用的两个例子中,必须在`ESunnyCommodityInfo`前面显示说明为`struct`.否则在编译Golang代码时会报错`error: unknown type name 'ESunnyCommodityInfo'`

**第二种方式**

结构体采用如下方式定义.
``` C++
// 品种编码结构.
typedef struct {
    char* ExchangeNo;               ///< 交易所编码
    char  CommodityType;            ///< 品种类型
    char* CommodityNo;              ///< 品种编号
}ESunnyCommodity;
```

其它结构体或函数可以直接引用`ESunnyCommodity`.
``` C++
// 品种信息.
struct ESunnyCommodityInfo {
    ESunnyCommodity Commodity;            ///< 品种
    char*                  CommodityName;        ///< 品种名称,GBK编码格式
    char*                  CommodityEngName;     ///< 品种英文名称
    double                 ContractSize;         ///< 每手乘数
    double                 CommodityTickSize;    ///< 最小变动价位
    int                    CommodityDenominator; ///< 报价分母
    char                   CmbDirect;            ///< 组合方向
    int                    CommodityContractLen; ///< 品种合约年限
    char                   IsDST;                ///< 是否夏令时,'Y'为是,'N'为否
    ESunnyCommodity RelateCommodity1;     ///< 关联品种1
    ESunnyCommodity RelateCommodity2;     ///< 关联品种2
};

int QryCommodity(ESunnyCommodityInfo** info, int* len);
```

## Golang中引用C语言
**头文件和库的引用**
``` go
package wrapper

/*
#cgo CFLAGS: -I../esunny/include
#cgo LDFLAGS: -L../esunny/lib -lesunny -lTapQuoteAPI

#include <stdlib.h>
#include "export.h"
#include "tap.h"
*/
import "C"

import (
"encoding/json"
"errors"
"quote/source/tap/config"
"time"
"unsafe"

"github.com/tal-tech/go-zero/core/logx"
"github.com/tal-tech/go-zero/core/threading"
)
```
如上所述,在`package`下面通过注释的方式引入C.
* `#cgo CFLAGS: -I../esunny/include`,表示引用的C头文件的目录.
* `#cgo LDFLAGS: -L../esunny/lib -lesunny -lTapQuoteAPI`,表示引用的C动态库的路径及名字.
* `#include <stdlib.h>`,引入系统头文件,**如果引用了函数`C.free`,就需要该头文件**.
* `#include "export.h"`,表示引用的动态库对应的头文件.
* `import "C"`,必须紧跟在注释后面,中间不能有空行,否则会报错.

**调用C函数**
``` go
func (api *ESApi) SubscribeTick() error {
	for i := range api.contractDatas {
		// 把golang的string类型转换为C的char*
		cExchangeNo := C.CString(api.contractDatas[i].Contract.ExchangeNo)
		cCommoditNo := C.CString(api.contractDatas[i].Contract.CommodityNo)
		cContractNo := C.CString(api.contractDatas[i].Contract.ContractNo1)
		// C.char把golang的byte转换为C的char
		ret := C.SubscribeTick(cExchangeNo, cCommoditNo, C.char(api.contractDatas[i].Contract.CommodityType), cContractNo)

		// 调用C.free释放内存.
		C.free(unsafe.Pointer(cExchangeNo))
		C.free(unsafe.Pointer(cCommoditNo))
		C.free(unsafe.Pointer(cContractNo))
		if ret != 0 {
			logx.Errorf("subscrbe tick failed: %d, exchangeNo: %s, commoditNo: %s, contractNo: %s, commoditType: %s",
				ret, api.contractDatas[i].Contract.ExchangeNo, api.contractDatas[i].Contract.CommodityNo,
				api.contractDatas[i].Contract.ContractNo1, string(api.contractDatas[i].Contract.CommodityType))

			continue
		}

		logx.Infof("subscrbe tick successfully, exchangeNo: %s, commoditNo: %s, contractNo: %s, commoditType: %s",
			api.contractDatas[i].Contract.ExchangeNo, api.contractDatas[i].Contract.CommodityNo,
			api.contractDatas[i].Contract.ContractNo1, string(api.contractDatas[i].Contract.CommodityType))
	}

	return nil
}
```

**访问C的结构体数组**

C函数原型:
``` C++
/**
 * 查询品种信息,在外部要释放info对应的内存.
 * @param out info 品种信息数组.
 * @param out len 数组长度.
 *
 * @return 错误码,0-表示成功.
 */
int QryCommodity(struct ESunnyCommodityInfo** info, int* len);
```

Golang调用代码:
``` go
func (api *ESApi) QryCommodity() error {
	// 查询品种.
	api.commodityDatas = nil
	// 定义C中结构体ESunnyCommodityInfo的指针变量.
	var commodityInfo *C.struct_ESunnyCommodityInfo
	// 定义C中的int变量
	var commodityLen C.int
	// 调用C函数.
	ret := C.QryCommodity(&commodityInfo, &commodityLen)
	if ret != 0 {
		logx.Errorf("qry commodity err: %d", ret)

		return errors.New("query commodity failed")
	}

	// 把C中的int变量强制类型转换为golang中的int.
	cLen := int(commodityLen)
	// 强制转换为uintptr,后续需要做指针运算.
	commodityPointer := uintptr(unsafe.Pointer(commodityInfo))
	api.commodityDatas = make([]SourceCommodityInfo, 0, cLen)
	for i := 0; i < cLen; i++ {
		// 强制转换为C的结构体对象.
		info := *(*C.struct_ESunnyCommodityInfo)(unsafe.Pointer(commodityPointer))
		// 指针运算,指向下一个结构体对象.
		commodityPointer += unsafe.Sizeof(info)
		data := SourceCommodityInfo{
			Commodity: SourceCommodity{
				// 把C的char*转换为golang的string类型.
				ExchangeNo:    C.GoString(info.Commodity.ExchangeNo),  ///< 交易所编码
				// 把C的char转换为golang的byte类型.
				CommodityType: byte(info.Commodity.CommodityType),     ///< 品种类型
				CommodityNo:   C.GoString(info.Commodity.CommodityNo), ///< 品种编号
			}, ///< 品种
			CommodityName:        C.GoString(info.CommodityName),    ///< 品种名称,GBK编码格式
			CommodityEngName:     C.GoString(info.CommodityEngName), ///< 品种英文名称
			// 把C的double转换为golang的float64类型.
			ContractSize:         float64(info.ContractSize),        ///< 每手乘数
			CommodityTickSize:    float64(info.CommodityTickSize),   ///< 最小变动价位
			// 把C的int转换为golang的int类型.
			CommodityDenominator: int(info.CommodityDenominator),    ///< 报价分母
			CmbDirect:            byte(info.CmbDirect),              ///< 组合方向
			CommodityContractLen: int(info.CommodityContractLen),    ///< 品种合约年限
			IsDST:                byte(info.IsDST),                  ///< 是否夏令时,'Y'为是,'N'为否
			RelateCommodity1: SourceCommodity{
				ExchangeNo:    C.GoString(info.RelateCommodity1.ExchangeNo),  ///< 交易所编码
				CommodityType: byte(info.RelateCommodity1.CommodityType),     ///< 品种类型
				CommodityNo:   C.GoString(info.RelateCommodity1.CommodityNo), ///< 品种编号
			}, ///< 关联品种1
			RelateCommodity2: SourceCommodity{
				ExchangeNo:    C.GoString(info.RelateCommodity2.ExchangeNo),  ///< 交易所编码
				CommodityType: byte(info.RelateCommodity2.CommodityType),     ///< 品种类型
				CommodityNo:   C.GoString(info.RelateCommodity2.CommodityNo), ///< 品种编号
			}, ///< 关联品种2
		}

		api.commodityDatas = append(api.commodityDatas, data)
		body, _ := json.Marshal(data)
		logx.Info(string(body))
	}

	// 调用C.free释放内存.
	C.free(unsafe.Pointer(commodityInfo))

	return nil
}
```

**总结**
* 通过引入`C`包,来访问C代码."C"不是一个具体的包名,可理解成一个伪包,C语言所有语法元素均在该伪包下面.
* `import "C"`是必须的,且其与上面的C代码之间不能有空行,必须紧密相连.
* `C.CString`可以把golang的string类型转换为C的char*(后续需要显示调用`C.free`来释放内存),`C.GoString`可以把C的char*转换为golang的string类型.
* 其它基础类型可以直接强制类型转换.
* 访问C的结构体对象时,采用`C.struct_xxx`来定义结构体对象,之后可以直接访问到结构体成员.
* 访问C的结构体数组时,是通过指针运算来顺序访问每个对象的,需要使用`uintptr`和`unsafe.Pointer`.

## C语言中引用Golang
易盛行情API中涉及实时行情推送,需要有对应的回调函数.

C中定义回调函数原型:
``` C++
#ifdef __cplusplus
extern "C" {
#endif

/**
 * 外部函数,仅定义原型,接收tick数据回调.
 * @param in info tick数据.
 *
 */
extern void OnTick(struct ESunnyTickWhole* info);

/**
 * 外部函数,仅定义原型,api断开连接回调.
 * @param in errorCode 错误码.
 *
 */
extern void OnDisconnect(int errorCode);

#ifdef __cplusplus
}
#endif
```

Golang中函数实现:
``` go
//export OnTick
func OnTick(info *C.struct_ESunnyTickWhole) {
}

//export OnDisconnect
func OnDisconnect(errorCode C.int) {
}
```
如上,采用`//export`方式来申明函数是导出的.注意`//`和`export`之间不能有空格.



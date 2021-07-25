# 记录cgo调用C实现的加解密静态库中遇到的问题


## 起因
公司有个公共的加解密库,供所有后端C++服务调用的,但最近要使用Go来实现个服务需要用到加解密,而Go并没有提供AES-256-ECB的加解密库,所以决定用cgo来调用这个公共的加解密库.

## Window
在window下加解密库是提供的DLL,用Go的`syscall.NewLazyDLL`可以非常方便的加载DLL,window下基本没有遇到障碍.
``` go
package main

import (
	"fmt"
	"syscall"
	"unsafe"
)

// MICrypt 接口.
type MICrypt struct {
	MIFreeSafeHandle    *syscall.LazyProc
	MIGetDecryptDataLen *syscall.LazyProc
	MIGetEncryptDataLen *syscall.LazyProc
	MIGetSafeHandle     *syscall.LazyProc
	MILoad              *syscall.LazyProc
	MITransDecrypt      *syscall.LazyProc
	MITransEncrypt      *syscall.LazyProc
}

func main() {
	lib := syscall.NewLazyDLL("crypto64.dll")
	c := &MICrypt{
		MIGetSafeHandle:     lib.NewProc("MIGetSafeHandle"),
		MIFreeSafeHandle:    lib.NewProc("MIFreeSafeHandle"),
		MIGetDecryptDataLen: lib.NewProc("MIGetDecryptDataLen"),
		MIGetEncryptDataLen: lib.NewProc("MIGetEncryptDataLen"),
		MILoad:              lib.NewProc("MILoad"),
		MITransDecrypt:      lib.NewProc("MITransDecrypt"),
		MITransEncrypt:      lib.NewProc("MITransEncrypt"),
	}

	// 待加密字符串.
	msg := []byte("I am test trans crypto!")
	h, _, _ := c.MIGetSafeHandle.Call()
	c.MILoad.Call(uintptr(0), 0, h)
	var iLen int32
	var srcLen int = len(msg)
	c.MIGetEncryptDataLen.Call(uintptr(unsafe.Pointer(&iLen)), uintptr(unsafe.Pointer(&msg[0])), uintptr(srcLen), h)
	buf := make([]byte, iLen)

	// 加密.
	c.MITransEncrypt.Call(uintptr(unsafe.Pointer(&buf[0])), uintptr(iLen), uintptr(unsafe.Pointer(&msg[0])), uintptr(srcLen), h)
	fmt.Println(buf)

	dstLen := len(buf)
	var iDLen int32
	c.MIGetDecryptDataLen.Call(uintptr(unsafe.Pointer(&iDLen)), uintptr(unsafe.Pointer(&buf[0])), uintptr(dstLen), h)

	// 再解密.
	newSrc := make([]byte, iDLen)
	c.MITransDecrypt.Call(uintptr(unsafe.Pointer(&newSrc[0])), uintptr(iDLen), uintptr(unsafe.Pointer(&buf[0])), uintptr(dstLen), h)
	fmt.Println(string(newSrc))
	c.MIFreeSafeHandle.Call(h)
}
```

## Linux
在Linux下加解密库提供的是静态库,调用方式完全不同于windows,碰到很多问题.

### 不支持C++中的引用`&`
``` 
# aesecb
./aesecb.go:27:2: could not determine kind of name for C.MIGetDecryptDataLen
./aesecb.go:20:2: could not determine kind of name for C.MIGetEncryptDataLen
cgo: 
gcc errors for preamble:
In file included from ./aesecb.go:6:0:
/home/sky/code/imp/2nd/crypto/include/ISafeInterface.h:111:50: error: expected ';', ',' or ')' before '&' token
 _DLL_EXP_API int32_t MIGetEncryptDataLen(int32_t &iRevLen, const char *pData, int32_t iLen, intptr_t pSafeHandle);
```

**解决方案**
把引用修改为指针
``` C++
_DLL_EXP_API int32_t MIGetEncryptDataLen(int32_t* iRevLen, const char *pData, int32_t iLen, intptr_t pSafeHandle);
```

### undefined reference
``` 
$ go build -x
WORK=/tmp/go-build163613860
mkdir -p $WORK/b001/
cd /home/sky/go/path/src/aesecb
CGO_LDFLAGS='"-g" "-O2" "-L/home/sky/code/imp/2nd/cryptogo/lib/Linux_x86_64" "-lmism" "-lstdc++"' /home/sky/go/go1.14/pkg/tool/linux_amd64/cgo -objdir $WORK/b001/ -importpath aesecb -- -I/h
ome/sky/code/imp/2nd/cryptogo/include -I $WORK/b001/ -g -O2 ./aesecb.gocd $WORK
gcc -fno-caret-diagnostics -c -x c - -o /dev/null || true
gcc -Qunused-arguments -c -x c - -o /dev/null || true
gcc -fdebug-prefix-map=a=b -c -x c - -o /dev/null || true
gcc -gno-record-gcc-switches -c -x c - -o /dev/null || true
cd $WORK/b001
TERM='dumb' gcc -I /home/sky/go/path/src/aesecb -fPIC -m64 -pthread -fmessage-length=0 -fdebug-prefix-map=$WORK/b001=/tmp/go-build -gno-record-gcc-switches -I/home/sky/code/imp/2nd/cryptogo
/include -I ./ -g -O2 -o ./_x001.o -c _cgo_export.cTERM='dumb' gcc -I /home/sky/go/path/src/aesecb -fPIC -m64 -pthread -fmessage-length=0 -fdebug-prefix-map=$WORK/b001=/tmp/go-build -gno-record-gcc-switches -I/home/sky/code/imp/2nd/cryptogo
/include -I ./ -g -O2 -o ./_x002.o -c aesecb.cgo2.cTERM='dumb' gcc -I /home/sky/go/path/src/aesecb -fPIC -m64 -pthread -fmessage-length=0 -fdebug-prefix-map=$WORK/b001=/tmp/go-build -gno-record-gcc-switches -I/home/sky/code/imp/2nd/cryptogo
/include -I ./ -g -O2 -o ./_cgo_main.o -c _cgo_main.ccd /home/sky/go/path/src/aesecb
TERM='dumb' gcc -I . -fPIC -m64 -pthread -fmessage-length=0 -fdebug-prefix-map=$WORK/b001=/tmp/go-build -gno-record-gcc-switches -o $WORK/b001/_cgo_.o $WORK/b001/_cgo_main.o $WORK/b001/_x00
1.o $WORK/b001/_x002.o -g -O2 -L/home/sky/code/imp/2nd/cryptogo/lib/Linux_x86_64 -lmism -lstdc++# aesecb
/tmp/go-build163613860/b001/_x002.o: In function `_cgo_2c40edecf6fd_Cfunc_MIGetDecryptDataLen':
/tmp/go-build/cgo-gcc-prolog:69: undefined reference to `MIGetDecryptDataLen'
/tmp/go-build163613860/b001/_x002.o: In function `_cgo_2c40edecf6fd_Cfunc_MIGetEncryptDataLen':
/tmp/go-build/cgo-gcc-prolog:92: undefined reference to `MIGetEncryptDataLen'
/tmp/go-build163613860/b001/_x002.o: In function `_cgo_2c40edecf6fd_Cfunc_MIGetSafeHandle':
/tmp/go-build/cgo-gcc-prolog:109: undefined reference to `MIGetSafeHandle'
/tmp/go-build163613860/b001/_x002.o: In function `_cgo_2c40edecf6fd_Cfunc_MILoad':
/tmp/go-build/cgo-gcc-prolog:131: undefined reference to `MILoad'
/tmp/go-build163613860/b001/_x002.o: In function `_cgo_2c40edecf6fd_Cfunc_MITransDecrypt':
/tmp/go-build/cgo-gcc-prolog:156: undefined reference to `MITransDecrypt'
/tmp/go-build163613860/b001/_x002.o: In function `_cgo_2c40edecf6fd_Cfunc_MITransEncrypt':
/tmp/go-build/cgo-gcc-prolog:181: undefined reference to `MITransEncrypt'
/tmp/go-build163613860/b001/_x002.o: In function `_cgo_2c40edecf6fd_Cfunc_MIFreeSafeHandle':
/tmp/go-build/cgo-gcc-prolog:49: undefined reference to `MIFreeSafeHandle'
collect2: error: ld returned 1 exit status
```
由于在编译加解密库mism时,是使用的g++编译器,编译出来的函数名会加上修饰符,导致cgo找不到对应的函数
```
# 查看符号表
$ nm libmism.a
...
0000000000000490 T _Z12MIMD5DecryptPcRiPKci
000000000000040f T _Z12MIMD5EncryptPcRiPKci
000000000000031f T _Z14MITransDecryptPciPKcil
00000000000002c9 T _Z14MITransEncryptPciPKcil
...
```
可以看到函数名已经发生变化了.

**解决方案**
修改代码以C方式编译
``` C++
#ifdef __cplusplus
extern "C"
{
#endif
......
    _DLL_EXP_API intptr_t MIGetSafeHandle();
......
#ifdef __cplusplus
}
#endif
```
再次编译后,查看符号表,可以看到函数名没有发生变化.
```
$ nm libmism.a
......
0000000000000040 T MIFreeSafeHandle
00000000000001d0 T MIGetDecryptDataLen
0000000000000190 T MIGetEncryptDataLen
0000000000000000 T MIGetSafeHandle
0000000000000070 T MILoad
0000000000000150 T MITransDecrypt
0000000000000110 T MITransEncrypt
......
```

### 库依赖
继续编译,又报了undefined reference
```
$ go build aesecb.go 
# command-line-arguments
./libmism.a(AESEncryptHandle.cpp.o): In function `AESEncryptHandle::Encrypt(char*, int&, char const*, int const&)':
AESEncryptHandle.cpp:(.text+0x135): undefined reference to `EVP_aes_256_ecb'
AESEncryptHandle.cpp:(.text+0x145): undefined reference to `EVP_EncryptInit'
AESEncryptHandle.cpp:(.text+0x15c): undefined reference to `EVP_EncryptUpdate'
AESEncryptHandle.cpp:(.text+0x171): undefined reference to `EVP_EncryptFinal'
AESEncryptHandle.cpp:(.text+0x179): undefined reference to `EVP_CIPHER_CTX_cleanup'
./libmism.a(AESEncryptHandle.cpp.o): In function `AESEncryptHandle::Decrypt(char*, int&, char const*, int const&)':
AESEncryptHandle.cpp:(.text+0x215): undefined reference to `EVP_aes_256_ecb'
AESEncryptHandle.cpp:(.text+0x225): undefined reference to `EVP_DecryptInit'
AESEncryptHandle.cpp:(.text+0x23c): undefined reference to `EVP_DecryptUpdate'
AESEncryptHandle.cpp:(.text+0x251): undefined reference to `EVP_DecryptFinal'
AESEncryptHandle.cpp:(.text+0x259): undefined reference to `EVP_CIPHER_CTX_cleanup'
collect2: error: ld returned 1 exit status
```
这是加解密库是调用的openssl来实现的,而在go代码里只显示链接了加解密库

**解决方案**
修改go代码,还要额外链接openssl的库.
``` go
#cgo LDFLAGS: -L../2nd/crypto/lib/Linux_x86_64 -L../3rd/openssl-OpenSSL_1_0_2-stable/lib/Linux_x86_64 -lmism -lssl -lcrypto -lstdc++
```

继续编译,又报了undefined reference
```
$ go build aesecb.go 
# command-line-arguments
./libcrypto.a(dso_dlfcn.o): In function `dlfcn_globallookup':
dso_dlfcn.c:(.text+0x11): undefined reference to `dlopen'
dso_dlfcn.c:(.text+0x24): undefined reference to `dlsym'
dso_dlfcn.c:(.text+0x2f): undefined reference to `dlclose'
./libcrypto.a(dso_dlfcn.o): In function `dlfcn_bind_func':
dso_dlfcn.c:(.text+0x354): undefined reference to `dlsym'
dso_dlfcn.c:(.text+0x412): undefined reference to `dlerror'
./libcrypto.a(dso_dlfcn.o): In function `dlfcn_bind_var':
dso_dlfcn.c:(.text+0x484): undefined reference to `dlsym'
dso_dlfcn.c:(.text+0x542): undefined reference to `dlerror'
./libcrypto.a(dso_dlfcn.o): In function `dlfcn_load':
dso_dlfcn.c:(.text+0x5a9): undefined reference to `dlopen'
dso_dlfcn.c:(.text+0x60d): undefined reference to `dlclose'
dso_dlfcn.c:(.text+0x645): undefined reference to `dlerror'
./libcrypto.a(dso_dlfcn.o): In function `dlfcn_pathbyaddr':
dso_dlfcn.c:(.text+0x6d1): undefined reference to `dladdr'
dso_dlfcn.c:(.text+0x731): undefined reference to `dlerror'
./libcrypto.a(dso_dlfcn.o): In function `dlfcn_unload':
dso_dlfcn.c:(.text+0x792): undefined reference to `dlclose'
collect2: error: ld returned 1 exit status
```
缺少ld引用.

**解决方案**
修改go代码,还要额外链接ld的库.
``` go
#cgo LDFLAGS: -L../2nd/crypto/lib/Linux_x86_64 -L../3rd/openssl-OpenSSL_1_0_2-stable/lib/Linux_x86_64 -lmism -lssl -lcrypto -lstdc++ -ldl
```

### 类型映射错误
```
$ go build aesecb.go 
# command-line-arguments
./aesecb.go:16:10: assignment mismatch: 3 variables but _Cfunc_MIGetSafeHandle returns 1 values
./aesecb.go:17:18: cannot use uintptr(0) (type uintptr) as type *_Ctype_char in argument to _Cfunc_MILoad
./aesecb.go:20:31: cannot use uintptr(unsafe.Pointer(&iLen)) (type uintptr) as type *_Ctype_int in argument to _Cfunc_MIGetEncryptDataLen
./aesecb.go:20:63: cannot use uintptr(unsafe.Pointer(&msg[0])) (type uintptr) as type *_Ctype_char in argument to _Cfunc_MIGetEncryptDataLen
./aesecb.go:20:97: cannot use uintptr(srcLen) (type uintptr) as type _Ctype_int in argument to _Cfunc_MIGetEncryptDataLen
./aesecb.go:22:26: cannot use uintptr(unsafe.Pointer(&buf[0])) (type uintptr) as type *_Ctype_char in argument to _Cfunc_MITransEncrypt
./aesecb.go:22:60: cannot use uintptr(iLen) (type uintptr) as type _Ctype_int in argument to _Cfunc_MITransEncrypt
./aesecb.go:22:75: cannot use uintptr(unsafe.Pointer(&msg[0])) (type uintptr) as type *_Ctype_char in argument to _Cfunc_MITransEncrypt
./aesecb.go:22:109: cannot use uintptr(srcLen) (type uintptr) as type _Ctype_int in argument to _Cfunc_MITransEncrypt
./aesecb.go:27:31: cannot use uintptr(unsafe.Pointer(&iDLen)) (type uintptr) as type *_Ctype_int in argument to _Cfunc_MIGetDecryptDataLen
./aesecb.go:27:31: too many errors
```
基本上都是用错了类型,参考类型映射来修改.
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

最后完整代码如下
``` go
package main                                     

/*                                               
#cgo CFLAGS: -I../2nd/crypto/include
#cgo LDFLAGS: -L../2nd/crypto/lib/Linux_x86_64 -L../3rd/openssl-OpenSSL_1_0_2-stable/lib/Linux_x86_64 -lmism -lssl -lcrypto -lstdc++ -ldl                    
#include "ISafeInterface.h"                      
*/                                               
import "C"                                       
import (                                         
    "fmt"                                        
    "unsafe"                                     
)                                                

func main() {                                    
    msg := []byte("I am test trans crypto!")  
    h := C.MIGetSafeHandle()                     
    C.MILoad(nil, C.int(0), h)                   
    var iLen int32                               
    var srcLen int = len(msg)                    
    C.MIGetEncryptDataLen((*C.int)(&iLen), (*C.char)(unsafe.Pointer(&msg[0])), C.int(srcLen), h)
    buf := make([]byte, 44)                      
    C.MITransEncrypt((*C.char)(unsafe.Pointer(&buf[0])), C.int(iLen), (*C.char)(unsafe.Pointer(&msg[0])), C.int(srcLen), h)
    fmt.Println(buf)                             
                                                 
    dstLen := len(buf)                           
    var iDLen int32                              
    C.MIGetDecryptDataLen((*C.int)(&iDLen), (*C.char)(unsafe.Pointer(&buf[0])), C.int(dstLen), h)
                                                 
    fmt.Println(iDLen)                           
    newSrc := make([]byte, iDLen)                
    C.MITransDecrypt((*C.char)(unsafe.Pointer(&newSrc[0])), C.int(iDLen), (*C.char)(unsafe.Pointer(&buf[0])), C.int(dstLen), h)
    fmt.Println(string(newSrc))                  
    C.MIFreeSafeHandle(h)
}
```

### go tool cgo
调用该命令会在当前目录生成_obj文件夹,在里面文件可以看到类型转换的信息.[参考命令](https://wiki.jikexueyuan.com/project/go-command-tutorial/0.13.html)
```
# sky @ localhost in ~/go/path/src/aesecb/_obj [11:33:09] 
$ ll 
total 48
drwxr-xr-x 2 sky sky 4096 Sep 10 11:32 .
drwxr-xr-x 3 sky sky   92 Sep 10 11:32 ..
-rw-r--r-- 1 sky sky 6264 Sep 10 11:32 _cgo_.o
-rw-r--r-- 1 sky sky  605 Sep 10 11:32 _cgo_export.c
-rw-r--r-- 1 sky sky 1547 Sep 10 11:32 _cgo_export.h
-rw-r--r-- 1 sky sky   13 Sep 10 11:32 _cgo_flags
-rw-r--r-- 1 sky sky 5427 Sep 10 11:32 _cgo_gotypes.go
-rw-r--r-- 1 sky sky  416 Sep 10 11:32 _cgo_main.c
-rw-r--r-- 1 sky sky 2020 Sep 10 11:32 aesecb.cgo1.go
-rw-r--r-- 1 sky sky 5710 Sep 10 11:32 aesecb.cgo2.c
```


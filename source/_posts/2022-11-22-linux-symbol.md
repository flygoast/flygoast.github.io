---
title: Linux动态链接库符号冲突解决
date: 2022-11-22 00:55:35
tags:
- Linker
- Loader
categories: MISC
---
最近遇到一个`so`库符号冲突的问题, 可以总结为: 

* 动态库`so1`中静态编译了某基础库
* 动态库`so2`中动态链接了该基础库的另一版本
* 可执行程序动态链接了这两个`so`
* 程序执行到`so2`中函数时, 调用了`so1`中的基础库的符号, 而恰好该基础库两个版本的同名函数不兼容, 因而出现了崩溃.

下面通过`demo`代码来说明如何解决这个问题.

<!--more-->

##### 基础库`libhello v1`

目录结构如下:
```bash
[root@default symbols]# tree libhello1
libhello1
├── hello.c
├── hello.h
└── Makefile

0 directories, 3 files
```

`hello.c`:
```c
#include <stdio.h>

void print_hello() {
    printf("hello v1.0\n");
}
```

`hello.h`:
```c
extern void print_hello();
```

`Makefile`:
```makefile
CFLAGS=-fPIC

static:
	gcc -c $(CFLAGS) *.c
	ar -rcu libhello.a *.o

clean:
	rm -rf *.o
	rm -rf *.a
```

在`libhello1`目录下执行`make`将基础库`libhello v1`编译为静态库: `libhello.a`.

##### 基础库`libhello v2`

目录结构如下:
```bash
[root@default symbols]# tree libhello2/
libhello2/
├── hello.c
├── hello.h
└── Makefile

0 directories, 3 files
```

`hello.c`:
```c
#include <stdio.h>

void print_hello() {
    printf("hello v2.0\n");
}
```

`hello.h`:
```c
extern void print_hello();
```

`Makefile`:
```makefile
CFLAGS=-fPIC

so1:
	gcc -c $(CFLAGS) *.c
	gcc -o libhello.so -shared $(CFLAGS) *.o

clean:
	rm -rf *.o
	rm -rf *.so
```

在`libhello2`目录下执行`make`将基础库`libhello v2`编译为动态库: `libhello.so`.

##### 动态库`so1`

目录结构:
```bash
[root@default symbols]# tree so1/
so1/
├── Makefile
├── so1.c
└── so1.h

0 directories, 3 files
```

`so1.c`:
```c
#include <stdio.h>
#include "hello.h"

void hello_so1() {
    printf("hello in so1\n");

    print_hello();
}
```

`so1.h`:
```c
extern void hello_so1();
```

`Makefile`:
```makefile
CFLAGS=-I../libhello1 -fPIC
LDFLAGS=

so1:
	gcc -c $(CFLAGS) *.c
	gcc -o libso1.so -shared $(CFLAGS) $(LDFLAGS) *.o ../libhello1/libhello.a

clean:
	rm -rf *.o
	rm -rf *.so
```

动态库`so1`使用静态库`libhello.a`, 在`so1`目录下执行`make`生成`libso1.so`.

##### 动态库`so2`

目录结构:
```bash
[root@default symbols]# tree so2/
so2/
├── Makefile
├── so2.c
└── so2.h

0 directories, 3 files
```

`so2.c`:
```c
#include <stdio.h>
#include "hello.h"

void hello_so2() {
    printf("hello in so2\n");

    print_hello();
}
```

`so2.h`:
```c
extern void hello_so2();
```

`Makefile`:
```makefile
CFLAGS=-I../libhello2/ -L../libhello2/ -fPIC
LDFLAGS=-Wl,-rpath=../libhello2/

so2:
	gcc -c $(CFLAGS) *.c
	gcc -o libso2.so -shared $(CFLAGS) $(LDFLAGS) *.o -lhello

clean:
	rm -rf *.o
	rm -rf *.so
```

动态库`so2`动态链接基础库`libhello.so`, 在`so2`目录下执行`make`生成`libso2.so`.

##### 可执行程序

目录结构:
```bash
[root@default symbols]# tree main/
main/
├── main.c
└── Makefile

0 directories, 2 files
```

`main.c`:
```c
#include <stdio.h>

#include "so1.h"
#include "so2.h"

int main(int argc, char **argv) {
    hello_so1();

    hello_so2();

    return 0;
}
```

`Makefile`:
```makefile
CFLAGS=-I../so1/ -I../so2/
LDFLAGS=-L../so1/ -L../so2/ -Wl,-rpath=../so1/,-rpath=../so2/

so2:
	gcc -c $(CFLAGS) *.c
	gcc -o main $(CFLAGS) $(LDFLAGS)  -lso1 -lso2 *.o

clean:
	rm -rf *.o
	rm -rf main
```

可执行程序`main`动态链接`so1`和`so2`, 在`main`目录下执行`make`生成可执行程序`main`.

##### 整体测试程序结构

```bash
[root@default symbols]# tree .
.
├── libhello1
│   ├── hello.c
│   ├── hello.h
│   └── Makefile
├── libhello2
│   ├── hello.c
│   ├── hello.h
│   └── Makefile
├── main
│   ├── main.c
│   └── Makefile
├── so1
│   ├── Makefile
│   ├── so1.c
│   └── so1.h
└── so2
    ├── Makefile
    ├── so2.c
    └── so2.h

5 directories, 14 files
```

##### 分析与解决

执行`main`的结果:
```bash
[root@default main]# ./main
hello in so1
hello v1.0
hello in so2
hello v1.0
```

从结果可以看到`hello_so2`调用了`libso1.so`中的`print_hello`函数.

我们查看`libso1.so`的符号表:
```bash
[root@default main]# nm ../so1/libso1.so  |grep print_hello
0000000000000711 T print_hello
```

`T/t`表示代码区的符号, `T`表示是全局可见符号, `t`表示库内部本地可见符号.

`readelf`的输出更容易区分:
```bash
[root@default main]# readelf -s ../so1/libso1.so  |grep print_hello
     9: 0000000000000711    18 FUNC    GLOBAL DEFAULT   11 print_hello
    51: 0000000000000711    18 FUNC    GLOBAL DEFAULT   11 print_hello
```

`libso1.so`中的`print_hello`为全局可见符号.

`libso2.so`中只包含对`print_hello`的引用:
```bash
[root@default main]# readelf -s ../so2/libso2.so  |grep print_hello
     4: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND print_hello
    51: 0000000000000000     0 FUNC    GLOBAL DEFAULT  UND print_hello
```

另一个`print_hello`符号定义位于`libhello.so`中:
```bash
[root@default main]# readelf -s ../libhello2/libhello.so |grep print_hello
     9: 00000000000006a5    18 FUNC    GLOBAL DEFAULT   11 print_hello
    50: 00000000000006a5    18 FUNC    GLOBAL DEFAULT   11 print_hello
```

这两个中符号定义中, 哪个生效是如何决定的呢?

`Linux`动态链接器(如, `/lib64/ld-linux-x86-64.so.2`)在加载动态链接库中的符号到全局符号表时, 如果相同的符号已经存在, 则后加入的符号将被忽略, 这叫做`全局符号介入: Global symbol interpose`. 而`Linux`动态链接器加载所依赖的动态库是按照广度优先的顺序进行的. 以我们的例子来说就是, `main`依赖`libso1.so`和`libso2.so`, 因而先加载`libso1.so`和`libso2.so`, 接下来再处理`libso1.so`和`libso2.so`的依赖才会加载到`libhello.so`. 而`libso1.so`和`libso2.so`的加载顺序是由链接时的顺序决定的. 

可以使用`ldd`命令查看`main`的依赖和加载顺序:
```bash
[root@default main]# ldd main
	linux-vdso.so.1 =>  (0x00007ffc11d11000)
	libso1.so => ../so1/libso1.so (0x00007fb5c94fe000)
	libso2.so => ../so2/libso2.so (0x00007fb5c92fc000)
	libc.so.6 => /lib64/libc.so.6 (0x00007fb5c8f2e000)
	libhello.so => ../libhello2/libhello.so (0x00007fb5c8d2c000)
	/lib64/ld-linux-x86-64.so.2 (0x00007fb5c9700000)
```

如果修改`main`程序的`Makefile`, 将`so1`和`so2`的顺序颠倒, 编译的程序依赖为:
```bash
[root@default main]# ldd main
	linux-vdso.so.1 =>  (0x00007fffab9ea000)
	libso2.so => ../so2/libso2.so (0x00007f91ac77b000)
	libso1.so => ../so1/libso1.so (0x00007f91ac579000)
	libc.so.6 => /lib64/libc.so.6 (0x00007f91ac1ab000)
	libhello.so => ../libhello2/libhello.so (0x00007f91abfa9000)
	/lib64/ld-linux-x86-64.so.2 (0x00007f91ac97d000)
```

可以看到这次先加载`libso2.so`了.

我们的实验程序先加载`libso1.so`, 因而`libso1.so`中的`print_hello`被加载到全局符号表中, 后续`libhello.so`中的`print_hello`符号忽略, 因而`main`程序中`hello_so2`调用的`print_hello`实际为`libso1.so`中的`print_hello`.

我们可以使用`LD_PRELOAD`环境变量来指定优先加载`libhello.so`, 运行结果:
```bash
[root@default main]# LD_PRELOAD=../libhello2/libhello.so ./main
hello in so1
hello v2.0
hello in so2
hello v2.0
```

可以看到这次`hello_so1`和`hello_so2`的`print_hello`都来自`libhello.so`. 但这并不是我们所希望看到的结果, 我们希望`libso1.so`使用它自己的`print_hello`. 这可以通过链接选项`-Bsymbolic`来实现. 根据[`ld`的文档](https://linux.die.net/man/1/ld), `-Bsymbolic`选项可以让动态库优先使用自己的符号: 
```plain
-Bsymbolic
When creating a shared library, bind references to global symbols to the definition within the shared library, if any. Normally, it is possible for a program linked against a shared library to override the definition within the shared library. This option is only meaningful on ELF platforms which support shared libraries.
```

我们修改`so1`的`Makefile`, 加上`-Bsymbolic`选项:
```makefile
CFLAGS=-I../libhello1 -fPIC
LDFLAGS=-Wl,-Bsymbolic

so1:
    gcc -c $(CFLAGS) *.c
    gcc -o libso1.so -shared $(CFLAGS) $(LDFLAGS) *.o ../libhello1/libhello.a

clean:
    rm -rf *.o
    rm -rf *.so
```

在`so1`目录重新执行`make`后, 再次运行`main`程序:
```bash
[root@default main]# LD_PRELOAD=../libhello2/libhello.so ./main
hello in so1
hello v1.0
hello in so2
hello v2.0
```

可以看到`hello_so1`和`hello_so2`各自调用了正确的`print_hello`.

这样能够得到我们所期望的结果, 但手动去指定加载动态库的方法既费时费力,又不具备通用性. 还需要寻找更优雅的解决方案. 我们的例子中, `so1`使用静态库`libhello.a`, 只是自用, 并不需要将这些符号提供给其他动态库使用. 我们应该控制这些符号的可见性. Linux动态库中的符号默认可见性为全局, 可以使用编译选项`-fvisibility=hidden`将符号默认可见性修改对外不可见, 需要由外部使用的符号需要显示声明, 如:
```
void __attribute ((visibility("default"))) hello_so1()
```

但`so1`中的`proto_hello`符号来源于`libhello.a`, `-fvisibility=hidden`对来自静态库的符号并不生效. 可以使用链接选项`-Wl,--exclude-libs,ALL`来将所有静态库的符号屏蔽.

我们修改`so1`的`Makefile`:
```makefile
CFLAGS=-I../libhello1 -fPIC
LDFLAGS=-Wl,-Bsymbolic -Wl,--exclude-libs,ALL

so1:
    gcc -c $(CFLAGS) *.c
    gcc -o libso1.so -shared $(CFLAGS) $(LDFLAGS) *.o ../libhello1/libhello.a

clean:
    rm -rf *.o
    rm -rf *.so
```

重新编译`so1`后再次执行`main`:
```bash
[root@default main]# ./main
hello in so1
hello v1.0
hello in so2
hello v2.0
```

`hello_so1`和`hello_so2`都调用了正确的符号, 符合我们的希望.

查看`so1`的符号, 可以看到`print_hello`的可见性为`LOCAL`, 不再是全局可见:
```bash
[root@default main]# readelf -s ../so1/libso1.so |grep print_hello
    41: 00000000000006c1    18 FUNC    LOCAL  DEFAULT   11 print_hello
```

##### 参考

* https://stackoverflow.com/questions/2222162/how-to-apply-fvisibility-option-to-symbols-in-static-libraries
* https://stackoverflow.com/questions/6562403/i-dont-understand-wl-rpath-wl
* https://stackoverflow.com/questions/37531846/nm-symbol-output-t-vs-t-in-a-shared-so-library
* https://linux.die.net/man/1/ld
* https://www.gnu.org/software/gnulib/manual/html_node/LD-Version-Scripts.html
* https://sourceware.org/binutils/docs-2.20/ld/VERSION.html
* https://www.technovelty.org/c/symbol-versions-and-dependencies.html
* https://www.baeldung.com/linux/shared-object-filenames
* https://tldp.org/HOWTO/Program-Library-HOWTO/shared-libraries.html
* https://itnewbee.org/linux%E4%B8%8B%E5%8A%A8%E6%80%81%E5%BA%93%E5%8F%8A%E7%89%88%E6%9C%AC%E5%8F%B7%E6%8E%A7%E5%88%B6/
* https://rockhong.github.io/shared-library-intro.html
* https://www.akkadia.org/drepper/dsohowto.pdf
* https://www.akkadia.org/drepper/symbol-versioning
* https://cnblogs.com/welhzh/p/6730839.html
* https://www.cnblogs.com/lsgxeva/p/8257784.html
* https://blog.blahgeek.com/glibc-and-symbol-versioning/
* https://www.caichinger.com/elf.html
* https://refspecs.linuxfoundation.org/LSB_3.0.0/LSB-PDA/LSB-PDA.junk/symversion.html
* https://zohead.com/archives/mod-elf-glibc/

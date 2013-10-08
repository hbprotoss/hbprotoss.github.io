<!-- 
.. link: 
.. description: 
.. tags: C, Linux, Programming
.. date: 2013/10/08 09:18:31
.. title: Linux下编译链接动态库
.. slug: linuxxia-bian-yi-lian-jie-dong-tai-ku
-->

记录下Linux下编译和链接动态库的过程。

##一、 编写动态库
头文件so.h：  
```c
#ifndef  SO_H
#define  SO_H

int add(int a, int b);


#endif  /*SO_H*/

```

实现文件so.c：
```c
#include "so.h"

int add(int a, int b)
{
	return a + b;
}

```

##二、编译动态库
```bash
gcc so.c -fPIC -shared -o libtest.so
```
解释一下各个参数含义：

1. -fPIC：生成位置无关代码([Position Independent Code](http://en.wikipedia.org/wiki/Position-independent_code ""))，只有生成PIC才能通过虚拟页映射达到可执行代码在进程间共享，从而节省内存的目的。
2. 说明我们要生成的是动态库so(Shared Object)文件，从而进行动态链接。

通过`nm -g libtest.so`可以看到，导出符号表中已经有`add`这个符号了：
```bash
$ nm -g libtest.so 
0000000000000670 T add
0000000000201030 B __bss_start
                 w __cxa_finalize@@GLIBC_2.2.5
0000000000201030 D _edata
0000000000201038 B _end
0000000000000684 T _fini
                 w __gmon_start__
0000000000000540 T _init
                 w _ITM_deregisterTMCloneTable
                 w _ITM_registerTMCloneTable
                 w _Jv_RegisterClasses
```

##三、使用动态库
###1. 隐式链接(编译时链接)
编写主程序main.c：
```c
#include <stdio.h>
#include "so.h"

int main(int argc, char *argv[])
{
	printf("%d\n", add(1, 2));
	return 0;
}

```

使用`gcc main.c -L. -ltest -o test`进行编译。

* -L：添加库文件的搜索路径
* -l：指定需要链接的库。该名称是处在头lib和后缀.so中的名称，如上动态库libtest.so的l参数为-l test

此时通过`readelf test -d`已经能看到生成的可执行文件test的Dynamic section里依赖libtest.so了
```bash
$ readelf test -d

Dynamic section at offset 0xe18 contains 25 entries:
  Tag        Type                         Name/Value
 0x0000000000000001 (NEEDED)             Shared library: [libtest.so]
 0x0000000000000001 (NEEDED)             Shared library: [libc.so.6]
......
```

dynamic symbols中也有一个undefined symbol(add)
```bash
$ nm -D test 
                 U add
0000000000601048 B __bss_start
0000000000601048 D _edata
0000000000601050 B _end
00000000004007b4 T _fini
                 w __gmon_start__
0000000000400578 T _init
                 w _ITM_deregisterTMCloneTable
                 w _ITM_registerTMCloneTable
                 w _Jv_RegisterClasses
                 U __libc_start_main
                 U printf
```

在执行隐式链接的程序之前要注意设置`LD_LIBRARY_PATH`环境变量，或者把前面生成的libtest.so复制到系统路径下，否则会找不到动态库。
```
$ ./test 
./test: error while loading shared libraries: libtest.so: cannot open shared object file: No such file or directory

$ export LD_LIBRARY_PATH=.

$ ./test 
3
```

###2. 显式链接(运行时链接)
编写主程序dyn_main.c
```c
#include <stdio.h>
#include <dlfcn.h>

int main(int argc, char *argv[])
{
	void *dl = NULL;
	int (*add)(int a, int b);
	dl = dlopen( "./libtest.so", RTLD_LAZY);
	if( dl == NULL )
	{
		printf("so loading error.\n");
		return 1;
	}
	add = (int(*)(int, int))dlsym(dl, "add");
	if( dlerror() != NULL )
	{
		printf("fun load error.\n");
		return 1;
	}
	printf("%d\n", add(1, 2));
	return 0;
}
```

使用`gcc dyn_main.c -ldl -o dyn_test`编译。

这时通过`readelf dyn_test -d`可以发现，dyn_test不依赖libtest.so：
```
$ readelf dyn_test -d

Dynamic section at offset 0xe18 contains 25 entries:
  Tag        Type                         Name/Value
 0x0000000000000001 (NEEDED)             Shared library: [libdl.so.2]
 0x0000000000000001 (NEEDED)             Shared library: [libc.so.6]
......
```

dyn_test的dynamic symbols中也没有add：

```
$ nm -D dyn_test 
                 U dlerror
                 U dlopen
                 U dlsym
                 w __gmon_start__
                 w _ITM_deregisterTMCloneTable
                 w _ITM_registerTMCloneTable
                 w _Jv_RegisterClasses
                 U __libc_start_main
                 U printf
                 U puts
```

运行程序也不需要设置`LD_LIBRARY_PATH`环境变量

```bash
$ ./dyn_test 
3
```

-----
参考资料：

1. [Linux动态库的编译与使用 转载](http://hi.baidu.com/linuxlife/item/ed8e5d0c6e86b491a2df4366 "")
2. [详细分析Linux动态库的使用方式](http://os.51cto.com/art/201003/186246.htm "")
3. [How do I list the symbols in a .so file](http://stackoverflow.com/questions/34732/how-do-i-list-the-symbols-in-a-so-file "")
4. [Import names in ELF binary](http://stackoverflow.com/questions/10480657/import-names-in-elf-binary "")
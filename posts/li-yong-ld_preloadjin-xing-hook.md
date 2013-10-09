<!-- 
.. link: 
.. description: 
.. tags: C, Linux, Hook, Security
.. date: 2013/10/09 11:44:50
.. title: 利用LD_PRELOAD进行hook
.. slug: li-yong-ld_preloadjin-xing-hook
-->

好久没玩hook这种猥琐的东西里，今天在Linux下体验了一把。

loader在进行动态链接的时候，会将有相同符号名的符号覆盖成LD_PRELOAD指定的so文件中的符号。换句话说，可以用我们自己的so库中的函数替换原来库里有的函数，从而达到hook的目的。这和Windows下通过修改import table来hook API很类似。相比较之下，LD_PRELOAD更方便了，都不用自己写代码了，系统的loader会帮我们搞定。但是LD_PRELOAD有个限制：只能hook动态链接的库，对静态链接的库无效，因为静态链接的代码都写到可执行文件里了嘛，没有坑让你填。

上代码

先是受害者，我们的主程序main.c，通过strcmp比较字符串是否相等：

```c
#include <stdio.h>
#include <string.h>

int main(int argc, char *argv[])
{
	if( strcmp(argv[1], "test") )
	{
		printf("Incorrect password\n");
	}
	else
	{
		printf("Correct password\n");
	}
	return 0;
}
```

然后是用来hook的库hook.c：

```c
#include <stdio.h>
#include <string.h>
#include <dlfcn.h>

typedef int(*STRCMP)(const char*, const char*);

int strcmp(const char *s1, const char *s2)
{
	static void *handle = NULL;
	static STRCMP old_strcmp = NULL;

	if( !handle )
	{
		handle = dlopen("libc.so.6", RTLD_LAZY);
		old_strcmp = (STRCMP)dlsym(handle, "strcmp");
	}
	printf("hack function invoked. s1=<%s> s2=<%s>\n", s1, s2);
	return old_strcmp(s1, s2);
}
```

因为hook的目标是strcmp，所以typedef了一个STRCMP函数指针。由于hook的目的是要控制函数行为，所以需要从原库libc.so.6中拿到“正版”strcmp指针，保存成old_strcmp以备调用。

Makefile：

```bash
test: main.c hook.so
	gcc -o test main.c

hook.so: hook.c
	gcc -fPIC -shared -o hook.so hook.c -ldl

```

执行：

```bash
$ LD_PRELOAD=./hook.so ./test 123
hack function invoked. s1=<123> s2=<test>
Incorrect password

$ LD_PRELOAD=./hook.so ./test test
hack function invoked. s1=<test> s2=<test>
Correct password
```

其中有一点不理解的是，`dlopen`打开libc.so.6能拿到“正版”strcmp地址，打开libc.so就是hook后的地址。照理说libc.so不是libc.so.6的一个软链吗？为什么结果会不一样嘞？

-----
参考资料：

[Reverse Engineering with LD_PRELOAD](http://www.exploit-db.com/papers/13233/ "")
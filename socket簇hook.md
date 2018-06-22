### 运行时动态链接
应用程序可以 Hook 系统函数，OS 提供的支持是允许运行时加载和链接共享库。Linux提供了四个函数 `dlopen（），dlsym（），dlclose（），dlerror（）`实现了动态链接加载器的接口，详细用法和功能请看 man 手册。

Libco 对 read、write 等函数的 Hook 主要是用到 `dlsym` 这个函数，原型为 `void * dlsym（void * handle ，const char * symbol ）;`，进程可以使用这个函数获取特定符号的地址，参数为 句柄和符号名称，将此符号地址返回给调用方。这个函数提供了一个特殊的伪句柄 `RTLD_NEXT`，它允许从调用方链接映射列表中的下一个关联目标文件获取符号，也就是说程序可以在符号范围内查找下一个符号。这样可以使用 `RTLD_NEXT` ，通过在前面的目标文件中插入目标文件中的函数，然后扩充原始函数的处理。

拿我们熟悉的 `malloc` 来举例子，假设我们想申请空间的同时统计申请的次数，定义一个自己的 `malloc`，在程序中调用 `malloc` 时重定位到自己定义的函数。


```
	#include <stdio.h>
	#include <dlfcn.h>
    #include <stdlib.h>
    static unsigned int count = 0;
    void *malloc(size_t size)
    {
        void *(*myMalloc)(size_t) = dlsym(RTLD_NEXT, "malloc");
        printf("call my malloc\n");
        count++;
        return myMalloc(size);
    }
```
这样就告诉链接器将 `malloc` 这个符号的定义解析到后面的共享库，而不要在本模块查找这个符号。我们把这个文件编译成动态链接库。
接下来编写测试代码，调用一下我们自定义的 `malloc`。

```
    #include <stdio.h>
    #include <stdlib.h>

    int main()
    {
        void *p = malloc(2);
        free(p);
        printf("Good\n");
        return 0;
    }
```
![](http://test-1252727452.costj.myqcloud.com/2018-06-22%2015-31-25%20%E7%9A%84%E5%B1%8F%E5%B9%95%E6%88%AA%E5%9B%BE.png)


### libco 的实现

libco 就是通过 dlsym 和 RTLD_NEXT 结合起来将 socket 函数封装起来。

```
ssize_t read( int fd, void *buf, size_t nbyte )
{
	HOOK_SYS_FUNC( read );
	
	if( !co_is_enable_sys_hook() )
	{
		return g_sys_read_func( fd,buf,nbyte );
	}
	rpchook_t *lp = get_by_fd( fd );

	if( !lp || ( O_NONBLOCK & lp->user_flag ) ) 
	{
		ssize_t ret = g_sys_read_func( fd,buf,nbyte );
		return ret;
	}
	int timeout = ( lp->read_timeout.tv_sec * 1000 ) 
				+ ( lp->read_timeout.tv_usec / 1000 );

	struct pollfd pf = { 0 };
	pf.fd = fd;
	pf.events = ( POLLIN | POLLERR | POLLHUP );

	int pollret = poll( &pf,1,timeout );

	ssize_t readret = g_sys_read_func( fd,(char*)buf ,nbyte );

	if( readret < 0 )
	{
		co_log_err("CO_ERR: read fd %d ret %ld errno %d poll ret %d timeout %d",
					fd,readret,errno,pollret,timeout);
	}

	return readret;
	
}
```
首先调用 `HOOK_SYS_FUNC` ，这个宏调用了 `dlsym`，把这个宏展开就是初始化 `g_sys_read_func` 函数，将系统提供的 `read` 函数地址赋给它， 其他函数像 `write` 类似。
```
	#define HOOK_SYS_FUNC(name) if( !g_sys_##name##_func ) { g_sys_##name##_func = (name##_pfn_t)dlsym(RTLD_NEXT,#name); }
```

然后判断这个协程是否允许 IO hook，如果不允许那就直接调系统提供的 `read` 了。
然后以 `fd` 为 `index` 从全局数组中得到一个 `rpchook_t`，这个结构记录了这个 `fd` 的一些信息，比如读写超时、标识是否非阻塞等。
下面调用 `poll`，注册可读事件，然后 `yield` 让出CPU。等到被唤醒后返回继续 `read`，返回读到的数据大小。

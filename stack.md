通常会创建数量很多的协程来支持高并发，这时候就要考虑协程栈内存的占用情况了。该如何设计协程栈？


### 独立栈

```

struct stCoRoutineAttr_t

{

	int stack_size;

	stShareStack_t*  share_stack;

	stCoRoutineAttr_t()

	{

		stack_size = 128 * 1024;

		share_stack = NULL;

	}

}__attribute__ ((packed));

```

share_stack 为 NULL，stack_size 为 128K，libco 默认的是每一个协程独享一个栈空间，是用下面这种结构描述的：

```

struct stStackMem_t

{

	stCoRoutine_t* occupy_co; 		// 被哪个协程占有

	int stack_size; 				// 栈大小

	char* stack_bp; //stack_buffer + stack_size

	char* stack_buffer; 			// 栈空间



};

```

在协程创建的时候，会调用 `co_alloc_stackmem` 从堆内存中分配一个固定大小的内存（128K）作为该协程的运行栈。

```

/////////////////for copy stack //////////////////////////

stStackMem_t* co_alloc_stackmem(unsigned int stack_size)

{

	stStackMem_t* stack_mem = (stStackMem_t*)malloc(sizeof(stStackMem_t));

	stack_mem->occupy_co= NULL;

	stack_mem->stack_size = stack_size;

	stack_mem->stack_buffer = (char*)malloc(stack_size);

	stack_mem->stack_bp = stack_mem->stack_buffer + stack_size;

	return stack_mem;



```

接下来将这个申请的栈空间交给这个新的协程。

```

	lp->stack_mem = stack_mem;



	lp->ctx.ss_sp = stack_mem->stack_buffer;

	lp->ctx.ss_size = at.stack_size;

```



### 共享栈

libco 也提供了共享栈模式，可以设置若干个协程共享同一个运行栈。

如果要用共享栈模式，首先调用 `co_alloc_sharestack` 函数来分配共享栈的空间，共享栈结构的描述是`stShareStack_t`。count 是分配多少个共享栈， stack_size 是每个栈的大小，函数里面循环调用 `co_alloc_stackmem` 来申请每个栈。

```

stShareStack_t* co_alloc_sharestack(int count, int stack_size)

{

	stShareStack_t* share_stack = (stShareStack_t*)malloc(sizeof(stShareStack_t));

	share_stack->alloc_idx = 0;

	share_stack->stack_size = stack_size;



	//alloc stack array

	share_stack->count = count;

	stStackMem_t** stack_array = (stStackMem_t**)calloc(count, sizeof(stStackMem_t*));

	for (int i = 0; i < count; i++)

	{

		stack_array[i] = co_alloc_stackmem(stack_size);

	}

	share_stack->stack_array = stack_array;

	return share_stack;

}

```



```

struct stShareStack_t

{

	unsigned int alloc_idx;  		

	int stack_size;         

	int count;

	stStackMem_t** stack_array;

};

```

共享栈是一个数组，里面有 count 个元素，每个元素都是指向一段内存的指针 stStackMem_t。



新创建协程时，如果使用共享栈，会调用 `co_get_stackmem` 从刚分配的 `stShareStack_t` 中按照 `RoundRobin` 的方式取一个 `stStackMem_t` 出来，`alloc_idx` 就是起这个作用的。可以看出来这些栈就是共享的，所有协程都可以申请的到。

```

static stStackMem_t* co_get_stackmem(stShareStack_t* share_stack)

{

	if (!share_stack)

	{

		return NULL;

	}

	int idx = share_stack->alloc_idx % share_stack->count;

	share_stack->alloc_idx++;



	return share_stack->stack_array[idx];

}

```



独立栈是每个协程都有独立的栈，所有局部变量信息都保存在栈上。所有协程共享栈的时候如何保证在协程上下文切换时，顺利恢复执行环境？观察 `co_swap`函数，再真正进行 `swap context`之前会先更新这个栈当前归属的协程以及下一个协程，先对当前执行的协程进行 `save_stack_buffer(occupy_co);` 。

```

co_swap:

    else

	{

		env->pending_co = pending_co;

		//get last occupy co on the same stack mem

		stCoRoutine_t* occupy_co = pending_co->stack_mem->occupy_co;

		//set pending co to occupy thest stack mem;

		pending_co->stack_mem->occupy_co = pending_co;



		env->occupy_co = occupy_co;

		if (occupy_co && occupy_co != pending_co)

		{

			save_stack_buffer(occupy_co);

		}

	}

```

下面看一下 `save_stack_buffer` 做了什么。它会计算 `bp 和 sp` 之间的距离，得到目前这个协程使用的栈空间大小，然后重新 `malloc` 这么大的空间把栈上的内容全部复制过去，这些内容存在每个协程自己的结构 `stCoRoutine_t` 上 的 `save_buffer` 中，所以每个协程的拥有的内容也是独立的。这也叫做 `copy out`。



```

void save_stack_buffer(stCoRoutine_t* occupy_co)

{

	///copy out

	stStackMem_t* stack_mem = occupy_co->stack_mem;

	int len = stack_mem->stack_bp - occupy_co->stack_sp;



	if (occupy_co->save_buffer)

	{

		free(occupy_co->save_buffer), occupy_co->save_buffer = NULL;

	}



	occupy_co->save_buffer = (char*)malloc(len); //malloc buf;

	occupy_co->save_size = len;



	memcpy(occupy_co->save_buffer, occupy_co->stack_sp, len);

}

```



那么 `coctx_swap` 之后呢？此时协程是被 `resume` 唤醒了，这时就把保存的数据重新 `memcpy` 到共享栈上（`copy in`），再继续执行。

```

	//stack buffer may be overwrite, so get again;

	stCoRoutineEnv_t* curr_env = co_get_curr_thread_env();

	stCoRoutine_t* update_occupy_co =  curr_env->occupy_co;

	stCoRoutine_t* update_pending_co = curr_env->pending_co;

	

	if (update_occupy_co && update_pending_co && update_occupy_co != update_pending_co)

	{

		//resume stack buffer

		if (update_pending_co->save_buffer && update_pending_co->save_size > 0)

		{

			memcpy(update_pending_co->stack_sp, update_pending_co->save_buffer, update_pending_co->save_size);

		}

	}

```

可以看出共享栈切换上下文的前后，有一个备份栈内容和恢复的过程。



### 总结一下共享栈和独立栈各有什么特点。



独立栈：

- 每个协程分配独立的栈空间；

- 无需copy out、copy in，协程频繁切换对性能影响小；

- 栈空间大小固定，但是往往用不到这么大空间，可能会造成很大的内存浪费；

- 当然如果栈使用大小超过栈大小的话会导致栈溢出；

- 空间是独立的，那么可以修改另一个协程栈上的变量。



共享栈：

- 唯一的运行栈，大小自己指定，可以很大；

- 每个协程按需分配栈空间；

- 协程切换的时候备份栈，唤醒后恢复栈；

- 如果切换频繁，会多次备份恢复，性能影响大；

- 绝大多数协程使用的栈空间很小，copy 栈的开销也比较小；

- 内存占用更小。

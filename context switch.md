### X86-64 汇编基础

Libco切换协程那一段是用汇编实现的，所以想看懂还是要有一点汇编的基础，包括语法格式、寄存器类型、函数调用约定等等。Libco里面32位和64位都实现了，64位和32位相比，有了很多不同，包括寄存器类型由8个变为32个、新增了一些汇编指令、函数调用的过程、参数传递等等。现在PC基本都是64位了，重点也是看64位的。以一个例子来看：

```
/* add.c */
int Sum(int a, int b)
{
	return a + b;
}
int main()
{
	int ret = Sum(1, 2);
    printf("1 + 2 = %d\n", ret);

	return 0;
}
```

使用 GCC 的汇编器，使用这个命令 `gcc -S Add.c` 生成汇编代码文件 `Add.s`，内容如下：
```
	.file	"add.c"
	.text
	.globl	Sum

	.type	Sum, @function

Sum:

.LFB2:

	.cfi_startproc

	pushq	%rbp

	.cfi_def_cfa_offset 16

	.cfi_offset 6, -16

	movq	%rsp, %rbp

	.cfi_def_cfa_register 6

	movl	%edi, -4(%rbp)

	movl	%esi, -8(%rbp)

	movl	-4(%rbp), %edx

	movl	-8(%rbp), %eax

	addl	%edx, %eax

	popq	%rbp

	.cfi_def_cfa 7, 8

	ret

	.cfi_endproc

.LFE2:

	.size	Sum, .-Sum

	.section	.rodata

.LC0:

	.string	"1 + 2 = %d\n"

	.text

	.globl	main

	.type	main, @function

main:

.LFB3:

	.cfi_startproc

	pushq	%rbp                   # 栈基址指针入栈

	.cfi_def_cfa_offset 16

	.cfi_offset 6, -16

	movq	%rsp, %rbp

	.cfi_def_cfa_register 6

	subq	$16, %rsp

	movl	$2, %esi

	movl	$1, %edi

	call	Sum

	movl	%eax, -4(%rbp)

	movl	-4(%rbp), %eax

	movl	%eax, %esi

	movl	$.LC0, %edi

	movl	$0, %eax

	call	printf

	movl	$0, %eax

	leave

	.cfi_def_cfa 7, 8

	ret

	.cfi_endproc

.LFE3:

	.size	main, .-main

	.ident	"GCC: (GNU) 7.1.0"

	.section	.note.GNU-stack,"",@progbits

```

##### 寄存器

X86-64有16个寄存器，这些几乎都是通用的，不像早期处理器的不同寄存器的用途不同，现在基本上都是通用的，除了一些特殊的指令，比如和字符串处理有关的指令要用 %rdi和%rsi。这16个寄存器分别是：

![](http://test-1252727452.costj.myqcloud.com/regs.png)



上面代码是传统的 AT&T 语法，在类Unix系统上使用，像 DOS 和 Windows 上用Intel语法，可以用有无 % 来区分。还有一个区别是 AT&T 格式和 Intel 格式的指令的源操作数和目的操作数的顺序是相反的，AT&T 格式是源操作数在前，目的操作数在后；另一个区别是AT&T格式指令如果要控制操作数长度是通过指令来实现的，而Intel格式是通过在操作数前加限定来进行。


最常用的寻址的汇编指令是 mov，和大多数指令一样，通过单字母后缀来确定要移动的数据量。

| 前缀 | 全称 | Size |

|--------|--------|--------|

|  B |  BYTE   | 1 byte(8 bits)   |

|  W |  WORD   | 2 byte(16 bits)   |

|  L |  LONG   | 4 byte(32 bits)   |

|  Q |  QUADWORD  | 8 byte(64 bits)   |


比如 AT&T格式：`movb val,%al`，而Intel格式这样写：`mov al byte ptr val`。


##### 寻址方式

就是处理器根据给出的地址信息来寻找有效地址，取得数据地址的不同表达方式。


下面列举几种例子中用到的的寻址方式：


| 寻址方式 | 示例 | 注释  |

|--------|--------|--------|

|   立即数寻址 |  movq  $11, %rax   | 整数常量由美元符号+数值表示，把数字放到寄存器   |

|   直接寻址 |  movl  0x123, %rax   | 把 0x123指向内存数据放到寄存器   |

|   寄存器寻址 |  movq %rbx, %rax   | 直接使用寄存器的名字访问寄存器的值   |

|   间接寻址 |  movq (%rsp),%rax   | 如果寄存器中存放的是一个地址，访问这个地址中的数据时要在寄存器外面加上括号   |

|   相对基址寻址 |  movq -8(%rbp), %rax   | 指的是 %rbp 中存放的地址减 8 字节存储单元的值   |


##### 函数调用

在x86 32位机器中调用约定还比较简单，栈是从高地址到地址增长的，用来支持函数调用操作。单个函数调用操作使用的栈部分称为栈帧，栈帧的两端由两个指针（寄存器）来指定。ebp通常用作帧指针，esp用作栈指针，esp指向栈顶，随着数据的入栈和出栈移动，在函数执行过程中，对大部分数据的访问都基于帧指针ebp进行。

	

A(caller) 调用 B(callee)，传递给B的参数保存在A的栈帧中，通常是将参数从右向左压栈，再将调用后的返回地址（调用返回后继续执行的指令地址）压入栈中。此后将执行流程交给被调用方。



观察被调用函数汇编后的代码，通常会以这样 `push %ebp; move %ebp,%esp;sub $esp N;` 的形式开头，主要是为了方便访问参数和自身的局部变量。

```

     :         : 

     |  ...    | 

     |  para2  |   [ebp + 12] (2nd argument)

     |  para1  |   [ebp + 8]  (1st argument)

     |    RA   |   [ebp + 4]  (return address)

     |    FP   |   [ebp]      (old ebp value)

     |         |   [ebp - 4]  (1st local variable)

     :         :

     :         :

        frame

```

栈是往低地址增长的，esp指向当前栈顶处的元素。通常用 push 和 pop 指令可以把数据压入栈中或者从栈中弹出。给没有初值的数据分配空间，可以通过把 esp 递减适当的值，类似地，通过增加栈指针的值可以回收栈中已分配的空间。



指令 call 和 ret 用于处理函数调用和返回操作。call 的作用是把返回地址压入栈中并且跳转到被调用函数开始处执行，返回地址是紧跟在call后面一条指令的地址，所以当被调函数返回后就从这个位置继续执行。返回指令 ret 用于弹出栈顶处的地址并跳转到该地址处，在使用这个指令之前，需要先正确处理栈中内容，使esp指向先前指定的返回地址。通常处理内容这个功能是通过 leave指令完成，它包含两步 `mov %ebp,%esp; pop %ebp`。



X86-64 完整的约定很复杂，先了解以下几点：

- 整数参数包括指针按顺序放在寄存器 %rdi、%rsi、%rdx、%rcx、%r8 和 %r9 中，当参数超过6个，才会通过压栈的方式传参数；

- 函数的返回值存储在 %rax 中，%rsp栈指针寄存器指向栈顶；

- 被调用的函数可以使用任何寄存器，但如果它们发生了变化，则必须恢复寄存器 %rbx、%rbp和 %r12-%r15的值。

- r10、r11用作数据存储，遵循调用者使用规则，就是使用之前要先保存原值。

上面那张寄存器图最右边也标明了各寄存器的作用，以及是否需要保存。



##### 两整数相加汇编代码分析

现在看一下上面的汇编代码：

```

main:

	pushq	%rbp                   

	movq	%rsp, %rbp

```

从 main 函数的第一条指令看，首先将当前的帧指针 %rbp 入栈，函数调用结束后就可以从栈中取得函数调用前 %ebp 指向的位置，从而恢复栈到之前的样子。然后使当前栈指针指向新的位置。

```

	subq	$16, %rsp

	movl	$2, %esi

	movl	$1, %edi

```

申请 16字节的空间存放后面的临时变量 x，根据调用约定将传递给 Sum 函数的参数放入 %esi、%edi 中。编译器没有把需要保存的 %r10、%r11入栈，因为 main 函数中不会使用到这两个寄存器，所以不需要保存。

```

    call	Sum

```

将返回地址压入栈中，也就是紧跟在 call 后面一条指令的地址，这样当被调函数返回后就知道接下来该执行哪条指令。

```

Sum:

	pushq	%rbp

	movq	%rsp, %rbp

	

    movl	%edi, -4(%rbp)

	movl	%esi, -8(%rbp)

	movl	-4(%rbp), %edx

	movl	-8(%rbp), %eax

	addl	%edx, %eax

```

这里是调用 Sum 函数，前两行和上面一样，先保存 %rbp，然后更新栈指针，然后直接使用栈空间将两个局部变量保存，再相加，将结果保存在 %eax 中。

```

	popq	%rbp

	ret

```

上面局部变量保存并没有影响 rsp 的移动，执行完 `popq %rbp` 恢复调用前的帧指针，此时栈顶元素就是函数调用之后需要执行的下一条指令的地址，ret 指令等价于 `popq %rip`，这样就可以跳转到调用结束后的下一条指令处继续执行。





**后面还会看到 lea 指令，这个和 mov 有什么区别呢？**

mov 是计算内存地址，然后把里面的值读出来放在寄存器里；

lea 是计算内存地址，然后把内存地址本身放进寄存器里。



参考资料：[Introduction to X86-64 Assembly for Compiler Writers](https://www3.nd.edu/~dthain/courses/cse40243/fall2015/intel-intro.html)

### Libco 上下文切换源码分析(X86-64)



这个结构来表示协程运行的上下文环境。

```

#ifndef __CO_CTX_H__

#define __CO_CTX_H__

#include <stdlib.h>

typedef void* (*coctx_pfn_t)( void* s, void* s2 );

struct coctx_param_t

{

	const void *s1;     // routine

	const void *s2;     // routine 的参数

};

// 这个结构来表示一个协程的上下文环境

struct coctx_t

{

#if defined(__i386__)

	void *regs[ 8 ];    // 用来保存寄存器的值

#else

	void *regs[ 14 ];   // 64位一共16个，没用r10 r11

#endif

	size_t ss_size;     // 栈空间大小

	char *ss_sp;        // 栈指针,也就是栈的起始地址

	

};



int coctx_init( coctx_t *ctx );

int coctx_make( coctx_t *ctx,coctx_pfn_t pfn,const void *s,const void *s1 );

#endif

```

第一次 resume 启动一个协程的时候会初始化这个结构，代码如下：



```

#define RSP 0

#define RIP 1

#define RBX 2

#define RDI 3

#define RSI 4


#define RBP 5

#define R12 6

#define R13 7

#define R14 8

#define R15 9

#define RDX 10

#define RCX 11

#define R8 12

#define R9 13

//-------------

// 64 bit

//low | regs[0]: r15 |

//    | regs[1]: r14 |

//    | regs[2]: r13 |

//    | regs[3]: r12 |

//    | regs[4]: r9  |

//    | regs[5]: r8  | 

//    | regs[6]: rbp |

//    | regs[7]: rdi |

//    | regs[8]: rsi |

//    | regs[9]: ret |  //ret func addr

//    | regs[10]: rdx |

//    | regs[11]: rcx | 

//    | regs[12]: rbx |

//hig | regs[13]: rsp |

enum
{
	kRDI = 7,

	kRSI = 8,

	kRETAddr = 9,

	kRSP = 13,
};

//64 bit

extern "C"
{
	extern void coctx_swap( coctx_t *,coctx_t* ) asm("coctx_swap");
};

#elif defined(__x86_64__)
/*

 * param:

 *  ctx:协程使用的上下文环境对象

 *  pfn:协程对应的执行函数

 *  s s1:传给执行函数的参数

 */
int coctx_make( coctx_t *ctx,coctx_pfn_t pfn,const void *s,const void *s1 )
{
    // 取得栈指针，栈从高地址向低地址增长

	char *sp = ctx->ss_sp + ctx->ss_size;

    // -16LL的16进制为0xfffffffffffffff0,

    // sp & 上这个也就是最后16位清零，

	sp = (char*) ((unsigned long)sp & -16LL  );

	memset(ctx->regs, 0, sizeof(ctx->regs));


    // kRSP 是 13, 用来保存栈顶地址，这里减去了8字节，与后面切换协程时相对应

	ctx->regs[ kRSP ] = sp - 8;


    // kRETAddr 是 9, 用于存放要执行的函数地址，等coctx_swap执行结束自然会

	ctx->regs[ kRETAddr] = (char*)pfn;


    // 这个要根据函数调用约定，这两个参数要依次放入 rdi rsi

    // 这样启动协程的时候传给函数pfn

	ctx->regs[ kRDI ] = (char*)s;

	ctx->regs[ kRSI ] = (char*)s1;

	return 0;

}

int coctx_init( coctx_t *ctx )
{
	memset( ctx,0,sizeof(*ctx));

	return 0;
}

#endif

```

下面是 coctx_swap，挂起当前协程就是把相关寄存器保存到当前协程对应的 coctx_t 结构体中，然后恢复待唤醒协程的 coctx_t 到寄存器中，来完成协程的切换。

1. 保存上下文（寄存器）
2. 切换栈指针 rsp
3. 恢复上下文（寄存器）
4. 跳转恢复执行

```

#elif defined(__x86_64__)

	leaq 8(%rsp),%rax           # rax = *rsp + 8

    							# 这个时候栈顶存放的是当前的 %rip，返回地址

    							# 也就是当前协程挂起后被再次唤醒时需要执行的下一条指令的地址

    							# 后面会把这个地址保存到curr->ctx->regs[9]中，所以这里保存跳过


	leaq 112(%rdi),%rsp         # rsp = *rdi + 8*14;

    							# rdi 存放的是第一个参数的地址，也就是 curr->ctx的地址

    							# 然后加上需要保存的14个寄存器的长度，使rsp指向curr->ctx->regs[13]

	pushq %rax                  # curr->ctx->regs[13] = rax;

	pushq %rbx                  # curr->ctx->regs[12] = rbx;

	pushq %rcx                  # curr->ctx->regs[11] = rcx;

	pushq %rdx                  # curr->ctx->regs[10] = rdx;

	pushq -8(%rax) #ret func addr curr->ctx->regs[9] = *rax-8;

	pushq %rsi                  # curr->ctx->regs[8] = rsi;

	pushq %rdi                  # curr->ctx->regs[7] = rdi;

	pushq %rbp                  # curr->ctx->regs[6] = rbp;

	pushq %r8                   # curr->ctx->regs[5] = r8;

	pushq %r9                   # curr->ctx->regs[4] = r9;

	pushq %r12                  # curr->ctx->regs[3] = r12;

	pushq %r13                  # curr->ctx->regs[2] = r13;

	pushq %r14                  # curr->ctx->regs[1] = r14;

	pushq %r15                  # curr->ctx->regs[0] = r15;



	movq %rsi, %rsp             # rsp = rsi;

    							# rsi保存的是第二个参数的地址,现在rsp 指向pending_co->ctx->regs[0]

	popq %r15                   # r15 = pending_co->ctx->regs[0];

	popq %r14                   # r14 = pending_co->ctx->regs[1];

	popq %r13                   # r13 = pending_co->ctx->regs[2];

	popq %r12                   # r12 = pending_co->ctx->regs[3];

	popq %r9                    # r9 = pending_co->ctx->regs[4];

	popq %r8                    # r8 = pending_co->ctx->regs[5];

	popq %rbp                   # rbp = pending_co->ctx->regs[6];

	popq %rdi                   # rdi = pending_co->ctx->regs[7];

	popq %rsi                   # rsi = pending_co->ctx->regs[8];

	popq %rax                   # rax = pending_co->ctx->regs[9];

	popq %rdx                   # rdx = pending_co->ctx->regs[10];

	popq %rcx                   # rcx = pending_co->ctx->regs[11];

	popq %rbx                   # rbx = pending_co->ctx->regs[12];

	popq %rsp                   # rsp = pending_co->ctx->regs[13];rsp+=8;


    pushq %rax                  # rsp -= 8; rsp 存放的是rax的值

								# 此时栈顶存放的就是协程被唤醒后需要执行的下一条指令的地址了



    xorl %eax, %eax             # eax 清零

	ret                         # 弹出栈顶处的地址并跳转到这个地址

#endif

```


参考资料：

- www.cnblogs.com/ym65536/p/4542646.html


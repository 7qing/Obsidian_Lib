
Linux通过slab动态生成task_struct，那么在栈顶或栈底创建新的结构体thread_info即可，其中task指向其真正的task_struct结构体。

```c
struct thread_info {
	struct task_struct	*task;		//主要的进程描述符
	struct exec_domain	*exec_domain;
	__u32			flags;		
	__u32			status;		// 线程同步flags
	__u32			cpu;		//当前cpu
	int			preempt_count;
	mm_segment_t		addr_limit;
	struct restart_block    restart_block;
	void __user		*sysenter_return;
	unsigned int		sig_on_uaccess_error:1;
	unsigned int		uaccess_err:1;
};
```

### 内核stack和thread_info结构的关系

```c
union thread_union {
    struct thread_info thread_info;
    unsigned long stack[THREAD_SIZE/sizeof(long)];
};
 
#define THREAD_SIZE        16384 \\8k
#define THREAD_START_SP        (THREAD_SIZE - 16)
```

内核定义了一个thread_union的联合体，联合体的作用就是thread_info和stack共用一块内存区域。

![](computer_system/图片/thread_info与内核栈的关系.png)

我们获得当前内核栈的sp指针的地址，然后根据THREAD_SIZE对齐就可以获取thread_info结构的基地址，然后从thread_info.task就可以获取当前task_struct结构的地址了。

```c
#define get_current() (current_thread_info()->task)
#define current get_current()
 
/*
 * how to get the current stack pointer from C
 */
register unsigned long current_stack_pointer asm ("sp");
 
/*
 * how to get the thread information struct from C
 */
static inline struct thread_info *current_thread_info(void) __attribute_const__;
 
static inline struct thread_info *current_thread_info(void)
{
    return (struct thread_info *)
        (current_stack_pointer & ~(THREAD_SIZE - 1));
}
```

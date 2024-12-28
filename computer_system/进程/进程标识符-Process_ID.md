进程标识符（Process ID）是进程描述符中最重要的组成部分，是在当前 Linux 系统中唯一的一个非负整数，用于标识和对应唯一的进程。

在Linux中，有几个特殊的进程标识符所对应的进程。

* 进程标识符0：对应的是交换进程（swapper），用于执行多进程的调用。
* 进程标识符1：对应的是初始化进程（init），在自举过程结束时由内核调用，对应的文件是/sbin/init，负责 Linux 的启动工作，这个进程在系统运行过程中是不会终止的，可以说当前操作系统中的所有进程都是这个进程衍生而来的。
* 进程标识符2：可能对应页守护进程（pagedaemon），用于虚拟存储系统的分页操作。


```c
#include <sys/types.h> 
#include <unistd.h> 
pid_t getpid(void); 
pid_t getppid(void);
```


由于循环使用PID编号，内核必须通过管理一个pidmap-array位图来表示当前已分配的PID号和闲置的PID号。因为一个页框包含32768个位，所以在32位体系结构中pidmap-array位图存放在一个单独的页中。然而，在64位体系结构中，当内核分配了超过当前位图大小的PID号时，需要为PID位图增加更多的页。系统会一直保存这些页不被释放。

对于pid的分配，是在[`copy_process()`](/computer_system/进程/创建进程#`copy_process`)过程中完成的：
```c
static struct task_struct *copy_process(unsigned long clone_flags,
					unsigned long stack_start,
					struct pt_regs *regs,
					unsigned long stack_size,
					int __user *child_tidptr,
					struct pid *pid,
					int trace)
{
	struct task_struct *p;
	...
	if (pid != &init_struct_pid) {
		retval = -ENOMEM;
		pid = alloc_pid(p->nsproxy->pid_ns); // 分配pid
		if (!pid)
			goto bad_fork_cleanup_io;
	}

	p->pid = pid_nr(pid); //设置pid
	...
}
```

#### alloc_pid()

```c
struct pid *alloc_pid(struct pid_namespace *ns)
{
	struct pid *pid;
	enum pid_type type;
	int i, nr;
	struct pid_namespace *tmp;
	struct upid *upid;

	pid = kmem_cache_alloc(ns->pid_cachep, GFP_KERNEL); //分配pid结构体的内存
	if (!pid)
		goto out;

	tmp = ns;
	for (i = ns->level; i >= 0; i--) {
		nr = alloc_pidmap(tmp); //分配pid
		if (nr < 0)
			goto out_free;

		pid->numbers[i].nr = nr; //nr保存到pid结构体
		pid->numbers[i].ns = tmp;
		tmp = tmp->parent;
	}

	get_pid_ns(ns);
	pid->level = ns->level;
	atomic_set(&pid->count, 1);
	for (type = 0; type < PIDTYPE_MAX; ++type)
		INIT_HLIST_HEAD(&pid->tasks[type]); //初始化pid的hlist结构体

	upid = pid->numbers + ns->level;
	spin_lock_irq(&pidmap_lock);
	for ( ; upid >= pid->numbers; --upid)
		hlist_add_head_rcu(&upid->pid_chain,
				&pid_hash[pid_hashfn(upid->nr, upid->ns)]); //建立pid_hash的关联关系
	spin_unlock_irq(&pidmap_lock);

out:
	return pid;

out_free:
	while (++i <= ns->level)
		free_pidmap(pid->numbers + i);

	kmem_cache_free(ns->pid_cachep, pid);
	pid = NULL;
	goto out;
}
```

我们来分析几个比较重要的地方：

##### `kmem_cache_alloc(ns->pid_cachep, GFP_KERNEL);`
1. **`kmem_cache_alloc`**：
    - 这是内核中用于分配内存的函数，它从特定的内核缓存中分配对象。内核通过这种方法来提高分配效率，并减少内存碎片。
    - 这里用 `kmem_cache_alloc` 从 PID 缓存中分配一个新的 `pid` 结构。
    - 内核使用对象缓存（[slab 缓存]()）来存储固定大小的对象（如 `pid` 结构）。通过这种机制，可以避免频繁的通用内存分配带来的性能损耗，同时减少内存碎片化。
1. **`ns->pid_cachep`**：
    - `pid_cachep` 是当前命名空间 ([namespace](#pid_namespace结构体)) 中用于管理 PID 的缓存指针。每个命名空间都有自己独立的 PID 缓存，用于存储与 PID 相关的数据。
1. **`GFP_KERNEL`**：
    - 这是分配标志，表示分配是在内核上下文中执行的，允许内存分配时进行阻塞等待。这是内核中最常用的分配模式。
2. **`pid`**：
    - `pid` 是一个变量，用于保存分配到的 `pid` 结构（进程标识符结构）。

##### `alloc_pidmap(tmp)`

```c
static int alloc_pidmap(struct pid_namespace *pid_ns)
{
  //last_pid为上次分配出去的pid
  int i, offset, max_scan, pid, last = pid_ns->last_pid;
  struct pidmap *map;

  pid = last + 1;
  if (pid >= pid_max)
    pid = RESERVED_PIDS; //默认为300
    
  offset = pid & BITS_PER_PAGE_MASK; //最高位值置0，其余位不变
  map = &pid_ns->pidmap[pid/BITS_PER_PAGE]; //找到目标pidmap

  //当offset =0，则扫描一次;
  //当offset!=0，则扫描两次
  max_scan = DIV_ROUND_UP(pid_max, BITS_PER_PAGE) - !offset;
  for (i = 0; i <= max_scan; ++i) {
    if (unlikely(!map->page)) {
      void *page = kzalloc(PAGE_SIZE, GFP_KERNEL);
      spin_lock_irq(&pidmap_lock);
      if (!map->page) {
        map->page = page;
        page = NULL;
      }
      spin_unlock_irq(&pidmap_lock);
      kfree(page);
      if (unlikely(!map->page))
        break;
    }
    
    //当pidmap还有可用pid时
    if (likely(atomic_read(&map->nr_free))) {
      do {
        //当offset位空闲时返回该pid
        if (!test_and_set_bit(offset, map->page)) {
          atomic_dec(&map->nr_free); //可用pid减一
          set_last_pid(pid_ns, last, pid); //设置last_pid
          return pid;
        }
        //否则，查询下一个非0的offset值
        offset = find_next_offset(map, offset);
        根据offset转换成相应的pid
        pid = mk_pid(pid_ns, map, offset);
      } while (offset < BITS_PER_PAGE && pid < pid_max);
    }
    
    //当上述pid分配失败，则再次查找offset
    if (map < &pid_ns->pidmap[(pid_max-1)/BITS_PER_PAGE]) {
      ++map;
      offset = 0;
    } else {
      map = &pid_ns->pidmap[0];
      offset = RESERVED_PIDS;
      if (unlikely(last == offset))
        break;
    }
    pid = mk_pid(pid_ns, map, offset);
  }
  return -1;
}

```

##### `pid_namespace结构体`

```c
struct pid_namespace {
  struct kref kref;  
  /* 引用计数，用于管理 `pid_namespace` 的生命周期。通过 kref 机制，
     当引用计数减为 0 时，`pid_namespace` 会被释放。 */

  struct pidmap pidmap[PIDMAP_ENTRIES];  
  /* PID 位图数组，每个 pidmap 用于标记哪些 PID 是空闲的，
     哪些已被分配。位图是用来高效管理 PID 分配的核心数据结构。 */

  struct rcu_head rcu;  
  /* 用于 RCU（Read-Copy Update）机制，当 `pid_namespace` 被释放时，
     确保在延迟删除期间不会破坏并发访问。 */

  int last_pid;  
  /* 最近一次分配的 PID。这个字段用于生成下一个 PID（从上次分配的 PID 开始递增）。 */

  unsigned int nr_hashed;  
  /* 当前命名空间中被分配 PID 的数量。这个值反映了命名空间中活动进程的数量。 */

  struct task_struct *child_reaper;  
  /* 指向该命名空间中充当 `init` 进程的任务结构指针。
     如果命名空间中没有任务存活，`child_reaper` 会负责回收孤儿进程。 */

  struct kmem_cache *pid_cachep;  
  /* Slab 缓存，用于快速分配和回收 `pid` 结构体对象，
     减少内存分配的开销和碎片化。 */

  unsigned int level;  
  /* 命名空间的嵌套级别，`0` 表示初始命名空间，数字越大表示嵌套越深。 */

  struct pid_namespace *parent;  
  /* 指向父命名空间。如果是初始命名空间（最顶层），则为 NULL。 */

  // 省略部分字段 ...

  struct user_namespace *user_ns;  
  /* 与当前 `pid_namespace` 相关联的用户命名空间。
     用户命名空间定义了与权限和 UID/GID 有关的隔离。 */

  struct work_struct proc_work;  
  /* 用于延迟执行操作的 `work_struct` 结构体，
     主要和 `/proc` 文件系统的相关操作有关。 */

  kgid_t pid_gid;  
  /* PID 的组 ID，决定了 `/proc` 文件系统中 `pid` 的所有者组权限。 */

  int hide_pid;  
  /* 决定是否隐藏进程信息：
     - 0：所有用户均可查看其他进程的信息。
     - 1：仅允许用户查看与自己相关的进程。
     - 2：完全隐藏其他用户的进程信息。 */

  int reboot;  
  /* 用于存储命名空间是否允许重启操作。 */

  struct ns_common ns;  
  /* 通用命名空间结构，提供与内核命名空间接口相关的字段（如 `inum`）。 */
};

```

PID命名空间，这是为系统提供虚拟化做支撑的功能。
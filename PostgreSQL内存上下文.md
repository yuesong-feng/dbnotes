# PostgreSQL内存上下文

不管是什么样的数据库系统，存储管理的本质都是一样的：如何减少I/O次数。内存的访问速度至少是磁盘的数十万倍，所以通常读写磁盘所用的时间决定了数据库操作的总时间，而内存的访问时间可以忽略不计。因此，要尽可能的提高I/O命中率，让最可能被使用的文件块停留在内存中。除此之外，内存管理还是整个数据库系统的桥梁，每一个模块都会使用到内存进行函数运行、缓冲、消息传递等，内存管理对于数据库来说十分重要。

PostgreSQL7.1之前，大量以指针传值的查询可能会造成严重、不易排查的内存泄漏。从7.1版本开始，PostgreSQL使用内存上下文机制来管理内存。一个内存上下文就相当于一个进程的内存环境，每个进程的内存上下文组成一个树行结构，其根节点为`TopMemoryContext`，下面可以有很多字节点，如用于管理Cache的`CacheMemoryContext`、用于错误处理的`ErrorContext`、用于消息传递的`MessageMemoryContext`等，每个子节点又可以有自己的子节点。通过树形结构可以跟踪进程中内存上下文的创建和使用情况，每个节点定义如下：

```c
typedef struct MemoryContextData
{
	NodeTag		type;			/* 节点类型 */
    /* 以下两个字段放在这里可以减小字节对齐的浪费： */
    bool		isReset;		/* 为真表示从上次reset后没有分配过内存空间 */
	bool		allowInCritSection; /* 允许在critical section（临界区）使用palloc */
	Size		mem_allocated;	/* 该内存上下文中已经分配的内存大小 */
	const MemoryContextMethods *methods;	/* 内存处理函数的指针 */
	MemoryContext parent;		/* 父节点 */
	MemoryContext firstchild;	/* 第一个孩子节点 */
	MemoryContext prevchild;	/* 前一个兄弟节点 */
	MemoryContext nextchild;	/* 后一个兄弟节点 */
	const char *name;			/* 内存上下文的名称(用于调试) */
	const char *ident;			/* 内存上下文的id (用于调试) */
	MemoryContextCallback *reset_cbs;	/* reset/delete 回调函数的链表 */
} MemoryContextData;
typedef struct MemoryContextData *MemoryContext;
```

在任何时候都有一个当前的内存上下文，记录在全局变量`CurrentMemoryContext`里，进程在这个内存上下文中调用palloc函数来分配内存。在变化内存上下文时，可以使用`MemoryContextSwitchTo`切换。

`MemoryContext`中的`methods`字段是一系列包含了对内存上下文进行操作的函数，使用函数指针模拟虚函数设计，可以有不同的实现，定义如下：

```c
typedef struct MemoryContextMethods
{
	void	   *(*alloc) (MemoryContext context, Size size);        // 分配内存
	void		(*free_p) (MemoryContext context, void *pointer);   // 释放内存
	void	   *(*realloc) (MemoryContext context, void *pointer, Size size);   // 重分配内存
	void		(*reset) (MemoryContext context);                       // 重置内存上下文
	void		(*delete_context) (MemoryContext context);                 // 删除内存上下文
	Size		(*get_chunk_space) (MemoryContext context, void *pointer);  // 检查内存片段的大小
	bool		(*is_empty) (MemoryContext context);                        // 检查内存上下文是否为空
	void		(*stats) (MemoryContext context,                            // 内存上下文状态
						  MemoryStatsPrintFunc printfunc, void *passthru,
						  MemoryContextCounters *totals,
						  bool print_to_stderr);
#ifdef MEMORY_CONTEXT_CHECKING
	void		(*check) (MemoryContext context);                   // 检查所有内存片段
#endif
} MemoryContextMethods;
```

`MemoryContext`并不管理实际的内存块，仅仅作为一个头部记录在`AllocSetContext`结构里，来管理内存上下文之间的树形关系，真正拥有并管理内存块的是`AllocSet`，定义如下：

```c
typedef struct AllocSetContext
{
	MemoryContextData header;	/* 内存上下文头部 */
	/* 此上下文中分配的内存信息： */
	AllocBlock	blocks;			/* 内存块链表 */
	AllocChunk	freelist[ALLOCSET_NUM_FREELISTS];	/* 空闲内存片数组 */
	/* 此上下文分配内存相关参数： */
	Size		initBlockSize;	/* 初始内存块大小 */
	Size		maxBlockSize;	/* 最大内存块大小 */
	Size		nextBlockSize;	/* 下一个要分配的内存块大小 */
	Size		allocChunkLimit;	/* 分配内存片的大小阀值 */
	AllocBlock	keeper;			/* 保存在keeper中的内存块在内存上下文重值时会被保留不释放 */
	int			freeListIndex;	/* 此上下文在空闲上下文数组中的位置，不在即为-1 */
} AllocSetContext;
typedef AllocSetContext *AllocSet;
```

`AllocSet`所管理的内存区域被分成若干个内存块（AllocBlock），通过标准库函数`malloc`进行分配，形成一个链表，`blocks`字段指向这个内存块链表的头部：

```c
typedef struct AllocBlockData
{
	AllocSet	aset;			/* 该内存块所在的AllocSet */
	AllocBlock	prev;			/* 上一个内存块的地址 */
	AllocBlock	next;			/* 下一个内存块的地址 */
	char	   *freeptr;		/* 该内存块空闲区域的首地址 */
	char	   *endptr;			/* 该内存块的末地址 */
}			AllocBlockData;
typedef struct AllocBlockData *AllocBlock;
```

在每个内存块中可以进一步分配内存，产生的内存片段叫做内存片（AllocChunk），包括一个头部和数据区域，`AllocChunk`为内存片的头部，数据区域则紧跟在头部信息之后分配，通过`palloc`和`pfree`函数可以在内存上下文中申请、释放内存片，被释放的内存片将被加入到`freelist`中以备重复使用，`AllocChunk`定义如下：

```c
typedef struct AllocChunkData
{
	/* 内存片的实际大小，由于内存片都是以2的幂为大小进行对齐，因此申请的大小可能比实际大小要小*/
	Size		size;

#ifdef MEMORY_CONTEXT_CHECKING
	/* 调试内存使用情况时，存储实际的被使用的空间大小，如果是空闲内存片则为0 */
	Size		requested_size;

#define ALLOCCHUNK_RAWSIZE  (SIZEOF_SIZE_T * 2 + SIZEOF_VOID_P)
#else
#define ALLOCCHUNK_RAWSIZE  (SIZEOF_SIZE_T + SIZEOF_VOID_P)
#endif							/* MEMORY_CONTEXT_CHECKING */

	/* 如果需要，确保内存对齐 */
#if (ALLOCCHUNK_RAWSIZE % MAXIMUM_ALIGNOF) != 0
	char		padding[MAXIMUM_ALIGNOF - ALLOCCHUNK_RAWSIZE % MAXIMUM_ALIGNOF];
#endif

	void	   *aset;	// 该内存片所在的AllocSet，如果内存片为空闲，则用于链接其空闲链表
	// 该结构体后不能有内存对齐，本身应该是对齐的！
}		
typedef struct AllocChunkData *AllocChunk;
```

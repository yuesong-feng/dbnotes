# VFD——虚拟文件描述符
在操作系统中，每当一个进程打开一个文件，系统就会为该文件分配一个唯一的文件描述符，在Linux系统中是一个`int`类型的值。每个操作系统都会对一个进程能打开的文件数加以限制，用`ulimit -n`命令可以查看进程能打开的最大文件数。对于一个数据库系统，系统元数据和用户数据都可能保存在许多不同文件当中，而且常常会对大表进行排序、hash join等操作，需要经常打开大量文件，如果超过了操作系统的限制，就会出错或阻塞。为了解决这个问题，PostgreSQL使用了虚拟文件描述符（VFD）机制。

VFD的原理类似于进程池、内存池等池化技术，当进程申请打开一个文件时，总能分配一个虚拟的文件描述符，是真实文件描述符的一个上层封装。上层系统对文件进行操作时，只需要使用VFD即可，由VFD管理器转化为对实际文件描述符的操作。VFD使用LRU（最近最少使用）池来管理所有已经打开文件，当进程打开的文件个超过操作系统限制时，也可以申请一个新的VFD，从LRU链表中替换最长时间未使用的VFD并关闭相应的实际文件描述符。VFD也会管理一个空闲VFD链表，方便获取当前空闲的VFD。

```c
typedef struct vfd
{
	int			fd;				// 当前文件描述符
	unsigned short fdstate;		// 当前VFD的状态
	ResourceOwner resowner;		// 拥有者
	File		nextFree;		// 在空闲链表中表示下一个空闲的VFD
	File		lruMoreRecently;	// LRU链表中更近被使用的
	File		lruLessRecently;	// LRU链表中更远被使用的
	off_t		fileSize;		// 当前文件大小
	char	   *fileName;		// 当前文件名
	int			fileFlags;		// 打开文件时open()函数的flags
	mode_t		fileMode;		// 打开文件的模式
} Vfd;

static Vfd *VfdCache; 			 // VFD数组的指针
static Size SizeVfdCache = 0;	 // VFD数组的大小

static int	nfile = 0;			// 正在使用的VFD数量
```

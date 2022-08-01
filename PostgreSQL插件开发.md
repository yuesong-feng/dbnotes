# PostgreSQL插件开发

PostgreSQL中许多控制信息都是以系统表的形式来管理，这个特点决定了PostgreSQL比其他数据库更容易进行内核扩展。PostgreSQL还提供了丰富的数据库内核编程接口，允许开发者以插件的形式将自己的代码融入内核。

PostgreSQL插件开发非常简单，下面举一个例子，开发一个随机测试数据生成器。

插件名为`pg_testgen`，首先需要创建四个文件：

```bash
pg_testgen.control      # 插件名.control
pg_testgen.c            # 插件名.c
pg_testgen--1.0.sql     # 插件名--1.0.sql
Makefile                # 用于编译
```

`pg_testgen.control`是插件的控制文件，是一个只有五六行的模版，存放插件说明、默认版本号、模块路径、是否可重入等，只需要将插件名更改即可。
```
# pg_testgen extension
comment = 'PostgreSQL test generator'
default_version = '1.0'
module_pathname = '$libdir/pg_testgen'
relocatable = true
```

`Makefile`也基本上是模版，根据自己的插件名、文件名修改即可：

```makefile
# contrib/pg_testgen/Makefile
MODULE_big = pg_testgen
OBJS = \        # 所有.c文件生产的同名.o目标文件
	$(WIN32RES) \
	pg_testgen.o
PGFILEDESC = "pg_testgen - test data generator"

PG_CPPFLAGS = -I$(libpq_srcdir)
SHLIB_LINK_INTERNAL = $(libpq)

EXTENSION = pg_testgen  # 插件名
DATA = pg_testgen--1.0.sql  # 插件的sql文件

REGRESS = pg_testgen # sql、expected 文件夹下的测试sql文件名

ifdef USE_PGXS
PG_CONFIG = pg_config
PGXS := $(shell $(PG_CONFIG) --pgxs)
include $(PGXS)
else
SHLIB_PREREQS = submake-libpq
subdir = contrib/pg_testgen
top_builddir = ../..
include $(top_builddir)/src/Makefile.global
include $(top_srcdir)/contrib/contrib-global.mk
endif
```

`pg_testgen--1.0.sql`是插件的初始化sql文件，1.0是版本号，如果需要版本更新到1.1，则需要`pg_testgen--1.0--1.1.sql`文件。文件的内容如下：

```sql
\echo Use "CREATE EXTENSION pg_testgen" to load this file. \quit

CREATE FUNCTION rand_int(integer, integer)
    RETURNS integer
    AS 'MODULE_PATHNAME', 'rand_int'
    LANGUAGE C STRICT;
......
......
```
该文件必须在psql中通过`CREATE EXTENSION pg_testgen`加载，否则会报第一行的错误。下面的语句是PL/pgSQL语句，支持如建表、插入、创建函数等PostgreSQL操作。我们希望使用C语言进行数据库内核的插件开发，所以这里需要创建函数并关联到对应的C语言函数。这里的`rand_int(integer, integer)`是有两个整数作为参数函数，返回值是整型，加载插件后使用`rand_int(a, b)`即可使用。PL/pgSQL语句定义的函数和C语言函数一般同名。

`pg_testgen.c`文件是插件开发的主要文件，首先需要引入两个必须的头文件，并声明`PG_MODULE_MAGIC`确保不会错误加载共享库文件：
```c
#include "postgres.h"
#include "fmgr.h"

PG_MODULE_MAGIC;
```

对于在`pg_testgen--1.0.sql`中定义的C语言函数，需要在该文件中声明并实现：
```c
PG_FUNCTION_INFO_V1(rand_int);
Datum rand_int(PG_FUNCTION_ARGS){
    int32 ret = 123;
    PG_RETURN_INT32(ret);
}
```
注意每个需要在psql中调用的函数都需要用`PG_FUNCTION_INFO_V1(rand_int);`来注册，而只在扩展内部调用的函数则不需要，且最好声明为`static`函数。

对于参数，由于是两个int类型，可以通过下面的宏定义来获取：
```c
int32 arg1 = PG_GETARG_INT32(0);
int32 arg2 = PG_GETARG_INT32(1);
```
注意这里不能判断从psql中传入的参数个数，因为参数个数是可变的，对于C语言插件，函数名是唯一标识，不能支持C语言的函数重载。但PL/pgSQL定义的函数可以重载，如果需要重载函数，需要要在`pg_testgen--1.0.sql`中定义多个同名函数，这些函数都会调用同一个C函数，参数个数保存在`fcinfo->nargs`中，可以通过以下语句判断从psql中调用对应C函数的参数是否正确：
```c
Assert(fcinfo->nargs == 0 || fcinfo->nargs == 2);
```
这个判断表示该C函数会被两个`pg_testgen--1.0.sql`中定义的函数调用，参数个数分别为0和2。如果psql中调用函数时参数个数错误，会直接报错、不会往下执行。

对于不同的参数类型，需要用不同的宏定义来获取，可以在`"fmgr.h"`头文件中查看：
```c
PG_GETARG_INT32(0);
PG_GETARG_CHAR(1);
PG_GETARG_CSTRING(2);
PG_GETARG_FLOAT8(3);
......
```

对于不同的返回类型也需要用`"fmgr.h"`中的不同的宏定义：
```c
PG_RETURN_INT32(x);
PG_RETURN_TEXT_P(x);
PG_RETURN_FLOAT8(x);
PG_RETURN_BOOL(x);
......
```

下面是一个完整的例子，一个重载了两个PL/pgSQL函数的C函数，如果没有参数则返回一个随机整数，如果有两个参数[a, b]则返回一个在[a, b]间的随机整数：
```c
static inline int32 rand_int_internal(int min, int max){
    return rand() % (max - min + 1) + min;
}

PG_FUNCTION_INFO_V1(rand_int);
Datum rand_int(PG_FUNCTION_ARGS){
    Assert(fcinfo->nargs == 0 || fcinfo->nargs == 2);
    int32 min = fcinfo->nargs == 2 ? PG_GETARG_INT32(0) : 0;
    int32 max = fcinfo->nargs == 2 ? PG_GETARG_INT32(1) : INT32_MAX;
    PG_RETURN_INT32(rand_int_internal(min, max));
}
```

如果我们需要获取、返回一个`text`字符串类型，需要使用PostgreSQL提供的`text`结构体来封装，在分配内存时需要加上一个头部大小`CARHDRSZ`，`VARDATA(t)`表示t的数据部分起始地址，`VARDATA_ANY(t)`可以取到text的数据部分或者一个普通`Datum`的数据部分，`VARSIZE_ANY_EXHDR(t)`表示`text`除去头部的数据长度：
```c
// 一个大小为size的text，分配内存时需要加上头部大小
text *t = (text *)palloc(VARHDRSZ + size);
SET_VARSIZE(t, VARHDRSZ + size);

// 将数据拷贝到t的数据部分
memcpy((void *) VARDATA(t), /* destination */
           (void *) VARDATA_ANY(src), /* source */
           VARSIZE_ANY_EXHDR(src));   /* how many bytes */

PG_RETURN_TEXT_P(t);  // 返回改text的指针
```

如果需要返回多行数据，需要使用SRF(set-returning functions)，在`pg_testgen--1.0.sql`定义返回值时需要机上SETOF关键字：
```sql
CREATE FUNCTION rows_int(integer)
    RETURNS SETOF integer
    AS 'MODULE_PATHNAME', 'rows_int'
    LANGUAGE C STRICT;

CREATE FUNCTION rows_int(integer, integer, integer)
    RETURNS SETOF integer
    AS 'MODULE_PATHNAME', 'rows_int'
    LANGUAGE C STRICT;
```
在.c文件中，需要包含SRF头文件：
```c
#include "funcapi.h"
```
SRF函数是递归调用的，和普通函数使用方法相同，只是需要套进SRF模版，以下是一个很简单的例子，可以当作模版使用，该函数接收一个参数r或者三个参数[r, a, b]。如果只有一个参数，返回r行随机整数；如果是三个参数，则返回r行[a, b]间的整数。

```c
static inline int32 rand_int_internal(int min, int max){
    return rand() % (max - min + 1) + min;
}

PG_FUNCTION_INFO_V1(rows_int);
Datum
rows_int(PG_FUNCTION_ARGS)
{
    Assert(fcinfo->nargs == 1 || fcinfo->nargs == 3);
    
    FuncCallContext     *funcctx;
    int32 times = PG_GETARG_INT32(0);

    // 第一次调用SRF函数时的初始化
    if (SRF_IS_FIRSTCALL())
    {
        MemoryContext   oldcontext;

        /* create a function context for cross-call persistence */
        funcctx = SRF_FIRSTCALL_INIT();

        /* switch to memory context appropriate for multiple function calls */
        oldcontext = MemoryContextSwitchTo(funcctx->multi_call_memory_ctx);

       // 需要返回tuple的行数
        funcctx->max_calls = times;

        MemoryContextSwitchTo(oldcontext);
    }

    // 每次调用SRF函数都需要的设置
    funcctx = SRF_PERCALL_SETUP();

    if (funcctx->call_cntr < funcctx->max_calls)    // 还需要返回更多行
    {
        int32 min = fcinfo->nargs == 3 ? PG_GETARG_INT32(1) : 0;
        int32 max = fcinfo->nargs == 3 ? PG_GETARG_INT32(2) : INT32_MAX;
        SRF_RETURN_NEXT(funcctx, Int32GetDatum(rand_int_internal(min, max)));   // 返回一行
    }
    else    // 已经返回足够行
    {
        SRF_RETURN_DONE(funcctx);       // SRF函数结束
    }
}
```
`funcctx->max_calls`是需要调用SRF函数的次数，即返回的行数。

为了方便对插件进行功能测试，可以创建`sql`和`expected`目录，`sql`目录下的`pg_testgen.sql`是Makefile里`REGRESS`字段指明的测试sql，`expected`目录下同名的`pg_testgen.out`是对应测试sql的期望输出，使用`make check`可以很方便的进行测试。测试会生成`log`目录存放数据库的日志，生成`results`目录存放实际的输出。

开发插件完成后，将插件目录放在`contrib`目录下，比如`contrib/pg_testgen`，进入插件目录，使用`make`命令编译，然后`make install`安装插件。如果成功，用psql连接数据库，用`CREATE EXTENSION pg_testgen;`加载插件，即可使用。

一个最简单的、完整的插件源代码：[pg_testgen](https://github.com/yuesong-feng/pg_testgen)
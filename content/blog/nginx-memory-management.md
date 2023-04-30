+++
title = "Nginx Memory Management"
date = 2020-04-11
+++


## 1. Linux 的内存管理
```
+---------+  init system   +--------------+  page frame   +------------------+
| Bootmem | -------------> |              | ------------> |                  |
+---------+                |              |               |                  |
                           |              |               |                  |
                           |              |               |       Slab       | -------------------------------------------------------------------------+
                           |              |               |                  |                                                                          |
                           |              |               |                  |                                                                          |
                           |              |               |                  | --------------------------------------------------------------------+    |
                           |              |               +------------------+                                                                     |    |
                           |              |                 |                                                                                      |    |
                           |              |                 | small objects                                                                        |    |
                           |              |                 |                                                                                      |    |
                           | Buddy System |                 |                                        +- - - - - - - - - - - - - - - - - - - - - +  |    |
                           |              |                 |                                        ' userspace:                               '  |    |
                           |              |                 v                                        '                                          '  |    |
                           |              |  page frame   +------------------+  syscall (brk/mmap)   ' +----------+  malloc()   +-------------+ '  |    |
                           |              | ------------> |                  | --------------------> ' |  malloc  | ----------> | application | '  |    |
                           |              |               | Memory Mangement |                       ' +----------+             +-------------+ '  |    |
                           |              |               |                  |                       '                                          '  |    |
                           |              |               +------------------+                       +- - - - - - - - - - - - - - - - - - - - - +  |    |
                           |              |  page frame   +------------------+  small objects                                                      |    |
                           |              | ------------> |  Device Drivers  | <-------------------------------------------------------------------+    |
                           |              |               +------------------+                                                                          |
                           |              |  page frame   +------------------+  small objects                                                           |
                           |              | ------------> |   File System    | <------------------------------------------------------------------------+
                           +--------------+               +------------------+

```

在上图里，内核在启动的时候会加载 Bootmem 内存管理系统（这里使用 bitmap 索引内存并分配。然后等初始化到 Buddy System 之后，将未分配的内存全部交给 Buddy System, Buddy System 在 Linux 内核中是做页的分配 (一般来说，页的大小为4k)。再然后到了 Slab 内存管理系统，Slab 主要用于小内存的分配和释放，也就是小于一个页面大小的内存。这里 Slab 对内存的分配和释放起到了 Cache 的作用，在 Linux 内核中地位应该是内存管理模块的老大了。Slab 和 Buddy System 初始化完成之后，内核的内存管理系统基本上已经初始化完成了，这个时候我们其他的内核模块（图中的 Device Drivers 和 File System）申请释放内存就可以直接进行了。要了解更多关于 Slab 和 Buddy System 的信息，可以查看 procfs 中的 `/proc/buddyinfo` 和 `/proc/slabinfo`。

在用户态层面，用户态程序在执行 malloc 的函数调用的时候，会直接在 libc 函数库里面的 ptmalloc 里面执行 brk()/mmap() 系统调用进行虚拟内存的分配，这两个系统调用在内核的层面依然是在和 Slab/Buddy System 进行交互，当然，这里只是对虚拟内存的一些结构体进行初始化，并没有真正的分配到内存。其实这里说到的 ptmalloc 在用户态层面也是一种类似于 Slab 的内存分配 Cache 系统（类似 ptmalloc 的用户态 memory allocator 有 mimalloc tcmalloc jemalloc,我们可以视不同的场景而选用这些 malloc 实现）。真正触发物理内存的分配是在用户态程序在获得虚拟内存然后对该内存写之后，触发了缺页中断，这个时候在内核的中断处理过程中就会分配并映射物理内存到虚拟地址上，这个时候也同样是在 Buddy System 当中进行 malloc 页面。

## 2. Nginx 内存管理源码分析
### 1. ngx_alloc.c(h)
这里主要是 Nginx 的基础内存分配的代码，包装 libc 里面的内存分配函数，至于为什么需要包装呢？这是因为 Nginx 是跨平台的，可不止有 Linux 这一个平台上能跑 Nginx。

在 Linux 环境下手动展开了宏核心代码是这个样子：
```c
void *
ngx_alloc(size_t size, ngx_log_t *log)
{
    void  *p;

    p = malloc(size);
    if (p == NULL) {
        ngx_log_error(NGX_LOG_EMERG, log, ngx_errno,
                      "malloc(%uz) failed", size);
    }

    ngx_log_debug2(NGX_LOG_DEBUG_ALLOC, log, 0, "malloc: %p:%uz", p, size);

    return p;
}


void *
ngx_calloc(size_t size, ngx_log_t *log)
{
    void  *p;

    p = ngx_alloc(size, log);

    if (p) {
        ngx_memzero(p, size);
    }

    return p;
}

void *
ngx_memalign(size_t alignment, size_t size, ngx_log_t *log)
{
    void  *p;
    int    err;

    err = posix_memalign(&p, alignment, size);

    if (err) {
        ngx_log_error(NGX_LOG_EMERG, log, err,
                      "posix_memalign(%uz, %uz) failed", alignment, size);
        p = NULL;
    }

    ngx_log_debug3(NGX_LOG_DEBUG_ALLOC, log, 0,
                   "posix_memalign: %p:%uz @%uz", p, size, alignment);

    return p;
}
```
我们可以看到 Nginx 的 ngx_alloc() ngx_calloc() ngx_memalign() 三个函数只是对 libc malloc 函数的简单封装，也就是说仍然是在调用这些函数。我认为在这里还是有一个比较粗暴的点，就是在分配 ngx_calloc() 的时候我们是暴力的分配内存然后在将内存初始化为 0, 但是在 libc的实现里面 `calloc ！= malloc + memzero`,可能在 Nginx 的场景下, 使用这种实现和带 libc 带惰性初始化的实现没什么具体的性能差距。Nginx 的内存分配都是比较小块的，具体的信息可以看 stackoverflow 的这个问题 : 
[why-mallocmemset-is-slower-than-calloc](https://stackoverflow.com/questions/2688466/why-mallocmemset-is-slower-than-calloc)

### 2.ngx_shmem.c(h)
这里主要是 Nginx 底层对共享内存的创建与销毁相关实现。

在 Linux 环境下手动展开了宏核心代码是这个样子：
```c
typedef struct {
    u_char      *addr;
    size_t       size;
    ngx_str_t    name;
    ngx_log_t   *log;
    ngx_uint_t   exists;   /* unsigned  exists:1;  */
} ngx_shm_t;

ngx_int_t
ngx_shm_alloc(ngx_shm_t *shm)
{
    shm->addr = (u_char *) mmap(NULL, shm->size,
                                PROT_READ|PROT_WRITE,
                                MAP_ANON|MAP_SHARED, -1, 0);

    if (shm->addr == MAP_FAILED) {
        ngx_log_error(NGX_LOG_ALERT, shm->log, ngx_errno,
                      "mmap(MAP_ANON|MAP_SHARED, %uz) failed", shm->size);
        return NGX_ERROR;
    }

    return NGX_OK;
}


void
ngx_shm_free(ngx_shm_t *shm)
{
    if (munmap((void *) shm->addr, shm->size) == -1) {
        ngx_log_error(NGX_LOG_ALERT, shm->log, ngx_errno,
                      "munmap(%p, %uz) failed", shm->addr, shm->size);
    }
}
```
在现代 Linux 环境下默认使用的是 mmap 匿名映射来实现进程间的共享内存，至于为什么没有使用 shm_open 这种方式来实现， 因为 Nginx 是在 master fork to worker 这种形式来进行的，我们不需要在毫无关联的两个进程上进行通信，所以采用 mmap 匿名映射是最直接的方法。当然为了跨平台还有其他的宏定义会采用 `/dev/zero` 和 `System V` IPC 的方式，也是很有效的方法。

### 3.ngx_palloc.c(h)
接下来就是 Nginx 引以为傲的内存池上场了。Nginx中大多数的内存分配都是在内存池中完成的，在内存池中分配的内存在内存池被销毁时会被自动的释放。这就使得可以获得很高的内存分配性能，并使得内存控制更加简单。

直接看我加了注释的 `ngx_palloc.h`
```c
typedef void (*ngx_pool_cleanup_pt)(void *data);

typedef struct ngx_pool_cleanup_s  ngx_pool_cleanup_t;

// 释放内存池拥有的资源，也就是内存池在释放的时候会选择同步释放的资源，一般用于缓存文件的释放
struct ngx_pool_cleanup_s {
    ngx_pool_cleanup_pt   handler;
    void                 *data;
    ngx_pool_cleanup_t   *next;
};


typedef struct ngx_pool_large_s  ngx_pool_large_t;
/*
 * 内存池中的大内存，ngx_pool_large_s 是一个链表结构，结构本身存储在内存池中，
 * 指针指向操作系统 malloc 出来的对应大小内存
 */
struct ngx_pool_large_s {
    ngx_pool_large_t     *next;
    void                 *alloc;
};

// 内存池中的主要结构， ngx_pool_data_t 对应内存池的 Metadata，描述内存池的分配状态信息
typedef struct {
    u_char               *last;
    u_char               *end;
    ngx_pool_t           *next;
    ngx_uint_t            failed;
} ngx_pool_data_t;

/*
 * ngx_pool_s 是我们需要的内存池结构，其中包含了大内存和小内存，以及外挂在内存池生命周期的
 * 缓存文件结构
 */
struct ngx_pool_s {
    ngx_pool_data_t       d;
    size_t                max;
    ngx_pool_t           *current;
    ngx_chain_t          *chain;
    ngx_pool_large_t     *large;
    ngx_pool_cleanup_t   *cleanup;
    ngx_log_t            *log;
};


typedef struct {
    ngx_fd_t              fd;
    u_char               *name;
    ngx_log_t            *log;
} ngx_pool_cleanup_file_t;

// 创建内存池
ngx_pool_t *ngx_create_pool(size_t size, ngx_log_t *log);
// 销毁内存池，内存池中的大内存和小内存链表都会被释放
void ngx_destroy_pool(ngx_pool_t *pool);
// 重置内存池，将内存池中的数据结构初始化为开始的状态，并释放那些已经分配的大内存
void ngx_reset_pool(ngx_pool_t *pool);

// 从内存池中分配内存,根据大小来判断分配方式：大内存或者小内存
void *ngx_palloc(ngx_pool_t *pool, size_t size);
// 从内存池中分配内存，且不对其进行内存对齐
void *ngx_pnalloc(ngx_pool_t *pool, size_t size);
// 从内存池中分配内存，并将分配到的内存初始化为0
void *ngx_pcalloc(ngx_pool_t *pool, size_t size);
/*
 * 按照 alignment 对齐大小来分配内存，该内存无论大小都会在操作系统中进行 malloc，
 * 分配完外挂到大内存链表上 
 */
void *ngx_pmemalign(ngx_pool_t *pool, size_t size, size_t alignment);
// 提前释放大块内存
ngx_int_t ngx_pfree(ngx_pool_t *pool, void *p);

// 随着内存池的释放同步释放资源的操作
ngx_pool_cleanup_t *ngx_pool_cleanup_add(ngx_pool_t *p, size_t size);
void ngx_pool_run_cleanup_file(ngx_pool_t *p, ngx_fd_t fd);
void ngx_pool_cleanup_file(void *data);
void ngx_pool_delete_file(void *data);
```
直接画出内存池的结构图
```
+--------------------------------------------------------+---------------------------+
|       ngx_pool_t                                       |        memory block       |
+--------------------+---+-------+-----+-----+-------+-----------------+-------------+
| ngx_pool_data_t *d |   |       |     |     |       |   |             |             |
+--------------------+max|current|chain|large|cleanup|log|     used    |   unused    |
|last|end|next|failed|   |       |     |     |       |   |             |             |
+--+---+---+-------------+---+---+-----+--+--+----+--+---+-------------+-------------+
   |   |   |                 |            |       |                                  ^
   |   |   |                 |            |       |                    ^             |
   |   |   |                 |            |       |                    |             |
   |   +---------------------------------------------------------------+             |
   +---------------------------------------------------------------------------------+
+----------|                 |            |       |
+----------------------------+            |       +---------------------+
v                                         |                             |
+---------------+--------------+          v                             v
|  ngx_pool_t   | memory block |          +----------------+            +----------------------+
+----------+----+--------------+          |ngx_pool_large_t|            |  ngx_pool_cleanup_t  |
           |                              +--------+-------+            +--------+------+------+
+----------+                              | alloc  | next  |            | hander | data | next |
|                                         +--------+---+---+            +--------+------+---+--+
v                                                      |                                    |
+---------------+--------------+          +------------+                +-------------------+
|  ngx_pool_t   | memory block |          |                             |
+---------+-----+--------------+          |                             |
          |                               v                             v
          |                               +----------------+            +----------------------+
          |                               |ngx_pool_large_t|            |  ngx_pool_cleanup_t  |
          v                               +--------+-------+            +--------+------+------+
         NULL                             | alloc  | next  |            | hander | data | next |
                                          +--------+----+--+            +--------+------+---+--+
                                                        |                                   |
                                                        v                                   v
                                                       NULL                                NULL

```
结合这个图应该能理解 Nginx 的内存池的设计。区分大小内存块的主要依据是分配内存大小，在内存池分配的函数实现当中，我们定义了一个宏 `NGX_MAX_ALLOC_FROM_POOL` 来设置内存池的小块内存最大的大小，当然这个大小也由我们分配内存池的`size`（去除掉`ngx_pool_t`本身大小）共同决定，在两者中取最小值。若要求分配的内存大于该最小值，则使用大块内存分配，创建大块内存结构，然后使用操作系统的 `malloc` 进行大块内存分配并将 `alloc` 指针指向该内存地址。若要求分配的内存小于该内存，且该内存池里面的 `unused` 内存大小比 `size` 大，直接在该内存池里分配，并减少 `unused` 的大小。若要求分配的内存小于该内存，且该内存池里面的 `unused` 内存大小比 `size` 小，我们考虑再次开辟一个相同的内存池，并将上一个内存池里面的 `next` 指针指向该内存池，同时上一个内存池里面的 `failed` 变量加一，表示内存分配失败一次，然后我们会再第二个开辟的内存池里面分配我们需要的小内存。`Nginx` 这里还有一个比较粗暴一点的设定，就是在我们同一个内存池中的内存分配失败次数 `failed` 失败次数超过四次以后，该内存池中的 `current` 指针会指向下一块内存池子，这个设定是为了减少内存分配寻找的时间成本，当然，这在某种程度上也会造成内存碎片化的问题。在 Nginx 的内存池中，内存池这个概念并不是单单指一个 `ngx_pool_t` 这个结构，其实上他是指 `ngx_pool_t` 组成的一个链表，为了组成这个链表，我们在新增链表内的内存池的时候，其实上浪费了新增对象的一个 `ngx_pool_t` 的空间(后续新增的对象中某些数据结构的成员为`NULL`)，这个浪费比例于我们内存的大小成反比关系。上面几种情况的取舍问题，其实也是计算机系统在很多设计上的考虑，在很多的子系统、模块当中，对不同的场景就会有不同的最优解。

关于内存池的释放，直接看数据结构图，我们在该内存池的生命周期结束的时候，根据这个类链表结构的设计，我们的 `free` 就很简单了，其实就是跟着链表的指向一个一个的进行 `free`, 在`ngx_pool_cleanup_t` 中，我们同时也可以对内存池上外挂的文件调用设计的 `handler`进行清理。这样就做到了内存池的统一释放，非常契合 Nginx 的 Web Server 场景。

### 3.ngx_slab.c(h)
下面介绍一下 Nginx 中基于 `slab` 的内存分配机制。`slab` 主要用在 Linux 内核的内存管理子系统，主要用于小内存的分配。其工作是针对一些经常分配并释放的对象，如进程描述符等，基本上都是 Linux 内核里面的一些数据结构，如果直接采用伙伴系统来进行分配和释放，不仅会造成大量的内存碎片，而且处理速度也太慢。slab主要是以对象为单位来分配内存，简单来说分配这一类的内存，这类内存会组成一个列表，在内存使用完之后也不会释放到 `buddy system`，依旧是放在列表当中，等待下一次的相同内存分配请求，这样就起到了缓存的效果，直接提升了内存的分配效率，具体的信息可以参考 `Linux Procfs` 中的 `/proc/slabinfo` 信息。
许多场景下，不同的Nginx请求间必须交互后才能执行下去，例如限制一个客户端能够并发访问的请求数。可是Nginx被设计为一个多进程的程序，服务更健壮的另一面就是，Nginx请求可能是分布在不同的进程上的，当进程间需要互相配合才能完成请求的处理时，进程间通信开发困难的特点就会凸显出来。Nginx 的 Slab 内存池就是设计出来解决共享内存的分配问题的。这些共享内存区域通常是通过标准 Nginx 配置指令来创建的，比如 ssl_session_cache、proxy_cache_path、limit_req_zone、limit_conn_zone、和 upstream 的 zone 指令。

直接看我加了注释的 `ngx_slab.h`
```c
typedef struct ngx_slab_page_s  ngx_slab_page_t;

// 页面的描述数据结构
struct ngx_slab_page_s {
    uintptr_t         slab;
    ngx_slab_page_t  *next;
    uintptr_t         prev;
};

// slab 统计数据结构
typedef struct {
    ngx_uint_t        total;
    ngx_uint_t        used;

    ngx_uint_t        reqs;
    ngx_uint_t        fails;
} ngx_slab_stat_t;

// slab 内存池的元数据结构
typedef struct {
    ngx_shmtx_sh_t    lock;

    size_t            min_size;
    size_t            min_shift;

    ngx_slab_page_t  *pages;
    ngx_slab_page_t  *last;
    ngx_slab_page_t   free;

    ngx_slab_stat_t  *stats;
    ngx_uint_t        pfree;

    u_char           *start;
    u_char           *end;

    ngx_shmtx_t       mutex;

    u_char           *log_ctx;
    u_char            zero;

    unsigned          log_nomem:1;

    void             *data;
    void             *addr;
} ngx_slab_pool_t;

// slab 内存池元数据大小的初始化
void ngx_slab_sizes_init(void);
// slab 内存池初始化
void ngx_slab_init(ngx_slab_pool_t *pool);
// 内存分配
void *ngx_slab_alloc(ngx_slab_pool_t *pool, size_t size);
void *ngx_slab_alloc_locked(ngx_slab_pool_t *pool, size_t size);
void *ngx_slab_calloc(ngx_slab_pool_t *pool, size_t size);
void *ngx_slab_calloc_locked(ngx_slab_pool_t *pool, size_t size);
// 内存释放
void ngx_slab_free(ngx_slab_pool_t *pool, void *p);
void ngx_slab_free_locked(ngx_slab_pool_t *pool, void *p);
```

然后画出数据结构图
```
+---------------------+--------------------------------------------------------->+------------------+
|                     |         X                                                |    lock          |
|   ngx_slab_pool_t   |                                                          +------------------+
|                     |                                                          |     min_size     |
+---+----------------------------------------------------------------+           +------------------+
|   | ngx_slab_page_t |Min Slab Page                                 |           |    min_shift     |
|   +-----------------+     +                                        |           +------------------+
| s |                 |     |               +------------------------------------+    pages         |
| l |    + + + + +    |     |               |                        |           +------------------+
| o |    | | | | |    |     |to             |        +---------------------------+    last          |
| t |    v v v v v    |     |               |        |               |           +------------------+
| s |                 |     |               |        |               |           |     free         |
|   +-----------------+     v               |        |               |           +------------------+
|   | ngx_slab_page_t |Max Slab Page     +---------------------------------------+     status       |
+---------------------+  <---------------+  |        |               |           +------------------+
|   | ngx_slab_stat_t |Min Slab Page        |        |               |           |     pfree        |
|   +-----------------+     +               |        |               |           +------------------+
| s |                 |     |               |        |  +-------------------------|    start        |
| t |    + + + + +    |     |               |        |  |            |           +------------------+
| a |    | | | | |    |     |to             |        |  |  +---------------------+     end          |
| t |    v v v v v    |     |               |        |  |  |         |           +------------------+
| s |                 |     |               |        |  |  |         |           |     mutex        |
|   +-----------------+     v               |        |  |  |         |           +------------------+
|   | ngx_slab_stat_t |Max Slab Page        |        |  |  |         |           |     log_ctx      |
+---------------------+   <-----------------+        |  |  |         |           +------------------+
|   | ngx_slab_page_t +--+                           |  |  |         |           |     zero         |
|   +-----------------+  |                           |  |  |         |           +------------------+
| p |                 |  |                           |  |  |         |           |     log_nomen    |
| a |    + + + + +    |  |                           |  |  |         |           +------------------+
| g |    | | | | |    |  | pages number slab_page_t  |  |  |         |           |     data         |
| e |    v v v v v    |  |                           |  |  |         |           +------------------+
| s |                 |  |                           |  |  |         |           |     addr         |
|   +-----------------+  |                           |  |  |         +---------->+------------------+
|   | ngx_slab_page_t +--+                           |  |  |
+---+-----------------+    <-------------------------+  |  |
|  alignment memory   |                                 |  |
+------------------------+  <---------------------------+  |
|                     |  |                                 |
|                     |  |                                 |
|                     |  |                                 |
|                     |  |                                 |
|     used memory     |  |                                 |
|                     |  |pages number real page           |
|                     |  |                                 |
|                     |  |                                 |
|                     |  |                                 |
|                     |  |                                 |
|                     |  |                                 |
+------------------------+                                 |
|     unused memory   |                                    |
+---------------------+ <----------------------------------+

```
分析一下数据结构吧，这几个主要得结构都是紧紧埃在一起的，主要分为两部分，一部分是元数据区，实际内存分配区域。实际内存分配区域就是途中 `used memory` 和 `unused memory`一起的区域，元数据区域有四大块： `ngx_slab_pool_t` （内存池的结构），`ngx_slab_page_t` 组成的 `slots` 区域，该区域主要描述 `slab` 页面链表的头，大小从 8bit 的 Slab page 到 2048 Bit 的 Slab page, `ngx_slab_stat_t` 组成的 `stat` 区域，该区域是上面 Slab page 的统计信息，记录的是 上面 `slots` 里面的每个 Slab page 分配情况。`ngx_slab_page_t` 组成的真实内存区域的元信息，意义对应下面 `used memory` 里面的真实内存页面。
在初始状态，所有的内存都挂在了 `ngx_slab_pool_t` 下面的 `free` 下等待进行分配。分配的大致思路就是把每个实际的内存页面都进行对应的 Slab size (也就是 slots的位置对应的位数) 8Bit 16Bit 256Bit..... 这类的内存页面进行切分内存块，然后在 `ngx_slab_page_t`数据结构中的 slab 成员上建立 `bitmap` （如果实在放不下去就会到相应的实际内存页的前几个chunk建立bitmap）标记该页面内存的分配情况，同时用链表把对应的内存块连接起来，这样就可以快速的对半满的内存块进行内存分配 。在内存回收的情况下，只用在 `bitmap` 里面把相应的位置为 0,如果判断这个页面已经全部空闲了就会直接脱离该链表被扔进 `free` 链表里面。这样就保证了共享内存的高效分配。
在 Slab 共享内存分配中，为了保证高效的内存分配操作，大量使用了位运算的方式极致的压榨性能，同时，和 Linux 内核的 Slab 内存管理不同， `ngx_slab` 在处理内存分配的时候，分配到了非标准大小内存 (比如 129Bit = 128 Bit + 1Bit) 内存的时候， 会直接分配 256 Bit 的内存页面，这样牺牲了内存空间的效率，换来了时间上的高效利用，同时也尽量的减少了内存的碎片化。
Nginx 的共享内存使用加强版的 Slab allocator (至少从上面的概念中严格定义的 话) 进行管理。它将 slab allocator 和 buddy memory system 里的分级相结合，基本上实现了一个使用自定义内存空间 (Nginx 的共享内存) 的通用内存管理器。

## 3.  使用 ebpf 对 Nginx 内存管理的分析
todo

## 4.  Nginx 内存管理相关的插件
Tengine 里有两个内存分配信息相关的插件：`ngx_debug_pool` 和 `ngx_slab_stat`。

### 1. ngx_debug_pool
该模块可以提供 Nginx 内存池占用内存的状态信息。这里 Tengine 对 `ngx_palloc.c(h)` 进行了 patch, 直接在 ngx_pool_t 里面加上了 `stat` 成员，然后又设置了一个 `ngx_pool_stats` 全局变量，这样就可以直接获取 `ngx_pool_t` 内存池的信息了。
从 Tengine 源码中提取模块，进行编译
```shell
./configure --add-module=./modules/ngx_debug_pool
make && make install
``` 

添加进 `nginx.conf` 里
```
location = /debug_pool {
    debug_pool;
}
```
然后我们看见效果
```
$ curl 127.0.0.1/debug_pool
pid:58413
size:        8192 num:           5 cnum:           1 lnum:           0 ngx_http_alloc_request
size:      438830 num:           2 cnum:           1 lnum:          18 ngx_init_cycle
size:           0 num:           1 cnum:           0 lnum:           0 main
size:        1536 num:           5 cnum:           1 lnum:           9 ngx_event_accept
size:       438KB num:          13 cnum:           3 lnum:          27 [SUMMARY]
```
除了最后一行的每一行的输出内容都有相同的格式，如下：

```
"__size__: %12u __num__: %12u __cnum__: %12u __lnum__: %12u __\<function name\>__"
```

`size`: 当前内存池占用的内存
`num`: 内存池创建的个数（包括当前正在使用的内存池数量和已经被释放的内存池数量）
`cnum`: 当前正在使用的内存池数量
`lnum`: 该类内存池调用ngx_palloc_large()次数
`funcion name`: 创建该内存池的nginx C函数的函数名

通过创建该内存池的函数的函数名，我们可以知道各个模块的内存使用情况
例如：  
> `ngx_http_create_request`创建的内存池用于 HTTP 请求。因为大多数模块直接从该内存池上分配内存，所以很难区分具体哪个模块使用了内存。
> `ngx_event_accept`创建的内存池用于 TCP 连接。
> `ngx_init_cycle`创建的内存池用于解析 Nginx 的配置和保存其他全局数据结构。`ngx_http_lua_init_worker`用于指令`init_worker_by_lua`。

### 2. ngx_slab_stat
该模块可以提供 Slab 共享内存的分配信息，其实在 Nginx 官方源码里甚至为了这个需求改进了统计代码，但是统计功能的模块 [ngx_http_api_module](http://nginx.org/en/docs/http/ngx_http_api_module.html) 只有 Nginx Plus 商业版本才会有。 所以现在采用 Tengine 里面的模块进行统计。
注意： 该模块必须在NGINX版本高于 1.13.x上使用，如果需要在低版本上使用需要另外进行 patch,主要是因为 `ngx_slab` 在这个版本上有一个关于 `ngx_slab_pool_t` 中 `stat: ngx_slab_stat_t` 成员的变更，有了这个成员我们才能快乐的做 slab 内存分配统计。 
```
./configure --add-module=/path/to/ngx_slab_stat
make && make install
```


添加进 `nginx.conf` 里
```
limit_req_zone $binary_remote_addr zone=ttlsa_com:10m rate=1r/s;
location / {
    root   /usr/share/nginx/html;
    index  index.html index.htm;
    limit_req zone=ttlsa_com burst=5;
}

location /slab_stat 
{                                    
    slab_stat;                                                             
}
```
这里我用 `limit_req_zone` 来创建共享内存，这样好让我们能看到结果
```
╰─$ curl 127.0.0.1/slab_stat
* shared memory: ttlsa_com
total:       10240(KB) free:       10168(KB) size:           4(KB)
pages:       10168(KB) start:00007F649CFDF000 end:00007F649D9CF000
slot:           8(Bytes) total:           0 used:           0 reqs:           0 fails:           0
slot:          16(Bytes) total:           0 used:           0 reqs:           0 fails:           0
slot:          32(Bytes) total:         127 used:           1 reqs:           1 fails:           0
slot:          64(Bytes) total:           0 used:           0 reqs:           0 fails:           0
slot:         128(Bytes) total:          32 used:           2 reqs:           2 fails:           0
slot:         256(Bytes) total:           0 used:           0 reqs:           0 fails:           0
slot:         512(Bytes) total:           0 used:           0 reqs:           0 fails:           0
slot:        1024(Bytes) total:           0 used:           0 reqs:           0 fails:           0
slot:        2048(Bytes) total:           0 used:           0 reqs:           0 fails:           0
```
数据非常清晰，我就不需要过多解释了

## 5.  Nginx 内存管理的优化
Nginx 在内存管理上的优化和其他的用户态程序一样，在 `malloc` 的不同实现上的有性能优化的点。这是一个很大的议题了，中文圈里有很多文章让使用 `tcmalloc` `jemalloc` `mimmalloc` 这种来加速内存分配(其实在 Nginx 官方代码里甚至有支持 `gperftools` 也就是 `tcmalloc`)，但是就我个人认为，理论上 `ptmalloc` 是符合 Nginx 内存分配周期短这种场景的，在多线程环境上的优化的优势在 Nginx 也不是很大,更何况 nginx 自己的内存分配器也在一定程度上减小了内存碎片化。可能这个我会设计一个 `benchmark` 场景来对比各个 `malloc` 在 Nginx 上的效果 (todolist）。

当然也还有一些内核层面上的优化策略， 比如尽量避免 NUMA 计算机架构体系带来的跨 node 级别内存访问带来的高延迟，稳妥的建议是避免使用 NUMA 多 node 架构的服务器，当然，你自己去优化用户态的内存分配也是可以的。 还有一个就是 Linux 内核的透明大页的特性, 可以针对 Nginx 的使用场景，将 `transparent_hugepage` 的选项调为 `madvise` ，根据内存分配的建议来对已分配内存进行透明大页转换。

### 参考链接
1. [core/ngx_slab.c源文件分析](https://ivanzz1001.github.io/records/post/nginx/2018/12/07/nginx-source_part61_2)
2. [深入理解Nginx（第2版）](https://book.douban.com/subject/26745255/)
3. [20 years of Linux Virtual Memory](https://www.slideshare.net/ennael/kernel-recipes-2017-20-years-of-linux-virtual-memory-andrea-arcangeli)
4. [Optimizing web servers for high throughput and low latency](https://dropbox.tech/infrastructure/optimizing-web-servers-for-high-throughput-and-low-latency)
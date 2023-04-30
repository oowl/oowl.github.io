+++
title = "Nginx Network IO Module"
date = 2023-03-26
+++


在网上读过很多Nginx有关的关于网络IO模型的文章，总是感觉很凌乱，再加上最近一位老友密集追问下，重读Nginx源码，才知道自己和中文网络上的文章理解错误了很多东西。

先说 Nginx 的实现吧，这里只讨论 Linux 环境下的 epoll 网络模型。 先讨论一下 Nginx 是怎么监听并且处理连接的，在Nginx的master进程启动阶段读取所有配置，然后 bind + listen 相应的 socket端口，然后 fork 出来 n 个 worker 子进程，这些子进程会在 epoll 里面注册好这个 fork 出来继承的 listen fd。然后每个 worker 进程都会有一个 event loop 不停的去 epoll_wait 等待新的连接事件，如果是 listen fd 被 epoll_wait 到了，这个时候会尝试 accept 这个 fd 来获得新连接。和上述处理模型有关的有几个相关 Nginx 配置。下面的列表顺序就是时间线。

1. **`accept_mutex`** 这个是最老的配置，Nginx 的上述的这套模式，很多 worker 进程的 epoll_wait 同一个listen fd, 这个时候会产生一个经典的“惊群”效应，也就是来了一个新连接的话会唤醒所有的worker的 epoll_wait, 这个时候其实是只有一个 worker 会真正需要处理连接，其他的进程是“一场空”的，这个时候如果连接和worker的数目都很多的话，会导致很多不必要的上下文切换，所以Nginx给出了一个 **`accept_mutex`** 选项，在 1.11.3 版本之前，这个选项默认是给开的，逻辑是，开了这个选项的话， Nginx会在多进程内部有个用原子指令实现的多进程的锁，来保证 调用 listen fd 在一个时刻只能被一个worker epoll_wait，这样就能保证只会有一个进程被唤醒，避免了“惊群”问题。
    
    ```c
    ngx_int_t
    ngx_trylock_accept_mutex(ngx_cycle_t *cycle)
    {
        if (ngx_shmtx_trylock(&ngx_accept_mutex)) {
    
            ngx_log_debug0(NGX_LOG_DEBUG_EVENT, cycle->log, 0,
                           "accept mutex locked");
    
            if (ngx_accept_mutex_held && ngx_accept_events == 0) {
                return NGX_OK;
            }
    				// add listen fd to epoll
            if (ngx_enable_accept_events(cycle) == NGX_ERROR) {
                ngx_shmtx_unlock(&ngx_accept_mutex);
                return NGX_ERROR;
            }
    
            ngx_accept_events = 0;
            ngx_accept_mutex_held = 1;
    
            return NGX_OK;
        }
    
        ngx_log_debug1(NGX_LOG_DEBUG_EVENT, cycle->log, 0,
                       "accept mutex lock failed: %ui", ngx_accept_mutex_held);
    
        if (ngx_accept_mutex_held) {
    				// delete listen fd to epoll
            if (ngx_disable_accept_events(cycle, 0) == NGX_ERROR) {
                return NGX_ERROR;
            }
    
            ngx_accept_mutex_held = 0;
        }
    
        return NGX_OK;
    }
    ```
    
2. `````````**reuseport**````````` 时间来到 Linux 3.9 时代，内核引入了全新的一个 socket 选项，这个选项开启了之后，很多个进程可以同时监听一个端口，新连接进来的时候，内核根据“五元组”来hash出来一个连接来具体分配给某个进程，这样其实就也解决了惊群问题了。这里 Nginx 有个非常不常规的实现，一般来说使用 reuseport 的话会在每个进程内自己进行 bind + listen 动作，但是Nginx这里在master进程里一次性监听了 n 个相同端口的fd， 然后所有的 worker 分别 epoll_wait 自己对应的 fd, 所以开了reuseport 之后的 Nginx的 socket状态会看到很多个 master 进程 listen 同一个端口。这个选项在 Nginx 1.9.1 版本中引入，但是并没有成为默认配置。
    
    ```c
    ngx_int_t
    ngx_clone_listening(ngx_cycle_t *cycle, ngx_listening_t *ls)
    {
    #if (NGX_HAVE_REUSEPORT)
    
        ngx_int_t         n;
        ngx_core_conf_t  *ccf;
        ngx_listening_t   ols;
    
        if (!ls->reuseport || ls->worker != 0) {
            return NGX_OK;
        }
    
        ols = *ls;
    
        ccf = (ngx_core_conf_t *) ngx_get_conf(cycle->conf_ctx, ngx_core_module);
    
        for (n = 1; n < ccf->worker_processes; n++) {
    
            /* create a socket for each worker process */
    
            ls = ngx_array_push(&cycle->listening);
            if (ls == NULL) {
                return NGX_ERROR;
            }
    
            *ls = ols;
            ls->worker = n;
        }
    
    #endif
    
        return NGX_OK;
    }
    
    static ngx_int_t
    ngx_event_process_init(ngx_cycle_t *cycle)
    {
    		...
    		...
        /* for each listening socket */
    
        ls = cycle->listening.elts;
        for (i = 0; i < cycle->listening.nelts; i++) {
    
    #if (NGX_HAVE_REUSEPORT)
            if (ls[i].reuseport && ls[i].worker != ngx_worker) {
    						// jump to another worker listen fd
                continue;
            }
    #endif
    
            c = ngx_get_connection(ls[i].fd, cycle->log);
    
    	      ...
    
    #if (NGX_HAVE_REUSEPORT)
    
            if (ls[i].reuseport) {
    						// add listen fd read event to epoll
                if (ngx_add_event(rev, NGX_READ_EVENT, 0) == NGX_ERROR) {
                    return NGX_ERROR;
                }
    
                continue;
            }
    
    #endif
    
            if (ngx_use_accept_mutex) {
                continue;
            }
    		...
    		}
    
        return NGX_OK;
    }
    ```
    
3.  `EPOLLEXCLUSIVE` Linux 4.5 epoll 引入了这个参数，我觉得这个参数的引入才是真正的在尝试解决“惊群”问题（reuseport主要目的是为了多进程能够监听同一个端口，然后恰好解决了“惊群”问题）。这个选项告诉内核，内核在收到连接的时候，不要唤醒所有的监听进程，只唤醒一个。它降低了多个进程/线程通过epoll_ctl 添加共享fd 引发的惊群概率，使得一个事件发生时，只唤醒一个正在epoll_wait 阻塞等待唤醒的进程/线程（而不是全部唤醒）。这个选项在Nginx 1.11.3 引入，在这个版本之前Nginx的配置是默认开启了 accept_mutex 配置，在这之后就关闭了 accept_mutex 默认配置，因为默认有了 `EPOLLEXCLUSIVE` 方案来避免“惊群”问题。

所以网上有很多的文章说 Nginx默认开启了 reuseport 来避免“惊群”问题，这个理解其实不对的，reuseport 这个选项从来都没有在Nginx中默认打开过，其实也不需要打开。

相关链接

- [https://idea.popcount.org/2017-02-20-epoll-is-fundamentally-broken-12/](https://idea.popcount.org/2017-02-20-epoll-is-fundamentally-broken-12/)
- [https://idea.popcount.org/2017-03-20-epoll-is-fundamentally-broken-22/](https://idea.popcount.org/2017-03-20-epoll-is-fundamentally-broken-22/)
- [https://lwn.net/Articles/542629/](https://lwn.net/Articles/542629/)
- [https://lwn.net/Articles/667087/](https://lwn.net/Articles/667087/)
- [https://lpc.events/event/11/contributions/946/attachments/783/1472/Socket_migration_for_SO_REUSEPORT.pdf](https://lpc.events/event/11/contributions/946/attachments/783/1472/Socket_migration_for_SO_REUSEPORT.pdf)

## TODO

学习多进程的通信。

## 数据结构

查看`src/core`目录，关于共享内存数据结构有（借助chatGPT）：

相关的文件有：
* src/os/unix/ngx_shmem.h
* src/os/unix/ngx_shmem.c
* src/core/ngx_buf.c

`ngx_cycle_t`包含了运行时各种信息，比如配置连等。

## Worker进程创建

相关文件在：
* src/os/unix/ngx_process.c:ngx_spawn_process
* src/os/unix/ngx_process_cycle.c
        ngx_start_worker_processes
        ngx_master_process_cycle
        ngx_worker_process_cycle // 主要逻辑
* src/core/nginx.c:main

main() -> ngx_master_process_cycle -> ngx_start_worker_processes

Worker 进程的主要逻辑位于 ngx_worker_process_cycle 函数中。负责处理客户端请求、执行反向代理、负载均衡、处理静态文件、日志记录等核心功能。

比如，事件处理：一旦有事件触发，Worker 进程会根据事件的类型执行相应的处理逻辑。例如，如果是客户端的连接请求，Worker 进程会接受连接并处理客户端的请求。如果是上游服务器的响应，Worker 进程会将响应发送回客户端。

需要关注：src/event/ngx_event.c和src/http/ngx_http_request.c。

## 核心逻辑

master进程的主逻辑在ngx_master_process_cycle。worker进程的主逻辑在ngx_worker_process_cycle。

```
// 这是main函数末尾，多进程模式为master/worker
if (ngx_process == NGX_PROCESS_MASTER) {
	// master/worker 模式下要执行的代码
	ngx_master_process_cycle(cycle, &ctx);
} else {
	// 单进程模式下要执行的代码
	ngx_single_process_cycle(cycle, &ctx);
}


作者：herozem
链接：https://zhuanlan.zhihu.com/p/492319104
来源：知乎
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。

void ngx_master_process_cycle(ngx_cycle_t *cycle, ngx_master_ctx_t *ctx)
{
    char              *title;
    u_char            *p;
    size_t             size;
    ngx_int_t          i;
    sigset_t           set;
    struct timeval     tv;
    struct itimerval   itv;
    ngx_uint_t         live;
    ngx_msec_t         delay;
    ngx_core_conf_t   *ccf;

    // 设置感兴趣的信号，信号处理函数已经在 ngx_os_init 里设置了
    sigemptyset(&set);
    // ...

    // 设置进程名
    ngx_setproctitle(title);


    ccf = (ngx_core_conf_t *) ngx_get_conf(cycle->conf_ctx, ngx_core_module);

    // 此处创建worker进程
    ngx_start_worker_processes(cycle, ccf->worker_processes,
                               NGX_PROCESS_RESPAWN);
...
}
```

之后master本身就进入一个无限循环，通过 sigsuspend 函数阻塞自身，当收到信号时。

[参考知乎文章](https://www.zhihu.com/search?type=content&q=nginx%20%E7%9A%84cycle)

## worker如何工作

```
作者：herozem
链接：https://zhuanlan.zhihu.com/p/492319104
来源：知乎
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。

static void ngx_start_worker_processes(ngx_cycle_t *cycle, ngx_int_t n,
                                       ngx_int_t type)
{
    ngx_int_t         i;
    ngx_channel_t     ch;

    ch.command = NGX_CMD_OPEN_CHANNEL;
    while (n--) {
        // 起n个worker进程，执行 ngx_worker_process_cycle
        ngx_spawn_process(cycle, ngx_worker_process_cycle, NULL,
                          "worker process", type);
        ch.pid = ngx_processes[ngx_process_slot].pid;
        ch.slot = ngx_process_slot;
        ch.fd = ngx_processes[ngx_process_slot].channel[0];
        for (i = 0; i < ngx_last_process; i++) {
            if (i == ngx_process_slot
                || ngx_processes[i].pid == -1
                || ngx_processes[i].channel[0] == -1)
            {
                continue;
            }

			...
            /* TODO: NGX_AGAIN */
            ngx_write_channel(ngx_processes[i].channel[0],
                              &ch, sizeof(ngx_channel_t), cycle->log);
        }
    }
...
}
```

核心逻辑有两个，第一个是 ngx_spawn_process(...)， 第二个是 ngx_write_channel(...);

### ngx_spawn_process

逻辑是初始化各个模块，然后进入处理event的工作。

```
ngx_pid_t ngx_spawn_process(ngx_cycle_t *cycle,
                            ngx_spawn_proc_pt proc, void *data,
                            char *name, ngx_int_t respawn)
{
    // ...
    pid = fork();

    switch (pid) {
    case 0:
        ngx_pid = ngx_getpid();
		// 实际是ngx_worker_process_cycle
        proc(cycle, data); 
        break;
	...
    }
}

static void ngx_worker_process_cycle(ngx_cycle_t *cycle, void *data)
{
    // ...

    for (i = 0; ngx_modules[i]; i++) {
        if (ngx_modules[i]->init_process) {
            // 这里调用 init_process，此处就会初始化事件模块等
            if (ngx_modules[i]->init_process(cycle) == NGX_ERROR) {
                /* fatal */
                exit(2);
            }
        }
    }
    // ...
    for ( ;; ) {
        // worker 真正执行处理事件的地方
        ngx_process_events(cycle);
		...
    }
}
```

### master/worker的通信, ngx_write_channel()

```
// master/worker 之间进程间通信
ngx_int_t ngx_write_channel(ngx_socket_t s, ngx_channel_t *ch, size_t size,
                            ngx_log_t *log) 
{
    ssize_t             n;
    ngx_err_t           err;
    struct iovec        iov[1];
    struct msghdr       msg;

#if (HAVE_MSGHDR_MSG_CONTROL)
    union {
        struct cmsghdr  cm;
        char            space[CMSG_SPACE(sizeof(int))];
    } cmsg;

    if (ch->fd == -1) {
        msg.msg_control = NULL;
        msg.msg_controllen = 0;

    } else {
        msg.msg_control = (caddr_t) &cmsg;
        msg.msg_controllen = sizeof(cmsg);

        cmsg.cm.cmsg_len = sizeof(cmsg);
        cmsg.cm.cmsg_level = SOL_SOCKET; 
        cmsg.cm.cmsg_type = SCM_RIGHTS;
        *(int *) CMSG_DATA(&cmsg.cm) = ch->fd;
    }

#else
    if (ch->fd == -1) {
        msg.msg_accrights = NULL;
        msg.msg_accrightslen = 0;

    } else {
        msg.msg_accrights = (caddr_t) &ch->fd;
        msg.msg_accrightslen = sizeof(int);
    }

#endif

    iov[0].iov_base = (char *) ch;
    iov[0].iov_len = size;

    msg.msg_name = NULL;
    msg.msg_namelen = 0;
    msg.msg_iov = iov;
    msg.msg_iovlen = 1;

    n = sendmsg(s, &msg, 0);

    if (n == -1) {
        err = ngx_errno;
        if (err == NGX_EAGAIN) {
            return NGX_AGAIN;
        }

        ngx_log_error(NGX_LOG_ALERT, log, err, "sendmsg() failed");
        return NGX_ERROR;
    }

    return NGX_OK;
}

## 使用了共享内存和信号量的函数

### 共享内存的mutex

通过搜索查看`sem_t`在哪儿出现，结合`sem_<post|wait>`调用情况，找到在`src/core/ngx_shmtx.c:ngx_shmtx_<lock|wakeup>`封装信号量的处理。查看哪些函数调用了此函数，窥探IPC的机制。

src/stream, src/http, src/event模块都有使用。其中，stream 模块是用于处理 TCP 和 UDP 流量的模块。

## unix环境的共享内存

查看`src/os/unix/ngx_shmem.c:ngx_shm_alloc`调用地方，出现在：
* src/event/ngx_event.c:ngx_event_module_init()
* src/core/ngx_cycle.c:ngx_init_cycle()

因为cycle管理核心逻辑，从cycle部分开始看起。

```
// src/core/ngx_cycle.c:ngx_init_cycle()
ngx_cycle_t *
ngx_init_cycle(ngx_cycle_t *old_cycle) {
    ngx_shm_zone_t      *shm_zone, *oshm_zone;
    ngx_cycle_t         *cycle, **old;
...
    cycle = ngx_pcalloc(pool, sizeof(ngx_cycle_t));
...
    if (old_cycle->shared_memory.part.nelts) {
        n = old_cycle->shared_memory.part.nelts;
        for (part = old_cycle->shared_memory.part.next; part; part = part->next)
        {
            n += part->nelts;
        }

    } else {
        n = 1;
    }

    if (ngx_list_init(&cycle->shared_memory, pool, n, sizeof(ngx_shm_zone_t))
        != NGX_OK)
    {
        ngx_destroy_pool(pool);
        return NULL;
    }
...
    /* create shared memory */
    part = &cycle->shared_memory.part;
    shm_zone = part->elts;
...
    if (ngx_shm_alloc(&shm_zone[i].shm) != NGX_OK) {
       goto failed;
    }
...
}
```

在src/core/ngx_cycle.c中，ngx_shm_zone_t相关定义如下：

```
typedef struct ngx_shm_zone_s  ngx_shm_zone_t;
struct ngx_shm_zone_s {
    void                     *data;
    ngx_shm_t                 shm;
    ngx_shm_zone_init_pt      init;
    void                     *tag;
    void                     *sync;
    ngx_uint_t                noreuse;  /* unsigned  noreuse:1; */
};
```

有ngx_shm_t，这是我们关心部分。

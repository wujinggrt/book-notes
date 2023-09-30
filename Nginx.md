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

## 使用了共享内存和信号量的函数

通过搜索查看`sem_t`在哪儿出现，结合`sem_<post|wait>`调用情况，找到在`src/core/ngx_shmtx.c:ngx_shmtx_<lock|wakeup>`封装信号量的处理。查看哪些函数调用了此函数，窥探IPC的机制。

src/stream, src/http, src/event模块都有使用。其中，stream 模块是用于处理 TCP 和 UDP 流量的模块。
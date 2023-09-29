mmap有四种类型：
* Private file mapping: 对有关mappings的修改，不会影响背后的文件，也不会影响到比如继承下来的孩子进程，孩子进程看不到。使用了copy-on-write技术，通常用于加载程序文件、初始化数据段和加载库。
* Private anonymous mapping: child继承parent的mappings，但是parent和child进程都不会看到有关mappings的修改，用于创建新的mappings(zero-filled)。通常用于创建large blocks.
* Shared file mapping: 修改会影响背后的文件，其他继承的进程也会看得到修改。通常用于mmemery-mapped I/O和IPC.
* Shared anonymous mapping: 创建新的mappings(zero-filled)，没有copy-on-write的特性，映射到同一个页框(page frame)。通常用于IPC.

mappings are lost when a process performs an `exec()`, but are inherited by the child of a `fork()`. The child also **inherites** mapping type(`MAP_PRIVATE, MAP_SHARED`).

## API

```
#include <sys/mman.h>
void *mmap(void *addr, size_t length, int prot, int flags, int fd, off_t offset);
int munmap(void *addr, size_t length);
Returns starting address of mapping on success, or MAP_FAILED on error
```

* addr: the virtual addr at which the mapping is to be located. If we specify as `NULL`, the kernel would choose a suitable one. This is preferred.
* length: size in bytes.
* prot: A bit mask, we can OR it to make combination.
    * PROT_NONE: the region cannot be accessed
    * PROT_READ: can read
    * PROT_WRITE: can write
    * PROT_EXEC: can execute
* flags: `MAP_FIXED` means that `addr` must be page-aligned. It is quite common to provide `MAP_PRIVATE, MAP_SHARED`.
    * MAP_ANONYMOUS: Create an anonymous mapping—that is, a mapping that is not backed by a file.
    * MAP_LOCKED (since Linux 2.6) Preload and lock the mapped pages into memory in the manner of mlock().
* fd and offset: are used in file mappings. The offset specifies the starting point.
# mmap

四种类型：
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
* fd and offset: are used in file mappings. The offset specifies the starting point.对于匿名共享内存，指定-1和0即可。

## How to create anonymous mappings

* Specify `MAP_ANONYMOUS` in `flags` and `fd` as -1. Linux provides `MAP_ANON` for compatible reason.
* Open the `/dev/zero` virtual device file and pass to mmap.

/dev/zero是一个虚拟设备(virtual device)，读取总是返回0，写入总是被丢弃。

为什么会存在这两种方式？是因为使用MAP_ANONYMOUS方式来源于BSD，使用/dev/zero设备的方式来源于System V。一般来说，使用MAP_ANONYMOUS更为直观。为了考虑兼容性，一般编译参数定义USE_MAP_ANON，指定使用前者方式；反之，使用后者。

```
#ifdef USE_MAP_ANON
// anon
addr = mmap(NULL, shm->size, PROT_READ|PROT_WRITE,
         MAP_ANON|MAP_SHARED, -1, 0);)
#else
if (fd = open("/dev/zero", O_RDWR) == -1) {
    handle error ...
}
addr = mmap(NULL, size, 
        PROT_READ|PROT_WRITE, MAP_SHARED, fd, 0);
#endif
```

两种方式都会将mappings中的内容初始化为0，并且忽略offset参数，即传入0。

# POSIX semaphor

SUSv3 specifies two types of POSIX semaphor, **named semaphor** and **unnamed semaphor**.

Link with `-lpthread`

## named

```
#include <fcntl.h> /* Defines O_* constants */
#include <sys/stat.h> /* Defines mode constants */
#include <semaphore.h>
sem_t *sem_open(const char *name, int oflag, ...
/* mode_t mode, unsigned int value */ );
    Returns pointer to semaphore on success, or SEM_FAILED on error
```

oflag:
* 0 for opening an existing one.
* O_CREAT | O_EXCL for creating, failed if existed. Then we need 2 further params as following.

mode: O_WRONLY, O_RDONLY, O_RDWR.

value: initial value.

```
#include <semaphore.h>
int sem_close(sem_t *sem);
    Returns 0 on success, or –1 on error
int sem_unlink(const char *name);
    Returns 0 on success, or –1 on error
// 相比SysV，一次只能进行一个PV，不能组合起来。
// 如果sem<=0，blocks
int sem_wait(sem_t *sem);
    Returns 0 on success, or –1 on error
int sem_trywait(sem_t *sem);
    Returns 0 on success, or –1 on error
// 随机调度一个blocked的进程或者线程。
int sem_post(sem_t *sem);
    Returns 0 on success, or –1 on error
int sem_getvalue(sem_t *sem, int *sval);
    Returns 0 on success, or –1 on error
```

## Unamed

```
#include <semaphore.h>
int sem_init(sem_t *sem, int pshared, unsigned int value);
    Returns 0 on success, or –1 on error
int sem_destroy(sem_t *sem);
    Returns 0 on success, or –1 on error
```

pshared:
* 0: share between threads of the calling process. Typically, we specify `sem` as an address to a global variable or a heap allocated variable. The lifetime crosses the program.
* nonzero: share between process. It is wise to set `sem` as an address point a region of shared memory(SysV shm, mmap or POSIX shm). A child produced via `fork()` inherits those mappings.

Some implementation don't support process-shared semaphor(With the advent of NPTL, Linux had supported yet).

```C++
#include <semaphore.h>
#include <sys/mman.h>
#include <fcntl.h>
#include <sys/stat.h>
#include <unistd.h>

#include <iostream>
#include <string>
#include <thread>
#include <chrono>
using namespace std;

int main() {
	int child_pid = 0;
	int size = sizeof(sem_t) * 2 + 12;
	// int size = 12;
	void* addr = mmap(NULL, size, PROT_READ | PROT_WRITE, MAP_ANON | MAP_SHARED, -1, 0);
	if (addr == MAP_FAILED) {
		cerr << "Failed mmap\n";
		exit(EXIT_FAILURE);
	}
	sem_t *read_sem = (sem_t*)addr;
	sem_t *write_sem = read_sem + 1;
	if (sem_init(read_sem, 1, 0) == -1) {
		cerr << "Failed init sem\n";
		exit(EXIT_FAILURE);
	}
	if (sem_init(write_sem, 1, 1) == -1) {
		cerr << "Failed init sem\n";
		exit(EXIT_FAILURE);
	}
	uint8_t *data = reinterpret_cast<uint8_t*>(write_sem + 1);
	switch(child_pid = fork()) {
		case -1: {
			std::cerr << "failed" << endl;
			return -1;
		}
		case 0: {
			std::cout << "child:" << child_pid << endl;
			::sem_wait(write_sem);
			*reinterpret_cast<uint32_t*>(data) = 1;
			*reinterpret_cast<uint32_t*>(data + 4) = 2;
			cout << "child writed\n";
			::sem_post(read_sem);
			break;
		}
		default: {
			std::cout << "parent" << endl;
			::sem_wait(read_sem);
			cout << *reinterpret_cast<uint32_t*>(data) << endl;
			cout << *reinterpret_cast<uint32_t*>(data + 4) << endl;
			break;
		}
	}
	exit(EXIT_SUCCESS);
}
```
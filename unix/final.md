## Final!

### Terminal handling
`#include <termios.h>`

Terminal handling refers to how we can use the termios interface to modify terminal settings so it behaves differently

#### Key Functions & Things
`int isatty(int fd)`

Check if a file descriptor is a TTY

`int tcgetattr(int fd, struct termios *termios_p)`

Give it a file descriptor and a pointer to a termios struct. It will fill up the struct. **Returns 0 on success, -1 on error.**

`int tcsetattr(int fd, int optional_actions, const struct termios *termios_p);`

Same, but it will read the struct and apply changes to the terminal. `optional_actions` determines when the changes are written - immediately, after writing to fd, etc. **Returns 0 on success, -1 on error.**

`termios_p`

```
tcflag_t c_iflag;      /* input modes */
tcflag_t c_oflag;      /* output modes */
tcflag_t c_cflag;      /* control modes */
tcflag_t c_lflag;      /* local modes */
cc_t     c_cc[NCCS];   /* control chars */
```

#### Overview
- Create a `termios_p` struct
- Retrieve settings with `tcgetattr()`
- Modify the struct
    - Perform bitwise operations on struct members
- Update settings with `tcsetattr()`

#### Trivia
- These terminal settings affect things like:
    - Text appearing as you type it (`ECHO`)
    - Which signals are sent based on certain keypresses (`^C`, etc)
- Terminal settings can be viewed by running `stty -a`
- Settings regarding input start with I (eg `IGNCR`), output with O (eg `ONOCR`)

#### Errors
- If the file descriptor doesn't belong to a terminal

### Networking
`#include <sys/socket.h>`

Socket functions like `socket()` and `bind()`

`#include <arpa/inet.h>`

Conversion functions like `htonl()` and `ntohl()`

#### Key Functions & Things

`int socket(int domain, int type, int protocol)`

Creates a socket. This is necessary for server and client. `domain` is `AF_INET` for IPv4, `AF_INET6` for IPv6, `AF_UNIX` for unix domain sockets. `type` is `SOCK_STREAM` for TCP, `SOCK_DGRAM` for UDP, `SOCK_RAW` to skip transport layer, and `SOCK_SEQPACKET` for SCTP. *`protocol` is always 0.* **Returns a file descriptor.**

`int bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen)`

Binds to a socket. `sockfd` is a socket returned by `socket()`, `addr` says which interface and port to bind (`sockaddr` object), and `addrlen` which says how big that object is.

`int listen(int sockfd, int backlog)`

Put the socket in listening/passive mode. This opens up the port so connections can be made. 

`int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen)`

Accepts a connection. The `sockaddr` struct is only needed if you want information about the connecting client. If this information is not needed, `NULL` can be passed to `addr` and `addrlen`.

`int connect(int sockfd, const struct sockaddr *addr, socklen_t addrlen)`

Connect to a server using an open socket. **Returns 0 on success.**

Connect to a server using an open socket. **Returns 0 on success.**

    struct sockaddr_in sa;
    memset(&sa, 0, sizeof(sa));
    sa.sin_family = AF_INET;
    sa.sin_port = htons(9999);
    sa.sin_addr.sa_addr = aton("0.0.0.0");

Sample initialization of sockaddr_in.

#### Overview

- Server
    - socket()
    - bind()
    - listen()
    - clientfd = accept()
    - clientfd can be used to communicate with client
- Client
    - sockfd = socket()
    - connect()
    - sockfd can be used to communicate with server

#### Trivia

- `bind()` requires a `socklen_t` size because `sockaddr` objects can be different size depending on the protocol.
- Both read() and write() can be used in place of recv() and send(), but lack specialized socket stuff. read() is equivalent to recv() with flags set to 0. Not true on Windows.
- `recv()` returns 0 when the connection closes gracefully

#### Errors

- SIGPIPE on writing to disconnected socket
    - Fix by SIGPIPE handler or set SO_NOSIGPIPE option on socket


### Client / Server. I/O multiplexing.

`#include <sys/select.h>`

`pselect() and select()` allow us to monitor multiple file descriptors to check if they are ready for reading or writing. Examples include a client application that reads from stdin and a socket at the same time.

#### Key Functions & Things

`int select(int nfds, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout)`

`nfds` is the MAX_FD + 1. `readfds` is basically an array of the file descriptors you want to be able to read from. `writefds` is the same but for writing. `exceptfds` idk. `timeout` is a timeval struct that takes seconds and microseconds.

`int pselect(int nfds, fd_set *readfds, fd_set *writefds,  fd_set *exceptfds, const struct timespec *timeout, const sigset_t *sigmask);`

Same thing as above, except timeout is a `timespec` which takes seconds and *nanoseconds*. It also takes a sigmask, which specifies the signals that should be blocked for the duration of the pselect call. **This avoids the race condition that would be present by using sigprocmask and then select() right away**

Any of the pointers may be NULL. However, if both `readfds` and `writefds` are NULL, `select` blocks forever.

`select` modifies the fd_sets passed in. Ensure that they are re-initialized before the next call.

`FD_SET()` `FD_ISSET()` `FD_ZERO()`

#### Overview

- create an fd_set, empty it with `FD_ZERO(&fd_set)`
- add your file descriptors to it using `FD_SET(fd, &fd_set)`
- in a loop, call `select()`
    - It'll block until it hits timeout or an fd becomes ready
    - Use `FD_ISSET(fd, &fd_set)` to check if `fd` is ready
- `fd_set` gets reset after the call to `select()` so you need to rebuild it

#### Trivia

- `select()`'s timeout uses `timeval` (*seconds and microseconds*) while `pselect()'s` timeout uses `timespec` (*seconds and nanoseconds*)
- `select()` might modify its timeout variable to let the user know how much time was remaining. `pselect()` does not.
- `select()` cannot handle signals and does not take a sigmask. `pselect()` does. **select is pretty much pselect, but with sigmask set to NULL.**
- `pselect()` solves the race condition issue with `sigprocmask()` and `select()` combined. `pselect` is a single atomic syscall
- `fd_set` gets reset after every call to select
- timeouts: 0 seconds 0 nano/microseconds: return immediately; NULL timeout: block indefinitely

### Multi-threading: basics, mutual exclusion

`#include <pthread.h>`

Needs to be compiled with -lpthread.

#### Key Functions & Things

`int pthread_create(pthread_t *thread, const pthread_attr_t *attr, void *(*start_routine) (void *), void *arg);` 

Make the thread. You need to instantiate a pthread_t object and pass it to this function, where it will then be modified. Once the thread starts, it will call `start_routine` with the parameter `arg`.

`int pthread_join(pthread_t thread, void **retval)`

Wait for a thread to finish execution. After creating a bunch of threads, you could call this function so that the next piece of code that executes can be sure that all the threads are done. If `retval` is not null, it will contain a pointer to the return value from the thread (basically what was passed to `pthread_exit()`)

`int pthread_detach(pthread_t thread);`

Detaches a thread. You can do this in the spawned thread like so `pthread_detach(pthread_self());`

`void pthread_exit(void *retval)`

Exits the current running thread and returns whatever `retval` is.

`int pthread_mutex_init(pthread_mutex_t *mutex, const pthread_mutexattr_t *mutexattr);`

Used for mutex initalization. `mutexattr` can be NULL.

Three types of mutexes

- fast - `PTHREAD_MUTEX_INITIALIZER`
- recursive - `PTHREAD_RECURSIVE_MUTEX_INITIALIZER_NP`
- errorcheck - `PTHREAD_ERRORCHECK_MUTEX_INITIALIZER_NP`

`int pthread_mutex_lock(pthread_mutex_t *mutex);`

`int pthread_mutex_trylock(pthread_mutex_t *mutex)`

`int pthread_mutex_unlock(pthread_mutex_t *mutex)`

`int pthread_mutex_destroy(pthread_mutex_t *mutex);`

### Multi-threading: bounded buffers, condition variables

Condition variables are like mutexes. Mutexes synchronize threads by blocking access to data, while condtion variables synchronize threads by reading the value of the data

`pthread_cond_t cond = PTHREAD_COND_INITIALIZER;`

`int pthread_cond_init(pthread_cond_t *cond, pthread_condattr_t *cond_attr);`

Initialize a condition variable. `cond_attr` can be NULL.

`int pthread_cond_signal(pthread_cond_t *cond);`

Signal the cond variable.

`int pthread_cond_broadcast(pthread_cond_t *cond);`

Signal to ALL threads waiting on the cond variable.

`int pthread_cond_wait(pthread_cond_t *cond, pthread_mutex_t *mutex);`

Guess?

`int pthread_cond_destroy(pthread_cond_t *cond);`

Frees you of your worldly obligations.

### Multi-threading: deadlocks

pthread mutexs can be initialized with errorcheck_np. This will detect if the thread will deadlock when trying to obtain a mutex it already owns.

### Non-blocking I/O. Regular expressions.

    int flags = fcntl(fd, F_GETFL, 0);
    fcntl(fd, F_SETFL, flags | O_NONBLOCK);

Remember the flag name, you tool.

### Sys V IPC. Semaphores and shared memory.

`#include <sys/types.h>`

`#include <sys/ipc.h>`

`#include <sys/shm.h>`

Shared memory API allows virtual memory space of different processes to point to the same physical location in RAM. This allows for IPC because they can read and write to the same piece of memory.

#### Key Functions & Things

`int shmget(key_t key, size_t size, int shmflg)`

`key` is the unique key shared between the processes using the memory. Safest to generate using `ftok()`. `size` is how big of an allocation you want, and `shmflg` is the flags. 

`void *shmat(int shmid, const void *shmaddr, int shmflg)`

`shmid` is the id given to you by `shmget()`. `shmaddr` can be the address where you want the mapping, or NULL if you want one given to you. `shmflag` allows you to specify things like readonly etc. This function is necessary so that you can read and write to the shared memory.

`int shmdt(const void *shmaddr)`

This takes the virtual address given by `shmat()` and "detaches" from it. 

`int shmctl(int shmid, int cmd, struct shmid_ds *buf)`

Like fcntl but for shared memory. 

#### Overview

- create a unique key that can be determined by both processes
    - `ftok()` for example takes a current file and gives a key
- `shmget()` to allocate your memory
- `shmat()` to attach to it
    - you now have a virtual address that can be used as a regular pointer but can be shared across processes
- `shmdt()` to detach the piece of shared memory

#### Trivia

- If using `ftok()` to generate a key, the file passed to it must exist (it fstats the path)
- Though the virtual addresses between the two processes will differ, the underlying physical addresses will be the same

#### Semaphores

`int semget(key_t key, int nsems, int semflg);`

`int semid = semget(IPC_PRIVATE, 1, 0600);`

Get a semaphore. Note that the value is undefined!

    union semun {
        int              val;    /* Value for SETVAL */
        struct semid_ds *buf;    /* Buffer for IPC_STAT, IPC_SET */
        unsigned short  *array;  /* Array for GETALL, SETALL */
        struct seminfo  *__buf;  /* Buffer for IPC_INFO
                                    (Linux-specific) */
    };

`semun` union. Code has to define this!

    union semun se;
    se.val = 10;
    semctl(semid, 0, IPC_SET, se)

Initialization of semaphore.

`int semop(int semid, struct sembuf *sops, unsigned nsops);`

Semops...

    sembuf op;
    op.sem_num = 0;
    op.sem_op = 1;
    op.sem_flg = SEM_UNDO;
    semop(semid, &op, 1);

Up a semaphore.

`semctl(semid, 0, IPC_RMID);`

Remove a semaphore set.

##### Trivia

- These are Sys V semaphores. POSIX has a (simpler) semaphore implementation too.

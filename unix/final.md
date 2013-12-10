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
- Both read() and write() can be used in place of recv() and write(), but lack specialized socket stuff. read() is equivalent to recv() with flags set to 0. Not true on windows

### Client / Server. I/O multiplexing.

`#include <sys/select.h>`

`pselect() and select()` allow us to monitor multiple file descriptors to check if they are ready for reading or writing. Examples include a client application that reads from stdin and a socket at the same time.

#### Key Functions & Things

`int select(int nfds, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout)`

#### Overview

#### Trivia

- `select()`'s timeout uses `timeval` (*seconds and microseconds*) while `pselect()'s` timeout uses `timespec` (*seconds and nanoseconds*)
- `select()` might modify its timeout variable to let the user know how much time was remaining. `pselect()` does not.
- `select()` cannot handle signals and does not take a sigmask. `pselect()` does. **select is pretty much pselect, but with sigmask set to NULL.**

### Multi-threading: basics, mutual exclusion

### Multi-threading: bounded buffers, condition variables

### Multi-threading: deadlocks

### Non-blocking I/O. Regular expressions.

### Sys V IPC. Semaphores and shared memory.

### Shell scripting

### TBD


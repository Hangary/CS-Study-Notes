# Networking

## Port

To send a datagram (packet) to a host on the Internet using IPv4 (or IPv6) you need to specify the host address and a port. The port is an unsigned **16** bit number (i.e. the maximum port number is `65535`).

A process can listen for incoming packets on a particular port. 
- However, only processes with **super-user** (root) access can listen on ports < `1024`. 
- Any process can listen on ports `1024` or higher.
- An often used port is port 80: Port 80 is used for unencrypted http requests (i.e. web pages). For example, if a web browser connects to http://www.bbc.com/, then it will be connecting to port 80.

## Address

We can `getaddrinfo` to convert a domain into an `addrinfo`, which is a linked-list.
- Note: Error handling with getaddrinfo is a little different:
    - The return value is the error code (i.e. don't use `errno`)
    - Use `gai_strerror` to get the equivalent short English error text instead of `strerror`.

```C
struct addrinfo {
    int              ai_flags;
    int              ai_family;
    int              ai_socktype;
    int              ai_protocol;
    socklen_t        ai_addrlen;
    struct sockaddr *ai_addr;
    char            *ai_canonname;
    struct addrinfo *ai_next;
};
```

We can configure `addrinfo` to specify our requirements:
```C
struct addrinfo hints;
memset(&hints, 0, sizeof (hints));

hints.ai_family = AF_INET6; // Only want IPv6 (use AF_INET for IPv4)
hints.ai_socktype = SOCK_STREAM; // Only want stream-based connection
````

Typical usage:
```C
#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netdb.h>

struct addrinfo hints, *infoptr; // So no need to use memset global variables

int main() {
    hints.ai_family = AF_INET; // AF_INET means IPv4 only addresses

    int result = getaddrinfo("www.bbc.com", NULL, &hints, &infoptr);
    if (result) {
        fprintf(stderr, "getaddrinfo: %s\n", gai_strerror(result));
        exit(1);
    }

    struct addrinfo *p;
    char host[256];

    for (p = infoptr; p != NULL; p = p->ai_next) {

        getnameinfo(p->ai_addr, p->ai_addrlen, host, sizeof (host), NULL, 0, NI_NUMERICHOST);
        puts(host);
    }

    // free the memory allocated for the linked-list of addrinfo structs
    freeaddrinfo(infoptr);
    return 0;
}
```

## Socket

```C
int socket(int domain, int socket_type, int protocol);
```

The `socket` call creates an outgoing *socket* and returns a descriptor (sometimes called a 'file descriptor') that can be used with *read* and *write* etc. 
- In this sense it is the network analog of open that opens a file stream
- However, we need to connect the socket to something. 

```C
// Pull out the socket address info from the addrinfo struct:
connect(sockfd, p->ai_addr, p->ai_addrlen);
```

If you `fork`-ed after creating a socket file descriptor, all processes need to close the socket before the socket resources can be reused. If you shut down a socket for further read, all processes are affected because you've changed the socket, not the file descriptor. Well written code will `shutdown` a socket before calling `close` it.


## Server

Four important network calls for creating a server:
- `int socket(int domain, int socket_type, int protocol)`: creates a endpoint for networking communication. 
  - A new socket by itself is not particularly useful; though we've specified either a packet or stream-based connections, it is not bound to a particular network interface or port. 
  - Instead, socket returns a network descriptor that can be used with later calls to bind, listen and accept.
  - Sockets can be declared ***passive***. Passive sockets wait for another host to connect.
    - Passive sockets are particularly useful for servers.
    - Server sockets remain open when the peer disconnects. 
    - Since a TCP connection is defined by the sender address and port along with a receiver address and port, a particular server port there can be one passive server socket but multiple active sockets. One for each currently open connection. The server's operating system maintains a lookup table that associates a unique tuple with active sockets so that incoming packets can be correctly routed to the correct socket.
- `int bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen)`: associates an abstract socket with an actual network interface and port. 
  - It is possible to call bind on a TCP client however it's unusually unnecessary to specify the outgoing port.
  - By default, a port is released after some time when the server socket is closed. 
    - When the corresponding socket is closed, the port enters a "TIMED-WAIT" state.
    - This might lead to some errors because ports are occupied.
    - To be able to immediately reuse a port, specify `SO_REUSEPORT` before binding to the port.
- `int listen(int sockfd, int backlog)`: specifies the queue size for the number of incoming, unhandled connections i.e. that have not yet been assigned a network descriptor by accept 
  - Typical values for a high performance server are `128` or more.
- `int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen)`: Once the server socket has been initialized the server calls accept to wait for new connections. 
  - Unlike socket bind and listen, this call will block. i.e. if there are no new connections, this call will block and only return when a new client connects. 
  - The returned TCP socket is associated with a particular tuple (client IP, client port, server IP, server port) and will be used for all future incoming and outgoing TCP packets that match this tuple.
    - Note the accept call returns a new file descriptor. This file descriptor is specific to a particular client. It is common programming mistake to use the original server socket descriptor for server I/O and then wonder why networking code has failed.
- `int close(int fd)` and `int shutdown(int fd, int how)`:
  - Use the `shutdown` call when you no longer need to read any more data from the socket, write more data, or have finished doing both.
  - When you call `shutdown` on socket on the read and/or write ends, that information is also sent to the other end of the connection.
  - Use `close` when your process no longer needs the socket file descriptor.

Few notes for creating a server:
- Using the socket descriptor of the passive server socket
- Not specifying `SOCK_STREAM` requirement for getaddrinfo
- Not being able to reuse an existing port.
- Not initializing the unused struct entries
- The `bind` call will fail if the port is currently in use. Ports are per machine - not per process or user. In other words, you cannot use port `1234` while another process is using that port. Worse, ports are by default 'tied up' after a process has finished.

## Non-blocking I/O

POSIX lets you set a flag `NONBLOCK` on a file descriptor such that any call to `read()` on that file descriptor will return immediately, whether it has finished or not.

With your file descriptor in this mode, your call to `read()` will start the read operation, and while it's working you can do other useful work. This is called *non-blocking* mode, since the call to read() doesn't block.

To set a file descriptor to be non-blocking:
```C
// fd is my file descriptor
int flags = fcntl(fd, F_GETFL, 0);
fcntl(fd, F_SETFL, flags | O_NONBLOCK);
```

To create a non-blocking socket, we can add `SOCK_NONBLOCK` to the second argument to `socket()`:
```C
fd = socket(AF_INET, SOCK_STREAM | SOCK_NONBLOCK, 0);
```

Note: In addition to nonblocking `read()` and `write()`, any calls to `connect()` on a nonblocking socket will also be nonblocking. To wait for the connection to complete, use `select()` or epoll to wait for the socket to be writable.

When a file is in nonblocking mode and you call `read()`, it will return immediately with whatever bytes are available:
- Say `100` bytes have arrived from the server at the other end of your socket and you call `read(fd, buf, 150)`. Read will return immediately with a value of `100`, meaning it read `100` of the `150` bytes you asked for. 
- Say you tried to read the remaining data with a call to read`(fd, buf+100, 50)`, but the last 50 bytes still hadn't arrived yet. `read()` would return `-1` and set the global error variable `errno` to either `EAGAIN` or `EWOULDBLOCK`.

---------

To check whether an I/O has finished, POSIX provides serveral ways:
1. `select`
2. `epoll`

### Select

```C
int select(int nfds, 
           fd_set *readfds, 
           fd_set *writefds,
           fd_set *exceptfds, 
           struct timeval *timeout);
```

Given three sets of file descriptors, `select()` will wait for any of those file descriptors to become 'ready'.
- `readfds` - a file descriptor in `readfds` is ready when there is data that can be read or EOF has been reached.
- `writefds` - a file descriptor in `writefds` is ready when a call to `write()` will succeed.
- `exceptfds` - system-specific, not well-defined. Just pass `NULL` for this.

`select()` returns the total number of file descriptors that are ready. 
- If none of them become ready during the time defined by timeout, it will return `0`. 
- After `select()` returns, the caller will need to loop through the file descriptors in readfds and/or writefds to see which ones are ready. 
- As `readfds` and `writefds` act as both input and output parameters, when `select()` indicates that there are file descriptors which are ready, it would have overwritten them to reflect only the file descriptors which are ready. Unless it is the caller's intention to call `select()` only once, it would be a good idea to save a copy of readfds and writefds before calling it.

An example on `select()`:
```C
fd_set readfds, writefds;
FD_ZERO(&readfds);
FD_ZERO(&writefds);
for (int i=0; i < read_fd_count; i++)
    FD_SET(my_read_fds[i], &readfds);
for (int i=0; i < write_fd_count; i++)
    FD_SET(my_write_fds[i], &writefds);

struct timeval timeout;
timeout.tv_sec = 3;
timeout.tv_usec = 0;

int num_ready = select(FD_SETSIZE, &readfds, &writefds, NULL, &timeout);

if (num_ready < 0) {
    perror("error in select()");
} else if (num_ready == 0) {
    printf("timeout\n");
} else {
    for (int i=0; i < read_fd_count; i++)
    if (FD_ISSET(my_read_fds[i], &readfds))
        printf("fd %d is ready for reading\n", my_read_fds[i]);
    for (int i=0; i < write_fd_count; i++)
    if (FD_ISSET(my_write_fds[i], &writefds))
        printf("fd %d is ready for writing\n", my_write_fds[i]);
}
```

### Epoll

> `epoll` is not part of POSIX, but it is supported by Linux/ It is more effeicient way to wait for a large amount of file descriptors.
>
> `epoll()` arose out of the inefficiencies of `select()` and `poll()`, especially for handling **the C10k problem**. `epoll()` operates in $O(1)$ time, which is much more efficient than the older system calls $O(n)$.

`epoll()` is a Linux kernel system call for a scalable I/O event notification mechanism. Its function is to monitor multiple file descriptors to see whether I/O is possible on any of them.

`epoll()` provides two triggering modes, **edge triggered (ET)** and **level triggered (LT)**:
- In edge-triggered mode, a call to `epoll_wait` will return only when a new event is enqueued with the `epoll` object.
  - Edge-triggered mode is useful when there are multiple threads blocking on the same epoll descriptor.
- In level-triggered mode, `epoll_wait` will return as long as the condition holds.

#### How to use `epoll`

To use epoll, we firstly create a special file descriptor:
```C
epfd = epoll_create(1);
```

For each file descriptor you want to monitor with epoll, you'll need to add it to the epoll data structures using `epoll_ctl()` with the `EPOLL_CTL_ADD` option. You can add any number of file descriptors to it.
```C
struct epoll_event event;
event.events = EPOLLOUT;  // EPOLLIN==read, EPOLLOUT==write
event.data.ptr = mypointer;
epoll_ctl(epfd, EPOLL_CTL_ADD, mypointer->fd, &event)
```

To wait for some of the file descriptors to become ready, use `epoll_wait()`. The `epoll_event` struct that it fills out will contain the data you provided in event.data when you added this file descriptor. This makes it easy for you to look up your own data associated with this file descriptor.
```C
int num_ready = epoll_wait(epfd, &event, 1, timeout_milliseconds);
if (num_ready > 0) {
    MyData *mypointer = (MyData*) event.data.ptr;
    printf("ready to write on %d\n", mypointer->fd);
}
```

To change the type of operation you're monitoring, use `epoll_ctl()` with the `EPOLL_CTL_MOD` option.
```C
event.events = EPOLLIN;
event.data.ptr = mypointer;
epoll_ctl(epfd, EPOLL_CTL_MOD, mypointer->fd, &event);
```

To unsubscribe one file descriptor from epoll while leaving others active, use `epoll_ctl()` with the `EPOLL_CTL_DEL` option.
```C
epoll_ctl(epfd, EPOLL_CTL_DEL, mypointer->fd, NULL);
```

To shut down an epoll instance, close its file descriptor.
```C
close(epfd);
```
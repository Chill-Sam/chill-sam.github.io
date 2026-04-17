+++
title = "Building a TCP Server in C — devblog"
date = 2026-04-16
[taxonomies]
tags = ["c", "networking", "chillssh"]
+++
# Building a TCP Server in C — devblog

## Why?

I wanted to really understand how servers work at the system call level. Not "use a framework", not "call a library" — I mean actually reaching down into the kernel and building something that listens for connections, reads data, and writes it back.

The end goal of `chillssh` is a from-scratch SSH server that accepts connections and serves an application, built entirely by hand, no libssh, no OpenSSL. But before any SSH, before any crypto, before any protocol parsing, you need a working TCP server. This is how I got there.

---

## Step 1: The Simplest Possible Server

The first commit was about getting something that compiles, binds a port, and actually talks to a client. Nothing fancy.

The core flow is pretty mechanical once you've seen it once:

```c
int fd = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);
setsockopt(fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));
bind(fd, (struct sockaddr *)&addr, sizeof(addr));
listen(fd, SOMAXCONN);

while (true) {
    int client = accept(fd, ...);
    write(client, "Hello World!\n", 13);
    close(client);
}
```

`SO_REUSEADDR` is immediately important, without it, restarting the server while TIME_WAIT sockets are lingering will refuse to bind. Learned that one fast.

The accept loop blocks the process entirely on each connection. One client at a time, and we close immediately after writing. It's basically useless for anything real, but it was a useful foundation to build on.

I also set up a small logging system from the start — a header-only macro based logger with `LOG_ERROR`, `LOG_WARNING`, and `LOG_INFO` levels. Printing to stderr with `__FILE__` and `__LINE__` context. Nothing fancy but it saved me a lot of printf-debugging.

One thing I want to be deliberate about throughout this project: writing robust, safe, modern C. The Makefile compiles with `-Wall -Wextra -Wpedantic -std=c23`. C23 is the latest standard and the strict warning flags mean the compiler catches a lot of sloppiness before it becomes a bug.

---

## Step 2: Multiple Clients with epoll

The blocking accept loop obviously can't handle multiple clients. The naive fix is threads, but I wanted to go the event-driven route with `epoll` — Linux's high-performance I/O event notification interface.

The idea behind epoll is: instead of blocking on a single fd, you register a set of fds with an epoll instance and call `epoll_wait`. The kernel tells you which fds are ready, and you handle them one by one.

To make this work, every fd needs to be in non-blocking mode:

```c
int flags = fcntl(fd, F_GETFL);
fcntl(fd, F_SETFL, flags | O_NONBLOCK);
```

Then you create an epoll instance and register your server socket:

```c
int epoll_fd = epoll_create1(0);
struct epoll_event ev = { .events = EPOLLIN, .data.fd = socket_fd };
epoll_ctl(epoll_fd, EPOLL_CTL_ADD, socket_fd, &ev);
```

The event loop then becomes:

```c
while (true) {
    int n = epoll_wait(epoll_fd, events, MAX_EVENTS, -1);
    for (int i = 0; i < n; i++) {
        int fd = events[i].data.fd;
        if (fd == server_fd) {
            // new connection — accept and register with epoll
        } else {
            // data from existing client — read and echo back
        }
    }
}
```

The distinguishing trick here is comparing `fd` against the server fd to know whether an event is a new connection or incoming data. It works, but it's a bit fragile. You're relying on fd integer comparison to dispatch logic.

For echoing, handling `bytes == 0` is important — that's the client disconnecting gracefully. And `EAGAIN` on a non-blocking read just means there's no data right now, not an error.

---

## Step 3: Modularization and a Tagged Union for epoll Dispatch

The second epoll implementation worked, but it was 150+ lines of tangled logic in `main.c`. I wanted to break it into something that could actually grow. So I pulled it apart into:

- `server.c` / `server.h` — owns the listening socket, the epoll instance, and the client list
- `conn.c` / `conn.h` — represents a single client connection
- `epoll_ctx.h` — a tagged union that replaces the fragile fd comparison

The tagged union is the most interesting design decision here. Instead of using `epoll_event.data.fd` (an integer), I use `epoll_event.data.ptr` to point at a context struct:

```c
typedef struct {
    epoll_type_t type; // SERVER or CONN
    union {
        server_t *server;
        conn_t   *conn;
    };
} epoll_ctx_t;
```

Every server and every connection embeds one of these as its first field. When epoll fires an event, I cast `data.ptr` back to `epoll_ctx_t *` and switch on the type:

```c
epoll_ctx_t *ctx = events[i].data.ptr;
switch (ctx->type) {
case EPOLL_TYPE_SERVER:
    // accept new connection
case EPOLL_TYPE_CONN:
    // read/write data
}
```

This is much cleaner than integer comparison and scales naturally. If I add more event source types later, I just add a new `epoll_type_t` variant.

The `server_t` struct itself owns a fixed-size array of `conn_t *` pointers (64 slots for now), an `epoll_event` buffer, the listening fd, and the epoll fd. All the socket setup that used to live in `main` moved into `server_start()`, and the event loop became `server_poll()`.

`main.c` got dramatically simpler — it just parses args, sets up signal handling, creates the server, and loops on `server_poll()`.

### Signal handling

I also added proper signal handling in this pass. `SIGINT` and `SIGTERM` both set a `volatile sig_atomic_t running = 0` flag. The key detail is `SA_RESTART` is deliberately **not** set:

```c
sa.sa_flags = 0; // no SA_RESTART — we want epoll_wait to return EINTR
```

Without this, the signal would restart `epoll_wait` transparently and the loop would never notice. With it, the signal causes `epoll_wait` to return `-1` with `errno == EINTR`, which I treat as a clean shutdown signal.

### Logging upgrades

The logger also got an upgrade — colors via ANSI escape codes and timestamps:

```
[2026-04-16 10:08:11] [INFO] Connection received: 5
[2026-04-16 10:08:11] [WARN] src/server.c:306: read() call failed: Connection reset by peer
```

Errors and warnings include file/line info; info logs don't clutter the output with it.

---

## Current State

The server:
- Binds a TCP port
- Handles multiple concurrent connections via `epoll`
- Echoes received data back to the sender
- Cleans up gracefully on `SIGINT`/`SIGTERM`
- Compiles clean under `-Wall -Wextra -Wpedantic -std=c23`

The code is organized into modules with clear ownership and a reasonably extensible dispatch model. A solid foundation to build on.

Next up: crypto. Before `chillssh` can speak SSH, it needs to speak the cryptographic primitives that SSH is built on. I'll be implementing ChaCha20 from scratch, the stream cipher used in the `chacha20-poly1305` AEAD construction that modern SSH connections negotiate. More on that in the next post.

---
layout: default
title: "COROIO — Async I/O & Actors for Modern C++"
---

# COROIO

Coroutine-based async I/O library for C++23. Zero external dependencies in the core. Pluggable polling backends from `select` to `io_uring`. Built-in single-threaded actor system.

[GitHub](https://github.com/resetius/coroio) · [Get Started](/getting-started/) · [API Docs](/docs/html/index.html)

---

## Architecture

```
TLoop<TPoller>
├── TPollerBase     — timers, Sleep(), Yield(), Wakeup()
├── TSocket         — async TCP/UDP: Connect, Accept, ReadSome, WriteSome
├── TFileHandle     — async fd: ReadSome, WriteSome
└── Backends
    ├── TEPoll      Linux epoll
    ├── TKqueue     macOS / FreeBSD kqueue
    ├── TIOCp       Windows IOCP
    ├── TUring      Linux io_uring  (liburing)
    ├── TPoll       POSIX poll(2)
    └── TSelect     POSIX select(2)
```

## Echo Server in 15 Lines

```cpp
#include <coroio/all.hpp>
using namespace NNet;

template<typename TSocket>
TVoidTask handle(TSocket socket) {
    char buf[4096]; ssize_t n;
    while ((n = co_await socket.ReadSome(buf, sizeof buf)) > 0)
        co_await socket.WriteSome(buf, n);
}

template<typename TPoller>
TVoidTask serve(TPoller& poller, TAddress addr) {
    typename TPoller::TSocket listener(poller, addr.Domain());
    listener.Bind(addr); listener.Listen();
    while (true) handle(co_await listener.Accept());
}

int main() {
    TLoop<TDefaultPoller> loop;
    serve(loop.Poller(), TAddress{"::", 8888});
    loop.Loop();
}
```

`TVoidTask` is fire-and-forget — `handle()` runs concurrently for each client without blocking the accept loop. Swap `TDefaultPoller` for `TEPoll`, `TUring`, `TKqueue`, or `TSelect` at compile time — no other changes needed.

## Coroutine Types

| Type | Lifetime | Exceptions | Use for |
|---|---|---|---|
| `TVoidTask` | self-destructs | swallowed | detached tasks (per-client handlers, background loops) |
| `TFuture<void>` | RAII handle | propagated | tasks you need to await or check `.done()` |
| `TFuture<T>` | RAII handle | propagated | tasks that return a value |

## I/O Helpers

| Type | Purpose |
|---|---|
| `TByteReader(sock)` | Read exactly N bytes — `co_await r.Read(buf, n)` |
| `TByteWriter(sock)` | Write exactly N bytes — `co_await w.Write(buf, n)` |
| `TLineReader(fd, max)` | Newline-terminated reads via circular buffer |

## Networking Stack

| Module | Include | Description |
|---|---|---|
| DNS | `coroio/dns/resolver.hpp` | Async UDP resolver with caching; `THostPort` for host:port |
| HTTP Server | `coroio/http/httpd.hpp` | `TWebServer<TSocket>` + `IRouter` interface |
| WebSocket | `coroio/ws/ws.hpp` | Client-side, `ws://` and `wss://`, text frames |
| SSL/TLS | `coroio/ssl.hpp` | `TSslSocket<T>` drop-in TLS layer over any socket |
| Subprocess | `coroio/pipe/pipe.hpp` | Async stdin/stdout/stderr (Linux, macOS) |

## Actor System

Single-threaded actor system sharing the event loop — no locks, no threads.

```
TActorSystem
├── Mailbox per actor     — FIFO, paused while async handler runs
├── TActorId              — {NodeId, LocalActorId, Cookie}
├── TActorContext         — Send, Schedule, Ask, Sleep, Become
└── TNode per remote node — TCP connection, auto-reconnect, serialization
```

Three actor styles:

- **`IActor`** — synchronous `Receive()`, pure computation, tight dispatch loop
- **`ICoroActor`** — async `CoReceive()`, can `co_await` timers and replies
- **`IBehaviorActor + TBehavior`** — typed per-message `Receive()` overloads, switchable state machines

[Full actors documentation →](/actors/)

## Benchmarks

Actor ring topology — i5-11400F · Ubuntu 25.04 · 100 actors · batch 1024:

| Framework | Local msg/s | Distributed msg/s (0 B payload) |
|---|---|---|
| **COROIO** | **442,151** | **1,137,790** |
| Akka | 473,966 | 5,765 |
| CAF | 302,930 | 55,540 |
| YDB/actors | 151,972 | 182,525 |

[I/O benchmarks vs libevent →](/benchmarks/)

## Dependencies

| Component | Dependency |
|---|---|
| Core | C++ standard library + OS |
| io_uring backend | liburing |
| SSL/TLS | OpenSSL |
| All else | none |

## Projects Using COROIO

- [miniraft-cpp](https://github.com/resetius/miniraft-cpp) — minimalist Raft consensus in C++
- [Qumir](https://qumir.dev) — online compiler for the Kumir language ([GitHub](https://github.com/resetius/qumir))

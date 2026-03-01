---
layout: default
title: "Echo Client/Server"
permalink: /echoclientserver/
---

# Echo Client/Server

A walkthrough of the canonical COROIO example — full source at [echoclient.cpp](https://github.com/resetius/coroio/blob/master/examples/echoclient.cpp) and [echoserver.cpp](https://github.com/resetius/coroio/blob/master/examples/echoserver.cpp).

---

## Echo Server

### Design

```
main()
└── serve<TPoller>()              TVoidTask — accept loop, runs forever
      └── socket.Accept() ──────► client_handler()   TVoidTask per connection
                                    ReadSome → WriteSome loop
```

Each `client_handler` is a `TVoidTask`: fire-and-forget, starts immediately, destroys itself on disconnect. No thread pool, no callbacks.

### Core Logic

```cpp
#include <coroio/all.hpp>
using namespace NNet;

template<typename TSocket>
TVoidTask client_handler(TSocket socket, int buf_size) {
    std::vector<char> buf(buf_size);
    ssize_t n;
    try {
        while ((n = co_await socket.ReadSome(buf.data(), buf_size)) > 0)
            co_await socket.WriteSome(buf.data(), n);
    } catch (const std::exception& ex) {
        std::cerr << "client error: " << ex.what() << "\n";
    }
}

template<typename TPoller>
TVoidTask serve(TPoller& poller, TAddress addr, int buf_size) {
    typename TPoller::TSocket listener(poller, addr.Domain());
    listener.Bind(addr);
    listener.Listen();
    std::cerr << "Listening on " << listener.LocalAddr()->ToString() << "\n";
    while (true) {
        auto client = co_await listener.Accept();
        client_handler(std::move(client), buf_size);  // detach — no await
    }
}

int main() {
    TLoop<TDefaultPoller> loop;
    serve(loop.Poller(), TAddress{"::", 8888}, 4096);
    loop.Loop();
}
```

### Key Points

- `ReadSome` returns up to `buf_size` bytes — not guaranteed to fill the buffer. Use `TByteReader` when you need exactly N bytes.
- `WriteSome` writes at least 1 byte. Use `TByteWriter` for guaranteed full writes.
- `socket.Accept()` suspends only the `serve` coroutine; all `client_handler` coroutines continue running.

### Run

```sh
./echoserver --port 8888 --method epoll
./echoserver --port 8888 --method uring   # Linux io_uring
./echoserver --port 8888 --method select  # portable fallback
```

---

## Echo Client

### Design

```
client<TPoller>()   TFuture<void>
  ├── TFileHandle{stdin}      async stdin (fd 0)
  ├── TSocket                 TCP connection to server
  ├── TLineReader             newline-delimited reads from stdin
  ├── TByteWriter(socket)     write exact line bytes to server
  └── TByteReader(socket)     read exact echoed bytes back
```

### Core Logic

```cpp
template<typename TPoller>
TFuture<void> client(TPoller& poller, TAddress addr) {
    static constexpr int maxLine = 4096;
    using TSocket     = typename TPoller::TSocket;
    using TFileHandle = typename TPoller::TFileHandle;
    std::vector<char> in(maxLine);

    TFileHandle stdin_fd{0, poller};
    TSocket socket{poller, addr.Domain()};
    TLineReader lineReader(stdin_fd, maxLine);
    TByteWriter byteWriter(socket);
    TByteReader byteReader(socket);

    // 1 s timeout — required on Windows to detect "connection refused"
    co_await socket.Connect(addr, TClock::now() + std::chrono::milliseconds(1000));

    while (auto line = co_await lineReader.Read()) {
        co_await byteWriter.Write(line);                    // send full line
        co_await byteReader.Read(in.data(), line.Size());   // receive same # bytes
        std::cout << std::string_view(in.data(), line.Size());
    }
    co_return;
}

int main() {
    TLoop<TDefaultPoller> loop;
    auto h = client(loop.Poller(), TAddress{"127.0.0.1", 8888});
    while (!h.done()) loop.Step();
}
```

### Key Points

- `TLineReader.Read()` returns a `TLine` — a view into a circular buffer that may be split into two spans (`Part1`, `Part2`). `line.Size()` is the total length including the newline.
- `TByteWriter.Write(line)` and `TByteReader.Read(buf, n)` loop internally until the exact byte count is transferred.
- The client uses `TFuture<void>` (not `TVoidTask`) so the caller can drive the loop with `while (!h.done()) loop.Step()`.

### Run

```sh
./echoserver --port 8888 &
./echoclient --port 8888 --addr 127.0.0.1
# type a line, press Enter, see it echoed back
```

---

## Backend Selection

Both programs accept `--method`:

| `--method` | Type | Platform |
|---|---|---|
| `select` | `TSelect` | All |
| `poll` | `TPoll` | All |
| `epoll` | `TEPoll` | Linux |
| `uring` | `TUring` | Linux + liburing |
| `kqueue` | `TKqueue` | macOS / FreeBSD |
| `iocp` | `TIOCp` | Windows |

The coroutine code is identical across all backends — only the `TPoller` template parameter changes.

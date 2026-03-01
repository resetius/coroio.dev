---
layout: default
title: "Get Started"
permalink: /getting-started/
---

# Get Started

## Requirements

- C++23 compiler: GCC 13+, Clang 16+, MSVC 19.35+
- CMake 3.22+
- Optional: `liburing` (io_uring on Linux), OpenSSL (SSL/TLS)

## Build

```sh
git clone https://github.com/resetius/coroio.git
cd coroio
cmake -B build -DCOROIO_BUILD_EXAMPLES=ON
cmake --build build
```

Run the bundled examples:

```sh
./build/examples/echoserver --port 8888 --method epoll &
./build/examples/echoclient --port 8888 --addr 127.0.0.1 --method epoll
```

## Using as a CMake Dependency

```cmake
add_subdirectory(coroio)
target_link_libraries(my_target PRIVATE coroio)
```

Then include the master header:

```cpp
#include <coroio/all.hpp>
```

## Choosing a Backend

| Type | Backend | Platform |
|---|---|---|
| `TSelect` | select(2) | All |
| `TPoll` | poll(2) | All |
| `TEPoll` | epoll | Linux |
| `TUring` | io_uring | Linux (liburing required) |
| `TKqueue` | kqueue | macOS / FreeBSD |
| `TIOCp` | IOCP | Windows |
| `TDefaultPoller` | best available | current platform |

Backends are swapped by changing the template parameter — no other code changes:

```cpp
TLoop<TEPoll>   loop;   // Linux epoll
TLoop<TUring>   loop;   // Linux io_uring
TLoop<TKqueue>  loop;   // macOS / FreeBSD
TLoop<TSelect>  loop;   // portable fallback
```

## First Program — Repeating Timer

```cpp
#include <coroio/all.hpp>
using namespace NNet;

TVoidTask ticker(TDefaultPoller& poller) {
    for (int i = 0; ; ++i) {
        co_await poller.Sleep(std::chrono::milliseconds(500));
        std::cout << "tick " << i << "\n";
    }
}

int main() {
    TLoop<TDefaultPoller> loop;
    ticker(loop.Poller());
    loop.Loop();
}
```

`Sleep` suspends only the `ticker` coroutine. The event loop continues processing other I/O and timers while it waits.

## Reading Lines from stdin

```cpp
TFuture<void> readLines(TDefaultPoller& poller) {
    TFileHandle stdin_fd{0, poller};
    TLineReader lines(stdin_fd, 4096);

    while (auto line = co_await lines.Read()) {
        // line.Part1 + line.Part2 form the full line
        // (circular buffer may split across the boundary)
        std::string s = std::string(line.Part1) + std::string(line.Part2);
        std::cout << "Got: " << s;
    }
    co_return;
}
```

## Reading Exactly N Bytes

```cpp
TFuture<void> protocol(typename TPoller::TSocket& socket) {
    TByteReader reader(socket);
    TByteWriter writer(socket);

    // Read a 4-byte length-prefixed message
    uint32_t length = 0;
    co_await reader.Read(&length, sizeof length);

    std::vector<char> body(length);
    co_await reader.Read(body.data(), length);

    // Echo it back
    co_await writer.Write(&length, sizeof length);
    co_await writer.Write(body.data(), length);
    co_return;
}
```

## Running a Task and Waiting for It

```cpp
TFuture<void> task = myCoroutine(loop.Poller());
while (!task.done()) {
    loop.Step();
}
// Exceptions (if any) are rethrown here:
// task.get();  // not shown; check TFuture API for your version
```

Use `loop.Loop()` (runs until no more tasks) or `loop.Step()` (single poll iteration) depending on your application structure.

## Next Steps

- [Echo client/server walkthrough](/echoclientserver/)
- [Networking — DNS, HTTP, WebSocket, SSL/TLS, Subprocess](/networking/)
- [Actor system](/actors/)
- [Benchmarks](/benchmarks/)

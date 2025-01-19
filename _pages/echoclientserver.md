---
layout: default
title: "Echo Client/Server"
permalink: /echoclientserver/
---

# Echo Client/Server

This page demonstrates simple echo client and server examples using the asynchronous I/O
and networking library.

For more details and the full source code, see:
- [Echo Client Source](https://github.com/resetius/coroio/blob/master/examples/echoclient.cpp)
- [Echo Server Source](https://github.com/resetius/coroio/blob/master/examples/echoserver.cpp)

---

## Echo Client Example

The following is a simplified echo client program. It reads lines from standard input,
connects to an echo server, sends the data, and then prints the echoed response.

```cpp
#include <coroio/all.hpp>
#include <signal.h>
#include <cstring>
#include <cstdlib>
#include <iostream>

using namespace NNet;

template<bool debug, typename TPoller>
TFuture<void> client(TPoller& poller, TAddress addr)
{
    static constexpr int maxLineSize = 4096;
    using TSocket = typename TPoller::TSocket;
    using TFileHandle = typename TPoller::TFileHandle;
    std::vector<char> in(maxLineSize);

    try {
        TFileHandle input{0, poller}; // stdin
        TSocket socket{poller, addr.Domain()};
        TLineReader lineReader(input, maxLineSize);
        TByteWriter byteWriter(socket);
        TByteReader byteReader(socket);

        // Windows requires a timeout to detect 'connection refused'
        co_await socket.Connect(addr, TClock::now() + std::chrono::milliseconds(1000));
        while (auto line = co_await lineReader.Read()) {
            co_await byteWriter.Write(line);
            co_await byteReader.Read(in.data(), line.Size());
            if constexpr(debug) {
                std::cout << "Received: " << std::string_view(in.data(), line.Size()) << "\n";
            }
        }
    } catch (const std::exception& ex) {
        std::cout << "Exception: " << ex.what() << "\n";
    }

    co_return;
}

template<typename TPoller>
void run(bool debug, TAddress address)
{
    TLoop<TPoller> loop;
    TFuture<void> h;
    if (debug) {
        h = client<true>(loop.Poller(), std::move(address));
    } else {
        h = client<false>(loop.Poller(), std::move(address));
    }
    while (!h.done()) {
        loop.Step();
    }
}

void usage(const char* name) {
    std::cerr << name << " [--port 80] [--addr 127.0.0.1] [--method select|poll|epoll|kqueue] [--debug] [--help]" << std::endl;
    std::exit(1);
}

int main(int argc, char** argv) {
    TInitializer init;
    std::string addr = "127.0.0.1";
    int port = 0;
    std::string method = "select";
    bool debug = false;
    for (int i = 1; i < argc; i++) {
        if (!strcmp(argv[i], "--port") && i < argc-1) {
            port = atoi(argv[++i]);
        } else if (!strcmp(argv[i], "--addr") && i < argc-1) {
            addr = argv[++i];
        } else if (!strcmp(argv[i], "--method") && i < argc-1) {
            method = argv[++i];
        } else if (!strcmp(argv[i], "--debug")) {
            debug = true;
        } else if (!strcmp(argv[i], "--help")) {
            usage(argv[0]);
        }
    }
    if (port == 0) { port = 8888; }

    TAddress address{addr, port};
    std::cerr << "Method: " << method << "\n";

    if (method == "select") {
        run<TSelect>(debug, address);
    }
    else if (method == "poll") {
        run<TPoll>(debug, address);
    }
#ifdef HAVE_EPOLL
    else if (method == "epoll") {
        run<TEPoll>(debug, address);
    }
#endif
#ifdef HAVE_URING
    else if (method == "uring") {
        run<TUring>(debug, address);
    }
#endif
#ifdef HAVE_KQUEUE
    else if (method == "kqueue") {
        run<TKqueue>(debug, address);
    }
#endif
#ifdef HAVE_IOCP
    else if (method == "iocp") {
        run<TIOCp>(debug, address);
    }
#endif
    else {
        std::cerr << "Unknown method\n";
    }

    return 0;
}
```


## Echo Server Example
Below is a simplified echo server program. It listens on a specified port, accepts incoming connections, and for each client, echoes back any data it receives.

```cpp
#include <coroio/all.hpp>

using NNet::TVoidTask;
using NNet::TAddress;
using NNet::TSelect;
using NNet::TPoll;

#ifdef HAVE_EPOLL
using NNet::TEPoll;
#endif
#ifdef HAVE_URING
using NNet::TUring;
#endif
#ifdef HAVE_KQUEUE
using NNet::TKqueue;
#endif
#ifdef HAVE_IOCP
using NNet::TIOCp;
#endif

template<bool debug, typename TSocket>
TVoidTask client_handler(TSocket socket, int buffer_size) {
    std::vector<char> buffer(buffer_size);
    ssize_t size = 0;
    try {
        while ((size = co_await socket.ReadSome(buffer.data(), buffer_size)) > 0) {
            if constexpr(debug) {
                std::cerr << "Received: " << std::string_view(buffer.data(), size) << "\n";
            }
            co_await socket.WriteSome(buffer.data(), size);
        }
    } catch (const std::exception& ex) {
        std::cerr << "Exception: " << ex.what() << "\n";
    }
    if (size == 0) {
        std::cerr << "Client disconnected\n";
    }
    co_return;
}

template<bool debug, typename TPoller>
TVoidTask server(TPoller& poller, TAddress address, int buffer_size)
{
    typename TPoller::TSocket socket(poller, address.Domain());
    socket.Bind(address);
    socket.Listen();
    std::cerr << "Listening on: " << socket.LocalAddr()->ToString() << std::endl;
    while (true) {
        auto client = co_await socket.Accept();
        client_handler<debug>(std::move(client), buffer_size);
    }
    co_return;
}

template<typename TPoller>
void run(bool debug, TAddress address, int buffer_size)
{
    NNet::TLoop<TPoller> loop;
    if (debug) {
        server<true>(loop.Poller(), std::move(address), buffer_size);
    } else {
        server<false>(loop.Poller(), std::move(address), buffer_size);
    }
    loop.Loop();
}

void usage(const char* name) {
    std::cerr << name << " [--port 80] [--method select|poll|epoll|kqueue] [--debug] [--buffer-size 100000] [--help]" << std::endl;
    std::exit(1);
}

int main(int argc, char** argv) {
    NNet::TInitializer init;
    int port = 0;
    int buffer_size = 128;
    std::string method = "select";
    bool debug = false;
    for (int i = 1; i < argc; i++) {
        if (!strcmp(argv[i], "--port") && i < argc-1) {
            port = atoi(argv[++i]);
        } else if (!strcmp(argv[i], "--method") && i < argc-1) {
            method = argv[++i];
        } else if (!strcmp(argv[i], "--debug")) {
            debug = true;
        } else if (!strcmp(argv[i], "--buffer-size") && i < argc-1) {
            buffer_size = atoi(argv[++i]);
        } else if (!strcmp(argv[i], "--help")) {
            usage(argv[0]);
        }
    }
    if (port == 0) { port = 8888; }
    if (buffer_size == 0) { buffer_size = 128; }
    TAddress address{"::", port};
    std::cerr << "Method: " << method << "\n";
    if (method == "select") {
        run<TSelect>(debug, address, buffer_size);
    }
    else if (method == "poll") {
        run<TPoll>(debug, address, buffer_size);
    }
#ifdef HAVE_EPOLL
    else if (method == "epoll") {
        run<TEPoll>(debug, address, buffer_size);
    }
#endif
#ifdef HAVE_URING
    else if (method == "uring") {
        run<TUring>(debug, address, buffer_size);
    }
#endif
#ifdef HAVE_KQUEUE
    else if (method == "kqueue") {
        run<TKqueue>(debug, address, buffer_size);
    }
#endif
#ifdef HAVE_IOCP
    else if (method == "iocp") {
        run<TIOCp>(debug, address, buffer_size);
    }
#endif
    else {
        std::cerr << "Unknown method\n";
    }
    return 0;
}

```

This page demonstrates both echo client and echo server applications using various pollers.
Feel free to explore the full source code on GitHub:
[Echo Client Source](https://github.com/resetius/coroio/blob/master/examples/echoclient.cpp)
[Echo Server Source](https://github.com/resetius/coroio/blob/master/examples/echoserver.cpp)


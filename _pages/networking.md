---
layout: default
title: "Networking"
permalink: /networking/
---

# Networking

## DNS Resolution

```cpp
#include <coroio/dns/resolver.hpp>
#include <coroio/all.hpp>
using namespace NNet;

TFuture<void> lookup(TDefaultPoller& poller) {
    TResolver resolver(poller);                              // reads /etc/resolv.conf

    auto addrs = co_await resolver.Resolve("example.com");  // A record
    auto v6    = co_await resolver.Resolve("example.com", EDNSType::AAAA);

    // THostPort: literal IPs bypass DNS; hostnames go through TResolver
    TAddress addr = co_await THostPort("example.com", 443).Resolve(resolver);

    co_return;
}
```

`TResolver` caches results and deduplicates in-flight queries — concurrent `Resolve()` calls for the same name share a single UDP packet.

Resolve multiple names in parallel:

```cpp
TFuture<void> bulkResolve(TDefaultPoller& poller) {
    TResolver resolver(poller);
    std::vector<std::string> names = {"a.example.com", "b.example.com", "c.example.com"};

    // All queries are sent concurrently; results arrive in any order
    std::vector<TFuture<std::vector<TAddress>>> futures;
    for (auto& name : names)
        futures.push_back(resolver.Resolve(name));

    auto results = co_await All(std::move(futures));
    co_return;
}
```

---

## HTTP Server

```cpp
#include <coroio/http/httpd.hpp>
using namespace NNet;

struct TMyRouter : public IRouter {
    TFuture<void> HandleRequest(TRequest& req, TResponse& resp) override {
        if (req.Method() == "GET" && req.Uri().Path() == "/hello") {
            resp.SetStatus(200);
            resp.SetHeader("Content-Type", "text/plain");
            co_await resp.WriteBodyFull("Hello, World!\n");
        } else if (req.Method() == "POST" && req.Uri().Path() == "/echo") {
            auto body = co_await req.ReadBodyFull();
            resp.SetStatus(200);
            resp.SetHeader("Content-Type", "application/octet-stream");
            co_await resp.WriteBodyFull(body);
        } else {
            resp.SetStatus(404);
            co_await resp.WriteBodyFull("Not found\n");
        }
        co_return;
    }
};

int main() {
    TLoop<TDefaultPoller> loop;
    TMyRouter router;

    using TSocket = TDefaultPoller::TSocket;
    TSocket sock(loop.Poller(), AF_INET6);
    sock.Bind(TAddress{"::", 8080});
    sock.Listen();

    TWebServer<TSocket> server(std::move(sock), router);
    server.Start();
    loop.Loop();
}
```

### Request API

| Method | Returns | Description |
|---|---|---|
| `req.Method()` | `string_view` | HTTP method (`"GET"`, `"POST"`, …) |
| `req.Uri().Path()` | `string_view` | URL path |
| `req.Uri().Query()` | `string_view` | Query string (without `?`) |
| `req.Headers()` | `map<sv,sv>` | All request headers |
| `req.HasBody()` | `bool` | Non-zero `Content-Length` or chunked |
| `co_await req.ReadBodyFull()` | `string` | Read entire body |
| `co_await req.ReadBodySome(buf, n)` | `ssize_t` | Read body chunk |
| `req.BodyConsumed()` | `bool` | True once body is fully read |

### Response API

| Method | Description |
|---|---|
| `resp.SetStatus(code)` | HTTP status code (default 200) |
| `resp.SetHeader(name, value)` | Add response header |
| `co_await resp.SendHeaders()` | Flush status + headers |
| `co_await resp.WriteBodyFull(data)` | Set `Content-Length`, send headers + body |
| `co_await resp.WriteBodyChunk(data, n)` | `Transfer-Encoding: chunked` |

`WriteBodyFull` implies `SendHeaders`. `SetStatus`/`SetHeader` must be called before either.

### Streaming Response

```cpp
TFuture<void> HandleRequest(TRequest& req, TResponse& resp) override {
    resp.SetStatus(200);
    resp.SetHeader("Content-Type", "text/event-stream");
    co_await resp.SendHeaders();                  // headers only, no Content-Length

    for (int i = 0; i < 10; ++i) {
        std::string chunk = "data: " + std::to_string(i) + "\n\n";
        co_await resp.WriteBodyChunk(chunk.data(), chunk.size());
    }
    co_return;
}
```

---

## WebSocket Client

```cpp
#include <coroio/ws/ws.hpp>
#include <coroio/dns/resolver.hpp>
using namespace NNet;

TFuture<void> wsClient(TDefaultPoller& poller) {
    TResolver resolver(poller);
    TAddress addr = co_await THostPort("ws.kraken.com", 443).Resolve(resolver);

    TDefaultPoller::TSocket tcp(poller, addr.Domain());

    // wss:// — wrap with TLS
    TSslContext ctx = TSslContext::Client();
    TSslSocket ssl(std::move(tcp), ctx);
    ssl.SslSetTlsExtHostName("ws.kraken.com");    // SNI, must be before Connect
    co_await ssl.Connect(addr);

    TWebSocket ws(ssl);
    co_await ws.Connect("ws.kraken.com", "/v2");   // HTTP Upgrade

    // Subscribe to BTC/USD ticker
    co_await ws.SendText(
        R"({"method":"subscribe","params":{"channel":"ticker","symbol":["BTC/USD"]}})");

    while (true) {
        auto msg = co_await ws.ReceiveText();      // string_view, valid until next call
        std::cout << msg << "\n";
    }
    co_return;
}
```

### TWebSocket API

| Method | Description |
|---|---|
| `co_await ws.Connect(host, path)` | HTTP Upgrade handshake |
| `co_await ws.SendText(sv)` | Send a text frame (client masking per RFC 6455) |
| `co_await ws.ReceiveText()` | Receive next text frame |

Client-side only. Binary frames not supported. See [market data example](/marketdata/) for a working `wss://` demo.

---

## SSL/TLS

```cpp
#include <coroio/ssl.hpp>
using namespace NNet;

// ── Client ───────────────────────────────────────────────────────────────────
TSslContext ctx = TSslContext::Client();
TSslSocket ssl(std::move(tcpSocket), ctx);
ssl.SslSetTlsExtHostName("example.com");   // SNI — before Connect
co_await ssl.Connect(addr);

ssize_t n = co_await ssl.ReadSome(buf, sizeof buf);
co_await ssl.WriteSome(buf, n);

// ── Server (PEM files) ───────────────────────────────────────────────────────
TSslContext ctx = TSslContext::Server("server.crt", "server.key");

// ── Server (in-memory PEM) ───────────────────────────────────────────────────
TSslContext ctx = TSslContext::ServerFromMem(certPemPtr, keyPemPtr);

// In the accept loop:
auto raw = co_await listener.Accept();
TSslSocket<decltype(raw)> ssl(std::move(raw), ctx);
co_await ssl.AcceptHandshake();
// ssl is now ready for ReadSome / WriteSome
```

`TSslSocket<T>` has exactly the same `ReadSome`/`WriteSome` interface as a plain socket. Swap it in without touching the rest of your I/O logic.

Full example: [sslechoserver.cpp](https://github.com/resetius/coroio/blob/master/examples/sslechoserver.cpp) · [sslechoclient.cpp](https://github.com/resetius/coroio/blob/master/examples/sslechoclient.cpp)

---

## Subprocess / Pipe

Available on Linux and macOS.

```cpp
#include <coroio/pipe/pipe.hpp>
using namespace NNet;

TFuture<void> compress(TDefaultPoller& poller) {
    TPipe pipe(poller, {"gzip", "-c"});          // spawn gzip

    std::string input = "hello world\n";
    co_await pipe.WriteSome(input.data(), input.size());
    pipe.CloseWrite();                            // send EOF to child's stdin

    char buf[4096]; ssize_t n;
    while ((n = co_await pipe.ReadSome(buf, sizeof buf)) > 0)
        std::cout.write(buf, n);                  // compressed bytes

    int exit_code = co_await pipe.Wait();
    std::cout << "gzip exited: " << exit_code << "\n";
    co_return;
}
```

### TPipe API

| Method | Description |
|---|---|
| `co_await pipe.WriteSome(buf, n)` | Write to child's stdin |
| `co_await pipe.ReadSome(buf, n)` | Read from child's stdout |
| `co_await pipe.ReadSomeErr(buf, n)` | Read from child's stderr |
| `pipe.CloseWrite()` | Close stdin — sends EOF to child |
| `co_await pipe.Wait()` | `waitpid()` — returns exit status |
| `pipe.Pid()` | Child PID |

Pass `stderrToStdout = true` to the constructor to merge stderr into stdout (like `2>&1`).

Full example: [test_pipe.cpp](https://github.com/resetius/coroio/blob/master/tests/test_pipe.cpp)

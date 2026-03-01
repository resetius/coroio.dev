---
layout: default
title: "Market Data Feed"
permalink: /marketdata/
---

# Market Data Feed via WebSocket

Live demo — connecting to `wss://ws.kraken.com/v2` and subscribing to real-time ticker data for BTC/USD, XRP/USD, ETH/USD, and DOGE/USD:

<video style="display: block; margin: 0 auto; width: 100%;" autoplay muted loop playsinline>
  <source src="/video/marketdata.mp4" type="video/mp4">
  Your browser does not support the video tag.
</video>

---

## How It Works

```
TResolver                  DNS: ws.kraken.com → IP
    │
TSocket.Connect(addr)      TCP connection
    │
TSslContext::Client()
TSslSocket.Connect(addr)   TLS handshake (wss://)
    │
TWebSocket.Connect(...)    HTTP Upgrade to WebSocket
    │
ws.SendText(subscribe)     JSON subscription command
    │
ws.ReceiveText() loop      incoming ticker frames
```

Full source: [wsclient.cpp](https://github.com/resetius/coroio/blob/master/examples/wsclient.cpp)

---

## Complete Example

```cpp
#include <coroio/all.hpp>
#include <coroio/ws/ws.hpp>
#include <coroio/dns/resolver.hpp>
using namespace NNet;

// Read loop runs concurrently with the send loop
template<typename TWebSocket>
TFuture<void> readLoop(TWebSocket& ws) {
    while (true) {
        auto msg = co_await ws.ReceiveText();   // string_view, valid until next call
        std::cerr << msg << "\n";
    }
    co_return;
}

template<typename TPoller>
TFuture<void> marketData(TPoller& poller) {
    // 1. Resolve hostname
    TResolver resolver(poller);
    TAddress addr = co_await THostPort("ws.kraken.com", 443).Resolve(resolver);

    // 2. Open TCP connection
    typename TPoller::TSocket tcp(poller, addr.Domain());

    // 3. Wrap with TLS (wss://)
    TSslContext ctx = TSslContext::Client();
    TSslSocket ssl(std::move(tcp), ctx);
    ssl.SslSetTlsExtHostName("ws.kraken.com");   // SNI — must be before Connect
    co_await ssl.Connect(addr);

    // 4. WebSocket handshake
    TWebSocket ws(ssl);
    co_await ws.Connect("ws.kraken.com", "/v2");

    // 5. Subscribe
    co_await ws.SendText(R"({
        "method": "subscribe",
        "params": {
            "channel": "ticker",
            "symbol": ["BTC/USD", "XRP/USD", "ETH/USD", "DOGE/USD"]
        }
    })");

    // 6. Receive loop
    co_await readLoop(ws);
    co_return;
}

int main() {
    TInitializer init;
    TLoop<TDefaultPoller> loop;
    auto h = marketData(loop.Poller());
    loop.Loop();
}
```

---

## Plain WebSocket (`ws://`)

For unencrypted connections, skip TLS and connect the raw socket directly:

```cpp
typename TPoller::TSocket tcp(poller, addr.Domain());
co_await tcp.Connect(addr);

TWebSocket ws(tcp);
co_await ws.Connect("echo.websocket.org", "/");
```

---

## Reading Lines from stdin and Sending

The full `wsclient.cpp` example also reads from stdin and sends each line as a text frame — useful for interactive WebSocket sessions:

```cpp
TFileHandle stdin_fd{0, poller};
TLineReader lineReader(stdin_fd, 4096);

auto receiver = readLoop(ws);          // runs concurrently

while (auto line = co_await lineReader.Read()) {
    std::string s = std::string(line.Part1) + std::string(line.Part2);
    co_await ws.SendText(std::string_view(s));
}

co_await receiver;
```

---

## TWebSocket API

| Method | Description |
|---|---|
| `co_await ws.Connect(host, path)` | HTTP Upgrade handshake |
| `co_await ws.SendText(sv)` | Send text frame (auto-masked per RFC 6455) |
| `co_await ws.ReceiveText()` | Receive next text frame; `string_view` valid until next call |

Client-side only. Binary frames not supported.

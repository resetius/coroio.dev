---
layout: default
title: "MarketData Feed"
permalink: /marketdata/
---

# Using a WebSocket Client to Consume Market Data

In the video below, we connect to the public Kraken WebSocket endpoint: wss://ws.kraken.com/v2

We then send the following subscription command to receive real-time ticker information for BTC/USD, XRP/USD, ETH/USD, and DOGE/USD:

<video  style="display: block; margin: 0 auto; width: 100%; " autoplay muted loop playsinline>
  <source src="/video/marketdata.mp4" type="video/mp4">
  Your browser does not support the `video` tag.
</video>

## Minimal Example with coroio

[COROIO](https://github.com/resetius/coroio) includes a WebSocket client that supports both `ws://` and `wss://` protocols. Below is a simple example usage:

```cpp
TWebSocket ws(socket);

// Connect to the Kraken WebSocket endpoint
co_await ws.Connect("ws.kraken.com", "/v2");

// Send a subscription request
ws.SendText(R"({"method":"subscribe","params":{"channel":"ticker","symbol":["BTC/USD","XRP/USD","ETH/USD","DOGE/USD"]}})");

// Read incoming frames
auto data = co_await ws.ReceiveText();
std::cout << "Received message: " << data << std::endl;
```

For full implementation details, see the [WebSocket class in coroio](https://github.com/resetius/coroio/blob/master/coroio/ws.hpp).
To explore a complete working example, check out [wsclient.cpp](https://github.com/resetius/coroio/blob/master/examples/wsclient.cpp).


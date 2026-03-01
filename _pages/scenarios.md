---
layout: default
title: "Examples"
permalink: /scenarios/
---

# Examples

## I/O & Networking

**[Get Started](/getting-started/)**
Build, install, backend selection, first coroutine program, reading lines and bytes.

**[Echo Client/Server](/echoclientserver/)**
Async TCP echo server with per-client `TVoidTask` handlers. Echo client using `TLineReader`, `TByteWriter`, `TByteReader`. Backend selection walkthrough.

**[Market Data Feed](/marketdata/)**
Full `wss://` WebSocket client: DNS → TCP → TLS → WebSocket upgrade → subscribe → read loop. Kraken real-time ticker demo.

**[Networking — DNS, HTTP, WebSocket, SSL/TLS, Subprocess](/networking/)**
Recipes for every networking primitive:
- Async DNS with `TResolver` / `THostPort`
- HTTP server with `TWebServer` + `IRouter`
- WebSocket client (`TWebSocket`)
- TLS client and server (`TSslSocket`)
- Subprocess with async stdin/stdout (`TPipe`)

## Actor System

**[Actor System](/actors/)**
All three actor styles (`IActor`, `ICoroActor`, `IBehaviorActor + TBehavior`) with code examples. `TActorContext` API reference. Local and distributed setup. POD vs non-POD message serialization. Benchmarks.

## Benchmarks

**[I/O Benchmarks vs libevent](/benchmarks/)**
echo-server throughput on epoll / kqueue / IOCP / select across Linux, macOS, and Windows.

## Real-World Usage

- **[miniraft-cpp](https://github.com/resetius/miniraft-cpp)** — Raft consensus algorithm built on COROIO actors
- **[Distributed SQL](https://www.linkedin.com/posts/alexey-ozeritskiy_its-been-a-while-since-i-last-wrote-about-activity-7278402000516485120-Ibp0)** — distributed SQL database using COROIO for networking and [miniraft-cpp](https://github.com/resetius/miniraft-cpp) for consensus
- **[Qumir](https://qumir.dev)** — online compiler for the Kumir language; COROIO as the web server

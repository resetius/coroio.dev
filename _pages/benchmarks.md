---
layout: default
title: Benchmarks
permalink: /benchmarks/
---

# Benchmarks

## I/O Benchmarks vs libevent

Methodology from [libevent](https://libevent.org). Two tests:

1. **Single active connection** — measures overhead per event, shows scalability limits of select/poll.
2. **100 chained connections** — each connection chains a write to the next until 1000 writes have occurred, exercising the event loop heavily.

### Intel i7-12800H · Ubuntu 23.04 · Clang 16

<img src="https://github.com/resetius/coroio/blob/master/bench/bench_12800H.png?raw=true" width="800" alt="i7-12800H single connection"/>
<img src="https://github.com/resetius/coroio/blob/master/bench/bench_12800H_100.png?raw=true" width="800" alt="i7-12800H 100 connections"/>

### Intel i5-11400F · Ubuntu 23.04 · WSL2 (kernel 6.1.21.1-microsoft-standard-WSL2+)

<img src="https://github.com/resetius/coroio/blob/master/bench/bench_11400F.png?raw=true" width="800" alt="i5-11400F single connection"/>
<img src="https://github.com/resetius/coroio/blob/master/bench/bench_11400F_100.png?raw=true" width="800" alt="i5-11400F 100 connections"/>

### Apple M1 · MacBook Air 16G · macOS 12.6.3

<img src="https://github.com/resetius/coroio/blob/master/bench/bench_M1.png?raw=true" width="800" alt="M1 single connection"/>
<img src="https://github.com/resetius/coroio/blob/master/bench/bench_M1_100.png?raw=true" width="800" alt="M1 100 connections"/>

---

## Actor Benchmarks

Ring topology: N actors on a ring, each forwarding a message to the next. Seed messages are not counted toward throughput.

Hardware: **i5-11400F · Ubuntu 25.04**

### Local Ring — 100 actors, batch 1024

| Framework | msg/s |
|---|---|
| Akka | 473,966 |
| **COROIO** | **442,151** |
| CAF | 302,930 |
| YDB/actors | 151,972 |

### Distributed Ring — 10 processes, batch 1024, payload 0 bytes

| Framework | msg/s |
|---|---|
| **COROIO** | **1,137,790** |
| YDB/actors | 182,525 |
| CAF | 55,540 |
| Akka | 5,765 |

### Distributed Ring — 10 processes, batch 1024, payload 1024 bytes

| Framework | msg/s |
|---|---|
| **COROIO** | **860,188** |
| YDB/actors | 96,372 |

Benchmark source: [ping_actors.cpp](https://github.com/resetius/coroio/blob/master/examples/ping_actors.cpp)

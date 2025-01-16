---
layout: default
title: Benchmarks
permalink: /benchmarks/
---

# Benchmarks

## Methodology

Our benchmark methodology is based on the approach used by the [libevent library](https://libevent.org). We conduct two types of benchmarks:

1. **Single Active Connection**: Measures the time taken to serve one active connection, highlighting scalability issues in traditional interfaces like select or poll.

2. **Multiple Active Connections**: Measures the time taken to serve 100 active connections that chain writes to new connections until 1000 writes and reads have occurred, exercising the event loop multiple times.

## Performance Comparison

We compared the performance of different event notification mechanisms in Libevent and COROIO across various platforms:

### Intel i7-12800H (Ubuntu 23.04)

- OS: Ubuntu 23.04
- Compiler: Clang 16
- Libevent version: master 4c993a0e7bcd47b8a56514fb2958203f39f1d906 (Tue Apr 11 04:44:37 2023 +0000)

<img src="https://github.com/resetius/coroio/blob/master/bench/bench_12800H.png?raw=true" width="800" alt="i7-12800H Single Connection Benchmark"/>
<img src="https://github.com/resetius/coroio/blob/master/bench/bench_12800H_100.png?raw=true" width="800" alt="i7-12800H Multiple Connections Benchmark"/>

### Intel i5-11400F (Ubuntu 23.04, WSL2)

- OS: Ubuntu 23.04, WSL2
- Kernel: 6.1.21.1-microsoft-standard-WSL2+

<img src="https://github.com/resetius/coroio/blob/master/bench/bench_11400F.png?raw=true" width="800" alt="i5-11400F Single Connection Benchmark"/>
<img src="https://github.com/resetius/coroio/blob/master/bench/bench_11400F_100.png?raw=true" width="800" alt="i5-11400F Multiple Connections Benchmark"/>

### Apple M1 (MacOS 12.6.3)

- Device: MacBook Air M1 16G
- OS: MacOS 12.6.3

<img src="https://github.com/resetius/coroio/blob/master/bench/bench_M1.png?raw=true" width="800" alt="Apple M1 Single Connection Benchmark"/>
<img src="https://github.com/resetius/coroio/blob/master/bench/bench_M1_100.png?raw=true" width="800" alt="Apple M1 Multiple Connections Benchmark"/>

These benchmarks demonstrate the performance characteristics of [COROIO](https://github.com/resetius/coroio) compared to Libevent across different hardware and operating systems. For detailed analysis of the results, please refer to the graphs above.


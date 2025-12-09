---
layout: default
title: "COROIO: Efficient Asynchronous I/O Library for Modern C++"
---

# COROIO: Efficient Asynchronous I/O Library for Modern C++

[COROIO](https://github.com/resetius/coroio) is a modern C++ library that leverages coroutines for efficient asynchronous programming, offering non-blocking I/O operations across multiple platforms.

Quick links:
- Top-level README: https://github.com/resetius/coroio/blob/master/README.md
- Actors documentation: https://github.com/resetius/coroio/blob/master/coroio/actors/README.md

## Key Features

- **Coroutines**: Write asynchronous code in a clean and readable manner
- **Actor Model**: Message-passing concurrency with typed behaviors and async processing
- **Efficient Polling Mechanisms**: Support for various event notification systems, including:
  - select
  - epoll (Linux)
  - kqueue (FreeBSD, macOS)
  - IOCP (Windows)
- **Cross-Platform Compatibility**: Works on Linux, FreeBSD, macOS, and Windows
- **Efficient Socket and File Operations**: Optimized for network and file I/O tasks
- **Comprehensive Networking Support**:
  - Basic functionality for network interactions
  - Asynchronous DNS resolver
  - SSL/TLS layer for secure communications
  - WebSocket support for real-time, full-duplex communication

## Why Choose COROIO?

- **Simplicity**: Leverage coroutines for clear and concise asynchronous code
- **Efficiency**: Efficient I/O operations and event handling
- **Versatility**: Develop network applications and servers with ease
- **Future-Proof**: Built on modern C++ standards, ensuring long-term relevance and compatibility
- **Rich Feature Set**: From low-level socket operations to high-level protocols like WebSockets

Explore [COROIO](https://github.com/resetius/coroio) today and enhance your asynchronous programming experience!

[COROIO](https://github.com/resetius/coroio) is under active development, and we welcome feedback and contributions from the community.

For source code and implementation details, visit our [GitHub repository](https://github.com/resetius/coroio).

## Projects Using COROIO

Several open-source projects leverage COROIO for asynchronous I/O and actor-based concurrency:

- [miniraft-cpp](https://github.com/resetius/miniraft-cpp) — minimalist Raft implementation in C++
- [Qumir (online Kumir compiler)](https://qumir.dev) ([GitHub](https://github.com/resetius/qumir), [Website](https://qumir.dev))

## Dependencies (minimal by design)

The library is intentionally simple and has minimal dependencies:
- Core modules have no external dependencies beyond the C++ standard library and the OS APIs.
- Optional components:
  - io_uring support on Linux uses liburing (optional).
  - SSL/TLS uses OpenSSL (optional).
  - Other features (like WebSocket) are layered on top of the core building blocks.


# Contributing to web-stream reference implementation

Thank you for your interest in contributing! This document provides information on how to run unit tests and verify your changes.

## Running Unit Tests

Googletest suites validate handshakes and packet stream framing correctness. After compilation finishes, execute the test runners inside the [build/](build/) library path:

```bash
ctest --test-dir build --output-on-failure
```

Or run individual binaries directly:

- HTTP/1.1 Chunked Framing Test:
  ```bash
  ./build/buffer_event_web_stream_test
  ```
- HTTP/2 Layering Test:
  ```bash
  ./build/nghttp2_web_stream_test
  ```
- Handshake Validation Test:
  ```bash
  ./build/handshake_test
  ```

# Code Map: web-stream Reference Implementation

This document provides a comprehensive structural guide to the directories, libraries, interfaces, and source code files contained in this repository. 

## Architectural Overview

The reference implementation of the `web-stream` protocol (`draft-yoshino-wish`) is built as a highly modular C++17 library integrating:
1. **[`libevent`](https://libevent.org/)** for asynchronous I/O and event loop handling.
2. **[`wslay`](https://github.com/tatsuhiro-t/wslay)** for building and parsing WebSocket-like stream frames (custom-tailored to remove frame masking and accept new opcodes).
3. **[`picohttpparser`](https://github.com/h2o/picohttpparser)** for robust HTTP/1.1 handshake header parsing.
4. **[`nghttp2`](https://github.com/nghttp2/nghttp2)** for core HTTP/2 framing.
5. **[BoringSSL](https://boringssl.googlesource.com/boringssl)** for TLS connection security (replacing standard OpenSSL/Mbed TLS).

There are two primary modes of transport supported:
- **HTTP/1.1 Chunked Transfer** (re-using `libevent` `bufferevent` wrapper layer).
- **HTTP/2 Streaming** (binding `wslay` framed segments directly into-and-out-of `nghttp2` DATA frames).

---

## Core Interfaces & Opcode Definitions

- [src/web_stream.h](src/web_stream.h)
  The abstract interface defining the core lifecycle of a `web-stream` transaction. Both `BufferEventWebStream` (HTTP/1.1) and `NGHTTP2WebStream` (HTTP/2) implement this interface. Defines callbacks:
  - `OnMessage`: Invoked when a complete stream message arrives.
  - `OnClose`: Clean EOF closed at a message boundary.
  - `OnError`: Fired during standard errors or premature shutdowns.
  - Senders: `SendText`, `SendBinary`, and `SendMetadata`.
- [src/wish_opcodes.h](src/wish_opcodes.h)
  Defines the opcodes mapping to different frames of the `web-stream` specification:
  - `WEB_STREAM_OPCODE_TEXT = 1`
  - `WEB_STREAM_OPCODE_BINARY = 2`
  - `WEB_STREAM_OPCODE_METADATA = 3`

---

## Core Framers & Decoders

These classes are responsible for packing and unpacking packet stream payloads over the appropriate wire formats.

### 1. HTTP/1.1 Chunked (BufferEvent Support)
- [src/buffer_event_web_stream.h](src/buffer_event_web_stream.h) & [src/buffer_event_web_stream.cc](src/buffer_event_web_stream.cc)
  Constructed on top of `libevent` `bufferevent`. Combines:
  - Decoding `Transfer-Encoding: chunked` byte streams inside `ReadChunkedBytes`.
  - Piping the decoded payload to a non-masked, modified `wslay_event_context` parser.
  - Outputting chunked encodings (`0\r\n\r\n` closing etc.) when finishing message output streams.
- [src/handshake.h](src/handshake.h) & [src/handshake.cc](src/handshake.cc)
  Implements the asynchronous HTTP/1.1 client/server `POST` handshake with `picohttpparser`. Standardizes validation of the `Content-Type: application/web-stream` and `Transfer-Encoding: chunked` headers before passing the connection off to the main `BufferEventWebStream` session.

### 2. HTTP/2 Framing (NGHTTP2 Support)
- [src/nghttp2_web_stream.h](src/nghttp2_web_stream.h) & [src/nghttp2_web_stream.cc](src/nghttp2_web_stream.cc)
  Implements stream decoding over HTTP/2 virtual data channels. 
  - `OnDataChunk` buffers raw incoming HTTP/2 DATA frame chunks, feed-parsing them to `wslay`.
  - Senders feed outgoing bytes directly into an output buffer which is lazily drained via nghttp2 data-source read callbacks (`ReadSendData`) linked with `END_STREAM` control flags.

---

## Transport Layers, Clients, and Servers

This repository structures clients and servers cleanly into folders across four underlying network layer configurations:

### 1. Plain Cleartext HTTP/1.1 Chunked
- [src/plain_client.h](src/plain_client.h) & [src/plain_client.cc](src/plain_client.cc)
  Connects to a server, initiates a clean C++ HTTP/1.1 handshake, and hands over to `BufferEventWebStream` on completion.
- [src/plain_server.h](src/plain_server.h) & [src/plain_server.cc](src/plain_server.cc)
  Listens for TCP connections, handles parsing HTTP headers on incoming streams, and constructs standard `BufferEventWebStream` active pipelines.

### 2. Secure TLS HTTP/1.1 Chunked
- [src/tls_context.h](src/tls_context.h) & [src/tls_context.cc](src/tls_context.cc)
  Configures mTLS/TLS connections wrapper with BoringSSL contexts (`SSL_CTX`).
- [src/tls_client.h](src/tls_client.h) & [src/tls_client.cc](src/tls_client.cc)
  Performs handshake over secure OpenSSL/TLS sockets with host verification.
- [src/tls_server.h](src/tls_server.h) & [src/tls_server.cc](src/tls_server.cc)
  A secure TLS listener handling handshake parsing and mTLS client-certificate validation.

### 3. Cleartext h2c (HTTP/2 Cleartext)
- [src/h2_client.h](src/h2_client.h) & [src/h2_client.cc](src/h2_client.cc)
  Establishes plain cleartext HTTP/2 sessions and opens a stream to path `/` with `Content-Type: application/web-stream`.
- [src/h2_server.h](src/h2_server.h) & [src/h2_server.cc](src/h2_server.cc)
  Ingresses incoming connections, routes h2 requests, validates streams headers, and deploys `NGHTTP2WebStream` channels representing active paths.

### 4. TLS HTTP/2 (h2)
- [src/h2_tls_client.h](src/h2_tls_client.h) & [src/h2_tls_client.cc](src/h2_tls_client.cc)
  Establishes HTTP/2 connections over BoringSSL with ALPN matching `h2`.
- [src/h2_tls_server.h](src/h2_tls_server.h) & [src/h2_tls_server.cc](src/h2_tls_server.cc)
  Routes HTTP/2 over TLS connections, supporting both client/server ALPN context negotiation and request streaming.

---

## Patches

- [patches/wslay.patch](patches/wslay.patch)
  Crucial source patch file applied inside CMake's `FetchContent` step before building `wslay`. 
  - Extends `wslay_opcode` to include `WSLAY_METADATA_FRAME = 0x3u`.
  - Bypasses/disables client/server frame masking checks on both receive and transmit.

---

## Unit Tests

Unit tests are written using Googletest (`gtest`) and run to ensure the stream logic holds under mock networks:
- [src/handshake_test.cc](src/handshake_test.cc): Validates HTTP/1.1 handshakes with standard and corrupt header combinations.
- [src/buffer_event_web_stream_test.cc](src/buffer_event_web_stream_test.cc): Performs duplex end-to-end testing of `BufferEventWebStream` chunking.
- [src/nghttp2_web_stream_test.cc](src/nghttp2_web_stream_test.cc): Simulates packet transfers on core `NGHTTP2WebStream` frames.

---

## Examples

Standalone runnable client/server setups representing all protocol variants:
- **Plain (HTTP/1.1 cleartext)**: 
  - Server: [examples/plain_echo_server.cc](examples/plain_echo_server.cc)
  - Client: [examples/plain_hello_client.cc](examples/plain_hello_client.cc)
- **TLS (HTTP/1.1 TLS)**:
  - Server: [examples/tls_echo_server.cc](examples/tls_echo_server.cc)
  - Client: [examples/tls_hello_client.cc](examples/tls_hello_client.cc)
- **H2 (HTTP/2 cleartext)**:
  - Server: [examples/h2_echo_server.cc](examples/h2_echo_server.cc)
  - Client: [examples/h2_hello_client.cc](examples/h2_hello_client.cc)
- **H2 over TLS (HTTP/2 TLS)**:
  - Server: [examples/h2_tls_echo_server.cc](examples/h2_tls_echo_server.cc)
  - Client: [examples/h2_tls_hello_client.cc](examples/h2_tls_hello_client.cc)

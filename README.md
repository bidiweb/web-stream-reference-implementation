# web-stream reference implementation

This repository contains a reference implementation of the `web-stream` codec (over HTTP/1.1 chunked or HTTP/2 DATA framing), satisfying the `web-stream` specifications in https://github.com/bidiweb/wish (`draft-yoshino-wish`).

For a detailed walkthrough of the individual files, classes, and structural organization of this project, please consult the [CODE_MAP.md](CODE_MAP.md) directory guide.

---

## Design Overview

This reference implementation leverages several high-performance open-source networking backends:

- **[`libevent`](https://libevent.org/)**: Handles asynchronous I/O and manages socket event loop operations.
- **[`nghttp2`](https://github.com/nghttp2/nghttp2)**: Implements framing buffers and streams for the HTTP/2 transport mode.
- **[`wslay`](https://github.com/tatsuhiro-t/wslay)**: A lightweight WebSocket codec tweaked to serve `web-stream` framing protocol demands.
  - A custom patch, [patches/wslay.patch](patches/wslay.patch), is automatically applied during compilation. This patch strips WebSocket masking requirements (allowing unmasked framing overhead reduction) and defines a newly requested opcode mapping: `WSLAY_METADATA_FRAME = 0x3u`.
- **[`picohttpparser`](https://github.com/h2o/picohttpparser)**: Used for rapid parsing of HTTP/1.1 handshake headers.
- **[BoringSSL](https://boringssl.googlesource.com/boringssl)**: Provides modern cryptographic security and manages ALPN handshakes over secure transport runs.

---

## Directory Structure

- [src/](src/) contains the core library source code.
  - Architectural interfaces defining actions are located in [src/web_stream.h](src/web_stream.h).
- [patches/](patches/) stores source patching configurations for external dependencies.
- [examples/](examples/) includes fully runnable demonstration programs for each transport scenario.

For a full tour of headers, clients, servers, and helper objects, open [CODE_MAP.md](CODE_MAP.md).

---

## Build Instructions

### Prerequisites

To compile the library, tests, and example applications, ensure you have the required build tools and compilers installed:

```bash
sudo apt install build-essential cmake clang ninja-build
```

The build configurations use BoringSSL, Abseil, libevent, and nghttp2, which are fetched and constructed automatically during the CMake configure step.

### Release Build

To perform a fast, optimized release build using `clang` and `Ninja`:

```bash
cmake \
  -B build \
  -G Ninja \
  -DCMAKE_C_COMPILER=clang \
  -DCMAKE_CXX_COMPILER=clang++ \
  -DCMAKE_BUILD_TYPE=Release && \
  cmake --build build -- -j$(nproc)
```

### Debug Build

To compile a development snapshot with full debug symbols and diagnostic details:

```bash
cmake \
  -B build \
  -G Ninja \
  -DCMAKE_C_COMPILER=clang \
  -DCMAKE_CXX_COMPILER=clang++ \
  -DCMAKE_BUILD_TYPE=Debug && \
  cmake --build build -- -j$(nproc)
```

---

## Running Unit Tests

For details on how to build and execute individual tests or test suites, please see the [CONTRIBUTING.md](CONTRIBUTING.md) guide.

---

## Running the Examples

Run example binaries to experiment with different transport configurations. To display complete diagnostic logs in stdout/stderr, pass `--stderrthreshold=0` to the example processes.

The security certificates needed to run the TLS and HTTP/2 TLS examples are pre-configured and included under the [certs/](certs/) directory.

### 1. Plain HTTP/1.1 (Chunked over Cleartext TCP)

- **Server**:
  ```bash
  ./build/examples/plain_echo_server --stderrthreshold=0 --port=8080
  ```
- **Client**:
  ```bash
  ./build/examples/plain_hello_client --stderrthreshold=0 --host=127.0.0.1 --port=8080
  ```

### 3. TLS HTTP/1.1 (Chunked over Secure TLS TCP with mTLS)

- **Server**:
  ```bash
  ./build/examples/tls_echo_server --stderrthreshold=0 --port=8080 --ca_cert=certs/ca.crt --server_cert=certs/server.crt --server_key=certs/server.key
  ```
- **Client**:
  ```bash
  ./build/examples/tls_hello_client --stderrthreshold=0 --host=127.0.0.1 --port=8080 --ca_cert=certs/ca.crt --client_cert=certs/client.crt --client_key=certs/client.key
  ```

### 4. Plain HTTP/2 (Cleartext h2c Session)

- **Server**:
  ```bash
  ./build/examples/h2_echo_server --stderrthreshold=0 --port=8080
  ```
- **Client**:
  ```bash
  ./build/examples/h2_hello_client --stderrthreshold=0 --host=127.0.0.1 --port=8080
  ```

### 5. TLS HTTP/2 (Secure h2 TLS Session)

- **Server**:
  ```bash
  ./build/examples/h2_tls_echo_server --stderrthreshold=0 --port=8080 --ca_cert=certs/ca.crt --server_cert=certs/server.crt --server_key=certs/server.key
  ```
- **Client**:
  ```bash
  ./build/examples/h2_tls_hello_client --stderrthreshold=0 --host=127.0.0.1 --port=8080 --ca_cert=certs/ca.crt --client_cert=certs/client.crt --client_key=certs/client.key
  ```

---

## Future Goals

- Full support for HTTP/3 and secure QUIC environments is planned.


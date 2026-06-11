# Handoff

## Project State

**Directory:** `/home/clovolc/Work/apig`

## User Requests & Constraints (verbatim)

1. Use single-header only (no FetchContent, no gmock)
2. "if I want to implement my own [HTTP client] how many loc will it take?"
3. "implement basic async i/o, and note for future implementation"
4. No external dependencies beyond what's dropped in `include/`

## Key Decisions

| Decision | Rationale |
|---|---|
| Blocking I/O chosen initially | Simplest to implement, cpp-httplib is a single header |
| Hand-rolled mock (not gmock) | User wanted single-header only; avoids dependency |
| Local server for benchmarking | Public APIs rate-limit; need repeatable results |
| No SSL in current setup | Keeps it minimal; can be added later |

## Measured Performance

| Test | Requests | Duration | Throughput | Limiting factor |
|---|---|---|---|---|
| Mock (in-memory) | 1B | 233s | ~4.3M req/s | CPU (function call overhead) |
| Real HTTP (localhost) | 1M | 87s | ~11.5k req/s | Network I/O (blocking socket) |

## Gap Analysis: Bare-bones vs Production HTTP Client

| Area | Bare-bones (~350 LOC) | Production (3000+ LOC) |
|---|---|---|
| Connection reuse | None | Keep-alive pool, limits |
| Timeouts | None/hardcoded | Connect/read/write/max, configurable |
| Redirects | Manual | Auto-follow, loop detection |
| TLS/SSL | No | OpenSSL/MbedTLS/wolfSSL, cert verify |
| Chunked encoding | Basic | Trailer parsing, framing |
| Compression | No | gzip/brotli/zstd |
| Proxy | No | HTTP/SOCKS, proxy auth |
| Auth | No | Basic/Digest/Bearer/NTLM |
| Multipart | No | File uploads, form data |
| Streaming | No | Callbacks for content |
| Error handling | `if (!res)` | Granular codes, reconnect |
| DNS | `getaddrinfo` | Caching, happy eyeballs |
| HTTP/2 | No | Multiplexing |
| Security | None | Hostname verify, cert pinning |

## Pending Work: Implement Basic Async I/O

Async HTTP client using epoll.

### Architecture

```
EventLoop (epoll_wait)
  ├── Conn-1: SENDING     → epoll writes until EAGAIN → watch EPOLLOUT → ...
  ├── Conn-2: RECEIVING   → epoll reads until EAGAIN  → watch EPOLLIN  → ...
  ├── Conn-3: CONNECTING  → connect() → watch EPOLLOUT → check SO_ERROR
  └── ...
```

### Connection State Machine

```
CLOSED → CONNECTING → SENDING → RECEIVING_HEADERS → RECEIVING_BODY → DONE → CLOSED
                                                              ↓
                                                        KEEP_ALIVE → SENDING (next request)
```

### Implementation Order (Suggested)

1. **Socket RAII** — thin wrapper with non-blocking connect
2. **epoll event loop** — `epoll_create1`, `epoll_ctl`, `epoll_wait` with timeout
3. **HTTP request formatter** — build GET string: `GET /path HTTP/1.1\r\nHost: ...\r\n\r\n`
4. **Response parser** — state machine: status line → headers (key: value) → body (Content-Length or chunked)
5. **Connection pool** — reuse keep-alive connections, max per host
6. **TLS layer** — OpenSSL BIO pair or similar (future)

### What to Build

- `src/async_http_client.h` and `src/async_http_client.cpp`
- epoll wrapper
- HTTP parser
- Connection pool

### Risks & Gotchas

- **epoll is Linux-only** — macOS needs kqueue, Windows needs IOCP. Cross-platform requires abstraction.
- **Non-blocking connect** — `connect()` returns `-1` with `errno == EINPROGRESS`. Wait for `EPOLLOUT`, then check `SO_ERROR`.
- **Partial reads/writes** — always handle short returns and `EAGAIN`. Buffer incomplete data.
- **HTTP parsing pitfalls** — chunked encoding, duplicate headers, `Connection: close`, `Content-Length` vs chunked conflict.
- **Edge-triggered vs level-triggered** — edge-triggered needs full read/write until `EAGAIN`; level-triggered re-fires if not done.
- **Memory** — use ring buffers per connection to avoid reallocation.

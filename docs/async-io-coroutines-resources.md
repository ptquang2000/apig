# Async I/O & Coroutines — Reading List

## Books & Papers

| Resource | Covers |
|---|---|
| **"Efficient IO with io_uring"** (Jens Axboe, 2019) — [kernel.dk/io_uring.pdf](https://kernel.dk/io_uring.pdf) | Ring layout, SQPOLL, fixed buffers — the original design paper |
| **"Advanced Programming in the UNIX Environment"** (Stevens & Rago), Ch.14 | Blocking, non-blocking, signal-driven I/O, epoll, POSIX AIO |
| **"Linux Kernel Development"** (Love), Ch.13 | Block I/O stack, elevator, page cache — background io_uring builds on |
| **"What Every Systems Programmer Should Know About Concurrency"** (Matt Kline) — [pdf](https://assets.bitbashing.io/papers/concurrency-primer.pdf) | Memory ordering, lock-free rings, atomics |
| **P4003R0 — "Coroutines for I/O"** — [open-std.org](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2026/p4003r0) | C++20 coroutine internals, symmetric transfer, frame allocator propagation, TLS write-through cache |
| **P4100R0 — "The Network Endeavor"** — [open-std.org](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2026/p4100r0) | Three-layer I/O architecture (abstract/concrete/native), ABI stability, zero per-op allocation |

## Online Deep-Dives

| Resource | Covers |
|---|---|
| [kernel-internals.org/io-uring/](https://kernel-internals.org/io-uring/) | Architecture, rings, SQE/CQE, life of a request, SQPOLL, fixed buffers, multishot, networking |
| [io-uring vs epoll](https://kernel-internals.org/io-uring/io-uring-vs-epoll/) | Feature comparison, migration patterns, when to keep epoll |
| [io_uring for people who already know epoll](https://shells-angels.de/posts/io-uring-for-epoll-users/) (2026) | Practical translation table, sharp edges, buffer lifetimes, cancellation, setup flags |
| [Linux I/O Evolution Deep Dive](https://www.youngju.dev/blog/culture/2026-04-15-linux-io-evolution-blocking-select-poll-epoll-io-uring-async-deep-dive-guide-2025.en) | blocking → select → poll → epoll → io_uring, full timeline |
| [Complete Guide to Async I/O Models 2025](https://www.youngju.dev/blog/culture/2026-04-15-async-io-models-epoll-io-uring-reactor-guide-2025.en) | Reactor vs Proactor, async/await internals (JS/Python/Rust/Go), performance comparison |
| [Linux I/O Models — blocking, epoll, io_uring](https://systeminternals.dev/linux/io-models/) | Level vs edge-triggered, AIO history, benchmark context, tradeoffs |

## Coroutine / Runtime Internals

| Resource | Covers |
|---|---|
| **libunifex docs** — [github.com/facebookexperimental/libunifex](https://github.com/facebookexperimental/libunifex/blob/main/doc/overview.md) | sender/receiver vs coroutine awaitables, tail-recursion (symmetric transfer), heterogeneous results |
| **CPython `ceval.c`** — `_PyEval_EvalFrameDefault` bytecode loop | `SEND`/`RESUME`/`YIELD_VALUE` opcodes — no assembly, save/restore `PyFrameObject` |
| **Go runtime** — `runtime/proc.go` + `runtime/asm_amd64.s` | `gosave`/`gogo` — ~20-instruction register swap (SP, BP, PC + callee-saved) |
| **C++20 coroutines** — compiler-generated state machine enum | Compiler lowers `co_await` to switch-on-state; no runtime, no hand asm |

## Kernel Source to Read

```
io_uring/io_uring.c        — ring management, setup, enter
io_uring/rw.c              — read/write operations
io_uring/net.c             — accept, send, recv
io_uring/sqpoll.c          — SQPOLL kernel thread
io_uring/io-wq.c           — async worker thread pool
io_uring/rsrc.c            — fixed buffers/files
fs/eventpoll.c             — epoll core (ready-list swap, ep_poll_callback)
include/uapi/linux/io_uring.h  — ABI: SQE/CQE structs, opcodes, flags
```

## Libraries to Study

- **liburing** — [github.com/axboe/liburing](https://github.com/axboe/liburing) — official userspace library
- **tokio-uring** — Rust integration with design doc
- **Boost.Asio** — reference proactor implementation (epoll/IOCP/kqueue backend)
- **libuv** — Node.js foundation, cross-platform (epoll/kqueue/IOCP)

## Classic Reference

- **"The C10K Problem"** (Dan Kegel, 1999–2014) — [kegel.com/c10k.html](http://www.kegel.com/c10k.html)
- **"The Secret To 10 Million Concurrent Connections"** (Robert Graham)
- **ScyllaDB Engineering Blog** — Seastar / io_uring series

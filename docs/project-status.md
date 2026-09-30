# io_uring-echo-server project status

## Classification

Low-level C networking benchmark sketch (not a service): `io_uring` vs classic
`accept()` socket micro-benchmarks, with a minimal accept-and-send server and a
zero-byte throughput client/server pair.

## Status

Archive / research. Code is complete enough to compile and run each binary, but
the project is an abandoned personal experiment (sources dated 2024-12-03), not
a maintained tool. It is misnamed: **no echo path exists** — the server never
reads from clients or returns their data.

## Evidence

- `io_uring_server.c` — `io_uring` accept + one-shot `send("HTTP/1.0 200\n\nhello")`
  + close; `PORT`/`RING` env vars. Verified: compiled against `liburing 2.5`
  and probed on loopback — returned `hello` then closed the connection.
- `io_uring_zero_server.c` / `io_uring_zero_client.c` — send `BUF` zero bytes vs
  receive loop printing MiB/s. Verified: ran the pair on loopback; client
  reported ~1.2–3.3 GiB/s with `BUF=65536`. The client's host argument is now
  resolved with `getaddrinfo` (was `inet_pton`, numeric IPv4 only).
- `srv.c` — blocking `accept()` baseline, accepts/second. Compiles
  (with a `time.h` implicit-declaration warning).
- `cli.c` — `io_uring` connect-storm client, 512 in-flight connects, never
  receives. Compiles; `HOST`/`PORT` were made configurable (default
  `localhost:8000`) during publication.
- `makefile` — builds only `srv` and `cli`, then `docker build -tio .`; the
  `io_uring_server.c` rule is commented out and the zero pair has no rule.
- `dockerfile` — two-line stub (`FROM alpine`, `ADD srv cli`), no `CMD`; the
  binaries are glibc-linked and cannot run on Alpine/musl.
- `io_uring_server.bak.c` — dead code: earlier `enum {ACCEPT, SEND}` event
  variant, referenced by nothing; compiles but is never built.

## Boundaries

No tests, CI, or error-path coverage; loops `exit(1)` on first error. No read
path, no backpressure, no graceful shutdown, no HTTP semantics (fixed payload,
bare `\n`, no `Content-Length`). Environment used for verification: gcc 13.3
(Ubuntu), kernel 7.0.0, `liburing` headers extracted locally from
`liburing-dev 2.5-1build1` (the package was not installed system-wide). No
license file or headers exist, so default all-rights-reserved applies.

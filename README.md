# io_uring-echo-server

A small, unfinished `io_uring` networking benchmark collection: a minimal
accept-and-send server, a zero-byte throughput client/server pair, and a
plain-`accept()` baseline, written to measure `io_uring` against classic
socket syscalls.

## Why it exists

The project explores how `io_uring` performs for the two operations that
dominate a small network service — accepting connections and pushing bytes —
compared with a conventional blocking `accept()` loop. The name says "echo
server", but **nothing here echoes**: the server sends a fixed payload and
never reads from the client. Treat it as a personal benchmark sketch / archive,
not as a reusable echo or HTTP server.

## Build

Dependencies: a C compiler, `liburing` (headers + library), and `make`.
On Debian/Ubuntu:

```sh
sudo apt-get install -y build-essential liburing-dev
```

The checked-in `makefile` is incomplete (the `io_uring_server.c` rule is
commented out and it does not build the `io_uring_zero_*` pair), so build the
individual binaries explicitly:

```sh
gcc srv.c                   -Wall -O2 -luring -o srv
gcc cli.c                   -Wall -O2 -luring -o cli
gcc io_uring_server.c       -Wall -O2 -luring -o io_uring_server
gcc io_uring_zero_server.c  -Wall -O2 -luring -o io_uring_zero_server
gcc io_uring_zero_client.c  -Wall -O2 -luring -o io_uring_zero_client
```

Or use the makefile as-is (builds only `srv` and `cli`, then tries to build the
Docker image):

```sh
make          # gcc srv.c ...; gcc cli.c ... -l:liburing.a; docker build -tio .
```

`make` relies on `-march=native` and on `liburing.a` being on the linker path;
if it fails, use the explicit `gcc` lines above.

### Docker

The Dockerfile is a two-line stub:

```dockerfile
FROM alpine
ADD srv cli
```

```sh
make                              # produces ./srv and ./cli first
docker build -t io .
docker run --rm -it io sh         # no CMD: drops you in a shell
```

It does **not** work as a runnable image out of the box: there is no `CMD`, and
`alpine` uses musl libc while the `srv`/`cli` binaries are dynamically linked
against glibc (`/lib64/ld-linux-x86-64.so.2`), so executing them inside the
container fails. A working variant would need a glibc-based image plus
`liburing2`, or a fully static build (and an actual `CMD`).

## Run

Default port is `8000`; `RING` sets the io_uring queue depth and `BUF` the
payload size (bytes).

Static "hello" server (`io_uring` accept + one-shot send, then close):

```sh
PORT=8000 RING=256 ./io_uring_server
curl -v http://localhost:8000/     # body "hello", then curl reports "connection reset"
```

Plain `accept()` baseline (no `io_uring`; prints accepts per second):

```sh
./srv
```

Raw throughput pair — the zero server sends `BUF` zero bytes per client, the
client receives in a loop and prints MiB/s (arguments are `HOST PORT CONNECTIONS`):

```sh
PORT=8000 BUF=65536 RING=256 ./io_uring_zero_server
BUF=65536 ./io_uring_zero_client localhost 8000 1
```

Connection-open client (`io_uring`; target host/port come from `HOST`/`PORT`,
default `localhost:8000`; keeps 512 connects in flight forever and never
receives):

```sh
HOST=localhost PORT=8000 ./cli
```

## What is actually implemented vs not

Implemented (compiles and runs):

- `srv.c` — blocking `accept()` loop, counts accepted connections per second.
- `io_uring_server.c` — `io_uring` accept, then `send()`s the constant
  `HTTP/1.0 200\n\nhello`, then closes. `PORT`/`RING` env vars.
- `io_uring_zero_server.c` / `io_uring_zero_client.c` — accept + send `BUF`
  zero bytes vs connect + recv, printing MiB/s.
- `cli.c` — `io_uring` connect-storm client (512 concurrent connects, re-arming
  forever).

Not implemented / caveats (honest list):

- **It is not an echo protocol.** No code reads client data or echoes it back;
  the repo name is aspirational.
- **It is not an HTTP server.** The response is a fixed string with bare `\n`
  line endings, no status line fields, no `Content-Length`, and the connection
  is closed immediately.
- **`io_uring_server.bak.c` is dead code.** It is an earlier variant that tags
  completions with an `enum {ACCEPT, SEND}` packed into `user_data`; no
  makefile target or other file references it. Kept only for history.
- **The `makefile` does not build the io_uring server**: that line is commented
  out, and the `io_uring_zero_*` binaries have no rule at all.
- **The Dockerfile is a stub** (no `CMD`; Alpine/musl cannot run the
  glibc-linked binaries) — see above.
- No tests, no CI, no error-path coverage (the loops `exit(1)` on the first
  error), no read path, no backpressure, no graceful shutdown.
- The client `HOST`/`PORT` were originally hardcoded to a Docker-internal
  address; that was replaced with environment variables (default
  `localhost:8000`) while preparing this repo for publication. The zero client
  likewise now resolves its `HOST` argument with `getaddrinfo` instead of
  requiring a numeric IPv4 address.

## Status

`Status: archive / research` — a dated personal benchmark sketch (source files
last modified 2024-12-03). Not a maintained tool; expect it not to build or run
without edits on modern systems.

## License

No license. There is no `LICENSE` file and no license header in any source
file, so the default "all rights reserved" applies. Add an explicit license
before reusing or redistributing this code.

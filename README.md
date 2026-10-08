[English](README.md) | [中文](README.zh.md)

<p align="center"><img src="assets/logo.svg" width="110" alt="JsonStream"></p>

<h1 align="center">JsonStream</h1>

<p align="center">
  <a href="https://github.com/cuihairu/jsonstream/actions/workflows/ci.yml"><img src="https://github.com/cuihairu/jsonstream/actions/workflows/ci.yml/badge.svg" alt="ci"></a>
  <a href="https://github.com/cuihairu/jsonstream/actions/workflows/pages.yml"><img src="https://github.com/cuihairu/jsonstream/actions/workflows/pages.yml/badge.svg" alt="pages"></a>
  <a href="https://cuihairu.github.io/jsonstream/"><img src="https://img.shields.io/badge/docs-GitHub%20Pages-blue" alt="docs"></a>
  <a href="https://codecov.io/gh/cuihairu/jsonstream/branch/main"><img src="https://codecov.io/gh/cuihairu/jsonstream/branch/main/graph/badge.svg" alt="codecov"></a>
  <a href="https://pkg.go.dev/github.com/cuihairu/jsonstream"><img src="https://pkg.go.dev/badge/github.com/cuihairu/jsonstream.svg" alt="Go Reference"></a>
  <a href="LICENSE"><img src="https://img.shields.io/badge/License-MIT-blue.svg" alt="License: MIT"></a>
</p>

## Overview

JsonStream is a TCP-based custom binary framing protocol that carries JSON. A single connection runs five interaction patterns — request/response, streaming, duplex, one-way, and publish/subscribe — while heartbeats, disconnect recovery, compression and encryption, and credit-based backpressure all live inside the protocol, not reinvented at the application layer. The Go reference implementation uses only the standard library, with zero third-party dependencies (Go ≥ 1.24, [v0.1.0 release notes](https://github.com/cuihairu/jsonstream/releases/tag/v0.1.0)); the protocol specification is language-neutral ([docs/protocol.md](docs/protocol.md), the single source of truth for cross-language implementations), and implementations in other languages are planned.

The online documentation site is published automatically from main: <https://cuihairu.github.io/jsonstream/>. Item-by-item summaries of recent changes are in the [changelog](https://cuihairu.github.io/jsonstream/changelog).

## Quick Start in 30 Seconds

Zero third-party dependencies:

```bash
go get github.com/cuihairu/jsonstream
```

One route and one call:

```go
ln, _ := net.Listen("tcp", "127.0.0.1:9000")
s, _ := jsonstream.NewServer(ln, jsonstream.DefaultConfig())
s.Handle("math.add", func(r *jsonstream.Request) (any, error) {
    var in map[string]int
    if err := r.Decode(&in); err != nil {
        return nil, err
    }
    return map[string]int{"sum": in["a"] + in["b"]}, nil
})
go s.Serve()

c, _ := jsonstream.Dial(context.Background(), "127.0.0.1:9000", jsonstream.DefaultConfig())
m, _ := c.Request(context.Background(), "math.add", map[string]int{"a": 2, "b": 40})
var out map[string]int
_ = m.Decode(&out) // {"sum": 42}
```

## Core Features

- All five interaction patterns over a single TCP connection: request/response, streaming, duplex, one-way, and publish/subscribe; every data frame carries a Stream ID (control frames excepted), so multiplexing needs no negotiation
- Heartbeats are part of the protocol: bidirectional, independent PING/PONG; read-idle beyond 1.5× the interval declares the peer dead. TCP keepalive cannot detect a deadlocked process, which is why this protocol does not use it
- Automatic reconnection (exponential backoff + jitter); the server replays subscriptions and downstream frames within the retention window (at-least-once), and a new connection with the same session takes over from the old one
- Compression (flate), encryption (AES-256-GCM + 32-byte pre-shared key), and backpressure (credit) are all optional: negotiated at handshake, self-described by per-frame Flags, and zero runtime cost when off
- Metadata and Payload are separated: route/topic are always plaintext, so a gateway can authenticate, rate-limit, and forward without decoding business data
- 14-byte fixed header + 32-bit big-endian length prefix; a single frame's payload is capped at 16 MiB, and the decoder validates the length before allocating memory
- Quality baseline: 100% statement coverage on the library package (CI gate), five fuzz targets under continuous smoke testing, eight static checks with zero findings, and cross-compilation for five platforms

## Quick Examples

Method-by-method usage of streaming, duplex, subscriptions, and one-way is in [docs/api.md](docs/api.md); a complete walkthrough of the five interaction patterns is in [docs/getting-started.md](docs/getting-started.md). Compression, encryption, and backpressure are parameterized switches that take effect after handshake negotiation (enabled only when both sides opt in); all default to off with zero overhead:

```go
cfg := jsonstream.DefaultConfig()
cfg.Compress = true // payloads ≥64B go through flate
cfg.Encrypt = true  // AES-256-GCM; compress first, then encrypt
cfg.Key = key32     // 32-byte pre-shared key
cfg.Credit = 64     // connection-level credit window; effective value is the min of both sides
```

Initiation-style APIs all take a `ctx` first: `SendOneWay`/`Publish` (on both the Client and Server sides) follow the same convention as `Request`/`Stream`/`Channel`; the context bounds two waits — waiting for an available connection and waiting for the send to be enqueued (DESIGN §8.4). This signature was finalized before the initial release (v0.1.0), and no aliases are kept for older signatures — the old behavior is equivalent to passing `context.Background()`.

Runnable end-to-end programs are in [examples/](examples/): `examples/server` and `examples/client` cover six scenario groups — request/response, streaming cancellation, duplex, one-way, publish/subscribe, and server-initiated interactions.

## Documentation Site Guide

Every page on the documentation site is sorted by what you want to do:

### Application developers: using the library directly in a Go app

- Getting started: [Environment and installation](https://cuihairu.github.io/jsonstream/getting-started#环境与安装), [First request/response](https://cuihairu.github.io/jsonstream/getting-started#第一个请求-响应), [The five interaction patterns](https://cuihairu.github.io/jsonstream/getting-started#五种交互模式), [Parameterized switches](https://cuihairu.github.io/jsonstream/getting-started#参数化开关)
- Reference: [Client and Server methods](https://cuihairu.github.io/jsonstream/api#client), [All Config fields](https://cuihairu.github.io/jsonstream/api#config), [Error codes](https://cuihairu.github.io/jsonstream/api#错误), [Glossary](https://cuihairu.github.io/jsonstream/glossary)
- Troubleshooting and performance: [FAQ](https://cuihairu.github.io/jsonstream/faq), [Measured benchmarks](https://cuihairu.github.io/jsonstream/benchmarks) ([Where the overhead goes, item by item](https://cuihairu.github.io/jsonstream/benchmarks#逐项解读-开销主要在哪))

### SDK authors: gateways, proxies, and higher-level wrappers

- [Frame layer (low level)](https://cuihairu.github.io/jsonstream/api#帧层-低层): send and receive frames directly, bypassing interaction semantics
- [Metadata/Payload separation](https://cuihairu.github.io/jsonstream/protocol#_5-metadata-与-payload): route/topic are readable by middleware, so routing works without decrypting the payload
- [Handshake negotiation](https://cuihairu.github.io/jsonstream/protocol#_6-1-协商) and [backpressure (credit)](https://cuihairu.github.io/jsonstream/protocol#_7-9-背压-可选-credit-based): the semantics a gateway must handle for pass-through and rate limiting
- [Concurrency model at a glance](https://cuihairu.github.io/jsonstream/api#并发模型速览): which methods may be called concurrently from multiple goroutines

### Protocol implementers: reimplementing the protocol in other languages

- The [protocol specification v1](https://cuihairu.github.io/jsonstream/protocol) is the single source of truth; from this one document you should be able to write an interoperable implementation: [frame layout](https://cuihairu.github.io/jsonstream/protocol#_3-1-布局-大端序), [frame types](https://cuihairu.github.io/jsonstream/protocol#_4-帧类型), [handshake](https://cuihairu.github.io/jsonstream/protocol#_7-1-握手), [disconnect recovery](https://cuihairu.github.io/jsonstream/protocol#_8-断线重连恢复), [fragmented transfer](https://cuihairu.github.io/jsonstream/protocol#_3-4-分片传输-大消息)
- Trade-offs and boundaries of the Go reference implementation: [Architecture and trade-offs](https://cuihairu.github.io/jsonstream/DESIGN) ([Edge-case inventory](https://cuihairu.github.io/jsonstream/DESIGN#_8-边界情况清单), [Design decision log](https://cuihairu.github.io/jsonstream/DESIGN#_10-设计决策清单-备选方案与放弃理由)), [Design notes](https://cuihairu.github.io/jsonstream/design-notes), [Knowledge notes](https://cuihairu.github.io/jsonstream/NOTES)
- Comparisons and the original problem statement: [Comparison with WebSocket](https://cuihairu.github.io/jsonstream/websocket-comparison), [TCP stream characteristics and protocol landscape](https://cuihairu.github.io/jsonstream/tcp-and-landscape), [Assignment requirements](https://cuihairu.github.io/jsonstream/interview-requirements)

## Benchmark Summary

The numbers below are drawn from seven of the nine benchmarks in [bench_test.go](bench_test.go) (`go test -run '^$' -bench . -benchmem -count=10 -benchtime=1s`, i9-10880H / Go 1.27.1, the low-load window of 2026-10-01, as order-of-magnitude references; the compress+encrypt and encrypted end-to-end targets are not listed — all 9 targets appear below):

| Benchmark (bench_test.go) | Result |
|---|---|
| `BenchmarkFrameRoundTrip64B` / `1KiB` / `64KiB` | ~0.6µs / ~1.8µs / ~33µs |
| `BenchmarkTransformPlain` / `Encrypt` / `Compress` (~1.3KiB JSON) | ~14ns / ~7.7µs / ~27µs |
| `BenchmarkRequestResponsePlain` (loopback RTT on one machine) | ~0.12ms |

Across machines, compare relative relationships, not absolute values. This table and the 9-target × 10-round benchstat aggregate in [docs/benchmarks.md](docs/benchmarks.md) come from the same source (the same 2026-10-01 run, with methodology notes kept separately on each page); that page also retains the same set of numbers from the high-load window of 2026-09-28 for reference (same-target differences of 3.5~8.5× fall within the measured range). ns/op fluctuates noticeably on the shared container; what reproduces reliably are structural properties such as allocs/op and B/op. The compression path reuses flate coders per frame (`sync.Pool` + `Reset`); pooling made compression round trips 4.1× faster and cut allocations 162× (DESIGN §10-D11).

Quality and verification figures: statement coverage of the library package is 100.0% (the CI gate fails if it drops; statement counts vary with the Go toolchain version — both the 1.24 and stable legs measured 100.0%, so only the percentage is recorded without the denominator; both example packages are also at 100.0%). The Codecov badge shows 100% (`codecv.yml` sets the line-coverage target at 99%, currently ≈99.85% before rounding), and it does not redden CI (`fail_ci_if_error: false`, decoupled from the 100% statement-coverage gate). 212 test/benchmark/fuzz functions. `-race -count=1` is fully green (the same configuration as the CI gate) plus a goroutine-leak guard. Five fuzz targets run as CI smoke tests (long runs have accumulated execs on the order of ten million, confirming the fixes for items 1 and 2 in the "Real bugs caught by testing" section below). Eight static checks (vet/gofmt/gosec/revive/staticcheck/gocritic/nilness/govulncheck) and five-platform cross-compilation are all enforced as CI gates. Methodology and item-by-item data: [docs/DESIGN.md](docs/DESIGN.md) §11; commands to reproduce locally are under "Contributing" below.

## Contributing

Issues and PRs are welcome; before merging, please make local verification and CI green together (this repository has no CONTRIBUTING.md; this section is the contribution agreement).

### Local Verification

```bash
git clone https://github.com/cuihairu/jsonstream.git
cd jsonstream

# Build + full tests + static checks
go build ./...
go test ./...
go vet ./...

# Coverage: library package 100.0% (CI gate), examples/server 100.0%, examples/client 100.0%
go test -cover ./...

# Benchmarks
go test -bench . -benchtime 2s

# Fuzz smoke tests (same as CI: auto-discover every target per package, 20s each)
for pkg in $(go list ./...); do
  for t in $(go test -list 'Fuzz.*' "$pkg" | grep '^Fuzz'); do
    go test -run '^$' -fuzz "^${t}$" -fuzztime 20s "$pkg"
  done
done

# Deep-dive a single fuzz target (note: -fuzz accepts only a single package; ./... errors out)
go test -run '^$' -fuzz FuzzReadFrame -fuzztime 300s .

# End-to-end example: start the server in terminal 1
go run ./examples/server
# Run the client in terminal 2 (request/response, streaming cancellation, duplex, one-way, publish/subscribe, server-initiated)
go run ./examples/client
```

### Documentation Site Development

```bash
pnpm install      # first run; Node 22 / pnpm 12
pnpm dev          # local development, http://localhost:5173
pnpm build        # builds into docs/.vitepress/dist, with built-in dead-link checks
pnpm check:links  # internal-link casing and cross-page anchors; real HEAD probes for external links
```

### CI and Publishing

Pushes trigger two workflows: [ci](https://github.com/cuihairu/jsonstream/actions/workflows/ci.yml) (build + full tests + fuzz smoke + coverage gate) and [pages](https://github.com/cuihairu/jsonstream/actions/workflows/pages.yml) (documentation site build and publishing). A green PR that changes docs does not mean the live site is updated — the pages deploy runs only after merging into main, and consecutive pushes queue up serially in the deploy pipeline; three easy-to-trip-over details are in [docs/faq.md](docs/faq.md). The meaning of each gate (100% coverage, fuzz, static checks) is covered in the last paragraph of "Benchmark Summary" above and in [docs/DESIGN.md](docs/DESIGN.md) §11.

## Protocol and Frame Format

The frame-format specification (field by field, frame type by frame type) is authoritative at [docs/protocol.md](docs/protocol.md); this section is a guided tour.

```
 0               8               16              24              32
 +---------------+---------------+---------------+---------------+
 |     Magic "JS" (0x4A53)       |    Version=1  |     Flags     |
 +---------------+---------------+---------------+---------------+
 |     Type      |   Reserved    |          Stream ID            |
 +---------------+---------------+---------------+---------------+
 |                        Payload Length                          |
 +---------------+---------------+---------------+---------------+
 |   Meta Length (2B, if Flags.HasMeta) | Metadata (JSON, route/topic) |
 +---------------------------------------------------------------+
 |                     Payload (JSON)                             |
 +---------------------------------------------------------------+
```

TCP is a byte stream with no message boundaries, and framing options come down to three kinds: fixed length, delimiters, and length prefixes. JSON makes delimiters ambiguous, so this protocol uses a 14-byte fixed header + 32-bit big-endian length prefix: on the decoding side, "read the header → two `ReadFull` calls" is an unconditional operation with no cross-frame state machine — it can be unit-tested, fuzzed, and re-entered after dropping a frame at any position. Magic gives misconnections and port probes an immediate verdict; Version gives the handshake a basis for failing fast. The reserved bits of Flags must be zero; a nonzero value is treated as Malformed and the connection is closed, leaving the door open for future upgrades.

Against the WebSocket (RFC 6455 §5.2) frame format, two things are dropped and one is reshaped (point-by-point rationale in [docs/design-notes.md](docs/design-notes.md) §1):

- Fragmentation is reshaped, not done as FIN+continuation: the 16 MiB single-frame cap bounds per-frame memory; larger logical messages are split into consecutive chunks — the opening frame keeps its original type and sets `FlagFragmented`, and continuation/closing chunks use a separate `FRAGMENT` frame type. Runs cannot interleave, so the receiver needs no chunk sequence numbers, and the reassembly total is hard-capped by `MaxMessageSize` (default 64 MiB) ([protocol §3.4](docs/protocol.md); the spec is finalized and the Go reference implementation is in progress).
- No client masking: MASK defends against proxy cache poisoning from the browser era; dedicated client/server direct connections do not have that threat model.
- No 7/16/64-bit variable-length sizes: variable-length encoding saves 2~6 bytes on most frames at the price of a branching state machine in the decoder; the fixed 4-byte length's ceiling doubles as a memory gate (validate first, allocate second — the OOM switch is not in the peer's hands).

### Interaction Model

The four interaction primitives follow RSocket's division: `REQUEST→RESPONSE` (one question, one answer), `REQUEST+Flags.Stream→N×RESPONSE+COMPLETE` (streaming), `ONEWAY` (one-way; not even an error comes back), and `REQUEST+Flags.Channel` (bidirectional multi-frame). Every frame carries a Stream ID (control frames excepted), split odd/even by initiator (client odd, server even — the same idea as HTTP/2), so frame ownership is known without negotiation, and the counters on both ends increase monotonically across reconnections.

Two details worth knowing: request/response does not append a COMPLETE frame — the response frame carries finality by itself, saving one round trip per interaction; CANCEL is part of the stream — backpressure handles "producing faster than consuming", CANCEL handles "the consumer does not want it at all". The divergence from RSocket is one point: it splits three request kinds into three frame types, while here one `REQUEST` plus the Flags.Stream/Flags.Channel bits expresses them, keeping parse branches leaner; the cost is that middleware cannot predict the interaction pattern from the frame header alone, and v1 falls back to replying ERROR(PROTOCOL) when the Flags declaration contradicts the handler ([design-notes §2](docs/design-notes.md)).

### Other Mechanisms at a Glance

- Metadata/Payload separation: `route`/`topic` live in the frame-header Metadata section and are always plaintext, so gateways can authenticate, rate-limit, and route without decoding the payload; the cost is that routes can be profiled statistically by anything on the path ([design-notes §4](docs/design-notes.md)).
- Heartbeats: application-layer PING/PONG sent independently in both directions; read-idle beyond 1.5× the interval declares death. 1.5× is the compromise between tolerating one dropped-frame jitter and detecting death quickly ([protocol §7.2](docs/protocol.md)).
- Disconnect recovery: CONNECT carries a `session_id`; the server replays subscriptions and downstream frames within the retention window (default 30s, 4 MiB per-session cap) with at-least-once semantics, and a new connection with the same session takes over from the old one. The boundaries are written into the spec — TCP frames in flight are not guaranteed replay (non-idempotent operations rely on business IDs as a backstop), and upstream traffic is not cached ([protocol §8](docs/protocol.md)).
- Compression and encryption: per-frame Flags self-describe whether compression/encryption is applied; heartbeat frames are always plaintext and only large frames are compressed, so both coexist on one connection. The order is frozen as compress-then-encrypt (ciphertext does not compress); AES-256-GCM + 32-byte PSK. A PSK does not solve key distribution — in production, wrap TLS around it ([protocol §6](docs/protocol.md), [design-notes §3](docs/design-notes.md)).
- Backpressure: credit is granted per message and returned on delivery, off by default — when the allowance reaches zero, `Emit()` blocks; this is the only mechanism in the protocol that pushes back on the application's concurrency model. Request/response and ONEWAY are excluded from credit ([design-notes §5](docs/design-notes.md)).
- pub/sub and req/res share one connection: the same PUBLISH frame type carries dual semantics in the two directions; the asymmetry — a failed subscription answers ERROR while a failed delivery answers no frame — is deliberate and written into the spec ([design-notes §6](docs/design-notes.md)).

### Real Bugs Caught in Testing: Concrete Consequences of Not Designing This Way

Nine real defects hit in practice and fixed along the way — each one a design lesson:

1. **Decompression bomb** (caught by fuzzing): with no cap on decompression output, a single 16 MiB flate frame can expand by three orders of magnitude — not guarding this hands the OOM switch to the peer.
2. **Encoding not self-consistent** (caught by fuzzing): when Flags declared Metadata with empty content, the metaLen section was skipped and parse-encode stopped being inverses — serialization must satisfy round-trip identity.
3. **Credit broke at-least-once**: after a disconnect, acquiring credit failed immediately, handlers misjudged it and exited, the retention queue emptied, and replays were lost — connection-level flow control leaked into session-level recovery semantics; fixed by routing such failures into the retention queue.
4. **Double-ready select on a dead connection**: when the send channel had room, `select` randomly picked the dead branch and frames were silently lost — Go's select randomness is a real trap on error paths.
5. **Stream ID collision**: after reconnection the ID counter reset to zero, so new streams and migrated streams shared IDs, and a late COMPLETE from an old stream killed a new one — with multiplexing plus recovery, IDs must be monotonic across reconnections.
6. **Stale connection references**: subscription handles held the endpoint from creation time, so after reconnection, unsubscribe frames went to the dead connection and were silently swallowed — outbound traffic always takes the current endpoint, letting the compiler eliminate stale references.
7. **Responder streams did not migrate**: handlers of server-passive streams dangled after reconnection, producing frames into the void and able to burst the retention queue — session migration must move the handlers along with everything else.
8. **Hanging after disconnect**: server-initiated interactions hung forever after the connection died, waiting for a response that would never come — responses are upstream and not cached, so waiting is pointless; a disconnect is an immediate failure.
9. **Metadata limit off-by-one** (caught by gosec): the limit was coded as a round 64 KiB, but the length field is a uint16 — exactly 64 KiB would silently truncate to 0 and encode a garbled frame. A bound must equal what the field can express, not a convenient round number.

## Configuration Reference

`Config` is a value type: modifying the original variable after passing it to `Dial`/`NewServer` does not affect an established endpoint. The effective values are those delivered in CONNACK (the server is authoritative); `DefaultConfig()` returns the recommended full defaults. Field-by-field API details are in [docs/api.md](docs/api.md).

| Field | Type | Default | Semantics |
| --- | --- | --- | --- |
| `Heartbeat` | `time.Duration` | `DefaultHeartbeat` (10s) | PING interval; read-idle beyond 1.5× declares death. Zero takes the default; values below 1s clamp to 1s |
| `Compress` | `bool` | `false` | flate compression (only payloads ≥64B actually compress) |
| `Encrypt` | `bool` | `false` | AES-256-GCM; when enabled, `Key` must be 32 bytes |
| `Key` | `[]byte` | `nil` | 32-byte pre-shared key, never sent over the wire |
| `Credit` | `int` | `0` (off) | Connection-level credit window (message count); effective value = min of both sides; ≤0 on either side turns it off |
| `Auth` | `string` | `""` | Token passed through in CONNECT; the server validates it in `OnAuth` |
| `Retention` | `time.Duration` | `DefaultRetention` (30s) | Server-side session retention window; a negative value disables session recovery |
| `RetentionBytes` | `int` | `DefaultRetentionBytes` (4 MiB) | Per-session byte cap on the downstream retention queue; exceeding it forfeits recovery |
| `DialTimeout` | `time.Duration` | `DefaultDialTimeout` (5s) | Timeout for establishing the TCP connection |
| `Reconnect` | `*bool` | `nil` (= true) | When false, the client does not auto-reconnect |
| `BackoffInitial` | `time.Duration` | `DefaultBackoffInitial` (100ms) | Reconnect backoff starting point (exponential + jitter) |
| `BackoffMax` | `time.Duration` | `DefaultBackoffMax` (5s) | Reconnect backoff ceiling |
| `Logger` | `Logger` | `nil` (silent) | Adapts common implementations such as `*log.Logger` |

```go
type Logger interface{ Printf(format string, v ...any) }
```

## Ecosystem Positioning

One-sentence positioning: for backend service-to-service communication, take WebSocket's framing, RSocket's interaction model, and MQTT's session semantics — a slice from each. Capability boundaries versus WebSocket (RFC 6455):

- What WS has that this protocol lacks or explicitly does not do: native browser reachability, TLS as a first-class carrier with port-443 reuse, subprotocol/extension negotiation, text/binary opcode distinction (client Masking is explicitly unnecessary — raw TCP direct connections do not have that threat model).
- What the WS standard lacks that this protocol builds in: request/response, publish/subscribe, streaming, credit backpressure, disconnect recovery, and Stream ID multiplexing — in WS applications, all of these are hand-built.
- Point-by-point evidence (protocol sections and code locations) is in [docs/websocket-comparison.md](docs/websocket-comparison.md); a wider protocol comparison (MQTT / RSocket / HTTP/1.1→HTTP/2→HTTP/3 with a summary table) is in [docs/tcp-and-landscape.md](docs/tcp-and-landscape.md).

Flow-control design comparison (why credit, not a sliding window or leases):

| Model | Exemplars | Semantics | Complexity |
| --- | --- | --- | --- |
| Sliding window | TCP / HTTP/2 | Grants in bytes | High (byte-level accounting) |
| Credit per message | This protocol, Reactive Streams `request(n)` | Grants N messages, returned on delivery | Medium |
| Time-based lease | RSocket `LEASE` | Constrains rate, not in-flight volume | Low |

Byte-level accounting is over-engineering for JSON messages; LEASE guards against abuse, not backpressure (RSocket itself relies on subscriber request(n) for real flow control); counting in messages matches the unit application developers think in.

## Repository Layout and Multi-Language Plans

This repository plans to add implementations in other languages (Python/Rust/Java, among others); the layout is positioned accordingly:

- **The Go implementation stays at the repository root** (`go.mod` at the root, the standard Go ecosystem layout). The Go code will not move into a subdirectory — a module path change would break every existing import.
- **Future implementations in other languages enter as sibling subdirectories** (`python/`, `rust/`, …), independent of one another.
- **The coordination hub is the language-neutral protocol specification**: [docs/protocol.md](docs/protocol.md) is the single source of truth for all implementations, and interoperability of any implementation is judged against it; implementation-detail documents (DESIGN/NOTES, etc.) describe the Go reference implementation and do not constitute cross-language contracts.

# fin-websocket

RFC 6455 WebSocket client + server for Fin. Handshake, frame codec, masking,
ping/pong, close frames, and fragmentation reassembly over blocking TCP.

```fin
import { ws_server_listen, ws_server_accept, ws_recv_message, ws_send_text } from websocket::fin;

// server side
let srv = ws_server_listen(8081);
let acc = ws_server_accept(srv);          // HTTP Upgrade -> 101
let msg = ws_recv_message(acc.fd, true);  // reassembled, ping/pong handled
ws_send_text(acc.fd, "hello back", false);
```

## Installation

With [finn](https://github.com/M1778/finn) (manifest: `finn.toml`):

```toml
[packages]
fin-websocket = "M1778/fin-websocket@0.1.0"
```

Then `import { ... } from websocket::fin;`.

## Dependencies

- [`M1778/zhttp`](https://github.com/M1778/zhttp), commit `3ec72c6ddd47c41255cb2295ab107320a05708ce` — HTTP Upgrade parsing and the native socket transport.
- [`M1778/fin-crypto`](https://github.com/M1778/fin-crypto), commit `777990f2ef19ec7be6a69a628957ffb48da11974` — CSPRNG and token comparison.
- Fin std: `bits`, `strings`, `networking` (socket creation/address helpers).

`finn.toml` pins these pushed Git commits. The pinned zhttp preserves bytes after
HTTP headers and rejects duplicate WebSocket singleton headers. Both libraries
take the socket handle, the buffer types and the error source from one place:
zhttp's native transport, selected from the compiler's native target (ADR 0051
addendum). This module therefore declares no `send`/`recv` of its own — two
declarations of the same linker symbol with different signatures is exactly
how the two libraries used to disagree. `MSG_NOSIGNAL` on Linux suppresses the
signal a write to a closed peer would otherwise raise; macOS sets
`SO_NOSIGPIPE` on the socket instead, and Windows reports the error instead.
No invented release tags are needed. The older
`fin-websocket@0.1.0` tag does not include this work; use the current source.
After successful handoff the plain `HttpConn` wrapper is freed while preserving
the caller-owned descriptor.

### Why SHA-1 and base64 are vendored here

Two deliberate, documented exceptions to "don't duplicate":

1. **SHA-1.** `fin-crypto` v0.1.0 ships SHA-256 only, but the WebSocket
   accept key (RFC 6455 §4.2.2) requires SHA-1. A minimal SHA-1 is vendored
   in `src/websocket.fin` (`ws_sha1_hex`), verified against Python `hashlib`
   and `openssl`. This is filed as a **fin-crypto feature request**: when
   fin-crypto ships SHA-1, this module switches to it and the vendored copy
   is deleted.
2. **Base64 over byte arrays.** Std `encoding.fin`'s `base64_encode` takes a
   `string` and measures it with `strlen`, so it truncates binary digests and
   random keys at the first NUL byte. `ws_b64encode(data, len)` takes an
   explicit length. Proposed as a std addition; until then it lives here.

## API reference

### Handshake

| Function | Description |
|---|---|
| `ws_client_key() <string>` | Fresh `Sec-WebSocket-Key`: base64 of 16 random bytes |
| `ws_accept_key(client_key: string) <string>` | `Sec-WebSocket-Accept` for a client key (RFC 6455 §4.2.2) |
| `ws_client_request(host, port, path, key) <string>` | Build the HTTP Upgrade request (pure) |
| `ws_validate_server_response(resp, key) <bool>` | Check a 101 response + accept key, constant-time (pure) |
| `ws_check_handshake_headers(method, upgrade, connection, key, version) <bool>` | Validate extracted client headers (pure) |
| `ws_client_connect(host, port, path) <WsClientResult>` | Dial + handshake; returns the connected fd |
| `ws_server_listen(port) <&HttpServer>` | Listen (via zhttp) |
| `ws_server_accept(srv) <WsAcceptResult>` | Accept + server handshake; returns the raw fd |
| `ws_conn_close(fd) <bool>` / `ws_server_close(srv)` | Close a connection / listener |

Opcodes are `int` codes (backend limit: no enum-typed struct fields):
`0` continuation, `1` text, `2` binary, `8` close, `9` ping, `10` pong.

### Frames

| Function | Description |
|---|---|
| `ws_encode(fin, opcode, payload, payload_len, mask_key, mask_len) <WsBytes>` | Encode one frame; `mask_len` 0 or 4 (deterministic — pass the key) |
| `ws_decode(data, len) <WsDecodeResult>` | Decode one frame; `consumed` bytes used; incomplete input → `ok=false` |
| `ws_send_frame(fd, fin, opcode, payload, payload_len, masked) <bool>` | Encode + send (fresh random mask when masked) |
| `ws_recv_frame(fd) <WsDecodeResult>` | Read one frame from the socket |
| `ws_send_text / ws_send_binary / ws_send_ping / ws_send_pong` | Convenience senders |
| `ws_send_close(fd, code, reason, masked) <bool>` | Close frame with status code |
| `ws_recv_message(fd, from_client) <WsMsgResult>` | Reassemble fragmented messages; auto-replies pong, echoes close |
| `ws_free_bytes / ws_free_decode / ws_free_message` | Free result buffers (`&` — no address auto-taking) |

`ws_recv_message` enforces masking on **every** frame: `from_client=true`
requires masked input and sends unmasked replies; `false` requires unmasked
input and sends masked replies. Header validation rejects unknown opcodes,
RSV bits, fragmented/oversized control frames, nonminimal lengths and size
limits before reading a payload. Text is checked as complete UTF-8 after
reassembly, so codepoints may span fragments. Close status/reason validation
is recoverable; empty Close bodies and embedded NULs are echoed unchanged.

`WsDecodeResult.error_code` and `WsMsgResult.error_code` distinguish protocol
errors (1002), invalid UTF-8 (1007), and size limits (1009); zero means no
protocol classification, including incomplete input or transport failure.
On failure, `ws_recv_message` attempts a role-correct Close and shuts down socket
I/O. The caller **still owns and must close the fd**. The low-level
`ws_decode`/`ws_recv_frame` APIs are role-neutral and cannot validate whole
fragment sequences. `ws_encode` remains a deterministic low-level wire builder;
`ws_send_frame` and convenience senders reject invalid outbound frames with
`false`.

Handshake validators require exact HTTP fields and token boundaries, uppercase
`GET`, a canonical Base64 key decoding to 16 bytes, and a valid 101 response
with Upgrade, Connection and Accept. Unsolicited extensions/subprotocols are
rejected. Response duplicate required fields are rejected; on the server,
duplicate singleton rejection relies on the coordinated zhttp snapshot.

## Examples

- `examples/echo_server.fin` — echo server on `127.0.0.1:8081`
- `examples/echo_client.fin` — connects, sends "Hello, Fin!", prints the echo

## Verification status

**VERIFIED 2026-10-21** — frozen finc 0.4.0 (contract 3), Linux x86-64,
coordinated sibling zhttp and fin-crypto sources, executable builds at **O0
and O2**. All commands below exit zero; no Python peer or external test service
is needed. The socket suite uses Fin libc bindings for socketpair, fork and
loopback peers, with an alarm turning accidental blocking into a failed run.

From the workspace parent containing `fin-websocket`, `zhttp`, `fin-crypto`
and `.toolchain`, run this Bash block:

```bash
set -e
ROOT="$PWD"
mkdir -p "$ROOT/fin-websocket/out"
for opt in 0 2; do
  for suite in test_websocket test_socket bench_fragments; do
    "$ROOT/.toolchain/bin/finc" "$ROOT/fin-websocket/tests/$suite.fin" \
      --fin-libs "$ROOT/.toolchain/lib/std" \
      -I "$ROOT/fin-websocket/src" -I "$ROOT/zhttp/src" \
      -I "$ROOT/fin-crypto/src" -O"$opt" \
      -o "$ROOT/fin-websocket/out/${suite}_O$opt"
    "$ROOT/fin-websocket/out/${suite}_O$opt"
  done
done
```

- `test_websocket.fin`: SHA-1/Base64/RFC vectors, length roundtrips, strict
  handshake fields, canonical nonce/padding, and malformed responses.
- `test_socket.fin`: malformed header-only peers (no body, no EOF), mask
  directions, recoverable oversized Ping/bad Close, UTF-8 split across frames,
  interleaved binary Ping/Pong, empty Close, tiny/empty fragments, exact 16 MiB
  messages and overflow rejected before body reads, failed-accept descriptor
  counts, and first-frame preservation when coalesced with either handshake.
- `bench_fragments.fin`: alternating one-byte and empty fragments, payload
  integrity assertions, three messages per size. Observed 16/32/64 KiB results:
  **O0 72/152/291 ms; O2 46/90/176 ms**. These are environment-specific timings,
  not test thresholds; near-doubling illustrates linear growth rather than
  quadratic prefix copying.


## Known limitations (v0.1.0)

- Blocking TCP, IPv4 dotted quads only; no hostname resolution or TLS.
  Short I/O and EINTR are retried. On Linux `MSG_NOSIGNAL` avoids process
  termination on disconnected peers; on macOS `SO_NOSIGPIPE` does the same, and
  on Windows a write to a closed peer is reported as an error. No read/write
  deadlines: an indefinitely slow peer or unending empty/control frames can
  occupy a blocking caller.
- One frame / one reassembled message capped at 16 MiB. Geometric accumulator
  growth makes fragmented reassembly linear in bytes plus frame count; empty
  fragments do not copy the accumulated prefix.
- No compression, extensions, or subprotocol negotiation.
- UTF-8 validity of fragmented text is determined when the message completes.
- zhttp owns HTTP parsing and accept behavior; its permanent accept errors may
  still abort. Repeated Upgrade/Connection fields on server requests retain
  zhttp's last-value semantics rather than combining field lines.
- Allocation-failure behavior and concurrent readers on one fd are unsupported.
  Result payloads are caller-owned; free exactly once using the matching API.

## Changelog

See [CHANGELOG.md](CHANGELOG.md).

## License

MIT — see [LICENSE](LICENSE).

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

- [`M1778/zhttp@0.1.0`](https://github.com/M1778/zhttp) — server-side HTTP
  Upgrade handshake parsing (`http_listen` / `http_accept` /
  `http_read_request` / `http_write_raw`). Reused, not reimplemented.
- [`M1778/fin-crypto@0.1.1`](https://github.com/M1778/fin-crypto) — CSPRNG
  (`random_bytes`) for client keys and frame masks, `const_time_eq` for the
  accept-key comparison.
- Fin std: `bits`, `strings`, `networking` (raw socket syscalls).

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

`ws_recv_message` with `from_client=true` enforces RFC 6455 §5.1 (client
frames must be masked): an unmasked data frame gets a 1002 close and an
error.

## Examples

- `examples/echo_server.fin` — echo server on `127.0.0.1:8081`
- `examples/echo_client.fin` — connects, sends "Hello, Fin!", prints the echo

## Verification status

**VERIFIED 2026-10-20** — built and executed with finc 0.4.0 (contract 3).
`tests/test_websocket.fin` passes: FIPS 180-1 SHA-1 vectors (double-checked
against Python `hashlib` **and** `openssl`), the RFC 6455 §1.3 accept-key
vector, the RFC 6455 §5.7 masked-"Hello" frame bytes, 16-bit and 64-bit
extended-length round-trips (300 and 70000 bytes), close frames, and the pure
handshake validators. Socket I/O (`ws_client_connect`, `ws_server_accept`,
`ws_recv_message`) is exercised by `examples/echo_server.fin` +
`examples/echo_client.fin` against a live loopback peer (two processes).


## Known limitations (v0.1.0)

- Blocking TCP, IPv4 dotted quads only (inherits std `networking.fin`'s model;
  no hostname resolution, no TLS — run behind a TLS terminator).
- One frame / one reassembled message capped at 16 MB.
- `ws_server_accept` leaks the small zhttp `HttpConn` wrapper per connection
  (`http_close` would close the fd being handed to the caller); the fd itself
  does not leak.
- No compression (`permessage-deflate`) or extensions.
- Fragment reassembly copies per fragment (O(n²) for pathological
  fragmentation); fine for normal use.

## Changelog

See [CHANGELOG.md](CHANGELOG.md).

## License

MIT — see [LICENSE](LICENSE).

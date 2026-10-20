# Changelog — fin-websocket

## [0.1.0] — 2026-10-20
### Added
- RFC 6455 WebSocket client + server: HTTP Upgrade handshake (server via
  zhttp, client over raw TCP), frame encode/decode, client masking,
  ping/pong auto-handling, close frames with status codes, fragmentation
  reassembly with 1002 enforcement of masked client frames.
- Vendored SHA-1 (FIPS 180-1, pure Fin) for the accept key — fin-crypto
  v0.1.0 has SHA-256 only; filed as a fin-crypto feature request so the
  vendored copy can be deleted later.
- Vendored explicit-length base64 (`ws_b64encode`/`ws_b64decode`) — std
  `encoding.fin`'s base64 is strlen-limited and cannot encode binary.
- Pure-function handshake validators (`ws_client_request`,
  `ws_validate_server_response`, `ws_check_handshake_headers`) for testability.
- Test suite: FIPS 180-1 vectors (verified vs Python hashlib + openssl),
  RFC 6455 §1.3 accept key, §5.7 masked-frame bytes, 16/64-bit length
  round-trips, close frames, incomplete/RSV error paths.
- Examples: echo server + echo client. Full scaffold: finn.toml (deps
  `M1778/zhttp@0.1.0`, `M1778/fin-crypto@0.1.0`), CI, MIT LICENSE.

### Notes
- UNVERIFIED: no working finc in the build environment; tests not yet run.
- Known limits: blocking TCP/IPv4 dotted quads only, 16 MB frame cap, no
  TLS, no extensions, HttpConn wrapper leak per accepted connection.

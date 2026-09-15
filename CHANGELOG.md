# Changelog

All notable changes to modbus-nv are recorded here. The format is
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this
package follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html)
with the pre-1.0 rule that a breaking change bumps the MINOR number.

## 0.0.2 — 2026-09-15

README rewritten to the package README style guide (docs/writing-a-readme.md); no change to the interface.

## 0.0.1 — 2026-09-12

The **interface**: every signature and every effect row, and no bodies.
`stability = "draft"`, and the release is recorded `implemented = false`.

### Added

- `mbpdu` — `MbReq` and `MbRsp`, the nine function codes, the
  per-code quantity limits as functions, the eleven exception codes,
  the bit and word byte-count arithmetic, and the coil values.
- `mbrtu` — the frame, the address ranges, the CRC's low-byte-first
  order, the lengths that can be counted rather than timed, and the
  two timers with the specification's 19200-baud exception.
- `mbtcp` — the MBAP header, framing by counting, pairing by
  transaction and unit, and the RTU bridge a gateway needs.
- `mbmap` — `MbRegMap` over the four address spaces, `MbHandler[e]`,
  the range check made once, the bit and register packing, and the
  wire-versus-manual address conversion.
- `mbcrc` — the one module that reaches crc-nv.
- `mberr` — the transport faults, `is_retryable` and `is_local`.
- `mbclient` — the `host` module: `MbTransport[e]`, `MbOptions` with
  its backoff table, the TCP connection, the nine requests, and the
  poll-schedule arithmetic. `[net, time]`.

### Known

- **The load-bearing interface is that an exception is a field on the
  response.** A transport fault means nobody answered; an exception
  means the server answered and refused. They carry opposite
  instructions, and `Result<MbRsp, MbError>` with the exception as the
  error makes conflating them the natural thing to write. `MbError`
  has no exception variant and will not get one.
- **The layer is `core` with one `host` module**, not the plan's
  `host`: the consumer that decided it is a gateway on a
  microcontroller.
- **The count limits are functions rather than prose**, so a client
  checks before it sends rather than spending a round trip on a
  request a conforming server was always going to refuse.
- **The RTU silence is a number this package answers and the caller
  times**, with the specification's own fixed 1750 µs at and above
  19200 baud — because 3.5 characters at 115200 is 304 µs and nothing
  with an operating system can measure that.
- **A `@value` struct cannot be a boxed struct's field** (E2015), so
  `MbClient` carries its six option numbers flat and `client_options`
  assembles an `MbOptions`. The same rule kept `last_response` a
  separate reader and `mbmap.response_for` separate from
  `mbmap.serve`.
- **The device claim covers `mbpdu`, `mbrtu` and `mbtcp`** and
  `tests/embedded_probe.nv` builds them for a Cortex-M4. `mbcrc` is
  deliberately outside it.
- **The biggest missing row is a serial port in the standard
  library.** There is no surface for a baud rate, a parity setting or
  an inter-character timeout, so `mbclient` is TCP-only and an RTU
  client on a host brings its own transport — for a protocol whose
  serial half is the older and more widely deployed one.
- The file-access and diagnostic function codes, Modbus ASCII, a
  serial port and a device profile are deliberately outside.

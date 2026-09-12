# modbus-nv

**Status: NOT IMPLEMENTED — interface only.**

Modbus RTU and TCP: the protocol data unit for function codes 1 to 6,
15, 16 and 23 with the per-code count limits a conforming server
enforces, exceptions as responses rather than errors, RTU framing with
CRC-16/MODBUS and the inter-frame silence as a number the caller times,
the TCP MBAP header, a server-side address map with a handler trait,
and a client over sockets.

Every `pub fn` body is a `todo()`. The signatures and the effect rows
are published so the design can be reviewed and effect-checked before
anyone writes a body against it; `novo pkg add modbus-nv` resolves,
downloads and builds, and the first call panics with
`not implemented: modbus-nv.<module>.<fn>`.

## Build, run and test

```bash
novo pkg build          # type-check and effect-check every module; a library, so no binary
novo test               # the API suites — RED until the bodies land
novo doc .              # the reference page, with every example block compiled
```

There is nothing to run: this is a library, and at 0.0.1 every body is
a `todo()`, so `novo test` is red on purpose and the suites are the
protocol written as assertions.

## The one example that will work

Reading ten holding registers, and telling a refusal from a silence:

```novo
use mbclient
use mbpdu

let c = mbclient.connect("192.0.2.1", 502, mbclient.default_options())!

match mbclient.read_holding(c, 0, 10)
    // The transport worked.  What the SERVER said is a separate
    // question, and this is where most Modbus clients go wrong.
    Ok(data) =>
        let r = mbclient.last_response(c)
        if mbpdu.rsp_is_exception(r)
            // The server received the request, understood it, and
            // refused it.  Retrying exception 2 is retrying something
            // that will never succeed.
            report(mbpdu.exception_name(mbpdu.rsp_exception(r)))
        else
            use_registers(data)
    // Nobody answered.  This one IS worth retrying.
    Err(e) => if mberr.is_retryable(e) { retry() }
```

## The layer, and why

`core`, with one `host` module named in the manifest.

The plan placed the row at `host` because the client is the visible
half. The consumer that decided the layer is a **gateway**: a
sensorhub-class device with an RS-485 port on one side and a radio on
the other, decoding a request in a UART interrupt on a part with 64 KB
of RAM. Everything it needs — the protocol data unit, the RTU framing,
the address map — is arithmetic over integers the caller already holds.
`docs/publishing.md` § A package with a core and a host half is the
shape, and it is what makes the device claim a thing the audit
**builds**.

- **A device links** `mbpdu`, `mbrtu`, `mbtcp` and `mbmap`, plus
  `mbcrc` if it has no CRC peripheral. All are `[]` throughout, and
  `tests/embedded_probe.nv` builds the first three for
  `--target=nrf52-qemu`.
- **A laptop links** `mbclient`, the one module with a row: `[net]`
  for the socket and `[time]` for the deadline a request timeout needs.

## The load-bearing interface

**`MbReq` and `MbRsp`, and specifically that an exception is a field on
the response rather than an error.**

Modbus has two kinds of failure, and every implementation that
conflates them is wrong in a way that only shows up in the field:

- a **transport** fault — a timeout, a CRC mismatch, a closed socket —
  means nobody answered, and the request may be retried;
- an **exception** means the server answered perfectly. It received the
  request, understood it, and refused it, with a code saying why.

Those are opposite instructions. A client that turns exception 2 —
"that address does not exist on this device" — into a transport error
retries it forever against a device that will never say anything else.
A gateway that does the same cannot forward the refusal and invents a
timeout its own client then retries. And `Result<MbRsp, MbError>` with
the exception as the error makes both of those the natural thing to
write.

So `MbRsp` carries an `exception` field, `rsp_is_exception` is the
question, and `mberr.MbError` has **no exception variant and will not
get one**.

**The second load-bearing thing is that the count limits are
functions.** Modbus fixes a maximum quantity per function code — 2000
coils, 125 registers, 1968 coils for a multiple write, 123 registers,
and 125 read with 121 write for function 23 — and a request past its
limit gets exception 3 from a conforming server and silence from a bad
one. `mbpdu.max_count` is that table, so a client checks before it
sends and a wasted round trip does not happen.

**And the RTU silence is a number this package answers.** A Modbus RTU
frame has no delimiter: what separates two frames is 3.5 character
times of silence. A `core` package has no clock, so `mbrtu.silence_us`
gives the number and the caller times it — which is right, since the
thing that measures the gap is a UART interrupt on a device and a
serial port's inter-character timeout on a host. It carries the
specification's own exception: at 19200 baud and above the figure is a
fixed 1750 µs, because 3.5 characters at 115200 is 304 µs and nothing
running an operating system can measure that.

## The reference implementations

[`tokio-modbus`](https://github.com/slowtec/tokio-modbus) and
[`pymodbus`](https://github.com/pymodbus-dev/pymodbus) for the client
and server shapes, and the
[Modbus Application Protocol v1.1b3](https://modbus.org/specs.php),
[Serial Line v1.02](https://modbus.org/specs.php) and
[Messaging on TCP/IP v1.0b](https://modbus.org/specs.php)
specifications for the numbers — the quantity limits, the exception
codes, the 3.5-character silence and the MBAP header are all
transcriptions rather than choices.

## What is here

| Module | Layer | What it is |
| --- | --- | --- |
| `mbpdu` | `core` | `MbReq`, `MbRsp`, the nine function codes, the per-code limits, the exception codes, the byte-count arithmetic. |
| `mbrtu` | `core` | The frame, the addresses, the CRC's byte order, the length that can be counted, and the two timers. |
| `mbtcp` | `core` | The MBAP header, framing by counting, and pairing a reply by transaction AND unit. |
| `mbmap` | `core` | `MbRegMap`, `MbHandler[e]`, the range check made once, and the bit and word packing. |
| `mbcrc` | `core` | The one module that reaches crc-nv. |
| `mberr` | `core` | The transport faults, and no exception variant. |
| `mbclient` | `host` | `MbTransport[e]`, `MbOptions`, the connection, and the poll arithmetic. `[net, time]`. |

## crc-nv carries this exact algorithm

`crc-nv`'s `crc16_modbus()` is polynomial 0x8005 reflected, initialised
to 0xffff, with no final xor — which is **the** variant and not **a**
variant. Its published check value over the nine bytes `123456789` is
**0x4b37**, and a CRC that disagrees with another implementation's is
almost always the wrong variant rather than a wrong implementation.
Writing a second copy here would be two implementations of one
polynomial in one registry.

It is named only by `mbcrc`, which is not in the device probe: every
other module takes a CRC as an `Int`, so a gateway whose
microcontroller has a CRC peripheral passes its hardware's answer in.

## Three more things the design decided, and why

**The range check is in the map, not the handler.** A handler that had
to range-check would range-check in four places and one of them would
have the off-by-one — which is exactly the bug whose symptom is
exception 2 on the last address of a valid range. `mbmap.serves` is
written once, uses `mbpdu.req_last`, and a handler is called only for a
range that exists.

**Addresses are wire addresses.** Address 0 is the coil a manual calls
000001. The package does not convert, because the convention differs
per vendor and a library that guessed would be wrong for half its
users — but `mbmap.wire_from_manual` is a named place to do it, rather
than a `- 1` somewhere in a caller's own code where nobody can find it.

**A `@value` struct cannot be a boxed struct's field** (E2015), so
`MbClient` carries the six option numbers flat and `client_options`
assembles an `MbOptions` on demand. That keeps `MbOptions` unboxed
where it matters — in the retry arithmetic a poll loop runs every turn.

## What is deliberately outside

- **The file-access and diagnostic function codes** (7, 8, 11, 12, 17,
  20, 21, 43). Nine codes cover what a sensor, a drive and a gateway
  actually use; the rest are per-vendor in practice and each is a
  module over the same `MbReq` rather than a change to it.
- **Modbus ASCII.** A third framing nobody has specified a new device
  in for twenty years. It would be one more module over the same
  `mbpdu` if a consumer appeared.
- **A serial port.** `MbTransport[e]` is how one arrives, and
  `mbclient` is TCP because that is the transport the standard library
  has.
- **A device profile.** What register 40001 *means* is the device's,
  and a library that shipped a guess would be wrong for everything
  except the one device it was written against.

## Missing rows this package found

- **A serial port in the standard library.** `std.net` has sockets and
  `std.fs` has files, and there is no surface for a baud rate, a parity
  setting or an inter-character timeout. `mbclient` is TCP-only for
  that reason, and an RTU client on a host has to bring its own
  transport — which, for a protocol whose serial half is the older and
  more widely deployed one, is the biggest gap this package met.
- **modbus-ascii**, if a consumer appears. Named here so the decision
  is recorded rather than rediscovered.
- **A `@value` struct cannot be a fn value's parameter or a `Result`
  payload** (E2015), which shaped three surfaces here — `last_response`
  is a separate reader rather than a second return value, `MbClient`
  carries its options flat, and `mbmap.response_for` is separate from
  `mbmap.serve`. The constraint is recorded in the CHANGELOG rather
  than worked around.

## Licence

Apache-2.0.

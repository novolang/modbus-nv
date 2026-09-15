# modbus-nv

**Modbus** is a request-and-reply protocol by which one machine reads and
writes numbered values in another. It is how a programmable controller talks
to a variable-speed drive, a flow meter or a temperature transmitter. It is
specified by the Modbus Organization in
[Application Protocol v1.1b3](https://modbus.org/specs.php), with
[Serial Line v1.02](https://modbus.org/specs.php) and
[Messaging on TCP/IP v1.0b](https://modbus.org/specs.php) for the two ways of
putting it on a wire. This package brings both to novo-lang, on the client
side and the server side. It is built on
[crc-nv](https://novo-lang.org/packages/crc-nv).

**Status: NOT IMPLEMENTED — interface only.** Every function is declared with
its full signature, but every body is a `todo()` that panics when called. The
package is published so its design can be reviewed and depended on before it
is implemented. Version 0.1.0 will be the first working release.

## What Modbus is

A **client** sends a request and a **server** answers it. The request is a
**function code** and its arguments. Together they are the **protocol data
unit**, which is the part of the message that is the same on every wire.

A server holds four separate spaces of numbered values, and an address means
a different thing in each.

| Space | Width | Written by a client | Function codes |
| --- | --- | --- | --- |
| Coils | 1 bit | yes | 1, 5, 15 |
| Discrete inputs | 1 bit | no | 2 |
| Holding registers | 16 bits | yes | 3, 6, 16, 23 |
| Input registers | 16 bits | no | 4 |

An **exception** is a reply in which the server says it will not do what was
asked. The reply's function code has its top bit set and one byte says why:
the function is not supported, the address does not exist on this device, the
value is out of range, and so on.

Two ways of putting a protocol data unit on a wire matter today.

**Modbus RTU** runs over a serial line, usually RS-485, with many servers on
one pair of wires. A frame is a one-byte server address, the protocol data
unit, and a two-byte checksum. Nothing marks where a frame ends: what
separates two frames is **3.5 character times of silence** on the line.

**Modbus TCP** runs over a socket. A frame is a seven-byte header called the
**MBAP header** — a transaction identifier, a protocol identifier, a length
and a unit identifier — followed by the protocol data unit. The length field
means a receiver counts rather than waits.

| Quantity | Value |
| --- | --- |
| Function codes this package implements | 1 to 6, 15, 16, 23 |
| Longest protocol data unit | 253 bytes |
| Longest RTU frame, longest TCP frame | 256, 260 bytes |
| RTU frame overhead: address and checksum | 3 bytes |
| MBAP header | 7 bytes |
| TCP service port | 502 |
| Server addresses on a serial line | 1 to 247, and 0 for a broadcast |
| Coils read at once, registers read at once | 2000, 125 |
| Coils written at once, registers written at once | 1968, 123 |
| Function 23: registers read, registers written | 125, 121 |
| Bits per character on a serial line | 11 |
| Silence between frames at 19200 baud and above | 1750 µs |
| Highest address in any space | 65535 |
| Checksum of `123456789` under CRC-16/MODBUS | 0x4B37 |

## Install

```
novo pkg add modbus-nv
```

## Example

```novo
use std.bytes
use mbclient
use mbpdu
use mbtcp
use mberr

fn main() [io, net, time]
    match mbclient.connect("192.0.2.1", mbtcp.MB_TCP_PORT, mbclient.default_options())
        // Nobody answered. This is the failure that is worth retrying.
        Err(e) =>
            if mberr.is_retryable(e)
                println("retry")
            else
                println(e.message())
        Ok(c) =>
            // Ten holding registers starting at wire address 0.
            match mbclient.read_holding(c, 0, 10)
                Err(e) => println(e.message())
                Ok(data) =>
                    // The transport worked. What the server said is a
                    // separate question, and this is where it is asked.
                    let r = mbclient.last_response(c)
                    if mbpdu.rsp_is_exception(r)
                        // The server understood the request and refused it.
                        // Retrying that refusal will never succeed.
                        println(mbpdu.exception_name(mbpdu.rsp_exception(r)))
                    else
                        println("${bytes.len(data)} bytes of registers")
```

Build and test with `novo pkg build` and `novo test`. Today `novo test` fails
on purpose: every test reaches a `not implemented: modbus-nv.<module>.<fn>`
panic. The tests are the specification the implementation will have to
satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `mbpdu` | The request and the reply as values, the nine function codes, the per-code quantity limits, the exception codes, and the byte-count arithmetic both sides do. |
| `mbrtu` | The serial frame: the address byte, the checksum's byte order, how many bytes a frame will be once its header is read, and the three timings. |
| `mbtcp` | The MBAP header, framing by counting, pairing a reply with its request, and the conversion to and from an RTU address for a gateway. |
| `mbmap` | A server's address map: which spaces exist and where, the range check, the handler trait, and the bit and word packing a reply needs. |
| `mbcrc` | CRC-16/MODBUS, one-shot and streaming. The one module that reaches crc-nv. |
| `mberr` | The transport faults, and the two questions a caller asks of one. |
| `mbclient` | A client over TCP: the connection, the nine requests, the retry arithmetic and the poll schedule. |

## How to choose an entry point

**A gateway or a server on a device links `mbpdu`, `mbrtu`, `mbtcp` and
`mbmap`.** Add `mbcrc` when the part has no checksum peripheral. None of
those modules declares an effect.

**A client program links `mbclient`.** It opens the socket and keeps the
timeout and the retry policy.

**Implement `mbmap.MbHandler[e]` to be a server.** The map makes the range
check, so the handler is called only for addresses that exist. The trait's
effect parameter means one handler can cost input and output over a real
sensor and nothing at all in a test.

**Use `mbpdu` and `mbrtu` alone for a serial client on a host.** The standard
library has no serial port, so the bytes are the caller's to move. See "What
is not included".

## The rules a user needs

1. **A refusal by the server and a silence on the wire are different
   failures, and they need opposite reactions.** A transport fault — a
   timeout, a checksum mismatch, a closed socket — means nobody answered, and
   the request may be retried. An exception means the server received the
   request, understood it, and refused it. Retrying exception 2, "that
   address does not exist on this device", retries something that will never
   succeed.
2. **The exception is a field on the reply, not an error value.**
   `mbpdu.rsp_is_exception` asks, `mbpdu.rsp_exception` reads the code, and
   `mberr.MbError` has no exception arm.
3. **Check the quantity before sending.** Each function code has its own
   maximum, listed in the table above. A request past its limit gets
   exception 3 from a conforming server and silence from a bad one.
   `mbpdu.max_count` and `max_write_count` are that table.
4. **An RTU frame has no delimiter, and this package has no clock.** What
   separates two frames is 3.5 character times of silence.
   `mbrtu.silence_us` answers the figure and the caller times it, because the
   thing that measures the gap is a serial interrupt on a device and a serial
   port's inter-character timeout on a host.
5. **Above 19200 baud the silence is a fixed 1750 microseconds.** Serial Line
   v1.02 says so, because 3.5 characters at 115200 baud is 304 microseconds
   and nothing running an operating system can measure that.
6. **Addresses are wire addresses.** Address 0 is the coil a manual calls
   000001. This package converts nothing, because the convention differs by
   vendor. `mbmap.wire_from_manual` and `manual_from_wire` are the named
   place to do it, rather than a subtraction somewhere in a caller's own
   code.
7. **The range check belongs to the map, not to the handler.**
   `mbmap.serves` uses `mbpdu.req_last` and is written once. A handler that
   range-checked would do it in four places and one of them would have the
   off-by-one, whose symptom is exception 2 on the last valid address.
8. **A broadcast expects no reply.** Address 0 on a serial line goes to every
   server and none of them answers. `mbrtu.is_broadcast` and
   `mbrtu.expects_reply` are the two questions.
9. **A TCP reply is paired by transaction identifier and by unit
   identifier.** `mbtcp.mbap_answers` compares both. A gateway with several
   serial servers behind one socket gives them different unit identifiers,
   and a client that matched on the transaction alone would accept the wrong
   one.
10. **Take the checksum as a number the caller supplies.** Every module
    except `mbcrc` does, so a gateway whose part has a checksum peripheral
    passes its hardware's answer in.
11. **`MbClient` carries its six options as separate fields.** A `@value`
    struct may not be a field of a boxed struct (E2015), and `MbOptions` is
    unboxed so that the retry arithmetic a poll loop runs every turn costs
    nothing. `mbclient.client_options` assembles one on demand.

## Running on a microcontroller

The package states that its modules run on a device with no heap allocator,
and the compiler checks that claim on every build. Six of the seven modules
declare no effects and are inside it. The consumer that decided this is a
gateway: a device with a serial port on one side and a radio on the other,
decoding a request in a serial interrupt on a part with 64 KiB of memory.

`tests/embedded_probe.nv` is that claim as a program that either builds or
does not. It covers `mbpdu`, `mbrtu` and `mbtcp`.

```bash
novo build --target=nrf52-qemu tests/embedded_probe.nv
```

That command was run against this release. It produces a Cortex-M4
executable, `embedded_probe.elf`. The probe builds; it is not run, because
every function it calls is a `todo()` that would panic on the first line.

`mbclient` is outside the claim. It opens a socket and reads a clock, and one
host-only function anywhere in a compilation unit is an undefined symbol at
link time on a device, whether or not the firmware calls it. `mbcrc` is
outside the probe as well, which is why every other module takes a checksum
as a number.

## What is not included

- **A serial port.** The standard library has sockets and files and no
  surface for a baud rate, a parity setting or an inter-character timeout.
  `mbclient` is therefore TCP only, and an RTU client on a host brings its
  own transport through `mbclient.MbTransport[e]`. For a protocol whose
  serial half is the older and more widely deployed one, this is the largest
  gap the package meets.
- **The file-access and diagnostic function codes**, which are 7, 8, 11, 12,
  17, 20, 21 and 43. The nine implemented here are what a sensor, a drive and
  a gateway use. The rest are per-vendor in practice, and each would be a
  module over the same request type rather than a change to it.
- **Modbus ASCII.** A third framing, and nobody has specified a new device in
  it for twenty years. It would be one more module over the same protocol
  data unit if a consumer appeared.
- **A second copy of the checksum.** crc-nv's `crc16_modbus` is polynomial
  0x8005 reflected, initialised to all ones, with no final exclusive-or,
  which is the variant Modbus specifies rather than one of several. A
  checksum that disagrees with another implementation's is almost always the
  wrong variant rather than a wrong implementation.
- **A device profile.** What holding register 40001 means is the device's
  business, and a library that shipped a guess would be wrong for everything
  except the one device it was written against.

## Related packages

- [crc-nv](https://novo-lang.org/packages/crc-nv) is the checksum. Its
  published check value over `123456789` is 0x4B37.
- [can-nv](https://novo-lang.org/packages/can-nv) is the other industrial bus
  on the registry. CAN is a broadcast with no addresses in it and no
  request-and-reply. Modbus is one client asking one server.
- [heapless-nv](https://novo-lang.org/packages/heapless-nv) is where a
  gateway's frame buffers come from, on a part with no allocator.
- [bitfield-nv](https://novo-lang.org/packages/bitfield-nv) describes a
  register's bits. A holding register that packs several values is the same
  shape as a hardware register.
- `std.net` in the standard library is the socket `mbclient` opens.

## Tests

```bash
novo test                             # 100 tests
novo test tests/mbpdu_tests.nv        # 26: the function codes, the limits, the exceptions
novo test tests/mbframe_tests.nv      # 29: the RTU frame, the MBAP header, the timings
novo test tests/mbserver_tests.nv     # 27: the address map, the range check, the packing
novo test tests/mbclient_tests.nv     # 18: the retry arithmetic and the poll schedule
```

The numbers — the quantity limits, the exception codes, the 3.5-character
silence and the MBAP header — are transcriptions from the three Modbus
specifications rather than choices. The client and server shapes follow
[tokio-modbus](https://github.com/slowtec/tokio-modbus) and
[pymodbus](https://github.com/pymodbus-dev/pymodbus).

No test opens a socket or reads a clock. Every frame is a value the test
writes out and every timing is an argument.

The tests compile today and fail at run, each on the
`not implemented: modbus-nv.<module>.<fn>` panic that is its body. That is
the expected state of an interface release. They turn green one at a time as
bodies land.

## Implementation status

| Item | Implemented |
| --- | --- |
| Every `MB_*` constant in `mbpdu`, `mbrtu`, `mbtcp`, `mbmap`, `mbcrc` | yes (they are constants) |
| `mbpdu.MbReq`, `.MbRsp`, `mbtcp.MbMbap`, `mbmap.MbRegMap`, `mberr.MbError`, `mbclient.MbOptions`, `.MbClient` | declared |
| `mbmap.MbHandler[e]`, `mbclient.MbTransport[e]` | declared |
| `mbpdu`: the constructors, the accessors, the limits and the decoders | no |
| `mbrtu`: the frame, the checksum placement, the lengths and the timings | no |
| `mbtcp`: the header, the framing, the pairing and the gateway conversion | no |
| `mbmap`: the map, the range check, `serve`, `response_for` and the packing | no |
| `mbcrc.crc16`, `.crc16_start`, `.crc16_step`, `.crc16_update`, `.crc16_finish` | no |
| `mberr.describe`, `.is_retryable`, `.is_local` | no |
| `mbclient`: the connection, the nine requests, the retries and the schedule | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
